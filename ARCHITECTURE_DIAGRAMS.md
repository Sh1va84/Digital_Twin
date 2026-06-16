# Warehouse Digital Twin — Architecture Diagrams (Mermaid)

Render in any Mermaid-aware viewer (GitHub, VS Code Mermaid preview, mermaid.live).

---

## Phase 1 — Single Edge Box

```mermaid
flowchart TD
    subgraph EDGE["EDGE TIER — GPU box in warehouse (raw video stays here)"]
        direction TB
        CAM["CCTV Cameras<br/>RTSP H.264/H.265 · 1-8 cams"]
        RTSP["RTSP Reader Pool<br/>FFmpeg + NVDEC · tiered 1-5 fps"]
        FQ["Frame Queue<br/>bounded · drop-oldest"]
        BATCH["Batch Builder<br/>groups frames across cams"]
        YOLO["TensorRT YOLO (FP16)<br/>person · forklift · pallet · truck"]
        ROUTE["Detection Router"]
        TRACK["Per-Camera ByteTrack<br/>stable track_id"]
        HOMO["Homography Transform<br/>pixel to floor-plan metres"]
        EMIT["Edge Event Emitter<br/>+ local SQLite spool"]
        CAM --> RTSP --> FQ --> BATCH --> YOLO --> ROUTE --> TRACK --> HOMO --> EMIT
    end

    EMIT -->|"track events only · mTLS"| INGEST

    subgraph SERVER["SERVER TIER — VM / on-prem (no GPU)"]
        direction TB
        INGEST["Ingest API · FastAPI<br/>validate"]
        REDIS["Redis Stream<br/>buffer / backpressure"]
        LOG[("RAW EVENT LOG<br/>TimescaleDB · append-only<br/>SOURCE OF TRUTH")]
        CFG[/"Zone / Rule Config<br/>versioned JSON"/]
        DERIVE["Deriver (LIVE)<br/>+ Replayer (BATCH)<br/>same code path"]
        TWIN[("Twin State<br/>Redis snapshot")]
        METRIC[("Metrics<br/>occupancy · dwell · flow")]
        HEAT[("Heatmap Grid<br/>floor-plan density")]
        ALERT[("Alerts<br/>congestion · proximity")]
        API["FastAPI<br/>REST + WebSocket"]
        DASH["Next.js Dashboard<br/>live map · counts · heatmap · cam health"]

        INGEST --> REDIS --> LOG
        CFG --> DERIVE
        LOG --> DERIVE
        DERIVE --> TWIN & METRIC & HEAT & ALERT
        TWIN & METRIC & HEAT & ALERT --> API --> DASH
    end
```

---

## Phase 2 — Multi-Camera + WMS Integration

```mermaid
flowchart TD
    subgraph EDGEFLEET["EDGE FLEET — ~5 GPU boxes · 50+ cameras (raw video stays on-site)"]
        direction LR
        E1["Edge Box 1 · 12 cams<br/>decode > YOLO > ByteTrack<br/>> ReID embed > homography > emitter"]
        E2["Edge Box 2 · 12 cams<br/>decode > YOLO > ByteTrack<br/>> ReID embed > homography > emitter"]
        EN["Edge Box N ...<br/>same pipeline"]
    end

    E1 & E2 & EN -->|"track events + ReID embeddings · mTLS"| INGEST

    subgraph SERVER["SERVER TIER — Kubernetes (no GPU)"]
        direction TB
        INGEST["Ingest API · FastAPI"]
        KAFKA["Kafka / Redpanda<br/>durable event bus"]
        LOG[("RAW EVENT LOG<br/>TimescaleDB · append-only<br/>SOURCE OF TRUTH")]
        REID["Global Re-ID / Track Stitcher<br/>camera adjacency graph<br/>spatial-temporal + appearance"]
        WMS["WMS / WCS Connector<br/>orders · tasks · inventory · labor"]
        CFG[/"Zone / Rule Config<br/>versioned JSON"/]
        DERIVE["Deriver (LIVE) + Replayer (BATCH)<br/>same code · events + WMS + config"]

        TWIN[("Twin State<br/>Redis")]
        METRIC[("Metrics<br/>Timescale")]
        HEAT[("Heatmaps")]
        SAFE[("Safety / PPE<br/>+ Restricted Zone")]
        DOCK[("Dock / Yard<br/>state")]
        PLAN[("Labor-vs-Plan")]

        API["FastAPI · REST + WebSocket"]
        PLAY["Replay / Playback Service"]
        DASH["Next.js Dashboard<br/>warehouse-wide map · staffing-vs-plan<br/>safety · dock · historical scrubber"]

        INGEST --> KAFKA
        KAFKA --> LOG
        KAFKA --> REID
        REID -->|"global_track_id"| DERIVE
        WMS --> DERIVE
        CFG --> DERIVE
        LOG --> DERIVE
        DERIVE --> TWIN & METRIC & HEAT & SAFE & DOCK & PLAN
        TWIN & METRIC & HEAT & SAFE & DOCK & PLAN --> API
        LOG --> PLAY
        API --> DASH
        PLAY --> DASH
    end
```
