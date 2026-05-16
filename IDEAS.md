# IDEAS.md — OpenFoundry Use Cases & Exploration

> A living ideation document. Ideas range from immediately practical to long-term ambitious.
> Each section maps real-world problems to OpenFoundry capabilities.
> Last updated: 2026-05-16

---

## How to Read This Document

Each idea includes:
- **What**: the problem or opportunity
- **How OpenFoundry helps**: the specific platform capabilities that apply
- **Complexity**: `Low` / `Medium` / `High` — rough implementation effort
- **Value**: `Operational` / `Analytical` / `Strategic` — the primary payoff

---

## Section 1 — Work: Legionella Testing Company

> **Context**: A legionella risk management and testing business operating across a fragmented stack — a bespoke testing portal, HubSpot (CRM), Brightpearl (orders/accounting/stock), a third-party fulfilment warehouse (unreliable stock sync), Microsoft 365 (Teams, Outlook, SharePoint), and Asana for task management.

The core problem is **data fragmentation**: a customer, a site, an order, a test record, and a compliance document all live in different systems with no shared identity. OpenFoundry can be the connective tissue.

---

### 1.1 — Unified Customer & Site Ontology

**What**: Today a customer exists in HubSpot, their orders in Brightpearl, their test records in the testing portal, their documents in SharePoint, and their tasks in Asana — with no single record that links them.

**How OpenFoundry helps**:
- Define object types: `Customer`, `Site`, `Contract`, `TestRecord`, `Order`, `ComplianceDocument`, `RiskAssessment`
- Link types: `Customer → hasSite`, `Site → hasTestRecord`, `Order → belongsToSite`, etc.
- Use connectors to pull from HubSpot (REST API), Brightpearl (REST API), SharePoint (Graph API) into the ontology
- `ontology-query-service` powers graph traversal: "show me all overdue sites for this customer with open orders"

**Complexity**: Medium | **Value**: Strategic

---

### 1.2 — Reliable Stock Sync Between Brightpearl & Warehouse

**What**: The third-party warehouse manages physical stock and attempts to sync levels with Brightpearl — but the sync is unreliable, leading to overselling or phantom stock.

**How OpenFoundry helps**:
- `connector-management-service` polls both Brightpearl and warehouse APIs on a schedule
- `dataset-versioning-service` maintains a versioned, timestamped view of stock levels from both sources
- A reconciliation pipeline (DataFusion) computes deltas and flags discrepancies
- `workflow-automation-service` triggers correction actions (e.g. raise a Brightpearl stock adjustment) when drift exceeds a threshold
- Full lineage tracks every stock change back to its source event

**Complexity**: Medium | **Value**: Operational

---

### 1.3 — Compliance & Audit Trail (L8 ACOP / HSG274)

**What**: Legionella testing is heavily regulated (L8 Approved Code of Practice, HSG274 guidance). There is a legal obligation to demonstrate that testing was carried out on schedule, by a competent person, with results recorded and acted upon.

**How OpenFoundry helps**:
- `audit-compliance-service` stores an immutable, timestamped record of every test submission, every result, and every remedial action
- Cedar policies (`authorization-policy-service`) enforce who can submit, modify, or delete records — with every decision logged
- `lineage-service` traces a test result from raw submission → processed record → compliance report → customer-facing certificate
- Generate compliance reports on-demand via a Workshop app: "show me all sites that have not been tested in the last 12 months"
- `notification-alerting-service` proactively alerts when a site is approaching its next due date

**Complexity**: Medium | **Value**: Operational + Strategic

---

### 1.4 — Cross-System Order-to-Service Workflow

**What**: The process from a customer placing an order → kits dispatched → test carried out → results submitted → report delivered → invoice raised spans at least four systems with manual handoffs at each step.

**How OpenFoundry helps**:
- `workflow-automation-service` models the entire lifecycle as a workflow with stages: `Order Confirmed → Dispatch Approved → Kit Sent → Test Due → Result Received → Report Generated → Invoice Raised`
- Saga pattern handles failures: if dispatch fails, compensate; if result is overdue, escalate
- Each step is audited, observable, and retryable
- Actions on the `Order` object type (in the ontology) trigger downstream workflow steps
- Human-in-the-loop approvals for certificate sign-off

