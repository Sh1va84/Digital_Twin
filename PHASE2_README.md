# Warehouse Digital Twin — Phase 2 Architecture (Multi-Camera, WMS-Integrated)


**Scope:** Phase 2 = "Warehouse-Wide Operational Intelligence." Many cameras (50+), multiple edge boxes, one warehouse fully covered, **integrated with the WMS/WCS**.
**Builds on:** [PHASE1_ARCHITECTURE.md](PHASE1_ARCHITECTURE.md). Phase 1 proved one camera → trustworthy live map. Phase 2 makes it **whole-building** and **business-aware**.

> **Headline decision (read this first):** The most valuable thing in Phase 2 is **NOT multi-camera for its own sake** — it's **WMS integration**. Multi-camera gives you *coverage*; WMS integration gives you *meaning*. "23 people in Picking" becomes "Picking is 15% understaffed for tonight's order volume." We build both, but WMS is the value headline. Robots stay in Phase 3 — see §12.

---

## 0. What Phase 2 adds over Phase 1

| Capability | Phase 1 | Phase 2 | Difficulty |
|---|---|---|---|
| Cameras | 1–8, single box | 50+, multiple edge boxes | Medium |
| Tracking | Per-camera (independent IDs) | **Cross-camera global IDs (re-ID)** | **HARD — main risk** |
| Business context | None | **WMS/WCS/ERP join** | Medium (API) |
| PPE monitoring (helmet/vest) | ❌ | ✅ | Medium |
| Restricted-zone access | ❌ | ✅ | Easy (geometry) |
| Dock / yard optimization | ❌ | ✅ | Medium |
| Historical analytics | ❌ | ✅ (replay engine) | Easy (log already exists) |
| Event playback | ❌ | ✅ | Medium |
| Orchestration | Docker Compose, 1 host | **K8s server + fleet-managed edge** | Medium |

The Phase 1 event-log-as-source-of-truth design is exactly what makes historical analytics and playback nearly free in Phase 2 — we just replay the log.

---

## 1. Architecture (multi-edge → central server)

```
        ┌─── EDGE BOX 1 (12 cams) ───┐   ┌─── EDGE BOX 2 (12 cams) ───┐   ... up to 5 boxes
        │ decode→YOLO→ByteTrack      │   │ decode→YOLO→ByteTrack      │
        │ →ReID embed→homography     │   │ →ReID embed→homography     │
        │ →edge emitter + spool      │   │ →edge emitter + spool      │
        └──────────┬─────────────────┘   └──────────┬─────────────────┘
                   │ track events + ReID embeddings (mTLS, events only)
                   ▼                                  ▼
        ╔══════════════════════ SERVER TIER (K8s, no GPU*) ══════════════════════╗
        ║                                                                         ║
        ║   Ingest API (FastAPI) ──►  Kafka / Redpanda  (durable event bus)        ║
        ║                                     │                                    ║
        ║              ┌──────────────────────┼───────────────────────┐           ║
        ║              ▼                       ▼                       ▼           ║
        ║   ┌─────────────────┐      RAW EVENT LOG            WMS Connector         ║
        ║   │ Global Re-ID /  │      (TimescaleDB,            (orders, tasks,       ║
        ║   │ Track Stitcher  │◄──── source of truth)         inventory, labor)     ║
        ║   │ camera topology │              │                        │            ║
        ║   └────────┬────────┘              │                        │            ║
        ║            │ global_track_id        │                        │            ║
        ║            ▼                        ▼                        ▼            ║
        ║   ┌──────────────────────── DERIVER (live) + REPLAYER (batch) ─────────┐  ║
        ║   │  same code · consumes events + WMS + zone/rule config (versioned)   │ ║
        ║   └───┬─────────┬──────────┬──────────┬───────────┬──────────┬─────────┘  ║
        ║       ▼         ▼          ▼          ▼           ▼          ▼            ║
        ║  Twin State  Metrics   Heatmaps   Safety/PPE   Dock/Yard   Labor-vs-     ║
        ║  (Redis)    (Timescale) (grids)   + Restricted  state       Plan         ║
        ║       └─────────┴──────────┴──────────┴───────────┴──────────┘            ║
        ║                                   │                                       ║
        ║                 FastAPI (REST + WebSocket) ──► Next.js Dashboard          ║
        ║                 + Replay/Playback service ──► historical scrubber         ║
        ╚═════════════════════════════════════════════════════════════════════════╝
        * Optional small GPU on server only if ReID matching is moved server-side. Default: embeddings on edge, matching on CPU server-side.
```

---

## 2. The hard part: cross-camera tracking (global re-ID)

