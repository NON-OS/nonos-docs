# capsule_terminal (full reference)

`capsule_terminal` is the second-largest capsule in the tree, 232 source files and roughly 9,100 lines:
a GUI terminal with a real shell behind it. The shell has statement sequencing, pipelines, input and
output redirection, aliases, and variables, and a Unix-like command family spanning the filesystem, the
network, the service registry, and system introspection. This is the exhaustive reference; the
[terminal overview](../terminal.md) is the short version. App `app.terminal` on port 4722, capability
mask `0x1819`, an [app-skeleton](../writing-an-app.md) GUI app. The source is
`userland/capsule_terminal/`.

## Contents

- [The app shell](#the-app-shell)
- [The terminal model](#the-terminal-model)
- [The shell pipeline](#the-shell-pipeline)
- [Statement sequencing](#statement-sequencing)
- [Pipelines and redirection](#pipelines-and-redirection)
- [Command dispatch](#command-dispatch)
- [The command families](#the-command-families)
- [Filesystem, network, and service commands](#filesystem-network-and-service-commands)
- [Aliases and variables](#aliases-and-variables)
- [Rendering](#rendering)
- [Security analysis](#security-analysis)
- [Debugging](#debugging)
- [Honest scope](#honest-scope)
- [Source map](#source-map)

## The app shell

The entry (`src/main.rs`) hands `term::Terminal::new` to the skeleton's `run`, so the terminal is an
app-skeleton GUI app: the runtime owns the surface, window, input subscription, and paint loop, and the
terminal supplies the `App` implementation (a manifest for a normal window, an `on_event` that feeds
keystrokes to the line editor and, on Enter, the command interpreter, and a `paint` that draws the
frame). A self-test entry (`term::terminal::selftest`) is compiled under a feature flag.

## The terminal model

The whole terminal is one `State` (`src/term/state/types.rs:25`):

```
  history: History          the command history (up/down recall)
  scrollback: Scrollback    the output buffer, with capture support for pipelines
  cwd: Cwd                  the current working directory
  owner_pid: u32            this terminal's pid, resolved once for filesystem calls
  vars: Vec<(name, value)>  shell variables (set / unset / env)
  aliases: Vec<(name, expansion)>   command aliases
  last_status: bool         the last command's success, driving && and ||
```

The `term/` tree is a small terminal stack: a `grid` and a `vt` layer (a VT terminal emulator), a
`scrollback` with a capture mode (used to collect a command's output for a pipeline), `cwd` resolution,
a `prompt`, `history`, `theme`, a `banner`/`motd`, and `dur`/`rtc` for time display. The scrollback's
capture is the mechanism that makes pipelines work: a command's output can be redirected into an
in-memory line buffer instead of the screen.

## The shell pipeline

A submitted line becomes commands through a fixed sequence. At the top, `split_program`
(`src/command/dispatch/statements.rs:29`) splits the line into statements, then each statement runs
through `run` (`src/command/dispatch/run.rs:33`):

```
  run(state, argv):
      if argc == 0:                 Repaint
      if exit_check(args):          Exit
      (args_in, in_path) = split_input(args)    // pull out `< file`
      (cmd, redir)       = split(args_in)        // pull out `> file` / `>> file`
      piped = cmd contains "|"
      if no input, no pipe, no redirect:  return exec(state, cmd)   // the fast path
      lines = if in_path:  run_filters(read(in_path), cmd)
              elif piped:  run_pipeline(state, cmd)
              else:        capture(exec(state, cmd))
      if redir:  write_redirect(state, lines, append, path)
      else:      push each line to the scrollback
```

So a plain command executes directly and prints to the screen; a command with a pipe, an input
redirect, or an output redirect goes through the capture-and-fold path.

## Statement sequencing

`split_program` (`statements.rs:29`) splits a line into `(connector, statement)` pairs, respecting
single and double quotes so a separator inside a quoted string is literal:

```
  ;    Conn::Always   run the next statement unconditionally
  &&   Conn::And      run the next only if the previous succeeded
  ||   Conn::Or       run the next only if the previous failed
```

The connector gates on `state.last_status`, which every command sets, so `mk /a && ls /a` creates the
directory and lists it only if the create succeeded, and `read x || echo missing` prints the fallback
only if the read failed. Leading and trailing spaces are trimmed per statement, and an empty statement is
dropped.

## Pipelines and redirection

`run_pipeline` (`src/command/dispatch/pipeline.rs:27`) implements `a | b | c` by capture-and-fold: it
splits the args on `|`, executes the first segment with its output captured into a line buffer, and folds
each subsequent segment as a filter over the accumulated lines:

```
  run_pipeline(state, args):
      segments = split args on "|"
      begin_capture(); exec(segments[0]); lines = end_capture()
      for seg in segments[1..]:  lines = filter::apply(seg, lines)
      return lines
```

`run_filters` is the same fold seeded from a `< file` instead of a leading command, so `< data | grep x`
filters the file. Output redirection is `split` (`redirect.rs`): a trailing `> path` or `>> path` is
pulled off, the command's output is captured into lines, and `write_redirect` writes (or appends) those
lines to the file through the [vfs](vfs.md). Input redirection `< path` is pulled off by `split_input`
and read through the vfs. The redirection is real file I/O against the filesystem capsule, not a shell
fiction.

## Command dispatch

`exec` (`src/command/dispatch/exec.rs:24`) resolves the command word. A first set of verbs are the
terminal's own top-level builtins; everything else falls through to the `nox` command family:

```
  exec(state, args):
      match args[0]:
          "nox" | "help"                     -> nox::dispatch
          "about" / "version" / "whoami"     -> the info builtins
          "capsules" | "caps"                -> the capsule list
          "clear" / "display" / "echo"       -> screen and echo
          "history" / "market" / "motd"      -> history, market, message of the day
          "ping" / "service" | "svc"         -> network and service
          _                                  -> nox::dispatch   // fall through
```

## The command families

There are two command families. The top-level builtins (`src/command/builtin/`) are `about`, `capsules`,
`clear`, `display`, `echo`, `history`, `market`, `motd`, `ping`, `service`, `version`, and `whoami`. The
larger family is `nox` (`src/command/builtin/nox/dispatch.rs`), a Unix-like shell command set where each
command returns a success that feeds `state.last_status`:

```
  filesystem   where|pwd  in|cd  ls|dir  read|cat  write  copy|cp  mk|mkdir  rm|del
               mv|move  stat  find  du  touch  basename  dirname
  network      ping  ifconfig|ip  nslookup|host
  services     caps|ps  svc  apps|market  run|open  install
  system       id  sys  date  env  motd  display  clear
  shell        set  unset  alias  unalias  history  echo  whereis
```

Each has both a canonical name and common aliases (`ls`/`dir`, `read`/`cat`, `cp`/`copy`, `cd`/`in`,
`ps`/`caps`), so a user from a Unix background and a NØNOS-native user reach the same command. An
unrecognized word runs the `unknown` handler, which fails, so `nonexistent || echo no` prints the
fallback.

## Filesystem, network, and service commands

The filesystem commands are real operations against the [vfs](vfs.md) through the app skeleton's vfs
client: `ls` lists a directory, `read`/`cat` reads a file, `write` writes one, `cp`/`mv`/`rm`/`mkdir`
manipulate the tree, `stat` and `du` report metadata and usage, and `find` walks. Paths are resolved
against the terminal's `cwd` (`src/term/cwd/resolve.rs`), so relative paths work. The network commands are
also real: `ping` (`src/command/builtin/ping/`) resolves a host, builds an ICMP echo, sends it through the
network stack, and polls for the reply; `nslookup` resolves a name through DNS; `ifconfig`/`ip`
(`src/command/builtin/nox/ifconfig/`) queries the interface state. The service commands, `caps`/`ps`,
`svc`, `apps`, reach the service registry and the market to enumerate running capsules and available
apps, and `run`/`open` and `install` launch and install them.

## Aliases and variables

The shell has aliases and variables (`src/command/builtin/nox/{alias,unalias,set,unset}.rs`), stored in
`state.aliases` and `state.vars`. `alias name=expansion` records an expansion that `alias_expand`
(`src/command/dispatch/alias_expand.rs`) applies to a command before it runs, and `set name=value`
records a variable that `expand` (`src/command/dispatch/expand.rs`) substitutes in arguments. `env`
prints the variables and `unalias`/`unset` remove entries. So the shell supports the basic
customization a real shell does, all in the terminal's own state, no external shell process.

## Rendering

The `paint/` tree renders the current terminal frame into the app's [surface](../../subsystems/graphics/surfaces.md):
a header, a tab strip, the scrollback grid drawn row by row (`paint/draw_grid.rs`, `fetch_row.rs`), the
input line with a cursor (`draw_input_line.rs`, `draw_cursor.rs`), a footer, and a palette-driven theme
(`term/theme.rs`, `paint/fetch_palette.rs`). The render is a projection of the terminal `State`, and it
lands in the shared surface the compositor presents like any other window.

## Security analysis

The terminal looks like the most powerful capsule in the tree because it drives the filesystem, the
network, and the service registry from one prompt, but its authority is exactly the app envelope and
nothing more. Its capability mask is `0x1819`, which decodes to CoreExec, IPC, Memory,
GraphicsDisplayQuery, and GraphicsSurfaceCreate (bits from `src/capabilities/types.rs`). There is no
FileSystem bit, no Network bit, no hardware capability of any kind in that mask. The terminal cannot read
a block device, open a socket, or touch a device register on its own. Every command that appears to do
those things is an IPC call to a service that holds the real authority and applies its own checks. `ls`,
`read`, `write`, `cp`, `mv`, and `rm` are requests to the [vfs](vfs.md) capsule through the app
skeleton's vfs client; `ping` and `nslookup` are requests to the network stack; `caps`, `svc`, and
`apps` query the service registry and the market. The terminal marshals the argument bytes and renders
the reply, and the service on the far side decides whether the operation is allowed. This is the whole
reason a shell this capable is safe to ship: a bug in command parsing, redirection, or a pipeline cannot
escalate past what the vfs and the network already permit for this pid, because the terminal never held
more than the right to ask.

Two consequences fall out of that and both are stated in the honest scope below. There is no `exec` of a
filesystem binary and no arbitrary process spawn: `run` and `open` launch a signed, installed capsule by
name through the registry, so the shell cannot run code that was not verified and enrolled. And the
redirection is real file I/O against the vfs rather than a shell fiction, which means `> file` is subject
to the same vfs permission the `write` command is; the shell cannot write anywhere the vfs would refuse a
direct write. The pipeline is a synchronous capture-and-fold inside the terminal's own memory
(`command/dispatch/pipeline.rs`), so no second process and no shared buffer is involved, and a filter
stage sees only the lines the previous stage produced.

## Debugging

The terminal is spawned like any app, through `super::boot::capsule("APP-TERMINAL", ...)`
(`src/userspace/init/spawn_plan/apps.rs:87`), which prints `[APP-TERMINAL] capsule spawned` on success
and a spawn error otherwise. An absent line means the capsule never ran (signature or capability
failure), not that the shell is broken.

Once it is running, the useful property for debugging is that the shell surfaces failure through the same
channel a user sees: `state.last_status`. Every command sets it, and it drives `&&` and `||`
(`command/dispatch/statements.rs`), so `some_command || echo failed` turns a silent failure into a
printed line without any instrumentation. A command that reaches another capsule and comes back failed is
telling you the service call failed, not that the shell mis-parsed it; the fastest way to tell the two
apart is that an unrecognised word runs the `unknown` handler which also fails, so `nonexistent || echo
no` prints the fallback while a real command that fails against a live service prints that service's
error path. When a filesystem command fails, the split is between the terminal and the [vfs](vfs.md): the
terminal only builds the request and prints the reply, so a failing `read` with a working `ls` in the
same directory points at the specific vfs operation, not at the shell. When a network command fails,
`ping` and `nslookup` are the probes: `ping` (`src/command/builtin/ping/`) resolves, builds an ICMP echo,
sends it, and polls for the reply, so a `ping` that resolves but never replies isolates the failure to
the network path below name resolution.

Paths are the other common source of surprise, and they are resolved against the terminal's `cwd`
(`src/term/cwd/resolve.rs`), not the caller's, so a relative path that behaves unexpectedly is almost
always a `cwd` question: `where`/`pwd` prints the current directory the shell will resolve against, which
is the first thing to check before assuming a vfs bug.

## Honest scope

The terminal is a genuine shell, not a fixed menu: statement sequencing, pipelines, redirection, aliases,
and variables all work, and the command set spans the filesystem, network, services, and system. Its
limits are the honest ones of a small shell: there is no job control or background execution (`&`), no
subshells or command substitution, the pipeline is a synchronous capture-and-fold rather than concurrent
processes, and the command set is the built-in family rather than arbitrary executables (a `run`/`open`
launches a capsule, but there is no `exec` of a filesystem binary). The commands that reach other
capsules are subject to the terminal's capabilities, so a filesystem or network command works only
because the terminal was granted the capability to reach the [vfs](vfs.md) or the
[network](../networking-guide.md).

## Source map

```
  src/main.rs                              run(Terminal::new)
  src/term/state/types.rs                  State: scrollback, cwd, history, vars, aliases, last_status
  src/term/{grid, vt, scrollback, cwd, prompt, history, theme}/   the terminal stack
  src/command/dispatch/statements.rs       split_program: ; && || sequencing
  src/command/dispatch/run.rs              the top-level runner (input/redirect/pipe routing)
  src/command/dispatch/pipeline.rs         run_pipeline / run_filters (capture-and-fold)
  src/command/dispatch/redirect.rs         split / split_input, write_redirect (> >> < via vfs)
  src/command/dispatch/{alias_expand, expand}.rs   alias and variable expansion
  src/command/dispatch/exec.rs             the command-word dispatch
  src/command/builtin/                     the top-level builtins (about, capsules, ping, market, ...)
  src/command/builtin/nox/                 the Unix-like command family (ls, cat, cp, find, svc, ...)
  src/command/builtin/nox/ifconfig/, ping/ the real network commands
  src/paint/                               the frame rendering (grid, input line, cursor, tabs)
```
