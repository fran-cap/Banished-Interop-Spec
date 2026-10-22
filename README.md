# Banished Interop Spec

Shared standards for Banished community projects, so that engines, mods and tools built by different
people can work together. Multiplayer is the first topic; mod formats and other interop standards
are expected to follow.

## Participating projects

| Project | Maintainer |
| --- | --- |
| [banished-multiplayer](https://github.com/BarryBareth/banished-multiplayer) (retail-game hook) | @BarryBareth |
| [unbanished](https://github.com/tanepiper/unbanished) (Rust engine) | @tanepiper |
| [OpenBA neighbour-towns](https://github.com/dreadylocks/OpenBA/tree/neighbour-towns) (C++ engine extension) | @dreadylocks |
| [OpenBA](https://github.com/fran-cap/OpenBA) (C++ engine) | @fran-cap |

## Documents

- [Multiplayer: comparison and a common standard](docs/multiplayer/comparison-and-proposal.md):
  how the three existing multiplayer implementations work, and a layered proposal (relay, handshake,
  presence, two sync profiles, saves). Status: draft for discussion.
- [Save tail v2](docs/saves/save-tail-v2.md): how non-retail state (rule sets, per-town extension state,
  UI state) travels inside a `.sav` that retail still opens. Status: implemented in OpenBA, proposed for
  adoption.

## How changes are made

- A spec change is a pull request; discussion happens in issues.
- A standard is adopted once every participating maintainer has approved it.
- A spec describes wire formats and behaviour only. No game content (retail assets, shaders,
  `.rsc` / `.pkg` data) belongs in this repository.
