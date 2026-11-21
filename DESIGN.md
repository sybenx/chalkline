# chalkline — Design Intent

**Status:** Draft. Non-normative. This document states *what* chalkline is and *why*; `spec/BUNDLE.md` states *how*.

**Relationship to Cyberspace:** chalkline is an application layer built on Cyberspace Protocol v2. It proposes **no changes to the base protocol**. Every mechanism here uses a primitive v2 already provides. If a design goal cannot be met without amending v2, the goal is dropped, not the protocol.

---

## 1. The problem

The v2 spec lists sybil resistance as an accepted limitation: one person may hold many keypairs, each with an independent location and chain (§12.2). It also names key grinding as an acknowledged attack vector (§12.3, and already at `919494a`): spawning inside an aligned region of height h costs about 2^(3·(85−h)) key generations, so regions walled above roughly h75 can be spawn-camped by grinding. The spec frames grinding by region height, as a question of which regions stay isolated.

What the spec does not say is the economic version:

> **Key-grinding is teleportation.** A pubkey's coordinate is a direct function of the key. If grinding a keypair is a cheaper route to a target region than moving there, the distance metric is decorative. Grinding is a substitute for travel, not only a threat to remote regions.

This resolves more favorably than it first appears. Grinding cost is *exponential in coordinate precision* while movement cost is roughly polynomial in distance, so grinding buys a coarse neighborhood and never a precise location. There is a crossover radius below which movement is the only path. Above it, rebirth wins.

The spec's numbers place the crossover: grinding reaches regions walled above about h75, travel reaches about h50 (§12.3).

But it leaves a real gap the spec does not mention: **grinding work is unattestable.** A pubkey carries no evidence of how many candidates were discarded, and cannot: you can't prove you didn't get lucky on the first attempt. Movement work is published, chained, and verifiable. So cyberspace has a legible-work economy running alongside an illegible-work shortcut.

The consequence: **nothing yet makes identity persistence economically rational.** Absent value bound to a key, a fresh key is strictly better than a moved one.

## 2. The thesis

> **The one thing that cannot be transported to a fresh key is serial time.**

The temporal work axis makes each hop's seed derive from the previous movement event, so hop N cannot begin until hop N−1 is published. The chain is therefore *serially dependent*: its steps cannot be reordered, precomputed, or amortized. **Step count and order cannot be bought. Per-step latency can.**

- **Hops parallelize within a step.** Under hop proof v2 the residues are computed per prime and the re-roll search "MAY split the search across workers" (V2 §5.10, `:798`).
- **Sidesteps are plain SHA-256.** SHA-NI or a GPU is 50 to 10,000× faster than a CPU core, and ASICs apply (V2 §6.14, `:1253`, `:1267`; §12.3, `:2552`).
- **Handed roots cut a hop by about 19%** (V2 §6.12, `:1232`). The temporal axis stays per traveller.
- **Outsourcing is explicitly allowed.** HOSAKA sells hops and sidesteps computed for the customer's own chain position: "service, not resale" (`docs/what-can-be-sold.md` §5, `:144-158`; V2 §6.12, `:1234`).

So capital buys a faster step, not fewer steps.

**Exception — hyperjump.** The hyperjump seed is `sha256(HYPERSPACE_SEED_DOMAIN ‖ previous_event_id ‖ be64(b))` and its proof is a merkle root over per-block work. Every block's work shares one `previous_event_id`, so the whole set computes in parallel: hyperjump is a wide parallel workload behind a single serial link. Its cost therefore *does* compress with hardware. Hop, sidestep, and enter-hyperspace all retain strict serial dependency; hyperjump alone does not. See §4.1.

chalkline turns that property into a transferable artifact. An artifact is a bundle of coordinate placements, each bound to a link in its creator's movement chain. Its cost is denominated in serial steps, not wall-clock time, and it is therefore:

- **unforgeable**: reproducing it costs what making it cost
- **hardware-bounded, not hardware-proof**: hardware and paid services shorten each step, but no amount of either removes a step or reorders them
- **not purchasable as depth**: a rich attacker with a fresh key can buy faster steps but cannot retroactively manufacture the step count

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

