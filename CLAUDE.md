# Elderly Monitor — Claude Code Guide

## Project Overview
A Next.js hackathon app for monitoring elderly people at home using wireless accelerometer "bricks" placed in each room. Two interfaces: one for the elder, one for the family caregiver. Connected to Claude AI for health pattern analysis.

## Tech Stack
- **Framework:** Next.js 14.2.5 (App Router) + TypeScript
- **Database:** SQLite via Node.js built-in `node:sqlite` (no native compilation needed — requires Node 22.5+)
- **Styling:** Tailwind CSS 3.4
- **Charts:** Recharts
- **Icons:** lucide-react
- **AI:** `@anthropic-ai/sdk` — Claude Opus 4.6 with adaptive thinking + streaming

## Running the App
```bash
npm run dev   # starts on http://localhost:3000
```
Kill stale node processes if ports are occupied: `taskkill /F /IM node.exe`

## Key Files
| Path | Purpose |
|------|---------|
| `src/lib/db.ts` | SQLite schema + types (`Brick`, `SensorEvent`, `Anomaly`, etc.) |
| `src/lib/anomaly.ts` | Rule-based anomaly detection (accelerometer patterns + medication) |
| `src/app/elderly/page.tsx` | Elder-facing UI (large text, I'm OK button, medication list) |
| `src/app/family/page.tsx` | Family dashboard (tabs: Overview, Activity, Bricks, Medications, Anomalies) |
| `src/app/api/bricks/route.ts` | CRUD for named accelerometer bricks |
| `src/app/api/sensor/route.ts` | Record accelerometer readings (`brickName`, `magnitude`) |
| `src/app/api/medication/route.ts` | Medication schedule + logs |
| `src/app/api/anomalies/route.ts` | Query + dismiss anomalies |
| `src/app/api/ai-analysis/route.ts` | Streaming Claude AI analysis (summary, anomaly explain, pattern) |
| `src/app/api/seed/route.ts` | Seeds 48h demo data with 5 bricks + realistic accelerometer patterns |
| `src/components/family/BrickManager.tsx` | Add/delete bricks, show live status |
| `src/components/family/AnomalyList.tsx` | Anomaly cards with streaming AI Explain button |
| `src/components/ComplianceBanner.tsx` | GDPR + AI Act consent banner (localStorage) |

## Data Model

### Bricks (`bricks` table)
Named accelerometer devices placed in rooms. Each has a `name` (e.g. `bedroom`, `kitchen`) used as the location identifier for sensor events.

### Sensor Events (`sensor_events` table)
All events are `type = 'accelerometer'`. The `location` field holds the brick name. The `value` field holds magnitude as a string (e.g. `"3.50"`), scale 0–10:
- 0–1: nearly still
- 2–4: normal light movement
- 5–7: active movement
- 8–10: vigorous / potential fall

### Anomaly Detection Rules (in `anomaly.ts`)
1. No activity from any brick for 6+ hours during daytime (8am–10pm) → **High**
2. No kitchen brick activity during meal windows (breakfast/lunch/dinner) → **Medium**
3. High-magnitude activity (>4.0) at night (midnight–5am), 5+ readings in 90 min → **Medium**
4. Medication not confirmed within 2h of scheduled time → **High**

## API Contracts

### POST /api/sensor
```json
{ "brickName": "kitchen", "magnitude": 3.5 }
```

### POST /api/bricks
```json
{ "name": "bedroom" }
```
Names are lowercased and spaces replaced with underscores.

### POST /api/ai-analysis
```json
{ "type": "summary", "events": [...], "anomalies": [...], "medications": [...], "bricks": [...] }
{ "type": "explain-anomaly", "anomaly": {...}, "events": [...], "bricks": [...] }
{ "type": "pattern-analysis", "events": [...], "bricks": [...], "days": 3 }
```
Returns a streaming `text/plain` response.

## Environment
```
ANTHROPIC_API_KEY=sk-ant-...   # in .env.local
```

## Compliance
- **GDPR:** Legal basis is vital interests (Art. 6(1)(d)). Health data under Art. 9. 30-day retention for sensor data, 12 months for medication logs.
- **EU AI Act:** Limited Risk classification (Art. 6). AI outputs are advisory only — never medical diagnoses. Human review required before acting.
- No audio, video, or biometric data collected. Only movement magnitude.
- Consent banner on first visit stored in `localStorage` key `gdpr_consent_v1`.

## Demo Flow
1. Open `/family` → Demo Controls → **Seed Demo Data**
2. Creates 5 bricks (bedroom, kitchen, living_room, bathroom, hallway) + 48h of data
3. Built-in anomalies: 6h inactivity gap on day 2, missed morning medication on day 1, 3am high-activity event
4. AI health summary streams in automatically on page load
5. Click **AI Explain** on any anomaly for a live streaming explanation
6. **Bricks tab** shows per-brick activity, add/remove bricks
