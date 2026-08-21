# RedOxyde Kernel Architecture

> **Status: design.** Nothing here is implemented in this repository yet. This
> document describes what is being built and why, so the design can be argued
> with before it is committed to code. Open questions are listed at the end.

## What this is

RedOxyde is a content management system. Its extensibility comes from an
**extension host** that ships with it: a small, domain-agnostic layer that runs
the CMS itself and any third-party code alongside it, on equal terms. This is
the microkernel pattern, and this document calls that layer the **kernel**.

The kernel is the part that lets third-party code extend an application without
being able to break it. It provides two communication primitives — an event bus
and typed service calls — and runs every extension as an isolated WebAssembly
component. Extensions declare what they provide and consume in a manifest; the
kernel resolves those declarations, checks capabilities, and wires the
dependency graph before any extension code runs.

The kernel knows about extensions, buses, services, and capabilities. It does
not know about posts, users, HTTP, or anything else application-shaped. Those
live in **core** — the CMS itself, and the first extension the kernel loads. A
different application domain could ship a different core against the same
kernel.

That separation is the central idea: **core is just another extension** — the
privileged one, which runs first and happens to ship with the host. Every
mechanism described below works identically whether the two parties are
core-and-extension or extension-and-extension.

## The problem

WordPress proved that a hook-based plugin ecosystem creates enormous value. It
also demonstrated the failure modes of building one without isolation, and
those failures are structural rather than incidental.

**Hooks are registered by running code.** A plugin calls `add_action` during
bootstrap. To learn what a plugin hooks, you must execute it. There is no point
at which the host can audit intent before granting it — which means there is no
point at which the host can refuse.

**No isolation.** Plugins share one address space and one interpreter. One
plugin's fatal error takes down the request. Any plugin can read or overwrite
any other's state, redefine core behaviour, or exfiltrate whatever it can
reach.

**No capability model.** Installation is a single, total trust decision. A
plugin that only formats dates receives the same access to the database,
filesystem, and network as one that processes payments.

**No types across the boundary.** Filters pass whatever the emitter happened to
pass, and every subscriber guesses. Contract changes are discovered in
production.

**One language.** The plugin author pool is exactly the PHP pool, and the
performance ceiling is PHP's.

**Density.** Per-tenant memory is dominated by the interpreter, which caps how
many sites fit on a box and therefore what hosting costs.

Each of these is a consequence of the same root decision — extensions are
untyped code running inside the host process with ambient authority. RedOxyde
changes that decision and inherits different properties.

## Design principles

These are constraints, not preferences. Most of the design is derivable from
them.

**1. No VMs on the server.** Officially supported extension binaries are
ahead-of-time compiled machine code. Languages whose wasm toolchain embeds its
own runtime — a JS engine, CPython, a JVM — are explicit non-goals for this
project, however attractive the developer pool. An embedded interpreter costs
roughly 2–10 MB per extension before any plugin code runs; thirty extensions
would carry ~90 MB of pure runtime overhead, which erases the density advantage
that motivates using wasm at all.

**2. Capabilities are not advisory.** Every call between extensions passes
through a broker that checks it against the caller's declarations. There is no
trusted internal path and no bypass.

**3. The manifest is the source of truth.** Everything the kernel needs to know
about an extension is declared: buses and services provided and consumed,
subscriptions, capabilities. The kernel never infers these by inspecting or
executing the binary. If it is not declared, it does not happen.

**4. The kernel is application-domain-free.** No CMS concepts leak into it.

**5. One wire format across all languages.** No per-language dialects.

## Two primitives

Extensions need both fan-out notification and point-to-point calls. Collapsing
them into one primitive makes both worse, so there are two.

|  | **Bus** | **Service** |
|---|---|---|
| Shape | one owner → many subscribers | one caller → one provider |
| Modes | action (fire-and-forget), filter (value pipeline) | request → response |
| Failure | subscriber failure is logged; the emit still succeeds | caller receives a `Result` |
| Example | "a post was created" | "give me post 42" |

A **bus** is owned by exactly one extension and carries named **hooks**. The
owner publishes an interface definition describing the *type vocabulary* of the
bus — which events exist and what they carry — and nothing else. No function
signatures, no exports.

A **service** is a typed callable surface. Its interface definition lists
methods and their signatures.

Subscribers declare a **priority**; the engine dispatches in priority order.
Filters pass a value down that chain, each subscriber returning a possibly
modified value.

## The transport contract

