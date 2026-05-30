# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Emergency Management Platform · Created: 2026-05-22

## Philosophy

Emergency management is fundamentally about relationships: agencies coordinate with agencies, resources flow between organisations, personnel are assigned to positions that report to other positions, incidents trigger mutual-aid chains across jurisdictions, and alerts cascade through communication networks. A traditional relational model handles one-to-many and many-to-many relationships adequately, but struggles with recursive hierarchies, multi-hop traversals ("which agencies can provide Type-1 engines within 2 mutual-aid hops of this county?"), and dynamic relationship networks that change per-incident.

This model uses a dual-layer architecture: a property graph layer built on PostgreSQL (using `graph_node` and `graph_edge` tables with Apache AGE or ltree/recursive CTEs) handles relationship-intensive queries, while conventional relational tables handle operational CRUD for incidents, resources, notifications, and ICS forms. The graph layer excels at answering questions like:

- "Trace the mutual-aid resource chain for this deployment -- which agencies were involved?"
- "Show all organisations within 3 coordination hops of Larimer County OEM"
- "What is the full ICS command tree for this incident?"
- "Find the shortest resource-sharing path between Agency A and Agency B"
- "Identify all personnel who have worked together across multiple incidents"

This pattern is used by military command-and-control systems (relationship graphs between units, assets, and objectives), intelligence analysis platforms (entity-relationship networks), and supply chain systems (multi-hop logistics networks).

**Best for:** Multi-agency coordination environments, complex mutual-aid networks, resource chain analysis, and platforms that need relationship traversal queries across organisational boundaries.

**Trade-offs:**
- Pro: Multi-hop relationship queries (mutual-aid chains, ICS hierarchies, agency coordination) are natural and fast
- Pro: Dynamic relationship modelling -- edges can be created/removed per-incident without schema changes
- Pro: Powerful for AI-driven analysis: "which personnel have collaborated most effectively?" or "what is the resource dependency graph?"
- Pro: Conflict-of-interest and dual-role detection across agencies
- Pro: ICS command tree is a native graph structure, not a recursive CTE hack
- Con: Graph queries require learning a different query paradigm (Cypher or recursive CTEs)
- Con: Apache AGE or equivalent graph extension adds infrastructure complexity
- Con: Graph edges accumulate over time, requiring pruning or archival strategies
- Con: Two query languages (SQL + Cypher) can confuse developers
- Con: CRUD operations for day-to-day incident management don't benefit from graph structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIMS / ICS | ICS command hierarchy modelled as a directed graph; position-reports-to-position edges enable native tree queries |
| ESF Framework | ESF coordination modelled as edges between organisations and ESF functions; multi-agency ESF activation is a graph query |
| EDXL-RM | Resource request/fulfillment chains modelled as edges; mutual-aid hops are graph traversals |
| CAP v1.2 (OASIS) | Alert dissemination paths modelled as edges from alert to recipients/channels |
| NIEM 6.0 | Inter-agency exchange relationships modelled as edges; NIEM exchange packages map to subgraph exports |
| ISO 22320:2018 | Command-control-coordination structure directly represented as graph relationships |
| GeoJSON (RFC 7946) | Graph nodes carry geospatial attributes; PostGIS for spatial queries on node/edge locations |
| FEMA NIMS Resource Typing | Resource type hierarchy modelled as a directed graph (Category -> Kind -> Type levels) |
| FEMA Mutual Aid Guidelines | Mutual-aid agreement networks directly represented as bidirectional edges between organisations |

---

## Graph Layer Tables

