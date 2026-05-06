# Standards & API Reference

> Project: Emergency Management Platform · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO 22320:2018 — Emergency Management: Guidelines for Incident Management**
- URL: https://www.iso.org/standard/67851.html
- The primary international standard for emergency incident management. Specifies requirements for establishing command, control, and coordination processes; managing operational information; and collecting and sharing data to ensure relevant and accurate information is available throughout an incident. Core compliance target for any incident management platform.

**ISO 22301:2019 — Business Continuity Management Systems (BCMS)**
- URL: https://www.iso.org/standard/75106.html
- Provides the framework for establishing, implementing, maintaining, and continually improving a BCMS. Addresses threat identification, impact assessment, and response/recovery plans. Directly relevant to platforms that extend beyond emergency response into business continuity and recovery lifecycle management.

**ISO 22315 — Mass Evacuation**
- URL: https://www.iso.org/standard/50435.html
- Guidance on planning and managing large-scale evacuation operations. Relevant for resource allocation, zone mapping, and shelter management modules in an emergency management platform.

**ISO 31000:2018 — Risk Management Guidelines**
- URL: https://www.iso.org/standard/65694.html
- Internationally recognised risk management framework. Underpins preparedness and mitigation phases of the emergency management lifecycle; relevant for risk assessment modules and threat prioritisation workflows.

---

### OASIS Emergency Management Standards

**Common Alerting Protocol (CAP) v1.2 — OASIS Standard**
- URL: https://docs.oasis-open.org/emergency/cap/v1.2/CAP-v1.2-os.html
- XML-based data format for exchanging public warnings and emergency alerts across all communication channels and systems simultaneously. Formally adopted as an OASIS standard; required for IPAWS integration in the United States. CAP allows a single alert message to be disseminated over multiple pathways (WEA, EAS, web, mobile, radio) from one origination point. Any platform that supports public alerting must produce CAP-compliant messages.

**EDXL (Emergency Data Exchange Language) — OASIS Standard**
- URL: https://www.oasis-open.org/committees/emergency/
- A family of XML-based standards for emergency data exchange including EDXL-DE (Distribution Element), EDXL-HAVE (Hospital Availability Exchange), and EDXL-SitRep (Situation Reporting). Relevant for inter-agency data sharing, hospital capacity tracking during mass-casualty incidents, and structured situation report exchange.

**SAML 2.0 — OASIS Standard**
- URL: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- XML-based SSO federation standard used by enterprise and government identity providers. Emergency management platforms must support SAML 2.0 for single sign-on integration with state and federal identity management systems. Complements OAuth 2.0 / OIDC for modern API authentication.

---

### US Federal Standards & Frameworks

**NIMS — National Incident Management System**
- URL: https://www.fema.gov/emergency-managers/nims
- FEMA's national framework requiring all federally funded emergency management programs to use ICS-compatible software and NIMS-compliant command structures. Sets requirements for incident organisation (Incident Commander, Operations, Planning, Logistics, Finance/Administration), resource management, and communications interoperability. Compliance is a procurement prerequisite for most US government buyers.

**ICS — Incident Command System**
- URL: https://training.fema.gov/icsresource/
- Hierarchical command structure that defines roles, responsibilities, and documentation requirements for incident response. Platform data models must reflect ICS forms (ICS-201 through ICS-220), ICS position assignments, and span-of-control rules. The de facto operating model for US fire, law enforcement, public health, and emergency management.

**IPAWS — Integrated Public Alert and Warning System**
- URL: https://www.fema.gov/emergency-managers/practitioners/integrated-public-alert-warning-system
- FEMA-operated alert infrastructure unifying Wireless Emergency Alerts (WEA), Emergency Alert System (EAS), and NOAA Weather Radio. Platforms seeking to originate public alerts must be IPAWS-certified alert origination software using CAP v1.2 with the IPAWS profile. Developer resources at: https://www.fema.gov/emergency-managers/practitioners/integrated-public-alert-warning-system/technology-developers/ipaws-open

