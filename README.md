# HashiCorp Vault — Dynamic Database Secrets for PostgreSQL

## Project Overview

In most financial institutions, database credentials are **static** — hardcoded in application config files, shared across teams, and rotated infrequently. This is a critical risk: a single leaked password gives an attacker persistent, undetected access to sensitive data until someone manually intervenes.

This project demonstrates a **Zero Standing Access** approach to database credential management. I deployed HashiCorp Vault and integrated it with a PostgreSQL instance to generate **dynamic, just-in-time credentials** that are automatically revoked when their TTL expires — eliminating static passwords entirely.

This simulates the workflow of a **Database Security Engineer** or **IAM Engineer** implementing secrets management controls in a banking environment, directly addressing **PCI-DSS Requirement 8** (Identify Users and Authenticate Access).

---

## Tech Stack & Tools

| Component | Tool |
|---|---|
| **Secrets Manager** | HashiCorp Vault v1.15 |
| **Database** | PostgreSQL 15 |
| **Containerisation** | Docker Desktop (Windows) |
| **Networking** | Docker Bridge Network |
| **Environment** | Windows 11, PowerShell |

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Docker Network: vault-lab           │
│                                                      │
│   ┌──────────────────┐      ┌─────────────────────┐  │
│   │   Vault Server   │      │   PostgreSQL 15      │  │
│   │   (port 8200)    │─────▶│   (port 5433)        │  │
│   │                  │      │                      │  │
│   │  Database        │      │  - root (permanent)  │  │
│   │  Secrets Engine  │      │  - v-root-app-role-  │  │
│   │                  │      │    xxxxxx (dynamic,  │  │
│   │  TTL: 2 minutes  │      │    auto-expiring)    │  │
│   └──────────────────┘      └─────────────────────┘  │
│                                                      │
└─────────────────────────────────────────────────────┘

  App requests creds → Vault generates → TTL expires → Vault revokes
```

**Key Concepts Demonstrated:**
- **Dynamic Secrets** — credentials generated on demand, never stored
- **TTL-based Expiry** — credentials auto-revoked after 2 minutes
- **Least Privilege** — generated users receive SELECT-only access
- **Zero Standing Access** — no permanent application passwords exist

---

## Lab Execution Steps

### 1. Create a Shared Docker Network

Both containers need to communicate by hostname:

```bash
docker network create vault-lab
```

---

### 2. Deploy PostgreSQL

```bash
docker run --name vault-postgres \
  --network vault-lab \
  -e POSTGRES_PASSWORD=RootPass123 \
  -e POSTGRES_USER=root \
  -e POSTGRES_DB=bankdb \
  -p 5433:5432 \
  -d postgres:15
```

---

### 3. Deploy HashiCorp Vault in Dev Mode

The official Vault image requires a capability flag unsupported on Windows Docker Desktop. The fix is to pass the dev server command directly:

```bash
docker run --name vault-server \
  --network vault-lab \
  -e VAULT_DEV_ROOT_TOKEN_ID=root-token \
  -e SKIP_SETCAP=true \
  -p 8200:8200 \
  -d registry.hub.docker.com/hashicorp/vault:1.15 \
  vault server -dev -dev-root-token-id="root-token" -dev-listen-address="0.0.0.0:8200"
```

Verify Vault is unsealed:

```bash
docker exec -e VAULT_ADDR=http://127.0.0.1:8200 vault-server vault status
```

Expected output:
```
Sealed          false
Storage Type    inmem
Version         1.15.6
```

> **Screenshot:** Vault UI dashboard after login with root token
> *(add screenshot here)*

---

### 4. Enable the Database Secrets Engine

```bash
# Enter the container shell
docker exec -it vault-server sh

# Set environment variables
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root-token

# Enable the secrets engine
vault secrets enable database
```

Output:
```
Success! Enabled the database secrets engine at: database/
```

---

### 5. Connect Vault to PostgreSQL

```sh
vault write database/config/bankdb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-role" \
  connection_url="postgresql://{{username}}:{{password}}@vault-postgres:5432/bankdb?sslmode=disable" \
  username="root" \
  password="RootPass123"
```

Vault uses the root credentials **only** to manage dynamic user lifecycle — the root password is never exposed to applications.

> **Screenshot:** Vault database config written successfully
> *(add screenshot here)*

---

### 6. Create a Dynamic Role with TTL

```sh
vault write database/roles/app-role \
  db_name=bankdb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="2m" \
  max_ttl="5m"
```

This role:
- Generates a unique username and password on every request
- Grants **SELECT only** — least privilege by design
- Sets a **2 minute TTL** — credential auto-expires and is revoked

---

### 7. Generate a Dynamic Credential

```sh
vault read database/creds/app-role
```

Output:
```
Key                Value
---                -----
lease_id           database/creds/app-role/abc123xyz
lease_duration     2m
lease_renewable    true
password           A1s2-RandomGeneratedPassword
username           v-root-app-role-abc123xyz
```

Every call generates a completely new, unique credential that has never existed before.

> **Screenshot:** Dynamic credential generated by Vault
> *(add screenshot here)*

---

### 8. Verify the User Exists in PostgreSQL

Immediately after generation, confirm the user was created in the database:

```bash
docker exec -it vault-postgres psql -U root -d bankdb -c "\du"
```

Output shows the temporary Vault-managed user alongside permanent accounts:
```
                        List of roles
       Role name          |    Attributes
--------------------------+--------------
 root                     | Superuser, Create role, Create DB
 v-root-app-role-abc123   | Password valid until 2026-05-09 19:55:00
```

> **Screenshot:** Temporary user visible in PostgreSQL role list
> *(add screenshot here)*

---

### 9. Watch the Credential Auto-Revoke

After 2 minutes, run the same query:

```bash
docker exec -it vault-postgres psql -U root -d bankdb -c "\du"
```

The dynamic user is gone — automatically revoked by Vault when the TTL expired. No manual intervention required.

> **Screenshot:** User no longer present after TTL expiry
> *(add screenshot here)*

---

## Security Analysis

### The Problem This Solves

| Risk | Traditional Approach | Vault Dynamic Secrets |
|---|---|---|
| Static passwords | Same password for months/years | No static passwords exist |
| Shared credentials | Multiple apps use one DB user | Unique credential per request |
| Manual rotation | Rotated infrequently or never | Auto-revoked after TTL |
| Leaked credential impact | Persistent access until detected | Useless after 2 minutes |
| Audit trail | Who used the shared password? | Every credential tied to a lease |

---

## Compliance Mapping

| Control | PCI-DSS | CBN Cyber Framework |
|---|---|---|
| Unique user IDs per access request | Req 8.2.1 | CC-6 |
| Automatic credential expiry | Req 8.3.9 | CC-6 |
| Least privilege access | Req 7.2 | CC-6 |
| Access revocation | Req 8.3.4 | CC-7 |
| No shared credentials | Req 8.2.2 | CC-6 |

---

## Key Takeaways

- Vault's database secrets engine acts as a **credential broker** — applications never hold long-lived passwords
- **TTL enforcement** means a stolen credential has a maximum blast radius of 2 minutes
- **Least privilege at generation time** — the SQL role definition controls exactly what the dynamic user can do
- This pattern is directly applicable to **PAM (Privileged Access Management)** implementations in banking environments
- Vault's **lease system** creates a full audit trail of every credential issued, to whom, and when it was revoked

---

## 📬 Connect with Me

- **LinkedIn:** [Kyrian Onyeagusi](https://www.linkedin.com/in/kyrian-onyeagusi/)
- **Email:** c_kyrian@icloud.com
