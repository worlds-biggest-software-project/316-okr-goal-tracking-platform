# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: OKR & Goal Tracking Platform · Created: 2026-05-24

## Philosophy

An OKR platform manages a clear hierarchy: organisations contain teams, teams contain users, users own objectives, objectives contain key results, key results receive periodic check-ins, and initiatives (tasks) link to key results. Layered on top are planning cycles (quarterly), alignment relationships (parent-child objectives across levels), AI features (goal quality scoring, risk prediction, narrative generation), and data source integrations for automated progress updates. A normalized relational model gives each concept its own table with explicit foreign keys, enabling database-level enforcement of the cascading alignment chain and clean separation between structure (objectives, key results), progress tracking (check-ins, progress snapshots), collaboration (comments, reactions), and AI enrichment (quality scores, risk flags, narratives).

This mirrors the conceptual model that OKR practitioners already carry: an objective is an aspirational goal, key results are measurable outcomes that prove the objective is met, check-ins are periodic progress updates, and alignment means a team objective supports a company objective. Each maps to a table with typed columns and constraints.

The normalized model supports the Doerr/Grove OKR framework cleanly: objectives are qualitative and aspirational (no numeric target), key results are quantitative with start/current/target values and unit types, and check-ins capture both quantitative progress and qualitative confidence.

**Best for:** Teams building a production-grade OKR platform where data integrity across the objective→key result→check-in pipeline is critical, where cascading alignment needs explicit foreign key enforcement, and where SCIM provisioning, RBAC, and multi-quarter planning are first-class features.

**Trade-offs:**
- **Pro:** Database-enforced alignment chain from company objective to individual key result
- **Pro:** Check-ins as explicit rows enable trend analysis and cadence reporting
- **Pro:** Integration data sources as typed rows enable progress automation
- **Pro:** SCIM user provisioning modeled with explicit team membership
- **Con:** 22 tables — complex schema
- **Con:** High join count for "show all OKRs with key results, progress, and alignment"
- **Con:** AI features evolve faster than typed columns can follow
- **Con:** Integration source configuration varies by provider

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 30414 | Goal metrics align with human capital reporting standards |
| OKR Framework (Doerr/Grove) | Objective/key result structure follows canonical OKR definitions |
| SCIM 2.0 (RFC 7643/7644) | User and team provisioning from HRIS |
| SAML 2.0 / OIDC | SSO for enterprise identity management |
| OAuth 2.0 (RFC 6749) | Integration with Jira, Salesforce, GitHub APIs |
| OpenAPI 3.1 | REST API specification |
| ISO 8601 | All timestamps as TIMESTAMPTZ; date fields as DATE |
| GDPR | Employee performance data classified as personal data |

---

## Organisations, Teams & Users

```sql
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    fiscal_year_start_month INT NOT NULL DEFAULT 1 CHECK (fiscal_year_start_month BETWEEN 1 AND 12),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    parent_team_id UUID REFERENCES teams(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_teams_org ON teams(org_id);
CREATE INDEX idx_teams_parent ON teams(parent_team_id);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN (
        'owner', 'admin', 'manager', 'member', 'viewer'
    )),
    timezone TEXT NOT NULL DEFAULT 'UTC',
    scim_external_id TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_scim ON users(scim_external_id) WHERE scim_external_id IS NOT NULL;

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('lead', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name),
    CHECK (end_date > start_date)
);

CREATE INDEX idx_cycles_org ON cycles(org_id);
CREATE INDEX idx_cycles_active ON cycles(org_id, is_active) WHERE is_active = TRUE;
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
    confidence REAL CHECK (confidence BETWEEN 0.0 AND 1.0),
    data_source_id UUID REFERENCES data_sources(id),
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kr_objective ON key_results(objective_id);
CREATE INDEX idx_kr_owner ON key_results(owner_id);
CREATE INDEX idx_kr_data_source ON key_results(data_source_id) WHERE data_source_id IS NOT NULL;
```

---

## Check-ins