Cost model: an N-placement artifact is a lower bound of N serial steps by one key. Automation does not shorten the count. Hardware and paid services shorten each step (§2), not the number of steps.

**Payload location:** placement data goes in tags, not `content`. The reference CLI writes the sidestep proof hash into `content` (`nostr_event.py:331`), and the base spec neither requires nor forbids it, so `content` is not reliably free across event kinds.

### 4.2 Stats

Artifact and item properties derive as a **pure function of the region root** — the deterministic Cantor-root identifier for the aligned subtree containing the placement.

Rationale: any per-item nonce (an event id, a counter) can be ground at microseconds per attempt, which would make every item in the world maximum-rolled by anyone running a script. Grinding an *event id* is nearly free; grinding a *coordinate* is brutally expensive. Putting all variance in the coordinate removes the cheap search entirely.

Consequences, all desirable:
- Regions have fixed geology. Some yield better material than others.
- Prospecting is a real activity, and long moves have purpose beyond tourism.
- Two parties derive identical stats without communicating (v2's deterministic-regions property).

> **RESOLVED.** Coordinates are Morton-interleaved, not sliced (V2 §2.2, `CYBERSPACE_V2.md:195-207`; `coords.py:91-98`): bit `3i+1`=Z, `3i+2`=Y, `3i+3`=X, bit `0`=plane. Landing in an aligned cube of side 2^(85−n) therefore requires matching the top `3n` bits of the pubkey, uniform across axes, no cheap axis to grind. Stat derivation may read the region root directly; no cross-axis mixing is required.

**Plane.** Bit 0 of the coordinate selects the plane, and since coord256 is the raw pubkey, the low bit of a key determines its spawn plane: one bit, trivially ground, roughly half of all keys in each. `BUNDLE.md` MUST state whether artifacts are plane-scoped, and whether a bundle's placements may span planes. Upstream has since settled the base side: a region and its twin (same X, Y, Z and height in the other plane) share one `region_n` and one key (V2 §2.4, `:241`; §7.2, `:1330`), and one bag holds both planes, each entry's `C` deciding its plane (V2 §7.6, Planes, `:1455-1468`). Plane-blind stats therefore match the base spec's own region identity.

**Contrast: DECK-0008 terroir.** DECK-0008 (Seeds) derives a seed's "terroir" from the terrain `K` and its nibbles at the planting coordinate (§4.4) and takes the opposite approach to choice: coordinate choice is allowed ("Soil shopping ... it is allowed", §12, `:889`), and selection is priced because every planting pays its DNA's NIP-13 price (`:886`). chalkline still keeps all variance in the coordinate because a placement has no per-draw price: an anchor is one hop or sidestep, and a free per-item nonce would be ground at microseconds per attempt. Choosing a coordinate is fine here too; it costs travel.

### 4.3 Transfer

Two problems, addressed separately.

**Presence.** Transfer requires both keys at the same region, established by **chain braiding**: each party's next hop takes its temporal seed from the other's latest event id. Both chains then cross-reference at a shared point, both parties paid fresh non-precomputable serial work, and neither could have staged it in advance.

**Braiding cannot be expressed on base v2 today.** A hop's and a sidestep's temporal seed is always the mover's own `e` previous event id (V2 §5.3, §6.7), and a client MUST sign only from its own live head (V2 §8.7.3, `:1927`). There is no field in which one party's hop can be seeded from the other party's event. Upstream's route to this is the pending **encounter primitive**, "a contact event in which two chains reference each other's heads" (V2 §8.11.7, `:2150`; `docs/territory-conflict-game-layer.md` §4.1, `:118-124`, work list item 10, `:153`, status pending). chalkline should wait for it rather than invent a braiding convention.

Honest limit: braiding proves *region work plus interactive liveness*, not travel. The spec is explicit that region preimages can be computed without maintaining a chain. Do not build rules that assume traversal was forced.

**Double-spend.** Without global state, nothing prevents a holder from signing conflicting transfers to two parties. Resolution is tiered by what the trade already touches:

- **Paid trades (sats for artifact):** commit the transfer to the spend of a specific UTXO — a **single-use seal**. The chain never learns what an artifact is; it enforces only that the seal closes once. Because the buyer is already making a payment, this costs *zero additional on-chain footprint*: a taproot tweak on a transaction that was happening anyway. Not an extra step — a free rider on the payment.
- **Barter (artifact for artifact):** unsettled. Detectable via chain forks, not preventable. Documented as trust-based. This is a small slice of any real economy and is where scams are expected anyway.

**Fork risk to every artifact.** Under V2 §8.7.3 rule 4 (`:1914`) a forked chain is dead from the spawn, whatever came before the fork. A creator who later forks their own chain, by accident from two devices or on purpose, voids every placement anchored to it, including artifacts already sold. The seller can do this after the sale at the cost of one signature. **Open problem:** transfer must say whether an artifact survives its creator's chain dying, and if not, how a buyer is protected (see `spec/BUNDLE.md` §9.13).

**Design note:** a forked chain is permanent public evidence, so a chain-native fidelity bond exists as a fallback — a forked key's history becomes worthless, and rebuilding chain depth is unparallelizable. This bounds low-value fraud but fails to the exit scam, which is why seals exist for anything worth stealing.

### 4.4 Attunement

A received artifact is **inert until the new holder pays serial temporal work** scaled to the artifact's power. The price is stated in serial steps (a count of the holder's own hops or sidesteps), not in wall-clock time, since per-step latency can be bought (§2).

