# Claude Code on FreeBSD

Running Claude Code natively on FreeBSD via the **Linuxulator** (Linux ABI
compatibility layer) — no VM, no Docker, no full Linux userland required.

This is a companion piece to
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd).
Both documents answer the same question — "Claude Code has no native BSD
build, now what?" — but the answers diverge completely depending on which
BSD you're on, and that divergence is the interesting part.

## Why this is harder than it sounds

Claude Code ships as a Node.js CLI distributed via npm. The install matrix
covers Linux, macOS and Windows (WSL) — there is no `freebsd` or `openbsd`
target. Node.js itself has a FreeBSD port, and in principle a pure-JS npm
package would just run on top of it. In practice, Claude Code isn't pure
JS: it pulls in prebuilt native binaries (bundled per-platform) that only
ship for `linux-x64`, `linux-arm64`, `darwin` and `win32`. There's no
`freebsd-x64` variant to fetch, and no upstream plan to add one — BSD is a
tiny fraction of the developer desktop market.

So the real problem isn't "install Node on FreeBSD" (that part is a
one-line `pkg install`). It's "get a genuine `linux-x64` build of Node,
and everything Claude Code needs, running under FreeBSD" without asking
users to babysit a virtual machine for a CLI tool they'll open fifty
times a day.

## The OpenBSD answer, briefly

OpenBSD has no Linux binary compatibility layer at all (it had one, once,
decades ago; it was dropped on security grounds and never came back).
That leaves exactly one option: run an actual Linux kernel. The
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd)
writeup covers it in full — Alpine Linux under OpenBSD's native `vmd`
hypervisor (or Debian under QEMU/TCG if you don't have VT-x), then Docker
inside that VM, then `node:22-slim` inside the container. Three layers of
indirection to run one CLI tool. It works, and OpenBSD's own perimeter
(`pf`, `doas`, `pledge`) stays the real security boundary around the
whole stack — but it's heavy, and every `claude` invocation pays for two
extra virtualization layers it doesn't need.

## The FreeBSD answer: skip the VM entirely

FreeBSD kept (and actively maintains) the **Linuxulator** — a kernel-level
translation layer that maps Linux syscalls onto FreeBSD's own kernel
primitives. A genuine Linux ELF binary, dropped in the right place, just
runs. No hypervisor, no second kernel, no container runtime. This is the
same mechanism that lets FreeBSD run Steam, Skype clients, and a fair
amount of enterprise software that only ships Linux binaries.

That's the whole trick: fetch the official Linux build of Node.js
directly from nodejs.org (not from FreeBSD's package repo — that one is
a native FreeBSD binary, which won't run Claude Code's bundled Linux
native modules), unpack it under the Linuxulator's root, and point npm's
global install at it.

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
Claude Code's Node.js build wants a considerably newer glibc than CentOS
7 ships; `rl9` is the base most current guides converge on for anything
built after ~2022.

### 3. Fetch real Linux Node.js — not the FreeBSD port

```sh
cd /tmp
fetch https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz
mkdir -p /compat/linux/opt
tar xJf node-v22.14.0-linux-x64.tar.xz -C /compat/linux/opt
mv /compat/linux/opt/node-v22.14.0-linux-x64 /compat/linux/opt/node
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

### 5. Make it reachable from a normal FreeBSD shell

```sh
claude --version
# claude: command not found
```

`npm -g` installed it *inside* the Linux root, so nothing on the native
FreeBSD `$PATH` points at it yet. A one-line wrapper fixes that
permanently:

```sh
cat > /usr/local/bin/claude << 'EOF'
#!/bin/sh
export PATH=/compat/linux/opt/node/bin:$PATH
exec /compat/linux/opt/node/bin/claude "$@"
EOF
chmod +x /usr/local/bin/claude
```

**Gotcha**: the obvious-looking alternative —

```sh
exec /compat/linux/opt/node/bin/node /compat/linux/opt/node/bin/claude "$@"
```

— fails:

```
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".exe" for
.../claude-code/bin/claude.exe
```

`npm`'s global bin entry (`claude`) is a *shim script*, not the JS entry
point itself — it resolves platform and points `node` at the correct
real file. Calling `node` directly on the package's `bin` path bypasses
that resolution and grabs the bundled Windows stub instead. Exec the
shim (`.../bin/claude`), not `node` plus a guessed path.

### Verify

```sh
claude --version
# 2.1.261 (Claude Code)
```

From any shell, any working directory, no VM booted, no container
running.

## Side by side

| | OpenBSD | FreeBSD |
|---|---|---|
| Compat mechanism | none (dropped, security decision) | Linuxulator (kernel-level, maintained) |
| What actually runs Claude Code | Alpine Linux kernel, in a `vmd` VM | the FreeBSD kernel itself |
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

## Environment this was verified on

- FreeBSD 15.1-RELEASE, amd64
- `linux_base-rl9` 9.7
- Node.js v22.14.0 (official linux-x64 tarball)
- `@anthropic-ai/claude-code` 2.1.261

---

Part of a small series on running developer tooling on BSD without
reaching for a VM by default. See also:
[`claude-code-openbsd`](https://github.com/simplestringdev/claude-code-openbsd).
