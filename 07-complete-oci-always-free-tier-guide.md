# The Complete OCI Always Free Tier Guide for Developers

*Everything you can build on Oracle Cloud without spending a cent — a comprehensive guide to every Always Free service with real-world project ideas.*

---

## Introduction

Cloud computing does not have to cost money. Oracle Cloud Infrastructure offers one of the most generous Always Free tiers in the industry — and unlike AWS and Azure, it does not expire after 12 months.

I have spent the past several weeks building real projects on OCI's Always Free tier, and I can tell you: this is not a limited sandbox. You get production-grade services — databases, compute, storage, monitoring, and security — that run indefinitely at zero cost.

In this guide, I will map out every Always Free service, show you what you can build with each one, and share the automation scripts I created along the way.

## What Makes OCI's Free Tier Different

Most cloud providers offer free tiers with an asterisk:

| Provider | Free Tier Duration | Notable Limits |
|----------|-------------------|----------------|
| **OCI Always Free** | **Forever** | 4 OCPUs compute, 2 ADBs, 20GB storage |
| AWS Free Tier | 12 months (most services) | 750 hrs t2.micro, 5GB S3 |
| Azure Free | 12 months (most services) | 750 hrs B1S, 5GB Blob |
| GCP Free Tier | Forever (limited) | 1 e2-micro, 5GB storage |

OCI stands out with its permanent Always Free commitment and significantly more generous compute allocation.

## Always Free Services — The Complete List

### Compute

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **VM.Standard.A1.Flex (ARM)** | 4 OCPUs, 24GB RAM total | Run up to 4 VMs or 1 large VM |
| **VM.Standard.E2.1.Micro (AMD)** | 2 instances (1/8 OCPU, 1GB each) | Lightweight services, bastion hosts |
| **Boot Volume** | 200GB total | OS disks for your instances |
| **Block Volume** | 200GB total, 5 backups | Additional attached storage |

**The ARM allocation is exceptional.** 4 OCPUs and 24GB RAM is enough to run a Kubernetes cluster (K3s), a complete development environment, or multiple application servers.

Practical configurations:
- **1 VM:** 4 OCPUs, 24GB RAM — a powerful dev server
- **2 VMs:** 2 OCPUs + 12GB each — web server + database server
- **4 VMs:** 1 OCPU + 6GB each — microservices architecture

### Databases

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **Autonomous Database** | 2 instances (1 OCPU, 20GB each) | Full Oracle DB with APEX |
| **NoSQL Database** | 133M reads, 133M writes/month, 25GB/table | Key-value and document storage |
| **MySQL HeatWave** | 1 instance (50GB storage) | MySQL-compatible managed database |

**Two Autonomous Databases is generous.** You can run one for development and one for staging, or use one for OLTP and one for analytics.

### Storage

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **Object Storage** | 20GB Standard, 20GB Infrequent Access | File storage, backups, static assets |
| **Archive Storage** | 20GB | Long-term data retention |
| **Block Volume** | 200GB total | Attached disk storage for VMs |

### Networking

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **VCN** | 2 Virtual Cloud Networks | Complete network isolation |
| **Load Balancer** | 1 Flexible (10 Mbps) | Distribute traffic across backends |
| **Network Load Balancer** | 1 instance | Layer 4 load balancing |
| **Site-to-Site VPN** | 50 IPSec connections | Connect on-premises networks |
| **Outbound Data Transfer** | 10 TB/month | Serve content globally |
| **VCN Flow Logs** | Included with Logging | Network traffic audit |

**10 TB/month outbound transfer is incredible.** AWS charges $0.09/GB after 100GB. On OCI, you get 10,000 GB free.

### Security

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **Vault** | 20 key versions, 150 secret versions | Secrets management, encryption |
| **Bastion** | Unlimited sessions | Secure SSH access to private subnets |

### Observability

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **Monitoring** | 500M ingestion data points/month | Metric collection and alarms |
| **Logging** | 10 GB/month | Centralized log management |
| **Notifications** | 1M HTTPS, 1K email/month | Alert delivery |
| **APM** | 1,000 tracing events/hour | Application performance monitoring |

### Management

| Service | Free Allocation | What You Can Do |
|---------|----------------|-----------------|
| **Resource Manager** | Unlimited stacks | Terraform-based IaC |
| **Email Delivery** | 3,000 emails/month | Transactional email |

## Six Projects You Can Build Today

I built all of these using only Always Free resources. Each has a companion GitHub repository with automation scripts.

### Project 1: Automated Cloud Infrastructure

**What:** A script that provisions a complete VCN, security lists, subnet, and compute instance.

**Services Used:** Compute, Networking

**Key Learning:** OCI CLI automation, networking architecture, availability domain management, retry logic for capacity constraints.

