# Applications and Proof Capsules

These are the end-user applications and the proof capsules that demonstrate the platform. The apps are
[app-skeleton](../writing-an-app.md) GUI programs; the proofs are self-tests that assert a specific
guarantee at runtime and are honest about being demonstrations. This page gives one verified spec per
capsule. The [terminal](../terminal.md) has its own page.

## capsule_about

System information viewer. App `app.about` on port 4710, caps `0x1819`, entry
`userland/capsule_about/src/main.rs:27`.

- **Behavior**: renders tabbed panels of build, ABI, capability, product, trust, license, and uptime
  data (`src/about/data/`), with arrow and page navigation through a scrolling viewport and Tab to switch
  panels. A read-only inspector of the running system.

## capsule_calculator

An arithmetic calculator. App `app.calculator` on port 4720, caps `0x1819`, entry
`userland/capsule_calculator/src/main.rs:27`.

- **Behavior**: renders a button grid and keeps operand and pending-operation state (`src/calc/state.rs`),
  with equals, memory recall/store/add/subtract, percentage, and square root; the display formats
  integers and fractions. A self-contained app with no IPC beyond the window.

## capsule_file_manager

A filesystem browser. App `app.file_manager` on port 4724, caps `0x1819`, entry
`userland/capsule_file_manager/src/main.rs:27`.

- **Behavior**: loads directory entries on open and shows a filtered, sorted list with a preview pane
  (hex, text, or info), with arrow and page navigation, filtering, selection, and copy, paste, and delete
  through prompts. A split-pane browser over the [VFS](core-services.md), and one of the surfaces
  `fs_proofs` exercises.

## capsule_hello

The minimal example. App on `userland/capsule_hello/src/main.rs`, an app-skeleton GUI app.

- **Behavior**: draws the static text "hello, NONOS" with a subtitle "signed, attested capsule" in a
  small window and closes on Escape. It is the smallest complete `App` implementation, the reference to
  read first alongside [writing an app](../writing-an-app.md).

## capsule_settings

The settings editor. App `app.settings` on port 4728, caps `0x1819`, entry
`userland/capsule_settings/src/main.rs:27`.

- **Behavior**: loads a schema of editable fields (bool, u8, i8, string) and edits them through the
  [policy](core-services.md) service, issuing typed get and set ops to the policy port. It renders tabs
  and editable value fields with visual feedback. Honest gap: it needs the policy port to be up and has
  no fallback if it is absent.

## capsule_text_editor

A simple text editor. App `app.text_editor` on port 4726, caps `0x1819`, entry
`userland/capsule_text_editor/src/main.rs:27`.

- **Behavior**: edits `/notes.txt` (a 16 KiB buffer), with insert, backspace, open and save prompts, and
  clipboard copy and paste through the [clipboard](desktop.md) capsule. It persists only to `/notes.txt`;
  clipboard access is optional and its absence is ignored silently.

## capsule_snake

A snake game. App on `userland/capsule_snake/src/main.rs`, an app-skeleton GUI app.

- **Behavior**: keeps the snake body, food, and score with Ready, Running, Paused, and GameOver phases;
  arrow keys steer and space pauses, and each frame advances the game. Its RNG is a weak non-cryptographic
  PRNG, which is fine and stated for a game.

## capsule_ripgrep

A text-search tool contract. `userland/capsule_ripgrep/`.

- **Status: not implemented.** There is no application source; only a README defining the intended
  contract (a signed tool endpoint with the `0x19` capability mask, IPC, memory, and syscall only, no
  hardware or persistence). This is documented honestly as a defined-but-unbuilt capsule, not a working
  ripgrep port.

## capsule_wallet_nonos

An Ethereum and NOX wallet. App on `userland/capsule_wallet_nonos/src/main.rs`, an app-skeleton GUI app.