**FEMA CPG 101 — Comprehensive Preparedness Guide**
- URL: https://www.fema.gov/sites/default/files/2020-05/CPG_101_V2_11-2010.pdf
- Federal guidance for writing emergency operations plans (EOPs). Informs the workflow and document structure requirements for any platform that supports EOP creation, exercise planning, and plan maintenance.

**FEMA ESF Framework — Emergency Support Functions**
- URL: https://www.fema.gov/emergency-managers/national-preparedness/frameworks/response
- 15 Emergency Support Functions define inter-agency coordination responsibilities under the National Response Framework. Platform role/position models should support ESF assignments alongside ICS structure.

---

### W3C & IETF Standards

**GeoJSON — RFC 7946 (IETF)**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- Standard JSON format for encoding geographic data structures (points, lines, polygons, feature collections). The universal data interchange format for geospatial incident data: incident locations, evacuation zones, resource positions, and affected area polygons should all be GeoJSON-compatible. Supported natively by Leaflet, Mapbox, ArcGIS, and all major mapping libraries.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1)**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational HTTP semantics governing REST API design. All platform APIs should comply with HTTP/1.1 method semantics (GET, POST, PUT, PATCH, DELETE) and status code conventions.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Industry-standard framework for delegated API authorisation. Emergency management platforms must support OAuth 2.0 for secure API integrations with third-party systems (HR, CAD, notification services). Client Credentials and Authorization Code flows are the most applicable grant types.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Authentication layer built on OAuth 2.0. Required alongside OAuth 2.0 for user authentication in multi-agency deployments. Enables federated identity with state and federal identity providers.

**RFC 4287 — Atom Syndication Format**
- URL: https://datatracker.ietf.org/doc/html/rfc4287
- XML feed format used by IPAWS for distributing CAP alerts via HTTP Atom feeds. Platforms subscribing to the IPAWS All-Hazards Information Feed receive CAP messages wrapped in Atom feed entries.

**W3C SSE — Server-Sent Events**
- URL: https://www.w3.org/TR/eventsource/
- W3C standard for server-to-client streaming over HTTP. Preferred for real-time dashboard updates (incident status changes, resource assignments, incoming alerts) without the overhead of WebSocket connections.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- De facto standard for documenting REST APIs. All platform APIs should be described in OpenAPI 3.1 to enable auto-generated SDKs, interactive documentation (Swagger UI), and third-party integration tooling. Everbridge and D4H both publish OpenAPI-described APIs.

**NIEM — National Information Exchange Model**
- URL: https://www.niem.gov/
- XML/JSON data modelling framework for inter-agency information exchange across Justice, Emergency Management, and Homeland Security domains. NIEM 6.0 supports both XML Schema and JSON-LD representations. The NIEM Emergency Management domain defines standard data elements for incident type, severity, resource request, mutual-aid agreement, and situation report fields. Relevant for platforms targeting interoperability with state/federal systems.

**OGC API — Features (OGC Standard)**
- URL: https://ogcapi.ogc.org/features/
- Modern OGC standard for exposing geospatial feature data via REST/JSON. Successor to WFS (Web Feature Service). Relevant for platforms that expose incident, resource, or zone data as geospatial features to GIS clients such as ArcGIS or QGIS.

**OGC WMS / WFS (Web Map/Feature Service)**
- URL: https://www.ogc.org/standards/wms and https://www.ogc.org/standards/wfs
- Legacy OGC standards for consuming external geospatial data layers. ESRI ArcGIS, DLAN, and other GIS-integrated platforms use WFS/WMS to ingest authoritative government map layers (flood zones, evacuation routes, critical infrastructure).

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for validating and documenting JSON data structures. Should be used to define and validate all API request/response bodies, WebSocket message payloads, and CAP JSON translations.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OAuth 2.1 (Draft)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 and https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Mandatory for all API integrations. Client Credentials grant for machine-to-machine integrations (CAD, notification systems, weather feeds); Authorization Code + PKCE for user-facing applications.

**SAML 2.0**
- URL: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- Required for integration with government identity management systems (Active Directory Federation Services, state SSO portals). Most US EOC platforms must support SAML 2.0 SSO.

