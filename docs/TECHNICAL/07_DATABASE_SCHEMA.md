# 07 - Database Schema

## Visão Geral

PostgreSQL 14+ como banco principal. Schema organizado por domínio com isolamento por nicho/canal.

## Diagrama de Relacionamentos

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│  niches  │←───┐│ channels │←───┐│   videos     │
└──────────┘    │└──────────┘    │└──────────────┘
     ↑          │     ↑          │       ↑
     │          │     │          │       │
┌────┴─────┐   │┌────┴─────┐   │┌──────┴───────┐
│  facts   │───┘│ coverage │───┘│   shorts     │
└──────────┘    └──────────┘    └──────────────┘
     ↑                                ↑
     │                                │
┌────┴─────┐                   ┌──────┴───────┐
│learnings │                   │  uploads     │
└──────────┘                   └──────────────┘
                               ┌──────────────┐
                               │  workflows   │
                               └──────────────┘
                               ┌──────────────┐
                               │ checkpoints  │
                               └──────────────┘
                               ┌──────────────┐
                               │  analytics   │
                               └──────────────┘
```

## Schema Completo

### Core Tables

```sql
-- ========================================
-- NICHES
-- ========================================
CREATE TABLE niches (
    niche_id        VARCHAR(50) PRIMARY KEY,
    display_name    VARCHAR(200) NOT NULL,
    language        VARCHAR(10) DEFAULT 'pt-BR',
    config          JSONB NOT NULL,           -- niche.schema.json completo
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- ========================================
-- CHANNELS
-- ========================================
CREATE TABLE channels (
    channel_id      VARCHAR(50) PRIMARY KEY,  -- YouTube channel ID
    niche_id        VARCHAR(50) NOT NULL,
    channel_name    VARCHAR(200) NOT NULL,
    youtube_handle  VARCHAR(100),
    oauth_token     TEXT,                     -- encrypted
    oauth_refresh   TEXT,                     -- encrypted
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id)
);

CREATE INDEX idx_channels_niche ON channels(niche_id);

-- ========================================
-- VIDEOS
-- ========================================
CREATE TABLE videos (
    video_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      VARCHAR(50) NOT NULL,
    niche_id        VARCHAR(50) NOT NULL,
    youtube_id      VARCHAR(20),              -- após upload
    video_type      VARCHAR(20) NOT NULL,     -- 'longform' | 'short'
    title           VARCHAR(200),
    description     TEXT,
    tags            TEXT[],
    topic           VARCHAR(300) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- Status: draft → researching → scripted → producing →
    --         assembling → ready → uploaded → published

    -- Paths
    script_path     TEXT,
    audio_path      TEXT,
    video_path      TEXT,
    thumbnail_path  TEXT,
    subtitles_path  TEXT,

    -- Metadata
    duration_seconds  INTEGER,
    file_size_bytes   BIGINT,
    resolution        VARCHAR(20),
    word_count        INTEGER,

    -- Timestamps
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    published_at    TIMESTAMP,
    scheduled_for   TIMESTAMP,

    CONSTRAINT fk_channel FOREIGN KEY (channel_id)
        REFERENCES channels(channel_id),
    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id)
);