- **Behavior**: manages accounts and balances through the [keyring](core-services.md), fetches nonce,
  balance, and fees over JSON-RPC on TLS 1.3 (`src/wallet/tls13/`, `src/wallet/rpc/`), and signs and
  broadcasts Ethereum and NOX transactions (`src/wallet/event/sign_eth.rs`, `sign_nox.rs`), reading the
  NOX rail contracts from [policy](core-services.md). It is a real wallet with Home, Receive, Send, and
  Proofs views. Honest limits: it needs the keyring and policy ports at runtime, network probes may time
  out, and TLS validation anchors the GTS root.

## capsule_gui_proof

Proves the standard library drives a real GUI. App-skeleton GUI app,
`userland/capsule_gui_proof/src/app.rs`.

- **Behavior**: stores a label in a `nonos_std::collections::HashMap`, increments a click counter on
  button-down events, and paints the label (via `nonos_std::format`) and the count, closing on Escape. It
  is the runtime evidence that the [nonos_std crate](../nonos-std.md) collections and formatting work in a
  GUI capsule.

## capsule_std_proof

Proves unmodified crates.io crates run on the std PAL. A CLI proof,
`userland/capsule_std_proof/src/main.rs`.

- **Behavior**: parses JSON with `serde_json`, matches words with `regex`, and encodes with `base64`, all
  unmodified crates.io crates, and prints the results to stdout. It runs as a pure `std` binary through
  the [std PAL](../std-pal.md), and is the concrete evidence that unmodified Rust crates compile and run
  on NØNOS.

## capsule_input_proof

Proves input delivery to GUI apps. App-skeleton GUI app,
`userland/capsule_input_proof/src/proof/app.rs`.

- **Behavior**: latches "surface composited" on the first paint and logs every `InputEvent` it receives,
  rendering the latches and markers. Honest scope: the markers are latches rather than formal proofs, and
  debug output ordering can race the paint.

## capsule_input_probe

An input-router protocol probe. A server daemon (not a GUI app),
`userland/capsule_input_probe/src/main.rs:13`.

- **Behavior**: initializes a heap and runs a router-protocol server, listening for subscribe and grab
  requests and delivering `InputEvent` frames in the wire format, to exercise the input protocol. Honest
  scope: it is a protocol test harness, and parts of the server loop are scaffolding.

## capsule_proof_io

Proves the syscall interface behaves. A bare-metal proof, `userland/capsule_proof_io/src/main.rs:37`.

- **Behavior**: calls the time syscall 1024 times asserting success, then checks that a bad syscall tag
  returns `ENOSYS`, an invalid pointer returns `EFAULT`, an oversized length returns `EINVAL`, and four
  retired syscall tags each return `ENOSYS`. It exits 0 on pass and a nonzero code on the specific
  failure, printing a proof marker. It is the runtime check that the [syscall boundary](../../subsystems/syscall/boundary.md)
  rejects the malformed as designed.

## Security analysis

Every capsule on this page shares one property and it is the reason the list reads the way it does: none
of them holds a hardware capability. The GUI apps run with the app mask `0x1819`, which decodes to
CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate (bits from
`src/capabilities/types.rs`), and nothing else. There is no Mmio, no Irq, no Dma, no Pio, no
InputSource, and no DeviceEnum in that mask. So the calculator cannot touch a port, the file manager
cannot map a device register, and the text editor cannot post a synthetic keystroke. An app draws into a
surface the compositor owns and reaches the rest of the system only over IPC, where the service on the
other end applies its own checks. When `capsule_settings` edits a value it is issuing a typed op to the
policy port, and policy decides whether the write is allowed; the settings app has no privileged path of
its own. The tool capsules go narrower still: `capsule_ripgrep`'s intended mask is `0x19` (CoreExec,
IPC, Memory), the same three bits `capsule_proof_io` needs, which is the smallest useful mask in the
system, IPC and memory and the right to run.

