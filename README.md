# Deployment Protection Framework

**4-layer defense-in-depth for automated deployments — pre-deploy validation, post-deploy health checks, automatic rollback, and cooldown logic. Zero customer-visible outages.**

---

## The Problem

I was building a system to run a business while I was on a 4-month vacation. That means no one watching the dashboard. No one checking logs. No one rolling back when something goes wrong.

The failure mode I was most worried about wasn't a bad deploy. It was a bad deploy that *keeps deploying*. Rollback fires, triggers a new deploy, new deploy fails, system loops. Each recovery attempt makes things worse. By the time I checked in, the site had been broken for days and the database was full of failed deployment records trying to rerun.

The system needs to be smart enough to fail well.

This framework implements classic defense-in-depth at the deployment layer: two production n8n workflows (Agent 5: Website Deployment, Agent 14: Website Health Monitor), 24 nodes across four protection layers. Preventive controls stop bad deploys before they ship. Detective controls catch problems after they land. Corrective controls revert damage automatically. Circuit breakers prevent the system from thrashing itself into a worse state.

The result: zero customer-visible outages from bad deploys across the entire operational lifetime of the system.

---

## Architecture

```
┌──────────────────────────────────────────────┐
│              Deploy Triggered                  │
└─────────────────────┬────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │   Layer 1: Validation    │
         │   (Preventive Control)   │
         │                          │
         │   • Content checks       │
         │   • Compliance gates     │
         │   • Schema verification  │
         │   • Pre-flight rules     │
         └────────────┬────────────┘
                      │ PASS
         ┌────────────▼────────────┐
         │   Layer 2: Health Check  │
         │   (Detective Control)    │
         │                          │
         │   • Post-deploy probe    │
         │   • Status enum:         │
         │     healthy → degraded   │
         │     → failed             │
         │   • system_flags table   │
         └────────────┬────────────┘
                      │
              ┌───────▼───────┐
              │  Status?       │
              ├───────────────┤
              │ healthy → ✅   │
              │ degraded → ⚠️  │
              │ failed → 🔴    │
              └───┬───────┬───┘
                  │       │
                  │  ┌────▼───────────────┐
                  │  │ Layer 3: Rollback   │
                  │  │ (Corrective Control)│
                  │  │                     │
                  │  │ • Cloudflare auto-  │
                  │  │   revert            │
                  │  │ • Previous known-   │
                  │  │   good state        │
                  │  │ • Triggered by      │
                  │  │   health enum       │
                  │  └────────┬────────────┘
                  │           │
         ┌────────▼───────────▼────────────┐
         │   Layer 4: Cooldown              │
         │   (Circuit Breaker)              │
         │                                  │
         │   • Prevents cascading failures  │
         │   • Blocks re-deploy during      │
         │     recovery window              │
         │   • State tracked in             │
         │     system_flags                 │
         └─────────────────────────────────┘
```

### Layer Breakdown

**Layer 1 — Pre-Deploy Validation (Preventive):** Catches bad deploys before they reach production. Content compliance checks, schema compatibility verification, and pre-flight rules all execute before any code ships. If validation fails, the deploy never starts. This is the cheapest layer — stopping a problem here costs nothing.

**Layer 2 — Post-Deploy Health Checks (Detective):** Monitors system state after deployment using a health status enum stored in the `system_flags` runtime table. Health probes run on a schedule and update status after every check window. Transition logic is explicit:

- `healthy` = last 3 health checks passed
- `degraded` = 1–2 of last 3 checks failed → trigger warning alert
- `failed` = all 3 of last 3 checks failed → trigger Layer 3 rollback

The three-state enum matters: a binary healthy/unhealthy would collapse `degraded` into `failed` and fire rollback prematurely. `degraded` means "still serving traffic, investigate before acting." Any workflow can query current system state before executing:

```sql
SELECT value
FROM system_flags
WHERE key = 'deployment_health_status'
LIMIT 1;
```

If the result is `degraded` or `failed`, the calling workflow exits cleanly rather than operating on a potentially broken system.

