# Lab 09 — Ansible Automation Hub

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 3 hours (Theory: 35% / Hands-on: 65%)                                      |
| **Outcome**  | Understand Automation Hub (public vs. private), browse collections, configure Hub as content source in Controller, install Private Hub (optional), upload/sync collections, manage execution environments |
| **Platform** | Red Hat Ansible Automation Platform **2.4** — Public Hub (console.redhat.com), Private Hub (optional) |
| **Prerequisite** | Labs 02, 04, 05 completed — AAP 2.4 Controller installed, project with playbooks, Red Hat account with AAP subscription. |

---

## Automation Hub Overview

| Hub Type | URL | Purpose |
|----------|-----|---------|
| **Public Automation Hub** | https://console.redhat.com/ansible/automation-hub | Red Hat–hosted; certified and validated collections; requires subscription |
| **Private Automation Hub** | Self-hosted (e.g., `https://hub.lab.local`) | Enterprise content governance; custom collections; isolated networks |
| **Ansible Galaxy** | https://galaxy.ansible.com | Community content; no subscription required |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                        │
│  - console.redhat.com (Public Hub)                                                   │
│  - https://aap1.lab.local (Controller)                                              │
│  - https://hub.lab.local (Private Hub - optional)                                    │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│  PUBLIC AUTOMATION HUB  │  │  AUTOMATION CONTROLLER │  │  PRIVATE AUTOMATION HUB │
│  (console.redhat.com)   │  │  (aap1.lab.local)      │  │  (hub.lab.local)        │
│  - Certified collections│  │  - Projects             │  │  - Custom collections   │
│  - Validated content    │  │  - Job templates       │  │  - Synced from Public   │
│  - API token           │  │  - Galaxy credential   │  │  - Execution envs       │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Red Hat account | With Ansible Automation Platform subscription |
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` with playbooks (from Lab 05) |
| Optional | Second VM for Private Automation Hub (cannot install Hub on same node as Controller) |

---

## Part 1 — Theory: Understanding Automation Hub (63 minutes)

### Section 8.1 — Understanding Automation Hub

#### What Is Ansible Automation Hub?

**Ansible Automation Hub** is the centralized repository for Ansible content:

| Content Type | Description |
|--------------|-------------|
| **Collections** | Modules, roles, plugins, playbooks packaged for distribution |
| **Execution Environments** | Container images with Ansible Core, collections, and dependencies |
| **Validated Content** | Red Hat–curated automation (e.g., RHEL hardening, network config) |

Hub provides **discovery**, **download**, and **governance** of automation content.

#### Public vs. Private Automation Hub

| Aspect | Public Hub | Private Hub |
|--------|------------|-------------|
| **Hosting** | Red Hat (console.redhat.com) | Self-hosted in your environment |
| **Content** | Certified, validated, Galaxy sync | Custom + synced from Public/Galaxy |
| **Access** | Internet required; API token | Internal network; no internet for isolated envs |
| **Use case** | Standard deployments; connected networks | Air-gapped; compliance; custom content |
| **Subscription** | Required for certified content | Requires AAP subscription |

#### Red Hat Certified Content Collections

- **Certified** = Jointly supported by Red Hat and partners (e.g., Cisco, F5, VMware).
- **Validated** = Red Hat–curated, tested automation (e.g., RHEL, security).
- **Subscription required** — access via API token from console.redhat.com.

#### Community Content and Galaxy

| Source | Content | Support |
|--------|---------|---------|
| **Ansible Galaxy** | Community collections (e.g., `community.general`, `community.aws`) | Community; no SLA |
| **Automation Hub (certified)** | Red Hat + partner certified | Red Hat + partner support |
| **Automation Hub (validated)** | Red Hat validated | Red Hat support |

#### Why Use Automation Hub?

| Benefit | Description |
|---------|-------------|
| **Trusted content** | Certified collections are tested and supported |
| **Version control** | Pin collection versions for reproducibility |
| **Air-gapped support** | Private Hub syncs content for isolated networks |
| **Governance** | Approve and distribute only approved content |
| **Execution environments** | Pre-built containers for consistent runs |

---

### Section 8.2 — Content in Automation Hub

#### What Are Ansible Content Collections?

A **collection** is a packaged unit of Ansible content:

| Component | Purpose |
|-----------|---------|
| **Modules** | Reusable tasks (e.g., `community.docker.docker_container`) |
| **Roles** | Pre-built automation (e.g., `ansible.posix.sysctl`) |
| **Plugins** | Inventory, connection, callback plugins |
| **Playbooks** | Example playbooks |

Collections use **namespaces** (e.g., `community.aws`, `redhat.rhel`) to avoid naming conflicts.

#### Browsing and Discovering Collections

- **Public Hub:** https://console.redhat.com/ansible/automation-hub → **Collections**
- **Private Hub:** `https://<hub-url>` → **Collections**
- Filter by namespace, name, tags; view documentation and dependencies.

