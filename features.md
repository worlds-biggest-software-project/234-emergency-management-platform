# Emergency Management Platform — Feature & Functionality Survey

> Candidate #234 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Everbridge 360 | Critical Event Management (CEM) SaaS | Commercial / Enterprise | https://www.everbridge.com |
| WebEOC (Juvare) | Emergency Operations Center software | Commercial / Contact for quote | https://www.juvare.com |
| Veoci | No-code emergency management platform | Commercial / Contact for quote | https://veoci.com |
| D4H Incident Management | ICS-native incident management SaaS | Commercial / Per-user subscription | https://www.d4h.com |
| Noggin | Integrated resilience workspace | Commercial / Contact for quote | https://www.noggin.io |
| DisasterLAN (DLAN) | EOC and situational awareness platform | Commercial / Contact for quote | https://www.buffalocomputergraphics.com |
| AlertMedia | Emergency mass notification SaaS | Commercial / Per-user / contact | https://www.alertmedia.com |
| OnSolve (CodeRED) | Multi-channel mass notification platform | Commercial / Contact for quote | https://www.onsolve.com |
| Sahana EDEN | Open-source disaster management platform | Open Source / MIT | https://sahanafoundation.org/eden/ |

---

## Feature Analysis by Solution

### Everbridge 360

**Core features**
- Unified critical event management lifecycle: risk intelligence, communication, incident command, recovery
- Mass notification across email, SMS, voice, mobile app, Slack, and Teams
- Visual Command Center (VCC) — real-time geospatial common operating picture
- 350+ ecosystem integrations including HR systems (Workday, UKG Pro, Microsoft Entra ID), IT monitoring, and BCP tools
- CEM Orchestration: automated workflow triggers from risk events to communications
- Physical security integration and IoT sensor feeds
- Advanced reporting and analytics dashboards
- Mobile app with Crisis Management access for field commanders

**Differentiating features**
- Everbridge High Velocity CEM: purpose-built for speed of detection through resolution
- AI-assisted risk intelligence with automated threat correlation across global data feeds
- Broadest ecosystem integration library in the market (350+)
- Public-sector and federal government variants with IPAWS-compliant alerting

**UX patterns**
- Dashboard-centric with role-based views for executives, field operators, and IT staff
- Progressive disclosure of incident details; high-level status cards expand into full incident command tools
- Mobile-first incident activation and status updates via app
- Automated guided playbooks reduce cognitive load during high-stress events

**Integration points**
- REST API with OAuth 2.0 authentication; developer hub at developers.everbridge.net
- Swagger-documented API at api.everbridge.net
- Pre-built connectors for Slack, Microsoft Teams, ServiceNow, Workday, Salesforce, SIEM tools
- IPAWS and CAP-protocol alert origination

**Known gaps**
- High cost and complex licensing is a recurring complaint from mid-market and municipal government buyers
- ICS documentation (ICS-204 forms, IAPs) less mature than purpose-built EOC tools like WebEOC
- Steep initial configuration and onboarding effort

**Licence / IP notes**
- Fully commercial; publicly traded (formerly EVBG, now private after 2023 acquisition by Thoma Bravo)
- No open-source components exposed; API documented but proprietary

---

### WebEOC (Juvare)

**Core features**
- Board-based information management: customisable status boards for each ICS section
- Requests/Tasks board for resource requests and task assignments (field-accessible via mobile)
- Significant Events board tracking "who did what when" — real-time chronology
- File Library for document sharing and private folder management across positions
- ICS and ESF position-based role configuration
- WebEOC Nexus: customisable homepages preset for ICS roles (IC, Ops, Logistics, etc.)
- Integration with NWS weather alerts via activity log feed

**Differentiating features**
- Deep ICS/NIMS compliance baked into board architecture — designed for US EOC workflows
- WebEOC Nexus redesign supports rapid incident activation with pre-configured dashboards per ICS position
- Long track record in US county and state emergency management agencies
- EICS (Electronic Incident Command System) module for full ICS documentation

**UX patterns**
- Legacy board-based paradigm: each function has a dedicated board; no unified dashboard
- WebEOC Nexus introduces a modernised card-based home view with position-based presets
- Mobile access for field resource requests
- Significant configuration required before use; strong admin community (WebEOC Administrators Group)