**Complexity**: High | **Value**: Operational + Strategic

---

### 1.5 — HubSpot CRM Enrichment via Ontology

**What**: HubSpot holds contact and deal data but has no concept of site compliance status, test frequency, or risk level — information sales teams need when talking to customers.

**How OpenFoundry helps**:
- Ingest HubSpot contacts and deals via connector; link to `Site` and `TestRecord` objects in the ontology
- Derive computed properties: `RiskLevel`, `ComplianceScore`, `DaysUntilNextTest`, `OpenRemediationCount`
- Push enriched properties back to HubSpot via a scheduled action (bidirectional sync)
- Sales and account managers get a Workshop app showing everything about a customer without leaving context

**Complexity**: Medium | **Value**: Operational + Strategic

---

### 1.6 — Microsoft 365 Integration (Teams / Outlook / SharePoint)

**What**: Notifications and alerts currently exist only in the testing portal, invisible to people working in Teams and Outlook all day.

**How OpenFoundry helps**:
- `notification-alerting-service` with a Teams webhook delivery channel: push overdue test alerts, compliance breaches, stock issues directly into Teams channels or DMs
- SharePoint connector ingests compliance documents, certificates, risk assessments into the ontology — searchable via Vespa
- Outlook (Graph API) connector links emails to customer/site objects (contract renewals, correspondence history)
- A Teams-facing summary card for any site can be generated on demand via an action

**Complexity**: Low–Medium | **Value**: Operational

---

### 1.7 — Asana Task Tracking Linked to Ontology Objects

**What**: Asana tasks (remedial actions, follow-ups, report reviews) exist in isolation — not linked to the site or test record they refer to.

**How OpenFoundry helps**:
- Asana API connector syncs tasks into `Task` objects linked to `Site`, `TestRecord`, and `Order`
- Overdue remedial actions surfaced in compliance reports alongside the test results that triggered them
- Workflow automations can create Asana tasks as a side effect of ontology actions (e.g. "result flagged → create remedial action task in Asana")

**Complexity**: Low | **Value**: Operational

---

### 1.8 — AI-Assisted Risk Flagging

**What**: Experienced consultants spot patterns in test results that indicate elevated risk — e.g. repeated low-level detections at a particular point, seasonal spikes, post-shutdown anomalies. This knowledge is not systematised.

**How OpenFoundry helps**:
- Historical test results ingested into Iceberg via `dataset-versioning-service`
- ML pipeline (`pipeline-runner`, `model-catalog-service`) trains a risk classification model on labelled historical data
- `model-deployment-service` serves predictions: each new test result is scored for risk at submission time
- High-risk scores trigger automated alerts via `notification-alerting-service`
- `retrieval-context-service` + `agent-runtime-service` enables a compliance assistant: "summarise risk history for this site and recommend next steps"

**Complexity**: High | **Value**: Strategic

---

## Section 2 — Home Networking

> **Context**: A growing home network with managed and unmanaged switches, SBCs running Pi-hole and AdGuard Home, a mini-PC running pfSense, home servers on OMV7, multiple endpoints (laptops, desktops), Wi-Fi access points, and a stock EE router. Goal: unified visibility and management.

---

### 2.1 — Network Topology Ontology

**What**: Know exactly what is connected where — switches, ports, VLANs, subnets, devices — without having to remember or draw it.

**How OpenFoundry helps**:
- Object types: `NetworkDevice`, `Switch`, `Port`, `VLAN`, `Subnet`, `AccessPoint`, `Endpoint`, `SBC`
- Link types: `Device → connectedTo → Port`, `Port → onSwitch`, `Switch → inVLAN`, `Endpoint → assignedIP`
- Seed manually; keep current via SNMP polling connector (for managed switches) and ARP/DHCP lease connectors (for pfSense)
- Workshop app renders the full topology graph (Cytoscape) — click any node to see all connected devices, link speed, current traffic