The host implements exactly **four interfaces**, fixed at kernel compile time
and identical on every extension forever:

| Direction | Purpose | Interface |
|---|---|---|
| host → guest, fan-out | action and filter dispatch | `dispatcher` |
| guest → host, fan-out trigger | emitting an event | `emit` |
| guest → host, point-to-point | calling a service or a resource method | `service-call` |
| host → guest, point-to-point | receiving a call | `service-provider` |

Every bus's content rides inside that fixed envelope as **opaque bytes**. The
host routes; it does not decode. Endpoints encode and decode against the bus
owner's published interface at their own build time.

The consequence: **new buses, new hooks, and new extensions never require host
changes.** The kernel's contract is bounded and stable, and the dynamism lives
in routing tables and per-extension typing rather than in the host's surface.
New routing needs are met by varying parameters within these four — resource
handles reuse `service-call` with a richer target rather than adding a fifth
interface.

## How it fits together

```
   ┌──────────────────────── Host process ─────────────────────────┐
   │                                                               │
   │                    ┌─────────────────────┐                    │
   │                    │        Core         │                    │
   │                    │ privileged, native  │                    │
   │                    │  HTTP · storage ·   │                    │
   │                    │  entities · auth    │                    │
   │                    └──────────┬──────────┘                    │
   │                               │ typed calls, nothing encoded  │
   │                               ↕                               │
   │   ┌───────────────────────────────────────────────────────┐   │
   │   │                        Kernel                         │   │
   │   │                                                       │   │
   │   │   registry · linker · capability resolver             │   │
   │   │   bus engine · service broker                         │   │
   │   └───────────────────────────────────────────────────────┘   │
   │                             │                                 │
   └─────────────────────────────┼─────────────────────────────────┘
                                 │
              host → guest   dispatcher · service-provider
              guest → host   emit · service-call
                                 │
   ══════════════════════════════╪══════════════════════════════════
      wasm sandbox boundary — opaque bytes, no shared memory
   ══════════════════════════════╪══════════════════════════════════
                                 │
            ┌────────────────────┼────────────────────┐
            ↕                    ↕                    ↕
     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
     │ Extension A  │     │ Extension B  │     │ Extension C  │
     │  own store   │     │  own store   │     │  own store   │
     │  own memory  │     │  own memory  │     │  own memory  │
     └──────────────┘     └──────────────┘     └──────────────┘

     every link is two-way; A → B is A ↑ kernel ↓ B
```

The **registry** holds what every loaded extension provides and consumes. The
**linker** resolves those declarations into a dependency graph at load time.
The **capability resolver** decides whether a declared use is permitted. The
**bus engine** owns subscription tables and dispatches events in priority
order; the **service broker** routes point-to-point calls and enforces
protection levels on the way through.

Everything above the boundary is one process and calls each other with real
types. Everything crossing the boundary is encoded bytes through one of the
four interfaces. Core's own handlers and wasm extensions subscribe to the same
buses and appear on the same priority chains — the bus engine dispatches to
either transparently, and neither can tell which kind the other is. Core's
services are native Rust functions exposed through the same interface shapes an
extension would use, so calling one is indistinguishable from calling an
extension's.

A single dispatch, end to end: an extension calls `emit`; the broker checks it
may emit on that bus; the bus engine looks up subscribers for that hook, sorted
by priority; for each one it invokes `dispatcher` with the metadata and the
encoded payload; the subscriber decodes, runs, and — for a filter — returns a
new payload that becomes the input to the next subscriber in the chain. A
subscriber that fails is logged and the chain continues.

### Extensions are peers, not leaves

An extension can own a bus and expose services exactly as core does, and other
extensions can subscribe to and call them, through the same mechanism and the
same code path.

Declaring a use of `other-extension:some-bus` resolves through the same linker
code, against the same dependency graph, with the same capability check as
declaring a use of a core bus. The bus owner still controls who may emit on it;
the broker still enforces protection levels on every method. Nothing about
being an extension rather than core grants or withholds anything.

There is no direct channel between extensions. Traffic goes up into the kernel
and back down, which is what makes the capability check unavoidable — no path
bypasses it, and two extensions cannot trust each other privately.

The consequence is that extensions compose. An extension can extend another
extension: a bus a third party defined can accumulate subscribers a third party
wrote, all typed, all versioned, all permissioned. In WordPress this happens
through globally-visible hooks with no contract and no permission model; here
it is the same first-class mechanism the platform itself uses.

Wiring happens before any guest code runs:

```
   manifest
      │
      ├─► resolve every provider the extension declares it uses
      ├─► verify each subscribed hook exists and its kind matches
      ├─► check capabilities, prompting an administrator if required
      ├─► build the import table for this extension, and only this one
      ├─► instantiate the component
      ├─► register its exports with the bus engine and the broker
      └─► call the extension's initialiser
```

If any step fails, the extension does not load and nothing of it has executed.
The import table is built per extension from its declarations, so one
that never declared a use of something has no way to name it at runtime.

## Declarative subscriptions

Subscriptions are baked at build time and declared in the manifest. There is no
runtime `subscribe()`.

Before instantiating an extension, the kernel reads its manifest and can
therefore: resolve every provider, reject one whose dependency is missing,
verify each subscribed hook actually exists with a matching kind, check
capabilities, order the load graph, and pre-build the dispatch tables. All of
this happens with no guest code having executed.

This is the direct answer to "hooks are registered by running code." A handler
may no-op based on configuration, but the set of things an extension can hook is
fixed and inspectable before it is trusted with anything.

## Capabilities

Every bus subscription, bus emission, and service call is a permission checked
at the broker. Providers declare a protection level per surface, and the kernel
resolves it at load time:

- **granted** — automatic on declaration
- **prompt** — requires an administrator's approval
- **privileged-only** — restricted to extensions the host designates as
  privileged
- **signed** — restricted to extensions signed by a key the provider specifies

Consumers may narrow their own access — requesting only specific methods of a
service — which makes the install-time prompt precise: *this extension wants to
read posts but not create or delete them.*

Emission on a bus you do not own is restricted by default; bus owners control
who can fire their events.

## Isolation and density

Each extension gets its **own wasm store**. Not a shared store with logical
separation — an actual separate one.

A shared store would save a small amount of memory and would eliminate the
security model, since isolation would become a property of correct bookkeeping
rather than of the runtime. Payloads are serialised across the boundary either
way, so the shared-store saving is marginal.

Density is a primary design target: the point of compiling extensions AOT is to
fit substantially more tenants on a box than an interpreter-per-tenant model
allows. Dispatch cost and per-extension memory will be published with the
implementation.

## Languages

An extension is a WebAssembly component. **Any language that can produce one can
produce an extension** — the kernel loads what it is handed and does not care
what compiled it.

Official support is narrower, and deliberately so. The languages this project
supports, tests, and ships SDKs for are the ones that compile ahead of time with
no runtime tagging along, starting first with: **Rust** and **C++**. **MoonBit**
and **Zig** will be considered next. **TinyGo** carries a small runtime and
needs to be judged.

Languages whose toolchain bundles an interpreter — JavaScript via an embedded
engine, Python via CPython, anything on a JVM or CLR — will load and run. They
are simply not something this project ships or maintains. A bundled runtime
costs 2–10 MB per extension before a line of plugin code executes, and thirty of
those is ~90 MB of overhead buying nothing. Performance and resource usage are
the entire point of the design; that is not a trade this project makes.

Anyone who wants that tier is free to build and maintain it in their own
repository. The interface contracts are MIT precisely so that needs nobody's
permission. It just will not ship here.

The ergonomic layer differs per language — Rust uses attribute macros, others
will use their own idiom — but every language produces the same four exports
and imports, and interoperates on the same wire format.

## Alternatives considered

**Officially supporting embedded-interpreter languages.** Shipping and
maintaining JS or Python SDKs would open the project to a much larger developer
pool. Declined on memory: the per-extension runtime cost destroys the density
argument that justifies the whole approach. Such extensions still load — the
project simply does not ship or support that tier. Most likely to be revisited
if the memory math turns out to be wrong.

**Native dynamic libraries.** Loading extensions as shared objects with a Rust
interface. Rejected because there is no stable ABI to build on — `extern
"Rust"` is unstable and trait-object layout is not guaranteed — and because
dynamic libraries provide no isolation or capability enforcement whatsoever.

**Runtime hook registration.** The WordPress model, where extensions register
during a bootstrap phase. Rejected: it makes pre-execution auditing impossible,
which forecloses the capability model.

**A typed host interface per bus.** Instead of opaque bytes in a fixed
envelope, each bus could expose properly typed exports and let the component
model perform the demultiplexing. Technically possible — wasmtime's component
API exposes runtime type interpretation — and rejected for four reasons, the
first decisive.