**Integration points**
- REST API available via docs.juvare.com/webeoc-onnexa/rest-api/
- Session-based authentication with position and incident context
- Supports NWS weather alert feeds in the activity log
- Juvare Connect provides mutual-aid resource sharing across jurisdictions

**Known gaps**
- Dated UI despite WebEOC Nexus improvements; significant training required
- Mass notification is not native — requires integration with Everbridge or similar
- Limited third-party integration ecosystem compared to Everbridge
- Heavy configuration burden before first deployment

**Licence / IP notes**
- Commercial SaaS; Juvare is PE-backed (Francisco Partners)
- No open-source components

---

### Veoci

**Core features**
- No-code workflow builder for digitising emergency operations plans (EOPs) and incident action plans (IAPs)
- Real-time GIS mapping with automatic situational awareness feeds
- Resource requesting, tracking, and task assignment with built-in documentation
- Automated After-Action Reports (AARs) and FEMA form submissions
- Compliance tracking and audit trails
- Notifications and alerts with automatic reminders to task owners
- Integration with external data sources via API

**Differentiating features**
- No-code configuration means emergency managers — not IT staff — can build and adapt workflows
- Purpose-built AAR and FEMA form automation reduces post-incident reporting burden
- Strong government and utilities vertical focus
- Automatic feed of incoming data (weather, CAD, etc.) into EOC dashboards

**UX patterns**
- Guided workflow activation: click-through plan execution rather than free-form boards
- GIS map always visible alongside task lists; real-time colour-coded status indicators
- Mobile-accessible for field staff
- Significant initial plan-building effort required before benefit is realised

**Integration points**
- REST API for external system integration (documentation requires vendor contact)
- Connects to CAD systems, weather feeds, and geospatial data services
- Webhook-based alerts for task status changes

**Known gaps**
- Less ICS-native than WebEOC — requires manual configuration to match ICS hierarchy
- No built-in mass notification; relies on integrations
- API documentation not publicly available; contact-only access
- Smaller ecosystem than Everbridge

**Licence / IP notes**
- Commercial SaaS; privately held
- No open-source components

---

### D4H Incident Management

**Core features**
- Digital ICS forms: pre-loaded ICS-200 through ICS-220 forms, collaboratively completed
- Incident Action Plan (IAP) builder and meeting agenda generator
- Resource management: centralised database of personnel, equipment, and vehicles with availability and capability tracking
- Real-time incident tracking with shared situation reports
- Integrated communication and collaboration tools
- Detailed post-incident reporting and analytics
- Virtual EOC module for distributed incident command
- API for integration with third-party tools (Everbridge, Connect Rocket, Singlewire, Twilio)

**Differentiating features**
- ICS-native from the ground up — arguably the most ICS-compliant SaaS platform
- Uncomplicated UX designed for small-to-mid-sized agencies that can't afford large enterprise tools
- Per-user pricing makes it accessible for volunteer and small-budget organisations
- Native resource-tracking tied directly to ICS assignments

**UX patterns**
- Clean, task-oriented UI with minimal onboarding; designed for responders not IT staff
- Form-centric interaction model following ICS documentation flow
- Mobile-accessible for field personnel
- Streamlined activation flow from template to active incident in minutes

**Integration points**
- RESTful API documented at api.d4h.org/v2/documentation and developer hub at d4h.com/developer-hub
- Bearer-token authentication
- Team Manager API v3 for human resource data
- Pre-built integrations with Everbridge, Twilio, Singlewire Connect Rocket

**Known gaps**
- Limited mass notification capability — depends on integrations
- No native threat intelligence or risk intelligence feed
- GIS/mapping functionality less mature than DLAN or Veoci
- Smaller ecosystem for large multi-agency mutual-aid events

**Licence / IP notes**
- Commercial SaaS; per-user pricing (approx. $5,000–$15,000/year for small agencies)
- No open-source components

---

### Noggin