**NIST SP 800-53 — Security and Privacy Controls for Federal Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Federal baseline for security controls. Platforms seeking FedRAMP authorisation must implement NIST 800-53 Rev 5 controls. Relevant for access control (AC), audit and accountability (AU), incident response (IR), and system and communications protection (SC) control families.

**FedRAMP**
- URL: https://www.fedramp.gov/
- Federal Risk and Authorization Management Program certification required for cloud platforms processing federal agency data. Government-facing emergency management platforms must pursue FedRAMP Moderate or High authorisation for US federal contracts.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/download/soc-2-report-on-controls-at-a-service-organization-relevant-to-security-availability-processing-integrity-confidentiality-or-privacy
- Industry-standard security audit required by most enterprise and government buyers. Covers security, availability, processing integrity, confidentiality, and privacy trust service criteria. Over 73% of enterprise buyers require SOC 2 reports before vendor onboarding.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- Baseline web application security requirements. Injection, broken authentication, insecure direct object references, and misconfiguration are particularly relevant for multi-tenant emergency management platforms handling sensitive incident data.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic / Open Standard**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- GitHub: https://github.com/modelcontextprotocol
- Open protocol (2025-11-25 spec) enabling LLMs to interact with external tools, data sources, and services via a standardised interface. By 2026 MCP is natively supported by Anthropic, OpenAI, Google, and Microsoft. An emergency management platform could expose MCP servers for: incident status queries, resource availability checks, CAP alert drafting, and ICS form completion — enabling AI agents to assist operators or auto-populate documentation from natural-language input.

---

## Similar Products — Developer Documentation & APIs

### Everbridge 360

- **Description:** Enterprise Critical Event Management platform covering mass notification, incident command, risk intelligence, and CEM orchestration across 350+ integrations.
- **API Documentation:** https://developers.everbridge.net/home
- **API Reference (Swagger):** https://api.everbridge.net/
- **Developer Guide:** https://developers.everbridge.net/home/docs/ebs-gs-guide
- **Standards:** REST/JSON; OpenAPI-described; OAuth 2.0 (Password Grant, Client Credentials)
- **Authentication:** OAuth 2.0

---

### D4H Incident Management

- **Description:** ICS-native incident management SaaS targeting small-to-mid agencies; digital ICS forms, resource tracking, virtual EOC, and per-user affordable pricing.
- **API Documentation:** https://api.d4h.org/v2/documentation
- **Developer Hub:** https://www.d4h.com/developer-hub
- **Team Manager API v3:** https://api.team-manager.us.d4h.com/v3/docs
- **SDKs/Libraries:** No official SDKs; REST/JSON with bearer-token auth
- **Standards:** REST/JSON; OpenAPI-described
- **Authentication:** Bearer token

---

### Juvare WebEOC

- **Description:** Web-based EOC software widely deployed in US county and state emergency management agencies; board-based ICS/NIMS-compliant incident coordination.
- **API Documentation:** https://docs.juvare.com/webeoc-onnexa/rest-api/
- **Technical Guide (legacy v7.3):** http://www.govbids.com/storeddoc2/Florida/Documents/Amend/105328_4_3.PDF
- **SDKs/Libraries:** No official SDKs; session-based authentication with REST endpoints
- **Standards:** REST/JSON; session + bearer authentication
- **Authentication:** Session-based (username, password, position, incident context)

---

### AlertMedia

- **Description:** Modern emergency mass notification platform with two-way communication, human-vetted threat intelligence, and audience targeting by location and role.
- **API Examples (GitHub):** https://github.com/alertmedia/api_client_examples
- **Developer Guide:** Available via Customer Success Manager; not publicly documented
- **SDKs/Libraries:** Community example clients in Python/other languages on GitHub
- **Standards:** REST/JSON
- **Authentication:** API key (contact vendor for credentials)

---

### FEMA OpenFEMA API

