# Building a Zero-Cost Observability Stack on Oracle Cloud Infrastructure

*How to set up monitoring alarms, centralized logging, and automated notifications on OCI — without spending a dime.*

---

## Introduction

Observability is not optional. Whether you are running a single VM or a fleet of microservices, you need to know when things go wrong — before your users do.

The challenge? Most observability solutions cost money. Datadog, New Relic, and Splunk all charge based on data volume. Even self-hosted options like Prometheus + Grafana require compute resources to run.

Oracle Cloud Infrastructure provides a native observability stack — Monitoring, Logging, and Notifications — with generous Always Free allocations. In this post, I will show you how to automate the setup of a complete observability pipeline using the OCI CLI.

## The Observability Stack

Our stack consists of three OCI services working together:

```
┌─────────────────┐     ┌──────────────────────┐     ┌────────────────┐
│  OCI Monitoring  │────▶│  Notification Topic   │────▶│  Destinations  │
│  (Metric Alarms) │     │  (Event Router)       │     │  - HTTPS       │
└────────┬────────┘     └──────────────────────┘     │  - Email       │
         │                                            │  - Slack       │
         │                                            │  - PagerDuty   │
┌────────▼────────┐                                  └────────────────┘
│   OCI Logging    │
│  (Centralized)   │
└─────────────────┘
```

**Monitoring** collects metrics from your OCI resources, evaluates alarm conditions, and triggers notifications. **Logging** aggregates logs from services, VCN flow logs, and custom applications. **Notifications** routes alerts to the destinations you choose.

## Always Free Allocations

| Service | Free Allocation |
|---------|----------------|
| **Monitoring** | 500 million ingestion data points/month |
| **Notifications (HTTPS)** | 1 million deliveries/month |
| **Notifications (Email)** | 1,000 deliveries/month |
| **Logging** | 10 GB/month |

For context, 500 million data points per month means you can collect metrics from hundreds of resources at 1-minute intervals. This is more than enough for most workloads.

## Setting Up the Pipeline

### Step 1: Create a Notification Topic

A notification topic is a message bus. Alarms publish to it, and subscribers receive from it:

```bash
TOPIC_ID=$(oci ons topic create \
    --compartment-id "$COMPARTMENT_ID" \
    --name "infrastructure-alerts" \
    --description "Critical alerts for OCI infrastructure" \
    --query 'data."topic-id"' --raw-output)
```

### Step 2: Add Subscribers

You can add multiple subscribers with different protocols:

```bash
# HTTPS webhook (for Slack, PagerDuty, custom endpoints)
oci ons subscription create \
    --compartment-id "$COMPARTMENT_ID" \
    --topic-id "$TOPIC_ID" \
    --protocol "HTTPS" \
    --endpoint "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Email (requires confirmation)
oci ons subscription create \
    --compartment-id "$COMPARTMENT_ID" \
    --topic-id "$TOPIC_ID" \
    --protocol "EMAIL" \
    --endpoint "your-email@example.com"
```

Supported protocols: HTTPS, EMAIL, SLACK, PAGERDUTY, SMS, CUSTOM_HTTPS, and ORACLE_FUNCTIONS.

### Step 3: Create Monitoring Alarms

Now we create alarms that evaluate metrics and fire when thresholds are breached.

**CPU Alarm (Critical):**

```bash
oci monitoring alarm create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "high-cpu-alarm" \
    --metric-compartment-id "$COMPARTMENT_ID" \
    --namespace "oci_computeagent" \
    --query-text "CpuUtilization[5m].mean() > 80" \
    --severity "CRITICAL" \
    --destinations "[\"$TOPIC_ID\"]" \
    --body "CPU utilization exceeded 80% for 5 minutes." \
    --is-enabled true \
    --pending-duration "PT5M" \
    --resolution "1m"
```

Let us break down the key parameters:

- `--namespace "oci_computeagent"` — Metrics come from the compute agent running on instances
- `--query-text "CpuUtilization[5m].mean() > 80"` — MQL (Monitoring Query Language) expression. This calculates the 5-minute mean and triggers when it exceeds 80%
- `--pending-duration "PT5M"` — The condition must persist for 5 minutes before firing. This prevents alerts on transient spikes
- `--resolution "1m"` — Evaluate the alarm every minute

**Memory Alarm (Warning):**

```bash
oci monitoring alarm create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "high-memory-alarm" \
    --namespace "oci_computeagent" \
    --query-text "MemoryUtilization[5m].mean() > 85" \
    --severity "WARNING" \
    --destinations "[\"$TOPIC_ID\"]" \
    --body "Memory utilization exceeded 85%." \
    --is-enabled true \
    --pending-duration "PT5M" \
    --resolution "1m"
```

**Disk Alarm (Critical):**

```bash
oci monitoring alarm create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "high-disk-alarm" \
    --namespace "oci_computeagent" \
    --query-text "DiskBytesUsed[5m].mean() / DiskBytesTotal[5m].mean() * 100 > 90" \
    --severity "CRITICAL" \
    --destinations "[\"$TOPIC_ID\"]" \
    --body "Disk usage exceeded 90%." \
    --is-enabled true \
    --pending-duration "PT10M" \
    --resolution "1m"
```

