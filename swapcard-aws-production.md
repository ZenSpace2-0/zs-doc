# AWS Production Deployment

How to run the Swapcard ↔ ZenSpace bridge on AWS **without giving up the
lightweight posture** described in [PRODUCTION.md](PRODUCTION.md).

This doc is the AWS-specific counterpart to [PRODUCTION.md](PRODUCTION.md).
The app-level hardening (refuse-to-boot on default secrets, `/healthz`,
signing `/webhooks/zencore`, etc.) is platform-agnostic and lives in that
doc — follow it regardless of where you deploy.

For what the app does and how it works, see [DOCS.md](DOCS.md).

---

## 1. TL;DR

| | |
|---|---|
| **Compute** | AWS Lightsail — `nano_3_0` ($5/mo) or `micro_3_0` ($10/mo) instance |
| **Storage** | Lightsail block-storage disk, 20 GB, mounted at `/data` |
| **Backup** | Litestream → S3 (one bucket, in the same region as compute) |
| **DNS + TLS** | Route 53 + Caddy on the instance (auto Let's Encrypt) |
| **Region** | `ap-south-1` (Mumbai) — matches Fly's `bom`, closest to Swapcard/ZenCore traffic |
| **All-in cost** | **~$6-12/mo** (Lightsail) + ~$0.10/mo (S3) |
| **Migration time** | ~1-2 hours end-to-end |

If your only reason for moving is "AWS is what we use" — this is the path.
If you don't have an org reason, **stay on Fly** ([PRODUCTION.md §4](PRODUCTION.md)).

---

## 2. Why Lightsail (and not the fancier AWS services)

| Option | Verdict | Reason |
|---|---|---|
| **Lightsail** | ⭐ Recommended | Flat pricing, attached SSD as a real block device, runs Docker, supports static IP. Closest 1:1 mapping to the current Fly setup. |
| EC2 + EBS | ✅ Works | Cheaper at scale (reserved instances), but you own everything: VPC, security groups, AMI updates, patching cadence. Not lightweight. |
| ECS Fargate + EFS | ❌ Don't | EFS is NFS. SQLite over NFS works but adds latency to every `prepare`/`get` call on the webhook hot path. You'd lose `better-sqlite3`'s biggest advantage. |
| App Runner | ❌ Don't | No persistent local volume. Would force a SQLite → managed-DB rewrite just to fit the platform. |
| Elastic Beanstalk | ❌ Don't | Legacy. Slower deploys, fewer guarantees, more YAML. |
| EKS / Kubernetes | ❌ Don't | The app is one container. K8s is two orders of magnitude more complexity than it needs. |

**Lightweight rule applied:** Lightsail is the only AWS service whose
operational surface is comparable to Fly's. Anything else adds VPCs, IAM
policies, security groups, and CloudWatch dashboards to maintain — none of
which serve this app's traffic.

---

## 3. Architecture

```
┌──────────────┐                  ┌────────────────────────────┐
│   Swapcard   │ ───webhook────► │     Lightsail instance     │
│  (event mtg) │   (HTTPS)        │  ┌──────────────────────┐  │
└──────────────┘                  │  │  Caddy (TLS, :443)   │  │
                                  │  │       ↓              │  │
                                  │  │  Docker container    │  │
                                  │  │  (node + bridge)     │  │
                                  │  │       ↓              │  │
                                  │  │  /data/bridge.db ────┼──┼─► litestream ──► S3 (ap-south-1)
                                  │  └──────────────────────┘  │
                                  │   ↑ attached block disk    │
                                  └────────────────────────────┘
                                            ▲
                                            │
                                  Route 53 (A record → static IP)
```

One instance. One disk. One bucket. No load balancer, no NAT gateway, no
ECR, no CloudWatch agent. Static IP attached to the instance so DNS
doesn't have to change if the instance is recreated.

---

## 4. Step-by-step deploy

### Prerequisites

- AWS account with billing alerts already configured.
- AWS CLI v2 installed locally (`aws --version`).
- A domain you control (or use the Lightsail `*.amazonlightsail.com` URL with no TLS — not recommended).
- The bridge image either pushed to a registry, or built on the instance from the repo.

### Step 1: Create the Lightsail instance (5 min)

```bash
aws lightsail create-instances \
  --region ap-south-1 \
  --instance-names swapcard-bridge \
  --availability-zone ap-south-1a \
  --blueprint-id ubuntu_22_04 \
  --bundle-id nano_3_0 \
  --tags key=app,value=swapcard-bridge
```

Wait until status = `running`:

```bash
aws lightsail get-instance --region ap-south-1 --instance-name swapcard-bridge \
  --query 'instance.state.name'
```

### Step 2: Attach a static IP (2 min)

```bash
aws lightsail allocate-static-ip --region ap-south-1 --static-ip-name swapcard-bridge-ip
aws lightsail attach-static-ip   --region ap-south-1 --static-ip-name swapcard-bridge-ip \
  --instance-name swapcard-bridge
```

Note the IP — you'll point DNS at it.

### Step 3: Create + attach the data disk (3 min)

The instance's root disk is fine for the OS, but bridge data lives on a separately-attached disk so it survives instance recreation:

```bash
aws lightsail create-disk --region ap-south-1 \
  --disk-name swapcard-bridge-data \
  --availability-zone ap-south-1a \
  --size-in-gb 20

aws lightsail attach-disk --region ap-south-1 \
  --disk-name swapcard-bridge-data \
  --instance-name swapcard-bridge \
  --disk-path /dev/xvdf
```

### Step 4: Open the firewall (1 min)

```bash
aws lightsail put-instance-public-ports --region ap-south-1 \
  --instance-name swapcard-bridge \
  --port-infos \
    fromPort=22,toPort=22,protocol=TCP \
    fromPort=80,toPort=80,protocol=TCP \
    fromPort=443,toPort=443,protocol=TCP
```

SSH is open for setup; you can close port 22 to a fixed CIDR later.

### Step 5: Mount the data disk on the instance (5 min)

SSH in (`aws lightsail get-instance-access-details ...` or the Lightsail console's browser SSH), then:

```bash
sudo mkfs.ext4 /dev/xvdf
sudo mkdir -p /data
echo '/dev/xvdf /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a
df -h /data    # confirm it's mounted and shows ~20G
```

### Step 6: Install Docker + Caddy (5 min)

```bash
# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker

# Caddy (for TLS termination)
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

### Step 7: Configure Caddy as a reverse proxy (2 min)

Replace `/etc/caddy/Caddyfile` with:

```
bridge.yourdomain.com {
    reverse_proxy localhost:8080
}
```

```bash
sudo systemctl reload caddy
```

Caddy auto-provisions Let's Encrypt certs on first request. Make sure DNS already resolves to the static IP first (next step).

### Step 8: Point DNS at the static IP (5 min, via Route 53 or wherever you host DNS)

```bash
# Route 53 example
aws route53 change-resource-record-sets --hosted-zone-id ZXXXXXXXXXXXXX \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "bridge.yourdomain.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "YOUR.STATIC.IP.HERE"}]
      }
    }]
  }'
