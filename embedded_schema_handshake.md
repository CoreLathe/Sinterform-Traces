# Ad Hoc Abstraction Negotiation for Ultra-Constrained Embedded Devices

Problem: Design the protocol by which embedded devices with <64KB RAM autonomously negotiate shared abstractions (event schemas, coordinate systems, temporal references) when they enter RF range of each other, without pre-shared ontologies or cloud mediation. How do you prevent "abstraction collisions" where two nodes use the same event type name for physically incompatible phenomena, given they can't store large negotiation histories?

---

Okay, before I start designing anything I need to get concrete about what "embedded devices entering RF range" actually means. Soil moisture sensors on stakes with 8KB RAM and LoRa. Livestock ear tags, 16KB, BLE. Pipeline vibration monitors at 32KB on 802.15.4. Floating water quality buoys, 48KB, LoRa. Trail cameras with PIR triggers at the upper bound, 64KB, Zigbee. These devices encounter arbitrary other devices they've never seen — no shared firmware, no shared manufacturer, no shared deployment context. They have to discover what the other one *means* by its data.

The pathology to prevent is concrete. A pipeline vibration sensor and a seismograph both emit `{ type: "VIBRATION", magnitude: 7.2 }`. One means mm/s² on a steel pipe wall. The other means Richter-scale-adjacent ground motion. A downstream consumer fusing these streams would produce catastrophe.

I want to jump to a hash-based naming scheme but I should stop — what are the actual constraints first?

Hard constraints: <64KB RAM total means protocol state budget is maybe 2-8KB max. No persistent TCP, we're talking packet radio, often lossy. No cloud, no certificate authority, no DNS, no shared registry. Devices may be in range for seconds or hours. No assumption about clock sync. And the vocabulary of a given device is small — typically 3-15 event types, 1-5 measurement dimensions.

So the problem is really small-vocabulary bilateral ontology alignment under severe memory and bandwidth constraints.

---

## Identity and Capability Announcement

What does a device actually *have* to identify itself and its phenomena? Every device knows a hardware ID (MAC or fused serial, 6-8 bytes), what physical transducers it has (accelerometer, thermistor, pH probe, photodiode), the units and range of each transducer, and what events it derives from those transducers — threshold crossings, patterns, and so on.

Here's the insight: transducers are the ground truth, not event names. A name is a human convenience. A transducer-type plus unit plus range plus mounting-context is a physical fact. So the core data structure is what I'll call a Sensor Grounding Descriptor:

```
SENSOR GROUNDING DESCRIPTOR (SGD) — per measurement channel
  transducer_class : uint8   (enum: 0x01=accel, 0x02=temp, 0x03=humidity,
                     0x04=pressure, 0x05=pH, 0x06=photodiode, ...)
  axis_count       : uint4   (1-3)
  unit             : uint8   (SI enum: m/s², °C, Pa...)
  range_min        : float16
  range_max        : float16
  sample_rate_hz   : float16
  mounting         : uint8   (enum: ground, structure, fluid, air, body, unknown)
  precision_bits   : uint8
  Total: 13 bytes per channel
```

Wait — I said "no pre-shared ontologies" but I just introduced a `transducer_class` enum. That's a pre-shared ontology. I need to confront this tension directly.

Two devices that share literally nothing — not even the concept of "temperature" — cannot negotiate meaning. That's not a protocol problem, it's an epistemological impossibility. There has to be a minimal shared physical vocabulary. The question says "no pre-shared ontologies" but it can't mean "no shared physics." It means no pre-shared application-level ontologies — no shared event schemas, no shared coordinate conventions, no shared semantic naming. The physical SI unit system and basic transducer taxonomy are the equivalent of a shared alphabet. Without them there's no communication at all, just noise.

So we adopt a compact, fixed, physical-layer enum — roughly 128 entries for transducer classes, 64 entries for SI units. This is analogous to how TCP/IP assumes shared byte ordering. It's below the ontology layer. Call it the Physical Grounding Vocabulary, PGV — a ROM constant, maybe 256 bytes of enum tables. This is the protocol's one axiomatic dependency, and everything above it is negotiated.

---

## The Negotiation Protocol

The state machine is straightforward:

```
SILENT → ANNOUNCE → GROUND → ALIGN/DISJOINT → RESOLVE → FUSED → COMPACT → SILENT
```

SILENT is default, listening for beacons. On receiving a beacon or proximity trigger, a node broadcasts its own SGD manifest in an ANNOUNCE frame. On receiving a peer's manifest, it enters GROUND to compare SGD channels for physical compatibility. If compatible channels exist, ALIGN proceeds to exchange event derivation rules. If none overlap, DISJOINT — log the peer, stop, no harm done. RESOLVE detects and resolves abstraction collisions. FUSED means operating with shared event semantics. When the peer leaves range or times out, COMPACT compresses the session to a Peer Digest and returns to SILENT.

Memory budget per peer session: 128-512 bytes. A device can maintain 4-16 concurrent sessions in 2-4KB.

The announce frame itself is compact:

```
ANNOUNCE FRAME
  magic: 0xAD0C (2B), proto_version: uint8 (1B), device_id: uint48 (6B)
  sgd_count: uint4, event_count: uint4, sgd[]: 13B each, event[]: variable
  crc16: uint16 (2B)
```

For a device with 4 sensors and 6 events: roughly 10 + 52 + 60 = 122 bytes. Fits in a single LoRa packet.

---

## Abstraction Collision Detection and Resolution

This is the heart of the question. Let me be very precise about what an abstraction collision actually is by working through concrete cases.

`VIBRATION` on Node A means accelerometer on pipe, >2g threshold. `VIBRATION` on Node B means accelerometer on ground, STA/LTA ratio >3. Same name, same transducer class, different mounting and different derivation. `HIGH_TEMP` on A means thermistor >85°C on an engine block; on B it means thermistor >35°C in ambient air — same name, same transducer, different range semantics. `MOTION` from a PIR trigger versus `MOTION` from an accelerometer IMU — same name, different transducer class entirely. `FLOOD` as water level >2m versus `FLOOD` as soil moisture >90% — same name, different transducer class, different derivation, correlated but distinct phenomena.

Collisions exist on a spectrum from trivially detectable (different transducer class) to subtle (same transducer, different derivation parameters).

To detect them, devices need to describe *how* they derive an event from raw sensor data. Not the name — the recipe. This is the Event Derivation Descriptor:

```
EVENT DERIVATION DESCRIPTOR (EDD)
  event_local_id : uint8
  name_hash      : uint16  (CRC16 of ASCII name — display/debug only)
  source_sgd_idx : uint8   (which sensor channel)
  derivation_type: uint8   (enum: threshold, rate, window_avg,
                   sta_lta, pattern_match, compound, raw_forward)
  params[]:        derivation-type-specific
    THRESHOLD:   direction(uint2), value(float16), hysteresis(float16)
    STA_LTA:     sta_window_s(f16), lta_window_s(f16), ratio_thresh(f16)
    WINDOW_AVG:  window_s(f16), threshold(f16)
    COMPOUND:    operand_ids(uint8[2]), operator(uint4), time_window(f16)
  Typical size: 8-16 bytes
```

The critical design decision: `name_hash` is explicitly *not* the canonical identifier. It's metadata for human debugging. The canonical identity of an event is the tuple `(transducer_class, unit, mounting, derivation_type, params)`. Events are identified by their physical derivation, not by their name. This is the key anti-collision insight.

The detection algorithm runs in levels of increasing specificity:

```
detect_collision(local_edd, remote_edd):
  if name_hash mismatch → NO_COLLISION (different names, nothing to collide)
  // Same name_hash — true match or collision?
  if transducer_class mismatch → COLLISION_CLASS_MISMATCH
  if mounting mismatch         → COLLISION_CONTEXT_MISMATCH
  if derivation_type mismatch  → COLLISION_DERIVATION_MISMATCH
  if param_distance > threshold → COLLISION_PARAM_MISMATCH
  else → COMPATIBLE (genuinely the same physical phenomenon)
```

Now what do we *do* with a detected collision? Two options: rename one event with a disambiguating suffix, or create a mapping table preserving both identities. Can we afford mapping tables in <64KB RAM? A mapping entry is about 4 bytes (local_id → peer_id + resolution_code). With 15 events × 4 peers = 60 entries × 4 bytes = 240 bytes. Yes, easily affordable. Use the mapping table because it preserves information.