The one app that carries real value is `capsule_wallet_nonos`, and it is deliberately the least
trusted with secrets: it holds no private key, delegates all signing to the [keyring](keyring.md) over
IPC, and authenticates its RPC endpoint with a from-scratch TLS 1.3 handshake pinned to a single root.
Its full trust model is on the [wallet page](wallet-nonos.md). The point for this catalogue is that even
the wallet runs inside the same app capability envelope as the calculator; the difference is entirely in
which services it is allowed to call, not in any extra hardware authority.

The proof capsules are a separate category and it is worth being blunt about what they are. They are
self-tests, not applications. `capsule_proof_io` asserts that the syscall boundary rejects the malformed
(a bad tag returns `ENOSYS`, a bad pointer `EFAULT`, an oversized length `EINVAL`) and exits with a
status code that encodes pass or the specific failure. `capsule_std_proof` runs unmodified `serde_json`,
`regex`, and `base64` to show the std PAL carries real crates. `capsule_gui_proof` and
`capsule_input_proof` latch runtime facts (a HashMap drove a label, an `InputEvent` arrived) and paint
them. These prove a guarantee holds at runtime; they are demonstrations by design and are labelled as
such above rather than dressed up as features. `capsule_ripgrep` is the honest opposite: it is a defined
contract with no implementation, a README and a `Capsule.mk` but no `src/`, so it is listed as
not-implemented rather than as a working search tool.

## Debugging

The first question for any capsule here is whether it spawned at all, and the boot log answers it. Each
app is brought up through `super::boot::capsule(prefix, ...)`
(`src/userspace/init/spawn_plan/apps.rs`, `apps_tools.rs`), which routes to `capsule_boot::boot`
(`src/userspace/init/capsule_boot/run.rs:29`) and prints `boot_log::ok(prefix, "capsule spawned")` on
success or `boot_log::error(...)` on failure. So a live app produces a line naming it: `[APP-ABOUT]`,
`[APP-CALCULATOR]`, `[APP-FILE-MANAGER]`, `[APP-SETTINGS]`, `[APP-TEXT-EDITOR]`, `[APP-SNAKE]`,
`[APP-HELLO]`, `[APP-TERMINAL]`, `[APP-NONOS-WALLET]`, and `[APP-INPUT-PROOF]`, each followed by
`capsule spawned`. If that line is absent the capsule never ran, and the cause is upstream of the app: the
ELF failed signature verification or its manifest asked for a capability outside policy, so the spawn was
refused before any of the app's own code executed. The proof capsules print their own markers the same
way: `[STD-PROOF]` and `[RIPGREP]` come from `src/userspace/init/entry.rs:56` and `:65`.

If the capsule spawned but does nothing useful, the failure has moved to a service it depends on, and the
app is honest about which one. `capsule_settings` needs the policy port up and has no fallback if it is
absent, so a settings window that renders but cannot save is a policy-service problem, not a settings
bug. `capsule_text_editor`'s clipboard access is optional and its absence is ignored silently, so copy
and paste failing while editing still works points at the clipboard capsule. The proof capsules make the
cleanest debugging targets because their result is a status code and a printed marker rather than a UI:
`capsule_proof_io` exits 0 on pass and a nonzero code that names the specific check that failed, so its
exit status is the whole report. That property is why it is the first thing to run when the syscall
boundary is suspected.

## Source map

Each capsule's source is `userland/<name>/`, the app model is [writing an app](../writing-an-app.md),
and the kernel-side embed and spawn are under `src/userspace/capsule_<name>/`. The spawn plan that names
each capsule and its boot prefix is `src/userspace/init/spawn_plan/apps.rs` and `apps_tools.rs`, the
shared spawn-and-log wrapper is `src/userspace/init/capsule_boot/run.rs`, the proof-capsule markers are
in `src/userspace/init/entry.rs`, and the capability bits every mask above decodes against are in
`src/capabilities/types.rs`. The two deepest capsules have their own pages,
[capsule_wallet_nonos](wallet-nonos.md) and [capsule_terminal](terminal-full.md). Every entry is verified
against those trees with `file:line` references, and the proof and stub capsules are labeled as such
rather than presented as finished features.
