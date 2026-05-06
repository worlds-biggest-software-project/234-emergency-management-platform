# Emergency Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for incident command, resource tracking, public alerts, and recovery management — built on ICS, NIMS, and CAP standards.

The Emergency Management Platform unifies ICS-structured incident command, multi-channel mass notification, resource tracking, and recovery workflows in a single open-source system. It is designed for county and state emergency managers, fire and police EOC coordinators, hospital incident command teams, and corporate business-continuity managers who need a modern, affordable alternative to fragmented commercial tools.

---

## Why Emergency Management Platform?

- Incumbent platforms are expensive: agency-side tools typically range from $20,000–$250,000+/year, pricing out small jurisdictions and volunteer agencies.
- The market is fragmented — Everbridge dominates mass notification, WebEOC owns ICS documentation, DLAN leads on GIS — forcing agencies to integrate multiple vendors to cover one incident lifecycle.
- Legacy tools like WebEOC carry dated UIs and heavy configuration burdens; even modern entrants like Veoci require significant plan-building effort before delivering value.
- The only existing open-source option, Sahana EDEN, is MIT-licensed but has a dated UI, limited AI capabilities, and uncertain ongoing maintenance.
- Climate-driven disaster frequency and post-pandemic business-continuity demand are expanding the buyer base faster than incumbent licensing models can serve.

---

## Key Features

### Incident Command (ICS / NIMS)

- ICS-structured command interface with configurable section roles (Incident Commander, Operations, Planning, Logistics, Finance)
- Digital ICS form library (ICS-201 through ICS-220) with collaborative real-time completion
- Incident Action Plan (IAP) builder and meeting agenda generator
- Position-based role configuration and role-based access control
- Real-time incident chronology and audit trail logging

### Mass Notification & Public Alerting

- Multi-channel notification across SMS, email, voice, and mobile push
- Audience targeting by role, group, and geography (GPS / geofence)
- Two-way communication with confirmations, surveys, and location sharing
- CAP-compliant public alert origination with IPAWS integration
- Pre-built message templates for common emergency scenarios

### Situational Awareness & GIS

- Real-time GIS map with incident markers, resource locations, and external data layers
- Open geospatial support: GeoJSON and OGC WFS feeds
- Aggregation of weather, 911 CAD, and social media feeds into a common operating picture
- External data feed support for sensor and IoT inputs

### Resource & Recovery Management

- Resource database covering personnel, equipment, and vehicles with availability and capability tracking
- Resource requesting, assignment, and task tracking tied to ICS assignments
- Mutual-aid resource sharing across jurisdictions
- After-Action Report generator from incident chronology
- FEMA-compliant reporting and reimbursement-ready documentation

### Mobile Field Operations

- Mobile access for field responders
- Resource requests, location sharing, and status updates from the field
- Offline-tolerant field workflows for responders in degraded-connectivity environments

---

## AI-Native Advantage

AI shifts the platform from a system of record to an active operational partner. Natural-language incident logging lets responders dictate situation reports that are auto-structured into ICS-201, 202, and 204 drafts for human review. AI-curated common operating pictures aggregate social media, CAD feeds, weather, and sensor data into a unified real-time view. Constraint-based optimisation matches available mutual-aid personnel and equipment to incident priorities, and a single operator narrative can generate CAP-compliant, multi-language public alert drafts in seconds.

---

## Tech Stack & Deployment

The platform is designed around open emergency management standards: NIMS and ICS for command structure, CAP (OASIS) for alert exchange, IPAWS for public alerting, and the ESF framework for inter-agency coordination. Geospatial layers use open formats (GeoJSON, OGC WFS) rather than proprietary GIS dependencies. Deployment targets self-hosted operation for sovereignty-sensitive agencies as well as cloud hosting; APIs follow REST conventions with OAuth-style authentication patterns aligned with incumbent integration expectations.

---

## Market Context

The global incident and emergency management market was valued at approximately $75 million in 2017 and was projected to reach $423 million by 2026, with broader critical-event management extending the addressable market further (DHS S&T 2022; Gartner 2026). Incumbent agency-side platforms price from $20,000–$250,000+/year, with smaller ICS tools like D4H starting at $5,000–$15,000/year. Primary buyers are county emergency managers, state emergency management agency directors, fire and police EOC coordinators, hospital incident command teams, and corporate security and business-continuity managers.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
