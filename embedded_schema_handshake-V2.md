Problem: Design the protocol by which embedded devices with <64KB RAM autonomously negotiate shared abstractions (event schemas, coordinate systems, temporal references) when they enter RF range of each other, without pre-shared ontologies or cloud mediation. How do you prevent "abstraction collisions" where two nodes use the same event type name for physically incompatible phenomena, given they can't store large negotiation histories?

Let me think through this carefully.

So the problem is: tiny embedded devices, under 64KB of RAM total, come into RF range of each other and need to figure out shared abstractions — event schemas, coordinate systems, time references — without any pre-agreed ontology and without phoning home to a cloud. And the specific threat is "abstraction collisions" where two devices use the same event type name for completely different physical phenomena, and they can't store big negotiation logs to work this out over time.

Before I start designing anything, let me just sit with what concretely exists here.

These are devices with less than 64KB total RAM. Not free RAM — total. So after firmware, working memory is maybe 2 to 16KB. RF communication means packet-based, lossy, bandwidth-constrained — think LoRa, BLE, Zigbee, 802.15.4. No pre-shared ontology means devices may come from different manufacturers, different firmware versions, different application domains entirely. No cloud means everything resolves locally within RF range. And the core threat — two devices both emit `TEMP_HIGH` but one means ambient air temperature above 40°C and the other means motor winding temperature above 120°C.

What does "negotiating shared abstractions" actually mean at this scale? It's not OWL ontology alignment. Not semantic web. Not even JSON schema exchange. At 64KB total RAM we're talking about devices that can maybe hold 20 to 50 active abstraction bindings simultaneously. The negotiation protocol itself must fit in perhaps 1 to 2KB of code.

Wait — I'm already reaching toward protocol design and I haven't even mapped the failure modes yet. Let me get specific about what actually collides.

Concrete collision scenarios:

1. Name collision, different physics. Device A is a weather station, Device B is a motor controller, both broadcast `event_type: 0x03` meaning completely different phenomena.

2. Same physics, different frames. Two accelerometers both report `ACCEL_X` but their coordinate systems are rotated 90° relative to each other.

3. Same physics, different scales. Both report `TEMP` but one in Celsius, one in Fahrenheit, and neither has room to store a conversion table.

4. Temporal drift. Device A timestamps from boot, Device B timestamps from last GPS fix, neither can store the other's epoch.

5. Asymmetric capability. Device A senses temperature and humidity. Device B senses temperature and vibration. Their "temperature" concepts might align but they can't verify this without grounding.

OK so stepping back from these cases — the core problem is grounding without shared priors and without memory to build them incrementally. This is essentially the symbol grounding problem under extreme resource constraints.

---

Now. Two fundamental approaches are pulling at me simultaneously.

Branch A: structural identity. Don't negotiate names at all. Negotiate physical signatures. If two devices can demonstrate they're measuring the same physical phenomenon by correlation of readings, the name is irrelevant.

Branch B: minimal ontology bootstrap. Embed a tiny universal "physics alphabet" in firmware — not a full ontology, but enough dimensional analysis (mass, length, time, temperature, current — the SI base units) to detect collisions structurally.

Let me be honest about both. Branch A is elegant but requires simultaneous sensing of the same phenomenon, which may not be available. Branch B requires pre-shared structure, which the problem statement forbids. Neither alone is sufficient.

But hold on — the problem says "no pre-shared ontologies," not "no pre-shared anything." Every RF protocol already shares something: packet format, modulation scheme, at minimum a frequency band. The real question is what is the minimal shared primitive that enables abstraction negotiation without constituting an ontology?

Let me get very specific about what that primitive could be.

A dimensional signature. Not names, not units, not scales — just the dimensional exponents of what's being measured. Temperature is `[0,0,0,1,0,0,0]` in SI base dimensions (the 4th base quantity, thermodynamic temperature). Force is `[1,1,-2,0,0,0,0]` (kg·m·s⁻²). This is not an ontology — it's physics itself. It's 7 small integers. 7 bytes.

