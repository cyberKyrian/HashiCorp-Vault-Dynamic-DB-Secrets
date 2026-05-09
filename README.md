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

<img width="747" height="34" alt="image" src="https://github.com/user-attachments/assets/dce3636d-809a-4608-8c49-5c3829dc33f2" />


---

### 2. Deploy PostgreSQL

<img width="1302" height="98" alt="image" src="https://github.com/user-attachments/assets/169dfe0d-c9b0-4d1f-b6e2-5c22f9f66cd3" />

---

### 3. Deploy HashiCorp Vault in Dev Mode

The official Vault image requires a capability flag unsupported on Windows Docker Desktop. The fix is to pass the dev server command directly:

<img width="1302" height="429" alt="image" src="https://github.com/user-attachments/assets/1ecd0ede-b47c-420e-89ea-e8194242b969" />

Verify Vault is unsealed:

<img width="646" height="186" alt="image" src="https://github.com/user-attachments/assets/ba85210e-3387-4371-b01a-7fc9409ad6de" />


Vault UI dashboard after login with root token

<img width="920" height="383" alt="Screenshot 2026-05-09 231345" src="https://github.com/user-attachments/assets/2600acc8-a93d-44da-acb7-12c3d270fbaf" />

---

### 4. Enable the Database Secrets Engine

<img width="653" height="113" alt="Screenshot 2026-05-09 224803" src="https://github.com/user-attachments/assets/4c48e4d6-6ace-4e57-83b3-c1b2ef252868" />

---

### 5. Connect Vault to PostgreSQL

<img width="655" height="83" alt="Screenshot 2026-05-09 223211" src="https://github.com/user-attachments/assets/9c3900a4-fc83-4cb4-9089-0412e6e084ec" />

Vault uses the root credentials **only** to manage dynamic user lifecycle — the root password is never exposed to applications.

Vault database config written successfully

---

### 6. Create a Dynamic Role with TTL

<img width="653" height="113" alt="Screenshot 2026-05-09 224803" src="https://github.com/user-attachments/assets/83392368-ee26-44f0-bd77-0d5c8bc00856" />

This role:
- Generates a unique username and password on every request
- Grants **SELECT only** — least privilege by design
- Sets a **2 minute TTL** — credential auto-expires and is revoked

---

### 7. Generate a Dynamic Credential

<img width="661" height="131" alt="Screenshot 2026-05-09 225051" src="https://github.com/user-attachments/assets/4e063985-4d2b-4297-9275-06e4188e6c15" />
Output: every call generates a completely new, unique credential that has never existed before.

---

### 8. Verify the User Exists in PostgreSQL

Immediately after generation, confirm the user was created in the database:

```bash
docker exec -it vault-postgres psql -U root -d bankdb -c "\du"
```

Output shows the temporary Vault-managed user alongside permanent accounts:
<img width="887" height="137" alt="Screenshot 2026-05-09 225112" src="https://github.com/user-attachments/assets/caf5db5c-393a-4b51-af06-22b54bc2fb0a" />
Temporary user visible in PostgreSQL role list

---

### 9. Watch the Credential Auto-Revoke

After 2 minutes, run the same query:

<img width="923" height="186" alt="Screenshot 2026-05-09 225455" src="https://github.com/user-attachments/assets/72486468-835e-44c8-881e-c3991f3460c3" />

The dynamic user is gone — automatically revoked by Vault when the TTL expired. No manual intervention required.

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
- **Email:** mailto:c_kyrian@icloud.com
