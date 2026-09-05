# Claude Code on FreeBSD

Running Claude Code natively on FreeBSD via the **Linuxulator** (Linux ABI
compatibility layer) — no VM, no Docker, no full Linux userland required.

This is a companion piece to
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd).
Both documents answer the same question — "Claude Code has no native BSD
build, now what?" — but the answers diverge completely depending on which
BSD you're on, and that divergence is the interesting part.

## Why this is harder than it sounds

As of the versions this was written against (2.1.2xx), the
`@anthropic-ai/claude-code` npm package isn't a portable JavaScript CLI
with a few native add-ons — it's a thin installer wrapped around a
**precompiled native binary per platform**. `npm install` runs a
postinstall script that detects `process.platform`/`arch`, pulls the
matching `@anthropic-ai/claude-code-<platform>` optional dependency
(`linux-x64`, `linux-arm64`, `darwin-arm64`, `win32-x64`, …), and drops
that binary in place of a placeholder file. After install, running
`claude` never touches Node.js or JavaScript again — it execs the native
binary directly. (There's a JS fallback launcher, `cli-wrapper.cjs`, but
per its own comment it "is never invoked" unless you installed with
`--ignore-scripts`.)

The install matrix covers Linux, macOS and Windows. There's no
`freebsd-x64` entry, and — see [Prior art](#prior-art-and-context) below
— it isn't for lack of anyone asking.

So the real problem isn't "run a Node CLI on FreeBSD" (trivial, `pkg
install node`). It's "get a genuine `linux-x64` build of *both* Node
(needed once, to drive the installer) *and* the native `claude` binary
it fetches" running under FreeBSD, without asking users to babysit a
virtual machine for a CLI tool they'll open fifty times a day.

## The OpenBSD answer, briefly

OpenBSD has no Linux binary compatibility layer at all (it had one,
decades ago; it was dropped on security grounds and never came back).
That leaves exactly one option: run an actual Linux kernel. The
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd)
writeup covers it in full — Alpine Linux under OpenBSD's native `vmd`
hypervisor (or Debian under QEMU/TCG if you don't have VT-x), then Docker
inside that VM, then `node:22-slim` inside the container. Three layers of
indirection to run one CLI tool. It works, and OpenBSD's own perimeter
(`pf`, `doas`, `pledge`) stays the real security boundary around the
whole stack — but every `claude` invocation pays for two extra
virtualization layers it doesn't structurally need.

## The FreeBSD answer: skip the VM entirely

FreeBSD kept (and actively maintains) the **Linuxulator** — a kernel-level
translation layer that maps Linux syscalls onto FreeBSD's own kernel
primitives. A genuine Linux ELF binary, dropped in the right place, just
runs. No hypervisor, no second kernel, no container runtime. This is the
same mechanism that lets FreeBSD run Steam, Skype clients, and a fair
amount of enterprise software that only ships Linux binaries — and it
works just as well on a binary an npm postinstall script fetched for you
as on one you placed by hand.

### 1. Turn the Linuxulator on

```sh
sysrc linux_enable=YES
service linux start
```

This loads the kernel modules (`linux64.ko`, `linux_common.ko`, plus
whatever `linux.ko` needs) and mounts the compat filesystems:

```
linprocfs on /compat/linux/proc
linsysfs  on /compat/linux/sys
devfs     on /compat/linux/dev
fdescfs   on /compat/linux/dev/fd
tmpfs     on /compat/linux/dev/shm
```

**Gotcha**: if you `pkg install` a `linux_base-*` package *before* running
`service linux start`, the pre-install script fails with `Cannot install
package: kernel missing 64-bit Linux support` — the check runs against
whatever's loaded *right now*, not what's in `rc.conf`. Enable the
service first, install second.

### 2. Install a Linux userland base

```sh
pkg install linux_base-rl9
```

Rocky Linux 9 (glibc 2.34), not the older `linux_base-c7` (CentOS 7,
glibc 2.17) that a lot of FreeBSD Linuxulator guides still default to.
Claude Code's native binary wants a considerably newer glibc than CentOS
7 ships; `rl9` is the base most current guides converge on for anything
built after ~2022.

### 3. Fetch real Linux Node.js — not the FreeBSD port

Node itself is only needed to run `npm install`'s postinstall script,
which does the platform detection and fetches the real `claude` binary.
Grab the official Linux tarball rather than FreeBSD's native `node` port
(the native port would correctly report `process.platform === 'freebsd'`
to the installer — which isn't in its platform table at all):

```sh
NODE_VER=22.14.0   # or track https://nodejs.org/dist/latest-v22.x/
cd /tmp
fetch "https://nodejs.org/dist/v${NODE_VER}/node-v${NODE_VER}-linux-x64.tar.xz"
mkdir -p /compat/linux/opt
tar xJf "node-v${NODE_VER}-linux-x64.tar.xz" -C /compat/linux/opt
mv "/compat/linux/opt/node-v${NODE_VER}-linux-x64" /compat/linux/opt/node
```

Sanity check — this alone confirms the Linuxulator is doing its job,
before Claude Code enters the picture at all:

```sh
/compat/linux/opt/node/bin/node --version
# v22.14.0
```

If that prints a version instead of an exec-format error, the syscall
translation is working and the rest is just npm.

### 4. Install Claude Code

```sh
export PATH=/compat/linux/opt/node/bin:$PATH
npm install -g @anthropic-ai/claude-code
```

Because this `npm install` runs on a genuinely Linux-built Node (via the
Linuxulator), `process.platform` correctly reports `linux`, so the
postinstall script fetches the real `linux-x64` binary — not a stub, not
the wrong architecture. Confirm it landed:

```sh
ls -la /compat/linux/opt/node/lib/node_modules/@anthropic-ai/claude-code/bin/
# claude.exe  -> a real Linux ELF binary, despite the extension:
file /compat/linux/opt/node/lib/node_modules/@anthropic-ai/claude-code/bin/claude.exe
# ELF 64-bit LSB executable, x86-64, ... for GNU/Linux 3.2.0, ...
```

`bin/claude.exe` is a fixed filename the build tooling uses as a
placeholder/target across *all* platforms — on Windows it really is a
`.exe`; on Linux, postinstall just copies the native ELF binary over
that same filename. The `.exe` you're looking at post-install on Linux
is not a mistake and not a Windows binary; it's this package's naming
convention for "wherever `bin.claude` in `package.json` points."

### 5. Make it reachable from a normal FreeBSD shell

```sh
claude --version
# claude: command not found
```

`npm -g` installed it *inside* the Linux root, so nothing on the native
FreeBSD `$PATH` points at it yet. A one-line wrapper fixes that
permanently — and disables the auto-updater, so a `claude` invocation
mid-session never rewrites the binary you're currently running under the
Linuxulator out from under itself:

```sh
cat > /usr/local/bin/claude << 'EOF'
#!/bin/sh
export PATH=/compat/linux/opt/node/bin:$PATH
export DISABLE_AUTOUPDATER=1
exec /compat/linux/opt/node/bin/claude "$@"
EOF
chmod +x /usr/local/bin/claude
```

Update deliberately, when you choose to, instead:

```sh
PATH=/compat/linux/opt/node/bin:$PATH npm update -g @anthropic-ai/claude-code
```

**Gotcha I actually hit**: the obvious-looking alternative —

```sh
exec /compat/linux/opt/node/bin/node /compat/linux/opt/node/bin/claude "$@"
```

— fails:

```
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".exe" for
.../claude-code/bin/claude.exe
```

This has nothing to do with the file being a Windows binary (it isn't —
see step 4). It's `node`'s own module loader refusing to treat a file
named `.exe` as a script it can parse, full stop — it never gets far
enough to notice the file is actually a native ELF executable. `claude`
isn't a JavaScript entry point at all after install; there's no `node
whatever.js` invocation that's ever correct here. Exec the binary
(`.../bin/claude.exe`, or the `bin/claude` symlink `npm -g` creates
pointing at it) directly, never through `node`.

### Verify

```sh
claude --version
# 2.1.261 (Claude Code)
```

From any shell, any working directory, no VM booted, no container
running.

## If Claude hangs with no output at all

Not something this setup hit (`service linux start` mounts things
correctly out of the box), but reported by others running the
Linuxulator with hand-rolled `/etc/fstab` entries or inside a jail:
`fdescfs` mounted without the `linrdlnk` option makes `/proc/self/fd`
symlinks resolve the FreeBSD way instead of the Linux way, and Claude's
native binary hangs indefinitely at startup with zero output — no error,
nothing to grep for. If you're mounting by hand rather than via `service
linux start`:

```
fdescfs /compat/linux/dev/fd fdescfs rw,linrdlnk,late
```

## Running this inside a jail

This cluster also runs FreeBSD jails, so the natural next question is
whether the setup above works isolated from the host. It does, with one
placement change: `linux_enable` and the Linuxulator kernel modules
belong on the **host**, never inside the jail — a jail can't load kernel
modules. What moves *into* the jail is everything under `/compat/linux`
(the base, Node, the `claude` install) and, if you're managing jails with
Bastille or iocage, the `linux=enable`-style compat mount options belong
in the jail manager's own config, not in a jail-internal `/etc/fstab` —
the jail doesn't own those mounts, the host does, on the jail's behalf.

## Side by side

| | OpenBSD | FreeBSD |
|---|---|---|
| Compat mechanism | none (dropped, security decision) | Linuxulator (kernel-level, maintained) |
| What actually runs the `claude` binary | Alpine Linux kernel, in a `vmd` VM | the FreeBSD kernel itself |
| Extra layers | VM + Docker + container runtime | none |
| Per-invocation overhead | full second-kernel scheduling + container | syscall translation only |
| Security perimeter | OpenBSD `pf`/`doas`/`pledge` around the VM | native FreeBSD `pf`/`doas`, no extra boundary needed |
| Setup surface | VM image, `vm.conf`, Docker daemon, `Dockerfile` | one kernel module, one base package, one tarball |

Neither approach is "wrong" — they're the only two approaches each OS
*has*. OpenBSD's total lack of Linux compat is a deliberate, longstanding
security stance, not an oversight, and it buys real isolation the
Linuxulator doesn't offer (a Linux binary under the Linuxulator is still
talking to the *same* FreeBSD kernel, with no VM boundary between it and
the host). If the workload actually needs that isolation, the OpenBSD
answer's extra weight is the point, not a cost. For a CLI tool invoked
directly by a logged-in operator, FreeBSD's answer is the cheaper trade.

## Prior art and context

This isn't undiscovered territory — worth knowing about before you
reinvent it:

- **[`emulators/claude-code`](https://www.freshports.org/emulators/claude-code/)
  is already in the FreeBSD ports tree** (`pkg install claude-code`),
  presumably built on the same Linuxulator mechanism documented here.
  Version-lagged behind upstream at the time of writing (`2.1.204` in
  ports vs. `2.1.261` fetched directly via npm above) — plausibly a
  side effect of the per-platform-binary distribution model described
  above: a port maintainer has to notice and repackage the right native
  binary each release, rather than the package just recompiling from
  portable source.
- **[`insanityinside/claude-freebsd`](https://github.com/insanityinside/claude-freebsd)**
  — an actively maintained shell-script installer automating this same
  Linuxulator approach, including its own checks for the mount gotchas
  above.
- **[`anthropics/claude-code#30640`](https://github.com/anthropics/claude-code/issues/30640)**
  — "Native installer doesn't work on FreeBSD," 60 comments, closed
  without a native FreeBSD build landing. The Linuxulator route above is
  the practical answer in the meantime.

## Environment this was verified on

- FreeBSD 15.1-RELEASE, amd64
- `linux_base-rl9` 9.7
- Node.js v22.14.0 (official linux-x64 tarball)
- `@anthropic-ai/claude-code` 2.1.261

---

Part of a small series on running developer tooling on BSD without
reaching for a VM by default. See also:
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd).
