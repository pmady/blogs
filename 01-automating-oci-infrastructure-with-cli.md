# Automating OCI Infrastructure with the CLI: From Zero to a Running VM in One Script

*How I automated the provisioning of a complete Oracle Cloud Infrastructure environment — VCN, subnets, security lists, and a compute instance — using nothing but the OCI CLI and bash.*

---

## Introduction

If you have worked with any major cloud provider, you know the drill: click through a dozen console screens, configure networking, set up security rules, pick an image, launch an instance, and hope you did not miss anything. It is tedious, error-prone, and impossible to reproduce consistently.

Oracle Cloud Infrastructure (OCI) provides a powerful CLI that lets you automate all of this. In this post, I will walk you through a real-world script that provisions a complete Always Free environment — from an empty tenancy to a running ARM-based VM — in a single command.

## Why Automate with the OCI CLI?

Before we dive into code, here is why CLI automation matters:

- **Reproducibility** — Run the same script in any region, any tenancy, and get identical results
- **Speed** — What takes 15 minutes of clicking takes 2 minutes scripted
- **Documentation** — Your script IS your documentation. Anyone can read it and understand exactly what was deployed
- **Version Control** — Store infrastructure scripts in Git, track changes, review PRs
- **Error Handling** — Scripts can detect failures and retry intelligently

## The Architecture

Here is what our script will create:

```
┌─────────────────────────────────────────┐
│           VCN (10.0.0.0/16)             │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │     Subnet (10.0.1.0/24)         │  │
│  │                                   │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │   Compute Instance          │  │  │
│  │  │   VM.Standard.A1.Flex       │  │  │
│  │  │   1 OCPU | 6GB RAM         │  │  │
│  │  │   Oracle Linux 9.7         │  │  │
│  │  └─────────────────────────────┘  │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  Security List:                         │
│    Ingress: SSH(22), HTTP(80),          │
│             HTTPS(443), K8s(6443)       │
│    Egress:  All traffic                 │
└─────────────────────────────────────────┘
```

All of this fits within OCI's Always Free tier. Total cost: **$0**.

## Prerequisites

You need two things:

1. **An OCI account** — Sign up at [cloud.oracle.com](https://cloud.oracle.com). The Always Free tier never expires.
2. **OCI CLI** — Either use OCI Cloud Shell (pre-configured, zero setup) or install locally with:

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
oci setup config
```

## Step 1: Setting Up the Script

Every good infrastructure script starts with strict error handling:

```bash
#!/bin/bash
set -euo pipefail
```

The flags mean:
- `set -e` — Exit immediately if any command fails
- `set -u` — Treat unset variables as errors
- `set -o pipefail` — Catch errors in piped commands

Next, define your configuration:

```bash
COMPARTMENT_ID="ocid1.tenancy.oc1..your-tenancy-ocid"
SHAPE="VM.Standard.A1.Flex"
SHAPE_CONFIG='{"ocpus":1,"memoryInGBs":6}'
IMAGE_ID="ocid1.image.oc1.us-chicago-1.aaaa..."  # Oracle Linux 9.7 aarch64
```

## Step 2: Creating the Virtual Cloud Network

A VCN is OCI's equivalent of an AWS VPC. It is the foundation of your networking:

```bash
VCN_ID=$(oci network vcn create \
    --compartment-id "$COMPARTMENT_ID" \
    --cidr-block "10.0.0.0/16" \
    --display-name "My-VCN" \
    --dns-label "myvcn" \
    --query 'data.id' --raw-output)
```

Key points:
- `--cidr-block "10.0.0.0/16"` gives us 65,536 IP addresses
- `--dns-label` enables DNS hostname resolution within the VCN
- `--query 'data.id' --raw-output` extracts just the OCID from the JSON response

## Step 3: Configuring the Security List

OCI uses Security Lists (similar to AWS Security Groups) to control traffic. Every VCN comes with a default security list that we can update:

```bash
SL_ID=$(oci network vcn get --vcn-id "$VCN_ID" \
    --query 'data."default-security-list-id"' --raw-output)

oci network security-list update \
    --security-list-id "$SL_ID" \
    --ingress-security-rules '[
        {"source":"0.0.0.0/0","protocol":"6",
         "tcpOptions":{"destinationPortRange":{"min":22,"max":22}}},
        {"source":"0.0.0.0/0","protocol":"6",
         "tcpOptions":{"destinationPortRange":{"min":80,"max":80}}},
        {"source":"0.0.0.0/0","protocol":"6",
         "tcpOptions":{"destinationPortRange":{"min":443,"max":443}}}
    ]' \
    --egress-security-rules '[
        {"destination":"0.0.0.0/0","protocol":"all"}
    ]' \
    --force > /dev/null