Am I hand-waving the distinction between "shared physics" and "shared ontology" here? Let me check. An ontology assigns meaning and relationships. Dimensional analysis is a constraint system derivable from physical law. Two independently designed devices that measure force will arrive at the same dimensional signature if they track dimensions at all. This is closer to shared physical reality than shared convention.

But — many embedded devices don't track SI dimensions internally. A thermistor ADC just produces a 10-bit number. The firmware knows it's temperature, but that knowledge lives in code semantics, not in an exportable structure. So we need a registration format at firmware build time. This is a design constraint on participating devices: the protocol spec says "if you want to negotiate, you must be able to emit a dimensional signature for each of your data channels." That's about 7 bytes per channel. Feasible.

First design decision then: the minimal shared primitive is the SI dimensional exponent vector. This is not an ontology; it is a physical invariant.

---

Now let me formalize the collision problem against this primitive.

Each device D_i has channels. Each channel carries:

```
local_name:   uint8    (device-local event type ID)
dim_sig:      int8[7]  (SI dimensional exponents: kg, m, s, K, A, mol, cd)
range:        float16[2]  (physical range min/max in SI base units)
sample_rate:  float16  (Hz)

Total per channel: 1 + 7 + 4 + 2 = 14 bytes
Device with 8 channels: 112 bytes for full self-description
```

Abstraction collision now has a formal definition:

```
COLLISION(c_a, c_b) :=
  (c_a.local_name == c_b.local_name) ∧ (c_a.dim_sig ≠ c_b.dim_sig)

SHADOW(c_a, c_b) :=
  (c_a.dim_sig == c_b.dim_sig) ∧ (c_a.range ∩ c_b.range == ∅)
  // Same dimension but non-overlapping ranges — likely different phenomena
  // e.g., both "temperature" but 0–50°C vs 500–2000°C
```

Variable bindings to be explicit: c_a comes from device A, c_b from device B, different devices. dim_sig comparison is element-wise equality of all 7 exponents. Range intersection is interval intersection on SI-unit-converted ranges.

The key realization: collisions are detectable from the channel descriptors alone, without any negotiation history. Dimensional signatures make most collisions structurally visible.

---

Now the protocol itself. Let me let the structure of the problem dictate the phases. I'm going to call this GROUND — Grounded Resolution Of Unshared Naming Domains.

The question driving this section is: the protocol must handle encounter, descriptor exchange, collision detection, and binding establishment in bounded memory and bounded rounds. Let me follow the coherence of that sequence.

**Encounter phase.** When two devices enter RF range:

```
BEACON (broadcast, periodic)
┌──────────┬──────────┬───────────┬───────────┐
│ proto_ver │ dev_hash │ n_channels│ cap_flags │
│ uint4     │ uint16   │ uint4     │ uint8     │
└──────────┴──────────┴───────────┴───────────┘
Total: 4 bytes + preamble
```

`dev_hash` is a truncated hash of device identity. Not a full UUID — can't afford it. 16 bits gives collision probability of about 1/65536 per pair, acceptable for local RF neighborhoods under 100 devices. `cap_flags` are bit flags for protocol capabilities: supports correlation verification, supports temporal sync, and so on.

Quick sanity check — 4-byte beacon realistic? BLE advertising has 31 bytes available, LoRa up to 255, 802.15.4 has 127. Yes, fine.

**Descriptor exchange phase.** On detecting a new dev_hash, initiate directed exchange:

```
DESCRIBE (unicast, one per channel)
┌────────┬─────────┬────────┬────────┬──────────────┐
│ chan_id │ dim_sig │ range  │ rate   │ name_hash    │
│ uint4  │ int4[7] │ f16[2] │ f16    │ uint8        │
└────────┴─────────┴────────┴────────┴──────────────┘
Total: ~11 bytes per channel
```

