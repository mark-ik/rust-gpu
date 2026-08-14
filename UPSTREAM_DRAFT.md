# Upstream filing draft

Not sent. Two pieces: a bug report, and the PR text if they want the
fix as-is. Both kept short on purpose — the bug is one comparison and
the explanation should fit in a screen.

Branch: `mark-ik/prerelease-version-gate` on
`https://github.com/mark-ik/rust-gpu` (one commit, 13 lines, all
comment except the version rebuild).

---

## Issue

**Title:** Target spec selection picks the wrong variant on the pinned
nightly (`allows-weak-linkage` on rustc 1.97)

### Expected Behaviour

Building a shader crate with the toolchain `rust-toolchain.toml` pins
should work.

### Example & Steps To Reproduce

1. `cargo install --git https://github.com/Rust-GPU/cargo-gpu cargo-gpu`
2. Create a minimal shader crate depending on `spirv-std` from this
   repo's `main`, with a `rust-toolchain.toml` matching this repo's
   (`nightly-2026-05-22`).
3. `cargo gpu build --shader-crate . --auto-install-rust-toolchain \
    --spirv-builder-source https://github.com/Rust-GPU/rust-gpu \
    --spirv-builder-version <main sha>`

Result:

```
error: error loading target specification: allows-weak-linkage: unknown
field `allows-weak-linkage`, expected one of `llvm-target`, ...
```

### Cause

`TargetSpecVersion::from_rustc_version` gates on

```rust
if rustc_version >= Version::new(1, 97, 0) {
```

and the pinned toolchain reports `rustc 1.97.0-nightly (e96c36b6f
2026-05-21)`. In semver a pre-release sorts *before* its release, so
`1.97.0-nightly < 1.97.0`, the `Rustc_1_94_0` variant is selected, and
it emits `allows-weak-linkage` — the key rustc 1.97 removed, and the
one the `Rustc_1_97_0` variant exists to drop.

The window is one release wide: a `1.98.0-nightly` passes every gate
here, because the triple comparison dominates once the minor differs.
It only bites when the nightly's own release line is the gate, which
is exactly the case for the pinned toolchain.

### System Info

- Windows 11, `x86_64-pc-windows-msvc`
- rust-gpu `main` (`eb73c07`), cargo-gpu `43fd7df`
- `rustc 1.97.0-nightly (e96c36b6f 2026-05-21)`

### Possible fix

Compare on the version triple, ignoring the pre-release, since each
gate means "this rustc release line or later". Patch on
`mark-ik/prerelease-version-gate`; happy to open it as a PR.

Note the same fix has to reach `cargo-gpu`, which links
`rustc_codegen_spirv-types` directly, so patching this repo alone does
not change its behaviour.

---

## PR

**Title:** Compare rustc version gates on the triple, ignoring
pre-release

Target spec selection currently fails against the nightly this repo's
`rust-toolchain.toml` pins.

`from_rustc_version` gates on `rustc_version >= Version::new(1, 97, 0)`,
and the pinned toolchain reports `1.97.0-nightly`. semver sorts a
pre-release before its release, so the older variant is chosen and
emits `allows-weak-linkage`, which rustc 1.97 removed — the exact key
`Rustc_1_97_0` exists to drop.

Each gate in that function means "this rustc release line or later", so
this compares on the triple alone. Later nightlies already pass because
the triple comparison dominates; this makes the pinned one pass too.

Verified by building a compute shader crate end to end on
`nightly-2026-05-22` with the patch applied: SPIR-V is produced,
passes `spirv-val`, and runs under `wgpu` via
`PASSTHROUGH_SHADERS`.

---

## Disclosure

rust-gpu has no LLM policy of its own; `rust-lang/rust` adopted one on
2026-08-05 that is explicitly scoped to that repository. Following it
voluntarily as the nearest norm, and because its disclosure rule is
the right default:

> This bug was diagnosed and the patch drafted with AI assistance
> (Claude), reviewed and verified by me before filing.

Keep that line in whichever of the two we send.