```sql
-- ============================================================
-- GRAPH LAYER (Property Graph on PostgreSQL)
-- ============================================================
-- This can be implemented using:
-- 1. Apache AGE (graph extension for PostgreSQL with Cypher support)
-- 2. Native PostgreSQL with the tables below + recursive CTEs
-- 3. A separate Neo4j instance for graph queries with CDC sync
--
-- The schema below uses native PostgreSQL tables that work without extensions.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,                          -- 'Organisation', 'Person', 'Resource', 'Incident',
                                                            -- 'ICSPosition', 'Facility', 'ESFFunction', 'ResourceType'
    entity_id       UUID NOT NULL,                         -- FK to the corresponding relational table
    label           TEXT NOT NULL,                          -- human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',            -- node-specific properties for graph queries
    location        GEOMETRY(Point, 4326),                  -- optional geospatial position
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);
CREATE INDEX idx_gn_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_label ON graph_node(label);
CREATE INDEX idx_gn_props ON graph_node USING GIN(properties);
CREATE INDEX idx_gn_location ON graph_node USING GIST(location);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL,                          -- relationship type (see Edge Type Catalogue)
    properties      JSONB NOT NULL DEFAULT '{}',            -- edge-specific properties
    weight          NUMERIC DEFAULT 1.0,                    -- for weighted graph algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),     -- temporal validity
    valid_to        TIMESTAMPTZ,                            -- null = currently active
    incident_id     UUID,                                   -- null = permanent; set = incident-specific
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_incident ON graph_edge(incident_id);
CREATE INDEX idx_ge_valid ON graph_edge(valid_from, valid_to);
-- Composite index for common traversal pattern
CREATE INDEX idx_ge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_id, edge_type);
```

## Edge Type Catalogue

```
-- Organisational Relationships
MUTUAL_AID_PARTNER       Organisation -> Organisation    (mutual aid agreement)
PARENT_ORG               Organisation -> Organisation    (org hierarchy)
MEMBER_OF                Person -> Organisation           (employment/membership)
OPERATES                 Organisation -> Facility         (facility ownership)

-- ICS Command Relationships (per-incident)
REPORTS_TO               ICSPosition -> ICSPosition       (command hierarchy)
ASSIGNED_TO              Person -> ICSPosition            (position assignment)
COMMANDS                 ICSPosition -> Resource           (resource under command)

-- Resource Relationships
DEPLOYED_TO              Resource -> Incident             (resource deployment)
BASED_AT                 Resource -> Facility             (home base)
REQUESTED_BY             Resource -> Organisation         (resource request chain)
PROVIDED_BY              Resource -> Organisation         (mutual aid provision)
HAS_TYPE                 Resource -> ResourceType         (resource typing)

-- Incident Relationships
ACTIVATED_ESF            Incident -> ESFFunction          (ESF activation)
RESPONDING_TO            Organisation -> Incident         (agency response)
COORDINATES_WITH         Organisation -> Organisation     (per-incident coordination)

-- Alert Relationships
ALERT_TARGETS            Incident -> Person               (notification path)
ORIGINATED_ALERT         Incident -> CAP Alert            (alert origination)
```

---

## Relational Layer Tables

The relational layer handles operational CRUD. Each relational entity has a corresponding `graph_node` entry maintained via triggers or application-layer sync.

