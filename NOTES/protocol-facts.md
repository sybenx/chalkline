# Cyberspace Protocol & CLI — Verified Facts

Compiled 2026-08-25; re-audited adversarially 2026-08-28 against fresh clones.

## Provenance

- `arkin0x/cyberspace` (branch `master`) @ `919494a09c1f636145749d76e56502a76036b1f5` — "DECK-0001 v3: Hyperspace (ports, landfalls, stations, rides) (#18)", 2026-08-24. Confirmed current `master` HEAD on 2026-08-28.
  - `CYBERSPACE_V2.md` (spec version string `2026-03-16-h34-corrected`)
  - `RATIONALE.md`, `readme.md`
  - `decks/README.md`, `decks/DECK-0001-hyperspace.md`, `decks/landfall-reference.py`
- `arkin0x/cyberspace-cli` (branch `main`) @ `2dff9885de8cbdea61eae10c6663e17ddad63460` — "docs: add move viz to README", 2026-05-01. Confirmed current `main` HEAD on 2026-08-28.
  - `README.md`, `docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md`
  - `src/cyberspace_cli/`: `cli.py`, `nostr_event.py`, `nostr_keys.py`, `parsing.py`, `targets.py`, `chains.py`, `helptext.py`, `viz/move_viz.py`, `visualizer/{app,lcaplot_app,viz}.py`
  - `src/cyberspace_core/`: `coords.py`, `cantor.py`, `movement.py`, `terrain.py`, `sector.py`, `location_encryption.py`

**Audit note (2026-08-28).** An independent re-audit re-cloned both repos and re-checked every line-numbered citation in the previous revision of this document. All of them matched. No factual error was found in the previously written body text. The changes in this revision are **additions**, not corrections:

- §4 previously gave the temporal work seed for `hop` and `sidestep` only. DECK-0001 v3 defines two further action types with *different* K derivations and *different* seed preimages; those are now in §4.
- §5.5's canonical `int_to_bytes_be_min` encoding was cited by name throughout but never defined. It is now in §5.
- The base spec's `hop_n` / `sidestep_n` combination and the double-SHA-256 proof-hash derivation (§5.4, §5.6, §6.8) were absent. They are now in §5.
- DECK-0001 v3's stop geometry — ports, landfalls, stations — was absent entirely. It is now §7.
- Three items previously listed as UNKNOWN (`move viz` output, `3d`/`lcaplot` output, `HELP_TEXT`) were resolvable from source that simply had not been fetched. They are resolved in §8. The genuinely-unresolvable residue is stated there instead.

Every claim below is cited to a spec section (`§`) and/or a file path with line numbers as they existed at the commits above. Anything not directly confirmed in these sources is listed under **UNKNOWN** rather than inferred.

**Note on section numbers.** The CLI's own conformance checklist (`docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md`) and inline comments in `movement.py`/`terrain.py` cite section numbers (e.g. "§5.5.2.1" at `movement.py:101`, "§4.1" at `movement.py:14`, "§5.9") that do **not** match the section numbers in `CYBERSPACE_V2.md` as fetched (terrain K is §5.2 there; the sidestep domain constant is §6.4). This mismatch is reported as-is; it is not resolved or guessed at. See **UNKNOWN**.

---

## 1. Coordinate derivation from a pubkey

- Cyberspace coordinates are 256-bit integers (`coord256`) split into three 85-bit unsigned axes X, Y, Z plus a 1-bit plane selector `P` (85×3+1 = 256). (§2.1)
- Bit layout is **interleaved (Morton-style), not three contiguous 85-bit runs.** §2.2 (`CYBERSPACE_V2.md:179-191`) is explicit: "The three axes are **interleaved** into the 256-bit integer, not packed sequentially." Bit `0` (LSB) is the plane bit `P`; bits `3, 6, 9, ...` are the 85 X bits; bits `2, 5, 8, ...` are Y; bits `1, 4, 7, ...` are Z. "The result looks like: `XYZXYZXYZ...P`".
- Reference encode/decode is normative pseudocode in §2.3 (`CYBERSPACE_V2.md:195-214`) and is reproduced verbatim, down to variable names, at `coords.py:91-98` (`xyz_to_coord`) and `:101-109` (`coord_to_xyz`), with `AXIS_BITS = 85` at `coords.py:47`:

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
- **Consequence relied on downstream:** "coordinates that are spatially close share similar bit prefixes" (§2.2), which is what makes aligned Cantor subtrees efficient — and, per DECK-0001 §4.4, is what makes station lookup a prefix search.
- **Identity-to-coordinate rule:** "Your Nostr public key, the 256-bit number that *is* your identity, is also your coordinate. ... The coordinate in a spawn event MUST equal the event's public key." (§3.1, §3.2/§8.3)
- In the CLI this is an identity function on the 32 raw pubkey bytes — no hashing, truncation, or transformation: `_coord_hex_from_pubkey_hex` returns the 32-byte pubkey hex unchanged (`cli.py:108-112`). `spawn` calls it directly on the generated/imported pubkey (`cli.py:604-605`).
- The coordinate's plane bit is therefore whatever the low bit of the raw pubkey happens to be (the pubkey bytes are read directly as the interleaved 256-bit integer via `coord_to_xyz` — e.g. `cli.py:629-630`). No spec text found constrains spawn coordinates to a particular plane. See **UNKNOWN**.
- **Respawning:** publishing a new spawn event (`A=spawn`) at any time invalidates all prior movement events in the old chain because it has a newer timestamp; the keypair returns to its spawn coordinate (= its pubkey) and the old movement history remains on relays but is no longer part of the active chain. (§3.2, and §12.3 "Chain abandonment")

---

## 2. Event kinds and their required tags

### 2.1 Complete kind registry

Exactly **three** Nostr event kinds are claimed across the base spec and all DECKs. `decks/README.md:41` shows DECK-0001 is the only entry in the DECK registry, so this list is complete as of `919494a`.

| Kind | Purpose | Claimed by | Ref |
|---|---|---|---|
| `3333` | All movement events. Base: `spawn`, `hop`, `sidestep`. DECK-0001 adds `enter-hyperspace`, `hyperjump` — same kind, per the "locked action-kind pattern" | base spec + DECK-0001 | §8.1 (`:954`), §8.8 (`:1067`), DECK §Abstract (`:18`) |
| `33330` | Location-encrypted content events | base spec | §8.6 (`:1016`), `readme.md:88` |
| `321` | Block anchor events (Bitcoin stop metadata) | DECK-0001 only | DECK §2 (`:103`) |

`decks/README.md:11,15` states DECKs are the designated place "to allocate additional Nostr event kinds", and a DECK MAY "Define new Nostr event kinds" and "Define new `A` tag values for movement events (`kind=3333`)".

### 2.2 Event id and signature

Event `id` is computed per NIP-01: `sha256(json.dumps([0, pubkey, created_at, kind, tags, content], separators=(",", ":"), ensure_ascii=False).encode("utf-8"))`. (§8.2; implemented identically at `nostr_event.py:14-25` `serialize_event_for_id` and `:28-44` `compute_event_id_hex`.)

Published events MUST carry a valid Schnorr `sig` over `id`; the spec notes some prototypes may leave `sig` blank for local-only chains (§8.2). The CLI does exactly this: `new_event` always sets `"sig": ""` (`nostr_event.py:47-71`; comment at `:69-70`: "Signature is intentionally blank for now. We can sign at publish-time."). The CLI's own conformance checklist records this as its one known gap ("`sig` signing not yet performed in local event objects", `docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md:4`, `:97-98`).

### 2.3 Base protocol events (§8)

**Spawn** (kind 3333, §8.3) — `["A","spawn"]`; `["C","<coord_hex>"]` (64-char lowercase hex, MUST equal the event `pubkey`); sector tags `X`, `Y`, `Z`, `S`.
CLI: `nostr_event.py:89-92` (`make_spawn_event`), `content=""`.