```sql
CREATE TABLE check_ins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id UUID NOT NULL REFERENCES key_results(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    previous_value REAL NOT NULL,
    new_value REAL NOT NULL,
    confidence TEXT NOT NULL DEFAULT 'on_track' CHECK (confidence IN (
        'on_track', 'at_risk', 'off_track'
    )),
    note TEXT,
    source TEXT NOT NULL DEFAULT 'manual' CHECK (source IN (
        'manual', 'integration', 'ai_suggested'
    )),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_checkins_kr ON check_ins(key_result_id, created_at DESC);
CREATE INDEX idx_checkins_user ON check_ins(user_id, created_at DESC);
```

---

## Initiatives & Tasks

```sql
CREATE TABLE initiatives (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id UUID NOT NULL REFERENCES key_results(id) ON DELETE CASCADE,
    owner_id UUID NOT NULL REFERENCES users(id),
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'not_started' CHECK (status IN (
        'not_started', 'in_progress', 'completed', 'blocked', 'cancelled'
    )),
    priority TEXT NOT NULL DEFAULT 'medium' CHECK (priority IN (
        'critical', 'high', 'medium', 'low'
    )),
    due_date DATE,
    external_id TEXT,
    external_source TEXT,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_initiatives_kr ON initiatives(key_result_id);
CREATE INDEX idx_initiatives_owner ON initiatives(owner_id);
CREATE INDEX idx_initiatives_status ON initiatives(status) WHERE status NOT IN ('completed', 'cancelled');
```

---

## Comments & Collaboration

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    commentable_type TEXT NOT NULL CHECK (commentable_type IN (
        'objective', 'key_result', 'initiative', 'check_in'
    )),
    commentable_id UUID NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_target ON comments(commentable_type, commentable_id, created_at DESC);
CREATE INDEX idx_comments_user ON comments(user_id);
```

---

## Data Source Integrations

```sql
CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK (provider IN (
        'jira', 'github', 'salesforce', 'google_analytics',
        'hubspot', 'linear', 'asana', 'custom_webhook'
    )),
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'connected' CHECK (status IN (
        'connected', 'syncing', 'error', 'disconnected'
    )),
    credentials_enc TEXT,
    config TEXT,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, provider, name)
);

CREATE TABLE data_source_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    key_result_id UUID NOT NULL REFERENCES key_results(id) ON DELETE CASCADE,
    query_expression TEXT NOT NULL,
    transform_expression TEXT,
    last_value REAL,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_source_id, key_result_id)
);
```

---

## AI Features

```sql
CREATE TABLE ai_goal_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    objective_id UUID REFERENCES objectives(id) ON DELETE CASCADE,
    key_result_id UUID REFERENCES key_results(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'quality_score', 'risk_prediction', 'dependency_detection',
        'progress_forecast', 'coaching_suggestion'
    )),
    score REAL,
    summary TEXT NOT NULL,
    details TEXT,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_analysis_objective ON ai_goal_analyses(objective_id);
CREATE INDEX idx_ai_analysis_kr ON ai_goal_analyses(key_result_id);

CREATE TABLE ai_narratives (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    cycle_id UUID NOT NULL REFERENCES cycles(id),
    narrative_type TEXT NOT NULL CHECK (narrative_type IN (
        'executive_summary', 'quarterly_retrospective', 'team_report',
        'board_deck', 'all_hands'
    )),
    scope_team_id UUID REFERENCES teams(id),
    content TEXT NOT NULL,
    model_version TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'reviewed', 'published'
    )),
    published_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_narratives_cycle ON ai_narratives(org_id, cycle_id);