```sql
-- ============================================================
-- TENANCY & USERS
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    jurisdiction    TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    phone           TEXT,
    roles           TEXT[] NOT NULL DEFAULT '{}',
    skills          TEXT[] NOT NULL DEFAULT '{}',
    certifications  TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_skills ON app_user USING GIN(skills);
CREATE INDEX idx_user_certs ON app_user USING GIN(certifications);

-- ============================================================
-- ORGANISATIONS & FACILITIES
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    acronym         TEXT,
    org_type        TEXT NOT NULL,
    jurisdiction    TEXT,
    location        GEOMETRY(Point, 4326),
    contact         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_tenant ON organisation(tenant_id);
CREATE INDEX idx_org_location ON organisation USING GIST(location);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fac_org ON facility(organisation_id);
CREATE INDEX idx_fac_location ON facility USING GIST(location);

-- ============================================================
-- MUTUAL AID AGREEMENTS (Relational + Graph Edge)
-- ============================================================

CREATE TABLE mutual_aid_agreement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    agreement_type  TEXT NOT NULL,                          -- 'emac', 'intrastate', 'local', 'private'
    party_a_org_id  UUID NOT NULL REFERENCES organisation(id),
    party_b_org_id  UUID NOT NULL REFERENCES organisation(id),
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    resource_types  TEXT[],
    terms           TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Corresponding graph edges are created/updated via trigger:
-- INSERT INTO graph_edge (source_id, target_id, edge_type, properties)
-- VALUES (node_for(party_a_org_id), node_for(party_b_org_id), 'MUTUAL_AID_PARTNER',
--         jsonb_build_object('agreement_id', NEW.id, 'type', NEW.agreement_type))

-- ============================================================
-- INCIDENTS
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
    location        GEOMETRY(Point, 4326),
    affected_area   GEOMETRY(Polygon, 4326),
    jurisdiction    TEXT,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, incident_number)
);
CREATE INDEX idx_inc_tenant ON incident(tenant_id);
CREATE INDEX idx_inc_status ON incident(status);
CREATE INDEX idx_inc_location ON incident USING GIST(location);
CREATE INDEX idx_inc_area ON incident USING GIST(affected_area);

CREATE TABLE operational_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    period_number   INTEGER NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    objectives      TEXT[],
    weather         JSONB,
    safety_message  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (incident_id, period_number)
);

-- ============================================================
-- ICS POSITIONS & ASSIGNMENTS (Graph-Native)
-- ============================================================
-- ICS positions and their hierarchy live primarily in the graph layer.
-- The relational table below is the source of truth for position definitions;
-- the graph edges (REPORTS_TO, ASSIGNED_TO) represent the hierarchy.

CREATE TABLE ics_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    title           TEXT NOT NULL,
    section         TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- NOTE: parent_id is NOT in this table -- hierarchy is modelled as REPORTS_TO edges in graph_edge

CREATE TABLE ics_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    position_id     UUID NOT NULL REFERENCES ics_position(id),
    user_id         UUID REFERENCES app_user(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at     TIMESTAMPTZ
);
CREATE INDEX idx_assign_incident ON ics_assignment(incident_id);
-- Triggers maintain ASSIGNED_TO graph edges

-- ============================================================
-- ICS FORMS
-- ============================================================

CREATE TABLE ics_form (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    op_period_id    UUID REFERENCES operational_period(id),
    form_type       TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'draft',
    content         JSONB NOT NULL DEFAULT '{}',
    prepared_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_form_incident ON ics_form(incident_id);
CREATE INDEX idx_form_type ON ics_form(form_type);

-- ============================================================
-- RESOURCES
-- ============================================================

CREATE TABLE resource_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        TEXT NOT NULL,
    kind            TEXT NOT NULL,
    type_code       TEXT NOT NULL UNIQUE,
    type_name       TEXT NOT NULL,
    type_level      INTEGER NOT NULL,
    capabilities    JSONB NOT NULL DEFAULT '{}',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_res_tenant ON resource(tenant_id);
CREATE INDEX idx_res_status ON resource(status);
CREATE INDEX idx_res_location ON resource USING GIST(current_location);

CREATE TABLE resource_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    requesting_org_id UUID NOT NULL REFERENCES organisation(id),
    requested_by    UUID NOT NULL REFERENCES app_user(id),
    resource_type_id UUID NOT NULL REFERENCES resource_type(id),
    quantity        INTEGER NOT NULL DEFAULT 1,
    priority        TEXT NOT NULL DEFAULT 'routine',
    status          TEXT NOT NULL DEFAULT 'pending',
    fulfillment     JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_req_incident ON resource_request(incident_id);
-- Resource request/fulfillment creates graph edges:
-- REQUESTED_BY: Resource -> requesting Organisation
-- PROVIDED_BY:  Resource -> providing Organisation (on fulfillment)

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
    status          TEXT NOT NULL DEFAULT 'draft',
    stats           JSONB NOT NULL DEFAULT '{}',
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cap_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    incident_id     UUID REFERENCES incident(id),
    identifier      TEXT NOT NULL,
    sender          TEXT NOT NULL,
    sent            TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL,
    msg_type        TEXT NOT NULL,
    scope           TEXT NOT NULL,
    info_segments   JSONB NOT NULL DEFAULT '[]',
    raw_xml         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, identifier)
);
CREATE INDEX idx_cap_tenant ON cap_alert(tenant_id);

-- ============================================================
-- SITUATIONAL AWARENESS
-- ============================================================

CREATE TABLE incident_map_layer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    layer_type      TEXT NOT NULL,
    name            TEXT NOT NULL,
    geometry        GEOMETRY(Geometry, 4326) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_map_incident ON incident_map_layer(incident_id);
CREATE INDEX idx_map_geom ON incident_map_layer USING GIST(geometry);

-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_trail(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_trail(performed_at);

CREATE TABLE incident_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    entry_type      TEXT NOT NULL,
    title           TEXT NOT NULL,
    detail          JSONB,
    logged_by       UUID REFERENCES app_user(id),
    logged_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ilog_incident ON incident_log(incident_id);
CREATE INDEX idx_ilog_time ON incident_log(logged_at);
```

