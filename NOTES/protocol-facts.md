# Cyberspace Protocol & CLI: Verified Facts

Compiled 2026-08-25; re-audited adversarially 2026-08-28; re-pinned to current upstream 2026-11-20.

## Provenance

- `arkin0x/cyberspace` (branch `master`) @ `4cb8f1b` "DECK-0003 §1.18 Facing: how an avatar turns as it moves; its front is -Z (#113)", 2026-11-16.
  - `CYBERSPACE_V2.md` ("Last updated: October 8, 2026", `:4`). Chain rules revision `CHAIN_RULES_REVISION = "2026-10-08-residue-openings"` (§8.12, `:2154`). The string `2026-03-16-h34-corrected` is the GPS mapping version (§9.5, `:2341`), not the chain rules; the two are independent (§8.12, `:2249`).
  - `RATIONALE.md`, `readme.md`, `grandfathered-v1-hops.txt`, `grandfathered-v2-sidesteps.txt`, `chain-rules-2026-10-08-residue-openings.json`, `hop-residue-reference.py`
  - `decks/README.md`, `decks/DECK-0001-hyperspace.md`, `decks/DECK-0002-games.md`, `decks/DECK-0004-items-and-chests.md`, `decks/DECK-0008-seeds.md`, `decks/landfall-reference.py`, `decks/grandfathered-v1-hyperjumps.txt`
  - `docs/agents.md`, `docs/what-can-be-sold.md`, `docs/territory-conflict-game-layer.md`, `docs/analysis/deck-domains-ground-up-analysis.md`
- `arkin0x/cyberspace-cli` (branch `main`) @ `4e5dab7` "Avatars carry no proof of work: verify_avatar and the 128 KiB limit; kind 11333 (#34)", 2026-11-15.
  - `README.md`, `docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md`, `docs/spec-conformance/DECK-0001_v3_2026-08-24.md`
  - `src/cyberspace_cli/`: `cli.py`, `cloud_cli.py`, `nostr_event.py`, `targets.py`, `helptext.py`, `viz/move_viz.py`
  - `src/cyberspace_core/`: `coords.py`, `cantor.py`, `movement.py`, `residue.py`, `chain.py`, `grandfathered.py`, `terrain.py`, `sector.py`, `location_encryption.py`

Previous pins: `cyberspace@919494a` (2026-08-24) and `cyberspace-cli@2dff988` (2026-05-01).

**Re-pin note (2026-11-20).** Every line-numbered citation was re-checked against the new commits; all of the old base-spec line numbers had moved (for example the bit layout, old `:181`, is now §2.2 `:195`). Substantive changes since the old pins, beyond line numbers:

- Hops carry proof **version 2**, residue openings: a seal in `proof`, plus `mn` and `mp` tags (§5.5 below). Version 1 hops are invalid unless grandfathered.
- Sidesteps are at construction **version 3**: seeded leaves, 8 sampled openings and a re-roll price in `mn` (§5.6 below). The old unseeded `CYBERSPACE_SIDESTEP_V1` leaves described in the previous revision are invalid. §6.10 is now titled "Openings".
- The base spec now has normative chain resolution (§8.7.3): inauthentic events discarded, a fork kills the chain from the spawn, exactly-once tags, skipped unknown actions, virtual brackets, and named chain-rules revisions with golden vectors (§3 below). The old UNKNOWN about a base fork rule is resolved.
- DECK-0001 is **mandatory** ("movement is universal"), rides carry an `mn` re-roll price, and the zero-length ride was removed.
- The kind registry grew from three kinds to many (§2.1 below).
- The CLI is active again and tracks the chain rules and DECK-0001 v3 stop geometry (§9 below).

Every claim below is cited to a spec section (`§`) and/or a file path with line numbers at the commits above. Anything not directly confirmed in these sources is listed under **UNKNOWN** rather than inferred.

**Note on section numbers.** Inline comments in `movement.py` still cite an older numbering (for example "§5.5.2.1" at `movement.py:139`, "§5.5.3" at `:160`, "§5.7" at `:163`) that does not match `CYBERSPACE_V2.md` at HEAD (terrain K is §5.2; the hop proof hash is §5.6). Reported as-is. See **UNKNOWN**.

---

## 1. Coordinate derivation from a pubkey

- Cyberspace coordinates are 256-bit integers (`coord256`) split into three 85-bit unsigned axes X, Y, Z plus a 1-bit plane selector `P` (85×3+1 = 256). (§2.1, `:179-193`)
- Bit layout is **interleaved (Morton-style), not three contiguous 85-bit runs.** §2.2 (`CYBERSPACE_V2.md:195-207`): "The three axes are **interleaved** into the 256-bit integer, not packed sequentially." Bit `0` (LSB) is the plane bit `P`; bits `3, 6, 9, ...` are X; `2, 5, 8, ...` are Y; `1, 4, 7, ...` are Z. "The result looks like: `XYZXYZXYZ...P`".
- Reference encode/decode is normative pseudocode in §2.3 (`CYBERSPACE_V2.md:209-230`), reproduced at `coords.py:91-98` (`xyz_to_coord`) and `:101-109` (`coord_to_xyz`), with `AXIS_BITS = 85` at `coords.py:47`:

```python
def xyz_to_coord(x: int, y: int, z: int, plane: int = PLANE_DATASPACE) -> int:
    """Convert (x, y, z, plane) to a 256-bit interleaved coordinate."""
    coord = plane & 1
    for i in range(AXIS_BITS):
        coord |= ((z >> i) & 1) << (1 + i * 3)
        coord |= ((y >> i) & 1) << (2 + i * 3)
        coord |= ((x >> i) & 1) << (3 + i * 3)
    return coord
```

  Bit `3i+1` = Z bit `i`, `3i+2` = Y bit `i`, `3i+3` = X bit `i`, bit `0` = plane.
- **Consequence relied on downstream:** "coordinates that are spatially close share similar bit prefixes" (§2.2, `:207`), which makes aligned subtrees efficient and, per DECK-0001 §4.4 (`:276`), makes station lookup a prefix search.
- **Hex prefixes** (§2.5, `:243-282`, new since `919494a`): a string of 1 to 64 hex characters names a prefix region; fewer than 64 characters never fix the plane bit. A prefix region is not a §4.7 region and has no key.
- **Identity-to-coordinate rule:** "Your Nostr public key, the 256-bit number that *is* your identity, is also your coordinate." (§3.1, `:290`); "The coordinate in a spawn event MUST equal the event's public key" (§3.2, `:300`; §8.3, `:1775`).
- In the CLI this is an identity function on the 32 raw pubkey bytes: `_coord_hex_from_pubkey_hex` returns the 32-byte pubkey hex unchanged (`cli.py:137-141`). `spawn` calls it on the generated or imported pubkey (`cli.py:809`) and decodes the result with `coord_to_xyz` (`cli.py:834`).
- The plane bit of a spawn coordinate is whatever the low bit of the pubkey happens to be. No spec text constrains it. See **UNKNOWN**.
- **Planes and twins** (§2.4, `:232-241`; §7.2, `:1330`): a region and its twin (same X, Y, Z and height, other plane) share one `region_n`, one key, one lookup id and one bag per author; each bag entry's `C` decides its plane (§7.6 "Planes", `:1455-1468`).
- **Respawning:** "The newest spawn event always starts the active chain (§8.7.3), so a respawn ends the old chain ... Prior movement history remains on relays but is no longer part of the active chain." (§3.2, `:306`) A chain now ends in one of three ways (§3.2, `:314`): a respawn ends it; an invalid event **freezes** it at the last valid position (`:310`); an invalid newest spawn or a fork **kills** it, and the identity stands at its spawn coordinate (`:308`, `:312`). §12.3 still lists "Chain abandonment" (`:2551`).

---

## 2. Event kinds and their required tags

### 2.1 Kind registry

The base spec and the DECKs now claim many kinds. `decks/README.md` lists them in three places: the DECK registry (`:57-67`), "Kinds defined by the base spec" (`:69-74`) and "Reserved kinds" (`:76-77`).

| Kind | Purpose | Claimed by | Ref |
|---|---|---|---|
| `3333` | Every movement action: base `spawn`, `hop`, `sidestep`, `enter-virtual`, `exit-virtual`; DECK-0001 `enter-hyperspace`, `hyperjump`; any action inside a virtual bracket | base spec + DECK-0001 | §8.1 (`:1753-1755`), §8.8 (`:1937-1949`), `decks/README.md:71` |
| `33330` | Bag (location-encrypted content), addressable | base spec | §8.6 (`:1822-1827`), `decks/README.md:72` |
| `33335` | Sealed entry (one bag entry carried by reference) | base spec | §8.6 (`:1841`), `decks/README.md:73` |
| `11333` | Avatar | base spec | §8.10 (`:2008`), `decks/README.md:74` |
| `33332` | Public shards; reserved, not yet specified | base spec (reserved) | `decks/README.md:77` |
| `321` | Block anchor | DECK-0001 | DECK-0001 §2 (`:112-133`) |
| `33331` | SNO object | DECK-0003 | `decks/README.md:60` |
| `3330` | SNO bag item (shard) | DECK-0003 | `decks/README.md:60` |
| `3340` | Item | DECK-0004 | `decks/README.md:62` |
| `3341` | Chest | DECK-0004 | `decks/README.md:62` |
| `33340` | Item listing | DECK-0004 | `decks/README.md:62` |
| `23330` | Ephemeral chat envelope | DECK-0005 | `decks/README.md:63` |
| `23333` | Chat line (rumor inside the envelope) | DECK-0005 | `decks/README.md:63` |
| `1111` | Sealed comment (NIP-22 kind) | DECK-0006 | `decks/README.md:64` |
| `1063` | Hidden audio (NIP-94 kind, as a bag item) | DECK-0007 | `decks/README.md:65` |
| `3350`, `3351`, `3352` | DNA, seed, tend | DECK-0008 | `decks/README.md:66`; DECK-0008 table (`:86-91`) |
| `20000`-`29999` | NIP-01 ephemeral range, used for a game's rapid actions anchored to the chain; DECK-0002 defines no new kind | DECK-0002 | DECK-0002 `:25`, `:40`; `decks/README.md:59` |

**Taken in practice, not by any spec:** DECK-0008 records (`:96`) that public relays hold `kind 3334` and `kind 33334` events from another application and `kind 3338` events carrying image prompts, and that the nostr kind registry lists `3302` to `3313` for another application. The range `3330`-`3352` is therefore crowded.

