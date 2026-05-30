# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Emergency Management Platform · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form relational design: every concept gets its own table, every relationship is expressed through foreign keys or junction tables, and reference data is extracted into dedicated lookup tables aligned with industry standards (NIMS resource typing, CAP enumerations, ICS form types, ESF categories). The schema is designed to be the "system of record" that external systems integrate against, with strict referential integrity guaranteeing that an ICS assignment always points to a valid resource, a valid incident, and a valid operational period.

The approach mirrors how purpose-built ICS platforms like WebEOC and D4H structure their internal data: board/form-centric entities with position-based role assignments and hierarchical incident command trees. It also aligns with NIEM (National Information Exchange Model) data element naming conventions, making it straightforward to generate NIEM-compliant XML or JSON-LD exports for inter-agency exchange.

This model is best for organisations that value data integrity above all else, need complex cross-entity reporting (e.g., "show me all Type-1 engine resources deployed across three counties in the last 72 hours"), and operate under regulatory environments where audit and compliance queries must be answered from a single, consistent schema.

**Best for:** Government EOCs and agencies requiring strict ICS/NIMS compliance, complex multi-agency reporting, and NIEM-aligned data exchange.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints; complex queries are straightforward SQL joins
- Pro: Standards-aligned reference data (NIMS resource types, CAP enumerations, ESF categories) as first-class lookup tables
- Pro: Schema is self-documenting; new developers can understand relationships from the DDL alone
- Con: High table count (~55-65 tables) increases migration complexity
- Con: Schema changes for new ICS form types or jurisdiction-specific fields require DDL migrations
- Con: Write-heavy operations (e.g., rapid status updates during a major incident) may hit contention on heavily-joined tables
- Con: Multi-tenant isolation requires row-level security or per-tenant schema cloning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIMS / ICS | ICS position hierarchy modelled as recursive `ics_position` table; ICS forms (201-220) as typed `ics_form` entities with form-specific detail tables |
| CAP v1.2 (OASIS) | `cap_alert`, `cap_info`, `cap_resource`, `cap_area` tables mirror the four CAP segments with all required/optional fields |
| NIEM 6.0 | Column naming follows NIEM Emergency Management domain element conventions; export views generate NIEM-compatible JSON-LD |
| EDXL-RM v1.0 | Resource request/response messaging modelled in `resource_request` and `resource_fulfillment` tables |
| EDXL-HAVE | Hospital availability tracked in `facility_status` with bed counts and service capability fields |
| FEMA NIMS Resource Typing | `resource_type_definition` lookup table with Category, Kind, Type, and minimum capabilities |
| ISO 22320:2018 | Incident management command/control/coordination structure reflected in the ICS position and assignment model |
| GeoJSON (RFC 7946) | All geospatial columns use PostGIS `GEOMETRY` types; API layer serialises as GeoJSON |
| ESF Framework | 15 Emergency Support Functions as reference data in `esf_function` table; incidents and positions link to ESFs |
| ISO 3166-1/2 | Jurisdiction codes use ISO 3166 alpha-2 (country) and subdivision codes |

---

## Core Infrastructure Tables

```sql
-- ============================================================
-- TENANCY & AUTHENTICATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,                          -- e.g., "Larimer County OEM"
    slug            TEXT NOT NULL UNIQUE,                   -- URL-safe identifier
    jurisdiction    TEXT,                                   -- ISO 3166-2 code, e.g., "US-CO"
    tier            TEXT NOT NULL DEFAULT 'standard',       -- 'standard', 'enterprise', 'federal'
    settings        JSONB NOT NULL DEFAULT '{}',            -- tenant-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    phone           TEXT,
    password_hash   TEXT,                                   -- null if SSO-only
    auth_provider   TEXT NOT NULL DEFAULT 'local',          -- 'local', 'saml', 'oidc'
    external_id     TEXT,                                   -- IdP subject identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_app_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_app_user_email ON app_user(email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                          -- e.g., 'admin', 'incident_commander', 'field_responder'
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

## Organisation & Agency Tables

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    acronym         TEXT,
    org_type        TEXT NOT NULL,                          -- 'government', 'ngo', 'private', 'volunteer'
    jurisdiction    TEXT,                                   -- ISO 3166-2
    parent_org_id   UUID REFERENCES organisation(id),      -- hierarchical org structure
    contact_email   TEXT,
    contact_phone   TEXT,
    website         TEXT,
    address         JSONB,
    location        GEOMETRY(Point, 4326),                  -- PostGIS point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);
CREATE INDEX idx_organisation_location ON organisation USING GIST(location);

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    facility_type   TEXT NOT NULL,                          -- 'eoc', 'fire_station', 'hospital', 'shelter', 'warehouse'
    address         TEXT,
    location        GEOMETRY(Point, 4326),
    capacity        INTEGER,
    status          TEXT NOT NULL DEFAULT 'operational',    -- 'operational', 'degraded', 'closed'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_facility_tenant ON facility(tenant_id);
CREATE INDEX idx_facility_org ON facility(organisation_id);
CREATE INDEX idx_facility_location ON facility USING GIST(location);
```

