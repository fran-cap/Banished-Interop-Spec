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

## How changes are made

- A spec change is a pull request; discussion happens in issues.
- A standard (or one layer of it) is adopted once the maintainer of every project that implements
  it has approved it. Maintainers of projects that do not implement it are advisory.
- A maintainer may ask to be advisory for any standard; their approval is then not required.
- A spec describes wire formats and behaviour only. No game content (retail assets, shaders,
  `.rsc` / `.pkg` data) belongs in this repository.

## Licence

- Documents (specs, prose, diagrams): [CC BY 4.0](LICENSE).
- Code, schemas and example messages: [MIT](LICENSE-CODE), so they can be copied into any
  participating project, whatever its own licence.
- A contribution is made under the licence of the part it changes.
