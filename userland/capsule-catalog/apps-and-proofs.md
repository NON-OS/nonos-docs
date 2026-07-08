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

## Source

Each capsule's source is `userland/<name>/`; the app model is [writing an app](../writing-an-app.md), and
the kernel-side embed and spawn are under `src/userspace/capsule_<name>/`. Every entry is verified against
those trees with `file:line` references, and the proof and stub capsules are labeled as such rather than
presented as finished features.
