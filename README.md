# EcoEye — Territorial Surveillance System
> Hackathon build · PM: Arthur · Dev: [your name]

**Mission:** A spider-web mesh of old smartphones deployed across indigenous and quilombola territories. Each node runs lightweight ML detection, generates a mini flag on threat detection, and passes it through the mesh until it reaches the Spider (central server), which validates and fires alerts to local communities, IBAMA, and the Federal Police.

**Hackathon goal:** Not a working product — a compelling visual demonstration of the idea that any leader in the room can understand instantly.

---

## Core concept: the web

```
[Node] --mesh--> [Node] --mesh--> [Node]
                                     |
                              [Node] --mesh--> [SPIDER SERVER]
                                                      |
                                         +-----------+-----------+
                                     [Community]  [IBAMA]  [Federal Police]
```

Every node is a donated old smartphone. Nodes talk to each other via Meshtastic (LoRa radio mesh — no SIM, no internet required in the field). The spider is the only node that needs internet. Mini flags hop from node to node until they reach it.

---

## Stack

| Layer | Technology | Why |
|---|---|---|
| Node hardware | Old Android smartphones | Free via donation, no specialized hardware |
| Node mesh | Meshtastic (LoRa) | No internet, works in dense forest, free |
| Node ML | MediaPipe (on-device) | Runs on weak phones, Google-maintained, free |
| Mini flag format | Lightweight JSON over Meshtastic | Small packet, fits LoRa bandwidth |
| Spider (server) | Node.js + PostgreSQL (Neon) | Simple, fast to build |
| Alert delivery | Twilio (WhatsApp + SMS) | Communities already use WhatsApp |
| Demo frontend | React + Leaflet | Visual web demo for the presentation |
| Deploy | Vercel (frontend) + Railway (API) | Free tier, instant deploy |

---

## ML models (MediaPipe on Android)

| Threat | MediaPipe model | What it detects |
|---|---|---|
| Fire / smoke | Object Detector (custom EfficientDet-Lite) | Fire class, smoke class |
| Chainsaw / gunshot | Audio Classifier | Chainsaw, gunshot, engine sounds |
| Unauthorized human | Object Detector | Person class, active at night |
| Vehicle intrusion | Object Detector | Truck, motorcycle off-road |

All models run **on-device** — no internet needed for inference. MediaPipe's EfficientDet-Lite0 runs at ~30ms on a 5-year-old Android phone.

---

## Mini flag format

What each node sends through the mesh — kept tiny to fit LoRa packet limits:

```json
{
  "nid": "node-03",
  "t": "chainsaw",
  "c": 0.91,
  "lat": -2.8451,
  "lng": -52.4142,
  "ts": 1718123456
}
```

`nid` = node ID · `t` = threat type · `c` = confidence · `lat/lng` = GPS · `ts` = unix timestamp

Total: ~90 bytes. LoRa limit: ~250 bytes. Fits comfortably.

---

## Mesh flow (node to node to spider)

```
Node-03 detects chainsaw (confidence 0.91)
  → builds mini flag JSON
  → broadcasts via Meshtastic to nearest neighbor node
  → neighbor retransmits to next node (mesh routing, automatic)
  → mini flag hops across the web until it reaches the Spider
  → Spider receives, validates confidence threshold
  → Spider saves to DB with full metadata
  → Spider fires alerts:
      WhatsApp → community leaders (message + GPS location)
      SMS → regional IBAMA
      WhatsApp → Federal Police (if human detected alongside threat)
  → Dashboard map updates in real time
  → EcoProof bridge: evidence created with chain of custody
```

---

## Folder structure

```
ecoeye/
├── node-app/                      # Android app (or Python script for demo)
│   ├── detection/
│   │   ├── vision_detector.py     # MediaPipe object detection — fire, human, vehicle
│   │   ├── audio_classifier.py    # MediaPipe audio — chainsaw, gunshot
│   │   └── threat_scorer.py       # Combines signals, applies thresholds
│   ├── mesh/
│   │   ├── meshtastic_client.py   # Connects to Meshtastic device via serial/BLE
│   │   └── flag_broadcaster.py    # Builds mini flag + sends through mesh
│   └── config.py                  # NODE_ID, GPS_LAT, GPS_LNG, thresholds
│
├── spider/                        # Central server (cloud)
│   ├── routes/
│   │   ├── ingest.js              # POST /ingest — receives mini flag from mesh gateway
│   │   ├── alerts.js              # Handles alert dispatch logic
│   │   ├── nodes.js               # GET /nodes — node registry and status
│   │   └── events.js              # GET /events — event history for dashboard
│   ├── services/
│   │   ├── validator.js           # Validates confidence, deduplicates events
│   │   ├── notifier.js            # Twilio WhatsApp + SMS dispatch
│   │   ├── rule_engine.js         # Who to alert, escalation logic
│   │   └── ecoproof_bridge.js     # Sends evidence to EcoProof API
│   └── db/
│       └── schema.sql
│
├── dashboard/                     # React frontend — visual demo
│   ├── components/
│   │   ├── WebMap.tsx             # Leaflet map showing nodes as web/mesh
│   │   ├── MeshAnimation.tsx      # Animated mini flag traveling node to node
│   │   ├── EventFeed.tsx          # Live feed of incoming threats
│   │   ├── AlertLog.tsx           # Who was notified and when
│   │   └── NodeCard.tsx           # Individual node status
│   └── pages/
│       ├── Demo.tsx               # Main hackathon demo screen
│       └── EventDetail.tsx        # Threat detail + evidence
│
└── docs/
    ├── ARCHITECTURE.md
    └── DEMO_SCRIPT.md
```