`decks/README.md:11,15` still states that DECKs are the place "to allocate additional Nostr event kinds", and that a DECK MAY "Define new Nostr event kinds" and new `A` values for `kind=3333`.

### 2.2 Event id and signature

Event `id` is computed per NIP-01 (§8.2, `:1757-1763`), implemented at `nostr_event.py:17-28` (`serialize_event_for_id`) and `:31-47` (`compute_event_id_hex`).

`sig` "MUST be a valid Schnorr signature over the event `id`" (§8.2, `:1765`). The old allowance for prototypes that leave `sig` blank is gone. Instead: "An event whose `id` or `sig` fails these checks is not authentic, and a reader discards it before resolving a chain" (`:1765`; §8.7.3, `:1903`).

The CLI still builds every event with `"sig": ""` (`nostr_event.py:73`, comment at `:72`: "Signature is intentionally blank for now. We can sign at publish-time."). Its conformance checklist still records this as its known gap (`docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md:4`, `:98`; the latter cites `nostr_event.py:70`, which is stale). Under §8.7.3 an unsigned event is not authentic and is discarded before resolution.

### 2.3 Base protocol events (§8)

**Spawn** (§8.3, `:1767-1776`): `["A","spawn"]`; `["C","<coord_hex>"]` (64 lowercase hex, MUST equal `pubkey`); sector tags `X`, `Y`, `Z`, `S`, each exactly once and equal to the values computed from `C`. CLI: `make_spawn_event` (`nostr_event.py:92-95`), `content=""`.

**Hop** (§8.4, `:1778-1795`): `["A","hop"]`; `["e","<spawn_event_id>","","genesis"]`; `["e","<previous_event_id>","","previous"]`; `["c",...]`; `["C",...]`; `["proof","<seal_hex>"]` (the §5.10 seal); `["mn","<nonce_hex>"]` (16 lowercase hex); `["mp","<openings_hex>"]`; sector tags. `mp` is exactly `HOP_SAMPLES × (128 + 64 × depth)` characters (`:1793`). A hop with no `mn` and no `mp` is version 1 and MUST be rejected unless listed in `grandfathered-v1-hops.txt` (`:1795`). CLI: `make_hop_event` (`nostr_event.py:241-283`), `content=""`; it appends `mn` and `mp` after `proof` (`:281`).

**Sidestep** (§8.5, `:1797-1820`): `["A","sidestep"]`; both `e` tags; `c`, `C`; `["proof","<proof_hash_hex>"]`; `["mr","<M_x>:<M_y>:<M_z>"]`; `["mp","<x>:<y>:<z>"]` (per axis, `SIDESTEP_SAMPLES + 1` paths of `h` siblings each, `:1816`); `["mn","<nonce_hex>"]`; `hx`, `hy`, `hz`; sector tags. A segment of exactly `64 × h` characters is a v1 proof; a sidestep without `mn` is v2 and is rejected unless grandfathered (`:1818`). CLI: `make_sidestep_event` (`nostr_event.py:286-331`), same tag set.
**Observed asymmetry (still present):** the CLI sets `content = proof_hash_hex` on sidesteps (`nostr_event.py:331`) but `""` on spawn (`:95`) and hop (`:283`). §8.5 still specifies no `content`, and `content` is not one of the tags the chain rules read (§8.8, `:1957-1967`).

**Bag** (§8.6, `:1822-1857`): required `["d","<lookup_id_hex>"]`, `["encrypted","aes-256-gcm","<payload_base64>"]` and `["version","2"]`; optional `["H","<height>"]` (the height tag; `h` is read only as a legacy fallback, `:1849-1857`), `hint`, conditional sector tags, `contents` tags. The cipher is now in the spec (§7.6). CLI: `make_encrypted_content_event` (`nostr_event.py:219-238`); `encrypt` passes `algorithm="aes-256-gcm"` (`cli.py:1505`) and writes the height tag only with `--publish-height` (`cli.py:1458-1460`, `:1507`); AES-256-GCM with a 12-byte random nonce prepended (`location_encryption.py:91-99`).

### 2.4 Core action types (§8.8)

| `A` tag | Description | Proof type | Ref |
|---|---|---|---|
| `spawn` | Identity placement at pubkey-derived coordinate | None | §8.3 |
| `hop` | Movement via Cantor pairing tree | Residue openings of the Cantor roots (§5.9) | §8.4 |
| `sidestep` | Boundary crossing via Merkle hash tree | Merkle root (§6.4) | §8.5 |
| `enter-virtual` | Opens a virtual bracket; position held | None | §8.11 |
| `exit-virtual` | Closes it; position restored | None | §8.11 |

"All five use event `kind = 3333`." (§8.8, `:1949`) §8.9 (`:1973-2006`): DECKs MAY add kinds, `A` values and validation rules; DECK-0001 is listed as mandatory (`:1980`).

### 2.5 DECK-0001 events (mandatory extension)

**`enter-hyperspace`** (DECK §3.1, `:205-219`): `A`; `e` genesis; `e` previous; `c`; `C` (MUST equal `c`); `proof` per §3.2; sector tags from `C`. Each exactly once (`:217`). Optional `net`, informational and never read (`:219`; §1.4, `:104-108`).

**`hyperjump`** (DECK §5.2, `:302-320`): `A`; both `e` tags; `c`; `C` (destination stop); `from_height`; `B` (`B_to != B_from` always, `:311`); `as_of` (required on the first ride after boarding); `proof` (Merkle root, §5.4); `mp` (§5.5); `mn` (re-roll nonce, 16 lowercase hex, `:315`); sector tags. Optional `net`; `e` tags marked `hyperjump_from` / `hyperjump_to`.

**Block anchor** (kind 321, DECK §2.1, `:116-125`): `C`, `M`, `B`, `H`, `P`, sector tags from `C`. SHOULD carry `net` and `N`; MAY carry `header`, the raw 80-byte block header (§2.5, `:153-197`). Block times and bits come only from a header verified against an accepted `H`; a reader MUST NOT take times from `created_at` (`:165`) and MUST NOT order blocks or find the tip by `created_at` (`:171`).

**Legacy anchors** (DECK §2.2, `:127-129`): an anchor without `M` has its `C` treated as the merkle root; such events MUST NOT be a source of landfall coordinates directly.

**Anchor validation** (DECK §2.3, `:131-133`): valid iff `M` and `H` match the block at height `B` on **Bitcoin mainnet** (§1.4), `P` matches, and `C` equals the §1 derivation.

---

## 3. Movement chain rules

Normative source: §8.7.3 "Resolving a chain" (`:1899-1935`), §8.8 (`:1937-1971`), §8.9 (`:1973-2006`), §8.11 (`:2046-2150`), §8.12 (`:2152-2249`).

- Cyberspace state is "a per-pubkey, linear hash chain of signed Nostr events" (§8, `:1749`). "Hash chain continuity" (§1.3, `:168`).
- **Authentic events only** (`:1903`): before resolving, a reader MUST discard every event that is not a valid NIP-01 event signed by the identity's own pubkey. A discarded event is "not a spawn, ... not a link ..., not a branch of a fork, and ... not an invalid event". A branch through a discarded event is cut off (`:1907`).
- **Chain events** (`:1905`): only kind 3333 events take part.
- **Resolution** (`:1911-1915`): rule 1, the newest spawn (largest `created_at`, ties to larger id) starts the active chain even when invalid; rule 2, only events whose `e` genesis names it; rule 3, follow `e` previous links; rule 4, **forks**: more than one distinct event naming the current event as previous means the chain is **dead**, invalid from the spawn, identity at its spawn coordinate (`:1914`); rule 5, the head.
- A spawn is never a link (`:1917`). Rules read the first copy of each `e` marker (`:1921`).
- **Why a fork kills the chain** (`:1923-1925`): the earlier "older branch continues" rule allowed free rewinds and is withdrawn; this restores the v1 rule and DECK-0001's earlier §8.
- **Confirm the live head before signing** (normative for clients, `:1927`).
- **Validity and position** (`:1929`): resolution first, validity second. An invalid event makes the chain invalid from that event; the identity stands at its last valid position (frozen).
- **Exactly one `A` tag** (§8.8, `:1951`) and **each tag the chain rules read exactly once, with a well-formed value** (`:1953-1969`), with the table of read tags at `:1957-1967`. Grandfathered events may lack only the tags their list waives (`:1969`). Other tags (`client`, `net`, other `e` markers, `no-broadcast`, ...) are free.
- **Movement is universal** (§8.9, `:1982`): a DECK whose action can change position is **mandatory**; DECK-0001 is mandatory, so a verifier without it is non-conforming.
- **Unrecognized actions are skipped** (§8.9, `:1984-1996`): links still followed; position carried across (`c` of the next recognized action MUST equal the `C` of the nearest recognized action before it, so a skipped action that moved the identity invalidates the chain at the next recognized action); work after a skipped action is seeded by the skipped event's id; look-back rules see through skipped actions; head position is the last recognized `C`. A skipped action is checked only for authenticity, linkage and exactly one `A` tag (`:1991-1996`).
- **Virtual brackets** (§8.11, `:2046-2150`): `enter-virtual` (`:2054-2078`) holds the base position and names a game by a `p` tag marked `game` and a `region` tag; actions inside (`:2080-2092`) use game-chosen names, carry no base proof and do not move the identity; `hop`, `sidestep`, `enter-hyperspace`, `hyperjump`, `enter-virtual` are reserved inside (`:2115`); `exit-virtual` (`:2094-2109`) restores the base position. Rules at `:2111-2122`, base verification at `:2124-2132`. A bracket may be left unclosed (`:2119`).
- **Encounter primitive** (open, `:2150`): "a contact event in which two chains reference each other's heads" is not yet specified.
- **Chain rules revision** (§8.12, `:2152-2249`): `CHAIN_RULES_REVISION = "2026-10-08-residue-openings"` (`:2154`); revision table (`:2158-2162`). A verifier conforming to a revision "MUST reach every vector's verdict, its `position` and, for an invalid vector, its `invalid_at`" in that revision's golden vector file (`:2164`, `:2166`, `:2198-2204`): `chain-rules-2026-10-08-residue-openings.json` for the current revision, which carries its own test list of v1 hops to use in place of `grandfathered-v1-hops.txt` only while the vectors run (`:2216-2229`). Real chains with reference verdicts are in `chain-rules-2026-10-08-residue-openings-real.json` (§14, `:2632`), not normative.
- **Not a heartbeat:** "An avatar's last hop event remains its state indefinitely." (§5.1, `:536`)
- **Plane changes are valid mid-chain** (§8.7.1 step 2, `:1865`).
- **DECK-0001 chain rules** (§4.3 `:268-272`, §8 `:418-426`):
  - The first `hyperjump` after `enter-hyperspace` MUST carry `as_of` and have `from_height = station(C_e, as_of)`.
  - Each later `hyperjump` not separated by a `hop` or `sidestep` MUST have `from_height` equal to the previous ride's `B`.
  - A `hyperjump` whose previous action is neither `enter-hyperspace` nor `hyperjump` is invalid; an `exit-virtual` stands for the action before its bracket (`:425`).
  - No zero-length ride (§5.6, `:378-386`).
  - DECK §8 now defers forks to base §8.7.3 rule 4 (`:420`).
  - Stops are verified against Bitcoin mainnet; fewer than six confirmations SHOULD be treated as provisional (`:426`).
  - Exit (DECK §6, `:404-408`): an ordinary `hop` or `sidestep` whose `c` is the stop coordinate.