```

---

## Notifications & API

```sql
CREATE TABLE notification_preferences (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    channel TEXT NOT NULL CHECK (channel IN ('email', 'slack', 'teams', 'in_app')),
    event_type TEXT NOT NULL CHECK (event_type IN (
        'checkin_reminder', 'status_change', 'comment_mention',
        'goal_at_risk', 'cycle_starting', 'cycle_ending'
    )),
    is_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    PRIMARY KEY (user_id, channel, event_type)
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

## Example Queries

### Company OKR tree with alignment

```sql
WITH RECURSIVE okr_tree AS (
    SELECT o.id, o.title, o.level, o.status, o.progress,
           o.parent_objective_id, 0 AS depth
    FROM objectives o
    WHERE o.org_id = 'org-uuid'
      AND o.cycle_id = 'cycle-uuid'
      AND o.level = 'company'
    UNION ALL
    SELECT o.id, o.title, o.level, o.status, o.progress,
           o.parent_objective_id, t.depth + 1
    FROM objectives o
    JOIN okr_tree t ON t.id = o.parent_objective_id
)
SELECT * FROM okr_tree ORDER BY depth, title;
```

### Key result progress with check-in history

```sql
SELECT kr.title, kr.metric_type, kr.unit,
       kr.start_value, kr.current_value, kr.target_value,
       kr.progress, kr.confidence,
       ci.new_value, ci.confidence AS checkin_confidence,
       ci.note, ci.created_at AS checkin_at
FROM key_results kr
LEFT JOIN check_ins ci ON ci.key_result_id = kr.id
WHERE kr.objective_id = 'objective-uuid'
ORDER BY kr.sort_order, ci.created_at DESC;
```

### At-risk goals across the organisation

```sql
SELECT o.title AS objective, o.level, o.status,
       u.name AS owner, t.name AS team,
       o.progress, c.name AS cycle
FROM objectives o
JOIN users u ON u.id = o.owner_id
LEFT JOIN teams t ON t.id = o.team_id
JOIN cycles c ON c.id = o.cycle_id
WHERE o.org_id = 'org-uuid'
  AND c.is_active = TRUE
  AND o.status IN ('at_risk', 'off_track')
ORDER BY o.level, o.status;
```

### Check-in cadence compliance

```sql
SELECT u.name, COUNT(ci.id) AS checkin_count,
       MAX(ci.created_at) AS last_checkin
FROM users u
JOIN key_results kr ON kr.owner_id = u.id
JOIN objectives o ON o.id = kr.objective_id
LEFT JOIN check_ins ci ON ci.key_result_id = kr.id
    AND ci.created_at >= now() - INTERVAL '7 days'
WHERE o.org_id = 'org-uuid'
  AND o.cycle_id = 'cycle-uuid'
GROUP BY u.id, u.name
ORDER BY checkin_count ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Org & Teams | 4 | organisations, teams (self-referential hierarchy), users, team_members |
| Planning | 1 | cycles |
| OKRs | 2 | objectives (with parent_objective_id for alignment), key_results |
| Check-ins | 1 | check_ins |
| Initiatives | 1 | initiatives |
| Collaboration | 1 | comments (polymorphic) |
| Integrations | 2 | data_sources, data_source_mappings |
| AI | 2 | ai_goal_analyses, ai_narratives |
| Notifications & API | 2 | notification_preferences, api_keys |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Objective alignment via parent_objective_id** — `objectives.parent_objective_id` creates a self-referential tree enabling company → team → individual cascading. A recursive CTE traverses the full alignment tree. This is the standard pattern for OKR platforms and maps directly to the Doerr/Grove framework.

2. **Key results with typed metrics** — `key_results` stores `metric_type` (number, percentage, currency, boolean, custom), `unit`, and `start_value`/`current_value`/`target_value`. Progress is computed as `(current - start) / (target - start)`. This enables the platform to display progress bars and trend charts with proper units.

3. **Check-ins as explicit rows** — `check_ins` stores each progress update with previous/new values, confidence, note, and source (manual, integration, AI-suggested). This enables trend analysis, cadence compliance reporting, and AI-generated retrospectives from check-in history.

4. **Data source mappings** — `data_source_mappings` links a data source (Jira, Salesforce) to a specific key result with a query expression (e.g., "count of closed issues in project X"). The sync worker evaluates these mappings and creates check-ins with `source = 'integration'`.

5. **Teams as a tree** — `teams.parent_team_id` enables nested team hierarchies (Engineering → Backend → API Team). Combined with `objectives.team_id`, this supports the team-level alignment that OKR platforms need.

6. **AI analyses separate from goals** — `ai_goal_analyses` stores quality scores, risk predictions, dependency detections, and coaching suggestions as separate rows. New analysis types are added without schema changes. Each analysis references the model version for tracking AI accuracy over time.

7. **AI narratives for reporting** — `ai_narratives` stores generated executive summaries, quarterly retrospectives, and board decks. These are scoped to a cycle and optionally to a team, enabling both org-wide and team-level reporting.

8. **Polymorphic comments** — `comments` uses `commentable_type` and `commentable_id` to support comments on objectives, key results, initiatives, and check-ins without separate comment tables per entity.
