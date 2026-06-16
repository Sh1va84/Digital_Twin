flowchart TB
     subgraph EDGE["EDGE TIER — one GPU box per facility (near cameras)"]
         CAM["CCTV cameras<br/>(RTSP)"]
         FF["FFmpeg decode<br/>tiered 1–5 fps"]
         YOLO["YOLO detect<br/>(TensorRT)"]
         BT["ByteTrack<br/>single-camera tracking"]
         EM["Edge emitter<br/>produces TRACK EVENTS"]
         CAM --> FF --> YOLO --> BT --> EM
     end


     subgraph SERVER["SERVER TIER — VM / on-prem (no GPU)"]
         ING["(1) Ingest<br/>validate"]
         LOG[("(2) RAW EVENT LOG<br/>append-only · source of truth<br/>TimescaleDB hypertable")]
         DER["(3) Deriver — LIVE<br/>zone geometry + rules"]
         REP["(3') Replayer — BATCH<br/>same logic over history"]
         CFG[/"(5) Zone/Rule CONFIG<br/>versioned JSON"/]


         subgraph DERIVED["(4) DERIVED STORES"]
             MET[("Metrics<br/>occupancy · dwell · flow")]
             STATE[("Twin State<br/>current snapshot")]
             ALERTS[("Alerts<br/>rule crossings")]
         end


         API["(6) FastAPI<br/>REST + WebSocket"]
         UI["(7) Next.js dashboard<br/>layout · counts · heatmaps<br/>alerts · playback · cam health"]
         CLIPS[("Clip store<br/>local disk + retention cron")]


         ING --> LOG
         LOG --> DER
         LOG --> REP
         CFG --> DER
         CFG --> REP
         DER --> MET
         DER --> STATE
         DER --> ALERTS
         REP --> MET
         REP --> STATE
         REP --> ALERTS
         DERIVED --> API
         API --> UI
         CLIPS --> API
     end


     EM -->|"events only<br/>(raw video stays in building)"| ING
     EM -.->|"trigger clips"| CLIPS


     classDef truth fill:#fde68a,stroke:#b45309,stroke-width:2px,color:#000;
     classDef store fill:#dbeafe,stroke:#1e40af,color:#000;
     classDef config fill:#dcfce7,stroke:#15803d,color:#000;
     class LOG truth;
     class MET,STATE,ALERTS,CLIPS store;
     class CFG config;