The resolution exchange is a four-message handshake: both nodes independently run detection, exchange COLLISION_REPORTs, confirm with ACKs, and commit a deterministic resolution. Resolution types: IDENTICAL (full SGD+EDD match, merge as same type), DISAMBIGUATE (same name, different physics — tag events with a compact qualifier from the SGD difference), SUBSUMPTION (one event is a strict subset, like >85°C subsumes >35°C — mark parent/child), or ORTHOGONAL (same name, completely different transducer — scope with device_id prefix).

---

## Temporal and Spatial Reference Negotiation

Concrete case: a soil sensor and a weather station come into range. Neither has GPS. Neither has NTP. They need to correlate events in time.

Devices don't share a clock. They share a radio link with measurable round-trip time:

```
1. A sends TIMESYNC_PING { local_tick: A.tick₁ }
2. B responds: TIMESYNC_PONG { echo: A.tick₁, local_tick: B.tick₁, processing_delay: δ_B }
3. A computes: RTT = A.tick₂ - A.tick₁ - δ_B; offset ≈ B.tick₁ - A.tick₁ - RTT/2
4. Repeat 3×, median filter. Store: peer_offset as int32 (4 bytes).
```

No absolute time needed — only relative ordering within a session. If a third node C appears, A can provide the A→B offset, enabling transitive alignment C→A→B.

But does transitive clock alignment actually work? Let me check the drift. If A's crystal runs at 32.768 kHz ± 20 ppm and B's at ± 50 ppm, worst-case relative drift is 70 ppm = 70 µs/s = 252 ms/hour. For event correlation at the seconds granularity typical of environmental sensors, re-sync every 15-30 minutes is sufficient — 3 packets per re-sync. Viable.

For spatial reference: most of these devices don't have coordinates. Those that do (GPS-equipped) share them. The protocol handles this with a compact spatial descriptor — 12 bytes covering absolute lat/lon/alt if available, plus RSSI-based rough distance always available. If any node in a mesh has GPS, it becomes the spatial anchor and absolute coordinates propagate. If none do, positions are purely relative — a local coordinate frame rooted at the first announcing node. The frame can be FLOATING (no absolute reference), ANCHORED (at least one GPS node), or HYBRID (some anchored, some estimated).

---

## Memory Management

This is where the 64KB constraint bites hardest. Let me lay out a realistic memory map for a 32KB RAM device:

Stack gets 2KB, application state (sensor buffers, FSM) gets 4KB, the protocol gets 4KB — that's all we get — radio driver buffers take 1KB, heap/scratch 1KB, the rest is peripheral registers and DMA.

The 4KB protocol budget breaks down as follows. Own SGD and EDD manifests live in flash, zero RAM cost. Active peer sessions, at most 4 concurrently, each need: device_id (6B) + sgd_summary_hash (2B) + clock_offset (4B) + collision_map (32B for 16 pairs) + spatial_desc (12B) + session_state (2B) + last_seen_tick (4B) = 62 bytes per peer, 248 bytes for four. Peer Digest Cache for evicted peers: 32 entries at 7 bytes each = 224 bytes. Negotiation scratch buffer 256 bytes, packet buffer 256 bytes. Total: 984 bytes, well within the 4KB budget. Even on an 8KB device with a 1KB protocol budget, 4 active peers plus 8 digests fits.

The Peer Digest is the compaction mechanism. When a peer leaves RF range, its full 62-byte session compresses to 7 bytes:

```
PeerDigest {
  device_id_hash:   CRC16(device_id)       // 2B, local dedup
  sgd_summary_hash: CRC16(all SGD bytes)   // 2B, detect sensor changes
  collision_bitmap: which events collided   // 2B, up to 16 flags
  trust_score:      uint8                   // 1B, sessions without errors / total
}
```

On re-encounter, if a returning peer's `device_id_hash` and `sgd_summary_hash` match a cached digest, skip the full GROUND/RESOLVE sequence and jump to FUSED with a single confirmation packet. The collision bitmap tells us which events need disambiguation without re-negotiating. Eviction policy is LRU with trust-score weighting — high-trust, frequently-seen peers persist; one-time encounters evict first.