#### Understanding Namespaces

| Namespace | Owner | Example |
|-----------|-------|---------|
| `ansible.builtin` | Ansible Core | Core modules |
| `redhat.rhel` | Red Hat | RHEL automation |
| `community.aws` | Community | AWS modules |
| `cisco.ios` | Cisco | Network modules |

#### Red Hat Supported vs. Community Content

| Type | Support | Source |
|------|---------|--------|
| **Certified** | Red Hat + partner | Public/Private Hub |
| **Validated** | Red Hat | Public/Private Hub |
| **Community** | Community | Galaxy; can sync to Private Hub |

#### Collection Documentation and Dependencies

- Each collection has a **manifest** (`galaxy.yml`) with dependencies.
- Installing a collection can pull dependent collections automatically.
- Use `requirements.yml` to pin versions and avoid surprises.

---

### Section 8.3 — Private Automation Hub Setup

#### Installing Private Automation Hub

| Requirement | Details |
|-------------|---------|
| **Node** | Separate from Controller (cannot co-locate) |
| **OS** | RHEL 8.4+ or RHEL 9 (64-bit x86) |
| **Resources** | 8 GB RAM, 2 CPUs, 60 GB disk |
| **Installer** | AAP Platform Installer (same bundle as Controller) |

**Inventory example** (installer):

```ini
[automationhub]
hub.lab.local

[automationhub:vars]
automationhub_admin_password='YourPassword'
```

#### Basic Configuration

- **URL:** `https://hub.lab.local` (or your FQDN)
- **Admin user:** Created during install
- **Repositories:** `rh-certified`, `community` (remotes to sync from)

#### User and Permission Management

| Role | Capability |
|------|------------|
| **Admin** | Full access; manage users, remotes, content |
| **Content Admin** | Manage collections; approve/reject |
| **Namespace Owner** | Manage own namespace and collections |

#### Creating Repositories

- **rh-certified** — Sync from console.redhat.com (certified content)
- **community** — Sync from galaxy.ansible.com
- **Custom** — For uploaded internal collections

#### Content Approval Workflows

- Enable **require_content_approval** — uploaded collections need approval before publish.
- Content admins approve or reject; only approved content is available to users.

---

### Section 8.4 — Managing Content

#### Uploading Custom Collections

1. Build collection: `ansible-galaxy collection build`
2. In Private Hub: **Collections** → **Upload** → select `.tar.gz`
3. (If approval enabled) Content admin approves
4. Collection appears in the repository

#### Syncing Content from console.redhat.com

- **Remotes** → **rh-certified** → **Edit** → set Sync URL and API token
- **Sync** — pulls certified collections to Private Hub
- **Requirements file** — upload a `requirements.yml` to sync only selected collections

#### Managing Execution Environment Container Images

- **Execution Environments** in Hub — container images for Controller jobs
- Build with `ansible-builder`; push to Hub's container registry
- Controller references Hub as container registry URL

#### Content Signing and Verification

- **Signing** — GPG-sign collections for integrity
- **Verification** — `ansible-galaxy collection install ... --verify` with public key
- Configure signing service in Private Hub for automated signing

#### Version Management

- Collections have semantic versions (e.g., `1.2.3`)
- Pin in `requirements.yml`: `version: ">=1.0.0,<2.0.0"` or `version: "1.2.3"`
- Sync specific versions to Private Hub via requirements file

---

### Section 8.5 — Integrating Hub with Controller

#### Configuring Automation Hub as Content Source

