# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: OKR & Goal Tracking Platform · Created: 2026-05-24

## Philosophy

An OKR platform tracks how goals evolve over a planning cycle: objectives are drafted, revised, and finalized during planning; key results receive check-ins that shift their progress; confidence changes from on-track to at-risk; alignment relationships shift as strategy pivots. An event-sourced model captures every goal creation, every check-in, every status change, every alignment adjustment, and every AI analysis as an immutable event. The event store is the single source of truth; read models are materialised projections optimised for dashboards, alignment trees, and progress charts.

This is particularly powerful for an OKR platform because:
- **OKR progress is inherently temporal** — "How has this objective's confidence changed over the quarter?" is answered by replaying check-in and status-change events. Event sourcing makes temporal queries first-class.
- **Alignment changes need audit trails** — When a team objective is re-aligned from one company objective to another mid-quarter, the event stream captures why and when. This is critical for quarterly retrospectives.
- **Check-ins are naturally events** — Each weekly check-in is a discrete signal: "key result X moved from 15 to 22 with confidence 'at risk'." Event sourcing stores these as immutable observations.
- **AI coaching benefits from full history** — AI risk prediction and narrative generation work best when they can replay the full sequence of events (goal creation → early check-ins → status changes → alignment shifts) rather than reading current state.
- **Quarterly retrospectives need time-travel** — "What was the state of all company OKRs on May 1st?" is answered by event replay, not by querying current mutable state.

The result is an event store with 25+ event types covering OKR lifecycle, check-ins, alignment, AI analysis, and integration sync, plus 5 read model tables and 8 reference tables.

**Best for:** Teams building an OKR platform where quarterly retrospectives require temporal queries, where alignment change auditing matters for leadership visibility, and where AI features need full event history for coaching and narrative generation.

**Trade-offs:**
- **Pro:** Complete history — every check-in, status change, and alignment shift is immutable
- **Pro:** Temporal queries for quarterly retrospectives and time-travel
- **Pro:** AI coaching from full event replay rather than current-state snapshots
- **Pro:** Alignment change audit trail for leadership visibility
- **Con:** Eventual consistency between event store and dashboards
- **Con:** Alignment tree requires read model; not a direct recursive query
- **Con:** Event schema versioning adds operational complexity
- **Con:** More infrastructure than a CRUD model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope fields (ce_source, ce_type, ce_specversion) |
| OKR Framework (Doerr/Grove) | Event types map to OKR lifecycle stages |
| SCIM 2.0 | User provisioning events |
| ISO 30414 | Goal metrics in event payloads align with human capital reporting |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| GDPR | Data export and erasure events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL CHECK (stream_type IN (
        'objective', 'key_result', 'cycle', 'team',
        'integration', 'user'
    )),
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_version INT NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_id": "user-uuid", "actor_type": "user|system|integration|ai",
    --  "ce_source": "/okr/objectives", "ce_specversion": "1.0",
    --  "correlation_id": "uuid", "causation_id": "uuid"}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_h1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_h2 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_occurred ON event_store(occurred_at);
