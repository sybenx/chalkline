# chalkline — Design Intent

**Status:** Draft. Non-normative. This document states *what* chalkline is and *why*; `spec/BUNDLE.md` states *how*.

**Relationship to Cyberspace:** chalkline is an application layer built on Cyberspace Protocol v2. It proposes **no changes to the base protocol**. Every mechanism here uses a primitive v2 already provides. If a design goal cannot be met without amending v2, the goal is dropped, not the protocol.

---

## 1. The problem

The v2 spec lists sybil resistance as an accepted limitation: one person may hold many keypairs, each with an independent location and chain. The spec frames this as a *multi-presence* problem — one operator being in several places at once.

There is a sharper economic version that the spec does not name:

> **Key-grinding is teleportation.** A pubkey's coordinate is a direct function of the key. If grinding a keypair is a cheaper route to a target region than moving there, the distance metric is decorative.

This resolves more favorably than it first appears. Grinding cost is *exponential in coordinate precision* while movement cost is roughly polynomial in distance, so grinding buys a coarse neighborhood and never a precise location. There is a crossover radius below which movement is the only path. Above it, rebirth wins.

But it leaves a real gap: **grinding work is unattestable.** A pubkey carries no evidence of how many candidates were discarded, and cannot — you can't prove you didn't get lucky on the first attempt. Movement work is published, chained, and verifiable. So cyberspace has a legible-work economy running alongside an illegible-work shortcut.

The consequence: **nothing yet makes identity persistence economically rational.** Absent value bound to a key, a fresh key is strictly better than a moved one.

## 2. The thesis

> **The one thing that cannot be transported to a fresh key is serial time.**

The temporal work axis makes each hop's seed derive from the previous movement event, so hop N cannot begin until hop N−1 is published. The chain is therefore *serially dependent* and cannot be parallelized, precomputed, or amortized. Capital buys throughput; it does not buy latency.

**Exception — hyperjump.** The hyperjump seed is `sha256(HYPERSPACE_SEED_DOMAIN ‖ previous_event_id ‖ be64(b))` and its proof is a merkle root over per-block work. Every block's work shares one `previous_event_id`, so the whole set computes in parallel: hyperjump is a wide parallel workload behind a single serial link. Its cost therefore *does* compress with hardware. Hop, sidestep, and enter-hyperspace all retain strict serial dependency; hyperjump alone does not. See §4.1.

chalkline turns that property into a transferable artifact. An artifact is a bundle of coordinate placements, each bound to a link in its creator's movement chain. Its cost is denominated in wall-clock serial time, and it is therefore:

- **unforgeable** — reproducing it costs what making it cost
- **hardware-resistant** — serial chains scale with single-threaded latency and event propagation, both roughly flat since ~2005, unlike parallel throughput
- **not purchasable** — a rich attacker with a fresh key cannot retroactively manufacture depth

This is why chalkline is a demonstration and not just a game: it isolates the component of v2 that ages well.

## 3. Non-goals

- **Eliminating sybils.** Alts are permanent and fine. The goal is to *price* them, not prevent them.
- **Global consensus.** Cyberspace forgoes global state by design. chalkline must work without it.
- **Combat / PvP.** Without a consensus layer a loser can simply decline to publish. Out of scope.
- **Protocol amendment.** See header.
- **Preventing copies.** See §4.5.

## 4. Mechanisms

### 4.1 Artifacts

An artifact is a set of placements — a coordinate plus payload — each anchored to a movement event in the creator's chain, terminated by a signed completion event. Verification replays the chain and confirms each placement's anchor.

**Artifacts are defined by signer, not by region.** N keys can each place pixels in the same region simultaneously, so a region-scoped artifact is a commons and trivially parallelized. A signer-scoped artifact inherits the chain's serial dependency. This distinction is load-bearing.

**Placement anchors MUST be hop or sidestep events. Hyperjump events MUST NOT anchor a placement.** Hyperjump's per-block work parallelizes across the block index (§2), so anchoring to it would let a funded actor purchase artifact depth with parallel hardware — collapsing the serial-time thesis. Enter-hyperspace is serially seeded and may anchor, but its spatial slot is zeroed (`enter_n = π(0, cantor_t)`), so it carries no location and cannot place.

The resulting rule: **hyperjump to reach a region, hops to draw in it.** Travel may be bought; the artifact may not.

Cost model: an N-placement artifact is a lower bound on serial wall-clock time by one key. Automation does not shorten it. Hardware shortens it only within a single hop's tree computation, not between hops — so speedup is bounded, not eliminated.

**Payload location:** placement data goes in tags, not `content`. The base spec's sidestep events already use `content` for the proof hash, so `content` is not reliably free across event kinds.

### 4.2 Stats

Artifact and item properties derive as a **pure function of the region root** — the deterministic Cantor-root identifier for the aligned subtree containing the placement.