**Hop** (kind 3333, §8.4) — `["A","hop"]`; `["e","<spawn_event_id>","","genesis"]`; `["e","<previous_event_id>","","previous"]`; `["c","<prev_coord_hex>"]`; `["C","<coord_hex>"]`; `["proof","<proof_hash_hex>"]`; sector tags.
CLI: `nostr_event.py:140-160` (`make_hop_event`), `content=""`. Exact match.

**Sidestep** (kind 3333, §8.5) — `["A","sidestep"]`; both `e` tags; `["c",...]`, `["C",...]`; `["proof",...]`; `["mr","<M_x>:<M_y>:<M_z>"]` (per-axis Merkle roots, 64 hex chars each); `["mp","<proof_x>:<proof_y>:<proof_z>"]` (per-axis inclusion paths, `64×h` hex chars each, empty segment for `h=0`); `["hx","<int>"]`, `["hy",...]`, `["hz",...]` (per-axis LCA heights, base-10); sector tags.
CLI: `nostr_event.py:163-202` (`make_sidestep_event`). Tag list is an exact match.
**Observed asymmetry:** the CLI sets `content = proof_hash_hex` for sidesteps (`nostr_event.py:202`) but `content = ""` for spawn (`:92`) and hop (`:160`). §8.5 does not specify a `content` requirement either way, and the conformance checklist does not flag it. Reported as an observed asymmetry, not a deviation.

**Encrypted content** (kind 33330, §8.6) — `["d","<lookup_id_hex>"]`; optional `["h","<height_hint>"]`. §8.6 explicitly states "The encryption algorithm and ciphertext encoding (base64 vs hex) are out of scope for this spec; only lookup and key derivation are specified."
CLI (`nostr_event.py:118-137`, `make_encrypted_content_event`) adds two tags beyond the spec's set: `["encrypted","<algorithm>","<ciphertext_b64>"]` and `["version","2"]`, plus the optional `["h",...]` only when `--publish-height` is passed (`cli.py:1254-1258`, `:1303`). Algorithm is `"aes-256-gcm"` (`cli.py:1301`), implemented as AES-256-GCM with a 12-byte random nonce prepended to the ciphertext (`location_encryption.py:91-99`, `encrypt_with_location_key`). This is a CLI implementation decision, not a spec requirement.

### 2.4 Core action types (§8.8)

| `A` tag | Description | Proof type | Ref |
|---|---|---|---|
| `spawn` | Identity placement at pubkey-derived coordinate | None | §8.3 |
| `hop` | Movement via Cantor pairing tree | Cantor root | §8.4 |
| `sidestep` | Boundary crossing via Merkle hash tree | Merkle root | §8.5 |

"All three use event `kind = 3333`." (§8.8, `:1067`)

§8.9 (`:1069-1073`) states extensions ("DECKs") MAY introduce new event kinds, new `A` tag values, and additional validation rules applied only when the extension is in use.

### 2.5 DECK-0001 events (extension)

**`enter-hyperspace`** (kind 3333, DECK §3.1, `:152-162`) — `["A","enter-hyperspace"]`; `e` genesis; `e` previous; `["c","<current_coord_hex>"]`; `["C","<current_coord_hex>"]` (MUST equal `c`; the identity does not move); `["proof","<proof_hash_hex>"]` per §3.2; sector tags from `C`. Optional: `net`.

**`hyperjump`** (kind 3333, DECK §5.2, `:245-258`) — `["A","hyperjump"]`; `e` genesis; `e` previous; `["c","<origin_coord_hex>"]`; `["C","<destination_stop_coord_hex>"]`; `["from_height","<B_from>"]` (base-10); `["B","<B_to>"]` (base-10; `B_to != B_from` unless §5.6 applies); `["as_of","<A>"]` (station set bound, base-10; REQUIRED on the first ride after boarding; `A ≥ B_to`); `["proof","<merkle_root_hex>"]` per §5.4; `["mp","<openings>"]` per §5.5; sector tags from `C`. Optional: `net`; `e` tags with markers `hyperjump_from` / `hyperjump_to` referencing anchor events.