**Repository:** [github.com/pmady/oci-vm-quickstart-k8s](https://github.com/pmady/oci-vm-quickstart-k8s)

```bash
git clone https://github.com/pmady/oci-vm-quickstart-k8s.git
cd oci-vm-quickstart-k8s && chmod +x always_free_deploy.sh
./always_free_deploy.sh
```

### Project 2: Web Application with Oracle APEX

**What:** An Autonomous Database with APEX for building web applications directly in the database.

**Services Used:** Autonomous Database, APEX

**Key Learning:** Managed database provisioning, APEX low-code development, REST API creation, SQL Developer Web.

**Repository:** [github.com/pmady/oci-autonomous-db-apex](https://github.com/pmady/oci-autonomous-db-apex)

```bash
git clone https://github.com/pmady/oci-autonomous-db-apex.git
cd oci-autonomous-db-apex && chmod +x deploy_autonomous_db.sh
./deploy_autonomous_db.sh
```

### Project 3: Cloud Storage with Lifecycle Management

**What:** Automated Object Storage bucket with versioning, organized uploads, and lifecycle policies for cost optimization.

**Services Used:** Object Storage

**Key Learning:** Storage tier management, lifecycle policies, pre-authenticated requests, bulk operations.

**Repository:** [github.com/pmady/oci-object-storage-automation](https://github.com/pmady/oci-object-storage-automation)

```bash
git clone https://github.com/pmady/oci-object-storage-automation.git
cd oci-object-storage-automation && chmod +x deploy_object_storage.sh
./deploy_object_storage.sh
```

### Project 4: Secrets Management with OCI Vault

**What:** A vault with encryption keys and encrypted secrets for secure credential management.

**Services Used:** Vault, Key Management

**Key Learning:** Encryption key creation, secret storage and retrieval, Base64 encoding, runtime credential access.

**Repository:** [github.com/pmady/oci-vault-secrets-automation](https://github.com/pmady/oci-vault-secrets-automation)

```bash
git clone https://github.com/pmady/oci-vault-secrets-automation.git
cd oci-vault-secrets-automation && chmod +x deploy_vault_secrets.sh
./deploy_vault_secrets.sh
```

### Project 5: Observability Stack

**What:** Monitoring alarms, notification topics, and centralized logging for infrastructure observability.

**Services Used:** Monitoring, Logging, Notifications

**Key Learning:** MQL queries, alarm configuration, notification routing, log group management.

**Repository:** [github.com/pmady/oci-monitoring-notifications](https://github.com/pmady/oci-monitoring-notifications)

```bash
git clone https://github.com/pmady/oci-monitoring-notifications.git
cd oci-monitoring-notifications && chmod +x deploy_monitoring.sh
./deploy_monitoring.sh
```

### Project 6: NoSQL Database with SQL Queries

**What:** A NoSQL table with sample data and SQL-like queries including aggregation and JSON access.

**Services Used:** NoSQL Database

**Key Learning:** Flexible schema design, JSON column support, SQL-like queries on NoSQL data, read/write unit management.

**Repository:** [github.com/pmady/oci-nosql-database-automation](https://github.com/pmady/oci-nosql-database-automation)

```bash
git clone https://github.com/pmady/oci-nosql-database-automation.git
cd oci-nosql-database-automation && chmod +x deploy_nosql.sh
./deploy_nosql.sh
```

## Getting Started: Your First 30 Minutes

If you are new to OCI, here is what I recommend for your first session:

**Minutes 0-5:** Sign up at [cloud.oracle.com](https://cloud.oracle.com) and access the Console

**Minutes 5-10:** Open Cloud Shell (click the terminal icon in the top right) — OCI CLI is pre-configured

**Minutes 10-20:** Run the Autonomous Database script — it works instantly, no capacity issues:

```bash
git clone https://github.com/pmady/oci-autonomous-db-apex.git
cd oci-autonomous-db-apex && chmod +x deploy_autonomous_db.sh
./deploy_autonomous_db.sh
```

**Minutes 20-30:** Open the APEX URL from the output and sign in — you now have a web application platform

No credit card charges. No trial expiration. These services run forever.

## Tips for Maximizing the Free Tier

1. **Start with services that have no capacity constraints** — Autonomous Database, Object Storage, NoSQL, Vault, and Monitoring are always available instantly.

2. **ARM compute (A1.Flex) has capacity limitations** — Try during off-peak hours (early morning US time). Use the retry-across-all-ADs pattern.

3. **Use Cloud Shell for everything** — It is free, always available, and OCI CLI is pre-configured. No local setup needed.

4. **Monitor your usage** — OCI Console shows Always Free usage on the billing page. You cannot accidentally be charged if you only use Always Free shapes.

5. **Clean up unused resources** — While most Always Free resources do not count against limits when idle, terminated VCNs and databases free up capacity for new experiments.

6. **Use Resource Manager (Terraform) for repeatability** — Store your infrastructure as code and deploy it consistently.

## Common Pitfalls

- **Do not select non-free shapes** — Always verify "Always Free" is checked when creating resources
- **ARM vs AMD images** — A1.Flex needs `aarch64` images, E2.1.Micro needs `x86_64`. Mismatching gives a 404 error
- **Autonomous DB password requirements** — 12-30 characters, must include uppercase, lowercase, and number
- **VCN limit is 2** — Clean up old VCNs before creating new ones
- **Always Free databases cannot scale** — CPU and storage are fixed

## Conclusion

Oracle Cloud's Always Free tier is not a marketing gimmick. It provides real, production-grade cloud services that you can use to learn, experiment, and build projects that matter.

I have built six automated projects across compute, databases, storage, security, and monitoring — all at zero cost. The scripts are open source, the documentation is detailed, and everything runs on the free tier.

If you have been putting off learning cloud computing because of cost concerns, OCI removes that barrier entirely. Start building today.

---

*All projects referenced in this post use exclusively OCI Always Free tier resources. No charges will be incurred.*

**Tags:** `#OracleCloud` `#OCI` `#AlwaysFree` `#CloudComputing` `#FreeTier` `#Developer` `#DevOps` `#Tutorial`