```

Wait for `dig bridge.yourdomain.com +short` to return the IP before continuing.

### Step 9: Set up the S3 backup bucket (3 min)

```bash
aws s3api create-bucket --bucket swapcard-bridge-backup-prod \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

aws s3api put-bucket-versioning --bucket swapcard-bridge-backup-prod \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block --bucket swapcard-bridge-backup-prod \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Create an IAM user with **only** access to this bucket — never put the
root account's keys on the instance:

```bash
aws iam create-user --user-name swapcard-bridge-litestream
aws iam create-access-key --user-name swapcard-bridge-litestream
```

Attach a minimum-permission inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject", "s3:PutObject", "s3:DeleteObject",
      "s3:ListBucket", "s3:GetBucketLocation"
    ],
    "Resource": [
      "arn:aws:s3:::swapcard-bridge-backup-prod",
      "arn:aws:s3:::swapcard-bridge-backup-prod/*"
    ]
  }]
}
```

```bash
aws iam put-user-policy --user-name swapcard-bridge-litestream \
  --policy-name litestream-bucket-rw \
  --policy-document file://policy.json
```

Save the access key + secret — they go in `.env` on the instance.

### Step 10: Run the bridge container (5 min)

On the instance:

```bash
mkdir -p ~/swapcard-bridge && cd ~/swapcard-bridge

