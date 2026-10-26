# 🫧 foam - a bubblewrap wrapper

**foam** runs commands in a sandbox: primarly for AI agents but also for dev tooling like build tools, linters etc. It's a single Bash script around [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`), the same sandboxing tool Flatpak uses.

```bash
cd ~/projects/myapp
foam                      # interactive shell in a sandbox
foam -p myapp claude      # run a command in the "myapp" profile
```

Inside the sandbox, a program sees the current project directory, a read-only system and its own private home directory, plus network access if you allow it. Your real home, your SSH keys, your other projects, your desktop session and the rest of the host are out of reach.

Foam works the same directly on the host and inside distrobox or toolbox containers with any distro.

---

## Contents

- [Why](#why)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [What the sandbox can and can't do](#what-the-sandbox-can-and-cant-do)
- [Working directory rules](#working-directory-rules)
- [Profiles](#profiles)
- [Profile config reference](#profile-config-reference)
- [Git protection](#git-protection)
- [Audit mode](#audit-mode)
- [Network modes](#network-modes)
- [Seccomp filter](#seccomp-filter)
- [Testing that it works](#testing-that-it-works)
- [Limitations](#limitations)
- [Files and directories](#files-and-directories)

---

## Why

Development tools run a lot of code you never reviewed. Every `npm install` can execute install scripts from any package in the dependency tree, and an AI agent runs whatever commands it decides to. One compromised package or one bad decision is enough to:

- read your SSH keys, tokens, browser data and other projects
- plant something in your `~/.bashrc`, autostart or git hooks that runs later, outside any sandbox
- talk to your desktop session, password manager or container engine.

foam limits the damage to what the tool actually needs: the project you're working on and the tool's own home directory.

---

## Requirements

| Needed for | Requirement |
|---|---|
| Everything | `bash`, `bwrap` (package `bubblewrap`), GNU coreutils, findutils and sed. All of these are standard on Fedora, Debian, Ubuntu and Arch; BusyBox-based images like Alpine aren't supported. |
| Full protection | bubblewrap **0.8 or newer**, for `--disable-userns`. Older versions work but allow nested user namespaces. |
| Audit report | `git` |
| `seccomp yes` | `python3` and `libseccomp.so.2`, which is present wherever systemd is. No extra Python package is needed. |
| `net pasta` | `pasta`, package `passt` |

---

## Installation

```bash
install -Dm755 foam ~/.local/bin/foam
install -Dm644 foam.bash-completion ~/.local/share/bash-completion/completions/foam
```

`~/.local/bin` must be on your `PATH`. Your home directory is shared between the host and distrobox, so one installation serves both, as long as each environment has `bwrap`.

Tab completion needs the `bash-completion` package. It completes options and profile names. After the command name it completes plain file names, deliberately not the command's own completion; see [Limitations](#limitations).

---

## Usage

```
foam [-p PROFILE] [-t] [-u] [-n] [--] [COMMAND [ARGS...]]
```

| Option | Meaning |
|---|---|
| `-p PROFILE` | Profile to use (default: `default`) |
| `-t` | Throwaway home: an empty in-memory `$HOME`, discarded on exit |
| `-u` | Update mode: paths marked `protect` become writable, so you can install or update tools |
| `-n` | Dry run: print the exact sandbox command instead of running it |
| `-h` | Short help |

Without a command, foam starts an interactive Bash shell with the prompt `🫧[user@profile dir]$`.

```bash
foam                         # shell, profile "default"
foam make test               # one command
foam bash -c 'a | b'         # pipes and other shell syntax inside the sandbox
foam -p node npm test        # command in profile "node"
foam -t                      # throwaway home, nothing persists
foam -u -p node npm i -g x   # install a tool into a protected location
foam -n -p node              # see exactly what the sandbox gets
```

Shell syntax on foam's command line belongs to your outer shell: in `foam ls | grep x`, only `ls` runs in the sandbox. Use `foam bash -c '...'` to keep the whole pipeline inside.

Options only count before the command, so in `foam ls -la`, the `-la` goes to `ls`.

---

## What the sandbox can and can't do

foam builds the sandbox as an **allowlist**: the sandbox starts with an empty filesystem and nothing from the host exists inside unless foam adds it.

### Allowed

| What | Details |
|---|---|
| The current directory | Read-write, at its real path. This is the only real host directory a sandbox can write to. |
| Its profile home | Read-write and persistent: `~/.local/share/foam/<profile>/home`, mounted at your normal `$HOME` path. Each profile has its own, and `-t` gives an empty, temporary one instead. |
| The system | `/usr` and `/etc` read-only, plus `/bin`, `/lib` etc. as they exist on the system, so all installed programs work. |
| Private temp dirs | `/tmp`, `/var/tmp` and `/run` are empty, private and in memory, discarded on exit. |
| Network | Depends on the profile's [`net` setting](#network-modes): shared (default), none, or its own network through pasta. |
| Processes | Starting programs, threads and subprocesses, and debugging its own processes. |
| Git | `status`, `diff`, `log`, commits and branches. The parts of `.git` that can run code are read-only; see [Git protection](#git-protection). |
| Interactive work | A normal shell with history (per profile) and job control. Job control is lost only if your kernel still allows legacy `TIOCSTI` (`/proc/sys/dev/tty/legacy_tiocsti` isn't `0`) and `seccomp yes` is off. |

### Blocked

| What | How |
|---|---|
| Your real home, dotfiles, SSH and GPG keys, tokens, other projects | Not mounted. The sandbox sees its profile home instead. |
| Other profiles' homes and foam's configs | Not mounted, and foam refuses configs or working dirs that would expose them. |
| Host services: systemd, D-Bus, podman/docker, ssh-agent, gpg-agent, Wayland, PipeWire, X11 socket files, tmux | Host `/run` and `/tmp` aren't mounted, and foam refuses working dirs or `ro` paths that contain sockets. |
| Seeing or signalling other processes | Separate PID namespace. Inside, `ps` shows only the sandbox's own processes. |
| Devices: GPU, webcam, microphone, keyboards, disks, `/dev/kvm` | Minimal private `/dev` with only `null`, `zero`, `random`, `tty` and the like. |
| Gaining privileges: `sudo`, `su`, `pkexec`, setuid programs | The `no_new_privs` flag is set and all capabilities are dropped. |
| Creating new user namespaces | `--disable-userns` (bwrap 0.8+) and the seccomp filter. |
| Typing commands into your real terminal (`TIOCSTI`) | Blocked by the kernel (6.2+ when `legacy_tiocsti` is `0`), by the seccomp filter, or by a new session as a fallback. |
| Git hooks and code-running git config | Read-only. |
| Your environment variables (tokens, `SSH_AUTH_SOCK`, `DISPLAY`, ...) | The environment is rebuilt from scratch: `HOME`, `USER`, `LOGNAME`, `SHELL`, terminal, locale, `EDITOR`/`VISUAL`, and a cleaned `PATH`. |
| Fake commands planted in `PATH` | `PATH` entries inside your home, empty entries and relative entries (like `.`) are removed. Profiles add their own with `path`. |
| Inherited file descriptors | Everything except stdin, stdout and stderr is closed before the sandbox starts. |
| GUI apps | No display access, by design. (With `net yes`, see the X11 note under [Network modes](#network-modes).) |

With a profile option:

| Option | Additionally blocks |
|---|---|
| `net no` | All network access, including host localhost services and abstract sockets |
| `net pasta` | Host localhost services and abstract sockets (e.g. X11), while internet access keeps working |
| `seccomp yes` | Kernel keyring (Kerberos tickets), `TIOCSTI`/`TIOCLINUX`, io_uring, eBPF, userfaultfd, `perf_event_open`, mount syscalls, unusual socket types, 32-bit programs |

---

## Working directory rules

The working directory is the one place the sandbox can write on the host, so foam refuses directories where planted files would be dangerous:

| Refused | Why |
|---|---|
| `/`, your home, or any parent of your home | Would expose everything |
| Hidden directories in your home (`~/.config`, `~/.local`, `~/.ssh`, ...) | Dotfiles and app configs run code outside the sandbox |
| foam's config and data directories, and the directory foam is installed in | Would let a sandbox change its own rules |
| `$XDG_RUNTIME_DIR` (`/run/user/<uid>`), and anything containing `/run`, `/tmp`, `/var/tmp`, `/dev`, `/proc` or `/sys` | Full of host service sockets |
| Directories in your `PATH` (e.g. `~/bin`) | A planted command would run on the host |
| Directories with a socket or FIFO within 3 levels (`.git` aside) | The sandbox could talk to whatever listens there |

Subdirectories such as `/tmp/build`, or a USB drive under `/run/media`, are fine.

---

## Profiles

A **profile** is a separate, persistent home directory for the sandbox, plus an optional config file.

| | Path |
|---|---|
| Home | `~/.local/share/foam/<profile>/home` |
| Config | `~/.config/foam/<profile>.conf` (optional) |

Tools in different profiles can't read each other's configs, caches or tokens. A good rule of thumb:

- **Group tools that work together** and hold no secrets of their own, e.g. a `node` profile with npm, eslint, prettier and TypeScript.
- **Give credential-holding tools their own profile,** e.g. an AI agent with its API token, `gh`, or a cloud CLI. A malicious npm package in `node` then can't steal those tokens.
- **Use `-t`** for anything you're just trying out.

Two sandboxes using the same profile at the same time share its home live. Different profiles share nothing.

### Installing tools inside a profile

Install tools *inside* their profile, so their install scripts run sandboxed too:

```
# ~/.config/foam/node.conf
env NPM_CONFIG_PREFIX=~/.npm-global
path ~/.npm-global/bin
protect .npm-global
```

```bash
foam -u -p node npm install -g typescript   # -u: the protected dir is writable
foam -p node tsc --version                  # normal runs: tools can't be modified
```

A thin wrapper makes a sandboxed tool feel native. For example, `~/.local/bin/tsc`:

```bash
#!/bin/sh
exec foam -p node tsc "$@"
```

This doesn't loop: inside the sandbox, `~/.local/bin` belongs to the profile home and doesn't contain the wrapper.

### Managing profiles

```bash
ls ~/.local/share/foam/                 # list profiles
rm -rf ~/.local/share/foam/node         # reset a profile (reinstall tools with -u)
cp -a ~/.local/share/foam/node backup/  # back one up
```

---

## Profile config reference

Configs are **parsed, never executed**: no shell code runs from them, and a typo is reported with its file and line number.

- One directive per line.
- Lines *starting* with `#` are comments; a `#` later in a line is part of the value.
- A leading `~` means your home. Nothing else is expanded: no variables, no quotes.

| Directive | Meaning |
|---|---|
| `ro PATH` | Read-only access to a real file or directory, e.g. `ro ~/.gitconfig`. Skipped if it doesn't exist. Refused for `/run`, `/tmp`, `/var/tmp`, `/dev`, `/proc`, `/sys`, `$XDG_RUNTIME_DIR`, foam's own directories, and anything containing a socket or FIFO, because read-only doesn't stop connecting to a socket. Symlink targets are checked too. |
| `protect NAME` | A file or directory **directly** in the profile home (no `/`) stays read-only, except in `-u` runs. It must already exist (create it with a `-u` run) and must not be a symlink. |
| `path DIR` | Prepend `DIR` to `PATH` inside the sandbox. Several lines keep their order. |
| `env NAME=VALUE` | Set an environment variable inside the sandbox. `PATH`, `HOME` and `FOAM*` can't be set this way. |
| `net yes\|no\|pasta` | Network mode; see [Network modes](#network-modes). Default `yes`. |
| `seccomp yes\|no` | Syscall filter; see [Seccomp filter](#seccomp-filter). Default `no`. |
| `audit yes\|no` | Report all changes after each run; see [Audit mode](#audit-mode). Default `no`. |

Example profiles:

```
# ~/.config/foam/node.conf - npm and friends
ro ~/.gitconfig
env NPM_CONFIG_PREFIX=~/.npm-global
path ~/.npm-global/bin
protect .npm-global
protect .npmrc
protect .bashrc
seccomp yes
audit yes
```

```
# ~/.config/foam/agent.conf - an AI agent with its own token in its own home
net pasta
seccomp yes
audit yes
```

```
# ~/.config/foam/offline.conf - tests that need no network at all
net no
seccomp yes
```

`protect` only protects a path from *normal* runs. A `-u` run can be influenced by any writable file it reads, so also protect files like `.npmrc` and `.bashrc` in the profile home, and run `-u` commands non-interactively.

---

## Git protection

Host git (yours, your shell prompt's, your IDE's) runs code that a repository's `.git` names: hooks, plus commands in its config (`core.fsmonitor`, filters, pager, aliases, ...). A sandbox that could write those would get code execution on the host the next time you run `git status`, or simply `cd` into the repo with a git-aware prompt.

So in every repository up to 3 levels below the working directory (not inside dependency and build directories: node_modules, vendor, .venv, venv, \_\_pypackages__, .gradle), these parts of `.git` are **read-only**:

| Path | Why |
|---|---|
| `config`, `config.worktree` | Commands git runs, includes, `core.hooksPath` |
| `hooks/` | Scripts run by `git commit`, `push`, ... |
| `info/` | `exclude` can hide files from `git status`; `attributes` can hide diffs |
| `refs/replace/` | Replace refs change what `git log` and `git show` display |
| `modules/`, `worktrees/` | Submodule and worktree git dirs, each with its own config and hooks |

Objects, branches, tags, the index and logs stay writable, so **commits work inside the sandbox**. Commands that write the config fail inside; do these on the host:
- `git remote add`
- `git branch --set-upstream-to`
- `git config`
- `git worktree add`
- `git submodule update`

To make this work, foam creates missing `hooks/`, `info/` and `refs/replace/` directories and an empty `config.worktree` (otherwise the sandbox could create them itself). Empty ones change nothing for git. If any of these paths is a symlink, foam refuses to run: git never creates symlinks there, and bwrap would mount whatever one points to.

If foam finds more than one repository below the working directory, it warns. A sandbox can affect tools in all of them by planting files in their shared parent (e.g. a `node_modules` or `eslint.config.js`), so run foam inside a single project.

---

## Audit mode

With `audit yes`, foam reports everything the sandbox changed when it exits:

```
foam: changes during this run (full list: ~/.local/share/foam/node/last-changes.txt)
  new commits (review: git diff 2bf73dec3ac5 HEAD):
    fbab39c change a
  changed branches/tags:
    - refs/heads/master 2bf73de
    + refs/heads/master fbab39c
  git status:
     M src/index.js
    ?? new.txt
  project, files per top-level entry (incl. git-ignored):
       1289 node_modules
          1 src
          1 new.txt
  profile home, files per top-level entry:
         34 .npm
  WARNING: hooks changed in .husky (core.hooksPath), they run on the host:
    .husky/pre-commit
```

| Reported | How |
|---|---|
| Changed files | Every file whose **ctime** is newer than the start of the run, in the project (including git-ignored files) and the profile home. Unlike mtime, ctime can't be faked from userspace: setting an old date with `touch -d` updates it. |
| New commits and moved branches/tags | Compared with the state before the run |
| `git status` | Run on the host, pinned to the repository found before the run, with code-running settings switched off |
| Warnings | Replace refs, changed hooks in a tracked `core.hooksPath` directory (e.g. husky), and new nested `.git` directories, including one planted directly in the working directory |

The report goes to stderr. The full list is saved to `~/.local/share/foam/<profile>/last-changes.txt`, a place no sandbox can reach. File names and commit messages are sanitized, so terminal escape sequences or newlines in names can't hide or fake report lines.

Some things to know:
- `git status` shows *all* uncommitted changes, not just this run's; the file list is strictly this run. Committing or stashing between runs makes both match.
- Deleted git-ignored files aren't listed. Deleted tracked files show in `git status`.
- In audit mode, foam stays running as the sandbox's parent instead of replacing itself, so it can print the report. `Ctrl+C` still reaches the sandbox.

---

## Network modes

| `net` | Internet | Host localhost services | Abstract sockets (e.g. X11) |
|---|---|---|---|
| `yes` (default) | ✅ | ⚠️ reachable | ⚠️ reachable |
| `pasta` | ✅ | ❌ blocked | ❌ blocked |
| `no` | ❌ | ❌ blocked | ❌ blocked |

**Why `yes` isn't fully isolated:** sharing the host's network namespace also shares two things that aren't the internet:
- **Localhost services:** a browser's remote-debugging port, Jupyter, databases without auth, Docker over TCP, and so on.
- **Abstract Unix sockets,** which belong to the network namespace rather than the filesystem. Xorg and Xwayland listen on one, and GNOME allows same-user X11 connections without a cookie. A sandbox could then log keystrokes from X11 apps and inject input into them.

**`net pasta`** gives the sandbox its own network namespace. [pasta](https://passt.top) (also used by rootless podman) turns its traffic into normal connections from the host:
- no ports are forwarded in either direction
- the gateway address doesn't lead to the host's loopback (`--no-map-gw`)
- DNS goes through pasta's forwarder, since systemd-resolved listens on the host's loopback.

bwrap runs as root inside pasta's namespace, so foam maps your user and group IDs back and drops all capabilities. Servers started inside a `net pasta` sandbox aren't reachable from the host.

---

## Seccomp filter

`seccomp yes` adds a syscall filter, modeled on Flatpak's and built at each run by calling libseccomp directly from Python. Everything not listed is allowed.

| Blocked | Why |
|---|---|
| `add_key`, `keyctl`, `request_key` | The kernel keyring isn't namespaced and holds e.g. Kerberos tickets (Fedora's default credential cache) |
| `ioctl` `TIOCSTI`, `TIOCLINUX` | Typing into your terminal, console paste injection. Compared on 32 bits, so junk in the upper half of the argument can't slip past. |
| New user namespaces (`clone`/`unshare` with `CLONE_NEWUSER`) | Large kernel attack surface. |
| io_uring, `bpf`, `userfaultfd`, `perf_event_open`, `syslog`, `modify_ldt`, `uselib` | Rarely needed, and common starting points for kernel exploits |
| `mount`, `umount2`, `pivot_root`, `chroot`, new mount API | Not needed in the sandbox |
| Sockets other than Unix, IPv4, IPv6 and netlink | Modules for rarely used protocols are a classic source of kernel bugs |
| 32-bit (and x32) syscalls | Killed, so they can't be used to get around the rules |

Side effects:
- **32-bit programs are killed** (Wine, Steam, some old SDK tools).
- **Tools that need the kernel keyring fail inside,** e.g. `kinit`. That's intended.
- **Some programs quietly use a fallback:** Node.js drops io_uring and glibc drops `clone3`.
- **Job control works on every kernel,** because the filter blocks `TIOCSTI` itself, so foam doesn't need its new-session fallback.

---

## Testing that it works

These tests check the sandbox on your real system. Each test lists the expected result. If a test shows **NOT protected**, stop and investigate before trusting foam with real work.

### Setup

```bash
mkdir -p ~/.config/foam ~/foam-test && cd ~/foam-test
printf 'net no\n'                 > ~/.config/foam/t-offline.conf
printf 'seccomp yes\n'            > ~/.config/foam/t-seccomp.conf
printf 'net pasta\nseccomp yes\n' > ~/.config/foam/t-pasta.conf
printf 'audit yes\n'              > ~/.config/foam/t-audit.conf
foam -n | head -20    # dry run: read what the sandbox gets
```

### 1. Filesystem isolation

```bash
foam ls -a ~                      # expect: profile home only, not your real files
foam ls ~/.ssh                    # expect: No such file or directory
foam ls -a /tmp                   # expect: empty
foam ls -a /run                   # expect: foam.bashrc (plus systemd/ if resolv.conf points there),
                                  #         no user/, dbus/ or podman/
foam touch /usr/foam-test         # expect: Read-only file system
foam touch ~/foam-test/ok.txt && ls ok.txt   # expect: works (the working dir is writable)
```

### 2. Working directory refusals

```bash
(cd ~ && foam true)               # expect: refusing to expose ... (your home or a parent of it)
(cd ~/.config && foam true)       # expect: refusing ... (hidden directory in your home)
(cd /tmp && foam true)            # expect: refusing ... (contains /tmp)
mkdir -p ~/foam-test-sock && (cd ~/foam-test-sock &&
  python3 -c "import socket; socket.socket(socket.AF_UNIX).bind('s.sock')" && foam true)
                                  # expect: refusing ... (contains socket/FIFO s.sock)
rm -rf ~/foam-test-sock
```

### 3. Processes, privileges, environment

```bash
foam ps aux                       # expect: only bwrap, ps (and a shell)
foam sudo true                    # expect: fails (e.g. "must be owned by uid 0" or
                                  #   "no new privileges" flag is set)
foam unshare -U true              # expect: fails (Operation not permitted or No space left on device);
                                  #   succeeds only with bwrap older than 0.8 and seccomp off
TEST_SECRET=leak foam env | grep TEST_SECRET   # expect: no output
exec 7</etc/os-release; foam ls /proc/self/fd; exec 7<&-
                                  # expect: 0 1 2 and maybe 3 (ls's own), but no 7
foam ls /dev                      # expect: a handful of entries, no dri/, input/, video*, kvm
```

### 4. Terminal injection (TIOCSTI)

Run this in a real terminal:

```bash
foam python3 -c 'import fcntl, termios; fcntl.ioctl(0, termios.TIOCSTI, b"x")'
```

Expect an error: `Operation not permitted` or `Input/output error`. If an `x` appears in your next prompt instead, you're **NOT protected**.

### 5. Profile isolation and protect

```bash
foam -p t-a sh -c 'echo secret > ~/token'
foam -p t-b cat ~/token           # expect: No such file or directory
foam -t ls -a ~                   # expect: empty

printf 'protect .tools\n' > ~/.config/foam/t-prot.conf
foam -p t-prot true               # expect: protected ~/.tools doesn't exist yet, create it once with -u
foam -u -p t-prot mkdir ~/.tools
foam -p t-prot touch ~/.tools/x   # expect: Read-only file system
foam -u -p t-prot touch ~/.tools/x && echo ok   # expect: ok
rm -rf ~/.local/share/foam/t-prot/home/.tools
ln -s "/run/user/$(id -u)" ~/.local/share/foam/t-prot/home/.tools
foam -p t-prot true               # expect: protected ~/.tools is a symlink, check and remove it
```

### 6. Git protection

```bash
cd ~/foam-test && git init -q && echo a > a.txt && git add a.txt && git commit -qm init
foam sh -c 'echo "touch /tmp/pwned" > .git/hooks/pre-commit'   # expect: Read-only file system
foam git config core.fsmonitor "touch /tmp/pwned"              # expect: an error (config can't be replaced)
git config --get core.fsmonitor                                # expect: no output (unchanged)
foam sh -c 'echo b >> a.txt && git -c user.name=t -c user.email=t@t commit -qam inside && echo committed'
                                                               # expect: committed
ls /tmp/pwned                                                  # expect: No such file or directory
```

### 7. Seccomp

```bash
foam -p t-seccomp python3 -c 'import socket; socket.socket(socket.AF_PACKET, socket.SOCK_RAW)'
                        # expect: Address family not supported by protocol
                        # (without seccomp you'd get "Operation not permitted" instead)
foam -p t-seccomp keyctl show @s
                        # expect: fails with Operation not permitted (needs the keyutils package)
foam -p t-seccomp bash -c 'sleep 1 & wait; echo processes-ok'
                        # expect: processes-ok (normal work unaffected)
```

If you use Kerberos, `klist` inside a `seccomp yes` profile should find no tickets, while it shows them on the host.

### 8. Network modes

```bash
foam curl -sI https://example.com | head -1            # expect: HTTP/... 200 (net yes)
foam -p t-offline curl -m5 -sI https://example.com     # expect: fails, could not resolve host (net no)
```

### 9. pasta isolation (`net pasta`)

Start a server that listens **only on the host's localhost**:

```bash
python3 -m http.server 8765 --bind 127.0.0.1 &
SERVER=$!
```

Then check:

```bash
# 9.1 Internet and DNS work
foam -p t-pasta curl -sI https://example.com | head -1
                        # expect: HTTP/... 200
foam -p t-pasta getent hosts example.com
                        # expect: an IP address

# 9.2 Host localhost is NOT reachable (compare with net yes)
foam -p t-pasta curl -m3 http://127.0.0.1:8765         # expect: fails (connection refused / timeout)
foam curl -m3 -s http://127.0.0.1:8765 | head -2       # net yes, for comparison: a directory listing

# 9.3 The gateway address doesn't lead to the host either
gw=$(foam -p t-pasta sh -c "ip route | awk '/^default/ {print \$3; exit}'")
foam -p t-pasta curl -m3 "http://$gw:8765"             # expect: fails

# 9.4 Abstract X11 socket is NOT reachable (needs an X server or Xwayland running)
d=${DISPLAY#*:}; d=${d%.*}
foam -p t-pasta python3 -c "import socket; socket.socket(socket.AF_UNIX).connect('\0/tmp/.X11-unix/X$d'); print('CONNECTED: NOT protected')"
                        # expect: ConnectionRefusedError
foam python3 -c "import socket; socket.socket(socket.AF_UNIX).connect('\0/tmp/.X11-unix/X$d'); print('CONNECTED')"
                        # net yes, for comparison: usually CONNECTED, which is why pasta matters

# 9.5 Exit codes come through pasta
foam -p t-pasta sh -c 'exit 7'; echo $?                # expect: 7

kill $SERVER
```

If pasta stops with an unknown-option error: newer passt releases are replacing `--no-map-gw` with `--map-host-loopback none`. Adjust `pasta_opts` in the script to match your version.

### 10. Audit mode

```bash
cd ~/foam-test && echo node_modules/ > .gitignore
foam -p t-audit sh -c '
  echo c >> a.txt
  mkdir -p node_modules/x && echo evil > node_modules/x/i.js
  touch -d 2001-01-01 node_modules/x/i.js
  git -c user.name=t -c user.email=t@t commit -qam "sandbox commit"
  git init -q sub'
```

Expect a report listing:
- the commit `sandbox commit` and the moved branch
- `node_modules` among the changed files, even though git ignores it and the file's date was faked to 2001
- a **WARNING** about the new nested `.git` in `sub`.

### Cleanup

```bash
rm -rf ~/foam-test ~/.config/foam/t-*.conf ~/.local/share/foam/t-*
```

---

## Limitations

foam narrows what a sandboxed tool can reach, but some risks are outside what a wrapper script can control:

- **The project directory is a writable bridge, and this is the biggest real-world risk.** Anything on the host that later runs code from the project can carry an attack out:
  - IDE extensions that run `node_modules/.bin/eslint` or `.venv/bin/python`
  - `npm`, `npx`, `make` or `./gradlew` run outside foam
  - hooks in tracked directories like `.husky` (warned in audit mode)
  - files planted in a parent of several projects.

  Run project tooling *through* foam, open sandboxed projects in your IDE's restricted or untrusted mode, and review changes (audit mode helps) before running anything on the host.
- **Terminal output goes straight to your terminal.** A sandbox can write the clipboard with OSC 52 escape sequences (in terminals that allow it) or exploit bugs in your terminal emulator.
- **Some checks happen once, at start:** sockets created later, repositories more than 3 levels deep, and races with *another* concurrently running sandbox that can write a parent of the working directory.
- **Anything readable can be sent out while network is on:** the project, the profile home, `ro` paths and `/etc`.
- **There are no resource limits.** A sandbox can fill the disk or use all memory and CPU. Wrapping foam in `systemd-run --user --scope -p MemoryMax=4G foam ...` adds limits.
- **Pre-existing poisoning isn't undone.** In repositories you used with older versions of foam, check `.git/config` (`git config --show-origin --list`) and `.git/hooks` once. An `include.path` that points into the project is just as dangerous as a tracked hooks directory.
- **foam depends on bubblewrap and the kernel.** Keep both updated.

---

## Files and directories

| Path | Contents |
|---|---|
| `~/.config/foam/<profile>.conf` | Your profile configs (never visible to sandboxes) |
| `~/.local/share/foam/<profile>/home` | Profile homes |
| `~/.local/share/foam/<profile>/last-changes.txt` | Last audit report |
| `~/.local/share/foam/bashrc` | Shell init, given read-only to sandboxes as `/run/foam.bashrc` |
| `~/.local/share/foam/resolv.conf` | DNS config for `net pasta` |

foam respects `XDG_CONFIG_HOME` and `XDG_DATA_HOME`.

Inside the sandbox, `FOAM_PROFILE` holds the profile name, so scripts and your profile's `~/.bashrc` can detect it: `[[ -n $FOAM_PROFILE ]]`.
