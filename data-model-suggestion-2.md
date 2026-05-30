# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: OKR & Goal Tracking Platform · Created: 2026-05-24

## Philosophy

An OKR platform has a stable relational core (organisations contain teams, teams contain users, objectives contain key results, key results receive check-ins) surrounded by highly variable data: integration configurations differ by provider (Jira needs project keys, Salesforce needs SOQL queries, GitHub needs repo/label filters), AI analysis outputs evolve as models improve, notification preferences vary by channel, check-in metadata varies by source, and goal quality scoring dimensions change as coaching models mature. A JSONB hybrid keeps the invariant OKR hierarchy relational (every objective has a title, owner, cycle, and status) while configuration, AI-generated, and integration-specific data lives in JSONB columns.

This is particularly effective for an OKR platform because:
- **Integration configurations vary by provider** — Jira needs project keys and JQL filters; Salesforce needs object types and SOQL; GitHub needs repo, label, and milestone filters. JSONB stores each provider's config without provider-specific columns.
- **AI outputs evolve** — Today's AI scores goal quality and predicts risk; tomorrow's might detect cross-team dependencies or generate coaching narratives. JSONB avoids migrations as AI features mature.
- **Check-in metadata varies by source** — Manual check-ins have user notes; integration check-ins have provider data; AI-suggested check-ins have confidence scores and reasoning. JSONB handles all shapes.
- **Key result metric configurations are diverse** — Some key results track Jira issue counts, others track Salesforce pipeline value, others are manually updated booleans. JSONB stores the metric-specific configuration alongside the typed progress values.

The result is a 10-table schema that covers the full platform with dramatically simpler queries than the 16-table normalized model.

**Best for:** Rapid MVP development, teams shipping an OKR platform quickly, and platforms where multi-provider integrations, evolving AI features, and flexible metric configurations matter more than constraint enforcement on variable fields.

**Trade-offs:**
- **Pro:** 10 tables — dramatically simpler than the 16-table normalized model
- **Pro:** Integration configuration is schema-free — adding a new provider needs no migration
- **Pro:** AI features added without schema changes
- **Pro:** Check-in history inline on key results — single row read for full progress timeline
- **Con:** JSONB fields lack database-level constraints
- **Con:** Check-in trend queries require JSONB extraction
- **Con:** Application must validate integration configs and AI output shapes
- **Con:** Inline check-in arrays grow over multi-quarter lifecycles

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OKR Framework (Doerr/Grove) | Objective/key result structure |
| SCIM 2.0 | User provisioning; scim_external_id on users |
| SAML 2.0 / OIDC | SSO config in org JSONB |
| OAuth 2.0 | Integration provider auth in JSONB |
| OpenAPI 3.1 | REST API specification |
| ISO 30414 | Goal metrics align with human capital reporting |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| GDPR | Employee performance data handling |

---