I reduced dim_sig to int4[7] here — 4 bits each, so 3.5 bytes, pack to 4. SI exponents rarely exceed ±3 in practice. kg¹·m²·s⁻³ for power is about the most extreme common case. 4 bits signed gives ±7. More than sufficient.

`name_hash` is an 8-bit hash of the device's local name string. Not for identification — for detecting when two devices happen to use the same local name, enabling human-readable collision warnings.

Memory cost for tracking one neighbor with 8 channels: 11 × 8 = 88 bytes plus 2 for dev_hash plus overhead, roughly 100 bytes per neighbor. With a 2KB working memory budget for the protocol, that's about 20 simultaneous neighbors. Realistic for a local RF mesh.

**Collision detection phase.** Here's the detection algorithm:

```
function detect_collisions(my_channels[], their_channels[]):
    bindings = []
    for each mc in my_channels:
        for each tc in their_channels:
            if mc.dim_sig == tc.dim_sig:
                overlap = range_intersect(mc.range, tc.range)
                if overlap > 0:
                    // CANDIDATE ALIGNMENT
                    bindings.push({
                        local: mc.chan_id,
                        remote: tc.chan_id,
                        confidence: overlap_ratio(overlap, mc.range, tc.range),
                        status: UNVERIFIED
                    })
                else:
                    // SHADOW: same dimension, disjoint ranges
                    bindings.push({
                        local: mc.chan_id,
                        remote: tc.chan_id,
                        confidence: 0,
                        status: SHADOW
                    })
            else if mc.name_hash == tc.name_hash:
                // NAME COLLISION: same name, different physics
                // This IS the "abstraction collision"
                emit_warning(COLLISION, mc, tc)
    return bindings
```

Loop bounds: at most 8 × 8 = 64 iterations. O(1) per iteration. No memory scaling issue.

This is the collision prevention mechanism. Dimensional signatures make collisions structurally impossible for aligned channels and structurally detectable for misaligned ones. Two devices cannot accidentally agree that `TEMP_HIGH` means the same thing if one has dim_sig `[0,0,0,1,0,0,0]` (temperature) and the other has `[1,1,-2,0,0,0,0]` (force).

Now I need to test this against the null hypothesis that dimensional signatures are just another ontology in disguise. An ontology requires agreement on categories, relationships, and interpretive rules. Dimensional signatures require agreement only on what the 7 SI base quantities are — which is not a convention but a physical standard embedded in the International System of Units, itself grounded in physical constants (speed of light, Planck constant, etc.) since 2019. A device that measures temperature in any system can convert to kelvin exponents at build time. This is not ontological agreement; it is metrological grounding.

But wait — what about dimensionless quantities? Ratios, counts, probabilities, angles? These all have dim_sig `[0,0,0,0,0,0,0]`. They will collide. A "packet count" and a "humidity ratio" are both dimensionless.

I need to actually address this, not wave at it. For dimensionless channels we need a secondary discriminator. Options: a semantic tag byte — a coarse category from a small fixed enum (COUNT, RATIO, ANGLE, INDEX, STATUS, PROBABILITY, maybe 16 values, 4 bits). This is a minimal shared vocabulary, but it's 16 items, not an ontology. Or a statistical signature: mean, variance, and autocorrelation at lag-1, packed into 6 bytes. Two dimensionless quantities with wildly different statistical profiles are distinguishable. Or both — tag byte for coarse discrimination, statistical signature for verification.

Both is best. The tag byte handles the common case cheaply; the statistical signature handles edge cases without requiring vocabulary expansion. Memory cost: 1 + 6 = 7 additional bytes per dimensionless channel only.

Second design decision: dimensional signatures prevent collisions for dimensioned quantities. Dimensionless quantities get a secondary discriminator — coarse semantic tag plus statistical fingerprint.

---

