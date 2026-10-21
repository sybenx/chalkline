# chalkline — Artifact Bundle Specification

**Version:** `chalkline-bundle-2026-08-28-draft1`
**Status:** Draft. **Normative.**
**Layer:** Application layer over Cyberspace Protocol v2 (`CYBERSPACE_V2.md`, spec version string `2026-03-16-h34-corrected`). This document proposes **no** changes to the base protocol.
**Design intent:** `../DESIGN.md` (non-normative). Where this document and `DESIGN.md` disagree, this document governs implementations and `DESIGN.md` governs revisions to this document.
**Protocol ground truth:** `../NOTES/protocol-facts.md`, compiled against `arkin0x/cyberspace@919494a` and `arkin0x/cyberspace-cli@2dff988`. Every base-protocol citation below (`§n.n` with no document named means `CYBERSPACE_V2.md`) is transcribed there with file and line references.

---

## 1. Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174, and only when they appear in all capitals.

Two conformance classes are defined:

- A **producer** constructs and publishes bundles.
- A **verifier** consumes a bundle and returns `verify(bundle) -> {valid, errors[], stats, chain_depth}` per §7.

An implementation MUST NOT report a bundle as `valid` unless every **MUST** in §§4–7 is satisfied. A verifier MUST NOT reject a bundle solely for violating a **SHOULD**; it SHOULD surface such violations as non-fatal diagnostics.

Where this document says a value is **unassigned** or defers to §9 (Open Questions), an implementation MUST treat that value as a deployment parameter and MUST NOT claim interoperable conformance across differing choices of it.

### 1.1 Scope

In scope for v1: bundle structure, canonical serialization, the verification algorithm, and stat derivation.

Explicitly **out of scope** for v1, documented in `DESIGN.md` but not specified here: transfer and single-use seals (`DESIGN.md` §4.3), attunement (§4.4), and difficulty coupling (§5). Nothing in this document may be read as specifying, permitting, or constraining those mechanisms.

---

## 2. Terminology

- **coord256** — a 256-bit Cyberspace coordinate: three 85-bit unsigned axes X, Y, Z interleaved Morton-style plus a 1-bit plane selector at bit 0 (§2.1, §2.2). Bit `3i+1` is Z bit `i`, `3i+2` is Y bit `i`, `3i+3` is X bit `i`.
- **movement event** — a `kind 3333` event. Base actions: `spawn`, `hop`, `sidestep` (§8.8). DECK-0001 adds `enter-hyperspace`, `hyperjump`.
- **serially-seeded event** — a movement event whose temporal work seed derives from `previous_event_id` and whose work therefore cannot begin until the preceding event exists: `hop` (§5.3), `sidestep` (§6.7), `enter-hyperspace` (DECK-0001 §3.2).
- **anchor** — the movement event to which a placement is bound.
- **placement** — a single `KIND_PLACEMENT` event: one coordinate plus one payload, bound to one anchor.
- **bundle** — the complete evidence set for one artifact: a spawn event, a contiguous run of movement events, a set of placements, and one completion event, all under a single pubkey.
- **creator** — the pubkey that signs every event in the bundle.
- **region root** — the aligned-subtree spatial region integer `region_n` at a declared height, per §4.6/§4.7 and the derivation of §7.4. See §6.
- **base terrain function** — the coordinate-keyed terrain height `K` of §5.2, domain `CYBERSPACE_TERRAIN_K_V2`, range `[0, 16]`.
- **line terrain function** — DECK-0001's block-keyed `K_b`, domain `CYBERSPACE_HYPERSPACE_TERRAIN_V1`, `h_b = K_b + 6`, range `[6, 22]`.

> **Every use of the word "terrain" in this document means the base terrain function.** chalkline uses the base terrain function exclusively. The line terrain function is never invoked by a chalkline verifier, because hyperjump events can never anchor a placement (§4.4). Conflating the two is a consensus failure: they differ in domain string, in digest-byte extraction, and in range (`NOTES/protocol-facts.md` §4.1 vs §4.5).

Notation: `‖` is byte concatenation. `π(a,b) = (a+b)(a+b+1)/2 + b` is the Cantor pairing function (§4.2). `be32(n)` / `be64(n)` are `n` as 4 / 8 big-endian bytes. `byte(n)` is `n` as one unsigned byte. `sha256` is SHA-256 producing 32 raw bytes.

---

## 3. Constants

```
CHALKLINE_BUNDLE_VERSION   = "1"
CHALKLINE_STAT_DOMAIN      = b"CHALKLINE_STAT_V1"
CHALKLINE_STAT_EXPAND      = b"CHALKLINE_STAT_EXPAND_V1"
CHALKLINE_BUNDLE_LEAF      = b"CHALKLINE_BUNDLE_LEAF_V1"
PAD_LEAF                   = 32 zero bytes
KIND_PLACEMENT             = <unassigned — see §9.1>
KIND_COMPLETION            = <unassigned — see §9.1>
```

Any change to a preimage format defined in this document MUST bump the corresponding domain string, following the base spec's own rule for `TERRAIN_DOMAIN_V2` (§5.2). Domain strings are ASCII bytes with no terminator and no length prefix.