```

The `--force` flag skips the confirmation prompt, and redirecting to `/dev/null` suppresses the verbose JSON output.

## Step 4: Creating the Subnet

Subnets in OCI are regional by default (spanning all availability domains), which simplifies architecture:

```bash
SUBNET_ID=$(oci network subnet create \
    --compartment-id "$COMPARTMENT_ID" \
    --vcn-id "$VCN_ID" \
    --cidr-block "10.0.1.0/24" \
    --display-name "My-Subnet" \
    --dns-label "mysubnet" \
    --security-list-ids "[\"$SL_ID\"]" \
    --query 'data.id' --raw-output)

sleep 10  # Wait for subnet to become available
```

The `sleep` is important — subnets need a few seconds to transition to the AVAILABLE state before you can launch instances in them.

## Step 5: The Smart Instance Launch

This is where it gets interesting. OCI Always Free ARM instances (A1.Flex) are extremely popular, which means capacity can be limited. Instead of failing on the first try, our script tries **all availability domains** with automatic retries:

```bash
# Get ALL availability domains
readarray -t AD_LIST < <(oci iam availability-domain list \
    --compartment-id "$COMPARTMENT_ID" \
    --query 'data[*].name' --raw-output | \
    tr -d '[]"' | tr ',' '\n' | sed 's/^ *//;s/ *$//' | grep -v '^$')

# Try each AD, retry on capacity errors
for ((ROUND=1; ROUND<=60; ROUND++)); do
    for AD_NAME in "${AD_LIST[@]}"; do
        LAUNCH_OUTPUT=$(oci compute instance launch \
            --availability-domain "$AD_NAME" \
            --compartment-id "$COMPARTMENT_ID" \
            --shape "$SHAPE" \
            --shape-config "$SHAPE_CONFIG" \
            --subnet-id "$SUBNET_ID" \
            --display-name "My-Free-VM" \
            --image-id "$IMAGE_ID" \
            --query 'data.id' --raw-output 2>&1) || true

        if [[ "$LAUNCH_OUTPUT" == ocid1.instance.* ]]; then
            echo "Success! Instance launched in $AD_NAME"
            break 2
        elif echo "$LAUNCH_OUTPUT" | grep -q "Out of host capacity"; then
            echo "No capacity in $AD_NAME, trying next..."
        fi
    done
    sleep 30
done
```

This approach:
- Tries all 3 ADs in Chicago (or however many your region has)
- Handles the "Out of host capacity" error gracefully
- Retries every 30 seconds for up to 30 minutes
- Uses `break 2` to exit both loops on success

## Step 6: Getting the Results

Once the instance launches, we wait for it to reach the RUNNING state and grab its IP:

```bash
oci compute instance get --instance-id "$INSTANCE_ID" \
    --wait-for-state RUNNING \
    --wait-interval-seconds 10 > /dev/null 2>&1

PRIVATE_IP=$(oci compute instance list-vnics \
    --instance-id "$INSTANCE_ID" \
    --query 'data[0]."private-ip"' --raw-output)
```

## Lessons Learned

After building and debugging this script across multiple runs, here are the key takeaways:

1. **Always use `--query` and `--raw-output`** — OCI CLI returns verbose JSON by default. JMESPath queries with `--raw-output` give you clean values for variable assignment.

2. **Shape availability varies by region** — `VM.Standard.E2.1.Micro` is not available in every region. Always check with `oci compute shape list` first.

3. **Image architecture must match shape** — ARM shapes (A1.Flex) need `aarch64` images. AMD shapes need `x86_64` images. Mismatching them gives a cryptic 404 error.

4. **Subnets need time** — Do not try to launch instances immediately after creating a subnet. A short `sleep` prevents race conditions.

5. **Always Free capacity is limited** — The retry-across-all-ADs pattern is essential for A1.Flex instances. Early morning US time has the best availability.

## The Complete Script

The full script with all error handling and retry logic is available on GitHub:

**Repository:** [github.com/pmady/oci-vm-quickstart-k8s](https://github.com/pmady/oci-vm-quickstart-k8s)

Clone and run it:

```bash
git clone https://github.com/pmady/oci-vm-quickstart-k8s.git
cd oci-vm-quickstart-k8s
chmod +x always_free_deploy.sh
./always_free_deploy.sh
```

## What is Next?

With a running VM, you can:
- Install K3s for a lightweight Kubernetes cluster
- Deploy containerized applications
- Set up a development environment
- Run CI/CD agents

In my next post, I will cover how to build a web application using Oracle APEX — completely free, no compute instance needed.

---

*All code in this post uses OCI Always Free tier resources. No charges will be incurred.*

**Tags:** `#OracleCloud` `#OCI` `#Infrastructure` `#Automation` `#CLI` `#AlwaysFree` `#CloudComputing` `#DevOps`