## Incident Management Tables

```sql
-- ============================================================
-- INCIDENTS & OPERATIONAL PERIODS
-- ============================================================

CREATE TABLE incident (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    incident_number     TEXT NOT NULL,                      -- agency-assigned number
    name                TEXT NOT NULL,
    incident_type       TEXT NOT NULL,                      -- 'wildfire', 'flood', 'hazmat', 'earthquake', etc.
    status              TEXT NOT NULL DEFAULT 'active',     -- 'active', 'monitoring', 'demobilising', 'closed'
    severity            TEXT NOT NULL DEFAULT 'moderate',   -- 'minor', 'moderate', 'major', 'catastrophic'
    cause               TEXT,
    description         TEXT,
    location_name       TEXT,
    location            GEOMETRY(Point, 4326),
    affected_area       GEOMETRY(Polygon, 4326),
    jurisdiction        TEXT,                               -- ISO 3166-2
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ,
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, incident_number)
);
CREATE INDEX idx_incident_tenant ON incident(tenant_id);
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_location ON incident USING GIST(location);
CREATE INDEX idx_incident_affected_area ON incident USING GIST(affected_area);
CREATE INDEX idx_incident_start ON incident(start_time);

CREATE TABLE operational_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    period_number   INTEGER NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    objectives      TEXT,
    weather_forecast TEXT,
    safety_message  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (incident_id, period_number)
);
CREATE INDEX idx_op_period_incident ON operational_period(incident_id);

-- ESF Reference Data
CREATE TABLE esf_function (
    id              INTEGER PRIMARY KEY,                    -- ESF number (1-15)
    name            TEXT NOT NULL,                          -- e.g., "Transportation"
    description     TEXT,
    lead_agency     TEXT                                    -- e.g., "Department of Transportation"
);

CREATE TABLE incident_esf (
    incident_id     UUID NOT NULL REFERENCES incident(id),
    esf_id          INTEGER NOT NULL REFERENCES esf_function(id),
    lead_org_id     UUID REFERENCES organisation(id),
    activated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (incident_id, esf_id)
);
```

## ICS Command Structure Tables

```sql
-- ============================================================
-- ICS POSITIONS & ASSIGNMENTS
-- ============================================================

CREATE TABLE ics_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,                          -- 'IC', 'OSC', 'PSC', 'LSC', 'FSC'
    title           TEXT NOT NULL,                          -- 'Incident Commander', 'Operations Section Chief'
    section         TEXT NOT NULL,                          -- 'command', 'operations', 'planning', 'logistics', 'finance'
    parent_id       UUID REFERENCES ics_position(id),      -- hierarchical ICS tree
    is_template     BOOLEAN NOT NULL DEFAULT true,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ics_position_tenant ON ics_position(tenant_id);
CREATE INDEX idx_ics_position_parent ON ics_position(parent_id);

CREATE TABLE ics_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID NOT NULL REFERENCES operational_period(id),
    position_id     UUID NOT NULL REFERENCES ics_position(id),
    user_id         UUID REFERENCES app_user(id),
    organisation_id UUID REFERENCES organisation(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ics_assignment_incident ON ics_assignment(incident_id);
CREATE INDEX idx_ics_assignment_user ON ics_assignment(user_id);
CREATE INDEX idx_ics_assignment_period ON ics_assignment(op_period_id);
```

## ICS Forms Tables