## Core Tables

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "locale": "en",
    --  "notifications": {
    --    "checkin_reminder": {"email": true, "slack": true, "in_app": true},
    --    "goal_at_risk": {"email": true, "slack": false},
    --    "comment_mention": {"email": true, "slack": true}
    --  }}
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
    config JSONB NOT NULL DEFAULT '{}',
    -- {"fiscal_year_start_month": 1,
    --  "default_cycle_type": "quarterly",
    --  "checkin_cadence": "weekly",
    --  "sso": {"provider": "okta", "protocol": "saml",
    --          "entity_id": "https://...", "sso_url": "https://..."},
    --  "scim": {"enabled": true, "bearer_token_hash": "..."},
    --  "slack": {"webhook_url": "https://hooks.slack.com/...", "channel": "#okrs"},
    --  "branding": {"logo_url": "...", "accent_color": "#2563eb"}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    parent_team_id UUID REFERENCES teams(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    members JSONB NOT NULL DEFAULT '[]',
    -- [{"user_id": "uuid", "name": "Alice", "role": "lead"},
    --  {"user_id": "uuid", "name": "Bob", "role": "member"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_teams_org ON teams(org_id);
CREATE INDEX idx_teams_parent ON teams(parent_team_id);
```

---

## Planning Cycles

```sql
CREATE TABLE cycles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    cycle_type TEXT NOT NULL DEFAULT 'quarterly' CHECK (cycle_type IN (
        'annual', 'quarterly', 'monthly', 'custom'
    )),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    summary JSONB NOT NULL DEFAULT '{}',
    -- {"total_objectives": 24, "total_key_results": 72,
    --  "avg_progress": 0.65, "on_track": 15, "at_risk": 6, "off_track": 3,
    --  "ai_executive_summary": "Q2 saw strong progress on revenue objectives...",
    --  "ai_retrospective": "Three themes emerged from the quarter...",
    --  "ai_model_version": "claude-sonnet-4-6",
    --  "generated_at": "2026-06-30T10:00:00Z"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name),
    CHECK (end_date > start_date)
);

CREATE INDEX idx_cycles_org ON cycles(org_id);
```

---

## Objectives & Key Results

```sql
CREATE TABLE objectives (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    cycle_id UUID NOT NULL REFERENCES cycles(id),
    owner_id UUID NOT NULL REFERENCES users(id),
    team_id UUID REFERENCES teams(id),
    parent_objective_id UUID REFERENCES objectives(id),
    title TEXT NOT NULL,
    description TEXT,
    level TEXT NOT NULL DEFAULT 'team' CHECK (level IN (
        'company', 'team', 'individual'
    )),
    status TEXT NOT NULL DEFAULT 'on_track' CHECK (status IN (
        'on_track', 'at_risk', 'off_track', 'completed', 'cancelled'
    )),
    progress REAL NOT NULL DEFAULT 0.0 CHECK (progress BETWEEN 0.0 AND 1.0),
    is_stretch BOOLEAN NOT NULL DEFAULT FALSE,
    tags TEXT[] NOT NULL DEFAULT '{}',
    ai JSONB NOT NULL DEFAULT '{}',
    -- {"quality_score": 0.85,
    --  "quality_feedback": ["Objective is specific and inspiring",
    --                       "Consider adding a time-bound element"],
    --  "risk_score": 0.3, "risk_factors": ["2 of 3 KRs at risk", "no check-ins in 14 days"],
    --  "dependencies": [{"objective_id": "uuid", "team": "Platform",
    --                     "risk": "API migration blockes this team's integration goal"}],
    --  "coaching_suggestions": ["Break this into 2 smaller objectives for better focus"],
    --  "model_version": "v2.1", "analyzed_at": "2026-05-24T10:00:00Z"}
    comments JSONB NOT NULL DEFAULT '[]',
    -- [{"user_id": "uuid", "name": "Alice", "body": "Great ambition here!",
    --   "created_at": "2026-04-15T10:00:00Z"},
    --  {"user_id": "uuid", "name": "Bob", "body": "Should we align with Platform team?",
    --   "created_at": "2026-04-16T09:00:00Z"}]
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_objectives_org ON objectives(org_id, cycle_id);
CREATE INDEX idx_objectives_owner ON objectives(owner_id, cycle_id);
CREATE INDEX idx_objectives_team ON objectives(team_id, cycle_id);
CREATE INDEX idx_objectives_parent ON objectives(parent_objective_id);
CREATE INDEX idx_objectives_status ON objectives(org_id, status)
    WHERE status NOT IN ('completed', 'cancelled');
CREATE INDEX idx_objectives_tags ON objectives USING GIN (tags);
CREATE INDEX idx_objectives_ai ON objectives USING GIN (ai);

CREATE TABLE key_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    objective_id UUID NOT NULL REFERENCES objectives(id) ON DELETE CASCADE,
    owner_id UUID NOT NULL REFERENCES users(id),
    title TEXT NOT NULL,
    description TEXT,
    metric_type TEXT NOT NULL DEFAULT 'number' CHECK (metric_type IN (
        'number', 'percentage', 'currency', 'boolean', 'custom'
    )),
    unit TEXT,
    start_value REAL NOT NULL DEFAULT 0,
    current_value REAL NOT NULL DEFAULT 0,
    target_value REAL NOT NULL,
    progress REAL NOT NULL DEFAULT 0.0 CHECK (progress BETWEEN 0.0 AND 1.0),
    status TEXT NOT NULL DEFAULT 'on_track' CHECK (status IN (
        'on_track', 'at_risk', 'off_track', 'completed', 'cancelled'
    )),
    confidence REAL,
    check_ins JSONB NOT NULL DEFAULT '[]',
    -- [{"value": 15, "previous_value": 10, "confidence": "on_track",
    --   "note": "Closed 5 deals this week", "source": "manual",
    --   "user_id": "uuid", "created_at": "2026-04-15T10:00:00Z"},
    --  {"value": 22, "previous_value": 15, "confidence": "on_track",
    --   "note": null, "source": "integration",
    --   "integration": {"provider": "salesforce", "query": "closed_won_count",
    --                    "raw_value": 22, "synced_at": "2026-04-22T06:00:00Z"},
    --   "user_id": null, "created_at": "2026-04-22T06:00:00Z"},
    --  {"value": 25, "previous_value": 22, "confidence": "at_risk",
    --   "note": "Pipeline shrinking — need to accelerate outreach",
    --   "source": "manual", "user_id": "uuid",
    --   "created_at": "2026-04-29T10:00:00Z"}]
    integration JSONB,
    -- {"provider": "jira", "data_source_id": "uuid",
    --  "query": "project = PLAT AND status = Done AND resolved >= startOfQuarter()",
    --  "auto_update": true, "last_synced_at": "2026-05-24T06:00:00Z",
    --  "last_value": 22}
    -- {"provider": "salesforce", "data_source_id": "uuid",
    --  "query": "SELECT COUNT(Id) FROM Opportunity WHERE StageName = 'Closed Won'",
    --  "auto_update": true}
    initiatives JSONB NOT NULL DEFAULT '[]',
    -- [{"title": "Launch new sales playbook", "status": "in_progress",
    --   "owner_id": "uuid", "due_date": "2026-05-15", "priority": "high"},
    --  {"title": "Hire 2 SDRs", "status": "completed", "owner_id": "uuid",
    --   "completed_at": "2026-04-20"}]
    ai JSONB NOT NULL DEFAULT '{}',
    -- {"quality_score": 0.9,
    --  "quality_feedback": ["Key result is specific and measurable"],
    --  "progress_forecast": {"predicted_value": 28, "confidence": 0.7,
    --                        "predicted_status": "at_risk"},
    --  "model_version": "v2.1"}
    comments JSONB NOT NULL DEFAULT '[]',
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kr_objective ON key_results(objective_id);
CREATE INDEX idx_kr_owner ON key_results(owner_id);
```

---

## Data Source Integrations

```sql
CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider TEXT NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'connected' CHECK (status IN (
        'connected', 'syncing', 'error', 'disconnected'
    )),
    config JSONB NOT NULL DEFAULT '{}',
    -- Jira: {"base_url": "https://company.atlassian.net",
    --        "auth": {"type": "oauth2", "access_token_enc": "...", "refresh_token_enc": "..."},
    --        "default_project": "PLAT"}
    -- Salesforce: {"instance_url": "https://company.my.salesforce.com",
    --              "auth": {"type": "oauth2", "access_token_enc": "...", "refresh_token_enc": "..."},
    --              "api_version": "v58.0"}
    -- GitHub: {"org": "company", "auth": {"type": "pat", "token_enc": "..."},
    --          "default_repo": "main-app"}
    -- Custom: {"webhook_url": "https://example.com/okr-data",
    --          "webhook_secret_hash": "...", "format": "json"}
    last_synced_at TIMESTAMPTZ,
    sync_log JSONB NOT NULL DEFAULT '[]',
    -- [{"started_at": "2026-05-24T06:00:00Z", "completed_at": "2026-05-24T06:00:15Z",
    --   "key_results_updated": 5, "status": "success"},
    --  {"started_at": "2026-05-23T06:00:00Z", "error": "OAuth token expired",
    --   "status": "error"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, provider, name)
);

CREATE INDEX idx_data_sources ON data_sources(org_id);
```

---

## API Keys

```sql
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

## Example Queries

### Company OKR tree with alignment

```sql
WITH RECURSIVE okr_tree AS (
    SELECT id, title, level, status, progress, parent_objective_id, 0 AS depth
    FROM objectives
    WHERE org_id = 'org-uuid' AND cycle_id = 'cycle-uuid'
      AND level = 'company'
    UNION ALL
    SELECT o.id, o.title, o.level, o.status, o.progress,
           o.parent_objective_id, t.depth + 1
    FROM objectives o
    JOIN okr_tree t ON t.id = o.parent_objective_id
)
SELECT * FROM okr_tree ORDER BY depth, title;
```

### Key result with check-in trend

```sql
SELECT title, metric_type, unit, start_value, current_value, target_value,
       progress, confidence,
       jsonb_array_length(check_ins) AS checkin_count,
       check_ins->-1->>'value' AS latest_value,
       check_ins->-1->>'confidence' AS latest_confidence,
       check_ins->-1->>'created_at' AS latest_checkin_at
FROM key_results
WHERE objective_id = 'objective-uuid'
ORDER BY sort_order;
```

### At-risk goals with AI analysis

```sql
SELECT o.title, o.level, o.status, o.progress,
       o.ai->>'risk_score' AS risk_score,
       o.ai->'risk_factors' AS risk_factors,
       o.ai->'dependencies' AS dependencies,
       u.name AS owner, t.name AS team
FROM objectives o
JOIN users u ON u.id = o.owner_id
LEFT JOIN teams t ON t.id = o.team_id
WHERE o.org_id = 'org-uuid'
  AND o.cycle_id = 'cycle-uuid'
  AND o.status IN ('at_risk', 'off_track')
ORDER BY (o.ai->>'risk_score')::REAL DESC NULLS LAST;
```

### Integration sync status

```sql
SELECT provider, name, status, last_synced_at,
       sync_log->-1->>'status' AS last_sync_status,
       sync_log->-1->>'key_results_updated' AS last_krs_updated
FROM data_sources
WHERE org_id = 'org-uuid'
ORDER BY provider;
```

### Checkin cadence compliance (last 7 days)

```sql
SELECT u.name,
       COUNT(*) FILTER (WHERE (ci->>'created_at')::TIMESTAMPTZ >= now() - INTERVAL '7 days') AS recent_checkins
FROM users u
JOIN key_results kr ON kr.owner_id = u.id
JOIN objectives o ON o.id = kr.objective_id,
     jsonb_array_elements(kr.check_ins) AS ci
WHERE o.org_id = 'org-uuid'
  AND o.cycle_id = 'cycle-uuid'
GROUP BY u.id, u.name
ORDER BY recent_checkins ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 3 | users (settings JSONB), organisations (config, SSO, SCIM JSONB), teams (members JSONB) |
| Planning | 1 | cycles (summary with AI narrative JSONB) |
| OKRs | 2 | objectives (ai, comments JSONB), key_results (check_ins, integration, initiatives, ai, comments JSONB) |
| Integrations | 1 | data_sources (config, sync_log JSONB) |
| API | 1 | api_keys |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Check-ins inline on key results** — `key_results.check_ins` stores the full check-in history as a JSONB array. Most key results have 12-15 check-ins per quarter. The most common read pattern is "show me this key result with its progress trend" — embedding check-ins eliminates a join. Trade-off: per-check-in queries require JSONB extraction.

2. **Initiatives inline on key results** — `key_results.initiatives` stores linked tasks/initiatives as a JSONB array. Most key results have 2-5 initiatives. No separate `initiatives` table. Trade-off: cross-key-result initiative queries require JSONB unnesting.

3. **Integration config inline on key results** — `key_results.integration` stores the provider-specific query and sync configuration. The `data_source_id` links to the org-level data source for credentials. Adding a new integration provider (Linear, Asana, HubSpot) requires no schema change.

4. **AI analysis inline on objectives and key results** — `objectives.ai` and `key_results.ai` store quality scores, risk predictions, dependency detections, progress forecasts, and coaching suggestions. New AI features are added to this JSONB without migrations.

5. **Comments inline** — `objectives.comments` and `key_results.comments` store collaboration threads as JSONB arrays. For typical OKR usage (2-10 comments per goal), this avoids a separate table. Trade-off: no foreign key to users table on comment authors.

6. **Cycle summaries inline** — `cycles.summary` stores pre-computed aggregate metrics (total objectives, average progress, status distribution) and AI-generated executive summaries. The cycle summary is updated by a background worker after each check-in.

7. **Team members as JSONB** — `teams.members` stores the member list with roles as a JSONB array. For typical team sizes (5-20 members), this avoids a junction table. Trade-off: no foreign key enforcement on member user IDs.

8. **8 tables total** — Compared to 16 in the normalized model. The JSONB approach trades constraint enforcement for development speed, particularly effective for the many integration-specific and AI-evolving fields in an OKR platform.
