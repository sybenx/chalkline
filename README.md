# chalkline

chalkline is a design and spec for an application layer on top of Cyberspace Protocol v2 (a Nostr-based coordinate and movement-chain protocol from arkin0x/cyberspace). It describes "artifacts": bundles of placements at coordinates, each tied to a step in its creator's movement chain. Anyone can check a bundle, and the idea is that the cost of making one is counted in serial steps, which are hard to buy or skip. It proposes no changes to the base protocol.

There is no code in this repo yet. It is docs only.

## What is in the repo

- [DESIGN.md](DESIGN.md): the intent and reasoning (non-normative). Covers the problem, the thesis, non-goals, mechanisms, and open questions.
- [spec/BUNDLE.md](spec/BUNDLE.md): the draft bundle spec (normative). Covers bundle structure, canonical serialization, stat derivation, and the verification algorithm.
- [NOTES/protocol-facts.md](NOTES/protocol-facts.md): notes transcribed from the upstream protocol and its reference CLI, with file and line citations and a list of what is not confirmed.

## What it runs on / how to install

Nothing is runnable. There is nothing to install.

## Status

This is an early design and may change. The spec lists several open questions that must be settled before it leaves draft.

Built for my own use. It works on my setup. No support promised.