```sql
-- ============================================================
-- ICS FORMS (201-220)
-- ============================================================

CREATE TABLE ics_form_type (
    code            TEXT PRIMARY KEY,                       -- 'ICS-201', 'ICS-202', ..., 'ICS-220'
    title           TEXT NOT NULL,
    description     TEXT,
    sections        TEXT[] NOT NULL DEFAULT '{}'            -- section names within the form
);

CREATE TABLE ics_form (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    form_type       TEXT NOT NULL REFERENCES ics_form_type(code),
    status          TEXT NOT NULL DEFAULT 'draft',          -- 'draft', 'review', 'approved', 'superseded'
    prepared_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ics_form_incident ON ics_form(incident_id);
CREATE INDEX idx_ics_form_type ON ics_form(form_type);

-- ICS-201: Incident Briefing
CREATE TABLE ics_201_detail (
    form_id         UUID PRIMARY KEY REFERENCES ics_form(id),
    map_sketch_url  TEXT,                                   -- uploaded map/sketch image
    situation_summary TEXT,
    objectives      TEXT,
    current_actions TEXT,
    planned_actions TEXT
);

-- ICS-201 Resource Summary (section of ICS-201)
CREATE TABLE ics_201_resource_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id         UUID NOT NULL REFERENCES ics_form(id),
    resource_name   TEXT NOT NULL,
    resource_type   TEXT,
    quantity        INTEGER NOT NULL DEFAULT 1,
    eta             TIMESTAMPTZ,
    location        TEXT,
    notes           TEXT
);

-- ICS-202: Incident Objectives
CREATE TABLE ics_202_detail (
    form_id         UUID PRIMARY KEY REFERENCES ics_form(id),
    objectives      TEXT[] NOT NULL DEFAULT '{}',
    weather_forecast TEXT,
    safety_message  TEXT,
    attachments     TEXT[] DEFAULT '{}'                     -- list of attached form codes
);

-- ICS-204: Assignment List
CREATE TABLE ics_204_detail (
    form_id             UUID PRIMARY KEY REFERENCES ics_form(id),
    branch              TEXT,
    division_group      TEXT,
    staging_area        TEXT,
    operations_personnel JSONB,                             -- ops section contacts
    work_assignments    TEXT,
    special_instructions TEXT,
    communications_plan TEXT                                -- reference to ICS-205
);

-- ICS-204 Resource Assignment (sub-table of ICS-204)
CREATE TABLE ics_204_resource (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id         UUID NOT NULL REFERENCES ics_form(id),
    resource_id     UUID REFERENCES resource(id),
    leader_name     TEXT,
    leader_contact  TEXT,
    num_persons     INTEGER,
    reporting_location TEXT,
    reporting_time  TIMESTAMPTZ
);

-- ICS-209: Incident Status Summary
CREATE TABLE ics_209_detail (
    form_id             UUID PRIMARY KEY REFERENCES ics_form(id),
    percent_contained   NUMERIC(5,2),
    area_involved       NUMERIC,                            -- acres / sq km
    area_unit           TEXT DEFAULT 'acres',
    structures_threatened INTEGER,
    structures_damaged  INTEGER,
    structures_destroyed INTEGER,
    injuries            INTEGER DEFAULT 0,
    fatalities          INTEGER DEFAULT 0,
    evacuations         INTEGER DEFAULT 0,
    estimated_cost      NUMERIC(15,2),
    remarks             TEXT
);
```

## Resource Management Tables