**Core features**
- Integrated resilience workspace: emergency management, business continuity, operational risk, and crisis management in one platform
- Automated team activation: notifications, task assignment, and decision recording
- Customisable situational awareness dashboards with scrolling banners, live maps, and external data feeds (news, weather, social media, traffic)
- Built-in communication: chat, email, SMS, voice, and push notifications
- Full emergency management lifecycle support: mitigation, preparedness, response, recovery
- Compliance management and audit trails
- Third-party threat intelligence system connectors
- Computer vision integration (CCTV monitoring)

**Differentiating features**
- Broadest scope of any reviewed tool: combines emergency management with business continuity and operational resilience
- Strong corporate/enterprise market positioning — suited to large private-sector organisations
- Computer vision and CCTV integration for physical security situational awareness
- AWS Marketplace availability simplifies procurement for enterprise IT buyers

**UX patterns**
- Executive and operational views differentiated; senior leaders see summary dashboards while operators see task queues
- Guided activation playbooks; automated notification sequences reduce manual steps
- Progressive disclosure of incident details across the resilience lifecycle

**Integration points**
- REST API with integration-ready connectors (documentation via support portal)
- Integrations with Microsoft Teams, Outlook, ServiceNow, threat intelligence platforms
- Single sign-on (SSO) support
- Available on AWS Marketplace

**Known gaps**
- Less ICS/NIMS-native than WebEOC or D4H — designed for a broader global audience
- API documentation not publicly available; requires customer access
- Primarily enterprise/corporate; government EOC workflows need more configuration
- Cost is a barrier for smaller public-sector agencies

**Licence / IP notes**
- Commercial SaaS; venture-backed (Australian-founded)
- No open-source components

---

### DisasterLAN (DLAN)

**Core features**
- ESRI ArcGIS-integrated GIS mapping for real-time geospatial situational awareness
- Customisable dashboards and visual status boards
- Ticketing system for resource and task management
- Asset and finance tracking (FEMA reimbursement-ready)
- Incident Action Plan (IAP) and Situation Report (SitRep) creators
- Document and contact management
- After-Action Reporting (AAR)
- Aerial incident views and IoT-based asset tracking

**Differentiating features**
- Deepest native ESRI/ArcGIS integration in its class — GIS is a first-class citizen, not an add-on
- Finance tracking built for FEMA Public Assistance reimbursement workflows
- Predictive flood visualisation: rapid "what-if" scenario mapping for decision support
- Long-standing government focus (county and state agencies since 2002)

**UX patterns**
- Map-first interface: GIS view is primary; other panels overlay or adjoin
- Customisable dashboards allow agency-specific views
- Real-time status board updates visible across all connected users
- Relatively traditional UI; no major modern UX redesign noted

**Integration points**
- ESRI partnership; native ArcGIS feature services and web maps
- Web-based with mobile support
- Public API available (documentation contact-based)
- Supports mutual-aid workflows across jurisdictions

**Known gaps**
- Smaller vendor with limited third-party integration ecosystem
- Mass notification not native
- Less UX polish than newer entrants
- Limited documentation publicly available for API integration

**Licence / IP notes**
- Commercial SaaS by Buffalo Computer Graphics; privately held
- No open-source components; ESRI partnership implies ArcGIS licence dependency

---

### AlertMedia

**Core features**
- Multi-channel mass notification: email, SMS, voice, mobile app, desktop, Slack, Teams, WhatsApp, Telegram
- Two-way communication: recipients can respond with long-form messages, confirmations, survey responses, help requests, and location shares
- Expert-vetted threat intelligence: in-house Global Threat Intelligence team provides contextualised summaries
- Audience targeting by location (GPS/geofence), job role, or group
- Pre-built message templates for 150+ scenario types
- API integration for importing threat intelligence from external sources
- Delivery analytics dashboard to confirm message receipt

**Differentiating features**
- Proprietary human-vetted threat intelligence layer distinguishes it from automated-only feeds
- Best-in-class two-way communication with structured response options (confirmations, surveys, location sharing)
- Modern UX — most frequently cited for ease of use compared to legacy mass-notification tools
- Employee safety focus: field responders can request help and share location in real time

**UX patterns**
- Single-screen incident activation with guided message composition
- Template-driven message creation reduces cognitive load during activation
- Mobile-first field experience for both senders and recipients
- Real-time delivery and response dashboards

