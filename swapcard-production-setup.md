# Production Readiness

A focused plan for taking the Swapcard ↔ ZenSpace bridge from "working" to
"production-ready" — **without adding heavy infrastructure**. The app's
workload (low write rate, single operator, secrets-in-DB) is small, so the
goal here is reliability and safety, not scale.

For what the app does and how it works, see [DOCS.md](DOCS.md). This file
is purely about the operational/production posture.

---

## 1. Guiding principle: lightweight

Every recommendation in this doc was filtered through one rule:

> Add the smallest thing that solves the problem. Avoid managed services,
> background workers, and dependencies that bring their own ops surface.

Net additions if you follow the full plan below:

- **0 new npm packages**
- **1 binary added to the Docker image** (Litestream)
- **~80 lines of new code total**

Things deliberately **not** in this plan: Sentry, Prometheus, Redis,
message queues, Kubernetes, managed Postgres, multi-region failover.
These are real tools but they don't earn their weight at this app's scale.

---

## 2. Current state assessment

### What's already solid

| Area | Where | Notes |
|---|---|---|
| HMAC webhook verification | [src/lib/hmac.js](src/lib/hmac.js) | Constant-time compare, accepts hex/base64/base64url. |
| Idempotent migrations | [src/db.js:21-50](src/db.js#L21-L50) | `_migrations` tracking table, transactional apply. |
| Raw-body capture for HMAC | [src/server.js:38-50](src/server.js#L38-L50) | Captures exact bytes Swapcard sent. |
| Auth gate as global hook | [src/server.js:84-94](src/server.js#L84-L94) | Public prefixes explicit, redirects to `/login?next=…`. |
| Cookie security attrs | [src/lib/auth.js:99-107](src/lib/auth.js#L99-L107) | `HttpOnly`, `SameSite=Lax`, `Secure` conditional on HTTPS. |
| Per-mapping HMAC secrets | [src/db.js:55-87](src/db.js#L55-L87) | One bridge can safely host multiple events. |
| Operational runbook | [DOCS.md §13](DOCS.md) | Symptom → cause → fix table. |

### Real gaps (ordered by blast radius)

| # | Gap | Risk |
|---|---|---|
| 1 | Persistent volume can detach silently on Fly machine recreation | **All mappings + generated HMAC secrets lost.** Operators must redo every Swapcard webhook wiring. |
| 2 | `SESSION_SECRET` defaults to a placeholder ([auth.js:8](src/lib/auth.js#L8)) | App boots silently with an insecure default. Any attacker who reads the source can forge sessions. |
| 3 | `/webhooks/zencore` is unsigned (relies on URL secrecy) | Anyone with the URL can flip `connection_status`. Flagged in [DOCS.md §9](DOCS.md). |
| 4 | No `/healthz` route despite the prefix being allowlisted ([server.js:81](src/server.js#L81)) | Fly/load-balancer health checks can't distinguish "process up" from "DB readable." |
| 5 | No automated tests — `package.json` has no `test` script | HMAC verifier, event-ID normalization, and mapping resolver are exactly the code that breaks silently on refactor. |
| 6 | Single hardcoded admin credential, no login rate-limit | Brute-forceable if the URL is discovered. Tolerable for single-operator use; revisit if multi-operator is ever needed. |

---

## 3. Database recommendation: **stay on SQLite + add Litestream**

### Decision

| Option | Verdict | Why |
|---|---|---|
| **SQLite + Litestream → S3/R2** | ⭐ **Recommended** | Zero code changes. Continuous WAL replication to object storage. RPO ≈ seconds, RTO = `litestream restore`. ~$0.10/mo for storage. Single-writer is a fit, not a flaw. |
| Managed Postgres (Neon/Supabase) | Overkill | Generous free tier, but you'd rewrite [db.js](src/db.js), lose `better-sqlite3`'s synchronous speed on the webhook hot path, and gain nothing this workload needs. |
| Turso / libSQL | Reasonable plan B | SQLite-compatible managed service. Adds a network hop for every read — bad for the per-webhook lookup. |
| Fly volume snapshots only | Insufficient | Fly takes daily snapshots (5-day retention). Worst-case RPO is 24 hours. Acceptable for a hobby project, not production. |

### Why SQLite stays

- The mapping + synced_booking tables together are tiny (kilobytes per event).
- Reads dominate — every webhook does a primary-key lookup. `better-sqlite3` is **synchronous and in-process**: zero network, zero serialization.
- The schema is stable. There's no migration storm coming.
- The secrets live in the DB. Backup is the real problem to solve — not the engine.

---

## 4. Server recommendation: **stay on Fly.io**

### Decision

| Option | Verdict | Notes |
|---|---|---|
| **Fly.io** (current) | ⭐ **Recommended** | Already configured. Single machine + attached volume is exactly what this app needs. |
| AWS Lightsail | ✅ Equivalent | Closest AWS analog to Fly. ~$5/mo VPS + attached disk. Lightweight, but more setup steps (TLS, Caddy/Nginx, DNS) for no real gain. |
| AWS EC2 + EBS | ✅ Works | Cheaper at scale, but you own patching, systemd, TLS renewal. More ops than Fly. |
| AWS ECS Fargate + EFS | ⚠️ Works, ugly | EFS adds latency to every SQLite read (NFS, not local disk). Don't do this with SQLite. |
| AWS App Runner | ❌ Bad fit | Stateless by design — no persistent local volume. Would force a SQLite → Postgres rewrite just to fit the platform. |
| Railway / Render | ✅ Equivalent to Fly | No technical advantage for this app. |
| Hetzner / DigitalOcean droplet | ✅ Cheapest | $4-5/mo, more control, more ops. Only worth it if you're running other services on the same VM. |

### Why Fly stays

- The DB-wipe issue we diagnosed (volume not actually attached after machine recreation) is a **configuration** problem, not a Fly platform problem. The same misconfig would bite on Lightsail too.
- Fly is purpose-built for the "one container + one volume + webhook receiver" pattern.
- Migration cost is non-zero. There has to be a reason — cost, org mandate, growth — and there isn't yet.

### When to revisit

Move off Fly if any of these become true:
- The app needs to scale to multiple machines (SQLite + local disk caps you at one writer).
- An org-level AWS mandate appears.
- Fly's free tier / pricing changes in a way that hurts.

---

## 5. The plan (priority order)

Each step lists rough effort, code changes, and what risk it removes.

### Step 1: Verify the Fly volume is actually attached (10 min, 0 code)

The single highest-leverage thing. Without this, every other step is undermined.

```bash
flyctl volumes list -a swapcard-bridge
flyctl machines list -a swapcard-bridge
flyctl ssh console -a swapcard-bridge -C "df -h /data && ls -la /data"
flyctl logs -a swapcard-bridge | Select-String "databasePath|DB has no mappings"
```

What to look for:
- One volume named `data`, attached to exactly one machine.
- `/data` shows up in `df -h` as its own filesystem (not part of the rootfs overlay).
- Boot log shows `mappingRows: N` where N matches reality.

If the volume isn't attached, the fix:

```bash
flyctl volumes create data --region bom --size 1 -a swapcard-bridge
flyctl deploy -a swapcard-bridge
flyctl scale count 1 -a swapcard-bridge
```

**Risk removed:** silent total data loss on machine recreation.

---

### Step 2: Refuse to boot on default secrets in production (10 min, ~15 lines)

Add to the top of [src/server.js](src/server.js):

```js
if (process.env.NODE_ENV === 'production') {
  const insecure = [];
  if (!process.env.SESSION_SECRET || process.env.SESSION_SECRET.startsWith('change-me')) {
    insecure.push('SESSION_SECRET');
  }
  if (process.env.ADMIN_PASSWORD === 'bridge2026' || !process.env.ADMIN_PASSWORD) {
    insecure.push('ADMIN_PASSWORD');
  }
  if (insecure.length) {
    console.error(`Refusing to start: insecure defaults for ${insecure.join(', ')}.`);
    process.exit(1);
  }
}
```

**Risk removed:** silent boot with forgeable session tokens.

---

### Step 3: Add a real `/healthz` route (5 min, ~10 lines)

The prefix is already allowlisted at [src/server.js:81](src/server.js#L81) — there's just no handler. Add one that also probes the DB:

```js
app.get('/healthz', async (_req, reply) => {
  try {
    getDb().prepare('select 1').get();
    return reply.send({ ok: true });
  } catch (err) {
    return reply.code(503).send({ ok: false, error: err.message });
  }
});
```

Then in [fly.toml](fly.toml), add a real health check so Fly restarts on DB failure:

```toml
[[http_service.checks]]
  grace_period = "10s"
  interval = "30s"
  method = "GET"
  path = "/healthz"
  timeout = "5s"
```

**Risk removed:** zombie processes that respond to TCP but can't actually serve.

---

### Step 4: Sign `/webhooks/zencore` (30 min – 1 day, depends on ZenCore)

Currently relies on URL secrecy (see [DOCS.md §9](DOCS.md)). Two options:

- **Minimum:** add a path-secret — e.g. `/webhooks/zencore/<random-32-bytes>` — and verify on receipt. Lightweight, but secrets in URLs end up in logs.
- **Proper:** ask the ZenCore team to add HMAC signing (reuse the verifier in [src/lib/hmac.js](src/lib/hmac.js)). Coordinate the secret out-of-band.

**Risk removed:** anyone with the webhook URL can forge approval-status updates.

---

### Step 5: Add Litestream backup (1-2 hours, Docker change only)

Modify [Dockerfile](Dockerfile) to install Litestream and replicate to object storage:

```dockerfile
# Add after the existing apt-get block
ADD https://github.com/benbjohnson/litestream/releases/download/v0.3.13/litestream-v0.3.13-linux-amd64.tar.gz /tmp/
RUN tar -xzf /tmp/litestream-v0.3.13-linux-amd64.tar.gz -C /usr/local/bin/ && rm /tmp/litestream*

COPY litestream.yml /etc/litestream.yml

# Replace CMD with:
CMD ["litestream", "replicate", "-exec", "node src/server.js"]
```

A minimal `litestream.yml`:

```yaml
dbs:
  - path: /data/bridge.db
    replicas:
      - url: s3://swapcard-bridge-backup/bridge.db
        retention: 168h    # 7 days
        snapshot-interval: 24h
```

Pick a bucket: **Cloudflare R2** (no egress fees, free tier covers this) or **AWS S3** (in-region if you ever move to AWS). Set credentials via Fly secrets:

```bash
flyctl secrets set \
  LITESTREAM_ACCESS_KEY_ID="…" \
  LITESTREAM_SECRET_ACCESS_KEY="…" \
  -a swapcard-bridge
```

Recovery is a single command on a fresh machine:

```bash
litestream restore -o /data/bridge.db s3://swapcard-bridge-backup/bridge.db
```

**Risk removed:** total data loss when the Fly volume dies. RPO drops from 24h (volume snapshots) to seconds.

---

### Step 6 (optional): Three tests for the HMAC verifier (1-2 hours, 1 devDep)

The smallest possible test surface that meaningfully de-risks the codebase. Add Vitest as a devDep and write three cases:

- Valid signature in hex → accepts.
- Valid signature in base64url with `sha256=` prefix → accepts.
- Tampered body or wrong secret → rejects.

If GitHub is the source of truth, add a one-file CI workflow that runs `npm test` on every push. No coverage thresholds, no matrix builds — just a green check on PRs.

Skip this whole step if you're the only committer and prefer to keep the dep list at exactly its current size.

**Risk removed:** silent regression of the HMAC verifier on future refactors.

---

## 6. What we're explicitly NOT doing

| Skipped | Why |
|---|---|
| Sentry / external error tracking | Fly's stdout logs are enough at this scale. Re-evaluate if you ever miss a real incident because of log volume. |
| Prometheus / Grafana / OTel | Overkill for one operator + a handful of events. |
| Login rate-limiting | Fly's edge already absorbs the worst garbage. Reconsider if the URL gets leaked or you go multi-operator. |
| Multi-region / HA | SQLite + local disk caps at one writer by design. Single-region is fine; the app is recoverable in minutes from Litestream. |
| Migration to Postgres | The workload doesn't justify the rewrite cost. Re-evaluate only if SQLite hits a real limit. |
| Update-existing-booking flow ([DOCS.md §14](DOCS.md)) | Feature gap, not a production-readiness gap. Track it separately. |

---

## 7. Recovery runbook

What to do if production goes wrong. Add this to the team wiki.

### Symptom: DB suddenly empty

1. `flyctl ssh console -a swapcard-bridge -C "ls -la /data"` — is `bridge.db` present?
2. If absent: `flyctl volumes list -a swapcard-bridge` — is the volume attached to the running machine?
3. If volume is fine but DB is missing/corrupt:
   ```bash
   flyctl ssh console -a swapcard-bridge
   litestream restore -o /data/bridge.db s3://swapcard-bridge-backup/bridge.db
   exit
   flyctl apps restart swapcard-bridge
   ```
4. Verify in logs: `databasePath: /data/bridge.db, mappingRows: N` where N > 0.

### Symptom: webhooks failing with `signature did not verify`

See [DOCS.md §13](DOCS.md) — re-paste the bridge's secret into Swapcard Studio.

### Symptom: `/healthz` returns 503

The DB file is missing or unreadable. Same as "DB suddenly empty" above.

### Symptom: process won't start with "Refusing to start: insecure defaults"

You forgot a Fly secret. Re-run the `flyctl secrets set` block from [DOCS.md §4](DOCS.md).

---

## 8. Cost summary

If you do everything in this plan:

| Item | Cost |
|---|---|
| Fly.io (shared-cpu-1x, 256MB, 1GB volume) | ~$2-3/mo |
| Cloudflare R2 backup storage (≪ 1 GB) | ~$0/mo (free tier) |
| **Total** | **~$2-3/mo** |

No managed-DB bill, no APM bill, no logging-vendor bill.

---

## 9. Open questions

Things worth deciding before declaring "v1.0":

- **Webhook URL secrecy for ZenCore** — is the ZenCore team willing to add HMAC signing, or do we ship the path-secret workaround?
- **Operator handoff** — when the current operator leaves, who rotates `ADMIN_PASSWORD` and the per-mapping HMAC secrets? (Worth a 5-line section in the team runbook.)
- **`meeting_update` support** — yes/no. If yes, [DOCS.md §14](DOCS.md) has the spec.
- **Multi-operator?** If yes, single-credential auth becomes a real blocker and step 6 of section 2 turns into "swap to a real auth provider."