| Method | Where | Purpose |
|--------|-------|---------|
| **Galaxy Credential** | Organization → Galaxy Credentials | Controller uses this for project sync (collections) |
| **ansible.cfg** | CLI / build environments | `ansible-galaxy` and `ansible-builder` |

#### Authentication Setup

- **Public Hub:** API token from console.redhat.com → **Connect to Hub** → **Load token**
- **Private Hub:** Username + password, or API token from Hub UI
- **Credential type:** Ansible Galaxy / Automation Hub API Token

#### Automatic Collection Downloads

- When a project has `collections/requirements.yml`, Controller installs collections during **Project Sync**
- Collections are installed into the Execution Environment or project directory (depending on config)
- Galaxy Credential must be set at **Organization** level

#### Specifying Collections in Projects

**collections/requirements.yml** in project root:

```yaml
collections:
  - name: ansible.posix
    version: ">=1.5.0"
  - name: community.general
    version: "5.0.0"
  - name: redhat.rhel_system_roles
    version: "1.0.0"
```

Controller (or `ansible-galaxy`) installs these when the project syncs.

---

## Part 2 — Hands-On: Explore Public Automation Hub (25 minutes)

### Step 1 — Browse Public Hub and Create API Token

**Purpose:** Access certified content and obtain credentials for Controller integration.

**Detailed steps:**