*Capability enforcement.* Every call between extensions currently funnels
through one host function that performs one permission check. Typed imports
replace that with a generated host shim per method per service, each
responsible for its own check. Enforcement remains possible but correctness
moves into a code generator, for every third-party interface, permanently.
Today a codegen defect produces a compile error; under typed transport it
produces an extension calling something it was never granted, silently. One
auditable function is worth more than the ergonomics.

*Engine portability.* Runtime type interpretation is a Component Model feature
of one runtime. Interpreted-mode engines, which are the low-memory deployment
lever, have no equivalent.

*Subscription multiplicity.* A typed interface admits one implementation per
extension. Extensions legitimately subscribe to the same hook more than once at
different priorities. Recovering that means adding a subscription identifier to
every signature, which reconstructs the current design without its simplicity.

*Context coupling.* Handlers acquire request context as a scoped borrow rather
than a parameter. Typed exports would thread it through every signature, so
every bus interface would import the application's context types — collapsing
the separation between the kernel and the application built on it, and making
third-party buses depend on the CMS.

The performance argument does not favour typing either. The canonical ABI is
itself a marshalling format; typed transport substitutes one encoding for
another and adds host-side boxing of dynamic values. Encoding is not avoided by
typing it, because linear memories are separate address spaces.

Note also what the opaque design does not cost: payload typing is not dynamic.
Dispatch matches a subscription identifier — an integer — and calls a decode
path monomorphised at build time against the bus's WIT. The routing key is
dynamic; the handler is fully typed, in the manner of HTTP routing. Typed
authoring ergonomics, including listener traits with multiple handlers per
hook, remain available in each language's SDK without changing the wire.

**One shared wasm store.** Rejected; see *Isolation*.

**A single communication primitive.** Modelling services as request/response
buses, or events as fire-and-forget services. Rejected: the failure semantics
differ fundamentally — a subscriber failing must not fail the emit, whereas a
service call failing must reach the caller.

**Rust trait objects and a generic typed event API.** The natural in-process
design. Rejected because none of it survives the wasm boundary: no shared type
identity, no closures, no vtables. It remains available for host-internal
handlers, which compose on the same buses as wasm subscribers.

## Open questions

These are unsettled, and informed argument is welcome on all of them.

**Wire format.** Not settled. The candidates are the component model's
canonical ABI, a compact self-describing format such as CBOR, and a
Rust-centric binary format for an initial version. The trade is cross-language
implementation cost against encoding efficiency and tooling maturity.

**Asynchronous dispatch.** The design is synchronous. Whether buses need async
emit — and what that does to ordering guarantees and the dispatch loop — is
unresolved.

**Hot reload.** Replacing an extension that only subscribes is mechanical:
deregister its subscriptions, drop the instance, reinstantiate, re-register.
Replacing one that *owns* a bus or a service is a different problem, since
its consumers were validated at link time against the old descriptor. A
replacement that removes a hook orphans its subscribers; one that changes a
payload type leaves them decoding garbage. So the reload path has to re-run the
linker's compatibility checks against every live consumer and be willing to
refuse. Whether provider reload is supported at all, or only leaf reload, is
undecided.

Unloading also has to answer what happens to work in flight — drain or cancel,
the same choice as capability revocation below — and what happens to resource
handles the extension issued or holds. The handle registry already records an
owner, so invalidating by owner is straightforward; every holder then needs a
clean error rather than a dangling reference. That machinery is required
independently by idle eviction and by garbage-collected language SDKs, where no
borrow discipline prevents a handle outliving its dispatch. Three consumers,
one mechanism, none of it built.

A final question is whether the expensive version is needed. Under a
process-per-tenant topology with on-demand spawn, updating an extension and
retiring the tenant's process achieves replacement without any drain state
machine — the next request starts against the new artifact. What remains is
reload for extension authors iterating locally, which is single-tenant, has no
concurrent traffic, and is a much smaller feature than it first appears.

**Capability revocation in flight.** What should happen to a long-running
operation whose permission is revoked while it runs — hard cancellation, or
drain.

**Cross-language SDK parity.** How much the ergonomic layer should feel the
same across languages, given each will use its own idioms to reach identical
exports.

**Subscription identity.** Subscription IDs must agree between the manifest and
the compiled binary. Deriving them from a hash of stable inputs avoids drift
between separately generated artifacts.

## Responding to this document

The most useful feedback is specific: a failure mode in the transport design, a
memory measurement that contradicts the density argument, or experience with a
system that made the opposite choice.