- **Description:** FEMA's public REST API providing read-only access to disaster declarations, public assistance grants, individual assistance data, NFIP policies, and IPAWS archived alerts — a critical data source for situational awareness and historical analysis.
- **API Documentation:** https://www.fema.gov/about/openfema/api
- **Developer Resources:** https://www.fema.gov/about/openfema/developer-resources
- **GitHub Samples:** https://github.com/FEMA/openfema-samples
- **IPAWS Archived Alerts:** https://www.fema.gov/openfema-data-page/ipaws-archived-alerts-v1
- **SDKs/Libraries:** Python beta wrapper; R package (rfema)
- **Standards:** REST/JSON; no API key required
- **Authentication:** No authentication required (public read-only)

---

### FEMA IPAWS-OPEN (Alert Feed)

- **Description:** FEMA's real-time and archived CAP alert feed for all IPAWS-disseminated public alerts. Used to subscribe to Wireless Emergency Alerts, EAS messages, and NWS weather alerts in CAP format.
- **Feed Documentation:** https://www.fema.gov/emergency-managers/practitioners/integrated-public-alert-warning-system/technology-developers/ipaws-open
- **All-Hazards Information Feed:** https://www.fema.gov/emergency-managers/practitioners/integrated-public-alert-warning-system/technology-developers/all-hazards-information-feed
- **CAP Protocol Reference:** https://www.fema.gov/emergency-managers/practitioners/integrated-public-alert-warning-system/technology-developers/common-alerting-protocol
- **Standards:** HTTP Atom feed; CAP v1.2 XML messages with IPAWS profile
- **Authentication:** No authentication for public feed; IPAWS-OPEN system registration required for alert origination

---

### NWS Weather API

- **Description:** NOAA National Weather Service public REST API providing weather alerts, forecasts, and observation data in CAP/JSON format. Essential for ingesting weather-driven hazard triggers into situational awareness dashboards.
- **API Documentation:** https://www.weather.gov/documentation/services-web-alerts
- **Standards:** REST/JSON; GeoJSON feature collections for alert zones
- **Authentication:** No authentication required (public API); User-Agent header required

---

### Sahana EDEN

- **Description:** MIT-licensed open-source emergency and disaster management platform with organisation registry, HR, inventory, project tracking, GIS mapping, and CAP messaging.
- **GitHub Repository:** https://github.com/sahana/eden
- **Eden-Core (Framework):** https://github.com/sahana/eden-core
- **Foundation Site:** https://sahanafoundation.org/eden/
- **SDKs/Libraries:** Python (web2py framework); REST-ish API; community-maintained
- **Standards:** REST/JSON and XML; CAP message handling; OGC/OpenStreetMap mapping
- **Authentication:** Session-based; configurable per-deployment; LDAP integration available
- **Licence:** MIT

---

## Notes

**Emerging and evolving areas:**

1. **MCP for Emergency AI Agents**: The Model Context Protocol is gaining enterprise adoption in 2026 and is well-suited for emergency management AI agents that need to query incident state, draft ICS forms, or generate CAP alerts via natural language. No production emergency management platform has published an MCP server yet — this is a clear first-mover opportunity.

2. **NIEM JSON Adoption**: NIEM 6.0 introduced JSON-LD representations alongside legacy XML. Federal data exchange mandates are gradually shifting to JSON, but many legacy state and county systems still use NIEM XML. A new platform should support both NIEM JSON and XML serialisations for maximum interoperability.

3. **OGC API Transition**: The OGC is transitioning from WMS/WFS to the modern OGC API suite (Features, Maps, Processes). New platforms should implement OGC API — Features rather than legacy WFS, but must also consume WFS from authoritative government GIS services that have not yet migrated.

4. **FedRAMP Pathways**: FedRAMP authorization has historically taken 12–18 months. The FedRAMP 20x initiative (2026) aims to reduce this to weeks via automated continuous monitoring. An AI-native platform should design for FedRAMP Moderate compliance from the outset to avoid costly re-architecture for government procurement.

5. **Real-time Data Standards**: No single dominant standard exists yet for streaming real-time sensor and IoT data into emergency management platforms. MQTT (IoT messaging) and SSE (Server-Sent Events) are de facto approaches; OGC is developing Sensor Things API as a standardised solution.