This is not a smurf patch; it is the mechanism that makes the whole economy sybil-priced. Consider the courier swarm: expected minimum distance to a target across N random keys falls as roughly N^(−1/3), so a million ground keys — seconds of keygen — put one far closer than a single spawn would. Chain them in a relay, each doing one short leg.

Decomposition invariance does **not** stop this. It forbids one key from subdividing *its own* hop; a relay of distinct keys each making a genuinely short move from its own spawn walks straight past the invariant. It holds per-key, and the swarm is not one key.

Attunement taxes the relay in the dimension it cannot parallelize: a ten-link chain costs ten sequential attunements, each a fixed number of serial steps. Trade stays fluid, but arming a fresh smurf costs the one thing a rich main cannot wire over.

### 4.5 Copies and priority

A published bundle is verifiable, therefore readable, therefore reproducible — anyone can re-walk the path and produce a bit-identical artifact. This is acceptable and arguably correct: forging costs what making cost, like counterfeiting a painting by painting it.

What is scarce is the **first instance**. Priority comes from external timestamp anchoring (OpenTimestamps or equivalent), since Nostr timestamps are self-asserted and prove sequence but not duration. **This conflicts with upstream game rules.** Game-mechanic DECKs must be verifiable from a bounded set of events plus, if they need a clock, Bitcoin block headers, and `created_at` MUST NOT decide any game verdict (`decks/README.md:49-53`). Upstream's domains analysis rejects first-seen priority outright: OpenTimestamps "reintroduce[s] a trusted or paid ordering service", which makes priority "fairness by agreement" (`docs/analysis/deck-domains-ground-up-analysis.md:91`). Priority therefore stays an **option only if it is expressed through Bitcoin headers** (for example a commitment whose inclusion in a block a reader checks against a header per DECK-0001 §2.5), not through an attestation service. DECK-0008 floats the same idea for seeds: "Bitcoin timestamps (OpenTimestamps, or the block anchors of DECK-0001) would close it at a cost" (§12, `:887`) and a Bitcoin timestamp per commit (§13 item 4, `:906`). Neither is specified upstream yet.

Increasingly complex artifacts are increasingly unreplicated in practice. Beyond providing verifiable priority, replication is a market problem, not a protocol problem.

## 5. Difficulty coupling

