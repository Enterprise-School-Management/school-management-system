# 10 — Deployment Guide

Packaging, deployment, configuration, migrations, rollback, and backup operations for the School Management System.

**See also:** [01-Architecture.md](01-Architecture.md) · [07-DatabaseGuide.md](07-DatabaseGuide.md) · [08-SecurityGuide.md](08-SecurityGuide.md) · [04-DevelopmentGuide.md](04-DevelopmentGuide.md)

---

## 1. Deployment Model

The system runs on a **local school server** — one physical or virtual host per school sitting on the school LAN. Internet is a bonus, not a requirement.

Consequences:

- No Kubernetes, no cloud-managed databases at runtime.
- Docker Compose is the canonical packaging.
- UPS-backed hardware is expected (ZESA outages).
- Cloud backup is opportunistic, not on the hot path.

---

## 2. Host Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Storage | 500 GB SSD | 1 TB NVMe + backup disk |
| OS | Ubuntu Server 22.04+ or Debian 12+ | Ubuntu Server 24.04 LTS |
| Network | Gigabit LAN | Gigabit LAN + managed WiFi |
| UPS | Required | 30-min runtime minimum |
| Backup disk | Required | Separate physical device |

---

## 3. Docker Compose Topology

Production compose (`docker-compose.yml`) — **Not yet implemented**; target shape:

```yaml
services:
  web:
    image: school-sms:${VERSION}
    command: gunicorn school_management_system.asgi:application -k uvicorn.workers.UvicornWorker
    depends_on: [db, redis]
    env_file: .env
    ports: ["443:8000"]
  worker:
    image: school-sms:${VERSION}
    command: celery -A school_management_system worker -l info
    depends_on: [db, redis]
    env_file: .env
  beat:
    image: school-sms:${VERSION}
    command: celery -A school_management_system beat -l info
    depends_on: [db, redis]
    env_file: .env
  db:
    image: postgres:16
    volumes: [pg_data:/var/lib/postgresql/data]
    env_file: .env
  redis:
    image: redis:7
    volumes: [redis_data:/data]
  proxy:
    image: nginx:stable
    ports: ["80:80", "443:443"]
    volumes: [./nginx.conf:/etc/nginx/nginx.conf:ro, ./certs:/etc/certs:ro]

volumes:
  pg_data:
  redis_data:
```

---

## 4. Environment Configuration

Every install has its own `.env` derived from `.env.example`. Keys listed in [04-DevelopmentGuide.md §8](04-DevelopmentGuide.md).

### Per-environment values

| Environment | `DJANGO_DEBUG` | `DJANGO_ALLOWED_HOSTS` | Backup target |
|-------------|----------------|------------------------|---------------|
| Local dev | `true` | `127.0.0.1,localhost` | MinIO (local) |
| Staging (on-prem) | `false` | `staging.school.lan` | Separate disk |
| Production (school server) | `false` | `school.lan` | Backup disk + cloud (when online) |

`.env` files are excluded from git. Secrets hygiene: see [08-SecurityGuide.md §5](08-SecurityGuide.md).

---

## 5. Initial Install

1. Provision host per §2.
2. Install Docker and Docker Compose.
3. Copy the release tarball (or pull the image tag) to the host.
4. Copy `.env` from the provisioning vault.
5. Load TLS certs into `./certs/`.
6. Start: `docker compose up -d`.
7. Apply migrations: `docker compose exec web python manage.py migrate`.
8. Seed system data: `docker compose exec web python manage.py seed_system`.
9. Create the first `school_head`: `docker compose exec web python manage.py createsuperuser`.
10. Smoke test: `curl -k https://school.lan/api/v1/health`.

---

## 6. Release Workflow

1. Merge to `main` — CI builds and tags image as `school-sms:<commit-sha>`.
2. Tag a release: `vX.Y.Z` — CI re-tags the image accordingly and publishes release notes to [02-ChangeLog.md](02-ChangeLog.md).
3. Field-upgrade path (on school server):
   ```bash
   # Snapshot first
   ./scripts/backup.sh
   # Pull image (or copy tarball if offline)
   docker compose pull
   # Apply migrations
   docker compose run --rm web python manage.py migrate --check
   docker compose run --rm web python manage.py migrate
   # Cutover
   docker compose up -d
   # Verify
   curl -k https://school.lan/api/v1/health/ready
   ```

For fully offline schools, the release is distributed as a signed tarball on physical media.

---

## 7. Migrations Runbook

- Every release goes through `migrate --check` before `migrate`.
- Migrations that create indexes on tables with > 100k rows use `CONCURRENTLY` and are applied outside peak hours.
- Two-phase breaking changes are executed across two releases; see [07-DatabaseGuide.md §9](07-DatabaseGuide.md).
- Migration failures abort the deploy; prior image stays live. No partial migrations are acceptable.

---

## 8. Rollback

1. Halt new release: `docker compose stop`.
2. Tag the previous image: `docker tag school-sms:<prev> school-sms:current`.
3. If the forward migration was reversible:
   ```bash
   docker compose run --rm web python manage.py migrate <app> <prev-number>
   ```
4. If not reversible: restore from the pre-upgrade backup (§9).
5. Restart: `docker compose up -d`.
6. Incident note in [02-ChangeLog.md](02-ChangeLog.md).

---

## 9. Backup & Restore

### Backup

- **Postgres:** nightly `pg_dump` → backup disk; WAL archived continuously.
- **File storage:** daily rsync of uploads directory → backup disk.
- **Retention:** 30 days local, 90 days cloud (when cloud-backup configured).
- **Encryption:** backups encrypted with per-install key before leaving the server.

```bash
./scripts/backup.sh          # on-demand
# Automated nightly via systemd timer / cron
```

### Restore

1. Stop services: `docker compose stop web worker beat`.
2. Drop / recreate target DB.
3. `psql < <dump>`.
4. Replay WAL up to target timestamp for point-in-time recovery.
5. Restart services.
6. Run the DR assertions from [09-TestingGuide.md §6](09-TestingGuide.md).

RTO: 2 hours. RPO: 15 minutes.

---

## 10. Observability

- **Logs:** stdout → Docker log driver → journald; rotated by logrotate.
- **Metrics:** Django-Prometheus exporter on `/metrics` (**Not yet implemented** — Phase 3).
- **Dashboards:** Grafana as an optional add-on; not required for local school deployment.
- **Alerts:** email on CI failures, backup failures, failed migrations. SMS alerts for production incidents via the notifications channel.

---

## 11. Upgrades & Patching

- Monthly security patching window.
- Quarterly dependency review.
- Container base images rebuilt monthly; the built image is re-scanned by Trivy each build.
- **School-server OS** patched by the deploying operator, coordinated with the school calendar (not during exam windows).

---

## 12. Disaster Recovery Drills

Scheduled quarterly. See [09-TestingGuide.md §6](09-TestingGuide.md) for assertions.

---

## 13. Status

Deployment artefacts (`docker-compose.yml`, `nginx.conf`, `scripts/backup.sh`, `scripts/restore.sh`) are **Not yet implemented**. Scheduled to land in Phase 0 → Phase 1 per [11-Roadmap.md](11-Roadmap.md).