```

---

## Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1
);

-- Objective lifecycle
INSERT INTO event_type_registry VALUES
    ('objective.created', 'objective', 'Objective created', 1),
    ('objective.updated', 'objective', 'Objective title/description changed', 1),
    ('objective.status_changed', 'objective', 'Objective status changed (on_track, at_risk, etc)', 1),
    ('objective.aligned', 'objective', 'Objective aligned to parent objective', 1),
    ('objective.realigned', 'objective', 'Objective re-aligned to different parent', 1),
    ('objective.completed', 'objective', 'Objective marked completed', 1),
    ('objective.cancelled', 'objective', 'Objective cancelled', 1),
    ('objective.commented', 'objective', 'Comment added to objective', 1),
    ('objective.ai_analyzed', 'objective', 'AI quality/risk analysis completed', 1),

    -- Key result lifecycle
    ('key_result.created', 'key_result', 'Key result created under objective', 1),
    ('key_result.updated', 'key_result', 'Key result title/target changed', 1),
    ('key_result.checked_in', 'key_result', 'Check-in recorded with new value', 1),
    ('key_result.status_changed', 'key_result', 'Key result status changed', 1),
    ('key_result.confidence_changed', 'key_result', 'Confidence rating changed', 1),
    ('key_result.completed', 'key_result', 'Key result reached target', 1),
    ('key_result.integration_synced', 'key_result', 'Value updated from integration', 1),
    ('key_result.initiative_added', 'key_result', 'Initiative linked to key result', 1),
    ('key_result.initiative_completed', 'key_result', 'Linked initiative completed', 1),
    ('key_result.commented', 'key_result', 'Comment added to key result', 1),
    ('key_result.ai_analyzed', 'key_result', 'AI analysis completed', 1),

    -- Cycle lifecycle
    ('cycle.created', 'cycle', 'Planning cycle created', 1),
    ('cycle.activated', 'cycle', 'Cycle set as active', 1),
    ('cycle.completed', 'cycle', 'Cycle ended', 1),
    ('cycle.narrative_generated', 'cycle', 'AI narrative generated for cycle', 1),

    -- Team events
    ('team.created', 'team', 'Team created', 1),
    ('team.member_added', 'team', 'Member joined team', 1),
    ('team.member_removed', 'team', 'Member left team', 1),

    -- Integration events
    ('integration.connected', 'integration', 'Data source connected', 1),
    ('integration.synced', 'integration', 'Data source sync completed', 1),
    ('integration.sync_failed', 'integration', 'Data source sync failed', 1),
    ('integration.disconnected', 'integration', 'Data source disconnected', 1);
```

---

## Event Payload Examples

```sql
-- objective.created
-- {"title": "Become the market leader in SMB segment",
--  "description": "Expand SMB market share from 15% to 30%",
--  "level": "company", "cycle_id": "uuid",
--  "owner_id": "uuid", "team_id": null,
--  "is_stretch": true}

-- objective.realigned
-- {"previous_parent_id": "uuid-old-company-obj",
--  "new_parent_id": "uuid-new-company-obj",
--  "reason": "Strategy pivot: focusing on enterprise instead of SMB"}

-- key_result.checked_in
-- {"previous_value": 15, "new_value": 22,
--  "start_value": 0, "target_value": 50,
--  "progress": 0.44, "previous_progress": 0.30,
--  "confidence": "on_track", "previous_confidence": "on_track",
--  "note": "Closed 7 new accounts this week",
--  "source": "manual"}

-- key_result.integration_synced
-- {"previous_value": 22, "new_value": 28,
--  "progress": 0.56,
--  "provider": "salesforce", "data_source_id": "uuid",
--  "query": "SELECT COUNT(Id) FROM Opportunity WHERE StageName = 'Closed Won'",
--  "raw_response": {"totalSize": 28, "done": true}}

-- objective.ai_analyzed
-- {"quality_score": 0.85,
--  "quality_feedback": ["Objective is specific and inspiring",
--                       "Consider adding a time-bound element"],
--  "risk_score": 0.3,
--  "risk_factors": ["2 of 3 KRs at risk", "no check-ins in 14 days"],
--  "dependencies": [{"objective_id": "uuid", "team": "Platform",
--                     "risk": "API migration blocks integration goal"}],
--  "model_version": "v2.1"}

-- cycle.narrative_generated
-- {"narrative_type": "executive_summary",
--  "content": "Q2 progress: 18 of 24 objectives on track. Revenue objective...",
--  "scope_team_id": null,
--  "model_version": "claude-sonnet-4-6",
--  "metrics": {"total_objectives": 24, "on_track": 18, "at_risk": 4, "off_track": 2,
--              "avg_progress": 0.72}}
```

---

