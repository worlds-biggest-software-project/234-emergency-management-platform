# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Emergency Management Platform · Created: 2026-05-22

## Philosophy

This model keeps the structural backbone relational -- incidents, resources, organisations, and users are proper tables with typed columns, foreign keys, and indexes -- but uses PostgreSQL JSONB columns extensively for data that varies by jurisdiction, incident type, ICS form version, or agency configuration. Instead of creating a separate table for every ICS form type or a rigid column for every jurisdiction-specific field, the schema defines "core fields" relationally and "extension fields" in JSONB.

This is the approach that modern no-code platforms like Veoci use internally: a fixed set of system tables provide structure and referential integrity, while the actual form content and workflow configuration live in flexible JSON documents. It is also how Sahana Eden handles its modular data model -- core entities like `pr_person`, `org_organisation`, and `inv_item` have fixed columns, but supplementary data is stored in configurable "components" that map to JSON-like extension patterns.

The hybrid approach is ideal for a platform that must serve diverse agencies -- a county emergency manager in Colorado has different ICS form requirements than a hospital incident command team in New York -- without requiring schema migrations for every variation. It also supports rapid MVP development: the core schema can be deployed in weeks, with jurisdiction-specific fields added via JSONB without database changes.

**Best for:** Multi-jurisdiction deployments, rapid MVP development, and agencies that need to customise forms and workflows without waiting for schema migrations.

**Trade-offs:**
- Pro: Core relational integrity (incidents always have a type, status, and location) with flexible extension points
- Pro: No DDL migration needed for new form fields, jurisdiction-specific data, or custom attributes
- Pro: JSONB columns support GIN indexes for fast containment queries
- Pro: Schema can be deployed and adapted quickly -- ideal for MVP and iterative development
- Pro: Natural fit for multi-tenant platforms where each tenant customises their forms
- Con: JSONB fields lack column-level constraints -- application-layer validation is required
- Con: Complex reporting across JSONB fields requires jsonb_path_query or custom extraction
- Con: Schema documentation must live outside the DDL (in JSON Schema definitions) for JSONB structures
- Con: Risk of "JSONB creep" where developers put everything in JSONB, undermining the relational core
- Con: JSONB migration tooling is less mature than DDL migration tooling (e.g., Flyway, Alembic)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIMS / ICS | Core ICS structure (incidents, positions, assignments) is relational; ICS form content stored as JSONB documents with JSON Schema validation per form type |
| CAP v1.2 (OASIS) | CAP alerts stored as relational rows with core fields (identifier, sender, status, msg_type); info/area/resource segments stored as JSONB arrays |
| NIEM 6.0 | JSONB payloads follow NIEM element naming conventions; JSON-LD export views generate NIEM-compatible documents |
| EDXL-RM | Resource request payloads stored as JSONB with EDXL-RM field names for direct serialisation to XML |
| ISO 3166 | Jurisdiction codes in relational columns; jurisdiction-specific configuration in tenant JSONB settings |
| GeoJSON (RFC 7946) | PostGIS geometry columns for indexed spatial queries; GeoJSON documents in JSONB for non-indexed geographic metadata |
| JSON Schema | Every JSONB column has a corresponding JSON Schema definition that the application layer validates against |
| FEMA NIMS Resource Typing | Resource type definitions as relational reference data; resource-specific metrics in JSONB `capabilities` field |
| ESF Framework | ESF functions as relational reference data; ESF-specific configuration per incident in JSONB |

---

## Core Schema

```sql
-- ============================================================
-- TENANCY & CONFIGURATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    jurisdiction    TEXT,                                    -- ISO 3166-2
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "ics_form_schemas": {"ICS-201": {...json_schema...}},
    --   "custom_fields": {"incident": [{"name": "burn_permit_number", "type": "text"}]},
    --   "notification_channels": ["sms", "email", "voice"],
    --   "map_defaults": {"center": [-105.0, 39.7], "zoom": 10},
    --   "mutual_aid_partners": ["uuid1", "uuid2"]
    -- }
    branding        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    phone           TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    external_id     TEXT,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "certifications": ["ICS-100", "ICS-200", "ICS-300", "HAZWOPER"],
    --   "skills": ["swift_water_rescue", "hazmat_technician"],
    --   "languages": ["en", "es"],
    --   "emergency_contact": {"name": "Jane Doe", "phone": "+1234567890"}
    -- }
    roles           TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_email ON app_user(email);
CREATE INDEX idx_user_roles ON app_user USING GIN(roles);
CREATE INDEX idx_user_profile ON app_user USING GIN(profile);
```

