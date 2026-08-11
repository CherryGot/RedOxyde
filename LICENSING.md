# Licensing

RedOxyde is dual-licensed along a single architectural seam:

> **Contracts are MIT. Implementations are AGPL-3.0.**

Everything below follows from that one rule.

---

## The short version

**Writing a proprietary extension is fine. Embedding the kernel in a
proprietary native application is not.**

If your code talks to RedOxyde across the WebAssembly boundary — through the
published WIT transport interfaces — you are unencumbered, and you may license
your work however you like, including closed-source and commercial. If your
code links the kernel or core directly into your own binary, you are building
on AGPL-3.0 code and the AGPL applies to your combined work.

---

## Per-component licenses

| Component | License | SPDX |
| --- | --- | --- |
| WIT interface contracts — the transport interfaces and any published bus or service contract | MIT | `MIT` |
| Per-language SDKs | MIT | `MIT` |
| Anything emitted by the code generator | MIT | `MIT` |
| Kernel runtime | AGPL-3.0-or-later | `AGPL-3.0-or-later` |
| Build and packaging tooling | AGPL-3.0-or-later | `AGPL-3.0-or-later` |
| Core (the CMS) | AGPL-3.0-or-later | `AGPL-3.0-or-later` |

Full texts: [`LICENSE`](LICENSE) (AGPL-3.0) and [`LICENSE-MIT`](LICENSE-MIT).

Each crate declares its license in its `Cargo.toml` and carries a matching
`LICENSE` file. That declaration is authoritative for that crate.

---

## Why the seam is where it is

An extension is a separately-compiled `.wasm` artifact. It runs in its own
sandbox with its own linear memory and never links a byte of kernel or core
code. It reaches the platform by handing `(target, method, args)` bytes to
the host, which routes them. That is arms-length communication — closer to
calling a server over HTTP than to linking a library.

The only code an extension actually links is the MIT SDK and MIT-generated
wrappers, so an extension is unencumbered by construction.

---

## Generated code

**Output of the code generator carries no license obligation and belongs to
whoever ran it.** Type definitions, service wrappers, dispatcher demuxes, and
hook descriptors generated from any published WIT file may be used, modified,
and distributed under any terms, including proprietary ones.

This mirrors the GCC Runtime Library Exception — without such a grant, every
program built with the generator would inherit the generator's license.

---

## What this means for you

**Extension authors.** Build whatever you want, license it however you want,
sell it if you want. You depend only on MIT code. Nothing in the AGPL reaches
your extension.

**Hosting providers.** Running RedOxyde — modified or not — for your customers
is fine. If you *modify* the kernel or core and offer it over a network, AGPL
§13 requires you to offer those modifications' source to your users. Running
stock RedOxyde triggers nothing at all. Your infrastructure, control panel,
provisioning, and support tooling are yours and are not covered.

**Native embedders.** Linking `kernel` or `core` into your own Rust binary
creates a combined work under AGPL-3.0. You may absolutely do this — but the
result is AGPL, and you owe source to your users. If you want to build on
RedOxyde commercially without that obligation, build across the wasm boundary
instead. That path is fully supported and is the intended one.

**Contributors.** Contributions to the kernel and core are accepted under
AGPL-3.0-or-later; contributions to the SDK and contract layer under MIT.
Contribution mechanics — sign-off or CLA — will be settled before the project
starts accepting outside patches.

---

## Trademark

The RedOxyde name and logo are separate from the code license. The AGPL grants
rights to the software; it grants no rights to the mark. Forks are welcome and
always will be — but a fork may not present itself as RedOxyde, or use the name
in a way that implies endorsement or affiliation.

A formal trademark policy will accompany the first public release.

---

## Questions this does not answer

This document explains intent in plain language. It is not legal advice, and
where it and the license texts diverge, `LICENSE` and `LICENSE-MIT` govern.