**Layer 3 — Automatic Rollback (Corrective):** When health checks detect failure, Cloudflare automatically reverts to the previous known-good deployment. No human intervention required. The rollback target is always the last version that passed health checks — not just the previous version.

**Layer 4 — Cooldown Logic (Circuit Breaker):** The layer most teams skip. After a rollback, the system enters a cooldown window that blocks re-deployment. This prevents the cascade scenario: bad deploy → rollback → re-deploy (same bad code) → rollback → re-deploy. Cooldown state is tracked in `system_flags` and must expire or be manually cleared before the next deploy.

---

## Key Insight

**Each layer exists because the previous layer has a blind spot.**

Validation catches known-bad patterns, but can't predict emergent failures in production. Health checks detect production problems, but only after they've already happened. Rollback fixes detected problems, but can trigger new ones if the system redeploys too quickly. Cooldown prevents cascading failures, but adds latency to recovery.

No single layer is sufficient. The framework works because each layer compensates for a different failure mode:

| Layer | Catches | Misses |
|-------|---------|--------|
| Validation | Known-bad patterns | Novel failure modes |
| Health Check | Post-deploy degradation | Pre-deploy issues |
| Rollback | Detected failures | Silent degradation |
| Cooldown | Cascading re-deploys | Legitimate urgent fixes |

This is the same principle behind network security (firewall → IDS → endpoint protection → incident response) and physical security (locks → cameras → guards → alarms). Defense-in-depth isn't about building one perfect wall. It's about building four imperfect walls that cover each other's gaps.

---

## Results

| Metric | Value |
|--------|-------|
| Protection layers | 4 |
| Production n8n workflows | 2 (Agent 5: Website Deployment + Agent 14: Health Monitor) |
| Total nodes | 24 (expanded from 15 when protection layers were added) |
| Health check URLs | 6 (post-deploy verification) |
| Customer-visible outages | 0 |
| Health status states | 3 (`healthy`, `degraded`, `failed`) |
| Rollback mechanism | Cloudflare automatic revert via webhook |
| Cooldown window | 14 days (blocks re-deploy after rollback, requires manual clear) |
| Runtime state tracking | `system_flags` table |

---

## Why It Matters

Layered security controls where each layer catches what the previous missed. Pre-deploy validation = preventive controls. Health checks = detective controls. Automatic rollback = corrective controls. Cooldown = circuit breakers. Classic defense-in-depth applied to deployment infrastructure — the same pattern used in network security (firewall → IDS → endpoint protection → incident response) and physical security (locks → cameras → guards → alarms).

The practical implication: an autonomous system that runs unattended for months stays within safe operational bounds not because every failure is predicted, but because each layer is designed to compensate for a specific failure mode the previous layer can't see.

---

## Design Decisions

**Why an enum instead of a boolean for health?** A binary healthy/unhealthy misses the middle state. `degraded` means "still serving traffic but something's wrong" — it needs alerting, not rollback. The three-state enum gives operators time to investigate before automatic corrective action fires.

**Why cooldown instead of deploy rate limiting?** Rate limiting slows everything down equally. Cooldown specifically blocks re-deploys after a *failure*, which is the only time cascading is a risk. Normal deploys aren't throttled.

**Why `system_flags` instead of a deployment-specific table?** `system_flags` serves as a general-purpose runtime state table across the entire system. Health status, cooldown state, and feature flags all live in one place. One table to query for "can I deploy right now?"

---

## Built With

- **Infrastructure:** Cloudflare (hosting + automatic rollback)
- **Database:** Supabase PostgreSQL (`system_flags` for runtime state)
- **Automation:** n8n (deployment workflows + health check scheduling)
- **Monitoring:** Health status enum with alert triggers

Part of [HAIOS](https://mrminor-dev.github.io) — a Human-AI Operating System in production since October 2024.

---

## License

MIT

## Author

**Jordan Waxman** — [mrminor-dev.github.io](https://mrminor-dev.github.io)

14 years operations leadership — building human-AI infrastructure since 2025. The intersection is the moat.