**Problem.** Hardware improves; protocol costs are fixed in absolute terms. Bitcoin retargets because it has global consensus about hashrate — cyberspace explicitly forgoes global state, so it cannot retarget. Costs deflate monotonically. This is not an oversight; it is the price of permissionlessness.

**Upstream position.** The base spec has no difficulty adjustment by design: "Cantor tree difficulty is fixed by the mathematics ... no adjustment mechanism, no governance, and no parameters to tune" (V2 §13.3, `:2579-2583`; §9.3, `:2324`, which adds that difficulty "migrates upward through extension mechanisms (DECKs) over decades"). chalkline therefore treats coupling as **optional game-layer texture, not calibration**: it never changes what a proof costs, only what an artifact asks for. The Bitcoin data it reads should come from verified block headers (DECK-0001 §2.5, `:153-197`: the `bits` field, taken only from a header that hashes to an accepted `H`), which keeps it inside the game-mechanics rule that the only clock is Bitcoin headers.

Mitigating factors: 85 bits per axis is enormous against any realistic multi-decade throughput gain; storage walls grow exponentially and resist linear hardware gains; and serial chain depth barely erodes at all.

**Proposal.** chalkline scales artifact requirements as a function of **current Bitcoin difficulty**, read from DECK-0001 header `bits`: the only permissionless, globally-readable measure of deployed hardware. No new consensus required, and the chain is already being touched for seals.

**Symmetric, not ratcheting.** Requirements must fall as well as rise. An upward-only ratchet permanently privileges early artifacts and builds an aristocracy of the early. Aristocracy will form regardless; the design goal is that it remain *uprootable within the same universe*. Only then do the cycles have stakes. This is a one-line decision that is hard to reverse later.

**Difficulty coupling is a texture feature, not a fidelity compromise.** Bitcoin difficulty tracks SHA256 ASIC deployment and coin price, not general-purpose compute — it is a noisy, economically contaminated proxy. That divergence is not measurement error, it is *weather*. Retargets are discrete, so the world gets seasons with edges rather than drift. Cheap seasons are for making; expensive seasons are when what you made appreciates. Artifacts minted at a difficulty peak carry provably higher cost, so timestamps become vintage as well as provenance, verifiable by anyone from public data.

Someone will eventually file the proxy's imprecision as a bug. This paragraph is the answer.

## 6. Geography

Two independent sources of variation, which together give a blank coordinate space enough structure for an economy to have opinions:

- **Fixed geology** — region-derived stats (§4.2). Space varies by *where*.
- **Climate** — difficulty seasons (§5). Space varies by *when*.

DECK-1 hyperjumps add a third: Bitcoin blocks are projected into cyberspace by merkle root and serve as fast-travel anchors. This qualifies the "items must be carried" premise along the station network, and improves the design — **stations are cities, the space between them is wilderness.** Trade concentrates at anchors; remote regions stay remote because they are off-network.

It also supplies the structural answer to §5's aristocracy concern: a new block lands roughly every ten minutes, each one a new anchor. **The frontier regenerates continuously.** Early holders cannot own the map because the map keeps growing.

> **PARTIALLY RESOLVED.** The hyperspace DECK is `decks/DECK-0001-hyperspace.md`, "Hyperspace (Bitcoin block transit)," Status: Draft v3, last updated 2026-11-10, and now **mandatory** for every verifier (V2 §8.9, `:1980-1982`). Its work is a merkle root over per-block seeded computation with `h_b = K_b + 6`, range `[6,22]`, a higher floor than base terrain's `[0,16]`, but parallelizable across blocks (§2). Station distance is `d(p,q) = max(LCA heights)`.
>
> **Still open:** absolute hyperjump cost relative to hop cost, which sets attunement parameters (§4.4). Not required for v1 scope (§7).
>
> **Note:** DECK-0001 Appendix C names ONOSENDAI, `arkin0x/nth` and the DECK's own reference scripts, not `cyberspace-cli`. The CLI now implements v3 stop geometry and ride verification but does not publish rides (`NOTES/protocol-facts.md` §9).

