# Pizza-Along-Route — System Design Document

## 1) Overview: Goals & Context
**Project:** “Pizza-Along-Route” (working title) — a cyclist-centric tool that finds the **best pizza places** along a user-provided route.

**Primary user & interaction (MVP):** A **cyclist planning a ride** using a **CLI**. Inputs:
- GPX file for the route
- A kilometer range within that route to search (e.g., km 7–14)
- Maximum **crow-flies** distance off route (default ≤ 2 km; configurable)
- Optional flags to include **distance** (crow-flies) and **price** in scoring (rating is always included)

**Core outcome:** A **list** of candidate pizza places within a buffered **corridor** around the selected route segment, **sorted by an internal score** (rating/price/distance normalized 0–1), but **displayed without the score**. Each entry includes: Name, Rating, Price level, Crow-flies distance off route, and the Kilometer marker of the nearest route point. Results are **spatially spread** along the segment (light randomization within sub-segments) to avoid clustering near one end.

**Scope & constraints (MVP):**
- **Worldwide** coverage
- **Crow-flies** distance only for filtering & output (no bike-distance detour)
- Corridor built **client-side** (Shapely buffer); corridor queried via bounding-box approximation
- **Hybrid providers:** Google Places (ratings/price) as primary; OpenRouteService reserved for future routing
- Design is **stateless**, outputting structured data (JSON-like) to ease future migration to GUI / AWS containerized service

**Non-goals (MVP):** GUI, capacity estimation, bike-distance routing, persistent database (beyond optional local cache).

---

## 2) Functional & Non-Functional Requirements
### Functional
- Accept a **GPX file**; parse route and km markers.
- CLI params: `--range-start`, `--range-end`, `--offroute-distance` (km), `--use-distance`, `--use-price`.
- Build **corridor polygon** via Shapely buffer around selected km range; tile into bounding boxes.
- Query **Google Places** for pizza POIs in tiles; gather Name, Lat/Lon, Rating, Price level (if present).
- Compute **crow-flies** distance from each POI to the route; normalize metrics to [0–1].
- Internal scoring = mean of selected, available metrics (rating, optional price, optional distance); handle missing values by averaging only over present metrics.
- **Spatial spreading**: segment route and apply randomized selection of top items per segment.
- Output **CLI table** or **JSON** with fields: Name, Rating, Price, Distance (km, crow-flies), KM marker. (Internal score hidden.)

### Non-Functional
- **Performance:** ~10s target for a 15 km segment with modest POI density.
- **Scalability:** Stateless core; ready for containerization.
- **Reliability:** Retry/backoff on transient errors; optional local cache for dev.
- **Security:** `.env` for API keys; no secrets in logs.
- **Maintainability:** Modular: providers, geometry, scoring, output separated.
- **Extensibility:** Pluggable providers; future capacity & weights.
- **Compliance:** Respect Google/ORS ToS; attribution where required.
- **Usability:** Clear CLI help, readable output.

---

## 3) Architectural Drivers (Trade-offs, Risks, Assumptions)
**Trade-offs**
- Accuracy vs. performance: crow-flies now; routing later.
- Data source: Google Places (rich metadata) + ORS reserved.
- Corridor: client-side Shapely buffer (control, testability).
- Scoring: simple normalized mean (transparent, extensible).
- Diversity: lightweight randomization vs. heavy clustering.
- Execution: local CLI now; AWS container later.

**Risks & Mitigations**
- API quotas → RateLimiter, batching, caching in dev.
- Missing ratings/price → average over available metrics.
- Corridor tiling gaps/overlap → small overlaps, validation tests.
- User misconfig → robust CLI validation, helpful errors.
- Future migration complexity → strict module boundaries, stateless core.

**Assumptions**
- Valid GPX tracks; user comfortable with CLI.
- Google API key available; modest POI counts per query.
- Single-user MVP usage pattern.

---

## 4) High-Level Architecture (Components & Interactions)
**Components**
- **CLI Adapter** (args/validation/orchestration)
- **GPX Parser** (route points, cumulative distance, km markers)
- **Geometry Engine** (Shapely buffer; nearest km)
- **Corridor Tiler** (polygon → bounding boxes)
- **Places Provider (abstract)** → **GooglePlacesProvider** (primary)
- **RateLimiter & Batcher** (throttling, backoff)
- **Local Cache (dev)** (optional; ToS-compliant)
- **Normalization & Scoring** (0–1; mean over available metrics)
- **Spatial Spreader** (segmenting + randomized selection)
- **Kilometer Locator** (project POI to nearest route km)
- **Output Renderer** (table/JSON)

**Flow**
CLI → GPX Parser → Geometry Engine (buffer) → Corridor Tiler → RateLimiter → Places Provider → Scoring → Spatial Spreader → KM Locator → Output Renderer

---