**Coordinate system negotiation.** Let me get specific. Two accelerometers with different orientations. This isn't a naming problem — it's a frame problem. Both devices correctly report dim_sig `[0,1,-2,0,0,0,0]` (acceleration, m·s⁻²) but their X/Y/Z axes are arbitrarily oriented.

Key insight: you cannot negotiate coordinate alignment without either a shared external reference (gravity, magnetic north) or correlated motion.

```
FRAME_REF (extension to DESCRIBE for vector channels)
┌──────────┬──────────────────────────────────────────┐
│ ref_type │ ref_data                                 │
│ uint4    │ variable                                 │
│          │                                          │
│ 0x0: NONE (no reference available)                  │
│ 0x1: GRAVITY_ALIGNED (Z = gravity direction)        │
│ 0x2: MAGNETIC_NORTH (X = magnetic north)            │
│ 0x3: GRAVITY+NORTH (full 3D frame)                  │
│ 0x4: BODY_FIXED (axes relative to enclosure marks)  │
│ 0x5: CORRELATION_AVAILABLE (will do live alignment)  │
└──────────┴──────────────────────────────────────────┘
```

If both devices report GRAVITY_ALIGNED, their Z axes are parallel or antiparallel — sign convention still needs 1 bit. If one reports GRAVITY_ALIGNED and the other NONE, the aligned device can serve as reference and the other can attempt correlation-based alignment if they share a rigid body or experience common vibration.

I'm starting to design a full IMU alignment protocol and that's scope creep. The key structural point is this: vector quantities carry a frame_ref tag. Frame alignment is a post-binding operation that uses the negotiated channel bindings as input. The protocol provides the hooks — frame reference type, correlation capability flag — but frame alignment itself is application-layer.

Third design decision: vector quantities carry frame_ref tags. Frame alignment is post-binding, application-layer. The protocol provides hooks, not solutions.

---

**Temporal reference negotiation.** Concretely: two devices, different time bases.

```
Device A: timestamps from boot, 32-bit milliseconds, wraps at ~49.7 days
Device B: timestamps from GPS epoch, 32-bit seconds, wraps at ~136 years
```

These devices cannot align their clocks without exchanging at least one synchronized event.

```
TIME_SYNC protocol:
  Step 1: A sends PING with A's timestamp t_A1
  Step 2: B responds PONG with {t_A1, t_B1, t_B2}
          where t_B1 = B's time at PING receipt
                t_B2 = B's time at PONG send
  Step 3: A records t_A2 at PONG receipt

  Offset = ((t_B1 - t_A1) + (t_B2 - t_A2)) / 2
  (Standard NTP-style calculation)

  Preceded by EPOCH_DESC exchange:
  ┌───────────┬──────────┬──────────┐
  │ epoch_type│ tick_unit│ bit_width│
  │ uint4     │ uint8    │ uint4    │
  │ 0: BOOT   │ (μs exp) │ e.g. 32 │
  │ 1: GPS    │ e.g. 3   │          │
  │ 2: UNIX   │ (= ms)   │          │
  │ 3: CUSTOM │          │          │
  └───────────┴──────────┴──────────┘
  Total: 3 bytes
```

`tick_unit` as a power-of-10 microseconds exponent: 0 = 1μs, 3 = 1ms, 6 = 1s. Compact and unambiguous.

After TIME_SYNC, each device stores a single int32 offset per neighbor. 4 bytes. All subsequent event timestamps from that neighbor can be translated. Memory cost: 4 bytes per neighbor for temporal alignment. With 20 neighbors that's 80 bytes.

---

Now let me actually budget the RAM. This matters — the whole design lives or dies on whether it fits.