**Integration points**
- REST API for notification management (example clients on GitHub at github.com/alertmedia/api_client_examples)
- API integration for external threat intelligence ingestion
- Connects to HR/directory systems for contact population
- Contact-based access to full API documentation

**Known gaps**
- Not a full incident command or EOC platform — notification-focused
- No native ICS documentation or IAP builder
- No GIS situational awareness mapping
- Full API documentation requires vendor engagement, not publicly accessible

**Licence / IP notes**
- Commercial SaaS; raised $110M+ in venture funding
- No open-source components

---

### OnSolve (CodeRED / One Call Now)

**Core features**
- Multi-channel mass notification: call, SMS, email, mobile app, social media, WhatsApp
- Geofenced and GPS-targeted audience selection
- Pre-written message templates for 150+ emergency scenarios
- Two-way communication with confirmation and response tracking
- DataSync, XML via SFTP, and REST API for contact database management
- Risk intelligence integration for threat context
- Delivery analytics and message-impact dashboards
- Combination of CodeRED (public alerting), One Call Now (community notification), and Send Word Now (internal alerting) product lines

**Differentiating features**
- Public-facing CodeRED system widely used for community emergency alerts (municipality-focused)
- Three-product portfolio covers public alerting, community notification, and corporate internal communications in one vendor
- SFTP and XML data-sync for agencies with legacy data infrastructure
- Self-registration portal allows residents to opt in to local CodeRED alerts

**UX patterns**
- Product-line-specific portals rather than a unified interface
- Guided notification wizard for rapid alert creation
- Geographic map-based audience selection using polygon drawing or address import

**Integration points**
- REST API with DataSync for automated contact synchronisation
- XML-over-SFTP for legacy agency integration
- API integration with HR and business systems for contact management

**Known gaps**
- Limited operational coordination or incident command features
- No ICS documentation or resource management
- Less modern UX compared to AlertMedia
- Product fragmentation across three sub-brands creates confusion

**Licence / IP notes**
- Commercial SaaS; completed SPAC-adjacent merger; now part of Crisis24 portfolio
- No open-source components

---

### Sahana EDEN

**Core features**
- Organization Registry: directory of agencies, offices, warehouses, and field sites with geolocation
- Human Resources module: staff and volunteer management, skills tracking, assignment records
- Inventory module: supply chain, shipments, multi-catalogue item management
- Project Tracking: "Who's Doing What, Where, When" across multiple responding organisations
- Integrated mapping: every location-based record visualised on maps for situational awareness
- Messaging: email, SMS, Twitter integration; distribution group broadcasts
- CAP-compatible alert handling
- NIMS, ICS, and ESF structure support through configuration

**Differentiating features**
- Only fully open-source platform in the market with broad NIMS/ICS support
- Designed for humanitarian and multi-agency disaster response, not just single-agency EOC
- MIT licence permits unrestricted commercial use and customisation
- Modular architecture allows deployment of only needed modules (emergency, health, logistics, population, collaboration)

**UX patterns**
- Web-based, form-driven UI; functional but not modern
- Requires significant technical setup and configuration
- Suitable for organisations with in-house development capacity
- Mobile access via responsive web; no dedicated native app

**Integration points**
- Python/web2py-based; REST-ish API; GitHub repository at github.com/sahana/eden
- SMS gateway integration (Twilio, bulk SMS providers)
- GIS via OpenLayers and OpenStreetMap
- Data import/export in CSV, XML; CAP message handling

**Known gaps**
- UI is dated; limited UX investment
- No commercial support without engaging the Sahana Foundation or a third-party implementer
- Limited AI/ML capabilities
- Smaller active development community in recent years; codebase maintenance uncertain
- Not actively marketed — difficult to discover without deep research

**Licence / IP notes**
- MIT licence — unrestricted open source; GitHub: github.com/sahana/eden
- No known patent encumbrances
- Copyrights held by respective contributors

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-channel mass notification (SMS, email, voice, mobile app)
- ICS-structured incident management (Incident Commander, Operations, Planning, Logistics, Finance sections)
- Resource requesting and tracking across personnel and equipment
- Real-time GIS/mapping-based situational awareness
- Role-based access control with position-based views
- After-Action Reporting (AAR) with FEMA-compliant forms
- Audit trail and incident chronology logging
- Mobile access for field personnel