1. **Log in to console.redhat.com** with your Red Hat account.
2. **Navigate:** **Ansible** → **Automation Hub** (or go to https://console.redhat.com/ansible/automation-hub).
3. **Browse collections:** Click **Collections**. Filter by **Certified** or **Validated**. Search for `redhat.rhel` or `community.general`.
4. **Create API token:** **Connect to Hub** (or **Get API token**) → **Load token** → copy the **Offline token** (or generate new token). Save it securely.
5. **Note the URLs:**
   - **Galaxy API URL:** `https://console.redhat.com/api/automation-hub/content/published/` (trailing slash required)
   - **Auth URL:** `https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token`

---

## Part 3 — Hands-On: Integrate Public Hub with Controller (35 minutes)

### Step 2 — Create Galaxy Credential in Controller

**Purpose:** Allow Controller to download collections from Public Hub during project sync.

**Detailed steps:**

1. **Resources → Credentials → Add**
2. **Credential type:** Select **Ansible Galaxy / Automation Hub API Token**
3. **Name:** `Automation-Hub-Token`
4. **Organization:** `Default`
5. **Galaxy Server URL:** `https://console.redhat.com/api/automation-hub/content/published/`
6. **Auth Server URL:** `https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token`
7. **API Token:** Paste the token from Step 1
8. **Save**

### Step 3 — Add Galaxy Credential to Organization

**Purpose:** Make the credential available for project syncs in the organization.

**Detailed steps:**

1. **Access → Organizations** → open `Default`
2. **Galaxy Credentials** — click the search icon
3. **Add** `Automation-Hub-Token` (place it first if you have multiple)
4. **Save**

### Step 4 — Add requirements.yml to Project and Sync

**Purpose:** Specify collections for the project; Controller installs them during sync.

**Detailed steps:**

1. **Create `collections/requirements.yml`** in your project (Git repo or manual project):

```yaml
collections:
  - name: ansible.posix
    version: ">=1.0.0"
  - name: community.general
    version: ">=6.0.0"
```

2. **If using Git:** Push to repo. **If using Manual project:** Create on AAP server:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-playbooks/collections
sudo tee /var/lib/awx/projects/lab-playbooks/collections/requirements.yml > /dev/null << 'EOF'
collections:
  - name: ansible.posix
    version: ">=1.0.0"
  - name: community.general
    version: ">=6.0.0"
EOF
sudo chown -R awx:awx /var/lib/awx/projects/lab-playbooks
```

3. **In Controller:** **Resources → Projects** → open `Lab-Project` → **Sync** (refresh icon)
4. **Verify:** Project sync should succeed. Check job output — it should show collection installation from Automation Hub.

### Step 5 — Use a Collection in a Playbook

**Purpose:** Confirm collections are available and used in a job.

**Detailed steps:**

1. **Create a playbook** that uses a collection (e.g., `community.general.archive`):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/test_collection.yml > /dev/null << 'EOF'
---
- name: Test collection (community.general)
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Create temp file
      ansible.builtin.tempfile:
        state: file
      register: tmp
    - name: Create archive (community.general)
      community.general.archive:
        path: ["{{ tmp.path }}"]
        dest: /tmp/test-archive.tar.gz
    - name: Cleanup
      ansible.builtin.file:
        path: "{{ tmp.path }}"
        state: absent
    - name: Remove archive
      ansible.builtin.file:
        path: /tmp/test-archive.tar.gz
        state: absent
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/test_collection.yml
```

2. **Create job template** with this playbook. **Launch** the job — it should succeed, confirming `community.general` is installed. If the archive module fails, verify project sync completed and check the job output for collection installation messages.

---

## Part 4 — Hands-On: Private Automation Hub (Optional, 45 minutes)

> **Note:** Private Hub requires a **separate VM**. Controller and Hub cannot run on the same node. Skip this part if you do not have a second VM.

### Step 6 — Install Private Automation Hub

**Purpose:** Deploy Private Hub for air-gapped or governed content distribution.

**Detailed steps:**

1. **Prepare a RHEL 8/9 VM** (separate from Controller): 8 GB RAM, 2 CPUs, 60 GB disk.
2. **Download AAP installer** (same bundle as Lab 02) and extract.
3. **Create inventory** for Hub:

```ini
[automationhub]
hub.lab.local

[all:vars]
automationhub_admin_password='YourSecurePassword'
```

4. **Run installer:** `ansible-playbook -i inventory install.yml`
5. **Access Hub:** `https://hub.lab.local` (accept self-signed cert). Log in with admin and the password.

### Step 7 — Configure rh-certified Remote and Sync

**Purpose:** Sync certified collections from console.redhat.com to Private Hub.

**Detailed steps:**

1. **Log in to Private Hub** as admin.
2. **Collections → Remotes** → **rh-certified** → **Edit** (⋮ menu)
3. **URL:** Use the Sync URL from console.redhat.com (Automation Hub → Connect to Hub)
4. **Token:** Paste API token from console.redhat.com
5. **Save**
6. **Sync:** Click **⋮** → **Sync**. Wait for sync to complete.
7. **Verify:** **Collections** → filter by **Red Hat Certified** — collections should appear.

### Step 8 — Upload a Custom Collection (Optional)

**Purpose:** Add internal or custom content to Private Hub.

**Detailed steps:**

1. **Create a minimal collection** (on your laptop or AAP server):

```bash
mkdir -p my_namespace/my_collection
cd my_namespace/my_collection
cat > galaxy.yml << 'EOF'
namespace: my_namespace
name: my_collection
version: 1.0.0
EOF
mkdir -p roles/my_role/tasks
echo "- debug: msg='Hello from custom collection'" > roles/my_role/tasks/main.yml
ansible-galaxy collection build
```

2. **In Private Hub:** **Collections** → **Upload** → select `my_namespace-my_collection-1.0.0.tar.gz`
3. **Approve** (if content approval is enabled)
4. **Verify** the collection appears in the repository

### Step 9 — Configure Controller to Use Private Hub

**Purpose:** Point Controller to Private Hub instead of (or in addition to) Public Hub.

**Detailed steps:**

1. **Create Galaxy credential** for Private Hub:
   - **Galaxy Server URL:** `https://hub.lab.local/api/galaxy/content/rh-certified/` (or your repo path)
   - **Auth:** Username/password or token (depending on Hub config)
2. **Add to Organization** Galaxy Credentials — place Private Hub first if you want it as primary
3. **Sync a project** — collections should be pulled from Private Hub

---

## Part 5 — Real-Time Use Case Discussion (20 minutes)

### Case Study: Government Agency — Certified Content for Isolated Networks

**Background:**

A government agency runs automation in **air-gapped** and **isolated** networks. They cannot access console.redhat.com or the public internet from production. Compliance requires only approved, certified automation content.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| No internet in prod | Cannot download collections from Public Hub or Galaxy |
| Compliance | Only certified, approved content allowed |
| Multiple networks | Dev (connected), Staging (semi-isolated), Prod (air-gapped) |
| Content updates | Must control when new versions are introduced |

**Solution: Private Automation Hub**

| Component | Implementation |
|-----------|----------------|
| **Private Hub** | Installed in a network that can reach console.redhat.com (e.g., staging) |
| **Sync** | rh-certified remote syncs only approved collections (via requirements file) |
| **Content approval** | Enable `require_content_approval` — admins approve before publish |
| **Distribution** | Prod Controllers pull from Private Hub (reachable via internal network) |
| **Execution environments** | Build EEs with approved collections; push to Hub; Controllers use them |
| **Version pinning** | requirements.yml pins versions; no surprise updates |

**Workflow:**

```
console.redhat.com  →  (sync)  →  Private Hub (staging)  →  (internal)  →  Controllers (prod)
       ↑                                    ↑
   Certified content              Content approval gate
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Compliance | Manual review of every collection | Centralized approval in Hub |
| Update cycle | 3–6 months (manual) | 2–4 weeks (controlled sync) |
| Security | Unverified community content | Certified + signed only |
| Audit | Ad hoc | Full traceability (who approved, when) |

**Key takeaway:** Private Automation Hub enables **governed, certified content distribution** for isolated and compliance-sensitive environments. Sync from Public Hub in a connected zone, approve content, and distribute to air-gapped Controllers.

---

## Troubleshooting

### Project Sync Fails — "Could not find a collection"

| Symptom | Check |
|---------|-------|
| Galaxy credential missing | Add credential to Organization → Galaxy Credentials |
| Wrong URL | Use trailing slash; correct path (e.g., `/api/automation-hub/content/published/`) |
| Token expired | Generate new token in console.redhat.com |
| Collection not in Hub | Certified content requires subscription; check namespace (e.g., `redhat.rhel`) |

### Private Hub Sync Fails

| Symptom | Check |
|---------|-------|
| Network/firewall | Hub must reach console.redhat.com, sso.redhat.com |
| Token | Use token from console.redhat.com; org admin permissions |
| Proxy | Configure proxy in remote if behind corporate proxy |

### Collection Not Found in Job

| Symptom | Check |
|---------|-------|
| requirements.yml | Must be in `collections/requirements.yml` in project root |
| Project sync | Re-sync project after adding requirements.yml |
| Execution environment | If using custom EE, ensure EE has the collection or project sync installs to project |

### API Token Revoked

- Creating a new token **revokes** previous tokens. Update Controller credential and any scripts.

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Public vs. Private Hub | Hosting, content, use cases |
| Certified vs. Community | Support model; where to get each |
| Galaxy credential | Controller uses it for collection install during project sync |
| requirements.yml | Specifies collections and versions for projects |
| Namespace | Organizes collections (e.g., `community.aws`) |
| Private Hub sync | Remotes pull from console.redhat.com or Galaxy |
| Content approval | Governance for uploaded content |
| Execution environments | Container images; can be stored in Hub |

---

## Lab Completion Checklist

- [ ] Browsed Public Automation Hub; created API token
- [ ] Created Galaxy credential in Controller
- [ ] Added Galaxy credential to Organization
- [ ] Created collections/requirements.yml in project
- [ ] Project sync succeeded; collections installed
- [ ] Playbook using a collection ran successfully
- [ ] (Optional) Private Hub installed and configured
- [ ] (Optional) Synced certified content to Private Hub
- [ ] (Optional) Uploaded custom collection
- [ ] Can explain the government agency use case

---

## Quick Reference — Hub URLs (AAP 2.4)

| Hub | Galaxy API URL |
|-----|----------------|
| Public (console.redhat.com) | `https://console.redhat.com/api/automation-hub/content/published/` |
| Private (rh-certified repo) | `https://<hub-fqdn>/api/galaxy/content/rh-certified/` |
| Private (community repo) | `https://<hub-fqdn>/api/galaxy/content/community/` |

---

## Quick Reference — requirements.yml Format

```yaml
collections:
  - name: namespace.collection_name
    version: "1.0.0"          # Exact version
  - name: namespace.other
    version: ">=2.0.0,<3.0"  # Version range
  - name: namespace.third
    source: https://galaxy.ansible.com  # From Galaxy
```

---

*End of Lab 09*
