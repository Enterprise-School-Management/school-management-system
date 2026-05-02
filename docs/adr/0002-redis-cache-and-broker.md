# ADR-0002 — Redis Dual-Use: Cache and Message Broker

**Date:** 2026-05-02  
**Status:** Accepted  
**Deciders:** system_designer

---

## Context

Redis appears twice in the technology stack:

1. **Cache** — Django's cache backend for session data, query results, and rate-limit counters.
2. **Message broker** — Celery's broker backend for the async task queue (email, SMS, grading, analytics, backups).

Running both workloads on a single Redis instance is simple but creates a coupling risk: if the message broker queue depth spikes (e.g. a bulk notification job), it competes for memory and CPU with the cache, potentially evicting cache entries or slowing cache reads. Conversely, a misconfigured cache with no eviction policy could fill memory and block the broker.

The deployment target — an on-premises school server — is constrained in RAM and cannot run additional Redis instances during Phase 1.

## Decision

**Use a single Redis instance for both workloads in Phase 1**, separated by **database index**:

- `redis://redis:6379/0` — Django cache.
- `redis://redis:6379/1` — Celery broker and result backend.

Both databases live in the same process and share memory, but key namespaces do not collide. This provides logical separation at zero operational cost.

**Separation trigger for Phase 2+**: Move to two Redis instances (or two Redis clusters) when **any one** of the following conditions is met:

| Trigger | Threshold |
|---------|-----------|
| Celery queue depth | Regularly exceeds 5 000 pending tasks |
| Redis memory usage | Consistently above 70 % of available RAM |
| Cache eviction rate | > 5 % per minute (cache hit rate drops below 90 %) |
| School count | Installation serves > 20 concurrent schools |

When the trigger fires, split into:
- `redis-cache` — Django cache only; `maxmemory-policy allkeys-lru`.
- `redis-broker` — Celery broker + result backend; `maxmemory-policy noeviction`.

Both instances are still self-hosted. Cloud Redis (ElastiCache, Redis Cloud) is deferred until the hosted multi-school platform phase.

## Alternatives considered

| Alternative | Why rejected |
|-------------|--------------|
| Two Redis instances from day one | Doubles Docker Compose complexity and RAM consumption on constrained school hardware in Phase 1. Premature for a single-school deployment. |
| Use Django's database cache (Postgres) | Adds write load to the primary database for every cache read/miss. Worse performance than Redis and negates the benefit of caching. |
| RabbitMQ as broker, Redis as cache only | RabbitMQ is operationally heavier, adds another service, and requires a different skill set. Redis + Celery is well-understood by the team. |

## Consequences

### Positive

- Single Docker service to maintain in Phase 1.
- Database index separation prevents key collisions without code changes to the Django cache or Celery configuration.
- Clear, measurable thresholds make the split decision objective and observable.

### Negative / trade-offs

- Cache evictions under high broker load are possible if `maxmemory-policy` is set to `allkeys-lru`. Mitigation: configure a modest `maxmemory` limit and monitor in production.
- A Redis crash takes down both cache and broker simultaneously. Mitigation: Redis persistence (`appendonly yes`) and the school server UPS reduce this risk.

### Risks and mitigations

| Risk | Mitigation |
|------|-----------|
| Memory exhaustion takes down both cache and broker | Set `maxmemory` in `redis.conf`; alert when usage exceeds 60 %. |
| `noeviction` policy blocks cache writes when broker fills memory | Database index separation ensures Celery's `redis://…/1` and Django's `redis://…/0` have independent key spaces; set `maxmemory` with `allkeys-lru` on the shared instance during Phase 1. |
| Future split requires touching Django settings and Celery config | Both are environment-variable-driven (`CACHE_URL`, `CELERY_BROKER_URL`). Changing the URL is the only code change needed. |

## References

- [01-Architecture.md §9 — Technology Stack](../01-Architecture.md)
- [10-DeploymentGuide.md](../10-DeploymentGuide.md)
- [Celery Redis broker docs](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/redis.html)