```sql
-- ============================================================
-- RESOURCE MANAGEMENT (NIMS Resource Typing)
-- ============================================================

CREATE TABLE resource_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,                   -- NIMS categories: 'fire', 'law_enforcement', 'ems', 'public_works', 'sar'
    description     TEXT
);

CREATE TABLE resource_type_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id     UUID NOT NULL REFERENCES resource_category(id),
    kind            TEXT NOT NULL,                          -- NIMS Kind: 'personnel', 'equipment', 'team', 'supply'
    type_code       TEXT NOT NULL UNIQUE,                   -- e.g., 'ENGINE-TYPE1', 'USAR-TEAM-TYPE1'
    type_name       TEXT NOT NULL,
    type_level      INTEGER NOT NULL,                       -- NIMS Type 1-4 (1 = most capable)
    minimum_capabilities TEXT,
    component_requirements TEXT,
    metric_definitions JSONB,                               -- measurable capability metrics
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_resource_type_category ON resource_type_definition(category_id);

CREATE TABLE resource (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    resource_type_id UUID NOT NULL REFERENCES resource_type_definition(id),
    name            TEXT NOT NULL,
    identifier      TEXT,                                   -- serial number, badge number, etc.
    status          TEXT NOT NULL DEFAULT 'available',      -- 'available', 'assigned', 'out_of_service', 'reserved'
    home_base_id    UUID REFERENCES facility(id),
    current_location GEOMETRY(Point, 4326),
    capabilities    TEXT[],
    certifications  TEXT[],
    contact_info    JSONB,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_resource_tenant ON resource(tenant_id);
CREATE INDEX idx_resource_org ON resource(organisation_id);
CREATE INDEX idx_resource_type ON resource(resource_type_id);
CREATE INDEX idx_resource_status ON resource(status);
CREATE INDEX idx_resource_location ON resource USING GIST(current_location);

-- Resource Request (aligned with EDXL-RM)
CREATE TABLE resource_request (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id         UUID NOT NULL REFERENCES incident(id),
    requesting_org_id   UUID NOT NULL REFERENCES organisation(id),
    requested_by        UUID NOT NULL REFERENCES app_user(id),
    resource_type_id    UUID NOT NULL REFERENCES resource_type_definition(id),
    quantity            INTEGER NOT NULL DEFAULT 1,
    priority            TEXT NOT NULL DEFAULT 'routine',    -- 'immediate', 'priority', 'routine'
    needed_by           TIMESTAMPTZ,
    delivery_location   GEOMETRY(Point, 4326),
    delivery_address    TEXT,
    justification       TEXT,
    status              TEXT NOT NULL DEFAULT 'pending',    -- 'pending', 'approved', 'fulfilled', 'denied', 'cancelled'
    approved_by         UUID REFERENCES app_user(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_resource_request_incident ON resource_request(incident_id);
CREATE INDEX idx_resource_request_status ON resource_request(status);

-- Resource Assignment to Incident
CREATE TABLE resource_deployment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES resource(id),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    request_id      UUID REFERENCES resource_request(id),
    op_period_id    UUID REFERENCES operational_period(id),
    assigned_by     UUID REFERENCES app_user(id),
    deployed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    demobilised_at  TIMESTAMPTZ,
    assignment_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deployment_resource ON resource_deployment(resource_id);
CREATE INDEX idx_deployment_incident ON resource_deployment(incident_id);

-- Mutual Aid Agreement
CREATE TABLE mutual_aid_agreement (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    name                TEXT NOT NULL,
    agreement_type      TEXT NOT NULL,                      -- 'emac', 'intrastate', 'local', 'private'
    party_a_org_id      UUID NOT NULL REFERENCES organisation(id),
    party_b_org_id      UUID NOT NULL REFERENCES organisation(id),
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    resource_types      TEXT[],                             -- which resource types are covered
    terms               TEXT,
    document_url        TEXT,
    status              TEXT NOT NULL DEFAULT 'active',     -- 'draft', 'active', 'expired', 'terminated'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Notification & Alerting Tables

```sql
-- ============================================================
-- MASS NOTIFICATION & CAP ALERTS
-- ============================================================

CREATE TABLE notification_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,                          -- 'weather', 'fire', 'hazmat', 'shelter', 'evacuation'
    subject         TEXT NOT NULL,
    body            TEXT NOT NULL,
    channels        TEXT[] NOT NULL DEFAULT '{sms,email}',  -- 'sms', 'email', 'voice', 'push', 'wea'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_id     UUID REFERENCES incident(id),
    template_id     UUID REFERENCES notification_template(id),
    sender_id       UUID NOT NULL REFERENCES app_user(id),
    subject         TEXT NOT NULL,
    body            TEXT NOT NULL,
    channels        TEXT[] NOT NULL,
    target_type     TEXT NOT NULL,                          -- 'all', 'role', 'group', 'geo_area'
    target_criteria JSONB,                                  -- audience targeting rules
    geo_target      GEOMETRY(Polygon, 4326),                -- geographic targeting area
    status          TEXT NOT NULL DEFAULT 'draft',          -- 'draft', 'sending', 'sent', 'failed'
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notification_tenant ON notification(tenant_id);
CREATE INDEX idx_notification_incident ON notification(incident_id);