- **CLI-side chain-local rules** (not spec text):
  - `move` appends to the local chain and reads `events[0]["id"]` as genesis and `events[-1]["id"]` as previous (`cli.py:2570-2571`).
  - `--sidestep` cannot be combined with `--hyperjump` (`cli.py:2536`); `--hyperjump` refuses (`_refuse_to_ride`, `cli.py:1779-1795`, called at `:2598`).
  - While the chain's last event is a ride, ordinary `move` is blocked unless `--exit-hyperjump` is passed (`cli.py:2601-2604`).
  - Hops above `--max-lca-height` are refused locally or offered to HOSAKA (`cli.py:2437-2439`, `:2479`).
  - The CLI's chain verifier is `cyberspace_core/chain.py`, which declares `CHAIN_RULES_REVISION = "2026-10-08-residue-openings"` (`chain.py:111`) and keeps `PREVIOUS_REVISION` (`:114`).

---

## 4. Temporal-axis work: height K and seed, per action type

There are **two distinct terrain functions**: the base protocol's coordinate-derived K (§4.1 below) and DECK-0001's block-derived line terrain (§4.5 below). Do not conflate them.

### 4.1 Base-protocol terrain K (normative, §5.2)

Normative source: `CYBERSPACE_V2.md` §5.2 (`:538-567`), reproduced by `terrain.py:40-85` (`terrain_k`).

**Constants** (`:543-544`): `TERRAIN_DOMAIN_V2 = b"CYBERSPACE_TERRAIN_K_V2"` (`terrain.py:31`); `TERRAIN_CELL_BITS = [3, 7, 9, 11]` (default `cell_bits` at `terrain.py:46`). Any change bumps the domain string (`:546`); `terrain.py:29-30` records the V1 to V2 bump, and `TERRAIN_DOMAIN_V1` remains as a deprecated alias (`terrain.py:104`).

**Algorithm** (`:548-565`), given destination `(x2, y2, z2, plane)`:
1. For each `bits` in `[3, 7, 9, 11]`: align each axis to `2^bits`; `cell_coord = xyz_to_coord(bx, by, bz, plane)` as exactly 32 big-endian bytes (`terrain.py:76`); `digest = sha256(TERRAIN_DOMAIN_V2 || byte(bits) || cell_coord_bytes)`; `nibble_i = digest[0] & 0x0F`.
2. `word16 = (nibble_0 << 12) | (nibble_1 << 8) | (nibble_2 << 4) | nibble_3`.
3. `K = popcount(word16)`, in `[0, 16]` (`terrain.py:84-85`).

K is computed from the destination coordinate, **including the plane bit**, not from time or identity (§5.3 "Key insight", `:581`; `terrain.py:62` rejects planes outside `{0,1}`). Distribution approximately Binomial(16, 0.5), worst case `2^16` pairs (`:567`). `movement.py:19` sets `TEMPORAL_MAX_COMPUTE_HEIGHT = 17`.

### 4.2 Work seed: `hop`

§5.3 (`:569-579`), matched by `movement.py:142-158` inside `compute_hop_proof`:

1. `previous_event_id` from the `e` tag marked `previous`, parsed from lowercase hex (validated at `movement.py:143-150`).
2. `t = int.from_bytes(previous_event_id_bytes, "big") % (1 << 85)`.
3. `t_base = (t >> K) << K`; `cantor_t = compute_subtree_cantor(t_base, K)`.

### 4.3 Work seed: `sidestep`

§6.7 (`:1086-1100`): temporal binding "is **identical** to hop proofs (§5.2–§5.3)" (`:1088`). Implemented in `_sidestep_binding` (`movement.py:549-559`).

**Changed since `919494a`:** the old §6.7 note that an entity MAY precompute the spatial Merkle root is gone. Leaves are now seeded with `previous_event_id` (§6.4), so "the spatial Merkle root cannot be computed in advance of a crossing. An entity MUST perform the full spatial work between publishing its previous event and publishing the sidestep" (`:1098`). The temporal axis is retained but "no longer load-bearing for replay" (`:1096`).

### 4.4 Work seed: `enter-hyperspace` (DECK-0001)

DECK §3.2 (`:221-230`): `K` at `C` per base §5.2 (plane included); `t`, `t_base`, `cantor_t` per §5.3; `enter_n = π(0, cantor_t)`; `proof_hash = sha256(sha256(int_to_bytes_be_min(enter_n)))`. DECK §7 (`:412-414`) reserves `CYBERSPACE_ENTER_HYPERSPACE_V1` for a future boarding toll.

### 4.5 Work seed: `hyperjump` (DECK-0001)

DECK §5.3 (`:322-332`). Does **not** use the base terrain function.

**Constants** (DECK §5.1, `:286-300`):
```
K_LINE                    = 6
SAMPLES                   = 32
GRIND_HEIGHT              = 16
HYPERSPACE_TERRAIN_DOMAIN = b"CYBERSPACE_HYPERSPACE_TERRAIN_V1"
HYPERSPACE_SEED_DOMAIN    = b"CYBERSPACE_HYPERSPACE_SEED_V1"
HYPERSPACE_LEAF_DOMAIN    = b"CYBERSPACE_HYPERSPACE_LEAF_V1"
HYPERSPACE_GRIND_DOMAIN   = b"CYBERSPACE_HYPERSPACE_GRIND_V1"
HYPERSPACE_SAMPLE_DOMAIN  = b"CYBERSPACE_HYPERSPACE_SAMPLE_V2"
PAD_LEAF                  = 32 zero bytes
```

With `lo = min(B_from, B_to)`, `hi = max(...)`, blocks `b = lo+1 .. hi`, `n = hi - lo`:

1. **Line terrain.** `digest = sha256(HYPERSPACE_TERRAIN_DOMAIN || H_b)`; `K_b = popcount((digest[0] << 8) | digest[1])`, in `[0, 16]`.
2. **Height.** `h_b = K_b + K_LINE`, in `[6, 22]`.
3. **Seed.** `t_b = int(sha256(HYPERSPACE_SEED_DOMAIN || previous_event_id || be64(b))) mod 2^85`.
4. **Work.** `cantor_t_b = compute_subtree_cantor((t_b >> h_b) << h_b, h_b)`.
5. **Leaf.** `leaf_b = sha256(HYPERSPACE_LEAF_DOMAIN || be64(b) || int_to_bytes_be_min(cantor_t_b))`.

Every leaf shares one `previous_event_id`, so all blocks of a ride compute in parallel.

**Merkle root** (DECK §5.4, `:334-336`), padded with `PAD_LEAF` to a power of two.

**Re-roll price and openings** (DECK §5.5, `:338-376`): `G` comes from one height-16 Cantor tree per attempt over `seed_nonce = sha256(HYPERSPACE_GRIND_DOMAIN || previous_event_id || root || be64(nonce))`; `A = max(1, ceil(n / SAMPLES))`; the `mn` nonce must satisfy `G × A < 2^256` (`:344-351`). Sample indices `idx_i = int(sha256(HYPERSPACE_SAMPLE_DOMAIN || G || be32(i))) mod n` (`:355-361`). `mp` is the `SAMPLES` paths joined by `:` (`:363`).

**No zero-length ride** (DECK §5.6, `:378-386`): the zero-length ride of the old revision is removed; `B` MUST NOT equal `from_height`.

**Ride openings version 2** (DECK §5.8, `:392-400`): rides without `mn` are invalid except the 16 ids in `decks/grandfathered-v1-hyperjumps.txt`, whose root and openings are accepted without re-checking.

### 4.6 Actions with no work seed

- `spawn`, `enter-virtual`, `exit-virtual` and virtual actions: no proof (§8.8, `:1943-1947`; §8.11.2, `:2088`).
- Bags (kind 33330): no movement proof. Region keys are **non-temporal** (§6c below).
- Block anchors (kind 321): validated against Bitcoin, not Cantor or Merkle work.

---

## 5. Canonical integer encoding and proof hashes

### 5.1 `int_to_bytes_be_min` (normative, §5.5, `:590-602`)

Verbatim from the spec:

```python
def int_to_bytes_be_min(n: int) -> bytes:
    if n < 0:
        raise ValueError("expected non-negative")
    if n == 0:
        return b"\x00"
    # minimal big-endian bytes
    return n.to_bytes((n.bit_length() + 7) // 8, "big")
```

The `n == 0` case returns one `0x00` byte, **not** the empty string. CLI: `cantor.py:11`. Distinct from the fixed-width 32-byte encoding of coordinates in the terrain preimage (`terrain.py:76`).

### 5.2 Hop proof hash, version 1 (§5.4, §5.6, `:583-614`)

- `hop_n = π(region_n, cantor_t)` (§5.4, `:586`; `movement.py:161`).
- `proof_hash = sha256(sha256(int_to_bytes_be_min(hop_n)))`, lowercase hex (§5.6, `:608-612`; `movement.py:164-166`).

§5.6 now labels this **version 1** (`:614`). Under the current chain rules a hop's `proof` carries the version 2 seal instead. The definition stays because grandfathered v1 hops carry it, §5.7's worked example locks `K`, `cantor_t` and `hop_n` with it, and the sidestep proof hash is built the same way.

### 5.3 Sidestep proof hash (§6.8, `:1102-1118`)

`sidestep_n = π(region_m, cantor_t)`; `proof_hash = sha256(sha256(int_to_bytes_be_min(sidestep_n)))`, lowercase hex (`movement.py:558-559`). Unchanged; sidesteps still carry this in `proof`.

