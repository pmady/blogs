# OCI Object Storage: Automating Cloud Storage with Lifecycle Policies

*A practical guide to automating Oracle Cloud Object Storage — bucket creation, file organization, versioning, and cost-optimized lifecycle management using the OCI CLI.*

---

## Introduction

Every cloud project needs storage. Configuration files, backups, logs, application assets, machine learning datasets — data accumulates fast. Without a strategy, storage costs creep up silently.

Oracle Cloud Infrastructure Object Storage provides a simple, durable, and cost-effective solution for unstructured data. And with 20GB on the Always Free tier, you can start building without any cost.

But the real power is in **automation**. In this post, I will show you how to script the entire Object Storage lifecycle — from bucket creation to automated tier management — using the OCI CLI.

## Object Storage Fundamentals

Before diving into code, let us understand OCI Object Storage concepts:

| Concept | Description |
|---------|-------------|
| **Namespace** | A unique, tenancy-wide identifier (auto-assigned) |
| **Bucket** | A logical container for objects (like a folder) |
| **Object** | A file stored in a bucket (any size, any type) |
| **Storage Tier** | Standard, Infrequent Access, or Archive |
| **Versioning** | Track and recover previous versions of objects |
| **Lifecycle Policy** | Rules that automatically move or delete objects |

The storage tiers are designed for different access patterns:

```
┌─────────────────┐  ┌──────────────────────┐  ┌─────────────────┐
│    Standard      │  │  Infrequent Access   │  │    Archive       │
│                 │  │                      │  │                 │
│  Frequent reads │  │  Monthly access      │  │  Yearly access  │
│  Instant access │  │  Instant access      │  │  4-hour restore │
│  Highest cost   │  │  40% cheaper         │  │  90% cheaper    │
└─────────────────┘  └──────────────────────┘  └─────────────────┘
         │                     ▲                        ▲
         │    Lifecycle rule   │    Lifecycle rule       │
         └─────────────────────┘────────────────────────┘
```

## Always Free Tier

| Resource | Allocation |
|----------|-----------|
| Standard Storage | 20 GB |
| Infrequent Access | 20 GB |
| Outbound Data Transfer | 10 TB/month |
| API Requests | 50,000/month |

## Automating Bucket Creation

### Step 1: Get Your Namespace

Every OCI tenancy has a unique Object Storage namespace:

```bash
NAMESPACE=$(oci os ns get --query 'data' --raw-output)
echo "Namespace: $NAMESPACE"
```

### Step 2: Create a Bucket with Versioning

```bash
oci os bucket create \
    --compartment-id "$COMPARTMENT_ID" \
    --namespace "$NAMESPACE" \
    --name "my-app-data" \
    --storage-tier "Standard" \
    --public-access-type "NoPublicAccess" \
    --versioning "Enabled"
```

Key options:
- `--storage-tier` — Default tier for new objects (`Standard`, `InfrequentAccess`, `Archive`)
- `--public-access-type` — `NoPublicAccess` (default, secure), `ObjectRead` (public download), or `ObjectReadWithoutList`
- `--versioning` — `Enabled` keeps previous versions, `Disabled` overwrites silently

### Step 3: Upload Files with Organization

Object Storage does not have real folders, but it uses prefixes to simulate directory structure:

```bash
# Upload a config file to the "config/" prefix
oci os object put \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --name "config/app-settings.json" \
    --file "./app-settings.json" \
    --content-type "application/json"

# Upload logs to the "logs/" prefix
oci os object put \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --name "logs/2026-03/app.log" \
    --file "./app.log" \
    --content-type "text/plain"

# Upload a backup to the "backups/" prefix
oci os object put \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --name "backups/db-backup-20260301.sql.gz" \
    --file "./db-backup.sql.gz" \
    --content-type "application/gzip"
```

## Lifecycle Policies: Automated Cost Optimization

This is where Object Storage gets really powerful. Lifecycle policies automatically move objects between storage tiers or delete them based on age.

### Example: Three-Tier Lifecycle

```bash
oci os object-lifecycle-policy put \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --items '[
        {
            "name": "move-to-infrequent-after-30-days",
            "action": "INFREQUENT_ACCESS",
            "time-amount": 30,
            "time-unit": "DAYS",
            "is-enabled": true,
            "target": "objects",
            "object-name-filter": {
                "inclusion-prefixes": ["logs/"]
            }
        },
        {
            "name": "archive-after-90-days",
            "action": "ARCHIVE",
            "time-amount": 90,
            "time-unit": "DAYS",
            "is-enabled": true,
            "target": "objects",
            "object-name-filter": {
                "inclusion-prefixes": ["logs/", "backups/"]
            }
        },
        {
            "name": "delete-after-365-days",
            "action": "DELETE",
            "time-amount": 365,
            "time-unit": "DAYS",
            "is-enabled": true,
            "target": "objects",
            "object-name-filter": {
                "inclusion-prefixes": ["logs/"]
            }
        }
    ]'
```