```
GROUND Protocol Memory Budget (assuming 4KB allocation from 64KB device)

STATIC (compiled in, not negotiated):
  Own channel descriptors:     8 channels × 14 bytes  =  112 bytes
  Protocol state machine:      ~200 bytes code-resident state
  Subtotal:                    ~312 bytes

PER-NEIGHBOR (dynamic):
  dev_hash:                    2 bytes
  channel descriptors:         8 × 11 bytes = 88 bytes
  bindings:                    8 × 6 bytes  = 48 bytes
  time offset:                 4 bytes
  last_seen timestamp:         4 bytes
  link quality:                1 byte
  Subtotal:                    ~147 bytes per neighbor

  At 4KB budget: floor((4096 - 312) / 147) = 25 neighbors max

EVICTION POLICY:
  LRU by last_seen, with binding-count weighting
  (neighbors with more active bindings are stickier)

NEGOTIATION BUFFER (temporary, reusable):
  Packet assembly/parse:       64 bytes
  Collision detection scratch: 64 bytes
  Subtotal:                    128 bytes (shared, not per-neighbor)
```

And the concrete data structure:

```c
typedef struct {
    uint16_t dev_hash;
    struct {
        uint8_t chan_id : 4;
        int8_t  dim_sig[4];    // packed: 7 × 4-bit in 4 bytes
        uint16_t range[2];     // float16 min, max
        uint16_t rate;         // float16 Hz
        uint8_t  name_hash;
    } channels[8];
    struct {
        uint8_t local_chan : 4;
        uint8_t remote_chan : 4;
        uint8_t confidence;    // 0–255 scaled
        uint8_t status;        // UNVERIFIED, VERIFIED, SHADOW, COLLISION
    } bindings[8];
    int32_t time_offset;
    uint32_t last_seen;
    uint8_t link_quality;
} neighbor_t;  // ~147 bytes

neighbor_t neighbors[25];
```

Quick check on the dim_sig packing: 7 × 4-bit values = 28 bits, fits in 4 bytes (32 bits) with 4 bits spare. Pack as `[kg:4][m:4][s:4][K:4][A:4][mol:4][cd:4][spare:4]`. That works.

---

Now I need to be thorough about what can still go wrong even with dimensional signatures. Let me push the edges of this design.

Taxonomy of residual collision risks:

```
COLLISION TYPE               DIM_SIG CATCHES IT?    RESIDUAL RISK
────────────────────────────────────────────────────────────────────
Different physics,           YES (different          None
different dimensions         exponents)

Same dimension,              PARTIALLY (range        Two sensors measuring
different phenomenon         overlap check)          same-dimension, same-range
                                                     but different phenomena
                                                     (air temp vs water temp,
                                                     both 0–50°C)

Dimensionless quantities     NO (all zeros)          Handled by semantic tag +
                                                     statistical fingerprint

Same phenomenon,             N/A (not a collision    Handled by frame_ref
different frame              but a misalignment)

Same phenomenon,             PARTIALLY (range in     Handled by tick_unit for
different units/scale        SI resolves this)       temporal; SI conversion
                                                     for all others
```

The residual risk is same dimension, overlapping range, different physical source. Example: Device A measures indoor air temperature, 0–50°C. Device B measures skin temperature, 25–42°C. Both have dim_sig `[0,0,0,1,0,0,0]`, overlapping ranges.

For devices that flag CORRELATION_AVAILABLE in their capability bits, there's a verification step:

```
VERIFY_BINDING protocol:
  1. Both devices simultaneously sample their bound channels
     for N samples (N=16 sufficient for basic correlation)
  2. Exchange compressed sample vectors:
     16 × uint8 (normalized to channel range) = 16 bytes
  3. Compute Pearson correlation coefficient r
  4. If |r| > threshold (e.g., 0.7): binding VERIFIED
     If |r| < threshold: binding SUSPECT → SHADOW
```

Memory cost: 32 bytes temporary for two 16-byte sample vectors. Reusable buffer. Computation cost: Pearson correlation on 16 uint8 values — about 200 multiply-accumulates. Trivial even for 8-bit MCUs.