### 5.4 Enter-hyperspace proof hash (DECK §3.2) and ride proof

Enter-hyperspace: `sha256(sha256(int_to_bytes_be_min(π(0, cantor_t))))`. Ride: the `proof` tag is the raw Merkle root over per-block leaves (DECK §5.4).

### 5.5 Hop proof version 2: residue openings (§5.9 to §5.13)

Normative: §5.9 (`:678-766`), §5.10 (`:768-828`), §5.11 (`:830-868`), §5.13 (`:912-929`); event format §8.4 (`:1778-1795`); verification §8.7.1 (`:1861-1876`).

- **What stays.** The axis trees, `region_n`, `K`, `cantor_t` and `hop_n` are exactly as before (`:682`). Only the commitment changes.
- **Constants** (`:694-710`): `HOP_RESIDUE_PRIME_DOMAIN`, `HOP_RESIDUE_LEAF_DOMAIN`, `HOP_RESIDUE_GRIND_DOMAIN`, `HOP_RESIDUE_SAMPLE_DOMAIN` (all `_V1`); `HOP_SAMPLES = 16`; `RESIDUE_BLOCK = 8`; `PRIME_BITS = 61`; `WIDTH_DIVISOR = 4`; `HOP_GRIND_HEIGHT = 12`; `SMALL_ODD_PRIMES`; `HOP_REROLL_BUDGET` (heights 0 to 22). CLI: `residue.py:62-77`.
- **Hop primes** (`:730-740`): `p_j` derived from `SHA256(HOP_RESIDUE_PRIME_DOMAIN || previous_event_id || be64(j))`, a base-2 strong probable prime in `[2^61, 2^62)`. Fresh primes per hop, so residue tables cannot be reused or aimed (`:761-766`).
- **Residue count** (`:742-749`): `m = max(8, 8 × ceil(ceil(85 × 2^h_max / 244) / 8))`, so `Q` spans a quarter of the widest root. `r_j = hop_n mod p_j` (`:751`), computed with pairing modulo `p` (`:714-726`).
- **Seal** (`:770-777`): blocks of 8 residues hashed as leaves with the previous event id and block number, padded Merkle tree; the root is the `proof` tag.
- **Re-roll price** (`:781-798`): each attempt is one height-12 Cantor tree modulo `p_0`, seeded by `SHA256(HOP_RESIDUE_GRIND_DOMAIN || previous_event_id || seal || be64(nonce))`; price `A = max(1, B(h_x) + B(h_y) + B(h_z) + B(K))`; publish `mn` with `G × A < 2^256`. "a production prover MAY split the search across workers" (`:798`).
- **Draw and openings** (`:804-816`): 16 indices from `G`; each opening is its block's 8 residues plus `depth` siblings.
- **Golden vectors** (`:818-828`): `hop-residue-reference.py` MUST be reproduced exactly; `hop-residue-vectors.json` is the non-normative machine-readable copy.
- **Level 1 verification** (§5.11, `:832-841`; §8.7.1 steps 1 to 11, `:1863-1876`): derive everything from the event, replay one attempt, derive 16 indices, recompute 16 residues in word-sized arithmetic, climb 16 paths. About 4 ms per hop at h20. A verifier that skips the price check accepts the divisible-work forgery (`:1876`).
- **Cost** (§5.12, `:870-910`): producing a v2 hop costs about 1.16x to 1.22x a v1 hop; verifying is hundreds of times cheaper than producing.
- **Grandfathered v1 hops** (§5.13, `:912-929`): verifiers MUST reject v1 hops except those listed in `grandfathered-v1-hops.txt` (`:918`); a listed hop's `proof` is accepted without recomputation and everything else is checked (`:920`); no date cutoff (`:921`, `:929`). At `4cb8f1b` the file lists **2,658** ids from census 1 (read 2026-10-31; header of the file), last changed in `957021e`. §8.7.1 step 7 (`:1870`) applies the list.

### 5.6 Sidestep construction versions 2 and 3 (§6.4, §6.10, §6.15, §6.16)

- **Seeded leaves (v2)** (§6.4, `:1000-1035`): `seed_prefix = SIDESTEP_DOMAIN || previous_event_id || axis_byte || SEED_PAD`, exactly 64 bytes; `SIDESTEP_DOMAIN = b"CYBERSPACE_SIDESTEP_V2"` (`:1025`); `H_i = SHA256(seed_prefix || int_to_bytes_be_min(base + i))`. CLI: `movement.py:209` (domain), `:226-234` (`seed_prefix`).
- **Openings (v3)** (§6.10 "Openings", `:1129-1172`): re-roll price `G = SHA256(SIDESTEP_GRIND_DOMAIN || be64(nonce) || previous_event_id || M_x || M_y || M_z)` with `A = max(1, ceil(L / SIDESTEP_SAMPLES))` over total leaves `L` (`:1141-1147`); per axis, the destination path plus `SIDESTEP_SAMPLES = 8` sampled paths with indices from `G` (`:1151-1166`). Constants `SIDESTEP_GRIND_DOMAIN = b"CYBERSPACE_SIDESTEP_GRIND_V1"`, `SIDESTEP_SAMPLE_DOMAIN = b"CYBERSPACE_SIDESTEP_SAMPLE_V2"` (`:1159-1164`).
- **Verification** (§6.11, `:1174-1210`; §8.7.2, `:1878-1897`).
- **Version 2 rules** (§6.15, `:1271-1283`): v1 sidesteps are rejected with no grace period; three v1 events by two identities were invalidated at adoption.
- **Version 3 rules** (§6.16, `:1285-1296`): `mn` required; v2 sidesteps rejected except those listed in `grandfathered-v2-sidesteps.txt` (36 ids at `4cb8f1b`), whose roots and openings are accepted unchecked while geometry, linkage and `proof_hash` are still checked (`:1292`).
- **Hardware** (§6.14, `:1253`, `:1267`): sidestep work is plain SHA-256; "SHA-NI or a GPU is 50 to 10,000× faster for sidesteps"; "subject to ASIC acceleration". §12.3 (`:2552`) lists ASIC-accelerated sidesteps as an acknowledged attack vector.
- **Handed roots** (§6.12, `:1220-1234`): a recipient handed all three axis roots saves "about 19% of the work" of a hop (`:1232`); `cantor_t` is per traveller.

---

## 6. Region/sector alignment and how region roots are computed

### 6a. Sectors (§10, `:2446`)

- A sector is a cube of `2^30` Gibsons per axis (`:2448`): `sx = x >> 30` etc. CLI: `SECTOR_BITS = 30` (`cli.py:253`); `SECTOR_BITS_DEFAULT = 30` (`sector.py:9`); `xyz_to_sector_id` (`sector.py:28-31`).
- Every event that claims a coordinate MUST carry `X`, `Y`, `Z`, `S` in the formats of §10 "Tag formatting rules" (`:2467`). On movement events each sector tag now MUST appear exactly once and match `C` (§8.3, `:1776`, and likewise §8.4, §8.5).
- CLI generates them in `_sector_tags_from_coord_hex` (`nostr_event.py:77-89`); the `sector` command uses a local `sector_bits = 30` (`cli.py:863`).
- DECK-0001 §1.3 (`:98-102`): sector tags for stop coordinates come from `C`, not the merkle root.

### 6b. Aligned regions and their Cantor/Merkle roots

- **Aligned subtree** (§4.5, `:383-414`) and **LCA height** (§4.4, `:357-381`): `h = find_lca_height(v1, v2) = (v1 ^ v2).bit_length()`, `base = (v1 >> h) << h`.
- **Axis Cantor root** (§4.2, `:332-340`; §4.6, `:416-446`): `compute_subtree_cantor` (`movement.py:28-52`), `compute_axis_cantor` (`:55`); `cantor_pair` (`cantor.py:6`). §4.6 now notes that at height 0 a root is the position itself, so roots are unique across heights only for heights of 1 and above (`:444-446`).
- **3D combine for hops** (§4.7, `:448-472`): `region_n = π(π(cantor_x, cantor_y), cantor_z)`. CLI: `movement.py:136`; `location_encryption.py:21-38` (`derive_region_n_for_height`).
- **Axis Merkle root for sidesteps**: seeded per §5.6 above; streaming algorithm §6.5 (`:1037-1069`), CLI `compute_axis_merkle_root_streaming` (`movement.py:376`), `merkle_leaf` (`:251`), `merkle_parent` (`:256`).
- **3D combine for sidesteps** (§6.6, `:1071-1084`): `region_m = π(π(mx, my), mz)`; "There is no mixed mode" (`:1084`). CLI `movement.py:553`.
- **Multi-axis sidestep** (§6.9, `:1120-1127`): each crossing axis moves exactly 1 Gibson; temporal binding once; total work is the sum.
- **Inclusion paths** (§6.10, `:1133-1139`): sibling hashes from leaf to root; left/right from the leaf position.
- **A root prefix is not evidence of possession** (§4.9, `:510-518`): low bits of a root are cheap to compute and prove nothing.

### 6c. Region root to the exact bytes hashed (§7.2, normative, `:1316-1330`)

- `region_bytes = int_to_bytes_be_min(region_n)`; no padding, no domain prefix, no length prefix.
- `location_decryption_key = sha256(region_bytes)`; `lookup_id = sha256(location_decryption_key)` (`:1320-1322`); lowercase hex in tags (`:1328`).
- "Seeing `lookup_id` ... does not allow deriving `location_decryption_key` without the region preimage." (`:1324`)
- Not temporally bound (`:1326`). A region and its twin share the key (`:1330`).
- CLI: `derive_region_keys_from_region_n` (`location_encryption.py:42-46`), `derive_region_key_material_for_height` (`:49`).

**Discovery scanning** (§7.4, `:1354-1371`): heights `h = 1..H`, aligned bases, per-axis roots, `region_n`, then `lookup_id`. **Radius and caching** (§7.3, `:1332-1352`; §7.5, `:1373`).

### 6d. `cyberspace cantor` naming vs spec naming

The `cantor` command (`cli.py:1309-1441`) prints `region_n` as `cantor_number`, then `encryption_key_sha256` and `discovery_id_sha256_sha256` (`cli.py:1440-1441`): the spec's `location_decryption_key` and `lookup_id`.

---

## 7. Hyperspace stop geometry (DECK-0001 v3)

**Line, stop, port, landfall, station, ride, exit, block anchor** are defined in DECK "Terms" (`:32-41`).

### 7.1 Plane selection (DECK §1.1, `:47-58`)

