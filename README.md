# pulseengine-wasm-layers

The **`pulseengine-wasm`** realm: the bytecodealliance component-model
toolchain — `wasm-tools`, `wac`, `wkg`, `wit-bindgen-wrpc` — assembled into
signed varve layers.

```toml
# varve-realms.toml
[realm.pulseengine-wasm]
registry   = "oci://ghcr.io/pulseengine/wasm-layers"
trust-root = "f1300acb791d330da9b81951d4137f5b1fb0c5efc50213deee4d7a981be28745"
```

## Whose root signs this, and whose software it is

The payloads come from **bytecodealliance** releases. The key that signs these
layers is held by **PulseEngine**. That is why the realm is called
`pulseengine-wasm` and not `bytecodealliance`: a realm name is a claim about
who vouches, and naming a realm after an upstream would imply an endorsement
nobody gave.

Every payload is still verified against **its own repository's** cosign
identity or attestation before a byte is staged. What this realm vouches for is
the assembly — these exact bytes, from those exact releases, in one signed
manifest — not the software itself.

The public half of the root is committed here as `realm-root.pub`. varve's own
release ships `rolling.pub`, which is the **`pulseengine`** realm's root, not
this one — a realm that varve does not ship has to publish its own public half
where consumers can get it. The secret half is a repository secret and never a
file.

## Why this repository exists — the worked multi-layer example

varve has claimed layer composition since early on: one pin, two trust
universes, each layer keeping its own root and cadence. It was implemented,
tested and documented — and until varve v0.37.0 **a realm's `layer.toml` could
not express it at all**. `[[include]]` lived only in a hand-written deposit
spec, so the only demonstration of composition was a system test that generated
two throwaway roots in a temporary directory and deleted them.

This realm is the first half of a real one:

| realm | what it carries | root held by |
|---|---|---|
| `pulseengine` | the verification toolchain (rivet, spar, witness, …) | PulseEngine |
| **`pulseengine-wasm`** | **the component-model toolchain (this repo)** | **PulseEngine** |
| `covalent` | neither — it **composes** the two above | PulseEngine |

A consumer pins the `covalent` layer and gets all three realms' tools, with
each layer verified against **its own** realm's root. That is what composition
is for, and it is the thing that has never existed outside a test fixture.

## Four upstreams, four naming conventions

None of this is hypothetical, and none of it requires patching anything — it is
what a realm manifest has to be able to say:

| tool | shape | how the asset is named |
|---|---|---|
| `wasm-tools` | tarball per platform | `%U` — bytecodealliance's own `aarch64-macos` style, not a Rust triple |
| `wac` | bare binary | file is `wac-cli`, tool is `wac`; Linux builds are **musl**, so the four names are stated |
| `wkg` | bare binary | names ARE Rust triples, so `%T` derives all four |
| `wit-bindgen-wrpc` | bare binary | repo is `wrpc`, binary is `wit-bindgen-wrpc`; Linux musl again |

`varve-producer plan --manifest layer.toml` resolves all 16 payloads from those
four releases before anything is fetched.

## How a layer gets published

Same shape as `pulseengine-layers`, and the tooling here is a copy of it:

- **`layer.toml` is the layer.** Bumping a version there is the whole change.
- **`tools/scan-pins.py`** asks each upstream for its latest release. It reads
  every section the manifest defines, and it compares the tag an entry **asks
  for** (`release` when present, else `version`) — a payload whose version is
  its own number, not its release tag, is reported for a person rather than
  rewritten.
- **`tools/check-manifest.py`** refuses a repo pinned at two releases, a tag
  that is not upstream, a `vsix` template with no `%V`, and a document naming a
  payload the layer does not carry. "I could not ask" is never reported as "it
  is not there".
- **`.github/workflows/deposit.yml`** assembles, signs, sanity-installs and
  verifies against `realm-root.pub`, then publishes. If the secret is not the
  secret half of that root, it fails on the runner rather than on every
  consumer's machine.

## What is deliberately NOT here yet

- **No scheduled deposit.** `pulseengine-layers` scans and deposits every 15
  minutes unattended (its DD-030 records what that gives up). This realm
  deposits when dispatched, until there is a reason to do otherwise.
- **The tooling is copied, not shared.** Two repositories now carry the same
  four Python files. That is a known cost, taken deliberately to get a second
  realm standing; the moment a third appears, it should become a released
  package rather than a third copy.
