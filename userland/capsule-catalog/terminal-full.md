# capsule_terminal (full reference)

`capsule_terminal` is the flagship user application in the NONOS tree: a GUI window with a real shell
behind it. The shell has statement sequencing, pipelines, input and output redirection, aliases, shell
variables, tab completion, and a Unix-like command family spanning the filesystem, the network, the
service registry, the marketplace, and system introspection. This is the exhaustive reference; the
[terminal overview](../terminal.md) is the short version.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.terminal` on service port 4722 with a reply port on 4723, and its capability mask is `0x1819`
(`userland/capsule_terminal/Capsule.mk:11`). The source is `userland/capsule_terminal/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Command reference](#command-reference)
- [Keybindings and interaction](#keybindings-and-interaction)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The terminal is an ordinary NONOS GUI application. Its entry point hands its `App` implementation to the
skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the paint
loop, and the terminal supplies three things: a manifest for a normal window, an `on_event` that feeds
keystrokes to a line editor and, on Enter, to the command interpreter, and a `paint` that draws the
frame (`userland/capsule_terminal/src/main.rs:43`, `src/term/terminal/app_impl.rs:21`).

Behind the prompt is a genuine shell. A submitted line is split into statements joined by `;`, `&&`, and
`||`, each statement is a pipeline of commands, and each command is alias-expanded and variable-expanded
before it runs. A plain command executes directly and prints to the screen; a command with a pipe, an
input redirect, or an output redirect goes through a capture-and-fold path that collects its output into
an in-memory line buffer and folds each pipe stage as a filter over the accumulated lines
(`src/command/dispatch/run.rs:33`, `src/command/dispatch/pipeline.rs`). The terminal is a signed capsule
spawned as part of the desktop fleet at boot, and it can host up to nine independent shell tabs in one
window (`src/term/terminal/tabs.rs:24`).

## Identity

Everything the kernel and the service registry need to name and reach the terminal comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `terminal` | `Capsule.mk:1` |
| Service handle | `app.terminal` | `Capsule.mk:2`, `src/userspace/capsule_terminal/spawn.rs:30` |
| Namespace | `systems.nonos.app.terminal` | `Capsule.mk:7` |
| Service endpoint | `service:4722:app.terminal` | `Capsule.mk:8`, `spawn.rs:31` |
| Reply endpoint | `reply:4723:endpoint.app.terminal.reply` | `Capsule.mk:9`, `spawn.rs:32` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `terminal` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_terminal` | `Capsule.mk:12` |

The mask `0x1819` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                bit()  1     types.rs:56
  0x0008  IPC                     bit()  8     types.rs:59
  0x0010  Memory                  bit() 16     types.rs:60
  0x0800  GraphicsDisplayQuery    bit() 2048   types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096   types.rs:68
  ------
  0x1819  = 1 + 8 + 16 + 2048 + 4096
```

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_terminal/spawn.rs:49`). There is no `Network` bit (4), no `FileSystem` bit (64),
and no hardware, driver, or DMA capability in the mask, which is the whole basis of the security
analysis below: the terminal can create a surface, ask the display for its size, and speak IPC, and every
command that appears to touch a file or the network is really an IPC call to a service that holds the
real authority.

## Command reference

A line is dispatched in a fixed sequence. `on_enter` echoes the line, records it in history, and splits
it into `(connector, statement)` pairs with `split_program`; for each statement it applies alias
expansion, then variable expansion, then tokenization, then runs it (`src/event/on_enter.rs:42`,
`src/command/dispatch/statements.rs:29`). The runner `run` pulls off any `< file`, `> file`, or `>> file`
and either fast-paths a plain command straight to `exec`, or takes the capture-and-fold path for pipes
and redirects (`src/command/dispatch/run.rs:33`). `exec` resolves the command word: a handful of verbs
are dispatched directly, and everything else falls through to the `nox` family
(`src/command/dispatch/exec.rs:24`, `src/command/builtin/nox/dispatch.rs:26`).

Two words are handled before dispatch. `exit` and `quit` end the shell (close the tab, or the window if
it is the last tab) (`src/command/builtin/exit_check.rs:17`). An empty line repaints and does nothing
(`run.rs:34`).

### Top-level verbs

These are matched in `exec` before the `nox` fall-through (`src/command/dispatch/exec.rs:29`). Several
are thin wrappers that the `nox` family also exposes under the same or a familiar name.

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `nox` | `nox [verb ...]` | dispatch into the `nox` family; no verb prints the `nox` help index | `exec.rs:30` |
| `help` | `help` | print the `nox` help index | `exec.rs:31` |
| `about` | `about` | print the capsule's self-description (CPL, wire, syscalls) | `exec.rs:32`, `builtin/about.rs:19` |
| `version` | `version` | print version, namespace, ABI, trust line | `exec.rs:33`, `builtin/version.rs:19` |
| `whoami` | `whoami` | print capsule identity (handle, namespace, CPL, signer) | `exec.rs:34`, `builtin/whoami.rs:19` |
| `capsules` / `caps` | `capsules` | list expected system services and whether each is live | `exec.rs:35`, `builtin/capsules.rs:47` |
| `clear` | `clear` | clear the scrollback | `exec.rs:38`, `builtin/clear.rs` |
| `display` | `display` | query the primary display size and pixel format | `exec.rs:39`, `builtin/display.rs:22` |
| `echo` | `echo [text ...]` | print the arguments joined by spaces | `exec.rs:40`, `builtin/echo.rs:21` |
| `history` | `history` | print the command history, numbered | `exec.rs:41`, `builtin/history_cmd.rs:22` |
| `market` | `market` | list the marketplace catalog through the market service | `exec.rs:44`, `builtin/market/run.rs:27` |
| `motd` | `motd` | print the banner or message of the day | `exec.rs:45`, `builtin/motd.rs` |
| `ping` | `ping <host>` | one-shot ICMP echo through the net.ip service | `exec.rs:46`, `builtin/ping/mod.rs:40` |
| `service` / `svc` | `service <name>` | resolve one service name to its port and pid | `exec.rs:47`, `builtin/service.rs:22` |

Note that `capsules` reports a fixed expected set of system services (ramfs, vfs, keyring, entropy,
crypto, market, the virtio and input drivers, the compositor, wm, desktop_shell, input_router, the
net.* stack, login, wallpaper) and marks each `[live]` or `[absent]` from a registry lookup; it is not a
live process table (`builtin/capsules.rs:22`).

### The nox family

Any word `exec` does not recognise, and anything after `nox`, is dispatched here
(`src/command/builtin/nox/dispatch.rs:35`). Each command returns a success boolean that is written to
`state.last_status`, which is what `&&` and `||` gate on. Commands marked infallible always report
success.

Filesystem, all resolved against the shell's `cwd` (`src/term/cwd/resolve.rs:19`) and run through the
app-skeleton vfs client:

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `where` / `pwd` | `where` | print the current directory | `dispatch.rs:36`, `nox/whereis.rs:19` |
| `in` / `cd` | `in <path>` | change directory (stat-checked; must be a directory) | `dispatch.rs:40`, `nox/enter.rs:24` |
| `ls` / `dir` | `ls [path]` | list the immediate children of a directory | `dispatch.rs:41`, `nox/ls.rs:24` |
| `read` / `cat` | `read <file>` | print a file, split on newlines (up to 64 KiB) | `dispatch.rs:45`, `nox/read.rs:25` |
| `write` | `write <file> <text ...>` | write text (joined by spaces, newline appended) to a file | `dispatch.rs:46`, `nox/write.rs:24` |
| `copy` / `cp` | `copy <src> <dst>` | duplicate a file or a directory subtree in the store | `dispatch.rs:47`, `nox/copy.rs:23` |
| `mk` / `mkdir` | `mk <dir>` | create a directory | `dispatch.rs:48`, `nox/mk.rs:23` |
| `rm` / `del` | `rm [-r] <path>` | remove a file (unlink) or directory (rmdir, `-r` recursive) | `dispatch.rs:49`, `nox/rm.rs:23` |
| `mv` / `move` | `mv <old> <new>` | move or rename a path | `dispatch.rs:50`, `nox/mv.rs:23` |
| `stat` | `stat <path>` | print type, size in bytes, and path | `dispatch.rs:51`, `nox/stat.rs:25` |
| `find` | `find [path]` | list the paths under a directory prefix | `dispatch.rs:52`, `nox/find.rs:23` |
| `du` | `du [path]` | sum the sizes of the files directly under a prefix | `dispatch.rs:53`, `nox/du.rs:25` |
| `touch` | `touch <file>` | create an empty file if it does not already exist | `dispatch.rs:54`, `nox/touch.rs:23` |
| `basename` | `basename <path>` | print the final path component | `dispatch.rs:55`, `nox/pathname.rs:19` |
| `dirname` | `dirname <path>` | print the parent path component | `dispatch.rs:56`, `nox/pathname.rs:33` |

`find` is a single-level prefix listing, not a recursive walk: it lists the paths the vfs returns for the
directory prefix (`nox/find.rs:31`). `du` sums only the files the same listing returns, not a recursive
tree (`nox/du.rs:41`).

Network, each a real request to a network service:

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `ping` | `ping <host>` | resolve the host, build an ICMP echo, send it through net.ip, poll for the reply | `dispatch.rs:106`, `builtin/ping/mod.rs:40` |
| `ifconfig` / `ip` | `ifconfig` | print the interface lease, or `net0: down` if none | `dispatch.rs:58`, `nox/ifconfig/run.rs:23` |
| `nslookup` / `host` | `nslookup <host>` | resolve a name to an A record through net.dns | `dispatch.rs:59`, `nox/nslookup.rs:29` |

Services, registry, and marketplace:

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `caps` / `ps` | `caps` | list expected services live/absent (same as `capsules`) | `dispatch.rs:64`, `nox/caps.rs:21` |
| `svc` | `svc <name>` | resolve one service to its port and pid | `dispatch.rs:68`, `nox/svc.rs:21` |
| `apps` / `market` | `apps` | list the marketplace catalog | `dispatch.rs:84`, `nox/apps.rs:21` |
| `run` / `open` | `run <app>` | focus an already-running desktop app by sending it an NCTL focus frame | `dispatch.rs:88`, `nox/run.rs:29` |
| `install` | `install <name> [argv ...]` | ask the installer to verify, load, and spawn a store capsule | `dispatch.rs:89`, `nox/install/run.rs:24` |

System and introspection:

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `id` | `id` | print capsule identity (delegates to `whoami`) | `dispatch.rs:72`, `nox/id.rs:21` |
| `sys` | `sys` | print version then about | `dispatch.rs:76`, `nox/sysinfo.rs:21` |
| `date` | `date` | print the RTC date and time | `dispatch.rs:57`, `nox/date.rs:22` |
| `env` | `env` | list shell variables (same as `set` with no args) | `dispatch.rs:60`, `nox/set.rs:24` |
| `motd` | `motd` | print the banner | `dispatch.rs:110`, `nox/motd.rs` |
| `display` | `display` | query the primary display | `dispatch.rs:118`, `nox/display.rs` |
| `clear` | `clear` | clear the scrollback | `dispatch.rs:122`, `nox/clear.rs` |
| `echo` | `echo [text ...]` | print the arguments | `dispatch.rs:80`, `nox/echo.rs` |
| `history` | `history` | print the numbered history | `dispatch.rs:114`, `nox/history.rs` |
| `help` | `help` | print the `nox` help index | `dispatch.rs:126`, `nox/help.rs:19` |

Shell state:

| Command | Syntax | What it does | Handler |
|---|---|---|---|
| `set` | `set [name value ...]` | list variables, or define one (used later as `$name`) | `dispatch.rs:90`, `nox/set.rs:24` |
| `unset` | `unset <name>` | remove a variable | `dispatch.rs:94`, `nox/unset.rs:20` |
| `alias` | `alias [name expansion ...]` | list aliases, or define one | `dispatch.rs:98`, `nox/alias.rs:23` |
| `unalias` | `unalias <name>` | remove an alias | `dispatch.rs:102`, `nox/unalias.rs:20` |

An unrecognised word runs the `unknown` handler, which prints `nox: unknown verb '<word>' (try: nox
help)` and reports failure, so `nonexistent || echo no` prints the fallback
(`src/command/builtin/nox/dispatch.rs:130`, `nox/unknown.rs:21`).

### Sequencing, pipelines, and redirection

- `a ; b` runs `b` unconditionally; `a && b` runs `b` only if `a` succeeded; `a || b` runs `b` only if
  `a` failed. Quoting is respected, so a separator inside quotes is literal
  (`src/command/dispatch/statements.rs:29`, gated in `src/event/on_enter.rs:42`).
- `a | b | c` runs `a` with its output captured, then folds `b` and `c` as filters over the accumulated
  lines (`src/command/dispatch/pipeline.rs:27`). The filter stages are real and finite:
  `grep [-i] [-v] <pat>`, `sort`, `uniq`, `cut`, `nl`, `wc`, `head [n]`, `tail [n]`
  (`src/command/dispatch/filter/mod.rs:24`; the `help` index lists the same set at `nox/help.rs:52`). An
  unknown filter word yields a `pipe: unknown filter <name>` line.
- `cmd > file` writes the captured output to a vfs file; `cmd >> file` reads the existing file and
  appends (`src/command/dispatch/redirect.rs:54`, `write_redirect.rs:30`).
- `< file` seeds the pipeline from a file instead of a leading command, so `< data | grep x` filters the
  file (`src/command/dispatch/run.rs:60`, `redirect.rs:31`).

### Aliases and variables

`alias name=...` is not the syntax; aliases and variables take space-separated arguments (`alias ll ls
-l`, `set greeting hello there`). An alias replaces the first word of a future line before it runs, a
single level deep (`src/command/dispatch/alias_expand.rs:22`). A variable is substituted where `$name`
appears, except inside single quotes; an undefined variable expands to nothing
(`src/command/dispatch/expand.rs:23`). Both live entirely in the terminal's own `State`
(`src/term/state/types.rs:34`), so there is no external shell process and no persistence across a
restart.

## Keybindings and interaction

Input arrives as key-down events. `on_event` ignores anything that is not a key-down, then the tab layer
gets first refusal on Ctrl chords, then `on_key` handles editing and the Ctrl chords the tab layer did
not claim (`src/event/on_event.rs:22`, `src/term/terminal/app_impl_on_event.rs:29`, `src/event/on_key.rs:32`).

Line editing and navigation (`src/event/on_key.rs`):

| Key | Action | Source |
|---|---|---|
| Printable 0x20..0x7E | insert at the cursor | `on_key.rs:64`, `on_printable.rs:21` |
| Backspace | delete the char before the cursor | `on_key.rs:41` |
| Delete | delete the char at the cursor | `on_key.rs:42` |
| Left / Right | move the cursor one column | `on_key.rs:43` |
| Home / End | move to start / end of line | `on_key.rs:45` |
| Up / Down | history recall filtered by the typed prefix | `on_key.rs:53`, `on_up.rs:21`, `on_down.rs:21` |
| Enter | run the line | `on_key.rs:40`, `on_enter.rs:26` |
| Tab | complete the command word or a vfs path | `on_key.rs:63`, `on_tab.rs:30` |
| Esc | close the window | `on_key.rs:39` |
| Page Up / Page Down | scroll the scrollback by a screen | `on_key.rs:55` |

History recall is prefix-aware: pressing Up captures whatever is typed so far as the prefix and walks
back through matching entries; Down walks forward and restores the original text once it runs past the
newest match (`src/event/on_up.rs`, `on_down.rs`). Tab completes the first token against the shell's
command table and any later token against vfs paths, extending the word by the candidates' common prefix
or listing them when it cannot extend (`src/event/on_tab.rs:30`, `complete.rs:21`).

Ctrl chords, handled in `on_ctrl` (upper and lower case codes both match) (`src/event/on_ctrl.rs:40`):

| Chord | Action | Source |
|---|---|---|
| Ctrl+A | move to start of line | `on_ctrl.rs:69` |
| Ctrl+E | move to end of line | `on_ctrl.rs:73` |
| Ctrl+U | clear the line | `on_ctrl.rs:57` |
| Ctrl+W | delete the word before the cursor | `on_ctrl.rs:61` |
| Ctrl+K | kill from the cursor to end of line | `on_ctrl.rs:65` |
| Ctrl+L | clear the scrollback and jump to the bottom | `on_ctrl.rs:45` |
| Ctrl+C | clear the line, reset history, print `^C` | `on_ctrl.rs:50` |
| Ctrl+Shift+C | copy the current line to the clipboard | `on_ctrl.rs:47`, `copy_line.rs:21` |
| Ctrl+V | paste printable clipboard bytes into the line | `on_ctrl.rs:43`, `paste_clipboard.rs:22` |

Note that Ctrl+C alone is a line/interrupt reset that prints `^C`; it does not signal or kill a child,
because the shell runs commands inline rather than as separate processes. Copy is Ctrl+Shift+C
specifically, so it does not collide with the interrupt.

Tabs, handled in `tab_command` before the per-tab handlers, and only when Ctrl is held
(`src/term/terminal/tabs.rs:29`):

| Chord | Action | Source |
|---|---|---|
| Ctrl+Shift+T | open a new tab (up to 9) | `tabs.rs:35` |
| Ctrl+Shift+W | close the active tab (or close the window if it is the last) | `tabs.rs:36` |
| Ctrl+Page Down / Ctrl+Page Up | switch to the next / previous tab | `tabs.rs:37` |
| Ctrl+1 .. Ctrl+9 | jump to tab 1..9 | `tabs.rs:39` |

A pointer ButtonDown on the tab strip selects or closes a tab through `tab_click`
(`src/term/terminal/app_impl_on_event.rs:24`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(Terminal::new)` in the normal
build; a self-test entry is compiled instead under the `nonos-autorun-selftest` feature
(`src/main.rs:36`). The four top-level modules are `command` (the shell), `event` (the input handlers),
`paint` (the renderer), and `term` (the terminal model) (`src/main.rs:22`).

The model is a `Terminal` holding a vector of shell tabs and the active index; each tab is one `State`
(`src/term/terminal/types.rs:21`). A `State` holds the line editor, the command history, the scrollback
buffer, the current directory, the cached owner pid, the shell variables and aliases, the last exit
status, and the rendered command blocks (`src/term/state/types.rs:25`). The `term/` tree also carries a
`grid` and a `vt` layer, a `prompt`, a `theme`, a `banner`/`motd`, and `dur`/`rtc` for timing.

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan
   (`src/userspace/init/spawn_plan/apps.rs:87`), which verifies the embedded ELF, cert, manifest, and
   attestation, registers `app.terminal` on port 4722, and logs `[APP-TERMINAL] capsule spawned`
   (`src/userspace/capsule_terminal/spawn.rs:36`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` creates the window from the manifest (a 520x300 Normal window titled `Terminal`,
   subscribing to key-down input) and drives the event and paint loop (`src/term/manifest.rs:24`).
3. Each key-down flows in through `on_event` to the active tab; Enter runs the line and the outcome tells
   the runtime whether to repaint or close (`src/event/on_enter.rs:66`).
4. `paint` projects the active tab's `State` into the surface: header, tab strip, the scrollback grid
   drawn row by row, the input line with a cursor, and a footer (`src/paint/compose.rs:18`,
   `src/paint/mod.rs:32`). The frame lands in the shared surface the compositor presents.

## Protocol and IPC

The terminal exposes no application opcodes of its own beyond what the app skeleton registers for it
(the `app.terminal` service on port 4722 and the reply inbox on 4723,
`src/userspace/capsule_terminal/spawn.rs`); the `run`/focus path also accepts the desktop's NCTL
focus-self control frame the same way the dock launcher sends it. Everything the shell does that reaches
outside the capsule is an outbound IPC call to another service. The calls it makes:

VFS, service `vfs_pool`, magic `0x4E4F5646`, resolved through the skeleton's vfs client
(`userland/app_skeleton/src/clients/vfs/types.rs`):

```
  OP_OPEN     1     open a path                types.rs:19
  OP_CLOSE    2     close a handle             types.rs:20
  OP_READ     3     read (read/cat, redirect)  types.rs:21
  OP_WRITE    4     write (write, touch, > >>) types.rs:22
  OP_STAT     5     stat (stat, cd, cp, rm)    types.rs:23
  OP_LIST     6     list (ls, find, du, tab)   types.rs:24
  OP_MKDIR    8     mkdir (mk)                 types.rs:25
  OP_UNLINK   9     unlink (rm on a file)      types.rs:26
  OP_RENAME   10    rename (mv)                types.rs:27
  OP_COPY     11    copy (cp)                  types.rs:28
  OP_RMDIR    12    rmdir (rm on a dir)        types.rs:29
```

The vfs reply carries a status word and the requested payload; the terminal surfaces the client's error
string verbatim on failure (for example `nox/read.rs:40`).

Clipboard, service `clipboard`, magic NCLP `0x43424930`
(`userland/app_skeleton/src/clients/clipboard/`): `OP_COPY 0x0002` for Ctrl+Shift+C
(`clipboard/copy.rs:22`) and `OP_PASTE 0x0003` for Ctrl+V (`clipboard/paste.rs:22`); the reply is a
status word for copy and a content-typed buffer for paste.

Network:

- DNS, service `net.dns`, magic `0x4E444E53`, `OP_RESOLVE_A 2`, 2 s timeout; the reply header carries a
  status and an A record (`src/command/builtin/nox/nslookup.rs:23`).
- IP, service `net.ip`, magic `0x4E495034`, `OP_SEND_PACKET 4` and `OP_POLL_PACKET 5` for the ICMP echo
  and its poll, 1 s deadline (`src/command/builtin/ping/mod.rs:30`).
- Interface state through the ifconfig client (`src/command/builtin/nox/ifconfig/`).

Service and registry lookups go through `mk_service_lookup` / `lookup_service`, used by `svc`, `service`,
`capsules`, `run`, and by every command that first caches the terminal's own pid
(`src/command/builtin/service.rs:30`, `src/command/dispatch/run.rs:83`).

Installer, service `installer`, wire `seq(4) | op(2) | pad(2) | body`, `OP_LOAD_BY_NAME 4`; the reply is
`seq(4) | status(4) | pid(4)` (`src/command/builtin/nox/install/call.rs:30`). After a successful load the
terminal drains the child's stdout from its `proc.<pid>` inbox into the window, bounded by a 5 s deadline
so a child that never exits cannot freeze the shell (`src/command/builtin/nox/install/run.rs:54`).

## Security analysis

The terminal looks like the most powerful capsule in the tree because it drives the filesystem, the
network, the registry, and the installer from one prompt, but its authority is exactly the app envelope
and nothing more. Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate (`Capsule.mk:11`, `src/userspace/capsule_terminal/spawn.rs:49`). There is no
Network bit, no FileSystem bit, and no hardware, driver, MMIO, or DMA capability. The terminal cannot
read a block device, open a socket, or touch a device register on its own.

Every command that appears to do those things is an IPC call to a service that holds the real authority
and applies its own checks. `ls`, `read`, `write`, `cp`, `mv`, `rm`, `mkdir`, `stat`, `du`, `find`,
`touch`, and both redirection forms are requests to the `vfs_pool` service through the skeleton's vfs
client; `ping` and `nslookup` are requests to the network stack; `svc`, `service`, `caps`, `capsules`,
and `apps`/`market` query the registry and the market service. The terminal marshals the argument bytes
and renders the reply, and the service on the far side decides whether the operation is allowed. A bug in
command parsing, redirection, or a pipeline cannot escalate past what the vfs and the network already
permit for this pid, because the terminal never held more than the right to ask. Redirection is real
file I/O, so `> file` is subject to the same vfs permission a direct `write` is, and the pipeline is a
synchronous capture-and-fold inside the terminal's own memory (`src/command/dispatch/pipeline.rs`), with
no second process and no shared buffer.

Two commands deserve care because they cross into launching code:

- `run` / `open` does not spawn anything. Every desktop app is already running, so `run` resolves the
  target's service and sends it a single NCTL focus-self control frame to bring its window forward
  (`src/command/builtin/nox/run.rs:29`). It cannot create a process.
- `install` is the only path that brings a new capsule to life, and it does so at arm's length. The shell
  sends the installer nothing but a validated name and an argv blob; the installer reads the multi-MB
  artifacts from the store itself and runs the same verify-load-spawn path as any boot capsule
  (`src/command/builtin/nox/install/call.rs:37`). The name is constrained to ascii letters, digits, `_`,
  and `-`, at most 64 bytes, with no path separators, so the shell cannot point the installer outside the
  store (`src/command/builtin/nox/install/run.rs:97`). The install wire does carry a `REQUESTED_CAPS =
  u64::MAX` field, but that is a request, not a grant: the comment and the design are explicit that the
  verified manifest and the identity certificate ceiling are the real bounds, and this field only avoids
  restricting a capsule below what it legitimately declares (`install/call.rs:22`). So the terminal
  cannot run unsigned code, cannot exceed a capsule's own declared ceiling, and cannot execute a file off
  the filesystem the way a POSIX shell can; there is no `exec` of a store binary.

Tabs share nothing dangerous: each tab is an independent `State` with its own line, history, cwd,
variables, and aliases (`src/term/terminal/types.rs`), and closing the last tab closes the window
(`src/term/terminal/tabs.rs:52`). Isolation from other capsules is the kernel's, not the terminal's: the
terminal is a CPL 3 user binary that only speaks IPC and its own surface, and it is verified and enrolled
at spawn like every other capsule.

## How to contribute

The source lives at `userland/capsule_terminal/`. The shell is under `src/command/`, the input handlers
under `src/event/`, the renderer under `src/paint/`, and the terminal model under `src/term/`.

To add a new command:

1. Write the command module. Most commands live under `src/command/builtin/nox/` as one file per verb
   (for example `nox/stat.rs`), exposing a `pub fn run(state: &mut State, args: &[&[u8]]) -> bool` that
   returns success. A fallible command returns `false` on error after pushing an error line; an
   infallible command can return through the `true` arm in the dispatch. Filesystem commands resolve
   their path with `crate::term::cwd::resolve` and cache the owner pid with `ensure_pid`
   (`src/command/builtin/nox/ensure_pid.rs`).
2. Re-export it from the family's `use` list and wire it into the match in
   `src/command/builtin/nox/dispatch.rs:17` and `:35`. Add any familiar alias as an extra match arm, the
   way `ls | dir` and `read | cat` are wired.
3. If it should be a top-level verb rather than a `nox` sub-command, add the arm to
   `src/command/dispatch/exec.rs:29` instead (and keep the module under `src/command/builtin/`).
4. Add the command name and any aliases to the completion table in `src/event/complete.rs:21` so Tab
   completes it, and add a line to the help index in `src/command/builtin/nox/help.rs:19`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_terminal/Capsule.mk:14`):

```
  make nonos-mk-terminal          build the capsule ELF
  make nonos-mk-terminal-sign     produce the id cert, manifest, and attestation trailer
  make nonos-mk-terminal-verify   verify the signed artifacts against the trust anchor
  make nonos-mk-check-terminal-keys   check the per-capsule signing keys exist
```

For a running desktop that includes the terminal, `make nonos-mk-terminal-prod` builds the full desktop
GUI image, and `make nonos-mk-terminal-only-prod` builds the terminal-only kernel profile
(`Makefile:1165`, `Makefile:1168`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every command returns errors as pushed lines and a `false` status, never a
panic; the release profile is `panic = "abort"`, `Cargo.toml`); modular files, one unit per file, with
`mod.rs` used only for re-exports; and the AGPL header at the top of every source file, matching the
header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[APP-TERMINAL]
capsule spawned` (tag `APP-TERMINAL`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`).

Failure modes and where to look:

- Terminal opens but no input reaches it. The window subscribes only to key-down and ignores everything
  else (`src/event/on_event.rs:23`). If keys do nothing, the input path into the app (compositor, wm,
  input_router) is the suspect, not the shell; `caps` shows whether those services are live
  (`src/command/builtin/nox/caps.rs`).
- Command not found. An unrecognised word prints `nox: unknown verb '<word>' (try: nox help)` and reports
  failure (`src/command/builtin/nox/unknown.rs:21`). Because every command sets `state.last_status`, the
  fastest probe is `<word> || echo no`: it distinguishes an unknown verb from a known command that failed
  against a live service (`src/command/dispatch/statements.rs`).
- A filesystem command fails while a nearby one works. The split is between the terminal and the vfs: the
  terminal only builds the request and prints the vfs client's error string, so a failing `read` next to a
  working `ls` in the same directory points at that specific vfs op, not the shell. Paths resolve against
  the shell's `cwd`, not the caller's, so check `where` first before assuming a vfs bug
  (`src/term/cwd/resolve.rs`).
- A network command fails. `ping` and `nslookup` are the probes. `ping` emits a specific line for each
  stage (dns unavailable, unknown host, no route, unreachable, timed out, send failed), so the message
  isolates the failure (`src/command/builtin/ping/mod.rs:45`). `nslookup` prints `dns unavailable` or
  `not resolved` (`src/command/builtin/nox/nslookup.rs`).
- Install denied or silent. `install` emits `[TERMINAL-INSTALL] load ok` or `[TERMINAL-INSTALL] load
  failed` as debug markers, and `[TERMINAL-MOUT] output drained` when the child produced output
  (`src/command/builtin/nox/install/run.rs:38`). A load failure returns the installer's negative status;
  a name with a path separator or a bad character is rejected in the shell before any IPC
  (`src/command/builtin/nox/install/run.rs:97`).

The `nonos-autorun-selftest` build emits graded `[TERMINAL-TEST]` serial markers that exercise the
unproven shell paths on a boot harness (`src/term/terminal/selftest.rs`); it is off by default and gated
behind the feature (`Cargo.toml`).

## Source map

```
  src/main.rs                              _start -> run(Terminal::new)
  src/term/terminal/                       Terminal: tabs, active index, App impl
  src/term/state/types.rs                  State: line, history, scrollback, cwd, vars, aliases, last_status
  src/term/{grid,vt,scrollback,cwd,prompt,history,theme,line}/   the terminal stack
  src/command/dispatch/statements.rs       split_program: ; && || sequencing
  src/command/dispatch/run.rs              the top-level runner (input/redirect/pipe routing)
  src/command/dispatch/exec.rs             the command-word dispatch
  src/command/dispatch/pipeline.rs         run_pipeline / run_filters (capture-and-fold)
  src/command/dispatch/filter/             the pipe filters (grep sort uniq cut nl wc head tail)
  src/command/dispatch/redirect.rs         split / split_input, write_redirect (> >> < via vfs)
  src/command/dispatch/{alias_expand,expand}.rs   alias and variable expansion
  src/command/builtin/                     the top-level verbs (about, capsules, ping, market, service)
  src/command/builtin/nox/                 the Unix-like family (ls cat cp mv rm find svc run install ...)
  src/command/builtin/nox/{ifconfig,ping,install}/   the network and installer commands
  src/event/                               the input handlers (keys, ctrl chords, tab, history, clipboard)
  src/paint/                               the frame rendering (header, tabstrip, grid, input, cursor, footer)
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_terminal/          the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs    the desktop-fleet spawn entry
  userland/app_skeleton/src/clients/       the vfs, clipboard, and discovery clients the shell calls
  nonos-mk/capsule.mk                      the generated nonos-mk-terminal[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