---

## Graph Query Examples

### Traverse the ICS Command Tree for an Incident

```sql
-- Find the full ICS command hierarchy from Incident Commander down
WITH RECURSIVE command_tree AS (
    -- Start from the Incident Commander node
    SELECT
        gn.id AS node_id,
        gn.label AS position,
        gn.properties->>'section' AS section,
        0 AS depth,
        ARRAY[gn.label] AS path
    FROM graph_node gn
    JOIN graph_edge ge ON ge.target_id = gn.id
    JOIN graph_node inc ON inc.id = ge.source_id
    WHERE inc.node_type = 'Incident'
      AND inc.entity_id = $incident_id
      AND gn.node_type = 'ICSPosition'
      AND gn.properties->>'code' = 'IC'
      AND ge.edge_type = 'COMMANDS'
      AND ge.valid_to IS NULL

    UNION ALL

    -- Traverse REPORTS_TO edges downward
    SELECT
        child.id,
        child.label,
        child.properties->>'section',
        ct.depth + 1,
        ct.path || child.label
    FROM command_tree ct
    JOIN graph_edge ge ON ge.target_id = ct.node_id
    JOIN graph_node child ON child.id = ge.source_id
    WHERE ge.edge_type = 'REPORTS_TO'
      AND ge.incident_id = $incident_id
      AND ge.valid_to IS NULL
)
SELECT * FROM command_tree ORDER BY depth, position;
```

### Find Mutual-Aid Resource Availability Within N Hops

```sql
-- Which agencies can provide Type-1 engines within 2 mutual-aid hops?
WITH RECURSIVE aid_network AS (
    -- Start from the requesting agency
    SELECT
        gn.id AS node_id,
        gn.entity_id AS org_id,
        gn.label AS org_name,
        0 AS hops,
        ARRAY[gn.entity_id] AS visited
    FROM graph_node gn
    WHERE gn.node_type = 'Organisation'
      AND gn.entity_id = $requesting_org_id

    UNION ALL

    -- Traverse MUTUAL_AID_PARTNER edges
    SELECT
        partner.id,
        partner.entity_id,
        partner.label,
        an.hops + 1,
        an.visited || partner.entity_id
    FROM aid_network an
    JOIN graph_edge ge ON (ge.source_id = an.node_id OR ge.target_id = an.node_id)
    JOIN graph_node partner ON partner.id = CASE
        WHEN ge.source_id = an.node_id THEN ge.target_id
        ELSE ge.source_id
    END
    WHERE ge.edge_type = 'MUTUAL_AID_PARTNER'
      AND ge.valid_to IS NULL
      AND an.hops < 2
      AND partner.entity_id != ALL(an.visited)
)
SELECT DISTINCT
    an.org_id,
    an.org_name,
    an.hops,
    COUNT(r.id) AS available_engines
FROM aid_network an
JOIN resource r ON r.organisation_id = an.org_id
JOIN resource_type rt ON rt.id = r.resource_type_id
WHERE rt.type_code = 'ENGINE-TYPE1'
  AND r.status = 'available'
  AND an.hops > 0  -- exclude the requesting org itself
GROUP BY an.org_id, an.org_name, an.hops
ORDER BY an.hops, available_engines DESC;
```

### Personnel Collaboration Network