# .env
cat > .env <<'EOF'
NODE_ENV=production
PORT=8080
DATABASE_PATH=/data/bridge.db
PUBLIC_URL=https://bridge.yourdomain.com

ADMIN_EMAIL=ops@yourorg.com
ADMIN_PASSWORD=<long random>
SESSION_SECRET=<openssl rand -hex 32>

LITESTREAM_ACCESS_KEY_ID=<IAM access key>
LITESTREAM_SECRET_ACCESS_KEY=<IAM secret>
EOF

chmod 600 .env
```

Pull and run the image (adapt to wherever you push it):

```bash
docker run -d \
  --name swapcard-bridge \
  --restart=always \
  --env-file ~/swapcard-bridge/.env \
  -p 127.0.0.1:8080:8080 \
  -v /data:/data \
  swapcard-bridge:latest
```

Verify:

```bash
docker logs swapcard-bridge | tail -20
curl https://bridge.yourdomain.com/healthz
```

You should see `{"ok":true}` and the boot log line with `databasePath: /data/bridge.db, mappingRows: N`.

---

## 5. Litestream backup (the only piece worth its weight)

If you've already added Litestream per [PRODUCTION.md §5 Step 5](PRODUCTION.md), the only change for AWS is the replica URL:

```yaml
# litestream.yml (baked into the image or mounted at /etc/litestream.yml)
dbs:
  - path: /data/bridge.db
    replicas:
      - url: s3://swapcard-bridge-backup-prod/bridge.db
        region: ap-south-1
        retention: 168h
        snapshot-interval: 24h
```

The container's `CMD` becomes `litestream replicate -exec "node src/server.js"`.

Recovery on a fresh instance:

```bash
litestream restore -o /data/bridge.db s3://swapcard-bridge-backup-prod/bridge.db
docker restart swapcard-bridge
```

**RPO ≈ seconds. RTO ≈ minutes.** This is the single biggest reliability
win in the entire plan.

---

## 6. CloudWatch — what to keep, what to skip

The lightweight rule applies: **don't install the CloudWatch agent unless you have a real reason.** Lightsail already shows CPU/network in its console; that covers 90% of the questions you'd ask.

| Signal | Where it lives by default | Install agent? |
|---|---|---|
| Instance CPU / network | Lightsail console (free) | No |
| App logs | `docker logs swapcard-bridge` | No — they're already structured (pino JSON). Pipe to CloudWatch only if you need search across N instances, which you don't. |
| `/healthz` failures | Caddy access log + Lightsail console | No |
| S3 backup freshness | One CloudWatch alarm on bucket `LastWriteTime` | **Yes — this one** |

Set up exactly one alarm: "alert if `swapcard-bridge-backup-prod` hasn't had a write in 24 hours." That tells you if Litestream is broken. Skip everything else until you actually need it.

```bash
aws cloudwatch put-metric-alarm \
  --region ap-south-1 \
  --alarm-name swapcard-bridge-backup-stale \
  --namespace AWS/S3 \
  --metric-name NumberOfObjects \
  --statistic Average \
  --dimensions Name=BucketName,Value=swapcard-bridge-backup-prod Name=StorageType,Value=AllStorageTypes \
  --period 86400 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --alarm-actions <your-SNS-topic-arn>