CREATE INDEX idx_videos_channel ON videos(channel_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_type ON videos(video_type);
CREATE INDEX idx_videos_niche ON videos(niche_id);
CREATE INDEX idx_videos_published ON videos(published_at);

-- ========================================
-- SHORTS (Extensão de videos para shorts-specific data)
-- ========================================
CREATE TABLE shorts (
    short_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL,            -- ref na tabela videos
    source_video_id UUID,                     -- long-form de origem (se extraction)
    source_type     VARCHAR(20) NOT NULL,     -- 'extraction' | 'repurpose' | 'original'
    segment_start   FLOAT,                    -- timestamp início no source
    segment_end     FLOAT,                    -- timestamp fim no source
    hook_text       TEXT,                     -- hook reescrito

    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id),
    CONSTRAINT fk_source FOREIGN KEY (source_video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_shorts_source ON shorts(source_video_id);
```

### Knowledge Tables

```sql
-- ========================================
-- FACTS
-- ========================================
CREATE TABLE facts (
    fact_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    fact_text       TEXT NOT NULL,
    source_url      TEXT,
    source_name     VARCHAR(200),
    confidence      DECIMAL(3,2) DEFAULT 0.80,
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP,
    used_count      INTEGER DEFAULT 0,
    last_used_at    TIMESTAMP,
    verified_by     VARCHAR(50),
    tags            TEXT[],

    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id)
);

CREATE INDEX idx_facts_niche ON facts(niche_id);
CREATE INDEX idx_facts_category ON facts(niche_id, category);
CREATE INDEX idx_facts_tags ON facts USING GIN(tags);
CREATE INDEX idx_facts_expires ON facts(expires_at) WHERE expires_at IS NOT NULL;

-- ========================================
-- LEARNINGS
-- ========================================
CREATE TABLE learnings (
    learning_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    learning_text   TEXT NOT NULL,
    evidence        JSONB,
    confidence      DECIMAL(3,2) DEFAULT 0.70,
    validated       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW(),
    video_id        UUID,

    CONSTRAINT fk_niche FOREIGN KEY (niche_id)
        REFERENCES niches(niche_id),
    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_learnings_niche ON learnings(niche_id);
CREATE INDEX idx_learnings_type ON learnings(niche_id, type);

-- ========================================
-- COVERAGE
-- ========================================
CREATE TABLE coverage (
    coverage_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    niche_id        VARCHAR(50) NOT NULL,
    channel_id      VARCHAR(50) NOT NULL,
    topic           VARCHAR(300) NOT NULL,
    subtopics       TEXT[],
    video_id        UUID,
    video_type      VARCHAR(20),
    published_at    TIMESTAMP,
    performance     JSONB,

    CONSTRAINT fk_channel FOREIGN KEY (channel_id)
        REFERENCES channels(channel_id),
    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_coverage_topic ON coverage(niche_id, topic);
CREATE INDEX idx_coverage_channel ON coverage(channel_id);
```

### Workflow Tables

```sql
-- ========================================
-- WORKFLOWS
-- ========================================
CREATE TABLE workflows (
    workflow_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL,
    workflow_type   VARCHAR(30) NOT NULL,     -- 'longform_production' | 'shorts_extraction'
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- Status: pending → running → paused (checkpoint) →
    --         running → completed | failed
    current_step    VARCHAR(50),
    total_steps     INTEGER,
    error_message   TEXT,
    retry_count     INTEGER DEFAULT 0,
    max_retries     INTEGER DEFAULT 3,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_workflows_status ON workflows(status);
CREATE INDEX idx_workflows_video ON workflows(video_id);

-- ========================================
-- CHECKPOINTS
-- ========================================
CREATE TABLE checkpoints (
    checkpoint_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL,
    video_id        UUID NOT NULL,
    checkpoint_type VARCHAR(30) NOT NULL,
    -- Types: script_review | audio_review | broll_review | final_review
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Status: pending → approved | rejected | revision_requested
    reviewer_notes  TEXT,
    reviewed_at     TIMESTAMP,
    reviewed_by     VARCHAR(100),
    data            JSONB,                   -- checkpoint-specific data
    created_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id)
        REFERENCES workflows(workflow_id),
    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_checkpoints_status ON checkpoints(status);
CREATE INDEX idx_checkpoints_workflow ON checkpoints(workflow_id);

-- ========================================
-- UPLOADS
-- ========================================
CREATE TABLE uploads (
    upload_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL,
    channel_id      VARCHAR(50) NOT NULL,
    youtube_id      VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Status: pending → uploading → processing → live | failed
    scheduled_for   TIMESTAMP,
    uploaded_at     TIMESTAMP,
    error_message   TEXT,
    retry_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id),
    CONSTRAINT fk_channel FOREIGN KEY (channel_id)
        REFERENCES channels(channel_id)
);

CREATE INDEX idx_uploads_status ON uploads(status);
CREATE INDEX idx_uploads_scheduled ON uploads(scheduled_for);
```

### Analytics Tables

```sql
-- ========================================
-- ANALYTICS (snapshots diários)
-- ========================================
CREATE TABLE analytics (
    analytics_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL,
    youtube_id      VARCHAR(20) NOT NULL,
    snapshot_date   DATE NOT NULL,

    -- YouTube metrics
    views           INTEGER DEFAULT 0,
    likes           INTEGER DEFAULT 0,
    comments        INTEGER DEFAULT 0,
    shares          INTEGER DEFAULT 0,
    watch_time_hours DECIMAL(10,2) DEFAULT 0,
    avg_view_duration_seconds INTEGER DEFAULT 0,
    ctr             DECIMAL(5,2),              -- click-through rate %
    avg_retention   DECIMAL(5,2),              -- avg % watched
    impressions     INTEGER DEFAULT 0,
    subscribers_gained INTEGER DEFAULT 0,

    created_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id),
    CONSTRAINT uq_video_date UNIQUE (video_id, snapshot_date)
);

CREATE INDEX idx_analytics_video ON analytics(video_id);
CREATE INDEX idx_analytics_date ON analytics(snapshot_date);

-- ========================================
-- PRODUCTION METRICS
-- ========================================
CREATE TABLE production_metrics (
    metric_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL,
    step_name       VARCHAR(50) NOT NULL,
    duration_seconds DECIMAL(10,2),
    api_calls       INTEGER DEFAULT 0,
    api_cost        DECIMAL(10,4) DEFAULT 0,
    error_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_video FOREIGN KEY (video_id)
        REFERENCES videos(video_id)
);

CREATE INDEX idx_prodmetrics_video ON production_metrics(video_id);
```

### System Tables

```sql
-- ========================================
-- JOB QUEUE
-- ========================================
CREATE TABLE job_queue (
    job_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_type        VARCHAR(50) NOT NULL,
    payload         JSONB NOT NULL,
    priority        INTEGER DEFAULT 5,         -- 1 (highest) to 10 (lowest)
    status          VARCHAR(20) DEFAULT 'queued',
    -- Status: queued → processing → completed | failed | dead
    attempts        INTEGER DEFAULT 0,
    max_attempts    INTEGER DEFAULT 3,
    locked_by       VARCHAR(100),
    locked_at       TIMESTAMP,
    scheduled_for   TIMESTAMP DEFAULT NOW(),
    completed_at    TIMESTAMP,
    error_message   TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_jobs_status ON job_queue(status, priority, scheduled_for);
CREATE INDEX idx_jobs_type ON job_queue(job_type);

-- ========================================
-- SYSTEM CONFIG
-- ========================================
CREATE TABLE system_config (
    key             VARCHAR(100) PRIMARY KEY,
    value           JSONB NOT NULL,
    description     TEXT,
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

## Migrations

Usar tool de migration (Alembic para Python):

```
migrations/
├── 001_initial_schema.sql
├── 002_add_knowledge_tables.sql
├── 003_add_workflow_tables.sql
├── 004_add_analytics.sql
└── 005_add_job_queue.sql
```

## Queries Comuns

```sql
-- Vídeos pendentes de checkpoint
SELECT v.title, c.checkpoint_type, c.status
FROM checkpoints c
JOIN videos v ON c.video_id = v.video_id
WHERE c.status = 'pending'
ORDER BY c.created_at;

-- Performance por nicho (últimos 30 dias)
SELECT n.display_name, COUNT(v.video_id) as total_videos,
       AVG(a.ctr) as avg_ctr, AVG(a.avg_retention) as avg_retention
FROM niches n
JOIN videos v ON v.niche_id = n.niche_id
JOIN analytics a ON a.video_id = v.video_id
WHERE a.snapshot_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY n.display_name;

-- Knowledge reuse rate
SELECT niche_id,
       COUNT(*) as total_researches,
       SUM(CASE WHEN strategy = 'cache_only' THEN 1 ELSE 0 END) as cache_hits,
       ROUND(SUM(CASE WHEN strategy = 'cache_only' THEN 1 ELSE 0 END)::decimal
             / COUNT(*) * 100, 1) as reuse_rate_pct
FROM production_metrics
WHERE step_name = 'research'
GROUP BY niche_id;

-- Custo total por mês
SELECT DATE_TRUNC('month', created_at) as month,
       SUM(api_cost) as total_cost,
       COUNT(DISTINCT video_id) as videos_produced,
       ROUND(SUM(api_cost) / COUNT(DISTINCT video_id), 4) as cost_per_video
FROM production_metrics
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month DESC;
```

## Backup & Manutenção

```sql
-- Limpar facts expirados (job semanal)
DELETE FROM facts WHERE expires_at < NOW();

-- Limpar analytics antigos (manter 6 meses)
DELETE FROM analytics WHERE snapshot_date < CURRENT_DATE - INTERVAL '180 days';

-- Vacuum após deletes grandes
VACUUM ANALYZE facts;
VACUUM ANALYZE analytics;
```

---

**Anterior:** [06_THUMBNAIL_SYSTEM.md](./06_THUMBNAIL_SYSTEM.md)
**Próximo:** [08_API_SPECIFICATIONS.md](./08_API_SPECIFICATIONS.md)