**Complexity**: Medium | **Value**: Operational

---

### 2.2 — DNS Analytics (Pi-hole / AdGuard Home)

**What**: Pi-hole and AdGuard Home log every DNS query but offer only basic built-in analytics. Richer analysis (trends, per-device breakdowns, block rate over time, top blocked domains) requires something more capable.

**How OpenFoundry helps**:
- Connector (REST API or log tail) ingests query logs from both Pi-hole and AdGuard Home
- `dataset-versioning-service` stores versioned daily snapshots; `pipeline-runner` aggregates into Iceberg
- Analytical dashboards in Workshop: top queried domains, top blocked categories, per-device query volume, anomaly detection (device suddenly querying 10x normal volume)
- Alert if block rate drops dramatically (could indicate a DNS resolver bypass)

**Complexity**: Low–Medium | **Value**: Analytical

---

### 2.3 — pfSense Firewall Metrics & Security Alerting

**What**: pfSense generates rich firewall state — blocked IPs, interface traffic, VPN tunnels, DHCP leases — but this is hard to correlate with the wider network picture.

**How OpenFoundry helps**:
- pfSense REST API / syslog connector ingests firewall state into OpenFoundry
- `notification-alerting-service` alerts on: new unknown MAC address, blocked traffic spike from an internal host, WAN IP change, VPN tunnel down
- Traffic volume pipelines feed Iceberg for bandwidth trend analysis
- DHCP lease events linked to `Endpoint` objects in the network ontology (automatic inventory population)

**Complexity**: Medium | **Value**: Operational

---

### 2.4 — Device & Firmware Inventory

**What**: Keeping track of firmware versions, last-updated dates, and open CVEs across SBCs, switches, and APs is tedious and done ad-hoc.

**How OpenFoundry helps**:
- Object type `NetworkDevice` with properties: `firmware_version`, `last_checked`, `vendor`, `model`, `end_of_life_date`
- A scheduled pipeline queries vendor APIs or NVD (National Vulnerability Database) for CVEs matching device firmware versions
- `notification-alerting-service` flags devices with known CVEs or firmware more than N versions behind
- Workshop app: firmware currency dashboard

**Complexity**: Medium | **Value**: Operational

---

### 2.5 — Network Change Management Workflow

**What**: Making changes to switch configs, VLAN assignments, or firewall rules carries risk. Currently there's no structured change record.

**How OpenFoundry helps**:
- `workflow-automation-service` models a lightweight change approval: `Proposed → Reviewed → Applied → Verified`
- Every change action on a `Switch` or `FirewallRule` object is audited
- Post-change verification: connector re-polls device state after N minutes; workflow auto-closes if state matches intent
- Rollback action defined as a saga compensation step

**Complexity**: Medium | **Value**: Operational

---

### 2.6 — Unified Network Monitoring Dashboard

**What**: Currently monitoring is fragmented — some in pfSense dashboard, some in OMV7 web UI, some in the Pi-hole UI. No single pane.

**How OpenFoundry helps**:
- Workshop app aggregates: interface up/down status, WAN latency, DNS query rate, top bandwidth consumers, VPN health, disk usage on OMV7 nodes
- All data ingested via respective connectors into the ontology / Iceberg
- Alerting via Teams (or any webhook) when any node goes offline

**Complexity**: Low–Medium | **Value**: Operational

---

## Section 3 — Music Collection

> **Context**: A large physical and digital music collection. Physical records catalogued on Discogs (partially), digital libraries across multiple locations, Navidrome as a self-hosted streaming server, FiiO R7 as DAC and media streamer.

---

### 3.1 — Master Music Ontology

**What**: A single canonical view of the entire collection — physical and digital — with no duplicates, consistent metadata, and cross-referenced across sources.

