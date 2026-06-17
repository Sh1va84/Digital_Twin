# Hardware Cost — Warehouse Digital Twin (Single Warehouse, in ₹)

GPU edge box + server cost for **one warehouse**, for **Phase 1** (1–8 cameras, 1 edge box) and **Phase 2** (50+ cameras, ~5 edge boxes).

> **Read this first — assumptions:**
> - Prices are **Indian market estimates, 2026**, in ₹, **before 18% GST**. A GST line is added per phase.
> - Prices fluctuate with **USD/INR and GPU availability** — treat as ±15%.
> - **Existing CCTV is reused** (that's the core value of this project). New cameras are an *optional* line, not assumed.
> - **Software = ₹0 licensing** — the whole stack is open source (YOLO, ByteTrack, FFmpeg, FastAPI, PostgreSQL/TimescaleDB, Redis, Next.js, Prometheus, Grafana).
> - This covers **hardware/infra only.** Engineering/development effort is a separate cost (noted at the end).

---

## 0. Quick summary

| | Budget option | **Recommended** | Premium |
|---|---|---|---|
| **Phase 1** (1 edge box + server + infra) | ~₹2,20,000 | **~₹3,50,000** | ~₹4,50,000 |
| **Phase 2** (5 edge boxes + server + infra) | ~₹11,00,000 | **~₹14,50,000** | ~₹19,00,000 |
| + 18% GST (recommended) | — | P1 **~₹4,13,000** · P2 **~₹17,11,000** | — |

> One-time capital cost. Compare with cloud GPU which is **recurring ₹4–7 lakh/year *per* GPU** (§5).

---

## 1. Phase 1 — GPU Edge Box (the "eyes")

This is the box that runs FFmpeg + YOLO + ByteTrack on the GPU, inside the warehouse.

### 1.1 Recommended build — RTX 4000 Ada (20 GB)

| Component | Spec | Cost (₹) |
|---|---|---|
| **GPU** | NVIDIA RTX 4000 Ada 20 GB (NVDEC + FP16) | 1,15,000 |
| CPU | 8-core (Ryzen 7 7700 / Core i7-13700) | 28,000 |
| Motherboard | B650 / B760 | 18,000 |
| RAM | 32 GB DDR5 | 9,000 |
| Storage | 1 TB NVMe SSD | 7,000 |
| PSU | 750 W 80+ Gold | 8,000 |
| Case + cooling | mid-tower + fans | 7,000 |
| Dual NIC | camera VLAN + uplink | 2,500 |
| Assembly + misc | cables, thermal | 5,500 |
| **Edge box total** | | **~₹2,00,000** |

### 1.2 Budget build — RTX 4060 Ti 16 GB

| Component | Spec | Cost (₹) |
|---|---|---|
| **GPU** | RTX 4060 Ti 16 GB | 52,000 |
| CPU | 6-core (Ryzen 5 7600) | 20,000 |
| Motherboard | B650 | 14,000 |
| RAM | 32 GB | 9,000 |
| Storage | 512 GB NVMe | 4,500 |
| PSU | 650 W | 6,500 |
| Case + cooling | | 6,000 |
| Dual NIC | | 2,500 |
| Assembly | | 5,000 |
| **Edge box total** | | **~₹1,20,000** |

### 1.3 Industrial / fanless alternative — Jetson Orin NX 16 GB
~₹90,000–1,10,000 for a dev kit + enclosure. **Lower power (~25 W), no fans, rugged** — good for dusty/hot warehouse corners or fewer cameras (handles ~4–6 cams comfortably). Trade-off: less headroom than a desktop GPU.

---

## 2. Phase 1 — Server (the "brain", NO GPU)

Runs ingest API, Redis, TimescaleDB, deriver, dashboard. **No GPU needed** — all AI already happened on the edge.

| Option | Spec | Cost (₹) |
|---|---|---|
| **Recommended** | Mini-PC / small tower · 8-core · 64 GB RAM · 2× 2 TB SSD | ~1,00,000 |
| Budget | 8-core · 32 GB RAM · 1 TB SSD (or reuse an existing VM) | ~60,000 |
| Premium | Rack server · 64–128 GB · RAID storage | ~1,50,000 |

---

## 3. Phase 1 — Supporting infrastructure

| Item | Cost (₹) |
|---|---|
| Managed VLAN switch (8–16 port, PoE if cameras need power) | 15,000 |
| UPS (1–2 kVA, protects edge + server) | 15,000 |
| Network cabling + installation | 20,000 |
| **Infra total** | **~₹50,000** |

### Phase 1 — TOTAL

| | Budget | **Recommended** | Premium |
|---|---|---|---|
| Edge box | 1,20,000 | 2,00,000 | 2,50,000 |
| Server | 60,000 | 1,00,000 | 1,50,000 |
| Infra | 40,000 | 50,000 | 50,000 |
| **Subtotal** | **2,20,000** | **3,50,000** | **4,50,000** |
| + 18% GST | 2,59,600 | **4,13,000** | 5,31,000 |

---

## 4. Phase 2 — Whole warehouse (50+ cameras, ~5 edge boxes)

One edge box handles ~12 cameras at 5 fps (~48% GPU — see PHASE2_README §6). 50 cameras → **5 boxes** (4 if you push it).

| Item | Qty | Unit (₹) | Total (₹) |
|---|---|---|---|
| **GPU edge boxes** (recommended build §1.1) | 5 | 2,00,000 | 10,00,000 |
| **Server** (bigger — 64–128 GB RAM, RAID, more storage) | 1 | 1,50,000 | 1,50,000 |
| Network — multiple PoE/VLAN switches for 50 cameras | — | — | 1,00,000 |
| Cabling + installation (50-camera scale) | — | — | 50,000 |
| UPS (larger / multiple units) | — | — | 50,000 |
| System integration + commissioning | — | — | 50,000 |
| **Subtotal** | | | **~₹14,00,000–14,50,000** |
| + 18% GST | | | **~₹16,50,000–17,11,000** |

**Save ₹2L:** use 4 edge boxes instead of 5 if camera layout allows → ~₹12,50,000 subtotal.
**Budget variant** (RTX 4060 Ti boxes): 5 × ₹1,20,000 + server + infra ≈ **₹11,00,000**.

### Optional — if cameras DON'T already exist
| Item | Qty | Unit (₹) | Total (₹) |
|---|---|---|---|
| IP CCTV cameras (1080p, RTSP) | 50 | 4,000–8,000 | 2,00,000–4,00,000 |

*Most warehouses already have CCTV — this is usually ₹0. Only budget it if adding/upgrading cameras.*

---

## 5. Operating cost per year (OPEX)

| Item | Phase 1 | Phase 2 |
|---|---|---|
| **Electricity** (₹8/kWh, 24×7) | ~₹35,000/yr | ~₹1,40,000/yr |
| — *edge box(es) ~350 W each + server ~150 W* | (1 box) | (5 boxes) |
| Maintenance / spares (~10% of hardware/yr) | ~₹35,000/yr | ~₹1,40,000/yr |
| Internet (events only — tiny, ~6 Mbps) | negligible | negligible |
| Software licensing | ₹0 | ₹0 |
| **Approx OPEX/year** | **~₹70,000** | **~₹2,80,000** |

> Note: raw video never leaves the building, so there are **no cloud storage or bandwidth/egress fees** — a big recurring saving (see WHY_LOCAL_SERVER.md).

---

## 6. Why local beats cloud (cost proof)

A comparable **cloud GPU instance** (T4 / A10-class) costs roughly **$0.5–1.0/hour**. Running 24×7:

```
$0.7/hr × 24 × 365 ≈ $6,100/yr ≈ ₹5,10,000 / year  — PER GPU, recurring, forever
```

- **Phase 1:** one local edge box = **~₹2,00,000 once**. Cloud equivalent = **~₹5,00,000+ every year**. Local pays for itself in **< 5 months**.
- **Phase 2:** 5 local boxes ≈ **₹10,00,000 once**. Cloud equivalent ≈ **₹25,00,000+ per year** — *plus* video egress/storage fees. Local pays back in **~5 months**, then saves ₹20L+/year.
- After year 1, local hardware is **already paid off**; cloud bills keep growing.

**Verdict:** for an always-on video workload, **buy local**. Cloud only makes sense later as a thin multi-site reporting layer (small summaries, not video).

---

## 7. What this does NOT include (separate budget lines)

| Item | Note |
|---|---|
| **Engineering / development** | Building the software (models, pipeline, dashboard, integration). This is usually the **largest** cost — a multi-month effort by a CV/full-stack/DevOps team. Budget separately. |
| **Data labeling** | Annotating warehouse frames to fine-tune YOLO (CVAT/Roboflow time or service). |
| **WMS integration** (Phase 2) | Per-WMS adapter work (SAP/Oracle/legacy differ). |
| **AMC / support contract** | Optional ongoing support. |

---

### Bottom line (one paragraph)
> For a single warehouse, **Phase 1 hardware is ~₹3.5 lakh (~₹4.1 lakh with GST)** — one GPU edge box plus a no-GPU server and basic network/UPS. **Phase 2 (whole building, 50+ cameras) is ~₹14.5 lakh (~₹17 lakh with GST)** for five edge boxes plus a larger server and network. Running cost is small (~₹70k/yr Phase 1, ~₹2.8L/yr Phase 2) because video stays on-site — no cloud fees. The same compute in the cloud would cost **₹5–25 lakh *per year*, recurring**, so local hardware pays for itself in under six months. The biggest cost not shown here is **engineering/development**, which should be budgeted on its own.