But I need to be honest about a limitation here. This only works if the two channels should be correlated. Indoor air temp and skin temp in the same room might correlate weakly — both respond to HVAC changes — or not at all. The correlation check is a positive confirmation tool, not a negative exclusion tool. Absence of correlation doesn't prove different phenomena; it might mean different locations or different response times.

Fourth design decision: correlation verification is a one-way ratchet. High correlation upgrades confidence. Low correlation flags for human review but doesn't auto-reject. This prevents false negatives from destroying valid bindings.

The full binding lifecycle then:

```
UNVERIFIED ──correlation──→ VERIFIED
     │                          │
     │ (no correlation          │ (device leaves RF range)
     │  capability)             │
     │                          ▼
     │                      DORMANT ──re-encounter──→ VERIFIED
     │                                                (skip negotiation,
     │                                                 verify time offset only)
     ▼
  ACTIVE_UNVERIFIED ──timeout──→ EXPIRED ──evict──→ (freed)
     │
     │ (collision detected post-hoc)
     ▼
  COLLISION ──human_review──→ RESOLVED | REJECTED
```

Wait — does this handle the case where a device is replaced by a different device with the same dev_hash? Hash collision. If Device A remembers Device B's bindings as DORMANT, and a new Device C arrives with the same dev_hash, A would skip negotiation and apply stale bindings. That's a real failure mode.

Mitigation: on re-encounter, always exchange a session_nonce — 2 bytes, random. If the nonce doesn't match the stored one, treat as new encounter regardless of dev_hash match. Memory cost: 2 bytes per neighbor. Acceptable. The binding lifecycle is sound with this addition.

---

The full protocol state machine:

```
                    ┌──────────┐
                    │   IDLE   │ (listening for beacons)
                    └────┬─────┘
                         │ recv BEACON with unknown dev_hash
                         ▼
                    ┌──────────┐
                    │ EXCHANGE │ send/recv DESCRIBE for all channels
                    └────┬─────┘
                         │ all descriptors received
                         ▼
                    ┌──────────┐
                    │  DETECT  │ run collision detection algorithm
                    └────┬─────┘
                         │
                    ┌────┴────┐
                    │         │
              collisions   no collisions
              detected     (or only SHADOWs)
                    │         │
                    ▼         ▼
              ┌──────────┐  ┌──────────┐
              │   WARN   │  │   BIND   │ create bindings
              │  (emit   │  └────┬─────┘
              │ COLLISION│       │
              │  events) │       │ correlation capable?
              └────┬─────┘       │
                   │        ┌────┴────┐
                   │       yes        no
                   │        │         │
                   │        ▼         ▼
                   │  ┌──────────┐  ┌──────────┐
                   │  │  VERIFY  │  │  ACTIVE  │
                   │  │(correlate│  │(unverified│
                   │  │ samples) │  │ bindings) │
                   │  └────┬─────┘  └──────────┘
                   │       │
                   │       ▼
                   │  ┌──────────┐
                   │  │  ACTIVE  │ (verified bindings)
                   │  └──────────┘
                   │
                   ▼
              ┌──────────┐
              │  ACTIVE  │ (with collision warnings
              │ PARTIAL  │  on flagged channels)
              └──────────┘
```

Total packet types: BEACON (4B), DESCRIBE (11B × n_channels), TIME_SYNC (12B round-trip), VERIFY_REQ (2B), VERIFY_DATA (18B), COLLISION_WARN (6B). Six packet types. Fits in a 3-bit type field.

---

Now — the problem specifically asks about devices that can't store large negotiation histories. The protocol as designed stores zero negotiation history. It stores only current neighbor state (147 bytes per neighbor), current bindings embedded in that state, no logs, no transaction records, no negotiation transcripts.

Why this works: the dimensional signature approach makes negotiation idempotent. If two devices lose contact and re-encounter, they can re-negotiate from scratch and arrive at the same bindings, modulo time offset recalculation. There is no accumulated state that would be lost. The only "history" that matters is the session_nonce for detecting device replacement, and that's 2 bytes — not a history, a checkpoint.