**How OpenFoundry helps**:
- Object types: `Artist`, `Album`, `Track`, `Release` (edition/pressing), `Format` (Vinyl/CD/FLAC/MP3/etc.), `Label`, `Genre`
- Link types: `Release → byArtist`, `Release → hasTrack`, `Album → hasRelease`, `Artist → onLabel`
- Connectors: Discogs API (physical collection + want list), Navidrome API (digital library), local filesystem scanner (unindexed files)
- `entity-resolution-service` deduplicates: same album appearing in Discogs, Navidrome, and a local folder becomes one canonical `Album` object with three linked `Release` objects

**Complexity**: Medium | **Value**: Analytical + Strategic

---

### 3.2 — Collection Gap Analysis & Want List Intelligence

**What**: The Discogs want list is a flat list with no intelligence — no price history context, no "this pressing vs. that pressing" analysis, no alerting.

**How OpenFoundry helps**:
- Discogs API connector syncs want list and marketplace listings into the ontology
- Pipeline computes per-release: median price, price trend (last 90 days), number of copies available, median grade available
- Alert when a wanted item drops below a target price or when a near-mint copy appears
- Workshop app: want list ranked by value opportunity, price history charts

**Complexity**: Medium | **Value**: Analytical

---

### 3.3 — Listening History Analytics (Navidrome)

**What**: Navidrome keeps play counts and last-played timestamps but offers no deeper trend analysis — no listening streaks, genre breakdowns over time, "most played by month", etc.

**How OpenFoundry helps**:
- Navidrome API connector ingests scrobble/play events into Iceberg via a dataset or sink
- Pipelines compute: top artists by month, genre listening distribution, time-of-day patterns, listening streaks
- Workshop app: personal music analytics dashboard — think Last.fm wrapped, self-hosted

**Complexity**: Low–Medium | **Value**: Analytical

---

### 3.4 — Collection Valuation

**What**: Knowing the current market value of the physical collection requires manually checking Discogs for every record.

**How OpenFoundry helps**:
- Scheduled pipeline pulls Discogs marketplace data for every owned release
- Computes: current median price, last-sale price, estimated total collection value
- Trend dashboard: collection value over time, biggest movers (up and down), most valuable items
- Alert if a specific record spikes in value (artist death, anniversary reissue announcement effect)

**Complexity**: Low–Medium | **Value**: Analytical

---

### 3.5 — Unlogged Physical Records Discovery

**What**: Some physical records have never been logged on Discogs. Finding which ones requires going through shelves manually.

**How OpenFoundry helps**:
- Workshop app with barcode-scan input: scan a record's barcode → query Discogs API → pre-fill a new `Release` object → one-click add to collection
- AI-assisted: `agent-runtime-service` can suggest the correct Discogs release from partial metadata (label number, pressing year, catalogue number)
- Flag records in the ontology as `logged: false` until confirmed

**Complexity**: Low | **Value**: Operational

---

### 3.6 — AI Music Discovery & Recommendation

**What**: With a large and eclectic collection, finding what to play next — or what to buy next — that fits the current mood is an unsolved problem.

**How OpenFoundry helps**:
- `retrieval-context-service` builds vector embeddings from track/album metadata, genre tags, listening history, and Discogs community reviews
- `agent-runtime-service` powers a conversational discovery interface: "play me something in the vein of this album but more electronic" → searches vector store → returns ranked recommendations from the owned collection or Discogs marketplace
- Mood-tagging pipeline using `ai-kernel-go` to classify albums by energy, tempo, instrumentation from metadata and user feedback

**Complexity**: High | **Value**: Strategic

---

### 3.7 — FiiO R7 Playback Integration

**What**: The FiiO R7 is the primary listening endpoint but exists outside any data ecosystem — playback events are invisible.

**How OpenFoundry helps**:
- FiiO R7 runs Android; if it exposes any API or DLNA/UPnP events, a connector captures playback events
- Alternatively: Navidrome → FiiO playback events scrobbled back → Navidrome API
- All playback events feed the listening history pipeline (Section 3.3)
- Alert when the R7 goes offline / stops responding on the network

**Complexity**: Low (if API available) – Medium | **Value**: Analytical

---

## Section 4 — Self-Hosting Infrastructure