`plane = merkle_root_int & 1`, with `merkle_root_int` read from the standard display hex. Plane 1: **port**, `C = merkle_root_int`. Plane 0: **landfall**, `C` from the block hash (§1.2). Miners cannot steer a block hash except by discarding valid blocks (`:58`).

### 7.2 Landfall derivation (DECK §1.2, `:60-96`)

`LANDFALL_DOMAIN = b"CYBERSPACE_LANDFALL_V1"` (`:62`); arithmetic in the decimal context of `CYBERSPACE_V2.md` §9.5 (`:66`); steps 1 to 11 at `:68-79` (seed, uniform sphere direction, WGS84 surface intersection, ECEF km, axis permutation per §9.4, u85 conversion per §9.7 step 9, `xyz_to_coord(..., plane = 0)`). Golden vectors: eight mainnet heights at `:85-94`. Reference: `decks/landfall-reference.py` (5,029 bytes, unchanged), not executed here. See **UNKNOWN**.

### 7.3 The station (DECK §4, `:241-278`)

**Distance** (§4.1, `:245-253`): `d(p, q) = max(find_lca_height(px, qx), find_lca_height(py, qy), find_lca_height(pz, qz))`, plane bits ignored.

**Definition** (§4.2, `:255-266`): `station(C_e, A)` is the stop in `Stops(A)` minimising `d(C_e, C_s)`, ties to the lowest height, where `A` is the `as_of` bound on the first ride; `A` MUST exist on Bitcoin mainnet and be `≥ B_to`.

**Position semantics** (§3.3, `:232-237`): after `enter-hyperspace` the identity is still at `C`; it moves only with a ride; a `hop` or `sidestep` right after boarding cancels it; boarding at a stop makes that stop the station, and the first ride must still leave it (no zero-length ride).

---

## 8. Exact output format of each CLI command

All output is via `typer.echo(...)`, to stdout unless `err=True` is noted (stderr). Most calls print one newline-terminated line; a few pass a string with embedded newlines and so print several lines in one call (`format_quote` in `cloud_move.py`, the commit message in `viz/move_viz.py`). Lines below are literal f-string templates as they appear in `cli.py`/`cloud_cli.py`/`cloud_move.py`/`targets.py`/`viz/move_viz.py` at `cyberspace-cli@4e5dab7`, with source lines cited.

The complete command surface at `4e5dab7` is 41 commands. 32 come from the Typer decorators in `cli.py` on `app`, `chain_app`, `config_app`, `target_app`, `hyperjump_app`, `geoid_app` and `move_app` (declared `cli.py:93-99`, mounted with `add_typer` at `cli.py:100-110`); `move` itself counts once, as its `@move_app.callback()` (`cli.py:2417`). The other 9 are the HOSAKA cloud group on `cloud_app`, declared in `cloud_cli.py:26` and mounted as `cloud` at `cli.py:110`. No other module under `src/cyberspace_cli/` defines Typer commands. All 41 are covered below.

**`cyberspace spec`** (`cli.py:770-786`): attempts `webbrowser.open("https://github.com/arkin0x/cyberspace/blob/master/readme.md", new=2)`, then **always** prints that URL on its own line; if the open failed or raised, additionally prints `(Could not auto-open browser; URL printed above.)` to stderr (`cli.py:786`).

**`cyberspace help`** (`cli.py:764-767`): prints `HELP_TEXT.strip()`, the module-level raw string in `cyberspace_cli/helptext.py` (109 lines; the string opens at `helptext.py:1` with `HELP_TEXT = r'''` and closes at `:109`). `.strip()` removes only the leading and trailing newlines of the raw literal; the interior is emitted byte-for-byte. Content is a prose usage guide: title "Cyberspace CLI (v2)", a note that relay publishing is intentionally not implemented (`:4-5`), a "Quick start" venv/pip block (`:7`), a command list (`:42`), and a closing "Security note" (`:106`) that the private key is stored locally in plaintext ("Treat it like a hot wallet key"). Since `2dff988` it names `wss://relay.onosendai.tech` as the example relay, describes the DECK-0001 v3 `hyperjump show` options, and says `encrypt` writes the height tag as `H`. It does not mention the `cloud` group, and it still describes `move --hyperjump` and `hyperjump to` as moving (`:61`, `:74`) although both now refuse (see below). Not reproduced in full here; it is a static literal in the repo.