This creates a three-stage pipeline:

```
Day 0-30:  logs/* in Standard tier (frequent access)
Day 30-90: logs/* in Infrequent Access (40% cheaper)
Day 90+:   logs/* and backups/* in Archive (90% cheaper)
Day 365+:  logs/* automatically deleted
```

The result? You never pay full price for data you rarely access, and old logs clean themselves up.

## Practical Patterns

### Pattern 1: Backup Rotation

Keep the last 30 days of backups in Standard, archive older ones:

```bash
oci os object-lifecycle-policy put \
    --namespace "$NAMESPACE" \
    --bucket-name "db-backups" \
    --items '[{
        "name": "archive-old-backups",
        "action": "ARCHIVE",
        "time-amount": 30,
        "time-unit": "DAYS",
        "is-enabled": true,
        "target": "objects"
    }]'
```

### Pattern 2: Pre-Authenticated Requests

Share files securely without giving bucket access:

```bash
# Create a pre-authenticated request (valid for 7 days)
oci os preauth-request create \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --name "share-report" \
    --access-type "ObjectRead" \
    --object-name "reports/quarterly-report.pdf" \
    --time-expires "2026-03-15T00:00:00Z"
```

This generates a unique URL that anyone can use to download the file — no OCI credentials needed. The link expires automatically.

### Pattern 3: Bulk Upload with Parallelism

Upload an entire directory efficiently:

```bash
oci os object bulk-upload \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --src-dir "./local-data/" \
    --prefix "uploads/2026-03/" \
    --parallel-upload-count 10 \
    --overwrite
```

The `--parallel-upload-count 10` flag uploads 10 files simultaneously, dramatically speeding up large transfers.

### Pattern 4: Sync Local Directory

Keep a local directory in sync with a bucket:

```bash
oci os object sync \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --prefix "sync/" \
    --src-dir "./local-sync/"
```

This uploads only changed files — similar to `rsync` but for cloud storage.

## Querying Bucket Contents

### List Objects with Metadata

```bash
oci os object list \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --query 'data[*].{"name":"name","size":"size","modified":"time-modified"}' \
    --output table
```

### List by Prefix (Folder)

```bash
oci os object list \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --prefix "logs/" \
    --query 'data[*].name'
```

### Get Bucket Size

```bash
oci os bucket get \
    --namespace "$NAMESPACE" \
    --bucket-name "my-app-data" \
    --query 'data.{"objects":"approximate-count","size":"approximate-size"}'
```

## Security Considerations

1. **Keep buckets private** — Use `NoPublicAccess` unless you specifically need public access
2. **Use pre-authenticated requests** for sharing — They expire automatically
3. **Enable versioning** for important data — Protects against accidental deletion
4. **Use IAM policies** to restrict bucket access — Principle of least privilege
5. **Enable server-side encryption** — OCI encrypts all objects by default with Oracle-managed keys. Use customer-managed keys (OCI Vault) for additional control.

## Comparing with Alternatives

| Feature | OCI Object Storage | AWS S3 | Azure Blob | GCS |
|---------|-------------------|--------|------------|-----|
| **Free tier** | 20 GB (permanent) | 5 GB (12 months) | 5 GB (12 months) | 5 GB (permanent) |
| **Storage tiers** | 3 | 6 | 4 | 4 |
| **Lifecycle policies** | Yes | Yes | Yes | Yes |
| **Versioning** | Yes | Yes | Yes | Yes |
| **Outbound transfer** | 10 TB free | 100 GB free | 5 GB free | 1 GB free |

OCI's 10 TB free outbound transfer is exceptional — AWS charges $0.09/GB after 100 GB.

## The Complete Script

**Repository:** [github.com/pmady/oci-object-storage-automation](https://github.com/pmady/oci-object-storage-automation)

```bash
git clone https://github.com/pmady/oci-object-storage-automation.git
cd oci-object-storage-automation
chmod +x deploy_object_storage.sh
./deploy_object_storage.sh
```

## Conclusion

Object Storage is the backbone of cloud data management. With lifecycle policies, you can automate cost optimization without any manual intervention. OCI's Always Free tier gives you 20 GB to experiment with — enough for real workloads, not just toy examples.

The key takeaway: **do not just store data — manage it**. Set up lifecycle policies from day one, enable versioning for important data, and keep buckets private by default.

---

*All resources in this post use OCI Always Free tier. No charges will be incurred.*

**Tags:** `#OracleCloud` `#ObjectStorage` `#CloudStorage` `#Automation` `#OCI` `#AlwaysFree` `#DataManagement` `#DevOps`