---

## The Full Anti-Collision Argument

Let me now state precisely why this protocol prevents abstraction collisions.

Events are never identified by name. The name_hash is explicitly metadata; identity is the physical derivation tuple. Physical grounding is non-negotiable — the SGD is transmitted in full and validated at the PGV layer, so if two events come from different transducer classes they are by construction different event types regardless of name. Same-class collisions are caught by mounting plus derivation: two accelerometers, one on a pipe and one on the ground, have different mounting values, and the GROUND phase flags them as context-distinct. Same-class, same-mounting collisions are caught by parameter distance: two ground-mounted accelerometers with different detection algorithms or different thresholds get flagged by derivation comparison. The only events that merge are physically compatible events — same transducer class, same mounting context, same derivation type, parameters within tolerance. These events genuinely refer to the same physical phenomenon. And resolution is deterministic and symmetric: both nodes run the same algorithm on the same data.

But wait, edge cases. Two identical sensors on the same device (internal + external thermistor) have different `source_sgd_idx` values and produce different EDDs — no collision. A firmware update changing a node's EDD while keeping its device_id means the cached `sgd_summary_hash` won't match, triggering full re-negotiation — correct behavior. CRC16 hash collision on `device_id_hash` in the digest cache — two different devices hashing to the same value — has probability 1/65536 per pair, and requiring *both* `device_id_hash` and `sgd_summary_hash` to collide simultaneously drops it to roughly 1/4.3 billion. Mitigation: on FUSED fast-path entry, exchange one confirmation packet with full `device_id`; if mismatch, fall back to full negotiation. One extra packet in the rare case, zero cost normally.

One scope limitation stated explicitly: adversarial nodes deliberately crafting SGDs to match another node's profile are not addressed. This is an ontology alignment protocol, not a security protocol. Authentication and integrity are an orthogonal layer.

---

## Emergent Network Properties

What happens when this isn't just two nodes but a mesh of 20 that come and go?

When Node C arrives in range of Node A (which has an active session with Node B), A runs collision detection on C's events against both its own and B's cached events. If a C–B collision is detected, A can proxy a collision report: "Node B uses this name_hash for ground-mounted accelerometer STA/LTA — your event with the same name is pipe-mounted threshold." C caches this as a pre-resolved digest for B. When C later encounters B directly, negotiation is near-instant. Cost: a few extra bytes in A's response to C.

Over time, devices that frequently co-locate develop increasingly efficient digest caches and the network develops what amounts to an ontology gradient — dense clusters of co-located devices converge on shared abstractions quickly, while bridge devices that move between clusters carry ontology mappings across the gradient. A livestock ear tag that moves from Pasture A (soil sensors, weather station) to Pasture B (water quality buoys, fence monitors) carries digests from A's devices. When B's devices encounter A's devices for the first time, perhaps via a relay, the ear tag's cached digests accelerate their negotiation. The ear tag stores 32 digests at 7 bytes each = 224 bytes, well within its 16KB RAM budget. And the digests only help with name collision avoidance, not full session state, so the transitive propagation is lightweight. This holds.

---

## Protocol Stack Summary

```
Layer 4: APPLICATION     — Fused event streams with collision-free semantics
Layer 3: RESOLUTION      — Collision detection, disambiguation, mapping tables
Layer 2: GROUNDING       — SGD + EDD exchange, clock sync, spatial alignment
Layer 1: DISCOVERY       — Beacon, announce, peer session management
Layer 0: PGV (ROM)       — Transducer class enum + SI unit enum (~256 bytes)
         Radio           — LoRa, BLE, 802.15.4, etc (out of scope)
```

The design principles in brief: names are not identity (events identified by physical derivation tuple). Ground before abstract (SGD exchanged before EDD). Collisions are structural not lexical (detection based on transducer/mounting/derivation mismatch). Memory is bounded (4 active peers + 32 digests in <1KB, digest compaction on session end). No negotiation history needed (only current state plus compact digests). Minimal shared substrate (PGV in ROM, everything else negotiated at runtime). Graceful degradation (no GPS → relative coords, no overlap → harmless DISJOINT, partial overlap → partial fusion).