> **Context**: Multiple machines running a mix of Docker and native services, spread across home servers and SBCs. Philosophy: self-host everything possible, minimise dependence on cloud/big-tech platforms.

---

### 4.1 — Service Inventory Ontology

**What**: As the number of self-hosted services grows, keeping track of what's running where — and what depends on what — becomes its own job.

**How OpenFoundry helps**:
- Object types: `Machine`, `Service`, `Container`, `Volume`, `Port`, `Domain`, `Certificate`, `DNSRecord`
- Link types: `Service → runsOn → Machine`, `Service → dependsOn → Service`, `Service → exposesPort`, `Domain → resolvedBy → DNSRecord`
- Seed from Docker API (list containers per host), OMV7 API, Ansible inventory, or manual entry
- Full dependency graph via Cytoscape in Workshop: "if this machine goes down, what services are affected?"

**Complexity**: Medium | **Value**: Operational + Strategic

---

### 4.2 — Unified Service Health Dashboard

**What**: Each service has its own health page or dashboard. There is no single view showing the health of the entire self-hosted stack.

**How OpenFoundry helps**:
- `connector-management-service` polls `/healthz`, `/ping`, or equivalent endpoints for every registered service
- Health status modelled as a property on the `Service` object; status changes written to Iceberg for trend analysis
- Workshop app: traffic-light health dashboard; click any red service to see last N health check results and logs
- `notification-alerting-service`: alert via Teams or webhook when any service goes unhealthy for > N minutes

**Complexity**: Low | **Value**: Operational

---

### 4.3 — Docker Container Lifecycle Management

**What**: Container restarts, image updates, and resource usage are monitored per-host but not aggregated across machines.

**How OpenFoundry helps**:
- Docker API connector on each host streams container events (start, stop, restart, OOM kill) into the ontology
- Object type `ContainerEvent` linked to `Container` and `Machine`
- Pipeline aggregates: restart frequency per container (flag flapping services), OOM events, long-running containers with stale images
- Alert when a container has restarted more than N times in 24 hours

**Complexity**: Low–Medium | **Value**: Operational

---

### 4.4 — SSL Certificate Expiry Tracking

**What**: Self-hosted services often run with certificates managed by Certbot, Caddy, or manually — and expiry is easy to miss.