`KIND_PLACEMENT` and `KIND_COMPLETION` MUST NOT be `3333`, `33330`, or `321` — the complete set of kinds claimed by the base spec and DECK-0001 as of `919494a` (`decks/README.md:41` confirms DECK-0001 is the sole registry entry). They MUST be registered in chalkline's own DECK before publication to a shared relay set.

---

## 4. Bundle structure

### 4.1 Composition

A bundle is a set of Nostr events. It MUST consist of, and MUST NOT contain anything other than:

1. exactly one **spawn event** (`kind 3333`, `A=spawn`);
2. one or more **movement events** (`kind 3333`, `A ∈ {hop, sidestep, enter-hyperspace, hyperjump}`) forming a contiguous chain rooted at the spawn event;
3. one or more **placement events** (`KIND_PLACEMENT`);
4. exactly one **completion event** (`KIND_COMPLETION`).

Every event in the bundle MUST carry the same `pubkey`. **Artifacts are scoped by signer, not by region** (`DESIGN.md` §4.1); a verifier MUST reject any bundle containing events from more than one pubkey.

A bundle MUST be self-contained: a verifier MUST be able to complete §7 using only the events in the bundle, with no relay access and no network dependency. A verifier MUST NOT fetch events to complete verification, and MUST NOT treat a bundle as valid on the basis of an event it was not given.

### 4.2 The movement chain span

Let `E_spawn` be the spawn event and `E_term` the anchor of the highest-indexed placement. The bundle's movement events MUST form the exact, gapless, linear chain from `E_spawn` to `E_term` inclusive:

- Each movement event other than the spawn MUST carry `["e","<spawn_event_id>","","genesis"]` and `["e","<previous_event_id>","","previous"]` (§8.4, §8.5).
- Each `previous_event_id` MUST resolve to another event in the bundle, and each movement event other than `E_term` MUST be referenced as `previous` by exactly one other movement event in the bundle.
- The bundle MUST NOT contain movement events after `E_term`.
- No two movement events in the bundle may share a `previous_event_id`. Two events sharing one are a fork; a verifier MUST reject such a bundle. (This rule is stated explicitly only in DECK-0001 §8; the base spec's §12.1 is marked non-normative and says only that forking is detectable. See §9.9.)

`hyperjump` events MAY appear within the span. They are travel, not drawing (`DESIGN.md` §6: *hyperjump to reach a region, hops to draw in it*). They MUST be verified for chain validity like any other event in the span, and they MUST NOT contribute to `chain_depth` (§7.10).

### 4.3 Placement event

`kind = KIND_PLACEMENT`. Tags MUST be emitted in exactly this order:

| # | Tag | Value | Notes |
|---|---|---|---|
| 1 | `v` | `"1"` | `CHALKLINE_BUNDLE_VERSION` |
| 2 | `e` | `<spawn_event_id>`, `""`, `"genesis"` | 4-element tag, NIP-01 marker form |
| 3 | `e` | `<anchor_event_id>`, `""`, `"anchor"` | 4-element tag |
| 4 | `C` | `<coord_hex>` | 64 lowercase hex; MUST equal the anchor's `C` tag |
| 5 | `i` | `<index>` | base-10, no leading zeros, no `+` |
| 6 | `payload` | `<codec>`, `<value>` | 3-element tag; see §4.6 |
| 7 | `X` | `<sx>` | sector tags per §10, derived from `C` |
| 8 | `Y` | `<sy>` | |
| 9 | `Z` | `<sz>` | |
| 10 | `S` | `"<sx>-<sy>-<sz>"` | |

`content` MUST be the empty string `""`. Placement data goes in tags because `content` is not reliably free across kinds — the reference CLI already writes the proof hash into `content` on sidestep events (`nostr_event.py:202`), and §8.5 neither requires nor forbids it (`DESIGN.md` §4.1).

A placement event MUST carry no tags beyond those above. A verifier MUST reject a placement with an unrecognised tag name; this preserves bit-identical reproducibility (§8.3) by forbidding producer-private tag extensions from silently altering the event id.

### 4.4 Anchor eligibility

> **A placement's anchor MUST be a `hop` or `sidestep` event.**

- A `hyperjump` event MUST NOT anchor a placement. Its per-block work shares one `previous_event_id` across every block in the ride and therefore computes in parallel (DECK-0001 §5.3; `DESIGN.md` §2). Allowing it to anchor would let a funded actor purchase artifact depth with parallel hardware, which collapses the serial-time thesis the format exists to express.
- A `spawn` event MUST NOT anchor a placement. It carries no proof and no work (§8.8).
- An `enter-hyperspace` event MUST NOT anchor a placement. It is serially seeded, but its spatial slot is the literal `0` (`enter_n = π(0, cantor_t)`, DECK-0001 §3.2) and its `C` tag MUST equal its `c` tag — it carries no new location, so there is nothing to place. See §9.5.
- Each anchor id MUST be referenced by **at most one** placement in the bundle. One movement event yields one placement. This follows from the cost model in `DESIGN.md` §4.1 — *an N-placement artifact is a lower bound on serial wall-clock time by one key* — which is false if a single hop can carry N placements.
- Placements MUST NOT anchor to movement events outside the bundle's span, and every anchor MUST be present in the bundle.

Anchors need not be contiguous: the chain MAY contain hops and sidesteps that anchor nothing, and hyperjumps that cannot. See §9.6.

### 4.5 Completion event

`kind = KIND_COMPLETION`. Tags MUST be emitted in exactly this order:

| # | Tag | Value | Notes |
|---|---|---|---|
| 1 | `v` | `"1"` | |
| 2 | `e` | `<spawn_event_id>`, `""`, `"genesis"` | |
| 3 | `e` | `<terminal_anchor_event_id>`, `""`, `"terminal"` | anchor of the highest-indexed placement |
| 4 | `n` | `<placement_count>` | base-10 |
| 5 | `root` | `<bundle_root_hex>` | 64 lowercase hex; §5.5 |
| 6 | `h` | `<stat_height>` | base-10, `1 ≤ h ≤ 84`; §6.2 |

`content` MUST be the empty string.

The completion event claims no coordinate and therefore MUST NOT carry `X`, `Y`, `Z`, or `S` sector tags — §10 requires them only of events that claim a coordinate. It is not a movement event, does not extend the movement chain, and MUST NOT be referenced as `previous` by any `kind 3333` event.

The completion event's id **is** the bundle id. It commits transitively to every placement (via `root`), to the creator (via `pubkey` and genesis), to the terminal chain position, and to the stat height. External timestamp anchoring for priority (`DESIGN.md` §4.5) SHOULD target this id and nothing else.

### 4.6 Payload

The `payload` tag is `["payload", "<codec>", "<value>"]`. `<codec>` MUST be a lowercase ASCII token matching `[a-z0-9-]{1,32}`. `<value>` MUST be a string whose interpretation is defined entirely by `<codec>`.

Verifiers MUST treat the payload as opaque bytes for all purposes in §7. Payload content MUST NOT affect `valid`, `stats`, or `chain_depth`. The codec registry is unassigned; see §9.8.

Producers SHOULD keep payloads small. Relays impose event size limits that this document does not and cannot specify.

### 4.7 Signatures

Every event in the bundle MUST carry a valid Schnorr `sig` over its `id`, per §8.2.

§8.2 notes that some prototypes leave `sig` blank for local-only chains, and the reference CLI always does so (`nostr_event.py:47-71`, `"sig": ""`; recorded as its one known conformance gap at `docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md:97-98`). **chalkline grants no such exemption.** A bundle containing an event with an empty or invalid `sig` MUST be reported invalid. A bundle assembled directly from `cyberspace-cli` output will fail this check until the events are signed.

---

## 5. Canonical serialization

### 5.1 Integer encoding

Two integer encodings coexist in the base protocol and **which one applies is per-step**. Producers and verifiers MUST NOT substitute one for the other.

**(a) `int_to_bytes_be_min`** — normative at §5.5 (`CYBERSPACE_V2.md:503-515`):

```python
def int_to_bytes_be_min(n: int) -> bytes:
    if n < 0:
        raise ValueError("expected non-negative")
    if n == 0:
        return b"\x00"
    return n.to_bytes((n.bit_length() + 7) // 8, "big")
```

The `n == 0` branch returns the single byte `0x00`, **not** the empty string. This encoding carries no domain prefix, no length prefix, and no fixed-width padding. chalkline's canonical serialization MUST NOT assume otherwise.

**(b) Fixed-width 32-byte big-endian** — used for coordinates inside the terrain preimage only (§5.2 step 1.3; `coord.to_bytes(32, "big")` at `terrain.py:76`).

`region_bytes = int_to_bytes_be_min(region_n)` (§7.2, `CYBERSPACE_V2.md:833-845`). This is the exact byte value the region identifier is computed from, and chalkline's stat derivation (§6) takes it verbatim.

### 5.2 Text encodings

| Kind of value | Canonical form |
|---|---|
| Hash / digest / event id / proof | Lowercase hexadecimal, exactly 64 characters, no `0x` prefix |
| coord256 (`C`, `c` tags) | Lowercase hexadecimal, exactly 64 characters, no `0x` prefix |
| Sector components (`X`, `Y`, `Z`) | Base-10, no leading `+`, no leading zeros except `"0"` itself (§10) |
| Sector string (`S`) | Exactly `"<sx>-<sy>-<sz>"` (§10) |
| Placement index (`i`), count (`n`), height (`h`) | Base-10, no leading `+`, no leading zeros except `"0"` |
| Domain strings | ASCII bytes, no terminator, no length prefix |

A verifier MUST reject a bundle containing an uppercase hex digit, a `0x` prefix, a hex string of the wrong length, or a base-10 integer with a leading zero or sign, in any of the fields above. These are consensus-critical: they enter event ids and therefore change hashes.

### 5.3 Event id

Computed per NIP-01, unchanged from §8.2:

```
id = sha256(json.dumps(
        [0, pubkey, created_at, kind, tags, content],
        separators=(",", ":"), ensure_ascii=False
    ).encode("utf-8"))
```

Field order in this array is fixed by NIP-01 and MUST NOT be varied. Tag order **within** `tags` is part of the preimage; this is why §4.3 and §4.5 fix tag order rather than merely fixing tag membership. A verifier MUST recompute every event's id and MUST reject any event whose stated `id` differs.

### 5.4 Serialization order of the bundle

For transport and archival, a bundle SHOULD be serialized as a JSON array of its events in this order:

1. the spawn event;
2. the movement events, in chain order (`previous` linkage), ascending;
3. the placement events, in ascending `i`;
4. the completion event.

This order is **advisory and does not enter any hash.** A verifier MUST accept a bundle whose events arrive in any order, and MUST reconstruct order from `previous` linkage and `i` values rather than from array position. Nothing in verification may depend on transport order.

### 5.5 Bundle root

`root` is a Merkle root over the placement event ids. Given `N` placements with ids `id_0 .. id_{N-1}` ordered by ascending `i`:

1. `leaf_i = sha256(CHALKLINE_BUNDLE_LEAF ‖ be64(i) ‖ id_i_bytes)`, where `id_i_bytes` is the 32 raw bytes of the placement's event id.
2. Append `PAD_LEAF` (32 zero bytes) until the leaf count is a power of two. A single leaf requires no padding.
3. Build bottom-up with `parent = sha256(left ‖ right)`, exactly as §6.4 step 5.
4. `root` is the surviving 32-byte value, lowercase hex.

For `N = 1`, `root = leaf_0`. Leaves are domain-separated and index-bound, so an internal node can never be reinterpreted as a leaf and a placement cannot be relocated within the tree without changing the root.

### 5.6 Determinism

All arithmetic in this document is exact integer arithmetic. No step defined here uses floating point or the decimal context of §9.5 — that context is required only by DECK-0001 landfall derivation, which chalkline never invokes. Implementations MUST NOT introduce rounding, and MUST NOT rely on platform integer width; `region_n` and Cantor intermediates are unbounded.

---

## 6. Stat derivation from the region root

Artifact and item properties derive as a **pure function of the region root** (`DESIGN.md` §4.2). No event id, counter, timestamp, index, payload, or pubkey may enter the derivation. This is the whole point: grinding an event id is nearly free, grinding a coordinate is exponentially expensive, so all variance lives in the coordinate and there is no cheap search.

Because coordinates are Morton-interleaved (§2.2), landing in an aligned cube of side `2^(85−n)` requires matching the top `3n` bits of the pubkey — uniform across axes, with no cheap axis to grind. Stat derivation MAY therefore read the region root directly; no cross-axis mixing is required.

### 6.1 Which region root

> The stat region root is the aligned subtree **containing the placement coordinate**, computed per the §7.4 discovery derivation. It is **not** the `region_n` from the anchor's hop proof (§4.7 with LCA height).

This distinction is load-bearing and easy to get wrong. A hop's proof `region_n` is computed over the aligned subtree covering *both* endpoints, at `h = find_lca_height(v1, v2)` — so it depends on where the mover came from. Two placements at the identical coordinate, reached from different origins, would yield different stats. A verifier MUST use the destination-only derivation below.

### 6.2 Stat height

The stat height `h` is declared in the completion event's `h` tag and MUST satisfy `1 ≤ h ≤ 84`. It applies to every placement in the bundle. Whether `h` is a free parameter, a fixed protocol constant, or a per-placement value is unresolved; see §9.3.

### 6.3 Derivation

Given a placement's `C` tag, decode it to `(x, y, z, plane)` per §2.3 (`coord_to_xyz`). Then:

1. `bx = (x >> h) << h`; `by = (y >> h) << h`; `bz = (z >> h) << h`.
2. `rx = compute_subtree_cantor(bx, h)`; `ry`, `rz` likewise (§4.6).
3. `region_n = π(π(rx, ry), rz)` (§4.7).
4. `region_bytes = int_to_bytes_be_min(region_n)` (§5.1(a), §7.2).
5. `stat_seed = sha256(CHALKLINE_STAT_DOMAIN ‖ byte(h) ‖ region_bytes)`.
6. For `j = 0, 1, 2, ...`: `block_j = sha256(CHALKLINE_STAT_EXPAND ‖ stat_seed ‖ be32(j))`. The **stat stream** is `block_0 ‖ block_1 ‖ ...`, consumed left to right, big-endian.

Steps 1–4 are exactly §7.4's per-height discovery derivation and reuse the base spec's region identifier without modification. Steps 5–6 are chalkline's and are the only new preimages this document introduces for stats.

### 6.4 Domain separation is mandatory

> **A producer or verifier MUST NOT use `sha256(region_bytes)` as a stat seed, stat value, or any published chalkline value.**

`sha256(region_bytes)` **is** `location_decryption_key` for that region (§7.2, `CYBERSPACE_V2.md:833-845`). Publishing it, or any value from which it is recoverable, would hand out the decryption key for every `kind 33330` location-encrypted content event in that region. The base spec's security argument — that seeing `lookup_id` does not permit deriving `location_decryption_key` without the region preimage (§7.2, `:841`) — depends on that key not being republished by another application layer. Step 5's domain prefix is what keeps chalkline outside that construction.

Equally, `sha256(sha256(region_bytes))` is `lookup_id` and MUST NOT be used as a stat value.

### 6.5 Properties a verifier may rely on

- **Determinism without communication.** Two parties compute identical stats from the coordinate alone (§7.2's stable-region property), with no chain, no relay, and no interaction.
- **Not temporally bound.** The temporal axis of §5 is intentionally excluded from region identifiers; they do not change when someone moves through (§7.2 Note, `:843`). Stats are stable for the life of the coordinate space.
- **Regional, not per-placement.** Every placement inside the same aligned subtree at height `h` has the **same** stats. Regions have fixed geology; some yield better material than others.
- **Plane-blind.** `region_n` combines `rx, ry, rz` only. The plane bit is not an input at any step of §6.3. Two placements at identical `(x, y, z)` in opposite planes therefore derive **identical** stats. Contrast the base terrain function `K`, which *does* include the plane bit in its preimage (§5.2 step 1.3). This asymmetry is a property of the base spec, not a chalkline choice; whether chalkline should compensate is §9.2.

### 6.6 Output shape

`verify()` MUST return, for each placement, the `stat_seed` (64 lowercase hex) and the region root `region_n` it was derived from. The mapping from the stat stream onto named attributes, ranges, and rarity curves is **unassigned**; see §9.4. A verifier that does not implement a stat schema MUST still return seeds and region roots, and MUST NOT report `valid = false` on that account.

---

## 7. Verification algorithm

`verify(bundle) -> {valid, errors[], stats, chain_depth}`

A verifier MUST perform the steps below in order. A verifier SHOULD continue after a failure and accumulate into `errors[]` rather than aborting, so that a single call reports all defects; it MUST set `valid = false` if `errors[]` is non-empty. Every entry in `errors[]` SHOULD name the section of this document violated, so that `tests/vectors/` cases can assert on it.

**Step 1 — Intake and partition.**
Parse the bundle. Partition events by `kind` into spawn/movement (`3333`), placements (`KIND_PLACEMENT`), and completion (`KIND_COMPLETION`). Reject if any event has another kind, if there is not exactly one spawn event, if there is not exactly one completion event, if there are zero placements, or if any two events share an `id` (§4.1).

**Step 2 — Single-signer check.**
Reject unless every event carries an identical `pubkey` (§4.1).

**Step 3 — Encoding and shape check.**
For every event, reject on any violation of §5.2: wrong hex case, `0x` prefix, wrong hex length, malformed base-10 integer. For placements and the completion event, reject unless the tag list matches §4.3 / §4.5 exactly — same names, same order, same arity, no extras. Reject unless `content` is `""` on both (§4.3, §4.5).

**Step 4 — Event id recomputation.**
For every event, recompute `id` per §5.3 and reject on mismatch. All later steps MUST use the recomputed value.

**Step 5 — Signature verification.**
For every event, verify the Schnorr `sig` over the recomputed `id` against `pubkey` (§8.2). Reject on empty, malformed, or invalid signature (§4.7).

**Step 6 — Spawn validation.**
Reject unless the spawn event has `["A","spawn"]` and its `C` tag equals its own `pubkey` (§3.2, §8.3). Reject unless its sector tags match the §10 derivation from `C`.

**Step 7 — Chain reconstruction.**
Walk from the spawn event forward by `previous` linkage. Reject if any movement event's `previous` id is absent from the bundle; if any movement event's `genesis` id is not the spawn id; if two movement events share a `previous` id (fork, §4.2); if the walk does not visit every movement event in the bundle exactly once (orphans or cycles); or if any movement event's `c` tag does not equal its predecessor's `C` tag. Reject if the chain does not terminate at the anchor of the highest-indexed placement (§4.2).

**Step 8 — Movement proof verification.**
For every `hop` and `sidestep` in the span, verify the proof against the base protocol:

- 8a. Compute base terrain `K` at the event's destination `C`, **including the plane bit**: for each `bits` in `[3, 7, 9, 11]` in order, `digest = sha256(TERRAIN_DOMAIN_V2 ‖ byte(bits) ‖ cell_coord.to_bytes(32,"big"))` where `cell_coord = xyz_to_coord((x2>>bits)<<bits, (y2>>bits)<<bits, (z2>>bits)<<bits, plane)`; `nibble_i = digest[0] & 0x0F`; `word16 = (n0<<12)|(n1<<8)|(n2<<4)|n3`; `K = popcount(word16)`, which MUST lie in `[0,16]` (§5.2).
- 8b. Compute the temporal seed: `t = int(previous_event_id_bytes) mod 2^85`; `t_base = (t >> K) << K`; `cantor_t = compute_subtree_cantor(t_base, K)` (§5.3; identical for sidesteps per §6.7).
- 8c. For a `hop`: compute per-axis roots at `h = find_lca_height(v1, v2)` and `region_n = π(π(cantor_x, cantor_y), cantor_z)` (§4.6, §4.7); then `hop_n = π(region_n, cantor_t)`; `proof_hash = sha256(sha256(int_to_bytes_be_min(hop_n)))` (§5.4, §5.6). Reject unless it equals the `proof` tag.
- 8d. For a `sidestep`: verify the three axis Merkle roots in `mr` and the inclusion paths in `mp` against `hx`/`hy`/`hz` (§6.3–§6.5, §6.10); compute `region_m = π(π(mx, my), mz)` from the roots as big-endian integers (§6.6); then `sidestep_n = π(region_m, cantor_t)` and `proof_hash = sha256(sha256(int_to_bytes_be_min(sidestep_n)))` (§6.8). Reject unless it equals the `proof` tag. All three axes use Merkle roots; there is no mixed mode (§6.6).
- 8e. Plane changes are valid mid-chain; a verifier MUST support hops where `plane1 != plane2` (§8.7.1 step 2) and MUST NOT reject on that basis.
- 8f. For every event in the span, reject unless its sector tags match the §10 derivation from its `C` tag.
- 8g. `enter-hyperspace` and `hyperjump` events in the span MUST be verified per DECK-0001 §3.2 and §5.3–§5.5 respectively **if** the verifier implements DECK-0001. A verifier that does not implement DECK-0001 MUST reject a bundle whose span contains such an event, and MUST NOT skip them silently. Note that `cyberspace-cli` is not a reference implementation for DECK-0001 v3 and MUST NOT be treated as authoritative for hyperspace behaviour (`NOTES/protocol-facts.md` §9).

**Step 9 — Placement validation.**
For the placement set:

- 9a. Reject unless the `i` values are exactly `0 .. N-1`, each appearing once.
- 9b. For each placement, reject unless its `e`-`anchor` id resolves to a movement event in the span whose `A` is `hop` or `sidestep` (§4.4).
- 9c. Reject unless each anchor id is referenced by at most one placement (§4.4).
- 9d. Reject unless each placement's `C` tag equals its anchor's `C` tag exactly.
- 9e. Reject unless each placement's `e`-`genesis` id equals the spawn id.
- 9f. Reject unless the ordering of placements by `i` matches the ordering of their anchors along the chain. Placement order is chain order; it MUST NOT be derived from `created_at`.
- 9g. Reject unless each placement's sector tags match the §10 derivation from its `C`.

**Step 10 — Completion validation.**
Recompute `root` per §5.5 from the placement ids in ascending `i`; reject on mismatch with the `root` tag. Reject unless `n` equals `N`. Reject unless the `e`-`terminal` id equals the anchor of placement `N-1`. Reject unless `1 ≤ h ≤ 84`. Reject unless the `e`-`genesis` id equals the spawn id. Reject if the completion event carries sector tags (§4.5).

**Step 11 — Stat derivation.**
For each placement, derive `region_n` and `stat_seed` per §6.3 at the declared `h`. This step MUST NOT be able to fail for a bundle that reached it: it is a total function of a validated coordinate and a validated height. Populate `stats`.

**Step 12 — Depth accounting.**
Compute:

- `chain_depth` = the number of **serially-seeded** movement events in the span: `hop` + `sidestep` + `enter-hyperspace`. `hyperjump` events MUST NOT be counted, because a ride's per-block work shares one `previous_event_id` and computes in parallel (DECK-0001 §5.3; `DESIGN.md` §2). See §9.7.
- `placement_count` = `N`.
- `hyperjump_count` = the number of `hyperjump` events in the span, reported separately and never folded into `chain_depth`.

**Step 13 — Result.**
Return `{valid: errors.length == 0, errors, stats, chain_depth}`. A verifier SHOULD also return `placement_count`, `hyperjump_count`, `bundle_id` (the completion event id), and the per-placement region roots from Step 11.

---

## 8. What Verification Does Not Prove

A valid bundle is a narrow claim. This section is normative in one respect: **an implementation MUST NOT present any of the following as established by verification**, in a user interface, an API response, or documentation.

### 8.1 It does not prove elapsed wall-clock time

Verification proves *serial dependency* — that hop `N`'s temporal seed derives from hop `N−1`'s event id, so the work could not have been precomputed or amortized (§5.1, `:445`). It does not measure seconds. Per-hop cost varies with terrain `K` (binomial on `[0,16]`, so worst case is ~4,000× the best case in tree size) and with single-threaded hardware. Nostr `created_at` values are self-asserted and prove neither duration nor honesty. A bundle's cost is a lower bound expressed in *sequential computations by one key*, not in time. Converting that to a duration requires assumptions this document does not supply.

### 8.2 It does not prove travel or distance

Chain braiding, region preimages, and Cantor roots prove **work plus liveness**, not traversal. The base spec is explicit that region preimages can be computed without maintaining a chain. Implementations MUST NOT build rules that assume any distance was actually crossed (`DESIGN.md` §4.3). Hyperjump within a bundle's span proves less still: it is verifiable travel whose cost compresses with hardware.

### 8.3 It does not prove originality, priority, or uniqueness

A published bundle is verifiable, therefore readable, therefore reproducible. Anyone can re-walk the path and produce a **bit-identical** artifact — same placements, same root, different pubkey. This is accepted and arguably correct: forging costs what making cost (`DESIGN.md` §4.5). Priority is not a property of the bundle. It comes from external timestamp anchoring (OpenTimestamps or equivalent) over the completion event id, and a verifier that has not been given such an attestation MUST NOT report or imply "first" or "original."

### 8.4 It does not prove the absence of a fork

Step 7 detects forks *within the supplied events*. It cannot detect a conflicting event the verifier was not given. Establishing non-equivocation requires relay coverage that no verifier can guarantee and that this document deliberately does not require (§4.1). A bundle that verifies today may sit beside a conflicting event that surfaces tomorrow. Note also that the explicit fork-invalidation rule this document adopts appears only in DECK-0001 §8; the base spec's §12.1 is non-normative on the point (§9.9).

### 8.5 It does not prove the chain is still live

Publishing a new `spawn` event at any time invalidates all prior movement events in the old chain, because it carries a newer timestamp; the keypair returns to its spawn coordinate and the old history remains on relays but is no longer the active chain (§3.2, §12.3). A verified bundle is a statement about a chain segment as it existed, not about the creator's current state. See §9.10.

### 8.6 It does not prove one person, or a scarce person

Bundles are scoped by signer. A signer is a keypair, and one operator may hold arbitrarily many. Verification says nothing about how many bundles one human produced in parallel across many keys, nor about the courier-swarm construction where a relay of distinct ground keys each makes a short move from its own spawn. Eliminating sybils is an explicit non-goal (`DESIGN.md` §3); attunement (§4.4), which is what prices the swarm, is **not implemented in v1** and is not verified here.

### 8.7 It does not prove key custody, then or now

A valid signature proves the private key signed. It does not prove who held it, that the same party held it across the whole chain, or that they hold it now. A bundle transferred by handing over a private key verifies identically to one held continuously.

### 8.8 It does not prove ownership, transfer, or attunement

Transfer, single-use seals, and attunement are out of scope for v1 (§1.1). A verifier MUST NOT report an owner distinct from the signing pubkey, MUST NOT report a bundle as transferred, and MUST NOT report a bundle as attuned. Barter double-spend is detectable via chain forks but not preventable, and is documented as trust-based (`DESIGN.md` §4.3).

### 8.9 It does not prove cost in any fixed unit

Protocol costs are fixed in absolute terms while hardware improves, and cyberspace forgoes the global state that would be needed to retarget. Costs deflate monotonically. Difficulty coupling (`DESIGN.md` §5) is the intended mitigation and is **not implemented in v1**. A bundle minted at one moment and an identical bundle minted a decade later verify the same and are not the same expenditure.

### 8.10 It does not prove the payload is anything

Payloads are opaque (§4.6). Verification asserts nothing about their meaning, legality, provenance, encoding validity, or renderability, and MUST NOT be presented as endorsing them.

### 8.11 It does not prove stats are rare, good, or worth anything

Stat derivation is a deterministic function of the region root (§6). It reports what a region yields. It does not establish that the region is scarce, that the yield is desirable, or that anyone will pay for it. Two placements in the same aligned subtree at height `h` have identical stats, and there are `2^(3(85−h))` such subtrees.

### 8.12 It does not prove events were ever published

Verification runs over a supplied event set. It does not prove any event reached a relay, was propagated, was observed by anyone else, or is retrievable now. The serial-dependency argument assumes publication between hops; the bundle itself carries no evidence of it.

---

## 9. Open Questions

These are points where `DESIGN.md` is ambiguous or silent. They are recorded rather than resolved. Each MUST be settled before this document leaves draft status; §9.1, §9.2, and §9.3 block interoperability outright, because two implementations choosing differently will disagree on validity or on stats.

**9.1 Event kind numbers.** `DESIGN.md` §8 requires chalkline to "pick from a range clear of" `3333`, `33330`, and `321`, and to register the choice in chalkline's DECK. It does not name the numbers, nor state whether placements should follow DECK-0001's precedent of reusing an existing kind with a distinguishing tag versus minting new kinds. `DESIGN.md` argues for new kinds — placements are content bound to movement, not movement itself — but the specific values are unassigned. Nostr replaceable/addressable ranges also constrain the choice, and `DESIGN.md` does not say which behaviour is wanted.

**9.2 Plane scoping.** `DESIGN.md` §4.2 states that *"`BUNDLE.md` MUST state whether artifacts are plane-scoped, and whether a bundle's placements may span planes"* — and then does not say which. This document therefore does **not** state it, and the question is open. Three sub-questions:
  (a) May a bundle's placements span both planes? Plane changes are valid mid-chain (§8.7.1), so the base protocol permits it.
  (b) Since bit 0 of a coordinate is the plane and coord256 is the raw pubkey, the low bit of a key determines its spawn plane — one bit, trivially ground, roughly half of all keys in each. Does that matter for artifacts?
  (c) `region_n` excludes the plane bit (§6.5), so identical `(x,y,z)` in opposite planes yield identical stats, while base terrain `K` *does* include it. Should chalkline mix the plane into `stat_seed` to break that tie, or preserve the base spec's plane-blind region identity? Mixing it in would be a chalkline-local divergence from §7.2's region semantics.

**9.3 Stat height.** This document requires the completion event to declare `h` and applies it bundle-wide, but `DESIGN.md` §4.2 says only "the aligned subtree containing the placement" without naming a height. Open: whether `h` is producer-chosen (as specified here provisionally), a fixed protocol constant, per-placement, or a vector of heights yielding layered stats. A producer-chosen `h` is grindable in a weak sense — a creator can try several heights and publish the flattering one — which cuts against §4.2's rationale for putting all variance in the coordinate.

**9.4 Stat schema.** §6 specifies the seed and the byte stream; it does not specify attribute names, count, ranges, or rarity curves, because `DESIGN.md` §4.2 specifies only that stats are a pure function of the region root. Open: the mapping itself, and whether it is versioned separately from the bundle format.

**9.5 Enter-hyperspace as an anchor.** `DESIGN.md` §4.1 says enter-hyperspace *"is serially seeded and may anchor, but its spatial slot is zeroed ... so it carries no location and cannot place."* This document reads that as MUST NOT anchor a placement, since a placement is by definition a coordinate plus payload. But "may anchor" may have been intended to admit some future non-placing bundle element. If so, that element type is unspecified.

**9.6 Anchor contiguity.** This document permits hops and sidesteps in the span that anchor nothing. `DESIGN.md` does not address whether an N-placement artifact should be required to place on *every* eligible movement event, which would make placement density itself a cost signal, or whether sparse anchoring is intended. It also does not say whether hyperjumps mid-bundle should segment or annotate the artifact.

**9.7 Depth accounting.** §7 Step 12 counts `hop + sidestep + enter-hyperspace` as `chain_depth` and excludes `hyperjump`, on the ground that only the first three are serially seeded (`DESIGN.md` §2, §4.1). `DESIGN.md` does not define `chain_depth` for the v1 verifier signature. Open: whether enter-hyperspace should count at all (it is serial but does no spatial work), whether depth should be weighted by terrain `K` rather than counted, and whether depth should measure the whole span or only the segment containing placements.

**9.8 Payload codec registry.** §4.6 fixes the tag shape and leaves the codec set unassigned. `DESIGN.md` describes placements as "a coordinate plus payload" without characterising the payload. Open: the initial codec set, size limits, and whether codecs are registered in chalkline's DECK alongside kinds.

**9.9 Base-protocol fork rule.** §4.2 adopts DECK-0001 §8's rule — *two movement events with the same `previous_event_id` are a fork; both branches are invalid from that point* — and applies it to bundles. The base spec has no equivalent explicit rule: §12.1 (`:1312`) is under a heading marked non-normative and says only that a linear chain makes forking detectable. Whether the base protocol intends the same invalidation is not stated anywhere in the fetched sources (`NOTES/protocol-facts.md`, UNKNOWN). chalkline is importing an extension's rule into a base-protocol context.

**9.10 Respawn and completed bundles.** §8.5 notes a later spawn event invalidates the old chain (§3.2, §12.3). `DESIGN.md` does not say whether a bundle completed before a respawn remains valid, becomes historical-but-verifiable, or is destroyed. The fidelity-bond reasoning in §4.3 — *a forked key's history becomes worthless* — suggests intent, but says nothing about honest respawns.

**9.11 Signature requirement versus the reference CLI.** §4.7 requires valid signatures, which means no bundle can be assembled from unmodified `cyberspace-cli` output (`"sig": ""` at `nostr_event.py:47-71`). `DESIGN.md` does not address the tooling gap. Open: whether v1 tests should include unsigned fixtures for the local-chain case §8.2 contemplates.

**9.12 Sidestep `content` asymmetry.** The reference CLI sets `content = proof_hash_hex` on sidesteps but `""` on spawn and hop (`nostr_event.py:202`, `:92`, `:160`); §8.5 specifies no `content` requirement either way and the CLI's own conformance checklist does not flag it (`NOTES/protocol-facts.md`, UNKNOWN). Since `content` enters the event id (§5.3), two producers disagreeing here produce different sidestep ids for the same movement — which breaks the bit-identical reproducibility §8.3 depends on. chalkline cannot fix this in its own events; it inherits it. Open: whether `tests/vectors/` should pin one convention for sidesteps in the span.

---

## 10. Test vectors

`tests/vectors/` MUST contain valid and invalid cases, each invalid case naming the section of this document it violates, matching the `errors[]` identifiers of §7. At minimum, invalid cases SHOULD cover: multi-signer bundle (§4.1); fork in span (§4.2); hyperjump anchor (§4.4); spawn anchor (§4.4); two placements on one anchor (§4.4); placement `C` ≠ anchor `C` (§7 Step 9d); non-contiguous `i` (Step 9a); `i` order disagreeing with chain order (Step 9f); wrong `root` (Step 10); wrong `n` (Step 10); uppercase hex (§5.2); leading-zero integer (§5.2); reordered tags producing an id mismatch (§5.3); empty `sig` (§4.7); non-empty `content` on a placement (§4.3); `h` out of range (§6.2); and the `N = 1` root case (§5.5).

Valid cases SHOULD include: a single-placement bundle; a bundle whose span contains a plane-changing hop (§7 Step 8e); a bundle whose span contains anchoring-free hops (§4.4); and a bundle with two placements in the same aligned subtree at `h`, asserting identical stat seeds (§6.5).

---

## 11. References

- `CYBERSPACE_V2.md` — Cyberspace Protocol v2, spec version `2026-03-16-h34-corrected`, at `arkin0x/cyberspace@919494a`.
- `decks/DECK-0001-hyperspace.md` — Hyperspace (Bitcoin block transit), Draft v3, 2026-08-24, same commit.
- `../NOTES/protocol-facts.md` — verified transcription of both, with file and line citations, and an explicit `UNKNOWN` section for what is not established.
- `../DESIGN.md` — chalkline design intent (non-normative).
- NIP-01 — Nostr event serialization and id computation.
- RFC 2119, RFC 8174 — requirement keywords.