Rationale: any per-item nonce (an event id, a counter) can be ground at microseconds per attempt, which would make every item in the world maximum-rolled by anyone running a script. Grinding an *event id* is nearly free; grinding a *coordinate* is brutally expensive. Putting all variance in the coordinate removes the cheap search entirely.

Consequences, all desirable:
- Regions have fixed geology. Some yield better material than others.
- Prospecting is a real activity, and long moves have purpose beyond tourism.
- Two parties derive identical stats without communicating (v2's deterministic-regions property).

> **RESOLVED.** Coordinates are Morton-interleaved, not sliced (`CYBERSPACE_V2.md:181`, `coords.py:91-98`): bit `3i+1`=Z, `3i+2`=Y, `3i+3`=X, bit `0`=plane. Landing in an aligned cube of side 2^(85−n) therefore requires matching the top `3n` bits of the pubkey — uniform across axes, no cheap axis to grind. Stat derivation may read the region root directly; no cross-axis mixing is required.

**Plane.** Bit 0 of the coordinate selects the plane, and since coord256 is the raw pubkey, the low bit of a key determines its spawn plane — one bit, trivially ground, roughly half of all keys in each. `BUNDLE.md` MUST state whether artifacts are plane-scoped, and whether a bundle's placements may span planes.

### 4.3 Transfer

Two problems, addressed separately.

**Presence.** Transfer requires both keys at the same region, established by **chain braiding**: each party's next hop takes its temporal seed from the other's latest event id. Both chains then cross-reference at a shared point, both parties paid fresh non-precomputable serial work, and neither could have staged it in advance.

Honest limit: braiding proves *region work plus interactive liveness*, not travel. The spec is explicit that region preimages can be computed without maintaining a chain. Do not build rules that assume traversal was forced.

**Double-spend.** Without global state, nothing prevents a holder from signing conflicting transfers to two parties. Resolution is tiered by what the trade already touches:

- **Paid trades (sats for artifact):** commit the transfer to the spend of a specific UTXO — a **single-use seal**. The chain never learns what an artifact is; it enforces only that the seal closes once. Because the buyer is already making a payment, this costs *zero additional on-chain footprint*: a taproot tweak on a transaction that was happening anyway. Not an extra step — a free rider on the payment.
- **Barter (artifact for artifact):** unsettled. Detectable via chain forks, not preventable. Documented as trust-based. This is a small slice of any real economy and is where scams are expected anyway.

**Design note:** a forked chain is permanent public evidence, so a chain-native fidelity bond exists as a fallback — a forked key's history becomes worthless, and rebuilding chain depth is unparallelizable. This bounds low-value fraud but fails to the exit scam, which is why seals exist for anything worth stealing.

### 4.4 Attunement

A received artifact is **inert until the new holder pays serial temporal work** scaled to the artifact's power.

This is not a smurf patch; it is the mechanism that makes the whole economy sybil-priced. Consider the courier swarm: expected minimum distance to a target across N random keys falls as roughly N^(−1/3), so a million ground keys — seconds of keygen — put one far closer than a single spawn would. Chain them in a relay, each doing one short leg.

Decomposition invariance does **not** stop this. It forbids one key from subdividing *its own* hop; a relay of distinct keys each making a genuinely short move from its own spawn walks straight past the invariant. It holds per-key, and the swarm is not one key.

Attunement taxes the relay in the dimension it cannot parallelize: a ten-link chain costs ten sequential attunement periods. Trade stays fluid, but arming a fresh smurf costs the one thing a rich main cannot wire over.

### 4.5 Copies and priority

A published bundle is verifiable, therefore readable, therefore reproducible — anyone can re-walk the path and produce a bit-identical artifact. This is acceptable and arguably correct: forging costs what making cost, like counterfeiting a painting by painting it.

What is scarce is the **first instance**. Priority comes from external timestamp anchoring (OpenTimestamps or equivalent), since Nostr timestamps are self-asserted and prove sequence but not duration. Increasingly complex artifacts are increasingly unreplicated in practice. Beyond providing verifiable priority, replication is a market problem, not a protocol problem.

## 5. Difficulty coupling

**Problem.** Hardware improves; protocol costs are fixed in absolute terms. Bitcoin retargets because it has global consensus about hashrate — cyberspace explicitly forgoes global state, so it cannot retarget. Costs deflate monotonically. This is not an oversight; it is the price of permissionlessness.

Mitigating factors: 85 bits per axis is enormous against any realistic multi-decade throughput gain; storage walls grow exponentially and resist linear hardware gains; and serial chain depth barely erodes at all.

**Proposal.** chalkline scales artifact requirements as a function of **current Bitcoin difficulty** — the only permissionless, globally-readable measure of deployed hardware. No new consensus required, and the chain is already being touched for seals.

**Symmetric, not ratcheting.** Requirements must fall as well as rise. An upward-only ratchet permanently privileges early artifacts and builds an aristocracy of the early. Aristocracy will form regardless; the design goal is that it remain *uprootable within the same universe*. Only then do the cycles have stakes. This is a one-line decision that is hard to reverse later.

**Difficulty coupling is a texture feature, not a fidelity compromise.** Bitcoin difficulty tracks SHA256 ASIC deployment and coin price, not general-purpose compute — it is a noisy, economically contaminated proxy. That divergence is not measurement error, it is *weather*. Retargets are discrete, so the world gets seasons with edges rather than drift. Cheap seasons are for making; expensive seasons are when what you made appreciates. Artifacts minted at a difficulty peak carry provably higher cost, so timestamps become vintage as well as provenance, verifiable by anyone from public data.

Someone will eventually file the proxy's imprecision as a bug. This paragraph is the answer.

## 6. Geography

Two independent sources of variation, which together give a blank coordinate space enough structure for an economy to have opinions:

- **Fixed geology** — region-derived stats (§4.2). Space varies by *where*.
- **Climate** — difficulty seasons (§5). Space varies by *when*.

DECK-1 hyperjumps add a third: Bitcoin blocks are projected into cyberspace by merkle root and serve as fast-travel anchors. This qualifies the "items must be carried" premise along the station network, and improves the design — **stations are cities, the space between them is wilderness.** Trade concentrates at anchors; remote regions stay remote because they are off-network.

It also supplies the structural answer to §5's aristocracy concern: a new block lands roughly every ten minutes, each one a new anchor. **The frontier regenerates continuously.** Early holders cannot own the map because the map keeps growing.

> **PARTIALLY RESOLVED.** The hyperspace DECK is `decks/DECK-0001-hyperspace.md`, "Hyperspace (Bitcoin block transit)," Status: Draft v3, last updated 2026-08-24. Its work is a merkle root over per-block seeded computation with `h_b = K_b + 6`, range `[6,22]` — a higher floor than base terrain's `[0,16]`, but parallelizable across blocks (§2). Station distance is `d(p,q) = max(LCA heights)`.
>
> **Still open:** absolute hyperjump cost relative to hop cost, which sets attunement parameters (§4.4). Not required for v1 scope (§7).
>
> **Note:** `cyberspace-cli` is not a reference implementation for this DECK — Appendix C names ONOSENDAI and `arkin0x/nth`. The CLI's hyperjump code predates even the v2 draft and should not be treated as authoritative for hyperspace behaviour.

## 7. Scope for v1

Ship the smallest thing everything else depends on:

- `spec/BUNDLE.md` — canonical serialization and verification algorithm
- a verifier: `verify(bundle) -> {valid, errors[], stats, chain_depth}`
- `tests/vectors/` — valid and invalid cases, each naming the spec section it violates

Documented here but **not implemented in v1**: transfer and seals (§4.3), attunement (§4.4), difficulty coupling (§5).

Nothing else can be tested until an artifact exists that a stranger can verify. Once one does, the rest is content.

## 8. Open questions

Tracked in `NOTES/protocol-facts.md` (`UNKNOWN` section) and `NOTES/upstream.md`. Both items that previously gated specification work are now resolved (§4.2, §6).

**Event kinds — complete registry** (`decks/README.md:41` confirms DECK-0001 is the sole registry entry):

| Kind | Purpose | Claimed by |
|---|---|---|
| `3333` | All movement events — base `spawn`/`hop`/`sidestep`, plus DECK-0001 `enter-hyperspace`/`hyperjump` | base + DECK-0001 |
| `33330` | Location-encrypted content | base |
| `321` | Block anchor events | DECK-0001 |

chalkline needs at minimum a placement kind and a completion kind. Note the precedent: DECK-0001 put new *movement* actions on the existing `3333` rather than minting a kind, distinguishing them by tag. chalkline placements are content bound to movement, not movement itself, so they warrant their own kind. Pick from a range clear of all three above and register it in chalkline's DECK.

**Two distinct terrain functions exist.** The base spec derives K from four low nibbles across four hashes keyed on aligned cell coordinates, range [0,16]. DECK-0001 derives `K_b` from two full digest bytes of a single hash keyed on a block hash, with `h_b = K_b + 6`, range [6,22]. chalkline uses the base-spec function exclusively, consistent with the hop/sidestep anchor restriction (§4.1). `BUNDLE.md` must name which function it means at every point it says "terrain."

**Encoding.** `int_to_bytes_be_min` (`CYBERSPACE_V2.md:503-515`) is the normative integer encoding; the `n == 0` branch yields `b"\x00"`, not the empty string. `region_bytes` carries no domain prefix, no length prefix, and no fixed-width padding. chalkline's canonical serialization MUST NOT assume otherwise.