Is idempotent re-negotiation actually true though? What if Device A's firmware updates between encounters, changing its channel configuration? Then the old bindings should be invalidated. The re-negotiation from scratch correctly handles this — it's a feature, not a bug.

---

What else breaks this? Let me do one pass through the failure space.

Adversarial devices. A malicious device could claim false dimensional signatures. Out of scope — this is a protocol for cooperative devices. If security is needed, add a 4-byte HMAC to DESCRIBE packets using a pre-shared group key. But that's a security layer, not an abstraction negotiation concern.

Devices with no dimensional awareness. Legacy devices that just broadcast raw ADC values. They can't participate in the GROUND protocol. They're invisible to it. A gateway device could proxy for them if it knows their channel semantics.

Rapidly changing neighborhoods. 50 devices entering and leaving range every minute. Handled by LRU eviction with binding-count weighting. Devices with active, verified bindings persist; transient contacts are evicted.

Dimensional signature ambiguity. Voltage (kg·m²·s⁻³·A⁻¹) and EMF have the same dimensions. They're physically different but dimensionally identical. Accepted risk: at this level of resource constraint, we accept that dimensionally identical quantities with overlapping ranges may be falsely bound. Correlation verification catches most of these. The remaining cases require human or application-layer disambiguation.

No further structural vulnerabilities that aren't handled by existing mechanisms.

---

The complete system:

```
┌──────────────────────────────────────────────────────────────┐
│                     GROUND PROTOCOL                           │
│          (Grounded Resolution Of Unshared Naming Domains)     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  LAYER 1: PHYSICAL GROUNDING                                  │
│    Dimensional Signature: int4[7] SI base exponents           │
│    Range: float16[2] in SI base units                         │
│    Rate: float16 in Hz                                        │
│    → Prevents collisions between different physics            │
│                                                               │
│  LAYER 2: STATISTICAL DISCRIMINATION                          │
│    Semantic tag: uint4 (for dimensionless quantities)         │
│    Statistical fingerprint: mean/var/autocorr (6B)            │
│    → Handles dimensionless quantity collisions                │
│                                                               │
│  LAYER 3: CORRELATION VERIFICATION                            │
│    16-sample exchange, Pearson r, one-way ratchet             │
│    → Confirms same-dimension bindings empirically             │
│                                                               │
│  LAYER 4: FRAME & TIME ALIGNMENT                              │
│    frame_ref tag for vector quantities                        │
│    NTP-style time sync with epoch descriptor                  │
│    → Resolves coordinate and temporal misalignment            │
│                                                               │
│  MEMORY: 312B static + 147B/neighbor × 25 = ~4KB total       │
│  PACKETS: 6 types, largest 18 bytes                           │
│  NEGOTIATION HISTORY: 0 bytes (idempotent re-negotiation)     │
│  COLLISION DETECTION: O(n²) in channels, O(1) in neighbors   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

Final test against two null hypotheses.

First: "This is overengineered; devices should just use a shared registry of event type IDs." A shared registry is a pre-shared ontology, which the problem forbids. It also doesn't scale — every new device type requires registry updates, and registry versioning in disconnected networks is an unsolved coordination problem. The GROUND protocol requires only that devices can express their measurements in SI dimensional terms, which is a physical invariant, not a social convention.

Second: "Dimensional signatures are insufficient; you need full semantic descriptions." Full semantic descriptions don't fit in 64KB devices. Dimensional signatures catch roughly 90% of collisions structurally. The remaining 10% — same dimension, same range, different source — are caught by correlation verification or flagged for application-layer resolution. Perfect is the enemy of deployable.

The protocol is complete, fits within resource constraints, and addresses the core collision problem through physical grounding rather than ontological agreement. The key architectural insight is that physics itself is the shared ontology. SI dimensional analysis provides a universal, pre-existing, non-negotiable basis for abstraction alignment that costs 7 nibbles per channel to express.