### Differentiating Features
- Depth of ESRI/ArcGIS integration (DLAN)
- Human-vetted threat intelligence (AlertMedia)
- Broadest integration ecosystem with 350+ connectors (Everbridge)
- No-code workflow builder for non-technical emergency managers (Veoci)
- All-hazards, multi-lifecycle resilience scope (Noggin)
- Open source and modular architecture (Sahana EDEN)
- Native FEMA reimbursement finance tracking (DLAN)
- ICS-native form library from day one (D4H)

### Underserved Areas / Opportunities
- Unified platform that combines ICS command, mass notification, and resource management without requiring multiple vendor integrations
- AI-assisted situational awareness that aggregates social media, 911 CAD feeds, weather, and sensor data automatically
- Natural language processing to auto-populate ICS forms and situation reports from dictated or typed incident narratives
- Predictive resource optimisation: AI-matched personnel and equipment to incidents based on proximity, capability, and workload
- Real-time public-alerting draft generation from a single incident narrative, auto-formatted as CAP-compliant messages
- Accessible, affordable pricing for small and volunteer agencies (under $5,000/year) with full ICS capability
- Open-source alternative to Sahana EDEN with modern UI and active maintainers
- Cross-jurisdiction mutual-aid resource visibility without manual federation agreements

### AI-Augmentation Candidates
- Automated Common Operating Picture: replace manual data aggregation from multiple feeds with AI-curated situational awareness views
- Smart ICS form completion: LLM fills draft ICS-201, 202, 204 forms from narrative or CAD feed input; human reviews and approves
- Predictive incident spread modelling: wildfire, flood, hazmat plume — proactive resource pre-positioning suggestions
- AI-generated multi-language public alert drafts from a single operator narrative, immediately CAP-formatted
- Resource allocation optimiser: constraint-based AI matching available mutual-aid assets to incident priorities
- Post-incident AAR generation: auto-draft lessons-learned report from event chronology and task logs
- Anomaly detection in incoming data streams to flag new incident signals before human operators recognise them

---

## Legal & IP Summary

All commercial solutions (Everbridge, Juvare, Veoci, D4H, Noggin, DLAN, AlertMedia, OnSolve) are fully proprietary SaaS platforms with no open-source components exposed. Their APIs are documented but the underlying code is protected trade secrets. No known patent claims on core ICS workflow or mass notification features were identified in publicly available sources, though feature differentiation suggests de facto proprietary approaches to specific UX and automation patterns. Sahana EDEN is MIT-licensed with no known patent encumbrances, making it the only freely reusable codebase. An AI-native open-source platform building on ICS standards, CAP protocol, and open geospatial specifications (GeoJSON, OGC WFS) would face no obvious IP obstacles.

---

## Recommended Feature Scope

**Must-have (MVP)**
- ICS-structured incident command interface with configurable section roles (IC, Ops, Planning, Logistics, Finance)
- Digital ICS form library (ICS-201 through ICS-220) with collaborative real-time completion
- Multi-channel notification (SMS, email, voice, mobile push) with audience targeting by role and geography
- Resource database with assignment tracking and availability status
- GIS situational awareness map with real-time incident markers, resource locations, and external data layer support
- After-Action Report generator from incident chronology

**Should-have (v1.1)**
- CAP-compliant public alert origination with IPAWS integration
- AI-assisted ICS form drafting from narrative or voice input
- Automated situational awareness aggregation from weather, CAD, and social media feeds
- Mutual-aid resource sharing across jurisdictions
- Mobile app for field responders with resource request, location sharing, and status updates

**Nice-to-have (backlog)**
- Predictive incident spread modelling (wildfire, flood, hazmat)
- AI resource allocation optimiser
- Multi-language AI-generated public alert drafts
- Business continuity lifecycle integration (pre-incident preparedness, post-incident recovery)
- Computer vision / CCTV feed integration for physical situational awareness