```

---

## 7. Recovery runbook (AWS-specific bits)

For app-level symptoms see [PRODUCTION.md §7](PRODUCTION.md). AWS-only additions:

### Symptom: instance won't boot or is unreachable

1. Check Lightsail console → Networking → static IP still attached?
2. Check Networking → firewall ports 80/443 still open?
3. If the instance is corrupted, recreate it: snapshot the data disk first (`aws lightsail create-disk-snapshot`), spin a new instance, attach the existing data disk to the new instance, repeat steps 5-10. No data loss because `/data` is a separate disk.

### Symptom: Litestream is silent — no recent S3 writes

1. `docker logs swapcard-bridge | grep -i litestream` — what's it saying?
2. Verify the IAM user's access key isn't disabled: `aws iam list-access-keys --user-name swapcard-bridge-litestream`
3. Try a manual sync: `docker exec swapcard-bridge litestream snapshots /data/bridge.db`
4. Worst case: rotate IAM keys, update `.env`, `docker restart swapcard-bridge`.

### Symptom: TLS cert isn't renewing

Caddy logs: `sudo journalctl -u caddy -f`. Most common cause: DNS no longer points at the instance, or port 80 was closed (Let's Encrypt HTTP challenge needs it).

---

## 8. Cost summary

| Item | Monthly |
|---|---|
| Lightsail `nano_3_0` instance (1 vCPU, 512MB) | $5 |
| Lightsail `nano_3_0` upgrade to `micro_3_0` (1 vCPU, 1GB) — optional | +$5 |
| Lightsail block-storage disk, 20 GB | $2 |
| Lightsail static IP (attached) | Free |
| S3 storage (backup, ≪ 1 GB) | < $0.05 |
| S3 PUT requests (Litestream WAL writes) | < $0.10 |
| Route 53 hosted zone | $0.50 |
| CloudWatch alarm (1 alarm) | $0.10 |
| **Total** | **~$8/mo on `nano`, ~$13/mo on `micro`** |

About 4× what Fly costs for this workload. The premium buys you "we run on AWS." Decide if that's worth it.

---

## 9. What we're explicitly NOT doing on AWS

| Skipped | Why |
|---|---|
| ECR (private container registry) | Build on the instance or pull from Docker Hub. One image, infrequent updates. |
| Application Load Balancer | One instance, one IP. ALB is $20/mo of pure overhead. |
| RDS / Aurora | The whole point of staying lightweight is keeping SQLite. See [PRODUCTION.md §3](PRODUCTION.md). |
| ECS / EKS | One container. Orchestration is unjustified. |
| VPC peering, NAT gateway, private subnets | Lightsail handles networking. Don't bring an enterprise VPC for a single webhook receiver. |
| AWS WAF | Caddy is already terminating TLS; rate-limit at the app if you need to. WAF is $5/mo + per-request fees for no real benefit here. |
| Secrets Manager / Parameter Store | `.env` on a single instance is fine. Move here only if you go multi-instance. |
| CloudWatch Logs (full ingest) | stdout + `docker logs` is enough. Re-evaluate only if you miss a real incident because of it. |

---

## 10. Migrating from Fly to AWS

If you've been running on Fly and want to move:

1. **Run both in parallel for at least 24 hours.** Keep Fly receiving webhooks until the AWS instance is verified healthy.
2. **Snapshot the current Fly volume** before touching anything:
   ```bash
   flyctl ssh console -a swapcard-bridge -C "cp /data/bridge.db /tmp/bridge.db.snapshot"
   flyctl ssh sftp shell -a swapcard-bridge
   # get /tmp/bridge.db.snapshot to your laptop
   ```
3. **Upload the snapshot to the new instance:**
   ```bash
   scp bridge.db.snapshot ubuntu@<lightsail-ip>:/data/bridge.db
   docker restart swapcard-bridge
   ```
4. **Verify mapping count matches** between Fly and the AWS instance (boot log shows `mappingRows: N`).
5. **Switch DNS** (or update Swapcard webhook URLs row-by-row to point at the new host).
6. **Wait 24 hours.** Watch logs on both sides.
7. **Decommission Fly** only after a full day of clean operation on AWS.

---

## 11. Open questions

Decisions that need to happen before declaring "production on AWS":

- **Which AWS account?** Shared org account vs. a dedicated sub-account for the bridge. Strongly prefer a dedicated sub-account with a billing alert.
- **Who has SSH access?** Lightsail uses one key per region by default. If you need multiple operators, add their public keys to `~ubuntu/.ssh/authorized_keys` and rotate the default key out.
- **Disk size growth.** `synced_booking` grows linearly with meetings. 20 GB lasts effectively forever for this workload, but set a CloudWatch alarm on `DiskUtilization > 80%` if you want to be sure.
- **Do you want a staging instance?** Cheapest staging = a second Lightsail `nano` with a separate bucket. Useful before non-trivial deploys.