```sql
-- Find personnel who have worked together across multiple incidents
-- (useful for team building and identifying experienced partnerships)
SELECT
    p1.label AS person_a,
    p2.label AS person_b,
    COUNT(DISTINCT e1.incident_id) AS shared_incidents,
    array_agg(DISTINCT inc.label) AS incident_names
FROM graph_edge e1
JOIN graph_edge e2 ON e1.incident_id = e2.incident_id
    AND e1.edge_type = 'ASSIGNED_TO'
    AND e2.edge_type = 'ASSIGNED_TO'
    AND e1.source_id < e2.source_id  -- avoid duplicates
JOIN graph_node p1 ON p1.id = e1.source_id AND p1.node_type = 'Person'
JOIN graph_node p2 ON p2.id = e2.source_id AND p2.node_type = 'Person'
JOIN graph_node inc ON inc.node_type = 'Incident'
    AND inc.entity_id = e1.incident_id
WHERE e1.tenant_id = $tenant_id
GROUP BY p1.label, p2.label
HAVING COUNT(DISTINCT e1.incident_id) >= 3
ORDER BY shared_incidents DESC;
```

### Resource Deployment Chain Analysis

```sql
-- Trace the full resource flow for an incident:
-- which organisations provided what resources via which agreements?
SELECT
    provider.label AS providing_agency,
    resource.label AS resource_name,
    resource.properties->>'type_code' AS resource_type,
    ge_prov.properties->>'agreement_id' AS mutual_aid_agreement,
    ge_deploy.properties->>'deployed_at' AS deployed_at
FROM graph_edge ge_deploy
JOIN graph_node resource ON resource.id = ge_deploy.source_id
    AND resource.node_type = 'Resource'
JOIN graph_node inc ON inc.id = ge_deploy.target_id
    AND inc.node_type = 'Incident'
JOIN graph_edge ge_prov ON ge_prov.source_id = resource.id
    AND ge_prov.edge_type = 'PROVIDED_BY'
    AND ge_prov.incident_id = $incident_id
JOIN graph_node provider ON provider.id = ge_prov.target_id
WHERE ge_deploy.edge_type = 'DEPLOYED_TO'
  AND inc.entity_id = $incident_id
ORDER BY ge_deploy.properties->>'deployed_at';
```

### Temporal Relationship Reconstruction

```sql
-- What was the ICS command structure at a specific point in time?
SELECT
    pos.label AS position,
    person.label AS assigned_person,
    ge.valid_from AS assigned_at
FROM graph_edge ge
JOIN graph_node person ON person.id = ge.source_id AND person.node_type = 'Person'
JOIN graph_node pos ON pos.id = ge.target_id AND pos.node_type = 'ICSPosition'
WHERE ge.edge_type = 'ASSIGNED_TO'
  AND ge.incident_id = $incident_id
  AND ge.valid_from <= '2026-05-22T14:00:00Z'
  AND (ge.valid_to IS NULL OR ge.valid_to > '2026-05-22T14:00:00Z')
ORDER BY pos.properties->>'sort_order';
```

---

## Graph Synchronisation

The graph layer is kept in sync with the relational layer via PostgreSQL triggers:

```sql
-- Example trigger: sync organisation creation to graph_node
CREATE OR REPLACE FUNCTION sync_org_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, location, properties)
    VALUES (
        NEW.tenant_id,
        'Organisation',
        NEW.id,
        NEW.name,
        NEW.location,
        jsonb_build_object(
            'acronym', NEW.acronym,
            'org_type', NEW.org_type,
            'jurisdiction', NEW.jurisdiction
        )
    )
    ON CONFLICT (tenant_id, node_type, entity_id)
    DO UPDATE SET
        label = EXCLUDED.label,
        location = EXCLUDED.location,
        properties = EXCLUDED.properties,
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_org_graph_sync
    AFTER INSERT OR UPDATE ON organisation
    FOR EACH ROW EXECUTE FUNCTION sync_org_to_graph();

-- Example trigger: sync mutual aid agreement to graph edges
CREATE OR REPLACE FUNCTION sync_mutual_aid_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    source_node_id UUID;
    target_node_id UUID;
BEGIN
    SELECT id INTO source_node_id FROM graph_node
    WHERE node_type = 'Organisation' AND entity_id = NEW.party_a_org_id;

    SELECT id INTO target_node_id FROM graph_node
    WHERE node_type = 'Organisation' AND entity_id = NEW.party_b_org_id;

    -- Bidirectional edge for mutual aid
    INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, properties, valid_from, valid_to)
    VALUES (
        NEW.tenant_id, source_node_id, target_node_id, 'MUTUAL_AID_PARTNER',
        jsonb_build_object('agreement_id', NEW.id, 'type', NEW.agreement_type, 'resource_types', NEW.resource_types),
        NEW.effective_date, NEW.expiration_date
    );
    INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, properties, valid_from, valid_to)
    VALUES (
        NEW.tenant_id, target_node_id, source_node_id, 'MUTUAL_AID_PARTNER',
        jsonb_build_object('agreement_id', NEW.id, 'type', NEW.agreement_type, 'resource_types', NEW.resource_types),
        NEW.effective_date, NEW.expiration_date
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_mutual_aid_graph_sync
    AFTER INSERT ON mutual_aid_agreement
    FOR EACH ROW EXECUTE FUNCTION sync_mutual_aid_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (the relationship engine) |
| Tenancy & Users | 2 | tenant, app_user |
| Organisation & Facilities | 3 | organisation, facility, mutual_aid_agreement |
| Incident Management | 2 | incident, operational_period |
| ICS Structure & Forms | 3 | ics_position, ics_assignment, ics_form |
| Resource Management | 3 | resource_type, resource, resource_request |
| Notification & CAP | 2 | notification, cap_alert |
| Situational Awareness | 1 | incident_map_layer |
| Audit & Logging | 2 | audit_trail, incident_log |
| **Total** | **20** | Relational tables + 2 graph tables handle all relationships |

---

## Key Design Decisions

1. **Property graph on PostgreSQL** rather than a dedicated graph database (Neo4j). This keeps the entire system in one database, avoids cross-database consistency issues, and leverages PostgreSQL's mature ecosystem (PostGIS, JSONB, partitioning, RLS). For deployments needing maximum graph performance, the graph layer can be migrated to Neo4j or Apache AGE without changing the relational tables.

2. **ICS hierarchy as graph edges** (`REPORTS_TO`) rather than `parent_id` columns. This means the ICS command tree is a first-class graph query rather than a recursive CTE hack. The hierarchy can vary per incident without changing the position definitions, and temporal `valid_from`/`valid_to` fields enable reconstructing the command tree at any point in time.

3. **Mutual-aid networks as bidirectional edges** between Organisation nodes. Multi-hop traversal ("which agencies can help within 2 hops?") is a natural graph query that would be extremely difficult with relational joins across a many-to-many junction table.

4. **Temporal edge validity** (`valid_from`, `valid_to`) enables historical graph queries: "who was assigned where at 14:00?" or "what was the mutual-aid network before the agreement expired?" This is essential for after-action reviews and compliance audits.

5. **Incident-scoped edges** via the `incident_id` column on `graph_edge`. Edges like `ASSIGNED_TO`, `DEPLOYED_TO`, and `COORDINATES_WITH` are per-incident, while edges like `MUTUAL_AID_PARTNER` and `MEMBER_OF` are permanent (null `incident_id`). This prevents incident-specific relationships from polluting the standing organisational graph.

6. **Trigger-based sync** between relational tables and graph nodes/edges. The relational tables remain the system of record for CRUD operations; the graph layer is a derived view optimised for relationship queries. This means standard REST API operations work against relational tables, while graph queries are available for analytics, visualisation, and AI-powered analysis.

7. **Only 20 tables total** -- the graph layer absorbs the complexity that would otherwise require junction tables, hierarchy tables, and relationship-specific tables. The `graph_edge` table replaces what would be 8-10 separate junction/relationship tables in a normalised model.

8. **AI-native graph analysis**: the graph structure is ideal for LLM-powered queries like "identify the most connected agencies in this response network" or "suggest optimal resource sharing paths based on historical collaboration patterns." Graph embeddings can be generated from the node/edge structure for recommendation and prediction models.