## Organisation & Facility Tables

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    acronym         TEXT,
    org_type        TEXT NOT NULL,
    jurisdiction    TEXT,
    parent_org_id   UUID REFERENCES organisation(id),
    contact         JSONB NOT NULL DEFAULT '{}',
    -- contact example:
    -- {
    --   "primary_email": "oem@county.gov",
    --   "primary_phone": "+1-555-0100",
    --   "dispatch_phone": "+1-555-0199",
    --   "address": {"street": "123 Main", "city": "Boulder", "state": "CO", "zip": "80302"},
    --   "website": "https://county.gov/oem"
    -- }
    location        GEOMETRY(Point, 4326),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_tenant ON organisation(tenant_id);
CREATE INDEX idx_org_location ON organisation USING GIST(location);
CREATE INDEX idx_org_parent ON organisation(parent_org_id);

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    facility_type   TEXT NOT NULL,
    location        GEOMETRY(Point, 4326),
    capacity        INTEGER,
    status          TEXT NOT NULL DEFAULT 'operational',
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes example (hospital):
    -- {
    --   "beds_total": 250, "icu_beds": 30, "ventilators": 20,
    --   "trauma_level": 2, "helipad": true, "decon": true,
    --   "burn_unit": false, "pediatric_icu": true
    -- }
    -- attributes example (shelter):
    -- {
    --   "capacity_persons": 500, "ada_accessible": true,
    --   "pet_friendly": false, "generator": true,
    --   "kitchen": true, "showers": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_facility_tenant ON facility(tenant_id);
CREATE INDEX idx_facility_org ON facility(organisation_id);
CREATE INDEX idx_facility_location ON facility USING GIST(location);
CREATE INDEX idx_facility_attrs ON facility USING GIN(attributes);
```

## Incident Management Tables

```sql
-- ============================================================
-- INCIDENTS (Core relational + JSONB extensions)
-- ============================================================

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_number TEXT NOT NULL,
    name            TEXT NOT NULL,
    incident_type   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    severity        TEXT NOT NULL DEFAULT 'moderate',
    description     TEXT,
    location_name   TEXT,
    location        GEOMETRY(Point, 4326),
    affected_area   GEOMETRY(Polygon, 4326),
    jurisdiction    TEXT,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    esf_activations INTEGER[] DEFAULT '{}',                 -- activated ESF numbers
    -- JSONB for incident-type-specific and jurisdiction-specific fields
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example (wildfire):
    -- {
    --   "acres_burned": 1200,
    --   "percent_contained": 35,
    --   "fuel_type": "grass_and_brush",
    --   "fire_behavior": "wind_driven",
    --   "structures_threatened": 450,
    --   "structures_destroyed": 12,
    --   "evacuation_zones": ["Zone-A", "Zone-B"],
    --   "burn_permit": "2026-BP-0042",
    --   "nfdrs_rating": "extreme"
    -- }
    -- details example (flood):
    -- {
    --   "river_gauge_id": "USGS-06730200",
    --   "crest_forecast_ft": 14.5,
    --   "flood_stage_ft": 9.0,
    --   "inundation_map_url": "https://...",
    --   "dam_status": "normal"
    -- }
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, incident_number)
);
CREATE INDEX idx_incident_tenant ON incident(tenant_id);
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_type ON incident(incident_type);
CREATE INDEX idx_incident_location ON incident USING GIST(location);
CREATE INDEX idx_incident_area ON incident USING GIST(affected_area);
CREATE INDEX idx_incident_start ON incident(start_time);
CREATE INDEX idx_incident_details ON incident USING GIN(details);