This is the single biggest technical risk in Phase 2. Per-camera ByteTrack gives a `track_id` that is only meaningful **within one camera**. To say "Worker #88 walked from Receiving to Picking" you must stitch IDs across cameras.

**Approach (two complementary signals — don't rely on appearance alone):**
1. **Spatial-temporal handoff (primary, cheap, reliable).** Cameras are calibrated to a *shared floor plan* (Phase 1 homography). When a track exits camera A's field of view at world point `P` and a new track appears in camera B near `P` within `Δt`, they're the same object. A **camera adjacency graph** (which cameras' FOVs touch/overlap) constrains matching to neighbours only.
2. **Appearance embedding (secondary, disambiguation).** A lightweight ReID CNN (e.g. OSNet, 256-d embedding) produces a vector per track. Cosine similarity disambiguates when two candidates are spatially plausible. **Only computed in handoff zones**, not every frame — this keeps it cheap (see §6 math).

**Why this works in a warehouse specifically:** fixed cameras, known overlapping coverage, constrained movement along aisles → spatial-temporal matching does ~80% of the work. Appearance only breaks ties. This is far more reliable than appearance-only re-ID (which fails badly on people in identical hi-vis vests — and *everyone* in a warehouse wears the same vest).

**Honest limitation:** coverage gaps (un-cameraed corridors) create "teleport" ambiguity. Mitigate with a *gap-transit model* (expected travel time between adjacent FOVs) and accept that identity may reset across large blind spots. We track *flows and counts* reliably; perfect lifelong per-person paths are a stretch goal, not a promise.

---

## 3. WMS / WCS integration (the value engine)

The connector ingests, via the WMS API or a read-replica:
- **Tasks / pick waves** — who *should* be doing what, where, by when.
- **Orders / volume** — demand for the shift.
- **Inventory / slotting** — what's in which zone.
- **Labor / roster** — planned headcount per zone per shift.

**Join key = zone + time.** The twin already knows *actual* occupancy/activity per zone per minute (anonymous). The WMS knows *planned* work per zone. Joining them produces the metrics managers actually act on:

```json
{ "zone":"Picking", "ts":"...",
  "actual_workers":18, "planned_workers":24, "staffing_gap":-6,
  "open_pick_tasks":312, "throughput_picks_per_hr":410, "required_rate":520,
  "status":"behind_plan", "projected_miss_minutes":35 }
```

**Privacy note:** the twin stays **anonymous** (zone counts, not named individuals). The WMS supplies the *work* context; we never join a camera track to a named employee. Keep that boundary explicit — it's what makes the safety/labor story defensible.

---

## 4. Safety & dock modules (now cheap, because the twin exists)

- **PPE (helmet / vest):** run a small classifier on **person crops only**, **only in zones flagged `ppe_required`**. Not every person every frame — sample 1 Hz per track. Produces `ppe_violation` events with zone + dwell.
- **Restricted-zone access:** pure geometry — track's `world_xy` ∈ restricted polygon AND not authorized-class → alert. Near-zero compute.
- **Dock / yard:** dock occupancy (truck in dock polygon), queue length (trucks in yard staging), turnaround timer (truck-in → truck-out), dwell. Feeds the detention-penalty business case.

---

## 5. Historical analytics & playback (almost free)

Because the raw event log is append-only and complete, both are *replays*:
- **Historical analytics** = run the same Deriver over a past time range → trends, shift comparisons, zone utilization over weeks.
- **Event playback** = reconstruct twin state at any past timestamp and scrub forward (the dashboard reads historical twin snapshots instead of live). No video needed — we replay *events*, which is why this is cheap and privacy-safe.

---

## 6. Capacity & performance math (will it actually work?)

> **All numbers are engineering estimates for an RTX 4000 Ada-class GPU per edge box, 1080p H.264 cameras, YOLO11m FP16 TensorRT, 5 fps active / tiered. Validate against your real cameras — these are sizing math, not benchmarks.**

### 6.1 Per-edge-box compute (50 cameras across 5 boxes = 10 cams/box, design for 12)
Per frame, per camera:
- YOLO11m FP16 inference: **~4 ms**
- ReID embedding (only in handoff zones, avg ~2 crops/frame × 1.5 ms): **~3 ms**
- ByteTrack + homography (CPU, parallel to GPU): ~negligible to GPU

Effective GPU per frame ≈ **7–8 ms**.

```
Per box: 12 cameras × 5 fps          = 60 inferences/sec
GPU time                              = 60 × 8 ms = 480 ms per wall-second
GPU utilization                       ≈ 48%   ✅ comfortable headroom
NVDEC decode (12× 1080p)              ≈ well within NVDEC limit (16–24 streams)  ✅
```
**Verdict per box: 12 cameras at 5 fps runs at ~50% GPU. Safe. → 50 cameras = 5 boxes (with margin). Could squeeze to 4.**

### 6.2 Event volume (the part people get wrong)
Naive: 50 cams × 10 objects × 5 fps = **2,500 track events/sec**. At 5 fps the position stream is wasteful for the *log*.

**Mitigation — decouple tracking fps from logging fps:** track internally at 5 fps, **emit position updates to the log at 2 Hz**, plus discrete events (zone_enter/exit, dwell, alerts) as they happen.

```
Logged events     = 50 cams × 10 objects × 2 Hz ≈ 1,000 events/sec
Event size        ≈ 300 bytes
Raw write rate    = 1,000 × 300 B   = 300 KB/s = ~26 GB/day
TimescaleDB compression (~10–15×)   → ~2 GB/day persisted   ✅
90-day retention                    → ~180 GB compressed     ✅ (single disk)
```
Kafka/Redpanda at 1,000–2,500 msg/s is **<1%** of its capacity — a non-issue.

### 6.3 Network (events only — raw video never leaves the building)
```
Per edge box: ~200–500 events/sec × 300 B   = 60–150 KB/s
5 boxes total                                ≈ 0.3–0.75 MB/s on the LAN   ✅ trivial
```
Contrast: streaming 50 raw 1080p feeds would be ~200–400 Mbps. **This is the whole point of edge processing** — we send ~6 Mbps of events instead.

### 6.4 Cross-camera matching compute (server-side, CPU)
```
Active tracks across site (peak)     ≈ 500
Matching is NOT all-pairs — constrained to camera-adjacency + Δt window
Candidate comparisons / handoff      ≈ tens, not thousands
Cosine similarity on 256-d vectors   ≈ microseconds each
Total                                = a few ms/sec of CPU   ✅ negligible
```
The adjacency graph + time window is what keeps this from exploding into an O(n²) embedding search.

### 6.5 End-to-end latency budget (target < 2.5 s, glass → dashboard)
```
decode+sample ≤ 200 ms · inference ≤ 50 ms · track+ReID ≤ 80 ms
· network ≤ 100 ms · global stitch ≤ 150 ms · derive+WMS join ≤ 400 ms
· WS push ≤ 100 ms        TOTAL ≈ 1.1 s   ✅ inside budget
```

### 6.6 Storage growth summary
| Data | Rate | 90-day | 12-month rollups |
|---|---|---|---|
| Raw events (compressed) | ~2 GB/day | ~180 GB | continuous aggregates only |
| Metrics/aggregates | small | — | ~10–20 GB |
| Heatmap hourly snapshots | small | — | ~5 GB |
| Alert clips (edge, optional) | 7-day ring | local disk | not centralized |

**Overall verdict: YES, it works — on ~5 edge boxes + one modest server VM + Kafka + TimescaleDB.** The binding constraints are *not* compute or network. They are (in order): **(1) cross-camera re-ID accuracy, (2) WMS data quality, (3) calibrating 50 homographies.** All three are *engineering/data* problems, not capacity walls.

---

## 7. How Phase 2 changes the actual work process

| Role | Before (today) | After (Phase 2) | Mechanism |
|---|---|---|---|
| **Shift supervisor** | Walks the floor, eyeballs where people are, reacts | Sees live staffing-vs-plan per zone; rebalances *before* a zone falls behind | Twin actual-occupancy joined to WMS planned-labor |
| **Safety officer** | Manual CCTV spot-checks, finds violations after the fact | Real-time PPE + restricted-zone + proximity alerts, auto-logged with evidence | PPE classifier + zone geometry + proximity rules |
| **Dock manager** | Phone calls, clipboards, guesses on truck status | Live dock occupancy, queue length, turnaround timers; spots detention risk early | Dock/yard module + appointment-system join |
| **Ops / GM** | Reactive, anecdotal ("Picking felt slow today") | Quantified shift trends, zone utilization, throughput-vs-demand history | Historical replay + analytics |
| **Continuous improvement** | Gut-feel process changes | Evidence: replay the day a bottleneck happened, see exactly where flow broke | Event playback |

**The core shift:** from **reactive + anecdotal** to **proactive + quantified**. The twin doesn't replace people — it gives every role a *minutes-ahead* view instead of an *after-the-fact* one.

---

## 8. How Phase 2 creates value (with illustrative model — fill with real baselines)

> Numbers below are an **illustrative ROI model**, not a promise. Plug in this site's real figures during the pilot.

| Value lever | Mechanism | Illustrative impact |
|---|---|---|
| **Labor rebalancing** | Staffing-vs-plan visibility shifts people to where demand is | 3–8% labor productivity gain — often the single biggest line |
| **Dock turnaround** | Live queue/turnaround cuts truck dwell, avoids detention fees | 10–20% turnaround reduction; direct $ on detention penalties |
| **Safety incident reduction** | Real-time PPE + proximity + restricted-zone | Fewer recordables → lower insurance/comp + avoided downtime |
| **Congestion / flow** | Spot and clear bottlenecks before they cascade | Higher effective throughput with same headcount |
| **Management time** | Less floor-walking and CCTV scrubbing | Supervisor hours redirected to action |

**Why the value compounds vs Phase 1:** Phase 1 told you *what is happening*. Phase 2 tells you *whether what's happening matches the plan, across the whole building, and what it's costing you* — that's the gap between a dashboard people glance at and a system the operation runs on.

**Example back-of-envelope:** a site with 200 floor staff at $20/hr, 2 shifts, ~$16M/yr labor. A *conservative 4%* productivity gain ≈ **$640k/yr** — against ~5 edge boxes (~$15–20k hardware) + a server + integration effort. Payback is measured in **weeks**, not years, *if* the labor-rebalancing lever is real for this operation. **Validate that lever first** — it's where most of the ROI sits.

---

## 9. Deployment & ops at scale
- **Edge:** still Docker Compose *per box*, but now **fleet-managed** (Ansible or Balena/Portainer) — config, model engine, and zone files pushed centrally. Each box pinned to image tags; staged rollout.
- **Server:** **Kubernetes** (or Nomad/Swarm if k8s is overkill for one site) — ingest, Kafka, TimescaleDB, derivers, API, dashboard as services with HPA on the derivers.
- **Config:** zone/rule JSON is versioned in git; the 50 homographies live in a `cameras.json` per box. A bad config bumps a version and is replayable/revertible.
- **Observability:** Prometheus + Grafana — per-camera fps/health, per-box GPU, event lag, **re-ID match rate**, WMS sync freshness, end-to-end latency SLO (<2.5 s).

## 10. Reliability additions over Phase 1
- **Per-box independence:** one edge box down = that zone goes dark on the map (flagged), rest of building unaffected. No single point of vision failure.
- **Kafka durability:** events survive deriver restarts; replay from offset.
- **WMS outage:** twin degrades gracefully to anonymous-only mode (counts without plan context) — never blocks the live map.
- **Re-ID confidence gating:** low-confidence stitches are marked provisional, not asserted — better to show "≈" than a wrong path.

## 11. Honest risk register (will it work? where it might not)
| Risk | Severity | Mitigation |
|---|---|---|
| **Cross-camera re-ID accuracy** (identical vests, blind spots) | **High** | Spatial-temporal first, appearance second; adjacency graph; provisional gating; promise *flows/counts*, not perfect lifelong paths |
| **WMS data quality/latency** | High | Validate WMS API early; degrade gracefully; cache last-good |
| **50× calibration drift** | Medium | Calibration tool + periodic auto-sanity check; re-calibrate on camera bump |
| **PPE false positives** annoy staff | Medium | Zone-scoped, dwell-confirmed, tunable; measure false-alarm budget |
| **Scope creep toward robots** | Medium | Hard line: robots = Phase 3 (§12) |
| **Compute / network / storage** | **Low** | Math in §6 shows large headroom — not the constraint |

## 12. Why robots are still NOT in Phase 2
AMRs self-localize (LiDAR/SLAM) more precisely than CCTV ever will, and they already have a fleet manager. Drawing robots on the twin adds little. The *real* robot value — using the twin's unique **human/zone truth** to keep AMRs and people safely orchestrated — requires a **mature, trusted, accurate** twin first. Phase 2's job is to *earn that trust*. Connect robots in Phase 3, once the twin is the authoritative source of human/zone state a robot would be willing to listen to.

---

### One-paragraph summary
> Phase 2 covers the whole warehouse with ~50 cameras on ~5 edge boxes, stitches per-camera tracks into global identities using shared floor-plan geometry first and appearance second, and — crucially — **joins the twin to the WMS** so "what's happening" becomes "what's happening versus plan, and what it costs." It adds PPE, restricted-zone, and dock modules almost for free on top of the existing twin, and gets historical analytics and playback essentially free by replaying the append-only event log. The capacity math (§6) shows compute, network, and storage all run with large headroom — the real risks are re-ID accuracy, WMS data quality, and calibrating 50 cameras, all solvable engineering/data problems. The payoff is the jump from a live map people glance at to a system the operation is run from — with an ROI dominated by labor rebalancing that should be validated first.