### Step 4: Set Up Centralized Logging

Create a log group (a logical container for logs) and a custom log:

```bash
LOG_GROUP_ID=$(oci logging log-group create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "infrastructure-logs" \
    --description "Centralized infrastructure log group" \
    --query 'data.id' --raw-output)

LOG_ID=$(oci logging log create \
    --log-group-id "$LOG_GROUP_ID" \
    --display-name "app-custom-log" \
    --log-type "CUSTOM" \
    --query 'data.id' --raw-output)
```

Log types:
- **CUSTOM** — Application-generated logs pushed via the Logging API
- **SERVICE** — OCI service logs (VCN Flow Logs, Object Storage access logs, etc.)

### Step 5: Enable VCN Flow Logs (Optional)

If you have a VCN, enable flow logs for network audit:

```bash
oci logging log create \
    --log-group-id "$LOG_GROUP_ID" \
    --display-name "vcn-flow-log" \
    --log-type "SERVICE" \
    --configuration '{
        "source": {
            "category": "all",
            "resource": "<SUBNET_OCID>",
            "service": "flowlogs",
            "source-type": "OCISERVICE"
        }
    }'
```

VCN Flow Logs capture all network traffic in and out of your subnets — invaluable for security analysis and troubleshooting.

## Monitoring Query Language (MQL)

OCI Monitoring uses MQL to define alarm conditions. Here are practical examples:

```
# Average CPU over 5 minutes, per instance
CpuUtilization[5m]{resourceId = "ocid1.instance..."}.mean()

# Maximum memory across all instances
MemoryUtilization[1h].max()

# Network bytes received, grouped by instance
NetworkBytesIn[5m].rate()

# Database connections (for Autonomous DB)
CurrentConnections[5m].mean() > 50

# Object Storage: requests per minute
ObjectCount[1m].sum()
```

MQL supports aggregation functions (`mean`, `max`, `min`, `sum`, `count`, `rate`, `percentile`), time windows, and grouping — similar to PromQL but with OCI-native metric namespaces.

## Available Metric Namespaces

| Namespace | Metrics |
|-----------|---------|
| `oci_computeagent` | CPU, Memory, Disk, Network |
| `oci_autonomous_database` | CPU, Storage, Sessions, SQL Response Time |
| `oci_objectstorage` | Requests, Data Transfer, Object Count |
| `oci_vcn` | Bytes In/Out, Packets In/Out |
| `oci_lbaas` | Active Connections, Bandwidth, HTTP Responses |
| `oci_nosql` | Read/Write Units, Storage, Throttle Count |

## Alert Severity Levels

Choose severity based on impact and urgency:

| Severity | When to Use | Example |
|----------|-------------|---------|
| **CRITICAL** | Immediate action required | CPU > 95%, Disk > 95% |
| **ERROR** | Something is broken | Service health check failed |
| **WARNING** | Approaching threshold | Memory > 85%, Disk > 80% |
| **INFO** | Informational | Deployment completed |

## Real-World Alarm Strategy

For an Always Free compute instance, I recommend these four alarms:

```
1. CPU > 80% for 5 min    → WARNING  (investigate)
2. CPU > 95% for 2 min    → CRITICAL (immediate action)
3. Memory > 85% for 5 min → WARNING  (check for leaks)
4. Disk > 90% for 10 min  → CRITICAL (clean up or extend)
```

## Comparing with Other Solutions

| Feature | OCI Monitoring | CloudWatch | Azure Monitor | Datadog |
|---------|---------------|------------|---------------|---------|
| **Free metrics** | 500M data points | 10 custom metrics | 1K API calls | None |
| **Free notifications** | 1M HTTPS + 1K email | 1K SNS | 1K email | None |
| **Free logging** | 10 GB | 5 GB | 5 GB | None |
| **Custom metrics** | Yes | Yes | Yes | Yes |
| **MQL/Query language** | MQL | Metric Math | KQL | DQL |

OCI's free tier is the most generous of the major cloud providers for observability.

## The Complete Script

**Repository:** [github.com/pmady/oci-monitoring-notifications](https://github.com/pmady/oci-monitoring-notifications)

```bash
git clone https://github.com/pmady/oci-monitoring-notifications.git
cd oci-monitoring-notifications
chmod +x deploy_monitoring.sh
./deploy_monitoring.sh
```

## Conclusion

You do not need expensive third-party tools for basic observability. OCI's native Monitoring, Logging, and Notifications services provide a solid foundation — with generous free allocations that cover most personal and small-team workloads.

Start with the basics: CPU, memory, and disk alarms with email notifications. Then expand to custom metrics, VCN flow logs, and webhook integrations as your infrastructure grows.

---

*All resources in this post use OCI Always Free tier. No charges will be incurred.*

**Tags:** `#OracleCloud` `#Observability` `#Monitoring` `#Logging` `#DevOps` `#OCI` `#AlwaysFree` `#CloudOps`