CREATE TABLE operational_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    period_number   INTEGER NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    objectives      TEXT[],
    weather         JSONB,
    -- weather example:
    -- {
    --   "forecast": "SW winds 15-25 mph, gusts to 40, RH 8%",
    --   "temperature_high_f": 95,
    --   "wind_speed_mph": 25,
    --   "humidity_pct": 8,
    --   "nws_zone": "COZ039"
    -- }
    safety_message  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (incident_id, period_number)
);
CREATE INDEX idx_op_period_incident ON operational_period(incident_id);
```

## ICS Forms as JSONB Documents

```sql
-- ============================================================
-- ICS FORMS (Flexible JSONB Content)
-- ============================================================

-- JSON Schema registry for form validation
CREATE TABLE form_schema (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenant(id),             -- null = system-wide default
    form_type       TEXT NOT NULL,                          -- 'ICS-201', 'ICS-202', ..., 'custom'
    version         INTEGER NOT NULL DEFAULT 1,
    schema          JSONB NOT NULL,                         -- JSON Schema definition
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, form_type, version)
);

-- ICS positions (relational for hierarchy queries)
CREATE TABLE ics_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    title           TEXT NOT NULL,
    section         TEXT NOT NULL,
    parent_id       UUID REFERENCES ics_position(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ics_pos_tenant ON ics_position(tenant_id);

-- ICS assignments (relational for cross-referencing)
CREATE TABLE ics_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    position_id     UUID NOT NULL REFERENCES ics_position(id),
    user_id         UUID REFERENCES app_user(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at     TIMESTAMPTZ,
    notes           TEXT
);
CREATE INDEX idx_ics_assign_incident ON ics_assignment(incident_id);

-- Unified ICS form table with JSONB content
CREATE TABLE ics_form (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    form_type       TEXT NOT NULL,                          -- 'ICS-201', 'ICS-202', ..., 'ICS-220'
    form_schema_id  UUID REFERENCES form_schema(id),
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- All form content lives in JSONB
    content         JSONB NOT NULL DEFAULT '{}',
    -- ICS-201 content example:
    -- {
    --   "incident_name": "Marshall Fire Complex",
    --   "date_prepared": "2026-05-22",
    --   "time_prepared": "0830",
    --   "map_sketch_url": "https://...",
    --   "situation_summary": "Fast-moving grass fire...",
    --   "objectives": ["Protect structures in Zone A", "Establish evacuation corridor"],
    --   "current_actions": ["Engine companies deployed along Hwy 93", ...],
    --   "current_organisation": {
    --     "incident_commander": {"name": "Smith", "agency": "LCSO"},
    --     "operations": {"name": "Jones", "agency": "BFD"},
    --     "planning": null,
    --     "logistics": null,
    --     "finance": null
    --   },
    --   "resource_summary": [
    --     {"resource": "Engine 42", "type": "TYPE-1", "qty": 1, "eta": null, "location": "Hwy 93 / Marshall Rd"},
    --     {"resource": "Crew 7", "type": "HAND-CREW-TYPE2", "qty": 20, "eta": "0930", "location": "Staging Area 1"}
    --   ]
    -- }
    --
    -- ICS-204 content example:
    -- {
    --   "branch": "Branch 1",
    --   "division_group": "Division Alpha",
    --   "operations_chief": {"name": "Jones", "contact": "TAC-1"},
    --   "resources_assigned": [
    --     {"resource_id": "uuid", "leader": "Capt. Rivera", "contact": "TAC-2", "num_persons": 4, "reporting": "0600 at Staging 1"}
    --   ],
    --   "work_assignments": "Structure protection along Marshall Rd from Hwy 93 to Cherryvale",
    --   "special_instructions": "Watch for power line hazards; utility crew en route",
    --   "communications": {"freq": "151.2500", "system": "County TAC", "channel": "TAC-2"}
    -- }
    prepared_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_form_incident ON ics_form(incident_id);
CREATE INDEX idx_form_type ON ics_form(form_type);
CREATE INDEX idx_form_status ON ics_form(status);
CREATE INDEX idx_form_content ON ics_form USING GIN(content);
```

## Resource Management Tables

```sql
-- ============================================================
-- RESOURCE MANAGEMENT
-- ============================================================

CREATE TABLE resource_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        TEXT NOT NULL,                          -- NIMS category
    kind            TEXT NOT NULL,                          -- 'personnel', 'equipment', 'team', 'supply'
    type_code       TEXT NOT NULL UNIQUE,
    type_name       TEXT NOT NULL,
    type_level      INTEGER NOT NULL,
    capabilities    JSONB NOT NULL DEFAULT '{}',
    -- capabilities example:
    -- {
    --   "pump_capacity_gpm": 1000,
    --   "tank_capacity_gal": 750,
    --   "hose_length_ft": 1200,
    --   "crew_size": 4,
    --   "certifications_required": ["CDL", "EVD"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE resource (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    resource_type_id UUID NOT NULL REFERENCES resource_type(id),
    name            TEXT NOT NULL,
    identifier      TEXT,
    status          TEXT NOT NULL DEFAULT 'available',
    home_base_id    UUID REFERENCES facility(id),
    current_location GEOMETRY(Point, 4326),
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes example (vehicle):
    -- {
    --   "make": "Pierce", "model": "Enforcer", "year": 2022,
    --   "vin": "...", "license_plate": "...",
    --   "pump_capacity_gpm": 1500, "tank_gal": 750,
    --   "last_service_date": "2026-04-15",
    --   "mileage": 42000,
    --   "certifications": ["NFPA-1901"]
    -- }
    -- attributes example (person):
    -- {
    --   "badge_number": "4521",
    --   "rank": "Captain",
    --   "certifications": ["EMT-P", "HAZWOPER", "ICS-300"],
    --   "languages": ["en", "es"],
    --   "physical_fitness_date": "2026-03-01"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_resource_tenant ON resource(tenant_id);
CREATE INDEX idx_resource_status ON resource(status);
CREATE INDEX idx_resource_type ON resource(resource_type_id);
CREATE INDEX idx_resource_location ON resource USING GIST(current_location);
CREATE INDEX idx_resource_attrs ON resource USING GIN(attributes);

CREATE TABLE resource_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    requesting_org_id UUID NOT NULL REFERENCES organisation(id),
    requested_by    UUID NOT NULL REFERENCES app_user(id),
    resource_type_id UUID NOT NULL REFERENCES resource_type(id),
    quantity        INTEGER NOT NULL DEFAULT 1,
    priority        TEXT NOT NULL DEFAULT 'routine',
    needed_by       TIMESTAMPTZ,
    delivery_location GEOMETRY(Point, 4326),
    justification   TEXT,
    status          TEXT NOT NULL DEFAULT 'pending',
    fulfillment     JSONB,
    -- fulfillment example:
    -- {
    --   "approved_by": "uuid",
    --   "approved_at": "2026-05-22T10:00:00Z",
    --   "resources_assigned": ["uuid1", "uuid2"],
    --   "source_org": "Boulder Rural Fire",
    --   "mutual_aid_agreement": "uuid",
    --   "eta": "2026-05-22T11:30:00Z",
    --   "notes": "Dispatched via mutual aid; 2 engines from BRFD"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_request_incident ON resource_request(incident_id);
CREATE INDEX idx_request_status ON resource_request(status);

CREATE TABLE resource_deployment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES resource(id),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    request_id      UUID REFERENCES resource_request(id),
    op_period_id    UUID REFERENCES operational_period(id),
    deployed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    demobilised_at  TIMESTAMPTZ,
    assignment      JSONB NOT NULL DEFAULT '{}',
    -- assignment example:
    -- {
    --   "division": "Alpha",
    --   "branch": "Branch 1",
    --   "task": "Structure protection",
    --   "reporting_location": "Staging Area 1",
    --   "supervisor": "Capt. Rivera",
    --   "radio_channel": "TAC-2"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deploy_resource ON resource_deployment(resource_id);
CREATE INDEX idx_deploy_incident ON resource_deployment(incident_id);
```

## Notification & Alerting Tables

```sql
-- ============================================================
-- NOTIFICATIONS & CAP ALERTS
-- ============================================================

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_id     UUID REFERENCES incident(id),
    sender_id       UUID NOT NULL REFERENCES app_user(id),
    subject         TEXT NOT NULL,
    body            TEXT NOT NULL,
    channels        TEXT[] NOT NULL,
    targeting       JSONB NOT NULL DEFAULT '{}',
    -- targeting example:
    -- {
    --   "type": "geo_and_role",
    --   "geo_area": {"type": "Polygon", "coordinates": [...]},
    --   "roles": ["field_responder", "ems"],
    --   "organisations": ["uuid1"],
    --   "exclude_users": ["uuid2"]
    -- }
    status          TEXT NOT NULL DEFAULT 'draft',
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example:
    -- {
    --   "total_recipients": 342,
    --   "delivered": 338,
    --   "failed": 4,
    --   "read": 201,
    --   "responded": 45,
    --   "channels": {"sms": {"sent": 342, "delivered": 338}, "email": {"sent": 200, "delivered": 198}}
    -- }
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notif_tenant ON notification(tenant_id);
CREATE INDEX idx_notif_incident ON notification(incident_id);

CREATE TABLE notification_recipient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL REFERENCES notification(id),
    user_id         UUID REFERENCES app_user(id),
    channel         TEXT NOT NULL,
    destination     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    delivered_at    TIMESTAMPTZ,
    response        JSONB,
    -- response example:
    -- {
    --   "type": "confirmation",
    --   "value": "safe",
    --   "location": {"type": "Point", "coordinates": [-105.23, 39.95]},
    --   "message": "I'm okay, sheltering at home",
    --   "responded_at": "2026-05-22T09:15:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recip_notification ON notification_recipient(notification_id);
CREATE INDEX idx_recip_status ON notification_recipient(status);

-- CAP Alert (hybrid: core CAP fields relational, info/area/resource as JSONB)
CREATE TABLE cap_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_id     UUID REFERENCES incident(id),
    -- Core CAP alert-level fields (relational for queries)
    identifier      TEXT NOT NULL,
    sender          TEXT NOT NULL,
    sent            TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL,                          -- 'Actual', 'Exercise', 'System', 'Test', 'Draft'
    msg_type        TEXT NOT NULL,                          -- 'Alert', 'Update', 'Cancel', 'Ack', 'Error'
    scope           TEXT NOT NULL,                          -- 'Public', 'Restricted', 'Private'
    -- CAP info, area, and resource segments as JSONB arrays
    info_segments   JSONB NOT NULL DEFAULT '[]',
    -- info_segments example:
    -- [
    --   {
    --     "language": "en-US",
    --     "category": ["Fire"],
    --     "event": "Wildfire Warning",
    --     "urgency": "Immediate",
    --     "severity": "Extreme",
    --     "certainty": "Observed",
    --     "headline": "Wildfire Warning for Boulder County",
    --     "description": "Fast-moving wildfire...",
    --     "instruction": "Evacuate immediately via Hwy 36...",
    --     "effective": "2026-05-22T08:30:00-06:00",
    --     "expires": "2026-05-22T20:30:00-06:00",
    --     "areas": [
    --       {"area_desc": "Boulder County evacuation zone", "polygon": "39.95,-105.23 39.96,...", "geocode": {"FIPS6": "008013"}}
    --     ],
    --     "resources": [
    --       {"resource_desc": "Evacuation map", "mime_type": "image/png", "uri": "https://..."}
    --     ]
    --   }
    -- ]
    raw_xml         TEXT,                                   -- original CAP XML for archival
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, identifier)
);
CREATE INDEX idx_cap_tenant ON cap_alert(tenant_id);
CREATE INDEX idx_cap_sent ON cap_alert(sent);
CREATE INDEX idx_cap_status ON cap_alert(status);
CREATE INDEX idx_cap_info ON cap_alert USING GIN(info_segments);
```

## Situational Awareness Tables

```sql
-- ============================================================
-- SITUATIONAL AWARENESS
-- ============================================================

CREATE TABLE data_feed (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    feed_type       TEXT NOT NULL,
    source_url      TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "polling_interval_seconds": 60,
    --   "auth": {"type": "api_key", "key_header": "X-API-Key"},
    --   "parser": "nws_cap",
    --   "filters": {"severity": ["Extreme", "Severe"]}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_poll_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incident_map_layer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    layer_type      TEXT NOT NULL,                          -- 'marker', 'zone', 'route', 'perimeter'
    name            TEXT NOT NULL,
    geometry        GEOMETRY(Geometry, 4326) NOT NULL,      -- generic geometry type
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (marker):
    -- {"icon": "command_post", "label": "ICP", "description": "Incident Command Post"}
    -- properties example (zone):
    -- {"zone_type": "evacuation", "status": "mandatory", "population": 1200}
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_map_layer_incident ON incident_map_layer(incident_id);
CREATE INDEX idx_map_layer_geometry ON incident_map_layer USING GIST(geometry);

CREATE TABLE situation_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    report_number   INTEGER NOT NULL,
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    content         JSONB NOT NULL,
    -- content example:
    -- {
    --   "situation_summary": "Fire has crossed Hwy 93...",
    --   "weather": {"temp_f": 95, "wind_mph": 30, "rh_pct": 8},
    --   "damage": {"structures_destroyed": 12, "structures_damaged": 8},
    --   "casualties": {"injuries": 2, "fatalities": 0},
    --   "resources_deployed": {"engines": 15, "hand_crews": 4, "aircraft": 2},
    --   "next_steps": ["Request additional mutual aid", "Extend evacuation to Zone C"],
    --   "attachments": [{"name": "aerial_photo.jpg", "url": "https://..."}]
    -- }
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sitrep_incident ON situation_report(incident_id);
```

## Audit & Logging Tables

```sql
-- ============================================================
-- AUDIT TRAIL & CHRONOLOGY
-- ============================================================

CREATE TABLE incident_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    entry_type      TEXT NOT NULL,
    title           TEXT NOT NULL,
    detail          JSONB,
    -- detail example:
    -- {
    --   "action": "resource_deployed",
    --   "resource_id": "uuid",
    --   "resource_name": "Engine 42",
    --   "from_status": "available",
    --   "to_status": "assigned",
    --   "location": {"type": "Point", "coordinates": [-105.23, 39.95]}
    -- }
    logged_by       UUID REFERENCES app_user(id),
    source          TEXT DEFAULT 'user',
    logged_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_log_incident ON incident_log(incident_id);
CREATE INDEX idx_log_time ON incident_log(logged_at);

CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    -- changes example:
    -- {
    --   "old": {"status": "pending", "priority": "routine"},
    --   "new": {"status": "approved", "priority": "immediate"}
    -- }
    ip_address      INET,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant ON audit_trail(tenant_id);
CREATE INDEX idx_audit_entity ON audit_trail(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_trail(performed_at);
```

## After-Action & Cost Tracking

```sql
-- ============================================================
-- AFTER-ACTION & COST TRACKING
-- ============================================================

CREATE TABLE after_action_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "executive_summary": "...",
    --   "timeline": [{"time": "0830", "event": "Incident activated"}, ...],
    --   "strengths": ["Rapid initial response", "Effective mutual aid coordination"],
    --   "improvements": ["Radio interoperability gaps", "Delayed evacuation notification"],
    --   "recommendations": [
    --     {"finding": "Radio interop", "action": "Procure shared TAC channels", "assigned_to": "uuid", "due": "2026-08-01"}
    --   ]
    -- }
    prepared_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cost_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    cost_type       TEXT NOT NULL,
    description     TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    fema_category   TEXT,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "resource_id": "uuid",
    --   "resource_name": "Engine 42",
    --   "hours": 12,
    --   "rate_per_hour": 125.00,
    --   "op_period": 3,
    --   "receipt_url": "https://..."
    -- }
    recorded_by     UUID REFERENCES app_user(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cost_incident ON cost_record(incident_id);
```

---

## JSONB Query Examples

### Find all wildfire incidents where containment is below 50%

```sql
SELECT id, name, details->>'percent_contained' AS containment
FROM incident
WHERE incident_type = 'wildfire'
  AND status = 'active'
  AND (details->>'percent_contained')::numeric < 50
  AND tenant_id = $1;
```

### Search resources by certification

```sql
SELECT r.id, r.name, r.attributes->>'rank' AS rank
FROM resource r
WHERE r.tenant_id = $1
  AND r.status = 'available'
  AND r.attributes @> '{"certifications": ["HAZWOPER"]}';
```

### Find ICS-204 forms with specific division assignments

```sql
SELECT f.id, f.content->>'branch' AS branch, f.content->>'division_group' AS division
FROM ics_form f
WHERE f.incident_id = $1
  AND f.form_type = 'ICS-204'
  AND f.status = 'approved'
  AND f.content @> '{"division_group": "Division Alpha"}';
```

### Query CAP alerts by severity and area

```sql
SELECT id, identifier, info_segments->0->>'headline' AS headline
FROM cap_alert
WHERE tenant_id = $1
  AND status = 'Actual'
  AND info_segments @> '[{"severity": "Extreme"}]'
  AND sent > now() - INTERVAL '24 hours';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 2 | tenant, app_user (roles embedded as array) |
| Organisation & Facilities | 2 | organisation, facility (attributes in JSONB) |
| Incident Management | 2 | incident, operational_period |
| ICS Structure & Forms | 4 | ics_position, ics_assignment, form_schema, ics_form |
| Resource Management | 4 | resource_type, resource, resource_request, resource_deployment |
| Notification & CAP | 3 | notification, notification_recipient, cap_alert |
| Situational Awareness | 3 | data_feed, incident_map_layer, situation_report |
| Audit & Logging | 2 | incident_log, audit_trail |
| After-Action & Costs | 2 | after_action_report, cost_record |
| **Total** | **24** | Significantly fewer tables than normalized approach |

---

## Key Design Decisions

1. **Unified `ics_form` table with JSONB `content`** instead of separate detail tables per form type. One table handles all ICS form types (201-220), with JSON Schema validation in the application layer via `form_schema`. This eliminates the need for DDL migrations when a new form type is added or a jurisdiction needs custom fields.

2. **Incident `details` JSONB column** for type-specific and jurisdiction-specific fields. A wildfire incident has `acres_burned` and `percent_contained`; a flood has `river_gauge_id` and `crest_forecast_ft`. These live in JSONB rather than nullable columns or a polymorphic table hierarchy.

3. **Resource `attributes` JSONB column** for kind-specific metadata. A vehicle has `make`, `model`, `pump_capacity_gpm`; a person has `badge_number`, `rank`, `certifications`. The relational core (`name`, `status`, `resource_type_id`, `current_location`) ensures all resources can be queried uniformly, while attributes capture the details.

4. **CAP info segments as a JSONB array** rather than separate `cap_info`, `cap_area`, `cap_resource` tables. Since CAP alerts are typically queried by their top-level fields (status, msg_type, severity) and rarely JOINed across info segments, the nested JSONB structure is more natural and matches the XML document model. The `raw_xml` column preserves the original CAP message for IPAWS compliance.

5. **GIN indexes on all JSONB columns** enabling containment queries (`@>`) for fast filtering on JSONB fields. This makes queries like "find all resources with HAZWOPER certification" efficient without requiring a dedicated certifications table.

6. **Tenant `config` JSONB** stores per-tenant customisation including custom field definitions, form schema overrides, notification channel preferences, and map defaults. This avoids a proliferation of tenant configuration tables.

7. **24 tables vs. 44+ in the normalised model** -- nearly half the table count. This reduces migration complexity, simplifies the ORM layer, and makes the schema approachable for a small development team building an MVP.

8. **JSON Schema validation at the application layer** ensures that JSONB documents conform to expected structures without relying on database constraints. The `form_schema` table stores versioned JSON Schema definitions that can be updated without DDL changes, and the application validates JSONB content before writes.