**Block anchor** (kind 321, DECK §2.1, `:107-116`) — required: `["C","<stop_coord_hex>"]` (stop coordinate per §1 — a port's merkle root, or a landfall coordinate); `["M","<merkle_root_hex>"]` (for a port `M == C`); `["B","<height>"]` (base-10); `["H","<block_hash_hex>"]`; `["P","<prev_block_hash_hex>"]`; sector tags `X`,`Y`,`Z`,`S` computed from `C` (not from the merkle root — DECK §1.3). Anchors SHOULD include `net` (`mainnet` assumed if absent) and `N` (next block hash) once known.

**Legacy anchors** (DECK §2.2, `:120`, normative): anchors published before this revision carry `C = merkle_root` for every block and no `M` tag. "An implementation encountering a `kind 321` event without an `M` tag MUST treat its `C` as the merkle root, derive the plane per §1.1, and for plane-0 blocks derive the landfall coordinate itself from `H` per §1.2. Such events MUST NOT be used as a source of landfall coordinates directly."

**Anchor validation** (DECK §2.3, `:124`): an anchor is valid for a stop iff `M` and `H` match the block at height `B` on the selected network, `P` matches its previous block hash, and `C` equals the §1 derivation from `M` and `H`. How block data is obtained is out of scope.

---

## 3. Movement chain rules

- Cyberspace state is "a per-pubkey, linear hash chain of signed Nostr events." (§8 intro)
- Chain start: the spawn event is first; its `C` tag equals the pubkey (§3.2, §8.3).
- Chain extension: hop and sidestep events reference the chain via two `e` tags — marker `"genesis"` → spawn event id, marker `"previous"` → immediately preceding event id (§8.4, §8.5). "Hash chain continuity: each movement event references the previous one by its cryptographic hash, forming a linear chain of signed proofs." (§1.3)
- Each hop/sidestep carries `c` = coordinate before, `C` = coordinate after (§8.4, §8.5).
- **Forking.** §12.1 (`:1312`) says only: "**Single-location constraint (per keypair):** A valid, linear movement chain makes forking detectable." An explicit invalidation rule exists only in the extension: "Two movement events with the same `previous_event_id` are a fork; both branches are invalid from that point." (DECK §8, `:328`). The base spec has no equivalent explicit rule. See **UNKNOWN**.
- **Freshness / non-cacheability:** every hop and sidestep MUST include a temporal-axis proof seeded from `previous_event_id`, so "the temporal seed for hop N is not known until hop N-1 is complete. The work cannot be precomputed or amortized." (§5.1, `:445`). This exists specifically to prevent reusing a cached spatial Cantor root for cheap repeated hops through already-visited regions (§5.1; `RATIONALE.md` §2 "Ensuring Every Hop Costs Work").
- **Not a heartbeat:** "An avatar's last hop event remains its state indefinitely. The temporal axis work is paid only when moving." (§5.1, `:449`)
- **Plane changes are valid mid-chain:** "Plane changes are valid in v2; verifiers MUST support hops where `plane1 != plane2`." (§8.7.1, step 2)
- **DECK-0001 chain rules** (§4.3 `:213-215`, §8 `:329-331`, normative for the extension):
  - The first `hyperjump` after an `enter-hyperspace` MUST carry `as_of` and have `from_height = station(C_e, as_of)`.
  - Every subsequent `hyperjump` not separated from the previous one by a `hop` or `sidestep` MUST have `from_height` equal to the previous `hyperjump`'s `B`.
  - A `hyperjump` whose previous event is neither `enter-hyperspace` nor `hyperjump` is invalid.
  - `enter-hyperspace` MAY follow any movement action.
  - A stop's coordinate MUST be verified against Bitcoin consensus for the selected network; implementations SHOULD treat stops with fewer than six confirmations as provisional and avoid them as destinations.
  - Exit (DECK §6, `:314`): an identity leaves the line by publishing a `hop` or `sidestep` whose `c` tag is the stop coordinate it is located at. No new rule applies; the exit is validated by the base protocol.
- **CLI-side chain-local rules** (not spec text; enforced by the reference client):
  - `move` requires an active local chain and appends to `~/.cyberspace/chains/<label>.jsonl` (`README.md` "Local storage"; `cli.py:2206-2212` reads `events[0]["id"]` as genesis and `events[-1]["id"]` as previous).
  - `--sidestep` cannot be combined with `--hyperjump` in one `move` invocation (`cli.py:2197-2199`).
  - While the local chain's last event has `A=hyperjump`, ordinary `move` is blocked unless `--exit-hyperjump` is passed (`cli.py:2239-2245`).
  - Large hops are refused client-side if any axis LCA height exceeds `--max-lca-height` (config default via `cyberspace config set --max-lca-height`), because Cantor proof compute is O(2^h) (`cli.py:2304-2312`; `README.md`).
  - `--toward` hops repeatedly until the target `(x,y,z,plane)` is reached, `--max-hops` is hit, or the user interrupts (Ctrl+C), printing per-hop progress and preserving already-appended events (`cli.py:2419-2556`).

---

## 4. Temporal-axis work: height K and seed, per action type

There are **two distinct terrain functions** in play — the base protocol's coordinate-derived K (§4.1 below) and DECK-0001's block-derived line terrain (§4.4 below). They use different domain strings, different digest-byte extraction, and different ranges. Do not conflate them.

### 4.1 Base-protocol terrain K (normative, §5.2)

Normative source: `CYBERSPACE_V2.md` §5.2 (`:451-480`), reproduced exactly by `terrain.py:40-85` (`terrain_k`).

**Constants** (§5.2, `:456-457`):
- `TERRAIN_DOMAIN_V2 = b"CYBERSPACE_TERRAIN_K_V2"` — `terrain.py:31`.
- `TERRAIN_CELL_BITS = [3, 7, 9, 11]` (exactly four integers, in this order) — default parameter `cell_bits: Tuple[int, int, int, int] = (3, 7, 9, 11)` at `terrain.py:46`.
- "If any part of this terrain function changes ... the domain string MUST be bumped to a new value." (§5.2, `:459`) — `terrain.py:29-30` confirms the V1→V2 bump happened for exactly this reason ("Bumped from V1 → V2 when the extraction changed from full byte (32 bits) to low nibble (16 bits)"); `TERRAIN_DOMAIN_V1 = b"CYBERSPACE_TERRAIN_K_V1"` is kept only as a deprecated alias (`terrain.py:104`).

**Algorithm**, given hop destination `(x2, y2, z2, plane)`:
1. For each `bits` in `[3, 7, 9, 11]` in order:
   - `bx = (x2 >> bits) << bits`, similarly `by`, `bz`.
   - `cell_coord = xyz_to_coord(bx, by, bz, plane)`, encoded as exactly 32 big-endian bytes.
   - `digest = sha256(TERRAIN_DOMAIN_V2 || byte(bits) || cell_coord_bytes)`, where `byte(bits)` is a single unsigned byte.
   - `nibble_i = digest[0] & 0x0F` (low 4 bits of the digest's first byte).
2. `word16 = (nibble_0 << 12) | (nibble_1 << 8) | (nibble_2 << 4) | nibble_3`.
3. `K = popcount(word16)`, MUST be an integer in `[0, 16]`.

(`terrain.py:84-85` uses `int.bit_count()` as the popcount. The implementation builds `word` with `word = (word << 4) | nibble` in loop order, which is identical.)

K is computed **from the destination coordinate, including the plane bit** — not from wall-clock time and not from the mover's identity (§5.1 "Key insight", `:494`; `terrain.py:62-63` rejects `plane not in (0,1)`).

**Distribution:** K is the popcount of 16 pseudorandom bits, so approximately Binomial(n=16, p=0.5), mean 8, worst case `2^16 = 65,536` Cantor pairs (~100 ms). (§5.2 "Why this works"; `terrain.py:19-20`.) `movement.py:18` caps temporal compute at `TEMPORAL_MAX_COMPUTE_HEIGHT = 17`.

### 4.2 Work seed — `hop` (kind 3333, `A=hop`)

§5.3 (`:482-492`, normative), matched by `movement.py:139-157` inside `compute_hop_proof`:

1. `previous_event_id` = the 32-byte NIP-01 event id from the `e` tag marked `"previous"` (§8.4), parsed from lowercase hex.
2. `prev_id_int = int.from_bytes(previous_event_id_bytes, "big")`; `t = prev_id_int % (1 << 85)`.
3. `t_base = (t >> K) << K`; `cantor_t = compute_subtree_cantor(t_base, K)` — the same aligned-subtree Cantor-root construction used for spatial axes (§4.6), applied to the temporal seed at height K.

The CLI validates that `previous_event_id_hex` is exactly 64 lowercase hex chars before use (`movement.py:143-146`).

### 4.3 Work seed — `sidestep` (kind 3333, `A=sidestep`)

§6.7 (`:699-707`, normative): "The temporal binding mechanism for sidesteps is **identical** to hop proofs (§5.2–§5.3)." K per §5.2 at the destination coordinate; `t` from `previous_event_id` per §5.3; `cantor_t = compute_subtree_cantor(t_base, K)`. Implemented at `movement.py:410-416` inside `compute_sidestep_proof`, using the same `TEMPORAL_MAX_COMPUTE_HEIGHT`.

§6.7 (`:711`) notes an entity MAY precompute the *spatial* Merkle root in advance; the temporal component must still be computed fresh, since `previous_event_id` is only known after the preceding event is published.

### 4.4 Work seed — `enter-hyperspace` (kind 3333, DECK-0001)

DECK §3.2 (`:164-173`, normative). The entry proof is the base protocol's temporal axis at the current coordinate, **with no spatial component**:

1. `K` = base-protocol terrain K at `C` per `CYBERSPACE_V2.md` §5.2, **including the plane bit**.
2. `t`, `t_base`, `cantor_t` from `previous_event_id` and `K` per §5.3 — i.e. the same seed as a hop.
3. `enter_n = π(0, cantor_t)` — note the spatial slot is literal `0`.
4. `proof_hash = sha256(sha256(int_to_bytes_be_min(enter_n)))`, lowercase hex.

DECK §7 (`:322`) reserves the domain string `CYBERSPACE_ENTER_HYPERSPACE_V1` for a future boarding toll. Until that is specified, §3.2 is the entire entry cost.

### 4.5 Work seed — `hyperjump` (kind 3333, DECK-0001)

DECK §5.3 (`:260-270`, normative). This does **not** use the base-protocol terrain function at all.

**Constants** (DECK §5.1, `:231-239`):
```
K_LINE                    = 6
SAMPLES                   = 32
HYPERSPACE_TERRAIN_DOMAIN = b"CYBERSPACE_HYPERSPACE_TERRAIN_V1"
HYPERSPACE_SEED_DOMAIN    = b"CYBERSPACE_HYPERSPACE_SEED_V1"
HYPERSPACE_LEAF_DOMAIN    = b"CYBERSPACE_HYPERSPACE_LEAF_V1"
HYPERSPACE_SAMPLE_DOMAIN  = b"CYBERSPACE_HYPERSPACE_SAMPLE_V1"
PAD_LEAF                  = 32 zero bytes
```
"Any change to a preimage format MUST bump the corresponding domain string." (`:241`)

Let `lo = min(B_from, B_to)`, `hi = max(B_from, B_to)`. The ride passes blocks `b = lo+1, ..., hi`; `n = hi - lo` is its length. For each `b`, with `H_b` the block hash (32 bytes, display order) and `previous_event_id` the 32-byte id from this event's `e previous` tag:

1. **Line terrain.** `digest = sha256(HYPERSPACE_TERRAIN_DOMAIN || H_b)`; `word16 = (digest[0] << 8) | digest[1]`; `K_b = popcount(word16)`, in `[0, 16]`, binomial mean 8.
   *(Contrast §4.1: two full digest bytes from one hash, not four low nibbles from four hashes, and keyed on the block hash rather than on aligned cell coordinates.)*
2. **Height.** `h_b = K_b + K_LINE`, an integer in `[6, 22]`.
3. **Seed.** `t_b = int(sha256(HYPERSPACE_SEED_DOMAIN || previous_event_id || be64(b))) mod 2^85`, where `be64(b)` is the height as 8 big-endian bytes.
   *(Contrast §4.2: the seed is a hash of the previous event id **and the block height**, not the raw previous event id reduced mod 2^85. The per-block height binding is what makes each leaf unique within one ride.)*
4. **Work.** `t_base_b = (t_b >> h_b) << h_b`; `cantor_t_b = compute_subtree_cantor(t_base_b, h_b)` per `CYBERSPACE_V2.md` §4.6.
5. **Leaf.** `leaf_b = sha256(HYPERSPACE_LEAF_DOMAIN || be64(b) || int_to_bytes_be_min(cantor_t_b))`.

**Merkle root** (DECK §5.4, `:274`): order leaves by ascending `b`; append `PAD_LEAF` until the count is a power of two (a single leaf needs no padding); build bottom-up with `parent = sha256(left || right)`, exactly as `CYBERSPACE_V2.md` §6.4 step 5. The `proof` tag is the root, lowercase hex.

**Openings** (DECK §5.5, `:278-288`): with `n_pad` the padded leaf count and `depth = log2(n_pad)`, sample indices are `idx_i = int(sha256(HYPERSPACE_SAMPLE_DOMAIN || root || be32(i))) mod n` for `i` in `0 .. SAMPLES-1` (`be32(i)` is four big-endian bytes; indices are positions among the `n` real leaves, `0` meaning block `lo+1`). If `n < SAMPLES`, indices repeat; implementations MAY deduplicate. The `mp` tag value is the `SAMPLES` inclusion paths joined by `:`; each path is the `depth` sibling hashes from leaf level to root, concatenated as lowercase hex (`64 * depth` characters).

**Zero-length ride** (DECK §5.6, `:304`): if `station(C_e, B_to) == B_to`, then `n = 0`; the event carries `from_height == B`, a `proof` of 64 zero characters, and an empty `mp` value.

### 4.6 Actions with no work seed

- `spawn` (kind 3333, `A=spawn`): no proof, no temporal work (§8.8 proof type "None").
- Encrypted content events (kind 33330): carry no movement proof at all (§8.6 tag list). Region key derivation for these is explicitly **non-temporal** — see §6.
- Block anchor events (kind 321): carry no movement proof; DECK §2.3 validation is against Bitcoin consensus, not against Cantor/Merkle work.

---

## 5. Canonical integer encoding and proof hashes

### 5.1 `int_to_bytes_be_min` (normative, §5.5, `:503-515`)

This encoding is consensus-critical and is referenced by every hashing step in §§5–7 and by DECK-0001 §§3.2, 5.3. Verbatim from the spec:

```python
def int_to_bytes_be_min(n: int) -> bytes:
    if n < 0:
        raise ValueError("expected non-negative")
    if n == 0:
        return b"\x00"
    # minimal big-endian bytes
    return n.to_bytes((n.bit_length() + 7) // 8, "big")
```

Note the `n == 0` case returns a single `0x00` byte, **not** the empty string.

This is distinct from the fixed-width 32-byte big-endian encoding used for coordinates in the terrain preimage (§5.2 step 1.3, `coord.to_bytes(32, "big")` at `terrain.py:76`). Two different integer encodings coexist in the protocol; which one applies is per-step.

### 5.2 Hop proof hash (§5.4, §5.6, `:496-525`)

- `hop_n = π(region_n, cantor_t)` — the 4D hop preimage combining spatial (§4.7) and temporal (§5.3) work. (`movement.py:159`)
- `hop_bytes = int_to_bytes_be_min(hop_n)`
- `movement_proof_key = sha256(hop_bytes)` (32 bytes)
- `proof_hash = sha256(movement_proof_key)` (32 bytes)
- In Nostr tags, `proof_hash` MUST be lowercase hex.

§5.6 (`:519`) is explicit that this "intentionally differs from the stable spatial region identifiers used for location-based encryption and discovery (§7)".

### 5.3 Sidestep proof hash (§6.8, `:713-729`)

- `sidestep_n = cantor_pair(region_m, cantor_t)`
- `sidestep_bytes = int_to_bytes_be_min(sidestep_n)`
- `proof_key = sha256(sidestep_bytes)`
- `proof_hash = sha256(proof_key)` (32 bytes), lowercase hex in tags.

### 5.4 Enter-hyperspace proof hash (DECK §3.2)

`proof_hash = sha256(sha256(int_to_bytes_be_min(π(0, cantor_t))))`, lowercase hex. Same double-SHA-256 shape as the base protocol.

### 5.5 Hyperjump proof

The `proof` tag is **not** a double-SHA-256 of a paired integer — it is the raw Merkle root over the per-block leaves (DECK §5.4). This is the one action whose `proof` tag has a different construction shape from the others.

---

## 6. Region/sector alignment and how region roots are computed

Two *different* notions both get called "region" informally. They are kept distinct here per the spec's own terminology.

### 6a. Sectors (§10) — fixed grid for Nostr querying

- A **sector** is a cube of `2^30` Gibsons per axis: `sx = x_u85 >> 30`, `sy = y_u85 >> 30`, `sz = z_u85 >> 30`. (§10; `SECTOR_BITS = 30` at `cli.py:215`; `SECTOR_BITS_DEFAULT = 30` at `sector.py:9`; `xyz_to_sector_id` at `sector.py:28-31` performs exactly this right-shift.)
- Purpose: fit coordinates into u32-friendly buckets and let Nostr relays — which cannot query prefix ranges on tag values — be queried per-axis-slice. (§10 opening)
- Every event that claims a coordinate MUST include `["X","<sx>"]`, `["Y","<sy>"]`, `["Z","<sz>"]`, `["S","<sx>-<sy>-<sz>"]`, with `sx/sy/sz` as base-10 strings, no leading `+`, no leading zeros except `"0"` itself, and `S` exactly `"<sx>-<sy>-<sz>"`. (§10 "Tag formatting rules")
- The CLI generates these via `_sector_tags_from_coord_hex` (`nostr_event.py:74-86`) for every spawn/hop/sidestep/hyperjump event, and re-derives them for display in `cyberspace sector` (`cli.py:651-669`, which uses a function-local `sector_bits = 30` at `:659` rather than the module constant — same value).
- Sectors are **not** cryptographic: no root, no hash, a plain integer bucketing for tag-based relay filtering (§10; `sector.py` contains no hashing).
- DECK §1.3 (`:99`): for events carrying a stop coordinate, sector tags are computed from `C`, **not** from the merkle root.

### 6b. Aligned regions and their Cantor/Merkle roots — the cryptographic identifiers

- **Aligned subtree** of height `h` on one axis: `base = (v >> h) << h`, spanning `2^h` consecutive leaf values `[base .. base + 2^h - 1]` (§4.5). For a movement `v1 → v2`, the covering aligned subtree uses `h = find_lca_height(v1, v2) = (v1 ^ v2).bit_length()` (or 0 if `v1 == v2`) and `base = (v1 >> h) << h` (§4.4, §4.5).
- **Axis Cantor root** (hops): build a Cantor-pairing binary tree bottom-up over the `2^h` leaves; `π(a,b) = (a+b)(a+b+1)/2 + b`. `axis_root(v1,v2) = compute_subtree_cantor(base, h)` (§4.2, §4.6). Implementation: `movement.py:27-51` `compute_subtree_cantor`, `:54-57` `compute_axis_cantor`; `cantor.py:6-8` `cantor_pair`.
- **3D spatial combine for hops:** `region_n = π(π(cantor_x, cantor_y), cantor_z)` (§4.7). `region_n` is "the stable spatial region integer" — a region identifier, not a coordinate-pair identifier; many coordinate pairs inside the same aligned subtree share the same root, by design. Implementation: `movement.py:135` inside `compute_hop_proof`, and separately `location_encryption.py:20-38` `derive_region_n_for_height` for discovery/encryption use.
- **Axis Merkle root** (sidesteps, for boundaries too large to Cantor-compute): over the same aligned subtree, each leaf value `L` is hashed as `H_i = SHA256(SIDESTEP_DOMAIN || int_to_bytes_be_min(base+i))` with `SIDESTEP_DOMAIN = b"CYBERSPACE_SIDESTEP_V1"`, then paired bottom-up as `parent = SHA256(left || right)` until one 32-byte root `M_axis` remains; for `h=0`, `M_axis` is the single leaf hash (§6.3-§6.4). Reference: `movement.py:15` (`SIDESTEP_DOMAIN`), `:185-190` (`merkle_leaf`), `:193-195` (`merkle_parent`), `:198-278` (`compute_axis_merkle_root_streaming`), matching §6.5's streaming algorithm (stack of pending hashes, max depth `h`, `h × 32` bytes working memory).
- **3D spatial combine for sidesteps:** `mx, my, mz` = the three axis Merkle roots as big-endian integers; `region_m = π(π(mx, my), mz)` (§6.6, `:688-693`). "All three axes always use Merkle roots in a sidestep. There is no mixed mode; the proof type (Cantor or Merkle) is determined by the action type (`hop` or `sidestep`), not per-axis." (§6.6, `:697`) Reference: `movement.py:397`.
- **Multi-axis sidestep** (§6.9, `:731-738`): a single sidestep MAY cross boundaries on multiple axes; each axis computes its own root (or trivial single-leaf hash where `h = 0`); the three roots combine into `region_m` via §6.6; **temporal binding applies once to the combined proof, not per-axis**; total work is the sum of per-axis work.
- **Merkle inclusion proof** (§6.10, `:740-752`): `axis_proof = H_sibling_0 || ... || H_sibling_{h-1}`, each a 32-byte sibling hash at depth `i` (leaf = depth 0). The verifier determines left/right ordering at each level from the destination leaf's position in the subtree (deterministic from the leaf value). For trivial axes (`h = 0`) the inclusion proof is empty and the root IS the single leaf hash.

### 6c. Region root → the exact bytes hashed (§7.2, normative, `:833-845`)

Given a spatial region integer `region_n` (the 3D region identifier from §4.7 for some aligned region):

- **`region_bytes = int_to_bytes_be_min(region_n)`** — this is the exact byte value the region identifier is computed from. Per §5.1 above, that is the minimal big-endian encoding of `region_n`, with the single byte `0x00` for `region_n == 0`. There is no fixed-width padding, no domain-separation prefix, and no length prefix at this step.
- `location_decryption_key = sha256(region_bytes)` (32 bytes)
- `lookup_id = sha256(location_decryption_key)` (32 bytes)
- Both outputs are 32-byte digests; in Nostr tags they MUST be lowercase hex.

"Seeing `lookup_id` (which is published to help people find the content) does not allow deriving `location_decryption_key` without the region preimage." (§7.2, `:841`)

Implementation: `location_encryption.py:42-46` `derive_region_keys_from_region_n`, called from `derive_region_key_material_for_height` (`:49-64`), which first computes `region_n` via `derive_region_n_for_height` (`:20-38`) — identical aligned-subtree-then-Cantor-pair logic to §4.7, applied per requested height rather than per movement.

**Explicitly not temporally bound:** "the temporal axis used for hop proofs (§5) is intentionally *not* included here. Location-based identifiers and keys remain a stable function of spatial regions. They do not change when someone moves through." (§7.2 Note, `:843`)

**Discovery scanning** (§7.4, recommended, `:869-886`): at `(x,y,z)`, for heights `h = 1..H`, compute `bx = (x >> h) << h` (etc.), `rx = compute_subtree_cantor(bx, h)` (etc.), `region_n = π(π(rx, ry), rz)`, then `lookup_id` per §7.2. Implementations SHOULD cap `H` for interactive use.

**Discovery radius / caching** (§7.3, §7.5): choosing a subtree height chooses a discovery radius; an aligned subtree of height `h` changes across a move only if the move crosses a boundary at that height, i.e. `(v1 >> h) != (v2 >> h)` — so higher region roots can be cached across small moves.

### 6d. `cyberspace cantor` naming vs spec naming

The CLI's `cantor` command computes exactly `region_n` (labelling it `cantor_number`) and prints `encryption_key_sha256 = sha256(cantor_number)` and `discovery_id_sha256_sha256 = sha256(encryption_key_sha256)` (`cli.py:1225-1237`). These are the values the spec calls `location_decryption_key` and `lookup_id` (§7.2). Different output labels, no computational difference.

---

## 7. Hyperspace stop geometry (DECK-0001 v3)

This section covers the v3 model that the reference CLI does not implement (see §9).

**Line, stop, port, landfall, station, ride, exit** are defined in DECK "Terms" (`:33-41`). The line is the sequence of Bitcoin blocks ordered by height; every block is a stop with height `B`, block hash `H`, merkle root `M`, and stop coordinate `C`.

### 7.1 Plane selection (DECK §1.1, `:46-57`, normative)

Let `merkle_root_int` be the block's merkle root interpreted as a 256-bit big-endian integer from its standard hex display form ("the same convention block explorers use"; implementations MUST agree on this byte order).

```
plane = merkle_root_int & 1
```

- `plane = 1`: the stop is a **port**. `C = merkle_root_int` (ideaspace).
- `plane = 0`: the stop is a **landfall**. `C` is derived from the block hash per §1.2 (dataspace).

"For a landfall the root's X/Y/Z bits are not used as a position; the block hash supplies the position." Miners cannot steer a block hash except by discarding valid blocks (§9.4).

### 7.2 Landfall derivation (DECK §1.2, `:59-95`, normative)

**Constant:** `LANDFALL_DOMAIN = b"CYBERSPACE_LANDFALL_V1"` (ASCII bytes).
**Input:** the 32-byte block hash `H` in standard big-endian display byte order.
**Arithmetic:** every step is performed in the decimal context of `CYBERSPACE_V2.md` §9.5 (precision 96, `ROUND_HALF_EVEN`, the exact `PI_STR`, the deterministic Taylor `sin`/`cos` with `TRIG_EPS` and `TRIG_MAX_ITER`). Square roots use the context's correctly rounded square root. Operations MUST be performed in the order written, each rounded to the context.

1. `seed = sha256(LANDFALL_DOMAIN || H)` (32 bytes).
2. `u1 = int(seed[0:16]) / 2^128`, `u2 = int(seed[16:32]) / 2^128`, both as exact decimals (`int()` big-endian).
3. `lon = (2 * u1 - 1) * PI` (radians, in `[-PI, PI)`).
4. `z = 2 * u2 - 1` (in `[-1, 1)`).
5. `rxy = sqrt(1 - z * z)`.
6. `(sin_lon, cos_lon) = sincos(lon)` per §9.5.
7. `dx = rxy * cos_lon`, `dy = rxy * sin_lon`, `dz = z` — a unit ECEF direction, uniform on the sphere.
8. With `a = 6378137` m, `f = 1/298.257223563`, `b = a * (1 - f)`: `inv = sqrt((dx*dx + dy*dy)/(a*a) + (dz*dz)/(b*b))`, `r = 1 / inv` (metres).
9. `x_km = r * dx / 1000`, `y_km = r * dy / 1000`, `z_km = r * dz / 1000` (ECEF kilometres).
10. Permute into Cyberspace axes per §9.4 (`X_cs = X_ecef`, `Y_cs = Z_ecef`, `Z_cs = Y_ecef`) and convert km to u85 per §9.7 step 9 (`u = km * units_per_km + 2^84`, `ROUND_HALF_EVEN`, clamp).
11. `C = xyz_to_coord(x, y, z, plane = 0)`.

The result lies on the WGS84 ellipsoid at geodetic altitude 0. Distribution is uniform in geocentric solid angle (uniform in surface area to within the 0.3% flattening).

**Golden vectors (consensus locks):** eight mainnet block heights with their block hashes and expected landfall coord256 values are tabulated at `decks/DECK-0001-hyperspace.md:84-93` (heights 398, 100399, 300399, 363199, 500399, 700398, 900399, 950399). Reference implementation: `decks/landfall-reference.py` in the same repository (stdlib Python; executing it checks all eight vectors). This file is present at `919494a` and is 5,029 bytes. It was **not** executed or line-audited for this document — see **UNKNOWN**.

### 7.3 The station (DECK §4, `:184-221`, normative)

**Distance** (§4.1): for coordinates `p`, `q` with axis values `(px,py,pz)`, `(qx,qy,qz)`, plane bits ignored:
```
d(p, q) = max(find_lca_height(px, qx), find_lca_height(py, qy), find_lca_height(pz, qz))
```
`d` is the height of the smallest aligned cube containing both points — exactly the boundary a direct hop or sidestep between them would have to cross.

**Definition** (§4.2): let `C_e` be the coordinate of the identity's `enter-hyperspace` event, and `A` the **station set bound** declared in the `as_of` tag of the identity's first `hyperjump`. `A` MUST be a height that exists on the selected network and MUST be `≥ B_to`. Let `Stops(A)` be all stops with height `≤ A`.
```
station(C_e, A) = the stop s in Stops(A) minimising d(C_e, C_s),
                  ties broken by the lowest height
```
Travelers SHOULD declare the highest height they have synced.

**Position semantics** (§3.3, `:175-180`): after `enter-hyperspace` the identity's location is still `C`; it changes only when a `hyperjump` is published. A `hop` or `sidestep` published immediately after `enter-hyperspace` moves from `C` as usual and cancels the boarding. Publishing `enter-hyperspace` while already at a stop is valid; the station is then that stop (LCA 0).

---

## 8. Exact output format of each CLI command

All output is via `typer.echo(...)` — one line of plain text per call, newline-terminated, to stdout unless `err=True` is noted (stderr). Lines below are literal `f-string` templates as they appear in `cli.py`/`targets.py`/`viz/move_viz.py`, with source lines cited.

The complete command surface at `2dff988` is 32 commands, enumerated from the Typer decorators in `cli.py` (`app`, `chain_app`, `config_app`, `target_app`, `hyperjump_app`, `geoid_app`, `move_app` — declared `cli.py:65-71`). All 32 are covered below.

**`cyberspace spec`** (`cli.py:566-582`): attempts `webbrowser.open("https://github.com/arkin0x/cyberspace/blob/master/readme.md", new=2)`, then **always** prints that URL on its own line; if the open failed or raised, additionally prints `(Could not auto-open browser; URL printed above.)` to stderr.

**`cyberspace help`** (`cli.py:560-563`): prints `HELP_TEXT.strip()` — the module-level raw string in `cyberspace_cli/helptext.py` (104 lines; the string opens at `helptext.py:1` with `HELP_TEXT = r'''` and closes at `:104`). `.strip()` removes only the leading and trailing newlines of the raw literal; the interior is emitted byte-for-byte. Content is a prose usage guide: title "Cyberspace CLI (v2)", a note that relay publishing is intentionally not implemented, a "Quick start" venv/pip block, a command list, and a closing "Security note" that the private key is stored locally in plaintext ("Treat it like a hot wallet key"). Not reproduced in full here; it is a static literal in the repo.

**`cyberspace spawn`** (`cli.py:585-631`), in order:
```
Spawned.
chain: <label> (len=1)
npub: <bech32 npub1... string>
nsec: <bech32 nsec1... string>
coord: 0x<64-hex coord>
plane=<0 or 1> <dataspace|ideaspace>
```
(`cli.py:624-631`; npub/nsec are NIP-19 bech32 encodings per `nostr_keys.py`'s `encode_npub`/`encode_nsec`, charset `"qpzry9x8gf2tvdw0s3jn54khce6mua7l"` at `nostr_keys.py:7`.)

**`cyberspace whereami`** (`cli.py:634-648`):
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
(`cli.py:641-648`)

**`cyberspace sector`** (`cli.py:651-669`):
```
sector:
X=<sx>
Y=<sy>
Z=<sz>
plane=<0 or 1> <dataspace|ideaspace>
S tag: <sx>-<sy>-<sz>
```
(`cli.py:664-669`)

**`cyberspace bench`** (`cli.py:686-752`): a header, one line per LCA height tested, then a summary:
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
(`cli.py:705-752`)

**`cyberspace gps <lat,lon>` / `--coord`** (`cli.py:941-1102`):
- GPS→coord: `coord: 0x<hex>` then `xyz(u85): x=<x> y=<y> z=<z> plane=<p> <label>` (`cli.py:1101-1102`; both fields on one line for this direction).
- coord→GPS (`--coord`): `coord: 0x<hex>`, then `xyz(u85): x=<x> y=<y> z=<z> plane=<p> <label>` on one line, then `gps: lat=<%.10f> lon=<%.10f> alt_m=<%.3f>` (`cli.py:1018-1020`).

**`cyberspace cantor`** (`cli.py:1105-1237`): `from:` / `to:` blocks (`coord:`, `xyz:` sub-lines), then per axis `X`, `Y`, `Z` a block of `axis <name>:`, `v1=... v2=...`, `lca_height=...`, `subtree_base=...`, `subtree_range=[...] leaves=...`, then either the full per-level Cantor tree (`level_<n> (<count> nodes): [hex, hex, ...]`, only if `lca_height <= --max-height`) or `tree_levels: omitted (height <h> > --max-height <n>)`, followed by `cantor_root_hex=...` and `cantor_root_bytes=<n>`; finally a `combined:` block with `cantor_number_hex=...`, `cantor_number_bytes=<n>`, `encryption_key_sha256=<hex>`, `discovery_id_sha256_sha256=<hex>`. (`cli.py:1179-1237`)

**`cyberspace encrypt`** (`cli.py:1239-1307`): exactly one line — the created Nostr-style event as compact JSON (`json.dumps(event, separators=(",", ":"), ensure_ascii=False)`). (`cli.py:1307`)

**`cyberspace decrypt`** (`cli.py:1310-1390`):
```
height: <chosen_height>
lookup_id: <d_tag value>
<decoded plaintext, or base64 of raw bytes if not valid UTF-8>
```
(`cli.py:1385-1390`)

**`cyberspace scan`** (`cli.py:1393-1461`): one summary line per height, `h=<height> d=<lookup_id_hex> matches=<n>`, followed by an indented `  id=<id> pubkey=<pubkey> kind=<kind>` line per matching event (only present when `--events-file` is used). (`cli.py:1457-1461`)

**`cyberspace config show`** (`cli.py:413-419`):
```
default_max_lca_height: <int>
gps_geoid_model: <str>
```

**`cyberspace config set --max-lca-height N`** (`cli.py:421-435`): `Saved.` then `default_max_lca_height: <int>`.

**`cyberspace config set-geoid-model MODEL`** (`cli.py:438-456`): `Saved.` then `gps_geoid_model: <str>`.

**`cyberspace geoid doctor`** (`cli.py:457-513`): `config_default_model=<...>`, `effective_model=<...>`, `search_dirs:` followed by one path per line, then per checked model `model=<name>` and either (`status=available`, `path=...`, `grid=<w>x<h>`, `scale=<...>`, `offset=<...>`, optionally `size_bytes=<n>`) or (`status=missing`, `candidates:` + one path per line) or (`status=invalid`, `error=<...>`). (`cli.py:478-513`)

**`cyberspace target set <coord>`** (`cli.py:516-529`): `(current) <label> 0x<coord_hex>`.

**`cyberspace target use <label>`** (`cli.py:532-545`): `active_target: <label>`, or (stderr) `Unknown target label: <label>` with exit code 1.

**`cyberspace target list`** (`cli.py:548-557`, formatting in `targets.py:90-100` `format_target_list`): `(no targets yet)` if empty; otherwise one line per target, `<label> 0x<coord_hex>`, with a `(current) ` prefix on the active one.

**`cyberspace history`** (`cli.py:2614-2655`):
- `--json`: one line per event, each the compact-JSON event object, nothing else.
- default: `chain: <label> (showing <n> events)` header, then per event `<0000> <action:<5} id=<id> coord=0x<coord>` with ` proof=<proof>` appended if a `proof` tag is present (`cli.py:2645-2655`; index is Python `{i:04d}`, action field is `{action:5}`).
- Empty chain, non-JSON: `(empty chain) <label>`.

**`cyberspace chain list`** (`cli.py:2658-2671`): `(no chains yet)` if none; else one line per chain, `<*|space> <label> (len=<n>)`, `*` marking the active chain.

**`cyberspace chain use <label>`** (`cli.py:2674-2685`): `active_chain: <label>`, or stderr `Unknown chain: <label>` exit 1.

**`cyberspace chain status`** (`cli.py:2688-2723`):
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
(`cli.py:2715-2723`; empty chain instead prints `(empty chain) <label>`.)

**`cyberspace move --by / --to`** — single hop, printed by `_do_single_hop` (`cli.py:2380-2394`):
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
or, if a Cantor hop proof was used:
```
proof: <proof_hash>
terrain_k: <K>
```
or, if this was a hyperjump-to-height move (no proof object):
```
action: hyperjump
B: <hyperjump_to_height>
```

**`cyberspace move --toward`** — per-hop progress in the loop (`cli.py:2419-2556`): for a final plane-switch hop, after the `_do_single_hop` output, prints `hop: <n>`, `x=<x>`, `y=<y>`, `z=<z>`, `plane=<p> <label>` (`cli.py:2461-2465`); for an ordinary progress hop, `hop: <n>`, `x=<x1> remaining=<tx-x1>`, `y=<y1> remaining=<ty-y1>`, `z=<z1> remaining=<tz-z1>`, `plane=<p> <label>`, and if a proof was produced, `lca_height: X=<hx> Y=<hy> Z=<hz> limit=<hop_limit>` and `terrain_k: <K>` (`cli.py:2545-2552`). On arrival: `Arrived.` then `coord: 0x<coord_hex>` (`:2438-2440`). On `--max-hops`: `Stopped after max_hops=<n>.` then `coord: 0x<coord_hex>` (`:2443-2445`). On Ctrl+C: stderr `Interrupted after hops=<n>.` then `coord: 0x<coord_hex>`, exit code 130 (`:2553-2556`). When a hop requires temporarily raising the LCA-height limit, a stderr line precedes it: `LCA boundary encountered on axis <A[,B[,C]]>; temporarily increasing max_lca_height from <old> to <new> for this hop.` (`:2523-2528`).

**`cyberspace hyperjump show/to/next/prev`** — anchor display via `_print_hyperjump_anchor` (`cli.py:400-410`):
```
hyperjump_block_height=<n>
coord: 0x<coord_hex>
x=<x>
y=<y>
z=<z>
plane=<p> <label>
event_id=<id>          -- only if the anchor event has a non-empty id
```
`to`/`next`/`prev` (non-`--view`) additionally emit the `_do_single_hop` block above (with `action: hyperjump` / `B: <height>`) before this anchor block.

**`cyberspace hyperjump nearest`** (and cache/expand modes) — via `_print_ranked_hyperjumps` (`cli.py:1764-1799`):
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
where each direction hint is `<axis>+ (<delta>)`, `<axis>- (<delta>)`, or `<axis>= (0)` (`cli.py:316-321`, `_direction_hint`). If nothing is found: `No nearby hyperjumps found.` (plus a hint about `--expand`/`--cache` unless `--expand` was already used), or in cache/expand modes `No hyperjumps found in cache.` / `No hyperjumps found after expanding to radius=<n>. Try 'hyperjump sync' + 'hyperjump nearest --cache' for a global search.` (`cli.py:2051`, `:2083-2087`, `:2116-2119`).

**`cyberspace hyperjump sync`** (`cli.py:1802-1953`): progress lines per batch (`  Batch <n>: <count> event(s), <new> new — <total> total unique so far`), then on completion `Block range: <min> to <max>` (if any block heights parsed), `Cached <n> unique hyperjump(s) to <path>`, `(from <fetched> fetched + <existing> previously cached)`, `Estimated remaining: ~<n, thousands-separated> events` (the constant `940000` is hard-coded as the assumed line size, `cli.py:1953`). If nothing found: `No hyperjump events found on the relay.` (`:1935`)

**`--verbose`/`-v`** on any hyperjump-relay command additionally prints, around the query: `req_filter:` + compact-JSON filter, and after the `nak` subprocess runs, `nak_exit_code: <n>`, `nak_stdout:` (each output line, or `(empty)`), `nak_stderr:` (same). (`cli.py:251-253`, `:271-286`)

**`cyberspace move viz`** (`cli.py:2726-2735` → `viz/move_viz.py:521` `move_viz_command` → `:30` `run_move_viz`). This is a Textual TUI; the full-screen rendering is not line-oriented and is not characterised here (see **UNKNOWN**). Everything it writes to the terminal *outside* the TUI is fixed and is reproduced here:
- No local state: stderr `No state. Use \`cyberspace spawn\` first.`, exit 1 (`move_viz.py:527-528`).
- Textual not installed: stderr `TUI visualizer dependencies not installed.` then `Install: pip install 'cyberspace-cli[viz]'`, exit 1 (`move_viz.py:38-40`).
- On commit, after the TUI exits, it echoes a single string containing literal Rich-style markup — `typer.echo` does not render markup, so it is emitted verbatim (`move_viz.py:497`):
  ```
  \n[bold green]✓ Executing move:[/]\n  dx: [cyan]<+dx with thousands separators>[/]\n  dy: [cyan]<+dy>[/]\n  dz: [cyan]<+dz>[/]\n
  ```
  It then shells out to `subprocess.run(["cyberspace", "move", "--by", "<dx>,<dy>,<dz>"], capture_output=True, text=True)` and re-emits the child's `stdout.strip()` to stdout and `stderr.strip()` to stderr (`move_viz.py:500-510`). The stdout block is therefore exactly the `cyberspace move --by` format documented above. On a non-zero child exit it prints `\n[red]Move failed with exit code <n>[/]` (again with literal markup) and exits with the child's code (`:512-514`).
- On escape: `Cancelled.` (`:516`). Otherwise: `No changes.` (`:518`).

**`cyberspace 3d`** (`cli.py:850-938`) and **`cyberspace lcaplot`** (`cli.py:755-847`) open a matplotlib/Tk GUI window (`cyberspace_cli.visualizer.app.run_app` / `visualizer.lcaplot_app.run_app`). The rendered window contents are graphical and not characterised here (see **UNKNOWN**). Their terminal output on a missing-dependency failure is fixed, four stderr lines then exit 1:
```
3D visualizer dependencies are not installed.      -- or "LCA plot dependencies are not installed."
Install extras: pip install 'cyberspace-cli[visualizer]'
System deps: you may also need python3-tk.
Import error: <exception>
```
(`cli.py:914-918` for `3d`; `cli.py:828-832` for `lcaplot`.)

Error/validation messages (e.g. "No local state found. Run `cyberspace spawn` first.", `cli.py:95`; "Specify exactly one of --to, --by, or --toward.", `cli.py:2230`) exist throughout and are **not** enumerated exhaustively here; representative ones are cited only where reached while tracing normal control flow. Any claim that this document lists every possible stderr string would be false.

---

## 9. Divergence: reference CLI vs DECK-0001 v3

**The CLI's hyperjump implementation does not implement DECK-0001 Draft v3.** Evidence, all at `2dff988`:

- `nostr_event.py:95-115` (`make_hyperjump_event`) produces tags `["A","hyperjump"], ["e",...,"genesis"], ["e",...,"previous"], ["c",...], ["C",...], ["B", to_height]` plus sector tags — **no** `from_height`, `as_of`, `proof`, or `mp` tag, and `content=""`. Per DECK §5.2 all four of those are required tags.
- A repo-wide search of `src/`, `tests/`, `docs/`, and `README.md` for `enter-hyperspace`, `enter_hyperspace`, `as_of`, `from_height`, `station`, `landfall`, `K_LINE`, and `HYPERSPACE_` returns **zero** matches. None of DECK v3's mechanisms exist in the codebase in any form.
- The CLI tracks hyperjump position via `_hyperjump_block_height_from_event` (`cli.py:324-338`), which reads a plain `B` tag off the most recent chain event, and `_require_hyperjump_system_state` (`cli.py:341-355`), which errors ("You are not currently on the hyperjump system. Perform a valid hyperjump first.", exit code 2) unless the last chain event already has `A=hyperjump` with a valid `B`. There is no boarding, no station, no `as_of`, and no chain rule of the DECK §4.3 kind.
- `HYPERJUMP_KIND = 321` (`cli.py:214`) is used only to *query* anchor events from a relay via the external `nak` tool (`_nak_req_events`, `cli.py:218-307`); the CLI never publishes kind-321 anchors.
- `_query_hyperjump_anchor_for_height` (`cli.py:358-397`) reads the anchor's `C` tag verbatim and decodes it with `coord_to_xyz`. It never reads an `M` tag and contains no landfall derivation. Under DECK §2.2 that makes it a legacy-anchor consumer: for plane-0 blocks the coordinate it displays and moves to is the merkle root, which §2.2 says "MUST NOT be used as a source of landfall coordinates directly."

**Commit-date comparison.** The DECK is newer than the hyperjump code by 4 months and 21 days:

| Artifact | Last commit | Date |
|---|---|---|
| `decks/DECK-0001-hyperspace.md` | `919494a` "DECK-0001 v3: Hyperspace (ports, landfalls, stations, rides) (#18)" | 2026-08-24 |
| Newest commit touching hyperjump logic in `cli.py` (`git log -S"hyperjump"`) | `6302561` "feat: improved hyperjump nearest search with progressive expansion and local cache" | 2026-04-03 |
| Newest commit touching hyperjump logic in `nostr_event.py` (`git log -S"hyperjump"`) | `dd6834c` "dedicated hyperjump command for finding hyperjumps and moving between them once on the hyperjump system" | 2026-03-11 |
| Newest commit touching `cli.py` at all | `826387c` "fix: skip move callback when subcommand is invoked" | 2026-05-01 |

The `cyberspace-cli` repository as a whole has not been committed to since 2026-05-01, and the hyperjump tag builder has not changed since 2026-03-11 — before the v2 draft of 2026-04-16, let alone v3. The divergence is chronological, not a disagreement: the DECK was rewritten twice after the CLI code was last touched. DECK Appendix B (`:373`) discusses the relationship to earlier drafts; DECK Appendix C (`:379-389`) names the reference implementations as ONOSENDAI (web client) and `arkin0x/nth` (the kind-321 anchor publisher) — **not** `cyberspace-cli`. `cyberspace-cli` is not claimed by the DECK as a hyperspace reference implementation.

---

## UNKNOWN

Not confirmed from the fetched sources at the pinned commits. Explicitly not inferred.

**Not fetched / not executed:**
- The rendered screen content of `cyberspace move viz` (Textual TUI). `viz/move_viz.py` (532 lines) was read and its non-TUI terminal output is documented in §8, but the TUI widget layout, key handling, and on-screen text were not characterised.
- The rendered window content of `cyberspace 3d` and `cyberspace lcaplot` (matplotlib/Tk). `visualizer/app.py` (634 lines), `visualizer/lcaplot_app.py` (269 lines), and `visualizer/viz.py` (438 lines) were not read; only the dependency-guard stderr text in `cli.py` was verified.
- The full literal text of `HELP_TEXT` (`helptext.py`, 104 lines) is not reproduced here. Its structure is described in §8; the byte-exact content was not transcribed.
- `decks/landfall-reference.py` (5,029 bytes) was confirmed present but was neither read line-by-line nor executed. Whether it actually reproduces the eight golden vectors in DECK §1.2 is asserted by the DECK (`:95`) and is **not** independently verified here.
- The eight landfall golden vectors in DECK §1.2 were not recomputed. They are transcribed by reference only.
- `arkin0x/nth` and the ONOSENDAI web client (DECK Appendix C) were not fetched. The DECK's claims about them — that they reproduce the golden vectors byte for byte, and the header-blob format in `nth`'s `docs/HEADER-BLOBS.md` (DECK §2.4) — are unverified.
- The complete set of the CLI's error/validation strings. §8 documents success-path output; the stderr surface was traced only where normal control flow reached it.

**Unresolved discrepancies in the sources themselves:**
- The section-numbering mismatch between `CYBERSPACE_V2.md` as fetched (terrain K at §5.2, sidestep domain at §6.4) and the section numbers cited in `docs/spec-conformance/CYBERSPACE_V2_2026-03-16-h34.md` and in code comments (`movement.py:14` "§4.1", `movement.py:101` "§5.5.2.1", `movement.py:98` "§5.5.3 / §5.7"). These plausibly reflect an earlier draft's numbering, but nothing in either repo states so. Reported, not reconciled.
- Whether the base `CYBERSPACE_V2.md` defines an explicit equivocation/fork-invalidation rule. §12.1 (`:1312`) says only "A valid, linear movement chain makes forking detectable", and §12.1 is under a heading marked "(non-normative)". The explicit rule ("two events with the same `previous_event_id` are a fork; both branches are invalid from that point") appears only in `DECK-0001-hyperspace.md` §8, an extension document. Whether the base protocol intends the same rule is not stated anywhere found.
- Whether a spawn coordinate's plane bit is constrained. The spec states only that the coordinate MUST equal the pubkey (§3.2, §8.3); it never separately discusses the plane bit of spawn coordinates, and the CLI applies no constraint.
- Whether the CLI's `content = proof_hash_hex` on sidestep events (`nostr_event.py:202`) is intended. §8.5 specifies no `content` value; the conformance checklist does not mention it.
- Whether `cyberspace-cli` is intended to track DECK-0001 at all. The DECK's own reference-implementation list (Appendix C) does not name it, and the repo has no commits after 2026-05-01. Whether the divergence in §9 is abandonment, deliberate scope exclusion, or pending work is not stated in either repo.
- The behaviour of `cyberspace hyperjump to/next/prev --view` against a chain not currently on the hyperjump system, beyond the fact that view mode does not call `_require_hyperjump_system_state` (`cli.py:1549-1551` and equivalents). The interaction with arbitrary chain state was not traced further.