CREATE TABLE notification_recipient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL REFERENCES notification(id),
    user_id         UUID REFERENCES app_user(id),
    channel         TEXT NOT NULL,                          -- 'sms', 'email', 'voice', 'push'
    destination     TEXT NOT NULL,                          -- phone number, email, device token
    status          TEXT NOT NULL DEFAULT 'pending',        -- 'pending', 'delivered', 'read', 'failed', 'bounced'
    delivered_at    TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    response        TEXT,                                   -- two-way communication response
    responded_at    TIMESTAMPTZ
);
CREATE INDEX idx_notif_recip_notification ON notification_recipient(notification_id);
CREATE INDEX idx_notif_recip_status ON notification_recipient(status);

-- CAP Alert (Common Alerting Protocol v1.2)
CREATE TABLE cap_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_id     UUID REFERENCES incident(id),
    identifier      TEXT NOT NULL,                          -- CAP: unique message ID
    sender          TEXT NOT NULL,                          -- CAP: sender address
    sent            TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL,                          -- CAP enum: 'Actual', 'Exercise', 'System', 'Test', 'Draft'
    msg_type        TEXT NOT NULL,                          -- CAP enum: 'Alert', 'Update', 'Cancel', 'Ack', 'Error'
    source          TEXT,
    scope           TEXT NOT NULL,                          -- CAP enum: 'Public', 'Restricted', 'Private'
    restriction     TEXT,
    code            TEXT[],
    note            TEXT,
    references      TEXT,                                   -- sender,identifier,sent of referenced alerts
    incidents_ref   TEXT,                                   -- CAP incidents field
    ipaws_cog_id    TEXT,                                   -- IPAWS Collaborative Operating Group
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, identifier)
);
CREATE INDEX idx_cap_alert_tenant ON cap_alert(tenant_id);
CREATE INDEX idx_cap_alert_sent ON cap_alert(sent);

CREATE TABLE cap_info (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id        UUID NOT NULL REFERENCES cap_alert(id) ON DELETE CASCADE,
    language        TEXT NOT NULL DEFAULT 'en-US',
    category        TEXT[] NOT NULL,                        -- CAP enum: 'Geo', 'Met', 'Safety', 'Security', 'Rescue', 'Fire', 'Health', 'Env', 'Transport', 'Infra', 'CBRNE', 'Other'
    event           TEXT NOT NULL,
    response_type   TEXT[],                                 -- 'Shelter', 'Evacuate', 'Prepare', 'Execute', 'Avoid', 'Monitor', 'Assess', 'AllClear', 'None'
    urgency         TEXT NOT NULL,                          -- 'Immediate', 'Expected', 'Future', 'Past', 'Unknown'
    severity        TEXT NOT NULL,                          -- 'Extreme', 'Severe', 'Moderate', 'Minor', 'Unknown'
    certainty       TEXT NOT NULL,                          -- 'Observed', 'Likely', 'Possible', 'Unlikely', 'Unknown'
    audience        TEXT,
    event_code      JSONB,                                  -- {valueName: value} pairs
    effective       TIMESTAMPTZ,
    onset           TIMESTAMPTZ,
    expires         TIMESTAMPTZ,
    sender_name     TEXT,
    headline        TEXT,
    description     TEXT,
    instruction     TEXT,
    web             TEXT,
    contact         TEXT,
    parameter       JSONB                                   -- {valueName: value} pairs
);
CREATE INDEX idx_cap_info_alert ON cap_info(alert_id);

CREATE TABLE cap_area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    info_id         UUID NOT NULL REFERENCES cap_info(id) ON DELETE CASCADE,
    area_desc       TEXT NOT NULL,
    polygon         GEOMETRY(Polygon, 4326),                -- CAP polygon
    circle          JSONB,                                  -- {lat, lon, radius_km}
    geocode         JSONB,                                  -- FIPS, UGC, SAME codes
    altitude        NUMERIC,
    ceiling         NUMERIC
);
CREATE INDEX idx_cap_area_info ON cap_area(info_id);
CREATE INDEX idx_cap_area_polygon ON cap_area USING GIST(polygon);

