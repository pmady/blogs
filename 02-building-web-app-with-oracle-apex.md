# Building a Web Application in 10 Minutes with Oracle APEX on Always Free Tier

*How to go from zero to a running web application using Oracle Autonomous Database and APEX — no servers, no frameworks, no cost.*

---

## Introduction

Most developers associate building web applications with choosing a framework (React, Django, Rails), setting up a database, configuring a web server, and deploying to some hosting platform. What if I told you that Oracle provides a fully managed platform where you can build and deploy web applications directly inside your database — with zero infrastructure management?

That platform is **Oracle APEX** (Application Express), and it comes free with every Oracle Autonomous Database.

In this post, I will walk you through automating the provisioning of an Autonomous Database with APEX using the OCI CLI, and then building your first application.

## What is Oracle APEX?

Oracle APEX is a low-code development platform that runs entirely within the Oracle Database. Here is what makes it unique:

- **No separate application server** — APEX runs inside the database engine itself
- **SQL and PL/SQL powered** — If you know SQL, you can build apps
- **Built-in authentication** — User management out of the box
- **Responsive UI** — Modern, mobile-friendly interface components
- **REST API support** — Create and consume RESTful services
- **Free forever** — Included with every Oracle Database, including Always Free tier

## Why Autonomous Database + APEX?

The combination is compelling for several reasons:

| Feature | Benefit |
|---------|---------|
| **Fully Managed** | Oracle handles patching, backups, tuning, and scaling |
| **Auto-Secured** | Encrypted at rest and in transit, automated security patches |
| **Built-in APEX** | No installation — APEX workspace is ready immediately |
| **Always Free** | 1 OCPU, 20GB storage, runs indefinitely at zero cost |
| **99.95% SLA** | Enterprise-grade availability for free |

## Automating the Deployment

Instead of clicking through the OCI Console, let us script the entire process.

### The Script

```bash
#!/bin/bash
set -euo pipefail

COMPARTMENT_ID="ocid1.tenancy.oc1..your-tenancy-ocid"
DB_NAME="MYAPPDB"
DISPLAY_NAME="My-APEX-App-DB"
ADMIN_PASSWORD="WelcomeAPEX2026#"

# Create Always Free Autonomous Database
ADB_ID=$(oci db autonomous-database create \
    --compartment-id "$COMPARTMENT_ID" \
    --db-name "$DB_NAME" \
    --display-name "$DISPLAY_NAME" \
    --admin-password "$ADMIN_PASSWORD" \
    --cpu-core-count 1 \
    --data-storage-size-in-tbs 1 \
    --db-workload "OLTP" \
    --is-free-tier true \
    --is-auto-scaling-enabled false \
    --query 'data.id' --raw-output)

echo "Database created: $ADB_ID"

# Wait for AVAILABLE state
oci db autonomous-database get \
    --autonomous-database-id "$ADB_ID" \
    --wait-for-state AVAILABLE \
    --wait-interval-seconds 15 > /dev/null 2>&1

# Get APEX URL
APEX_URL=$(oci db autonomous-database get \
    --autonomous-database-id "$ADB_ID" \
    --query 'data."connection-urls"."apex-url"' --raw-output)

echo "APEX URL: $APEX_URL"
```

Key parameters explained:
- `--db-workload "OLTP"` — Transaction Processing, optimized for APEX workloads
- `--is-free-tier true` — This is the critical flag that ensures Always Free
- `--is-auto-scaling-enabled false` — Always Free databases cannot auto-scale
- `--cpu-core-count 1` — Always Free allows 1 OCPU maximum

### Running the Script

```bash
git clone https://github.com/pmady/oci-autonomous-db-apex.git
cd oci-autonomous-db-apex
chmod +x deploy_autonomous_db.sh
./deploy_autonomous_db.sh
```

The database takes about 2-5 minutes to provision. Once done, you will get an APEX URL like:

```
https://GD96DA588E7AC01-MYAPPDB.adb.us-chicago-1.oraclecloudapps.com/ords/apex
```

## Building Your First APEX Application

### Step 1: Access APEX

Open the APEX URL in your browser and sign in:
- **Workspace:** INTERNAL
- **Username:** ADMIN
- **Password:** (the one you set in the script)

### Step 2: Create a Workspace

APEX uses workspaces to organize applications:

1. Click **Create Workspace**
2. Choose **New Schema**
3. Workspace Name: `MY_WORKSPACE`
4. Schema Name: `MY_SCHEMA`
5. Set workspace admin password

### Step 3: Build the Application

Sign into your new workspace and:

1. Click **App Builder** → **Create Application**
2. Name: `Cloud Resource Tracker`
3. Add pages:
   - **Interactive Report** — View and search resources
   - **Form** — Add/edit resources
   - **Dashboard** — Charts and metrics
4. Click **Create Application**

APEX generates a complete application with:
- Login page with authentication
- Navigation menu
- Responsive layout
- CRUD operations on your data

### Step 4: Create a Data Model

In **SQL Workshop** → **SQL Commands**, run:

```sql
CREATE TABLE cloud_resources (
    id          NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        VARCHAR2(200) NOT NULL,
    service     VARCHAR2(100) NOT NULL,
    region      VARCHAR2(50),
    status      VARCHAR2(20) DEFAULT 'ACTIVE',
    monthly_cost NUMBER(10,2) DEFAULT 0,
    created_at  TIMESTAMP DEFAULT SYSTIMESTAMP
);

INSERT INTO cloud_resources (name, service, region, status, monthly_cost)
VALUES ('Production DB', 'Autonomous Database', 'us-chicago-1', 'AVAILABLE', 0);

INSERT INTO cloud_resources (name, service, region, status, monthly_cost)
VALUES ('Web Server', 'Compute A1.Flex', 'us-chicago-1', 'RUNNING', 0);

INSERT INTO cloud_resources (name, service, region, status, monthly_cost)
VALUES ('Backup Bucket', 'Object Storage', 'us-chicago-1', 'ACTIVE', 0);

COMMIT;
```

### Step 5: Add a REST API

APEX makes it simple to expose your data as REST endpoints:

1. Go to **SQL Workshop** → **RESTful Services**
2. Click **Register Schema with ORDS**
3. Enable REST for your table:

```sql
BEGIN
    ORDS.ENABLE_OBJECT(
        p_enabled      => TRUE,
        p_schema       => 'MY_SCHEMA',
        p_object       => 'CLOUD_RESOURCES',
        p_object_type  => 'TABLE',
        p_object_alias => 'resources'
    );
    COMMIT;
END;
/
```

Now you have REST endpoints:
- `GET /ords/my_schema/resources/` — List all resources
- `POST /ords/my_schema/resources/` — Create a resource
- `PUT /ords/my_schema/resources/:id` — Update a resource
- `DELETE /ords/my_schema/resources/:id` — Delete a resource

## Architecture Overview

```
┌──────────────────────────────────────────────┐
│        Oracle Autonomous Database             │
│                                              │
│  ┌────────────┐  ┌─────────────────────────┐ │
│  │   Oracle    │  │    Oracle APEX          │ │
│  │  Database   │  │  ┌─────────────────┐   │ │
│  │  (19c/23ai) │◄─│  │ Web Application │   │ │
│  │             │  │  │ (App Builder)   │   │ │
│  │  Tables     │  │  └─────────────────┘   │ │
│  │  Views      │  │  ┌─────────────────┐   │ │
│  │  PL/SQL     │  │  │   REST APIs     │   │ │
│  │             │  │  │   (ORDS)        │   │ │
│  └────────────┘  │  └─────────────────┘   │ │
│                  └─────────────────────────┘ │
│                                              │
│  HTTPS endpoint (auto-provisioned)           │
│  *.adb.us-chicago-1.oraclecloudapps.com     │
└──────────────────────────────────────────────┘
```

Everything runs inside the Autonomous Database. No separate compute, no web server, no load balancer, no SSL certificate management.

## APEX vs Traditional Web Frameworks

| Aspect | Traditional (React + Node + DB) | Oracle APEX |
|--------|-------------------------------|-------------|
| **Setup time** | Hours to days | 10 minutes |
| **Infrastructure** | 3+ services to manage | 1 service (Autonomous DB) |
| **SSL/HTTPS** | Manual or Let's Encrypt | Auto-provisioned |
| **Authentication** | Build or integrate | Built-in |
| **Database access** | ORM or raw queries | Native SQL/PL/SQL |
| **Hosting cost** | $20-100+/month | $0 (Always Free) |
| **Scaling** | Manual configuration | Automatic (paid tier) |

APEX is not a replacement for React or modern SPAs in every scenario. But for internal tools, dashboards, CRUD applications, and rapid prototyping, it is remarkably productive.

## Best Practices

1. **Use the App Builder wizards first** — They generate well-structured code that you can customize later
2. **Leverage Interactive Reports** — They provide built-in search, filter, export, and column management
3. **Use Template Components** — APEX's modern UI components are responsive by default
4. **Secure your APIs** — Use OAuth2 or APEX session authentication for REST endpoints
5. **Version your SQL scripts** — Keep DDL and seed data scripts in Git alongside your app export

## Cleanup

If you need to delete the database:

```bash
oci db autonomous-database delete \
    --autonomous-database-id "$ADB_ID" \
    --force
```

## Conclusion

Oracle APEX on Autonomous Database is one of the most underrated platforms in cloud computing. You get a fully managed database, a web application runtime, REST API generation, and enterprise authentication — all within the Always Free tier.

For developers who need to build data-driven applications quickly, APEX deserves serious consideration.

**Repository:** [github.com/pmady/oci-autonomous-db-apex](https://github.com/pmady/oci-autonomous-db-apex)

---

*All resources in this post use OCI Always Free tier. No charges will be incurred.*

**Tags:** `#OracleCloud` `#APEX` `#AutonomousDatabase` `#LowCode` `#WebDevelopment` `#AlwaysFree` `#OCI`