## Stream Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    snapshot_version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    timezone TEXT NOT NULL DEFAULT 'UTC',
    scim_external_id TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    fiscal_year_start_month INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    parent_team_id UUID REFERENCES teams(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    UNIQUE (org_id, slug)
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider TEXT NOT NULL,
    name TEXT NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'connected',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, provider, name)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_objectives (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES organisations(id),
    cycle_id UUID NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id),
    owner_name TEXT NOT NULL,
    team_id UUID,
    team_name TEXT,
    parent_objective_id UUID,
    title TEXT NOT NULL,
    description TEXT,
    level TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'on_track',
    progress REAL NOT NULL DEFAULT 0.0,
    is_stretch BOOLEAN NOT NULL DEFAULT FALSE,
    key_result_count INT NOT NULL DEFAULT 0,
    ai JSONB NOT NULL DEFAULT '{}',
    comments JSONB NOT NULL DEFAULT '[]',
    tags TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_obj_org ON rm_objectives(org_id, cycle_id);
CREATE INDEX idx_rm_obj_owner ON rm_objectives(owner_id, cycle_id);
CREATE INDEX idx_rm_obj_team ON rm_objectives(team_id, cycle_id);
CREATE INDEX idx_rm_obj_parent ON rm_objectives(parent_objective_id);
CREATE INDEX idx_rm_obj_status ON rm_objectives(org_id, status)
    WHERE status NOT IN ('completed', 'cancelled');

CREATE TABLE rm_key_results (
    id UUID PRIMARY KEY,
    objective_id UUID NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id),
    owner_name TEXT NOT NULL,
    title TEXT NOT NULL,
    metric_type TEXT NOT NULL,
    unit TEXT,
    start_value REAL NOT NULL DEFAULT 0,
    current_value REAL NOT NULL DEFAULT 0,
    target_value REAL NOT NULL,
    progress REAL NOT NULL DEFAULT 0.0,
    status TEXT NOT NULL DEFAULT 'on_track',
    confidence REAL,
    check_in_count INT NOT NULL DEFAULT 0,
    last_check_in_at TIMESTAMPTZ,
    integration_provider TEXT,
    ai JSONB NOT NULL DEFAULT '{}',
    initiatives JSONB NOT NULL DEFAULT '[]',
    comments JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_kr_obj ON rm_key_results(objective_id);
CREATE INDEX idx_rm_kr_owner ON rm_key_results(owner_id);

CREATE TABLE rm_check_in_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id UUID NOT NULL,
    value REAL NOT NULL,
    previous_value REAL NOT NULL,
    progress REAL NOT NULL,
    confidence TEXT NOT NULL,
    note TEXT,
    source TEXT NOT NULL,
    actor_id UUID,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_checkins ON rm_check_in_history(key_result_id, created_at DESC);