CREATE TABLE cap_resource (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    info_id         UUID NOT NULL REFERENCES cap_info(id) ON DELETE CASCADE,
    resource_desc   TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    size            BIGINT,
    uri             TEXT,
    deref_uri       TEXT,                                   -- base64 inline content
    digest          TEXT                                    -- SHA-1 hash
);
```

## Situational Awareness & GIS Tables

```sql
-- ============================================================
-- SITUATIONAL AWARENESS & GIS
-- ============================================================

CREATE TABLE data_feed (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    feed_type       TEXT NOT NULL,                          -- 'weather_nws', 'cad_911', 'social_media', 'sensor_iot', 'traffic', 'wfs'
    source_url      TEXT,
    polling_interval_seconds INTEGER DEFAULT 60,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    auth_config     JSONB,                                  -- encrypted credentials
    last_poll_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE situation_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    report_number   INTEGER NOT NULL,
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    situation_summary TEXT NOT NULL,
    weather_conditions TEXT,
    current_threat  TEXT,
    damage_assessment TEXT,
    resource_status TEXT,
    next_steps      TEXT,
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sitrep_incident ON situation_report(incident_id);

CREATE TABLE incident_marker (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    marker_type     TEXT NOT NULL,                          -- 'poi', 'hazard', 'road_closure', 'staging_area', 'command_post', 'helispot'
    label           TEXT NOT NULL,
    description     TEXT,
    location        GEOMETRY(Point, 4326) NOT NULL,
    icon            TEXT,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_marker_incident ON incident_marker(incident_id);
CREATE INDEX idx_marker_location ON incident_marker USING GIST(location);

CREATE TABLE incident_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    zone_type       TEXT NOT NULL,                          -- 'evacuation', 'hot', 'warm', 'cold', 'exclusion', 'shelter_in_place'
    name            TEXT NOT NULL,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    population_estimate INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_zone_incident ON incident_zone(incident_id);
CREATE INDEX idx_zone_boundary ON incident_zone USING GIST(boundary);
```

## Audit & Event Log Tables

```sql
-- ============================================================
-- AUDIT TRAIL & INCIDENT CHRONOLOGY
-- ============================================================

CREATE TABLE incident_log_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    entry_type      TEXT NOT NULL,                          -- 'status_change', 'resource_deployed', 'notification_sent', 'form_approved', 'note', 'communication'
    title           TEXT NOT NULL,
    description     TEXT,
    logged_by       UUID REFERENCES app_user(id),
    source          TEXT DEFAULT 'user',                    -- 'user', 'system', 'feed', 'ai'
    metadata        JSONB,
    logged_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_log_incident ON incident_log_entry(incident_id);
CREATE INDEX idx_log_time ON incident_log_entry(logged_at);
CREATE INDEX idx_log_type ON incident_log_entry(entry_type);

CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,                          -- 'create', 'update', 'delete', 'approve', 'login', 'export'
    entity_type     TEXT NOT NULL,                          -- table name
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant ON audit_trail(tenant_id);
CREATE INDEX idx_audit_entity ON audit_trail(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_trail(performed_at);
CREATE INDEX idx_audit_user ON audit_trail(user_id);
```

## After-Action & Recovery Tables

```sql
-- ============================================================
-- AFTER-ACTION REPORTS & RECOVERY
-- ============================================================

CREATE TABLE after_action_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    title           TEXT NOT NULL,
    executive_summary TEXT,
    timeline_narrative TEXT,
    strengths       TEXT[],
    areas_for_improvement TEXT[],
    recommendations TEXT[],
    prepared_by     UUID REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'draft',
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE improvement_plan_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aar_id          UUID NOT NULL REFERENCES after_action_report(id),
    finding         TEXT NOT NULL,
    recommendation  TEXT NOT NULL,
    assigned_to     UUID REFERENCES app_user(id),
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'open',           -- 'open', 'in_progress', 'completed', 'deferred'
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- FEMA Reimbursement Tracking
CREATE TABLE cost_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    cost_type       TEXT NOT NULL,                          -- 'labor', 'equipment', 'materials', 'contracts', 'other'
    description     TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    resource_id     UUID REFERENCES resource(id),
    op_period_id    UUID REFERENCES operational_period(id),
    fema_category   TEXT,                                   -- PA Category A-G
    documentation_url TEXT,
    recorded_by     UUID REFERENCES app_user(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cost_incident ON cost_record(incident_id);
```

## Facility Status (EDXL-HAVE Aligned)

```sql
-- ============================================================
-- FACILITY STATUS (EDXL-HAVE)
-- ============================================================

CREATE TABLE facility_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    incident_id     UUID REFERENCES incident(id),
    operational_status TEXT NOT NULL,                       -- 'normal', 'limited', 'evacuating', 'closed'
    clinical_status TEXT,                                   -- hospitals: 'normal', 'advisory', 'divert', 'closed'
    beds_available  INTEGER,
    beds_total      INTEGER,
    icu_available   INTEGER,
    ventilators_available INTEGER,
    decon_capability BOOLEAN DEFAULT false,
    helipad         BOOLEAN DEFAULT false,
    notes           TEXT,
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reported_by     UUID REFERENCES app_user(id)
);
CREATE INDEX idx_facility_status_facility ON facility_status(facility_id);
CREATE INDEX idx_facility_status_time ON facility_status(reported_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 4 | tenant, app_user, role, user_role |
| Organisation & Facilities | 2 | organisation, facility |
| Incident Management | 4 | incident, operational_period, esf_function, incident_esf |
| ICS Command Structure | 2 | ics_position, ics_assignment |
| ICS Forms | 9 | ics_form_type, ics_form, and 7 form-specific detail tables |
| Resource Management | 6 | resource_category, resource_type_definition, resource, resource_request, resource_deployment, mutual_aid_agreement |
| Notification & CAP | 7 | notification_template, notification, notification_recipient, cap_alert, cap_info, cap_area, cap_resource |
| Situational Awareness | 4 | data_feed, situation_report, incident_marker, incident_zone |
| Audit & Logging | 2 | incident_log_entry, audit_trail |
| After-Action & Recovery | 3 | after_action_report, improvement_plan_item, cost_record |
| Facility Status | 1 | facility_status (EDXL-HAVE) |
| **Total** | **44** | Plus additional ICS form detail tables as needed |

---

## Key Design Decisions

1. **Separate tables per ICS form type** rather than a single EAV or JSONB forms table. This preserves the ability to write straightforward SQL queries against specific form fields (e.g., `SELECT percent_contained FROM ics_209_detail`) and enables column-level constraints and indexing.

2. **NIMS resource typing as reference data** with a dedicated `resource_type_definition` table keyed by Category, Kind, and Type level. This lets the platform validate resource requests against NIMS standards and generate compliant resource reports.

3. **Full CAP v1.2 segment mapping** (alert -> info -> area, alert -> info -> resource) as separate relational tables rather than storing raw XML. This enables SQL-based alert queries, geospatial area searches via PostGIS, and structured reporting while still allowing XML/JSON serialisation for IPAWS submission.

4. **PostGIS geometry columns** on incidents, resources, facilities, markers, and zones for native geospatial indexing and queries. All geometry uses SRID 4326 (WGS 84) for GeoJSON compatibility.

5. **Row-level multi-tenancy** via `tenant_id` foreign keys rather than schema-per-tenant. This keeps the deployment footprint small while enabling tenant isolation through PostgreSQL Row Level Security policies.

6. **Recursive ICS position hierarchy** using a `parent_id` self-reference, enabling flexible ICS trees that can be queried with recursive CTEs:
   ```sql
   WITH RECURSIVE chain AS (
       SELECT id, title, parent_id, 1 AS depth
       FROM ics_position WHERE code = 'IC' AND tenant_id = $1
       UNION ALL
       SELECT p.id, p.title, p.parent_id, c.depth + 1
       FROM ics_position p JOIN chain c ON p.parent_id = c.id
   )
   SELECT * FROM chain ORDER BY depth;
   ```

7. **EDXL-RM alignment** in the resource request/fulfillment workflow: the `resource_request` table captures the same fields as an EDXL-RM RequestResource message, enabling bidirectional mapping for inter-agency resource messaging.

8. **Audit trail as a separate generic table** rather than embedded in each entity. This centralises compliance queries and supports NIST 800-53 AU (Audit and Accountability) control family requirements.