## 7. Scope for v1

Ship the smallest thing everything else depends on:

- `spec/BUNDLE.md` — canonical serialization and verification algorithm
- a verifier: `verify(bundle) -> {valid, errors[], stats, chain_depth}`
- `tests/vectors/` — valid and invalid cases, each naming the spec section it violates

Documented here but **not implemented in v1**: transfer and seals (§4.3), attunement (§4.4), difficulty coupling (§5).

Nothing else can be tested until an artifact exists that a stranger can verify. Once one does, the rest is content.

## 8. Open questions

Tracked in `NOTES/protocol-facts.md` (`UNKNOWN` section) and `NOTES/upstream.md`. Both items that previously gated specification work are now resolved (§4.2, §6).

**Event kinds.** The registry has grown well past three kinds; the full list with citations is in `NOTES/protocol-facts.md` §2.1. Kinds to avoid: base `3333`, `33330`, `33335`, `11333`, reserved `33332`; DECK `321`, `33331`, `3330`, `3340`, `3341`, `33340`, `23330`, `23333`, `1111`, `1063`, `3350`, `3351`, `3352`; DECK-0002 uses the ephemeral range `20000`-`29999`; and, taken in practice by other applications, `3334`, `33334`, `3338` and `3302`-`3313` (DECK-0008 `:96`). The `3330`-`3352` neighbourhood is crowded.

chalkline needs at minimum a placement kind and a completion kind, unless it adopts the proposal in §9 and expresses artifacts as existing DECK items. Note the precedent: DECK-0001 put new *movement* actions on the existing `3333` rather than minting a kind, distinguishing them by tag. chalkline placements are content bound to movement, not movement itself, so if they get kinds, they warrant their own. Pick from a range clear of everything above and register it in chalkline's DECK.

**Two distinct terrain functions exist.** The base spec derives K from four low nibbles across four hashes keyed on aligned cell coordinates, range [0,16]. DECK-0001 derives `K_b` from two full digest bytes of a single hash keyed on a block hash, with `h_b = K_b + 6`, range [6,22]. chalkline uses the base-spec function exclusively, consistent with the hop/sidestep anchor restriction (§4.1). `BUNDLE.md` must name which function it means at every point it says "terrain."

**Encoding.** `int_to_bytes_be_min` (V2 §5.5, `CYBERSPACE_V2.md:590-602`) is the normative integer encoding; the `n == 0` branch yields `b"\x00"`, not the empty string. `region_bytes` carries no domain prefix, no length prefix, and no fixed-width padding. chalkline's canonical serialization MUST NOT assume otherwise.

## 9. Relationship to upstream DECKs

**The niche.** Upstream has items but deliberately no scarcity. DECK-0004: "There is no scarcity in this DECK and no revocation. Games that need either build it on the chain later" (`decks/DECK-0004-items-and-chests.md:669`), and taking from a chest "removes nothing: the chest still holds what was taken" (`:235`). Upstream placement also costs no movement: "Talking, hiding, finding and placing do not move the chain ... Only movement is serialized" (`docs/agents.md:58`). chalkline is the missing piece both point at: things whose cost is bound to serial movement work on one chain.

> **PROPOSED, awaiting a decision.** Express chalkline artifacts on upstream formats instead of new kinds:
> - each placement is a DECK-0004 item (`kind 3340`) or a DECK-0003 SNO object in a bag, carrying a chalkline tag that binds it to the id of one `hop` or `sidestep` event on the creator's chain;
> - the anchor rule (§4.1), stat derivation (§4.2) and depth accounting stay chalkline's;
> - eventually, submit chalkline upstream as a game DECK under the game-mechanics rules of `decks/README.md:41-55` (Bitcoin headers as the only clock, no verdict on `created_at`, effects as a fixed point over the reference graph).
>
> Not decided: which DECK format, the tag name, and whether `spec/BUNDLE.md`'s own placement and completion kinds are dropped. Until then, `spec/BUNDLE.md` keeps its kinds unassigned (§9.1 there).