CREATE TABLE rm_cycles (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES organisations(id),
    name TEXT NOT NULL,
    cycle_type TEXT NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    stats JSONB NOT NULL DEFAULT '{}',
    narratives JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_cycles ON rm_cycles(org_id);

CREATE TABLE rm_alignment_map (
    objective_id UUID NOT NULL,
    parent_objective_id UUID,
    org_id UUID NOT NULL,
    cycle_id UUID NOT NULL,
    level TEXT NOT NULL,
    title TEXT NOT NULL,
    status TEXT NOT NULL,
    progress REAL NOT NULL,
    owner_name TEXT NOT NULL,
    team_name TEXT,
    key_result_summary JSONB NOT NULL DEFAULT '[]',
    -- [{"title": "Close 50 new accounts", "progress": 0.44, "status": "on_track"}]
    PRIMARY KEY (objective_id)
);

CREATE INDEX idx_rm_alignment ON rm_alignment_map(org_id, cycle_id, parent_objective_id);
```

---

## Example Queries

### Replay objective lifecycle

```sql
SELECT event_type, payload, metadata->>'actor_type' AS actor,
       occurred_at
FROM event_store
WHERE stream_type = 'objective'
  AND stream_id = 'objective-uuid'
ORDER BY event_version;
```

### Check-in trend for a key result

```sql
SELECT (payload->>'new_value')::REAL AS value,
       payload->>'confidence' AS confidence,
       payload->>'note' AS note,
       payload->>'source' AS source,
       occurred_at
FROM event_store
WHERE stream_type = 'key_result'
  AND stream_id = 'kr-uuid'
  AND event_type = 'key_result.checked_in'
ORDER BY occurred_at;
```

### Alignment changes during a quarter

```sql
SELECT payload->>'previous_parent_id' AS from_parent,
       payload->>'new_parent_id' AS to_parent,
       payload->>'reason' AS reason,
       metadata->>'actor_id' AS changed_by,
       occurred_at
FROM event_store
WHERE event_type = 'objective.realigned'
  AND occurred_at >= '2026-04-01' AND occurred_at < '2026-07-01'
ORDER BY occurred_at;
```

### Point-in-time state reconstruction (May 1st snapshot)

```sql
SELECT stream_id, event_type, payload
FROM event_store
WHERE stream_type = 'objective'
  AND occurred_at <= '2026-05-01'
ORDER BY stream_id, event_version;
-- Application replays events per stream to reconstruct May 1st state
```

### AI analysis trend

```sql
SELECT (payload->>'risk_score')::REAL AS risk_score,
       (payload->>'quality_score')::REAL AS quality_score,
       payload->>'model_version' AS model,
       occurred_at
FROM event_store
WHERE stream_type = 'objective'
  AND stream_id = 'objective-uuid'
  AND event_type = 'objective.ai_analyzed'
ORDER BY occurred_at;
```

### Integration sync health

```sql
SELECT payload->>'provider' AS provider,
       COUNT(*) FILTER (WHERE event_type = 'integration.synced') AS success_count,
       COUNT(*) FILTER (WHERE event_type = 'integration.sync_failed') AS failure_count,
       MAX(occurred_at) AS last_sync
FROM event_store
WHERE stream_type = 'integration'
  AND occurred_at >= now() - INTERVAL '7 days'
GROUP BY payload->>'provider';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_type_registry, stream_snapshots, projection_checkpoints |
| Reference Data | 6 | users, organisations, teams, team_members, data_sources, api_keys |
| Read Models | 5 | rm_objectives, rm_key_results, rm_check_in_history, rm_cycles, rm_alignment_map |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Check-ins as events** — Every `key_result.checked_in` event captures the previous value, new value, progress delta, confidence, note, and source. These events are the training data for AI progress forecasting and the source for check-in trend charts. The `rm_check_in_history` read model stores the flattened history for efficient queries.

2. **Alignment changes as events** — `objective.aligned` and `objective.realigned` events capture when and why an objective's parent changed. This enables quarterly retrospective queries like "which team objectives were re-aligned this quarter, and why?" — information lost in a mutable `parent_objective_id` column.

3. **AI analysis as events** — `objective.ai_analyzed` and `key_result.ai_analyzed` events capture each AI analysis run with scores, feedback, and model version. Replaying these events shows how AI risk scores evolved over the quarter, enabling model accuracy tracking.

4. **Cycle narratives as events** — `cycle.narrative_generated` events store AI-generated executive summaries and retrospectives. The event payload includes the metrics snapshot used to generate the narrative, enabling regeneration with a newer model without losing the original.

5. **Alignment map as read model** — `rm_alignment_map` is a denormalized projection of the full OKR tree: one row per objective with parent link, key result summary, and progress. This enables fast alignment tree rendering without recursive CTEs. The projection is rebuilt from objective events when alignment changes.

6. **Integration sync as event stream** — Each data source has its own stream. `integration.synced` and `integration.sync_failed` events create an audit trail of automated progress updates. The `key_result.integration_synced` event on the key result stream links the sync to the specific value change.

7. **Half-yearly partitions** — OKR events are lower-volume than monitoring checks, so half-yearly partitions balance performance with operational simplicity. Most queries target the current quarter's events.

8. **Separate check-in history read model** — `rm_check_in_history` stores one row per check-in for efficient trend queries (charting progress over time) without JSONB extraction. This complements the `rm_key_results` table which stores only the current state.
