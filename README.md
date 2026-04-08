# ai-ops-platform

A production AI operations platform that powers business operations, content, and client delivery as one integrated system. Built and operated by a single engineer.

This repo documents the architecture and design decisions. The source code is private.

## What It Does

Runs an entire business autonomously: task scheduling, agent orchestration, content pipeline, client delivery, health monitoring, and outbound sales infrastructure. One system, one database, one operator.

## Architecture

```
                         +------------------+
                         |   Telegram Bot   |
                         | (notifications)  |
                         +--------+---------+
                                  |
+-------------+          +--------+---------+          +----------------+
|  n8n / Make |          |    FastAPI        |          |  Next.js       |
|  (workflows)|--------->|    Backend        |--------->|  Dashboard     |
+-------------+          +--------+---------+          +----------------+
                                  |
                         +--------+---------+
                         |     SQLite        |
                         |  (44 tables,      |
                         |   6 views)        |
                         +--------+---------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
     +--------+------+  +--------+------+  +---------+-----+
     |  Agent        |  |  Scheduler    |  |  Health       |
     |  Orchestrator |  |  (launchd)    |  |  Monitor      |
     +---------------+  +---------------+  +---------------+
```

## System Stats

| Metric | Count |
|--------|-------|
| Database tables | 44 |
| Stable views (skill contract layer) | 6 |
| Automated skills | 44+ |
| Modules | 11 |
| Scheduled agents (launchd) | 5+ |
| Workflow integrations (n8n + Make.com) | 10+ |

## Core Components

### Agent Orchestrator
Polls for runnable tasks, routes them through an approval gate (Slack-based), then spawns an agent subprocess to execute. Approval flow prevents autonomous actions on anything flagged as requiring human review.

### Skill System (D>P>D)
Every automated skill follows Deterministic > Probabilistic > Deterministic:
1. Deterministic script gathers inputs, validates preconditions
2. Probabilistic LLM step makes decisions within constrained scope
3. Deterministic script validates output, logs telemetry, writes results

No LLM-to-LLM chains without deterministic validation between them.

### Database Layer
SQLite is the single source of truth. All connections go through a centralized connection manager with WAL mode for concurrent reads. Write transactions use immediate locking. Skills read from 6 stable views, never tables directly. Views are the contract -- table structure changes without breaking skills.

### Health Monitoring
Automated wiring verification runs after every build. Checks: database connectivity, API endpoint availability, view integrity, scheduled agent status, cross-module dependencies. Failures block deployment claims.

### Content Pipeline
Stateful content lifecycle: captured > drafted > approved > scheduled > published > tracked. Content files live in a flat folder. The database owns state, not the filesystem. Metrics flow to a unified table across all platforms.

### Scheduling
launchd is the canonical scheduler. Agents run on the host machine (Mac Mini) 24/7. Health checks, content collectors, and morning briefs run on cron cycles.

### Notification Layer
Telegram bot with topic-based routing. Alerts, daily briefs, agent status updates, and pipeline notifications all flow through structured message formats to dedicated topic threads.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend API | Python, FastAPI |
| Frontend | Next.js |
| Database | SQLite (WAL mode) |
| Agent orchestration | Python, Claude API |
| Workflow automation | n8n, Make.com |
| Infrastructure | macOS (Mac Mini M4), launchd |
| Monitoring | Custom health checks, Grafana |
| Notifications | Telegram Bot API |
| CI/CD | GitHub Actions |
| IaC | Terraform (for cloud resources) |
| Containerization | Docker |
| Languages | Python, Bash, TypeScript, YAML |

## Design Principles

1. **Single source of truth.** All state lives in SQLite. No CSV, JSONL, or markdown data stores.
2. **Views as contracts.** Skills read from views, not tables. Schema changes don't break consumers.
3. **Verify before done.** No build is complete without automated verification. Code written but not verified is not done.
4. **Deterministic gates.** Every LLM output passes through validation before it affects system state.
5. **Minimal blast radius.** Subagents are read-only. All writes happen in the main session. One writer at a time.

## What I Learned Building This

- Autonomous agents need kill switches. Approval gates saved me from bad executions more than once.
- SQLite WAL mode handles concurrent reads well, but concurrent writes from multiple agent processes will silently fail. One writer, enforced at the connection layer.
- Scheduling with launchd is more reliable than cron for always-on agents on macOS. Process recovery, logging, and keepalive are built in.
- LLM chains without validation between steps compound errors. D>P>D eliminates this by forcing a deterministic check at every handoff.
- The dashboard is the last thing you build, not the first. The database, API, and skills need to be stable before you visualize anything.

## Contact

Richard Igbinoba
richard.igbinoba@yahoo.com
linkedin.com/in/richardigbinoba
