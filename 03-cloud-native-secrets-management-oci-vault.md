# Cloud-Native Secrets Management with OCI Vault: Stop Hardcoding Your Credentials

*A hands-on guide to replacing hardcoded passwords, API keys, and connection strings with Oracle Cloud Vault — fully automated, fully encrypted, fully free.*

---

## Introduction

We have all done it. A database password in an environment variable. An API key in a config file. A connection string committed to Git. It works — until it does not. A leaked secret can compromise your entire infrastructure.

Cloud-native secrets management solves this by storing sensitive data in a dedicated, encrypted service and retrieving it at runtime. Oracle Cloud Infrastructure provides **OCI Vault** — a managed key and secret management service — completely free on the Always Free tier.

In this post, I will show you how to automate vault creation, encryption key generation, and secret storage using the OCI CLI.

## The Problem with Hardcoded Secrets

Consider this typical application configuration:

```bash
# config.sh — DO NOT DO THIS
export DB_PASSWORD="MyP@ssw0rd123"
export API_KEY="sk-live-abc123def456"
export DB_CONNECTION="jdbc:oracle:thin:@mydb_high"
```

What could go wrong?

- **Git commits** — Secrets end up in version history, even if you delete them later
- **Log exposure** — Environment variables appear in debug logs and crash reports
- **Shared access** — Everyone with repo access sees production credentials
- **Rotation pain** — Changing a password means updating every server and deployment
- **Compliance violations** — SOC 2, PCI DSS, and HIPAA all require secrets management

## The Solution: OCI Vault

OCI Vault provides three core capabilities:

1. **Vaults** — Logical containers that hold keys and secrets
2. **Master Encryption Keys** — AES or RSA keys used to encrypt/decrypt secrets
3. **Secrets** — Encrypted values (passwords, API keys, certificates) stored securely

```
┌─────────────────────────────────────────────────┐
│                   OCI Vault                      │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │         Master Encryption Key             │   │
│  │         (AES-256 / SOFTWARE)              │   │
│  └──────────┬───────────────────────────────┘   │
│             │ encrypts                          │
│  ┌──────────▼───────────────────────────────┐   │
│  │  Secret: db-password    = "encrypted..." │   │
│  │  Secret: api-key        = "encrypted..." │   │
│  │  Secret: connection-str = "encrypted..." │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  Endpoints:                                     │
│    Management: https://...management.kms.oci..  │
│    Crypto:     https://...crypto.kms.oci..      │
└─────────────────────────────────────────────────┘
```

## Always Free Tier Limits

| Resource | Free Allocation |
|----------|----------------|
| Vaults | Unlimited (DEFAULT type) |
| Master Encryption Keys | 20 key versions |
| Secrets | 150 secret versions |
| API Calls | Included |

This is more than enough for personal projects, development environments, and small production workloads.

## Automating Vault Setup with OCI CLI

### Step 1: Create the Vault

```bash
VAULT_ID=$(oci kms management vault create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "my-app-vault" \
    --vault-type "DEFAULT" \
    --query 'data.id' --raw-output)
```

Vault provisioning takes 2-5 minutes. We need to wait for it to become ACTIVE:

```bash
while true; do
    STATE=$(oci kms management vault get \
        --vault-id "$VAULT_ID" \
        --query 'data."lifecycle-state"' --raw-output)
    [ "$STATE" = "ACTIVE" ] && break
    sleep 15
done
```

### Step 2: Create a Master Encryption Key

Every secret in the vault is encrypted with a master key. We create an AES-256 key:

```bash
MGMT_ENDPOINT=$(oci kms management vault get \
    --vault-id "$VAULT_ID" \
    --query 'data."management-endpoint"' --raw-output)

KEY_ID=$(oci kms management key create \
    --compartment-id "$COMPARTMENT_ID" \
    --display-name "my-app-master-key" \
    --endpoint "$MGMT_ENDPOINT" \
    --key-shape '{"algorithm":"AES","length":32}' \
    --protection-mode "SOFTWARE" \
    --query 'data.id' --raw-output)
```

Two protection modes are available:
- **SOFTWARE** — Key stored in software (Always Free)
- **HSM** — Key stored in a Hardware Security Module (paid, FIPS 140-2 Level 3)

For most use cases, SOFTWARE protection is sufficient.

### Step 3: Store Secrets

Secrets must be Base64-encoded before storage:

```bash
# Encode and store a database password
DB_PASSWORD_B64=$(echo -n "WelcomeACE2026#" | base64)

SECRET_ID=$(oci vault secret create-base64 \
    --compartment-id "$COMPARTMENT_ID" \
    --vault-id "$VAULT_ID" \
    --key-id "$KEY_ID" \
    --secret-name "db-admin-password" \
    --description "Admin password for production database" \
    --secret-content-content "$DB_PASSWORD_B64" \
    --query 'data.id' --raw-output)
```

### Step 4: Retrieve Secrets at Runtime

This is where the real value is. Instead of hardcoding credentials, your application retrieves them:

```bash
# Retrieve and decode a secret
SECRET_VALUE=$(oci secrets secret-bundle get \
    --secret-id "$SECRET_ID" \
    --query 'data."secret-bundle-content".content' \
    --raw-output | base64 -d)

echo "Retrieved password: $SECRET_VALUE"
```

## Integration Patterns

### Pattern 1: Shell Script Configuration

Replace hardcoded values with runtime retrieval:

```bash
# Before (insecure)
DB_PASSWORD="MyP@ssw0rd123"

# After (secure)
DB_PASSWORD=$(oci secrets secret-bundle get \
    --secret-id "$DB_PASSWORD_SECRET_ID" \
    --query 'data."secret-bundle-content".content' \
    --raw-output | base64 -d)
```

### Pattern 2: Application Startup

For Python applications:

```python
import oci
import base64

secrets_client = oci.secrets.SecretsClient(config)

response = secrets_client.get_secret_bundle(secret_id=SECRET_OCID)
secret_value = base64.b64decode(
    response.data.secret_bundle_content.content
).decode('utf-8')
```

### Pattern 3: Kubernetes Integration

Use OCI Vault with the CSI Secrets Store Driver:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: oci-vault-secrets
spec:
  provider: oci
  parameters:
    secrets: |
      - name: db-password
        vaultId: ocid1.vault.oc1...
        secretId: ocid1.vaultsecret.oc1...
```

## Secret Rotation

OCI Vault supports multiple secret versions. To rotate a secret:

```bash
NEW_PASSWORD_B64=$(echo -n "NewSecureP@ss2026!" | base64)

oci vault secret update-base64 \
    --secret-id "$SECRET_ID" \
    --secret-content-content "$NEW_PASSWORD_B64"
```

The previous version is retained, allowing rollback if needed. Applications automatically get the latest version on their next retrieval.

## Security Best Practices

1. **Use IAM policies** — Restrict who can read vs. manage secrets
2. **Enable audit logging** — Track every secret access
3. **Rotate regularly** — Update secrets on a schedule
4. **Separate by environment** — Use different vaults for dev/staging/prod
5. **Never log secret values** — Mask sensitive data in application logs
6. **Use instance principals** — Avoid API keys when running on OCI compute

## Comparing OCI Vault with Alternatives

| Feature | OCI Vault | AWS Secrets Manager | Azure Key Vault | HashiCorp Vault |
|---------|-----------|--------------------|-----------------|--------------------|
| **Free tier** | 20 keys, 150 secrets | None ($0.40/secret/month) | 10K ops free | Open source (self-hosted) |
| **Managed** | Yes | Yes | Yes | Self-hosted or paid |
| **HSM support** | Yes (paid) | Yes | Yes | Yes (Enterprise) |
| **Secret rotation** | Manual + API | Auto-rotation | Auto-rotation | Auto-rotation |
| **OCI integration** | Native | N/A | N/A | Plugin |

OCI Vault's Always Free allocation makes it the clear winner for OCI-based workloads.

## The Complete Script

The full automation script is available on GitHub:

**Repository:** [github.com/pmady/oci-vault-secrets-automation](https://github.com/pmady/oci-vault-secrets-automation)

```bash
git clone https://github.com/pmady/oci-vault-secrets-automation.git
cd oci-vault-secrets-automation
chmod +x deploy_vault_secrets.sh
./deploy_vault_secrets.sh
```

## Conclusion

Hardcoding secrets is a habit that costs companies millions in breach remediation. OCI Vault provides a zero-cost path to proper secrets management. With 20 free encryption keys and 150 secret versions, there is no reason to keep passwords in your code.

Start small — move one secret to Vault today. Your future self (and your security team) will thank you.

---

*All resources in this post use OCI Always Free tier. No charges will be incurred.*

**Tags:** `#OracleCloud` `#Security` `#SecretsManagement` `#Vault` `#OCI` `#DevSecOps` `#AlwaysFree` `#CloudSecurity`