## 5) Major Components (Responsibilities, Interfaces, Tech Choices)
- **CLI Adapter:** `argparse/click`; orchestrates; no business logic.
- **GPX Parser:** `gpxpy`, `geopy.distance`; exposes `Route` with `get_segment(start_km,end_km)`.
- **Geometry Engine:** `shapely` buffer; utilities for nearest point/km on route.
- **Corridor Tiler:** polygon→bbox list with overlap; configurable tile size (~2×2 km).
- **Places Provider (abstract):** `search_pizza(bbox) -> [Place]`; primary `GooglePlacesProvider`.
- **RateLimiter & Batcher:** token bucket/semaphore; `tenacity` for retries/backoff on 429.
- **Local Cache:** `sqlite3` or JSON; TTL; dev-only, ToS-aware.
- **Normalization & Scoring:** rating/5; price → `1-(price/4)`; distance → `1-(d/max_d)`; internal mean.
- **Spatial Spreader:** segment route (3–5 km bins); rank then randomized pick with score bias.
- **Kilometer Locator:** projection to route; return nearest km marker.
- **Output Renderer:** `tabulate`/`rich`; JSON serializer for GUI handoff.

---

## 6) Data Flow & Control Flow (Key Workflows)
1. **Invoke CLI** with GPX + params → config loaded from `.env`.
2. **Parse GPX** → compute cumulative distance; slice km range.
3. **Build corridor** via buffer(radius = off-route km); **tile** to bboxes.
4. **Query Places** per bbox with RateLimiter; collect POIs (name, loc, rating, price).
5. **Compute crow-flies distance** POI↔route; **normalize** metrics; **score** (internal).
6. **Spatial spreading**: segment, rank, randomized selection.
7. **KM mapping** for each selected POI.
8. **Render output** (table/JSON), excluding internal score.

---

## 7) Deployment & Infrastructure Overview
**MVP (Local)**
- Python 3.x; Poetry; `.env` for keys; optional `config.yaml`.
- No server-side infra; optional local cache; logs to `~/.pizza_along_route/`.
- CI: GitHub Actions — lint (black/ruff), test (pytest), build (poetry).

**Future (Reference)**
- Container on AWS ECS/EKS; S3 for GPX, Redis/DynamoDB cache, optional RDS.
- REST/GraphQL service; worker queue for heavy jobs; autoscaling.
- Monitoring: CloudWatch metrics and alerts.
*(Kept as compatibility target; future docs will cover details.)*

---

## 8) Security, Compliance & Governance
**Security**
- Secrets in `.env` (local), later AWS Secrets Manager.
- No secrets in logs; HTTPS/TLS in future web service; least-privilege IAM.

**Compliance**
- Google ToS: attribution, no persistent caching beyond limits, respect quotas.
- ORS: attribution; per-account quotas.
- No PII stored; GPX stays local (MVP).

**Governance**
- Structured logs; clear error reporting; transparent scoring (documented).

---

## 9) Evolution & Roadmap (Phase 1 — MVP Only)
**Goals:** Deliver accurate CLI tool to validate geometry + API approach.

**Scope:** CLI; GPX input; km range; off-route limit; Shapely corridor; bbox tiling; Google Places; crow-flies distance; normalized internal scoring (rating + optional price + distance); spatial spreading; table/JSON output; optional local cache; stateless core.

**Non-goals:** GUI, capacity estimates, bike-distance routing, persistent DB, hosted deployment.

---

## 10) Documentation & Decision Tracking
**Docs**
- `README.md`: install, API keys, usage, examples, limits, attribution.
- `docs/architecture.md`: components, data flow, scoring, corridor, batching, rate limits, caching.
- Inline docstrings + type hints.
- `CHANGELOG.md`: semantic versioning.
- Tests: unit + integration; provider mocks.

**ADRs**
- Crow-flies MVP; Shapely corridor; Hybrid providers; CLI-first; Randomized spreading.

---

## 11) Summary Table of Key Choices & Rationales
| Category | Decision | Rationale | Future Flexibility |
|---|---|---|---|
| Interface | CLI-based MVP | Fast iteration | Wrap as REST later |
| Distance | Crow-flies only | Minimize API calls/complexity | Add routing distance later |
| Corridor | Shapely buffer (client-side) | Control, transparency | API-side queries possible |
| Filtering | BBox tiling | Batch queries | Tune tile size/overlap |
| Providers | Google Places + ORS | Rich data + routing option | Pluggable interface |
| Rate Limit | Local limiter + backoff | Quota compliance | Distributed throttling later |
| Caching | Optional local | Dev efficiency | Redis/DynamoDB later |
| Scoring | Normalized mean | Transparent, simple | Add weights/capacity |
| Diversity | Randomized spread | Avoid clustering | Spatial clustering later |
| Deployment | Local Python (Poetry) | Minimal setup | Containerized AWS later |
| Security | `.env` keys | Prevent leaks | Secrets Manager later |
| Docs | README + ADRs | Traceable decisions | Expand web docs later |
