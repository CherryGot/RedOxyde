# RedOxyde

A WebAssembly plugin kernel for extensible applications, written in Rust.

RedOxyde gives an application a hook and service system in the spirit of
WordPress — actions, filters, and inter-plugin APIs — but with extensions
compiled to WebAssembly components, isolated from each other, capability-gated,
and typed end to end. Extensions can be written in any language that targets
the Component Model.

**Status: pre-release, no implementation on this branch yet.** The architecture
is settled. A working proof of concept — kernel, SDKs, tests, benchmarks —
lives on the `canary` branch. The public implementation is being rebuilt here
from scratch.

## Licensing in one line

Writing a proprietary extension is fine. Embedding the kernel in a proprietary
native application is not.

The kernel and core are AGPL-3.0-or-later. The WIT transport contracts, the
per-language SDKs, and everything the code generator emits are MIT — so
extensions link only MIT code and carry whatever license their author chooses.
Details in [`LICENSING.md`](LICENSING.md).