**`cyberspace spawn`** (`cli.py:789-835`), in order:
```
Spawned.
chain: <label> (len=1)
npub: <bech32 npub1... string>
nsec: <bech32 nsec1... string>
coord: 0x<64-hex coord>
plane=<0 or 1> <dataspace|ideaspace>
```
(`cli.py:828-835`; npub/nsec are NIP-19 bech32 encodings per `nostr_keys.py`'s `encode_npub`/`encode_nsec` (`nostr_keys.py:142`, `:135`), charset `"qpzry9x8gf2tvdw0s3jn54khce6mua7l"` at `nostr_keys.py:7`.)

**`cyberspace whereami`** (`cli.py:838-852`):
```
coord: 0x<64-hex coord>
pubkey: <64-hex pubkey>
active_chain: <label or (none)>
xyz(u85):
x=<int>
y=<int>
z=<int>
plane=<0 or 1> <dataspace|ideaspace>
```
(`cli.py:845-852`)

**`cyberspace sector`** (`cli.py:855-873`):
```
sector:
X=<sx>
Y=<sy>
Z=<sz>
plane=<0 or 1> <dataspace|ideaspace>
S tag: <sx>-<sy>-<sz>
```
(`cli.py:868-873`)

**`cyberspace bench`** (`cli.py:890-956`): a header, one line per LCA height tested, then a summary:
```
bench: movement proof runtime by LCA height
timeout_s=<n> target_s=<n>
current_default_max_lca_height=<n>
lca_height=<h>: <elapsed>s
... (repeated per height, or "lca_height=<h>: >Ns (cancelled)" / "lca_height=<h>: error: <msg>" on failure)
---
Optimal Speed LCA: <h> (<t>s)
Recommendation: set --max-lca-height to <h> for interactive speed.
Persist: cyberspace config set --max-lca-height <h>
Per-command override: cyberspace move --max-lca-height <h> ...
```
(`cli.py:909-956`; the benchmark still times `compute_movement_proof_xyz`, the version 1 Cantor proof, not the version 2 `prove_hop` that `move` now uses.)

**`cyberspace gps <lat,lon>` / `--coord`** (`cli.py:1145-1306`):
- GPS to coord: `coord: 0x<hex>` then `xyz(u85): x=<x> y=<y> z=<z> plane=<p> <label>` (`cli.py:1305-1306`; all xyz fields on one line for this direction).
- coord to GPS (`--coord`): `coord: 0x<hex>`, then `xyz(u85): x=<x> y=<y> z=<z> plane=<p> <label>` on one line, then `gps: lat=<%.10f> lon=<%.10f> alt_m=<%.3f>` (`cli.py:1222-1224`).

**`cyberspace cantor`** (`cli.py:1309-1441`): `from:` / `to:` blocks (`coord:`, `xyz:` sub-lines), then per axis `X`, `Y`, `Z` a block of `axis <name>:`, `v1=... v2=...`, `lca_height=...`, `subtree_base=...`, `subtree_range=[...] leaves=...`, then either the full per-level Cantor tree (`level_<n> (<count> nodes): [hex, hex, ...]`, only if `lca_height <= --max-height`) or `tree_levels: omitted (height <h> > --max-height <n>)`, followed by `cantor_root_hex=...` and `cantor_root_bytes=<n>`; finally a `combined:` block with `cantor_number_hex=...`, `cantor_number_bytes=<n>`, `encryption_key_sha256=<hex>`, `discovery_id_sha256_sha256=<hex>`. (`cli.py:1383-1441`)

**`cyberspace encrypt`** (`cli.py:1443-1511`): exactly one line, the created Nostr-style event as compact JSON (`json.dumps(event, separators=(",", ":"), ensure_ascii=False)`). (`cli.py:1511`) The tags are `d`, `encrypted`, `version` and, only with `--publish-height`, the height tag, which is now written as `["H", "<height>"]` (`nostr_event.py:186`, `:231-237`), not lowercase `h` as at `2dff988`.

**`cyberspace decrypt`** (`cli.py:1514-1589`):
```
height: <chosen_height>
lookup_id: <d_tag value>
<decoded plaintext, or base64 of raw bytes if not valid UTF-8>
```
(`cli.py:1584-1589`; without `--height` it tries the height the event's tags state first, `H`, or a legacy `h` when no `H` is present, via `height_from_tags`, then scans `0..--max-height`, `cli.py:1545-1552`.)

**`cyberspace scan`** (`cli.py:1592-1660`): one summary line per height, `h=<height> d=<lookup_id_hex> matches=<n>`, followed by an indented `  id=<id> pubkey=<pubkey> kind=<kind>` line per matching event (only present when `--events-file` is used). (`cli.py:1658-1660`)

**`cyberspace config show`** (`cli.py:614-622`):
```
default_max_lca_height: <int>
gps_geoid_model: <str>
cloud_mode: <auto|ask|off>
cloud_api_url: <url>
cloud_auto_max_sats: <int>
```
(The last three are new since `2dff988`. Defaults `auto`, `https://arkin0x--hosaka-api-api-server.modal.run` and `0`, `config.py:17-19`.)

**`cyberspace config set --max-lca-height N`** (`cli.py:625-639`): `Saved.` then `default_max_lca_height: <int>`.

**`cyberspace config set-geoid-model MODEL`** (`cli.py:642-660`): `Saved.` then `gps_geoid_model: <str>`.

**`cyberspace geoid doctor`** (`cli.py:661-717`): `config_default_model=<...>`, `effective_model=<...>`, `search_dirs:` followed by one path per line, then per checked model (an empty line before every model after the first) `model=<name>` and either (`status=available`, `path=...`, `grid=<w>x<h>`, `scale=<...>`, `offset=<...>`, optionally `size_bytes=<n>`) or (`status=missing`, `candidates:` + one path per line) or (`status=invalid`, `error=<...>`). (`cli.py:682-717`)

**`cyberspace target set <coord>`** (`cli.py:720-733`): `(current) <label> 0x<coord_hex>`.

**`cyberspace target use <label>`** (`cli.py:736-749`): `active_target: <label>`, or (stderr) `Unknown target label: <label>` with exit code 1.

**`cyberspace target list`** (`cli.py:752-761`, formatting in `targets.py:90-100` `format_target_list`): `(no targets yet)` if empty; otherwise one line per target, `<label> 0x<coord_hex>`, with a `(current) ` prefix on the active one.

**`cyberspace history`** (`cli.py:2961-3002`):
- `--json`: one line per event, each the compact-JSON event object, nothing else.
- default: `chain: <label> (showing <n> events)` header, then per event `<0000> <action:<5} id=<id> coord=0x<coord>` with ` proof=<proof>` appended if a `proof` tag is present (`cli.py:2992-3002`; index is Python `{i:04d}`, action field is `{action:5}`).
- Empty chain, non-JSON: `(empty chain) <label>` (`cli.py:2980`).

**`cyberspace chain list`** (`cli.py:3005-3018`): `(no chains yet)` if none; else one line per chain, `<*|space> <label> (len=<n>)`, `*` marking the active chain.

**`cyberspace chain use <label>`** (`cli.py:3021-3032`): `active_chain: <label>`, or stderr `Unknown chain: <label>` exit 1.

**`cyberspace chain status`** (`cli.py:3035-3070`):
```
active_chain: <label>
length: <n> (hops=<n-1, min 0>)
genesis: <event id>
last:    <event id>
spawn:   0x<coord_hex>
current: 0x<coord_hex>
[warning: state coord != last chain coord   -- stderr, only if mismatched]
delta_xyz: dx=<int> dy=<int> dz=<int> (plane=<p> <label>)
```
(`cli.py:3062-3070`; empty chain instead prints `(empty chain) <label>`, `cli.py:3043`.)

**`cyberspace move --by / --to`**: single hop, printed by `_do_single_hop` (`cli.py:2611-2779`, output `cli.py:2763-2777`):
```
Moved. chain=<label> len=<n>
coord: 0x<coord_hex>
```
then, if a sidestep proof was used:
```
action: sidestep
proof: <proof_hash>
terrain_k: <K>
lca_heights: X=<hx> Y=<hy> Z=<hz>
```
or, if a hop proof was used:
```
proof: <64-hex seal>
terrain_k: <K>
```
The hop proof is now version 2 (residue openings, `prove_hop`, `cli.py:2705-2711`); the printed `proof:` is the seal that goes in the `proof` tag (`HopResidueProof.proof`, `cyberspace_core/residue.py:549-551`). The `mn` nonce and `mp` openings go into the event but are not printed. If the proof came from HOSAKA, two or four more lines follow:
```
source: HOSAKA cloud job <job_id> (<cost rounded up to whole sats> sats)
lookup_id: <lookup_id>                      -- only if the cloud object has one (a cloud hop)
location_decryption_key: kept in the cloud job record (cyberspace cloud status <job_id>)
```
(`cli.py:2773-2777`). The old third variant (`action: hyperjump` / `B: <height>`) is gone: `--hyperjump` now refuses before anything is computed (`cli.py:2597-2598`) with the `_refuse_to_ride` message on stderr, exit 2 (`cli.py:1779-1795`), which begins `This CLI cannot ride hyperspace yet: a ride (DECK-0001 v3) has to follow an enter-hyperspace action, ...` and ends `... so nothing was appended.` The hidden `--hyperjump-height` option still parses but is unused (`cli.py:2497-2502`, `:2533-2534`).

**HOSAKA cloud path inside `move`.** When a hop's tallest LCA height exceeds the local limit and cloud mode is not `off` (`cli.py:2662`), `run_cloud_move` (`cloud_move.py:152-254`) prints, before the `Moved.` block above, in order:
```
HOSAKA cloud <hop|sidestep>: <N sats|n/a> (<msats> msats)
  tier: <tier>  expected: <est_time>
  LCA heights: X=<hx> Y=<hy> Z=<hz> (cap <cap>)
  destination: x=<x> y=<y> z=<z> plane=<p>
  a sidestep lands 1 gibson past the wall; the rest of the journey continues from there   -- sidestep only
[Pay <sats> sats for this cloud <action>? [y/N]:     -- typer.confirm prompt, only when approval is needed]
HOSAKA job <job_id> (resume with: cyberspace cloud resume <job_id>)
```
(`format_quote`, `cloud_move.py:101-111`, emitted as one call at `:201`; prompt `:127`; job line `:219`.) If the job needs payment: `Pay <sats> sats to fund this job (invoice expires in <s> s):`, an ASCII QR code only if the optional `qrcode` package imports, `<bolt11>`, `lightning:<bolt11>` (`cloud_move.py:131-149`), then `  waiting for payment... <s> s left on the invoice` at most every 15 s (`:230`) and `Paid. Starting the job.` (`:233`). While computing: `  computing... <s> s elapsed` with `, about <pct>%` appended when the server reports progress, at most every 15 s (`:247`). After the result checks pass: `Verified cloud <action> (0 failed checks).` (`:291`). On Ctrl+C while waiting: `\nInterrupted. The job is safe on the server; finish it with: cyberspace cloud resume <job_id>` (`:251`), then exit 130 (`cli.py:2687-2688`). Failures, all stderr: `Cloud hop not submitted: <reason>` exit 2 (`cli.py:2677`), `HOSAKA result failed verification; nothing was appended:` plus `  - <failure>` per failed check, exit 3 (`cli.py:2680-2683`), `HOSAKA: <error>` exit 2 (`cli.py:2685`). With cloud mode `off` and a hop too tall, the refusal is the stderr `Move is too large for a single hop. LCA heights: ...` message, exit 2 (`cli.py:2694-2703`).

**`cyberspace move --toward`**: per-hop progress in the loop (`cli.py:2781-2910`): for a final plane-switch hop, after the `_do_single_hop` output, prints `hop: <n>`, `x=<x>`, `y=<y>`, `z=<z>`, `plane=<p> <label>` (`cli.py:2815-2819`); for an ordinary progress hop, `hop: <n>`, `x=<x1> remaining=<tx-x1>`, `y=<y1> remaining=<ty-y1>`, `z=<z1> remaining=<tz-z1>`, `plane=<p> <label>`, and if a proof was produced, `lca_height: X=<hx> Y=<hy> Z=<hz> limit=<hop_limit>` and `terrain_k: <K>` (`cli.py:2899-2906`). On arrival: `Arrived.` then `coord: 0x<coord_hex>` (`:2794-2795`). On `--max-hops`: `Stopped after max_hops=<n>.` then `coord: 0x<coord_hex>` (`:2799-2800`). On Ctrl+C: stderr `Interrupted after hops=<n>.` then `coord: 0x<coord_hex>`, exit code 130 (`:2908-2910`). When a hop requires temporarily raising the LCA-height limit, a stderr line precedes it: `LCA boundary encountered on axis <A[,B[,C]]>; temporarily increasing max_lca_height from <old> to <new> for this hop.` (`:2881-2886`); with cloud mode on and no `--sidestep`, `<new>` is capped at the local limit plus one, and taller crossings go to HOSAKA inside `_do_single_hop` (`:2877-2880`). If HOSAKA cannot be reached when the loop asks for its sidestep cap, stderr `HOSAKA unreachable (<error>); continuing without cloud compute.` (`cli.py:2561`).

**`cyberspace hyperjump show <blockheight>`** (`cli.py:1662-1776`). Rewritten since `2dff988` for DECK-0001 v3 stops. Options `--relay`, `--limit`, `--verbose/-v`, `--source relay|mempool`, `--block-hash`, `--merkle-root`, `--from`, `--from-gps`. Output (`cli.py:1751-1760`):
```
hyperjump_block_height=<n>
deck=DECK-0001 v3 (2026-08-24)
stop_kind=<port|landfall> (<ideaspace|dataspace>)
coord: 0x<coord_hex>
x=<x>
y=<y>
z=<z>
plane=<p> <label>
merkle_root=<hex>          -- only if the stop carries a merkle root
block_hash=<hex>           -- only if the stop carries a block hash
lat=<%.6f>                 -- these four only for a landfall (plane 0)
lon=<%.6f>
alt_m=0 (WGS84 ellipsoid surface)
maps=https://www.google.com/maps?q=<lat %.6f>,<lon %.6f>
```
(`deck=` value is `DECK_0001_VERSION`, `cyberspace_core/hyperspace.py:52`; the stop block is `_print_stop`, `cli.py:543-560`; a port is plane 1, a landfall plane 0, `hyperspace.py:55-56`.) Then, for a relay anchor, `anchor_format=v3 (C verified against M/H)` or `anchor_format=legacy (no M tag; C read as merkle root and the stop derived locally per DECK-0001 v3 s2.2)`, and `event_id=<id>` if the id is non-empty; for a stop derived without an anchor, `source=offline (derived locally; not an anchor event)` (with `--block-hash`/`--merkle-root`) or `source=mempool.space (derived locally; not an anchor event)` (with `--source mempool`). With `--from-gps` or `--from`, a distance block follows (`_print_stop_distance`, `cli.py:577-598`):
```
from=<gps <lat %.6f>,<lon %.6f> (WGS84 surface) | 0x<coord_hex>>
from_x=<x>
from_y=<y>
from_z=<z>
delta_gibsons: X=<+dx> Y=<+dy> Z=<+dz>
delta_km: X=<+%.6f> Y=<+%.6f> Z=<+%.6f>
lca_heights: X=<hx> Y=<hy> Z=<hz>
stop_distance_d=<d> (DECK-0001 v3 s4.1; smallest aligned cube containing both, side <%.3f> km)
straight_line_km=<%.3f>
geodesic_km=<%.3f> (WGS84 surface distance)                       -- landfall, and the origin has a GPS position
geodesic_km=n/a (port in ideaspace has no surface position)        -- port
```
(With `--from` the origin has a GPS position only when both it and the stop are in plane 0, `cli.py:1773-1775`; a landfall with no origin GPS prints no `geodesic_km` line.) Failures, stderr: `No hyperjump found for block height <n>. Try --source mempool, or --block-hash/--merkle-root for an offline derivation.` exit 1 (`cli.py:1743-1748`); `mempool.space lookup failed for height <n>: <error>` exit 1 (`:1732`); `Cannot derive stop: <error>` exit 2 (`:1726`).

**`cyberspace hyperjump to/next/prev`** (`cli.py:1798-1848`, `:1851-1898`, `:1901-1952`). Without `--view` all three now refuse at once with the `_refuse_to_ride` message on stderr, exit 2 (`cli.py:1835-1836`, `:1883-1884`, `:1933-1934`); they no longer emit a `_do_single_hop` block. With `--view` they print the anchor via `_print_hyperjump_anchor` (`cli.py:563-574`):
```
hyperjump_block_height=<n>
<the _print_stop block above, stop_kind= through maps=>
anchor_format=v3 (C verified against M/H)    -- or the anchor_format=legacy line above
event_id=<id>          -- only if the anchor event has a non-empty id
```
`next`/`prev --view` first require the active chain to end on a hyperjump; otherwise stderr `Your chain does not end on a hyperjump, so there is no block to step from (this CLI cannot ride yet).` exit 2 (`cli.py:453-457`). Not found: stderr `No hyperjump found for block height <n>.` exit 1. `prev` below 0: stderr `No previous hyperjump exists before block height 0.` exit 2 (`:1938`).

**`cyberspace hyperjump nearest`** (and cache/expand modes) (`cli.py:2246-2414`), via `_print_ranked_hyperjumps` (`cli.py:2034-2070`):
```
current: 0x<coord_hex>
x=<x>
y=<y>
z=<z>
plane=<p> <label>
search_radius=<n>        -- only present when a radius was used (expand/fixed-radius modes)
nearby_hyperjumps: <count>
```
then, for each ranked result (1-indexed):
```
<i>. id=<event_id>
coord=0x<coord_hex>
stop_kind=<port|landfall>
B=<block_height or "?">
x=<x>
y=<y>
z=<z>
plane=<p> <label>
sector_x=<sx>
sector_y=<sy>
sector_z=<sz>
sector_distance=<int>
direction=<x-hint> <y-hint> <z-hint>
suggested_move=cyberspace move --to <x>,<y>,<z>,<plane>
```
(`stop_kind=` is new since `2dff988`; `coord` is now the resolved DECK-0001 v3 stop coordinate, and anchors that do not resolve to a valid stop are dropped, `cli.py:2000-2011`.) Each direction hint is `<axis>+ (<delta>)`, `<axis>- (<delta>)`, or `<axis>= (0)` (`cli.py:419-424`, `_direction_hint`). `--count/-n N` keeps the first N results. If nothing is found: `No nearby hyperjumps found.` followed by `Hint: try --expand to progressively widen the search, or --cache with \`hyperjump sync\`.` (`cli.py:2406-2408`; the `if not expand` guard is always true there, since expand mode returns earlier, so the hint always prints); in cache mode `No hyperjumps found in cache.` (`:2341`); in expand mode `No hyperjumps found after expanding to radius=<n>. Try \`hyperjump sync\` + \`hyperjump nearest --cache\` for a global search.` (`:2373-2376`; the commands are now in backticks, not single quotes). With `--cache` and no cache file: stderr `No local cache found. Run \`cyberspace hyperjump sync\` first.` exit 1 (`:2333-2337`). Verbose only: `Searching radius=<r> ...` per expand step (`:2356`), `Clamped radius from <r> to <r'> (relay tag limit).` (`:2389`).

**`cyberspace hyperjump sync`** (`cli.py:2095-2243`): `Fetching hyperjump anchors from <relay, or wss://relay.onosendai.tech when none is named> ...` (`:2151`); with `--resume` and a cache, first `Resuming from existing cache ...` and `  Loaded <n> cached event(s).` (`:2137`, `:2149`), then `  Resuming: fetching events from timestamp <ts> back` and `  Current cache covers blocks <min> to <max>` (`:2176-2178`; the wording changed from "older than timestamp"). Then progress lines per batch, `  Batch <n>: <count> event(s), <new> new \u2014 <total> total unique so far` (`:2209-2212`; the separator is a literal U+2014 character in the f-string, written here as `\u2014`), and, verbose only, `  Batch <n>: empty \u2014 pagination complete.` (`:2193`, same character). On completion `Block range: <min> to <max>` (if any block heights parsed), `Cached <n> unique hyperjump(s) to <path>`, `(from <fetched> fetched + <existing> previously cached)`, `Estimated remaining: ~<n, thousands-separated> events` (the constant `940000` is hard-coded as the assumed line size, `cli.py:2239-2243`). If nothing found: `No hyperjump events found on the relay.` (`:2225`). Paging now ends only on an empty page or when the cursor cannot step further (`_next_sync_until`, `cli.py:2073-2092`), not on a short page.

**Relay selection and `--verbose`/`-v`** on any hyperjump-relay command. With no `--relay`, `_nak_req_events` (`cli.py:283-322`) asks `wss://relay.onosendai.tech` and falls back to `wss://cyberspace.nostr1.com` (`cli.py:242`, `:245`). If the first cannot be read, stderr `wss://relay.onosendai.tech could not be read (<error or 'nak failed'>); asking wss://cyberspace.nostr1.com` (`:311`); if it returned nothing and `-v` is set, stdout `No events on wss://relay.onosendai.tech; asking wss://cyberspace.nostr1.com` (`:317`). A named relay is asked alone. `-v` additionally prints, around each query: `req_filter:` + compact-JSON filter (`:359-360`), and after the `nak` subprocess runs, `nak_exit_code: <n>`, `nak_stdout:` (each output line, or `(empty)`), `nak_stderr:` (same) (`:380-394`). `nak` is now run with `--auth`, signing NIP-42 AUTH with the local key passed via `NOSTR_SECRET_KEY` in the environment (`:256-272`, `:343`). A non-zero `nak` exit is now an error (stderr `Nostr query failed: <stderr>` exit 1, `:306`, `:321`) unless the fallback relay is still to be tried. Also verbose only: `skipping anchor <id>: not a valid DECK-0001 v3 stop` (`:529`).

**`cyberspace move viz`** (`cli.py:3073-3082` to `viz/move_viz.py:521` `move_viz_command` to `:30` `run_move_viz`; `move_viz.py` is unchanged since `2dff988`). This is a Textual TUI; the full-screen rendering is not line-oriented and is not characterised here (see **UNKNOWN**). Everything it writes to the terminal *outside* the TUI is fixed and is reproduced here:
- No local state: stderr `No state. Use \`cyberspace spawn\` first.`, exit 1 (`move_viz.py:527-528`).
- Textual not installed: stderr `TUI visualizer dependencies not installed.` then `Install: pip install 'cyberspace-cli[viz]'`, exit 1 (`move_viz.py:38-40`).
- On commit, after the TUI exits, it echoes a single string containing literal Rich-style markup. `typer.echo` does not render markup, so it is emitted verbatim (`move_viz.py:497`):
  ```
  \n[bold green]✓ Executing move:[/]\n  dx: [cyan]<+dx with thousands separators>[/]\n  dy: [cyan]<+dy>[/]\n  dz: [cyan]<+dz>[/]\n
  ```
  It then shells out to `subprocess.run(["cyberspace", "move", "--by", "<dx>,<dy>,<dz>"], capture_output=True, text=True)` and re-emits the child's `stdout.strip()` to stdout and `stderr.strip()` to stderr (`move_viz.py:500-510`). The stdout block is therefore exactly the `cyberspace move --by` format documented above, including the HOSAKA lines if the hop went to the cloud. On a non-zero child exit it prints `\n[red]Move failed with exit code <n>[/]` (again with literal markup) and exits with the child's code (`:512-514`).
- On escape: `Cancelled.` (`:516`). Otherwise: `No changes.` (`:518`).

**`cyberspace 3d`** (`cli.py:1054-1142`) and **`cyberspace lcaplot`** (`cli.py:959-1051`) open a matplotlib/Tk GUI window (`cyberspace_cli.visualizer.app.run_app` / `visualizer.lcaplot_app.run_app`). The rendered window contents are graphical and not characterised here (see **UNKNOWN**). Their terminal output on a missing-dependency failure is fixed, four stderr lines then exit 1:
```
3D visualizer dependencies are not installed.      -- or "LCA plot dependencies are not installed."
Install extras: pip install 'cyberspace-cli[visualizer]'
System deps: you may also need python3-tk.
Import error: <exception>
```
(`cli.py:1118-1121` for `3d`; `cli.py:1032-1035` for `lcaplot`.)

**`cyberspace cloud ...`** (all in `cloud_cli.py`, new since `2dff988`). Shared failure paths: any HOSAKA error prints stderr `HOSAKA: <error>` and exits 2 (`_fail`, `cloud_cli.py:45-47`); commands that need the local identity (`quote`, `balance`, `deposit`, `claim`, `status`, `resume`) print stderr `No identity yet. Run \`cyberspace init\` first.` exit 1 when there is no state (`cloud_cli.py:29-34`). There is no `init` command; the identity comes from `cyberspace spawn`.

**`cyberspace cloud config [--mode auto|ask|off] [--api-url URL] [--auto-max-sats N]`** (`cloud_cli.py:50-76`): `Saved.` only if an option changed anything, then always:
```
cloud_mode: <auto|ask|off>
cloud_api_url: <url>
cloud_auto_max_sats: <int>
```
(`cloud_cli.py:73-76`; a bad mode prints stderr `--mode must be one of auto, ask, off` exit 2, `:61`.)

**`cyberspace cloud limits`** (`cloud_cli.py:79-87`): one `<field>: <value>` line per `Limits` field in declaration order (`hosaka.py:57-65`): `max_hop_height`, `max_sidestep_height`, `hop_min_msats`, `deposit_min_msats`, `deposit_max_msats`, `invoice_ttl_seconds`, `local_compute`, `hop_proof_version` (prints `None` when the server does not report it). Needs no identity.

**`cyberspace cloud quote (--to X | --by DX,DY,DZ[,PLANE])`** (`cloud_cli.py:90-122`): the `format_quote` block shown under the HOSAKA cloud path above (`cloud_move.py:101-111`), then `local ceiling: h<default_max_lca_height>; cloud_mode: <mode>` (`cloud_cli.py:122`). Usage errors, stderr exit 2: `Give exactly one of --to or --by.` (`:100`), `--by needs dx,dy,dz[,plane]` (`:108`).

**`cyberspace cloud balance`** (`cloud_cli.py:125-134`): `balance: <msats> msats (<sats rounded up> sats)`, then up to 10 ledger rows, each `  <created_at>  <kind:8> <delta_msats:>10> msats  <ref>` (`:132-134`).

**`cyberspace cloud deposit SATS [--wait/--no-wait]`** (`cloud_cli.py:137-160`):
```
deposit <deposit_id> for <sats> sats, expires at <expires_at>
<bolt11>
lightning:<bolt11>
Claim it after paying with: cyberspace cloud claim <deposit_id>      -- --no-wait only; then it returns
settled: <msats> msats; balance now <msats> msats                     -- default --wait, after payment
```
(`cloud_cli.py:150-160`; the wait itself prints nothing.)

**`cyberspace cloud claim DEPOSIT_ID`** (`cloud_cli.py:163-170`): `status: <status>  settled: <msats> msats  balance: <msats> msats`.

**`cyberspace cloud jobs`** (`cloud_cli.py:173-183`): `No cloud jobs recorded.`, or one line per locally recorded job, `<job_id>  <state:16> <action:8> -> x=<x> y=<y> z=<z>  chain=<label>`, unfinished jobs first (states other than `appended`, `abandoned`, `failed`, `expired`), newest first within each group. Reads only the local record; no network.

**`cyberspace cloud status JOB_ID`** (`cloud_cli.py:186-210`):
```
status: <status>  cost: <msats> msats  error: <error>
local: <state>  chain=<label>  event=<event_id>          -- only if a local record exists
```
then, for a completed job, one of: `seal: <seal>  mn: <mn>` plus, if the result has `region_n`, `lookup_id: <...>` and `location_decryption_key: <...>` (a version 2 hop); or `proof_hash: <...>`, `lookup_id: <...>`, `location_decryption_key: <...>` (a result with `hop_n`); or `proof_hash: <...>` alone (`cloud_cli.py:196-210`).

**`cyberspace cloud resume JOB_ID`** (`cloud_cli.py:213-292`): stderr `No local record for job <id>; nothing to resume.` exit 2 (`:218`); already done, stdout `Job <id> was already appended as event <event_id>.` (`:221`); chain head moved since the job was made, stderr `The chain <label> moved on since this job was created (head <head>, job bound to <prev>). The proof is worthless now; nothing appended.` exit 2 (`:229-234`). A still-pending job with an invoice prints `Waiting for the invoice to be paid...` and the bolt11 (`:241-242`); a balance still short after that, stderr `Balance still short; top up with \`cyberspace cloud deposit\`.` exit 2 (`:246`). After the result checks pass, `Verified cloud <action> (0 failed checks).` (`cloud_move.py:291`), then:
```
Moved. chain=<label> len=<n>
coord: 0x<coord_hex>
proof: <seal for a hop | proof_hash for a sidestep>
```
(`cloud_cli.py:290-292`; unlike `move`, no `action:`, `terrain_k:` or `source:` lines.) A failed check prints stderr `HOSAKA result failed verification; nothing appended:` and `  - <failure>` lines, exit 3 (`:263-266`).

Error/validation messages (e.g. "No local state found. Run `cyberspace spawn` first.", `cli.py:124`; "Specify exactly one of --to, --by, or --toward.", `cli.py:2589`) exist throughout and are **not** enumerated exhaustively here; representative ones are cited only where reached while tracing normal control flow. Any claim that this document lists every possible stderr string would be false. HOSAKA response fields (`tier`, `est_time`, `cap`, ledger rows, job results) are printed as the server returns them; their values are not characterised here.


---

## 9. Reference CLI vs DECK-0001 v3

**The divergence recorded at `2dff988` has largely closed.** The CLI was untouched from 2026-05-01 to 2026-09-02, then took 21 commits through 2026-11-15 (`git log 2dff988..4e5dab7`), among them "feat: DECK-0001 v3 stop coordinates (ports + landfalls) (#16)" (`ea71c0c`, 2026-09-06), "Sidestep version 3 and ride openings version 2" (`f6cc0d0`, 2026-09-28), "Chain rules 2026-09-28-virtual-brackets ..." (`b576a27`, 2026-10-08), "The ride builder writes every tag the chain rules read; move --hyperjump refuses instead of writing an invalid ride" (`b8caa40`, 2026-10-22), "Hop proof version 2: residue openings" (`d710b0f`, 2026-10-30) and "Chain rules revision 2026-10-08-residue-openings (#31)" (`5103c62`, 2026-10-31).

State at `4e5dab7`, per its own DECK-0001 conformance checklist (`docs/spec-conformance/DECK-0001_v3_2026-08-24.md`) and source:

- Stop coordinates (§1.1, §1.2): DONE, all eight landfall golden vectors (checklist `:11-13`).
- Anchors: reads `C`, `M`, `B`, `H`; legacy anchors handled per §2.2; `M`/`H` consistency checked but block data not verified against Bitcoin; header blobs (§2.4) TODO (checklist `:19-22`). `_query_hyperjump_anchor_for_height` (`cli.py:502`) now goes through `_stop_from_anchor` (`cli.py:464`). Default anchor relay `wss://relay.onosendai.tech` (`cli.py:244`).
- Boarding, station and rides: DONE **for verification** (`hyperspace.py`, `ride.py`, `chain.verify_chain`); the CLI does not publish boardings, and `move --hyperjump` and `hyperjump to/next/prev` without `--view` refuse (`_refuse_to_ride`, `cli.py:1779-1795`; checklist `:32-34`; `README.md:105-106`).
- `make_hyperjump_event` (`nostr_event.py:101-179`) now writes every tag the chain rules read on a ride, including `from_height`, `as_of`, `proof`, `mp` and `mn`, and refuses a zero-length ride.
- The grandfather lists are embedded in `cyberspace_core/grandfathered.py`, generated from `arkin0x/cyberspace@9210d6d` (`grandfathered.py:7-10`: 2,658 v1 hops, 36 v2 sidesteps, 16 rides).
- The checklist's header still says "Draft v3, last updated 2026-08-24" (`:3`) while its rows cite later revisions.

DECK-0001 Appendix C (`:478-493`) still names ONOSENDAI, NTH, `decks/landfall-reference.py` and `decks/hyperjump-reference.py` as reference implementations, not the CLI. The base spec §14 (`:2603-2611`) names the CLI as the reference for integer encoding, movement proofs and the GPS mapping.

---

## 10. Limitations and threat model (§12, non-normative)

- **What the protocol provides** (§12.1, `:2533-2539`): the single-location constraint now reads "A valid, linear movement chain makes forking detectable, and a fork kills the chain (§8.7.3)" (`:2535`).
- **What it does not provide** (§12.2, `:2541-2546`): physical location proof; trusted identity or sybil resistance ("One operator can control many keypairs", `:2544`); privacy by default; traversal necessity for decryption ("Region preimages can be computed directly without maintaining a movement chain", `:2546`).
- **Acknowledged attack vectors** (§12.3, `:2548-2553`): coordinate scanning; chain abandonment; ASIC-accelerated sidesteps (`:2552`); and **key grinding** (`:2553`): "Spawning inside an aligned region of height h costs about 2^(3·(85−h)) key generations. Regions walled above roughly h75 can be spawn-camped by grinding; travel reaches about h50 and hyperspace reaches stop neighbourhoods; regions walled between h50 and h75 therefore have the strongest isolation the protocol offers." The same bullet was already present at `919494a` (old `:1330`).
- **No difficulty adjustment** (§9.3, `:2324`; §13.3, `:2579-2583`): "Cantor tree difficulty is fixed by the mathematics ... no adjustment mechanism, no governance, and no parameters to tune." §9.3 adds that "difficulty migrates upward through extension mechanisms (DECKs) over decades."
- **Observer vs traveler** (§4.9, `:504`): a traveler's work is sequential and an observer's is not.
- **Hop work splits across workers** (§5.10, `:798`): the re-roll search MAY be split across workers; the residues themselves are computed per prime and batch-divided from each root (`:756`).

---

## 11. Paid proof service (HOSAKA)

- `docs/what-can-be-sold.md` §5 (`:144-158`): HOSAKA has sold hops and sidesteps "for months". "**§6.4 forbids resale. It does not forbid service.**" (`:150`) Each job is computed for one customer's chain position and is usable only by that customer (`:152`); "The moat is capital that depreciates, not knowledge that compounds" (`:158`).
- Base spec §6.12 (`:1234`): "This does not close the market in computation, only the market in reusable answers. Anyone may be paid to perform a traveller's own work for them."
- CLI: above the local `--max-lca-height` ceiling (default 16), `move` asks HOSAKA for a hop, or a sidestep when the height is above HOSAKA's hop cap, paid in sats over Lightning, and verifies every result at Level 1 before signing (`README.md:195-218`). Cloud hops are version 2 only (`README.md:211-218`). The `cloud` command group is in `cloud_cli.py` (§8).
- `docs/agents.md:58`: "Talking, hiding, finding and placing do not move the chain and may run freely within the budget. Only movement is serialized."

---

## UNKNOWN

Not confirmed from the fetched sources at the pinned commits. Explicitly not inferred.

**Not fetched / not executed:**
- The rendered TUI of `cyberspace move viz` and the GUI windows of `cyberspace 3d` and `cyberspace lcaplot`.
- The full literal text of `HELP_TEXT` (`helptext.py`).
- `decks/landfall-reference.py`, `decks/hyperjump-reference.py`, `hop-residue-reference.py` and the chain-rules golden vectors were neither executed nor checked against the CLI here. Whether the CLI passes `chain-rules-2026-10-08-residue-openings.json` is asserted by its `vectors/README.md` and commit messages, not verified.
- `arkin0x/nth`, ONOSENDAI and cyberspace-mcp were not fetched.
- Response payloads from HOSAKA (quotes, ledgers, job results) are printed by the CLI as received and are not characterised.
- The complete set of CLI error strings.

**Unresolved discrepancies in the sources themselves:**
- Section numbers in `movement.py` comments (for example "§5.5.2.1" at `:139`, "§5.7" at `:163`) do not match `CYBERSPACE_V2.md` at HEAD. Reported, not reconciled.
- Whether a spawn coordinate's plane bit is constrained. The spec requires only `C == pubkey` (§3.2, §8.3); §2.4 and §7.2 treat planes symmetrically; the CLI applies no constraint.
- Whether the CLI's `content = proof_hash_hex` on sidesteps (`nostr_event.py:331`) is intended. §8.5 specifies no `content`; it is not a tag the chain rules read, but it enters the event id.
- The CLI still leaves `sig` blank (`nostr_event.py:73`), while §8.2 and §8.7.3 now discard unsigned events as inauthentic. The CLI README says `sig` "is currently left blank so we can sign at publish-time" (`README.md:22`); no signing path was traced.
- The CLI's `cloud` "No identity" error points users at `cyberspace init`, which does not exist (§8).

**Resolved since the previous revision:**
- The base fork rule: §8.7.3 rule 4 (`:1914`) makes a forked chain dead from the spawn; DECK-0001 §8 (`:420`) defers to it.
- Whether the CLI tracks DECK-0001: it does now, for stop geometry and verification, not yet for publishing rides (§9).
