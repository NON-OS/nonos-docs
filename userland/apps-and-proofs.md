# Applications and Proof Capsules

This group is two things. The first is the set of end-user applications: the
[app-skeleton](writing-an-app.md) GUI programs a person actually opens and uses. The second is the
set of proof capsules: self-tests that assert one specific runtime guarantee and are honest about being
demonstrations rather than features.

The interactive apps each have their own dedicated reference page now, so this page does not repeat their
detail. It gives a one-line pointer to each and keeps full inline coverage only for the proof and stub
capsules that do not have a page of their own yet: `capsule_hello`, `capsule_ripgrep`,
`capsule_gui_proof`, `capsule_std_proof`, `capsule_input_proof`, `capsule_input_probe`, and
`capsule_proof_io`.

## Interactive applications

Each of these runs with the app capability mask `0x1819` (CoreExec, IPC, Memory, GraphicsDisplayQuery,
GraphicsSurfaceCreate) and holds no hardware authority. Follow the link for the full reference on any of
them.

| App | Endpoint | One-line summary |
| --- | --- | --- |
| [About](about/README.md) | `app.about`, port 4710 | Read-only introspection window over build, ABI, capability, display, uptime, and the AGPL-3 license text; writes nothing back. |
| [Calculator](calculator/README.md) | `app.calculator`, port 4720 | Self-contained keypad and fixed-point arithmetic engine that reaches nothing outside its own window; the cleanest least-privilege app in the tree. |
| [File manager](file-manager/README.md) | `app.file_manager`, port 4724 | Split-pane directory browser with previews that performs every file operation as an IPC call to the `vfs_pool` service, which holds the real authority. |
| [Settings](settings/README.md) | `app.settings`, port 4728 | System control panel whose Display, Network, and Security rows are live reads and writes against the `policy` service; it owns no policy of its own. |
| [Text editor](text-editor/README.md) | `app.text_editor`, port 4726 | Focused single-document editor over one fixed-capacity buffer, loading and saving through vfs and copying and pasting through clipboard; not an IDE. |
| [Snake](snake/README.md) | `app.snake`, port 4732 | The classic snake game as a signed capsule; the smallest complete interactive app in the tree, owning nothing but its own state and surface. |
| [Terminal](terminal/README.md) | see page | The shell and command surface; the largest app-skeleton app, reaching the filesystem, network, and installer over IPC. |
| [Wallet](wallet-nonos/README.md) | see page | An Ethereum and NOX wallet that holds no private key, delegates all signing to the keyring, and pins its RPC endpoint with a from-scratch TLS 1.3 handshake. |

## Proof and stub capsules

The tutorial capsule, the one unbuilt contract, and the runtime self-tests. Each has its own page; the
honest label on each is preserved.

| Capsule | Kind | One-line summary |
| --- | --- | --- |
| [hello](hello/README.md) | tutorial | The minimal complete `App`: draws static text in a small window and closes on Escape; the reference to read first alongside [writing an app](writing-an-app.md). |
| [ripgrep](ripgrep/README.md) | not implemented | A text-search tool contract with no application source, only a README defining the intended signed endpoint. Documented honestly as defined-but-unbuilt, not a working ripgrep port. |
| [gui-proof](gui-proof/README.md) | self-test | Runtime evidence that the [nonos_std](nonos-std.md) collections and formatting drive a real GUI: a HashMap label and a click counter painted through `nonos_std::format`. |
| [std-proof](std-proof/README.md) | self-test | Evidence that unmodified crates.io crates run on the [std PAL](std-pal.md): parses JSON with `serde_json`, matches with `regex`, encodes with `base64`, all as a pure `std` binary. |
| [input-proof](input-proof/README.md) | self-test | Latches surface composition on first paint and logs every `InputEvent`; honest scope, the markers are latches rather than formal proofs. |
| [input-probe](input-probe/README.md) | test harness | A router-protocol server daemon that exercises the subscribe, grab, and delivery wire format; a protocol test harness, parts of the loop are scaffolding. |
| [proof-io](proof-io/README.md) | self-test | Asserts the [syscall boundary](../../subsystems/syscall/boundary.md) rejects the malformed: bad tags return `ENOSYS`, a bad pointer `EFAULT`, an oversized length `EINVAL`; exits nonzero on the specific failure. |

## Security analysis

Every capsule on this page shares one property and it is the reason the list reads the way it does: none
of them holds a hardware capability. The GUI apps run with the app mask `0x1819`, which decodes to
CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate (bits from
`src/capabilities/types.rs`), and nothing else. There is no Mmio, no Irq, no Dma, no Pio, no
InputSource, and no DeviceEnum in that mask. So the calculator cannot touch a port, the file manager
cannot map a device register, and the text editor cannot post a synthetic keystroke. An app draws into a
surface the compositor owns and reaches the rest of the system only over IPC, where the service on the
other end applies its own checks. When settings edits a value it is issuing a typed op to the policy
port, and policy decides whether the write is allowed; the settings app has no privileged path of its
own. The tool capsules go narrower still: `capsule_ripgrep`'s intended mask is `0x19` (CoreExec, IPC,
Memory), the same three bits `capsule_proof_io` needs, which is the smallest useful mask in the system,
IPC and memory and the right to run.

The one app that carries real value is the wallet, and it is deliberately the least trusted with secrets:
it holds no private key, delegates all signing to the [keyring](keyring/README.md) over IPC, and authenticates
its RPC endpoint with a from-scratch TLS 1.3 handshake pinned to a single root. Its full trust model is
on the [wallet page](wallet-nonos/README.md). The point for this catalogue is that even the wallet runs inside
the same app capability envelope as the calculator; the difference is entirely in which services it is
allowed to call, not in any extra hardware authority.

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

If the capsule spawned but does nothing useful, the failure has moved to a service it depends on. The
[settings](settings/README.md) app needs the policy port up and has no fallback if it is absent, so a settings
window that renders but cannot save is a policy-service problem, not a settings bug. The
[text editor](text-editor/README.md)'s clipboard access is optional and its absence is ignored silently, so copy
and paste failing while editing still works points at the clipboard capsule. The proof capsules make the
cleanest debugging targets because their result is a status code and a printed marker rather than a UI:
`capsule_proof_io` exits 0 on pass and a nonzero code that names the specific check that failed, so its
exit status is the whole report. That property is why it is the first thing to run when the syscall
boundary is suspected.

## Source map

Each capsule's source is `userland/<name>/`, the app model is [writing an app](writing-an-app.md),
and the kernel-side embed and spawn are under `src/userspace/capsule_<name>/`. The spawn plan that names
each capsule and its boot prefix is `src/userspace/init/spawn_plan/apps.rs` and `apps_tools.rs`, the
shared spawn-and-log wrapper is `src/userspace/init/capsule_boot/run.rs`, the proof-capsule markers are
in `src/userspace/init/entry.rs`, and the capability bits every mask above decodes against are in
`src/capabilities/types.rs`. The interactive apps carry their own full references:
[about](about/README.md), [calculator](calculator/README.md), [file manager](file-manager/README.md), [settings](settings/README.md),
[text editor](text-editor/README.md), [snake](snake/README.md), [terminal](terminal/README.md), and
[wallet](wallet-nonos/README.md). The proof and stub capsules on this page keep their spec inline and are
labeled as self-tests or as not-implemented rather than presented as finished features. Every reference
above is verified against those trees.