---

## Database schema

```sql
CREATE TABLE nodes (
  id TEXT PRIMARY KEY,             -- e.g. "node-03"
  territory TEXT NOT NULL,
  lat DECIMAL(9,6),
  lng DECIMAL(9,6),
  status TEXT DEFAULT 'active',    -- active | offline | alert
  last_seen TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  node_id TEXT REFERENCES nodes(id),
  threat_type TEXT NOT NULL,       -- fire | chainsaw | human | vehicle | smoke
  confidence DECIMAL(4,3) NOT NULL,
  lat DECIMAL(9,6),
  lng DECIMAL(9,6),
  severity TEXT DEFAULT 'medium',  -- low | medium | high | critical
  status TEXT DEFAULT 'pending',   -- pending | alerted | resolved | false_positive
  raw_flag JSONB,                  -- original mini flag as received
  detected_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE alert_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID REFERENCES events(id),
  recipient_type TEXT NOT NULL,    -- community | ibama | federal_police
  channel TEXT NOT NULL,           -- whatsapp | sms
  recipient TEXT NOT NULL,
  status TEXT DEFAULT 'sent',
  sent_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE territory_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  territory TEXT NOT NULL,
  name TEXT NOT NULL,
  role TEXT NOT NULL,              -- community_leader | guard | ibama | federal_police
  phone TEXT,
  notify_on TEXT[] DEFAULT '{fire,chainsaw,human}',
  active BOOLEAN DEFAULT TRUE
);
```

---

## Detection thresholds

| Threat | Model | Threshold | Severity |
|---|---|---|---|
| Fire | MediaPipe Object Detector | > 0.80 | critical |
| Smoke | MediaPipe Object Detector | > 0.75 | high |
| Chainsaw | MediaPipe Audio Classifier | > 0.85 | critical |
| Gunshot | MediaPipe Audio Classifier | > 0.90 | critical |
| Human (after 6pm) | MediaPipe Object Detector | > 0.70 | high |
| Off-road vehicle | MediaPipe Object Detector | > 0.70 | medium |
| Fire + Chainsaw (same node) | combined score | — | critical + escalation |

---

## Node configuration (node-app/config.py)

```python
NODE_ID = "node-xingu-03"
GPS_LAT = -2.8451
GPS_LNG = -52.4142
TERRITORY = "Xingu Indigenous Land"

SPIDER_URL = "https://api.ecoeye.app/ingest"
API_KEY = "..."

VISION_CONFIDENCE_MIN = 0.80
AUDIO_CONFIDENCE_MIN = 0.85

NIGHT_MODE_START = "18:00"         # higher sensitivity after dark
CAPTURE_CLIP_SECONDS = 15          # clip buffer around detection
```

---

## Hackathon MVP

The goal is **understanding, not functionality**. Build just enough to make the idea undeniable to a room of community leaders.

### Must have
- [ ] `dashboard/WebMap.tsx` — map of the territory with nodes shown as a web
- [ ] `dashboard/MeshAnimation.tsx` — animated mini flag traveling from node to node to spider
- [ ] `spider/routes/ingest.js` — receives a simulated mini flag
- [ ] `spider/services/notifier.js` — fires a real WhatsApp message to one test number
- [ ] Live event appearing on the map when the spider receives the flag

### Nice to have
- [ ] Simulate multiple nodes with different threat types
- [ ] Show the flag hopping visually across 3-4 nodes before reaching spider
- [ ] Display the WhatsApp message on screen during the demo
- [ ] EcoProof integration slide

### Not needed
- Real Meshtastic hardware
- Real MediaPipe inference
- Multiple territories
- Auth

---

## Demo script (5 minutes, for community leaders)

1. **Open the map** — show the territory with 5-6 nodes spread across it, connected like a web
2. **"A guard at the edge of the territory hears something"** — trigger Node-01 detection (chainsaw)
3. **Animate:** the mini flag travels node to node across the web toward the spider
4. **Spider receives it** — red marker appears on the map, event in the feed
5. **WhatsApp fires on screen** — community leader gets the alert with GPS location
6. **"The same event becomes legal evidence"** — show EcoProof receiving the data
7. **Pitch:** "Any old donated smartphone becomes a guardian. The forest builds its own nervous system."

---

## EcoProof integration

```js
// spider/services/ecoproof_bridge.js
async function sendToEcoProof(event) {
  await fetch("https://api.ecoproof.ai/v1/evidence", {
    method: "POST",
    headers: { Authorization: `Bearer ${ECOPROOF_API_KEY}` },
    body: JSON.stringify({
      source: "ecoeye",
      type: event.threat_type,
      occurred_at: event.detected_at,
      location: { lat: event.lat, lng: event.lng },
      confidence: event.confidence,
      node_id: event.node_id,
      raw: event.raw_flag
    })
  });
}
```

---

*EcoEye is an open-source initiative for territorial protection.*
*Any donated smartphone becomes a node. The forest builds its own nervous system.*
