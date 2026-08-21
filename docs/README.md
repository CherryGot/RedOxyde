# RedOxyde documentation

| Document | What it's for |
| --- | --- |
| [explanation/kernel-architecture.md](explanation/kernel-architecture.md) | The extension host: the problems it targets, the approach, alternatives considered, and open questions |

Licensing lives at the repository root in [`LICENSING.md`](../LICENSING.md).

## How this is organised

Documentation is written in four modes, following
[Diátaxis](https://diataxis.fr). Each page picks one and stays in it — a
tutorial that stops to explain design derails the beginner, and reference that
teaches becomes unnavigable.

| Mode | Serves | Answers |
| --- | --- | --- |
| **Tutorial** | someone learning | "take me through my first extension" |
| **How-to** | someone working | "how do I subscribe to a filter?" |
| **Reference** | someone working | "what exactly does this interface do?" |
| **Explanation** | someone understanding | "why is the transport shaped this way?" |

## Growth plan

Folders appear when there is content to put in them, not before.

- `getting-started/` and `how-to/` — once there is something runnable
- `reference/` — once there is a stable API surface
- `contributing/` — once there are contributors. Deliberately outside the four
  modes: it documents the project, not the product