**How OpenFoundry helps**:
- Scheduled pipeline probes TLS endpoints (using Go's `crypto/tls` or a small connector) and records expiry date
- `Certificate` object type with property `expires_at`; alert when expiry is within 14 days
- Linked to the `Domain` and `Service` objects: see exactly which service is affected

**Complexity**: Low | **Value**: Operational

---

### 4.5 — Backup Verification Workflow

**What**: Backup jobs run on schedules but there is no systematic check that backups actually completed, are accessible, and are within recovery point objectives.

**How OpenFoundry helps**:
- `workflow-automation-service` models a weekly backup verification: `Backup Job Run → Snapshot Created → Integrity Check → RPO Verified`
- Connector reads backup job logs from OMV7 or Restic/rclone output
- `audit-compliance-service` maintains an immutable record: "on this date, backups were verified for these services"
- Alert if any backup job has not produced a new snapshot within the expected window

**Complexity**: Medium | **Value**: Operational

---

### 4.6 — Resource Usage Analytics

**What**: Understanding which machines and services are resource-constrained — and trending toward problems — requires aggregated metrics over time.

**How OpenFoundry helps**:
- Node Exporter / Prometheus connector ingests CPU, RAM, disk, network metrics from each machine
- Pipeline stores time-series data in Iceberg; DataFusion runs aggregations
- Workshop dashboard: resource usage per machine, per service, trending over last 7/30/90 days
- Predictive alert: disk filling faster than expected → flag before it hits capacity

**Complexity**: Medium | **Value**: Analytical

---

### 4.7 — Self-Hosted Service Discovery & Documentation

**What**: Documentation for self-hosted services tends to be ad-hoc — a mix of notes, GitHub READMEs, and memory.

**How OpenFoundry helps**:
- `Service` object type includes: `description`, `config_location`, `data_volume_path`, `backup_schedule`, `setup_notes`, `links_to_docs`
- Workshop app renders a self-hosted service catalogue: searchable, linked, filterable by machine or status
- `code-repository-review-service` can link services to their config repos / compose files
- Vespa-powered full-text search across all service documentation and notes

**Complexity**: Low | **Value**: Operational + Strategic

---

## Section 5 — Personal Finance & Budgeting

> **Context**: Connecting personal finance data for visibility, trend analysis, and planning — without exposing data to third-party SaaS tools.

---

### 5.1 — Unified Transaction Ontology

**What**: Bank transactions, credit card spend, and investment accounts live in separate apps with no unified view.

**How OpenFoundry helps**:
- Open Banking API connectors (UK: FCA-regulated providers) ingest transactions
- Object types: `Account`, `Transaction`, `Merchant`, `Category`, `Budget`
- AI-assisted categorisation: `agent-runtime-service` classifies uncategorised transactions using merchant name + amount + pattern history
- Workshop app: spending breakdown, category trends, budget vs. actual

**Complexity**: Medium | **Value**: Analytical

---

### 5.2 — Subscription & Recurring Cost Tracking

**What**: Subscriptions accumulate invisibly. Knowing total monthly committed spend requires manual audit.

**How OpenFoundry helps**:
- Pipeline identifies recurring transaction patterns → creates `Subscription` objects with `monthly_cost`, `next_renewal`, `vendor`
- Alert when a subscription renews or when a new recurring charge appears (unexpected subscription)
- Dashboard: total committed monthly spend, subscriptions due for renewal this month

**Complexity**: Low | **Value**: Operational

---

## Section 6 — Home Automation & IoT

> **Context**: Integrating smart home devices, sensors, and automations into OpenFoundry for richer analytics and cross-system intelligence.

---

### 6.1 — Home Assistant Integration

**What**: Home Assistant is the hub for devices and automations but operates in isolation from all other personal data.

**How OpenFoundry helps**:
- Home Assistant REST API / webhook connector ingests state change events
- Object types: `Room`, `Device`, `Sensor`, `Automation`, `SensorReading`
- Correlate home state with other data: high energy consumption periods, heating patterns vs. occupancy vs. weather
- Alert on anomalies: motion sensor triggered at unusual time, temperature sensor outside expected range

**Complexity**: Medium | **Value**: Analytical

---

### 6.2 — Energy Monitoring & Optimisation

**What**: Understanding where home energy is being used — and identifying waste — requires more than a smart meter app provides.

**How OpenFoundry helps**:
- Smart meter (SMETS2) data via DCC API connector; device-level data via smart plugs
- Pipeline builds hourly/daily/monthly energy profiles per circuit/device
- AI analysis: identify devices with anomalous consumption; recommend optimal schedules (e.g. run dishwasher at off-peak tariff)
- Track against energy tariff to calculate actual spend per device

**Complexity**: Medium–High | **Value**: Analytical + Operational

---

## Section 7 — Media & Entertainment Library

> **Context**: Beyond music — books, films, TV, and games — all tracked and managed without relying on external services.

---

### 7.1 — Unified Personal Media Ontology

**What**: Books on Goodreads (or not), films in Letterboxd (or not), games on RAWG or nowhere — scattered, incomplete, un-linked.

**How OpenFoundry helps**:
- Object types: `Book`, `Film`, `TVShow`, `Game`, `Creator`, `Studio`, `Platform`
- Connectors: Goodreads export, Letterboxd export, RAWG API, IGDB API, Open Library API
- Link to consumption records: `ReadingRecord`, `WatchRecord`, `PlaySession`
- Ratings and reviews stored as properties; recommendations via vector search

**Complexity**: Medium | **Value**: Analytical

---

### 7.2 — Jellyfin / Plex Library Integration

**What**: A self-hosted media server contains a film and TV library but play history, ratings, and watch status live only in the server.

**How OpenFoundry helps**:
- Jellyfin/Plex API connector ingests media library metadata and playback events
- Linked to `Film` and `TVShow` objects in the media ontology
- Analytics: watch frequency, genres by season, backlog size, completion rate for TV series
- Recommendation: "based on your watch history, here are films in your library you haven't watched yet that match your taste"

**Complexity**: Low–Medium | **Value**: Analytical

---

## Section 8 — Open Data & Research

> **Context**: Using OpenFoundry's data platform capabilities to work with public datasets — government data, ONS statistics, academic sources.

---

### 8.1 — Local Area Analytics

**What**: UK Open Data (data.gov.uk) contains planning applications, transport data, crime statistics, environmental data — none of it easy to query together.

**How OpenFoundry helps**:
- Connectors for data.gov.uk, ONS API, Environment Agency, Transport for London
- Datasets ingested into Iceberg; pipelines join and clean across sources
- Ontology models: `LocalAuthority`, `PlanningApplication`, `CrimeRecord`, `TransportLink`
- Workshop app: local area dashboard for any postcode — planning activity, crime trends, transport disruptions, air quality

**Complexity**: High | **Value**: Analytical + Strategic

---

### 8.2 — Platform as a Research Sandbox

**What**: OpenFoundry's pipeline and notebook infrastructure can serve as a local, privacy-respecting alternative to cloud-based data science environments.

**How OpenFoundry helps**:
- `notebook-runtime-service` provides Jupyter-compatible kernels backed by local compute
- `pipeline-expression` + DataFusion enables SQL-based data wrangling without leaving the platform
- `model-catalog-service` tracks experiments; `model-deployment-service` serves local models
- All work is version-controlled, lineage-tracked, and auditable — without any data leaving the self-hosted environment

**Complexity**: Medium | **Value**: Strategic

---

## Priority Matrix

| Idea | Section | Complexity | Value | Suggested Order |
|------|---------|-----------|-------|-----------------|
| Unified Customer & Site Ontology | 1.1 | Medium | Strategic | 1 |
| Reliable Stock Sync | 1.2 | Medium | Operational | 2 |
| Service Inventory Ontology | 4.1 | Medium | Strategic | 3 |
| Network Topology Ontology | 2.1 | Medium | Operational | 4 |
| Unified Service Health Dashboard | 4.2 | Low | Operational | 5 |
| Compliance & Audit Trail | 1.3 | Medium | Operational | 6 |
| DNS Analytics | 2.2 | Low–Medium | Analytical | 7 |
| Master Music Ontology | 3.1 | Medium | Strategic | 8 |
| Listening History Analytics | 3.3 | Low–Medium | Analytical | 9 |
| SSL Certificate Tracking | 4.4 | Low | Operational | 10 |
| Collection Valuation | 3.4 | Low–Medium | Analytical | 11 |
| AI Risk Flagging (Legionella) | 1.8 | High | Strategic | Later |
| AI Music Discovery | 3.6 | High | Strategic | Later |
| Local Area Analytics | 8.1 | High | Analytical | Later |

---

## Ideas Backlog (Quick Captures)

Smaller ideas that don't yet have full write-ups:

- **Personal CRM**: Track relationships, interactions, and follow-ups with people outside of work
- **Recipe & Meal Planning Ontology**: Ingredients, recipes, meal plans, shopping list generation
- **Document Management**: Replace SharePoint/Google Drive for personal document storage with full-text search (Vespa) and lineage
- **Photography Library**: Lightroom catalogue export + EXIF metadata → ontology; geospatial plot of photo locations
- **Vehicle Maintenance Log**: Service records, MOT dates, mileage tracking, parts replaced
- **Learning Tracker**: Books read, courses taken, skills acquired — linked to projects that used them
- **Weather & Microclimate**: Personal weather station data → Iceberg → seasonal pattern analysis
- **HAM Radio / SDR Logs**: If applicable — signal logs, contacts, frequency analytics
- **3D Printing Job Tracker**: Print jobs, filament usage, success/failure rates, model sources

---

*Add new ideas as they emerge. Mark implemented ideas with a checkmark and link to the relevant service/lib.*
