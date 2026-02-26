# Lab 04 — Manage Inventories, Credentials/Secrets & Integrations

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 25% / Hands-on: 75%)                                     |
| **Outcome**  | Create and manage static and dynamic inventories, configure host/group variables, build smart inventories, set up multiple credential types, integrate with Git and notification services, and understand external secret management |
| **Platform** | Red Hat Ansible Automation Platform 2.4 / 2.5                              |
| **Prerequisite** | Lab 02 completed — AAP installed with web UI access. Lab 03 optional (multi-node setup). |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                        │
│                         https://aap1.lab.local                                       │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ HTTPS (443)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                      │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Inventories │  │  Credentials │  │   Projects   │  │  Notification Templates  │ │
│  │  - Static    │  │  - Machine   │  │  - Git SCM   │  │  - Slack                 │ │
│  │  - Dynamic  │  │  - SCM       │  │  - Manual    │  │  - Email                 │ │
│  │  - Smart     │  │  - Vault     │  │              │  │  - Webhook               │ │
│  │              │  │  - AWS/Azure │  │              │  │                          │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ SSH (22)
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │ web1.lab.local│  │ app1.lab.local│  │ db1.lab.local  │
            │ [webservers]  │  │ [appservers]  │  │ [databases]    │
            │ env: prod     │  │ env: prod     │  │ env: prod      │
            └───────────────┘  └───────────────┘  └───────────────┘

External Integrations:
  - GitHub/GitLab (Source Control)
  - Slack (Notifications)
  - Email (Notifications)
  - AWS EC2 (Dynamic Inventory - optional)
```

> **Note:** In a minimal lab, you can use 2–3 VMs and assign multiple groups to the same host. Replace hostnames and IPs with your actual values.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) |
| Managed Nodes | 2–4 Linux VMs (RHEL, Ubuntu, AlmaLinux) — can reuse nodes from Lab 02 |
| Network | AAP can SSH to all managed nodes |
| Optional | GitHub/GitLab account for SCM integration; Slack workspace for notifications |
| Optional | AWS account for EC2 dynamic inventory demo |

---

## Part 1 — Theory: Understanding Inventories, Credentials & Integrations (60 minutes)

### Section 1 — Understanding Inventories

#### What Is an Inventory in Ansible?

An **inventory** is the list of hosts (servers, network devices, cloud instances) that Ansible manages. It answers the question: *"On which machines should this automation run?"*

| Concept | Description |
|---------|-------------|
| Inventory | A container for hosts and groups — scoped to an Organization |
| Host | A single managed target (IP, hostname, or alias) |
| Group | A logical collection of hosts (e.g., `webservers`, `prod`, `us-east`) |
| Variable | Key-value data attached to hosts or groups (e.g., `ansible_user`, `env`) |

#### Static vs. Dynamic Inventories

| Type | Description | When to Use |
|------|-------------|-------------|
| **Static** | Manually maintained list of hosts. You add/remove hosts in the UI or via API. | Small environments, predictable infrastructure, lab/POC |
| **Dynamic** | Automatically populated from an external source (cloud API, CMDB, script). Inventory is refreshed on sync. | Cloud environments (AWS, Azure, GCP), large-scale, frequently changing infrastructure |

#### Inventory Variables and Their Uses

Variables can be set at multiple levels. Ansible merges them with **variable precedence** (more specific overrides less specific):

| Level | Example | Use Case |
|-------|---------|----------|
| Host | `web1` has `http_port: 8080` | Host-specific overrides |
| Group | `webservers` has `ansible_user: deploy` | Shared config for a group |
| Group of groups | `prod` includes `webservers` + `appservers` | Environment-wide settings |
| Inventory | `all` has `ansible_python_interpreter: /usr/bin/python3` | Global defaults |

#### Organizing Hosts into Groups

Groups enable **targeted automation** — run a playbook only on `webservers`, or only on `prod` hosts.

```
all
├── prod
│   ├── webservers    (web1, web2)
│   ├── appservers    (app1, app2)
│   └── databases     (db1)
├── nonprod
│   ├── webservers    (web-dev1)
│   └── appservers    (app-dev1)
└── _ungrouped        (hosts not in any group)
```

#### Inventory Synchronization Basics

For **dynamic** inventory sources, AAP must **sync** to pull the latest hosts and groups:

| Sync Trigger | When It Happens |
|--------------|-----------------|
| **Manual** | Click **Sync all** (or sync icon) on the inventory source |
| **Update on launch** | Automatically syncs when a job template using this inventory is launched |
| **Scheduled** | Configure a schedule on the inventory source (e.g., every hour) |

After sync, hosts and groups are updated. Failed syncs show an error; fix the source (file path, script, cloud credentials) and sync again.

#### Smart Inventories

A **smart inventory** is a **filtered view** of existing inventories. You define filters (host name, group, variable) and AAP shows only matching hosts. The underlying hosts are not duplicated — it is a dynamic query.

| Use Case | Smart Inventory Filter Example |
|----------|-------------------------------|
| All production hosts | `groups__name__contains=prod` |
| Hosts in a specific region | `variables__region=us-east-1` |
| Web servers only | `groups__name=webservers` |
| Hosts with a variable set | `variables__env__isnull=false` |

---

### Section 2 — Credentials Management Fundamentals

#### Why Credentials Are Needed

Ansible needs to **authenticate** when it:

- Connects to managed hosts (SSH/WinRM)
- Clones code from Git
- Calls cloud APIs (AWS, Azure, GCP)
- Decrypts Ansible Vault–encrypted data
- Connects to network devices

Credentials are stored **encrypted** in the AAP database and are never exposed in job output or logs.

#### Types of Credentials in AAP

| Credential Type | Purpose | Typical Fields |
|-----------------|---------|----------------|
| **Machine** | SSH or WinRM access to Linux/Windows hosts | Username, SSH key or password, port |
| **Source Control** | Git (GitHub, GitLab, Bitbucket) | URL, username, token or SSH key |
| **Amazon Web Services** | AWS API access | Access key, secret key, or IAM role |
| **Microsoft Azure** | Azure API access | Subscription ID, tenant, client ID, secret |
| **Google Compute Engine** | GCP API access | Service account JSON |
| **Ansible Vault** | Decrypt vault-encrypted files | Vault password |
| **Network** | Network device access (Cisco, Juniper) | Username, password, enable secret |
| **OpenStack** | OpenStack cloud | Auth URL, project, username, password |
| **Red Hat Subscription Manager** | RHEL subscription | Username, password |
| **Vault** | HashiCorp Vault integration | Token, role ID, secret ID |

#### Best Practices for Credential Security

| Practice | Description |
|----------|-------------|
| Use SSH keys over passwords | Keys are more secure and support automation without prompts |
| Rotate credentials regularly | Especially after team changes or suspected compromise |
| Limit credential scope | Use RBAC so only authorized users/jobs can use sensitive creds |
| Use external secret managers | For high-security environments, integrate with Vault, CyberArk, or Azure Key Vault |
| Avoid sharing credentials | Each team member should use their own credentials where possible |
| Audit credential usage | Review who has access and when credentials were last used |

---

### Section 3 — External Secret Management (Overview)

#### When to Use External Secret Management

| Scenario | Recommendation |
|----------|----------------|
| Small lab, few servers | AAP built-in credential vault is sufficient |
| Enterprise, compliance requirements | Integrate with HashiCorp Vault, CyberArk, or Azure Key Vault |
| Credentials change frequently | External manager allows rotation without updating AAP |
| Centralized secret governance | One source of truth for all applications |
| Audit requirements (SOC2, PCI-DSS) | External tools provide detailed audit trails |

#### Integration Options (High-Level)

| Tool | Use Case | How It Works with AAP |
|------|----------|------------------------|
| **HashiCorp Vault** | Dynamic secrets, lease-based access | AAP uses Vault credential type; fetches secrets at job runtime |
| **CyberArk** | Enterprise PAM, privileged access | CyberArk Application Identity Manager (AIM) or Conjur integration |
| **Azure Key Vault** | Azure-centric environments | AAP can reference secrets from Key Vault via Azure credential + custom logic |
| **Red Hat Ansible Automation Platform** | Built-in | Credentials stored encrypted in Controller database |

> **Note:** Full external secret manager setup requires additional infrastructure. This lab focuses on AAP's built-in credentials. External integration is covered conceptually for awareness.

---

### Section 4 — Basic Integrations

#### Source Control (Git) Integration

| Component | Purpose |
|-----------|---------|
| **Project** | Links AAP to a Git repository (GitHub, GitLab, Bitbucket, internal Git) |
| **Source Control Credential** | Authenticates to the repo (HTTPS token or SSH key) |
| **Project Update** | Syncs the latest code from Git to AAP; can be manual or scheduled |
| **SCM Type** | Git, SVN, or Manual (local directory) |

#### Cloud Provider Integrations

| Cloud | Credential Type | Typical Use |
|-------|-----------------|-------------|
| AWS | Amazon Web Services | Dynamic inventory (EC2), cloud modules (ec2_instance, s3) |
| Azure | Microsoft Azure | Dynamic inventory (azure_rm), cloud modules |
| GCP | Google Compute Engine | Dynamic inventory, cloud modules |

#### Notification Integrations

| Type | Purpose |
|------|---------|
| **Email** | Send job status (success/failure) to a list of recipients |
| **Slack** | Post job results to a Slack channel via webhook |
| **Webhook** | Call an external URL with job metadata (custom integrations) |
| **PagerDuty** | Alert on job failures for on-call response |
| **IRC** | Legacy — post to IRC channel |

#### Webhook Integrations

A **webhook** allows external systems to **trigger** AAP jobs:

- CI/CD pipeline (Jenkins, GitLab CI) calls AAP API or webhook URL when a build completes
- Monitoring tool sends alert → webhook → AAP runs remediation playbook
- Git push event → webhook → AAP runs deployment

---

## Part 2 — Hands-On: Static Inventory and Groups (45 minutes)

### Step 1 — Create a Static Inventory

1. Log in to AAP: `https://aap1.lab.local`
2. Navigate to **Resources --> Inventories**
3. Click **Add --> Add inventory**
4. Fill in:
   - **Name:** `Lab-Static-Inventory`
   - **Organization:** `Default`
   **Description:** `Static inventory for lab - webservers, appservers, databases`
5. Click **Save**

### Step 2 — Create Groups

1. Open `Lab-Static-Inventory`
2. Click the **Groups** tab
3. Click **Add**
4. Create these groups one by one:

| Group Name | Description |
|------------|-------------|
| `webservers` | Web tier hosts |
| `appservers` | Application tier hosts |
| `databases` | Database tier hosts |
| `prod` | Production environment (parent group) |

5. After creating `prod`, edit it and add **Child Groups**: `webservers`, `appservers`, `databases` so that `prod` contains all three.

### Step 3 — Add Hosts to Groups

1. Click the **Hosts** tab
2. Click **Add**
3. Add hosts. Use your actual hostnames or IPs. Example:

| Host Name | Groups | Notes |
|-----------|--------|-------|
| `web1.lab.local` | webservers, prod | Or use `node1.lab.local` if you have only 2 nodes |
| `app1.lab.local` | appservers, prod | Can be same as web1 in minimal lab |
| `db1.lab.local` | databases, prod | Can be same as web1 in minimal lab |

> **Minimal lab:** If you have only 2 nodes (`node1`, `node2`), add both and assign `node1` to `webservers` and `prod`, and `node2` to `appservers` and `prod`. You can add the same host to multiple groups.

4. For each host, set **Variables** (click the host, then **Variables** tab) if needed:

```yaml
---
ansible_host: 192.168.1.20
env: prod
```

### Step 4 — Set Group Variables

1. Go to **Groups** tab
2. Click on `webservers`
3. Click **Variables** tab
4. Add:

```yaml
---
env: prod
tier: web
```

5. Repeat for `appservers` and `databases` with appropriate `tier` values.

### Step 5 — Verify Variable Precedence

Create a host variable that overrides the group:

1. Edit `web1.lab.local` (or your first host)
2. **Variables** tab, add:

```yaml
---
custom_http_port: 8080
```

When a playbook uses `{{ custom_http_port }}` on this host, it will get `8080`; other hosts in `webservers` without this variable would need a default or would fail.

---

## Part 3 — Hands-On: Dynamic Inventory (45 minutes)

### Option A — File-Based Dynamic Inventory (No Cloud Required)

AAP can use a **file** as an inventory source. The file can be updated externally (script, CI/CD) and AAP syncs it.

#### Step 6 — Create an Inventory File on the AAP Server

SSH to the AAP server and create a file:

```bash
sudo mkdir -p /var/lib/awx/inventory_files
sudo tee /var/lib/awx/inventory_files/dynamic_hosts.yml > /dev/null << 'EOF'
---
all:
  children:
    webservers:
      hosts:
        web1.lab.local:
          ansible_host: 192.168.1.20
          env: prod
        web2.lab.local:
          ansible_host: 192.168.1.21
          env: prod
    appservers:
      hosts:
        app1.lab.local:
          ansible_host: 192.168.1.22
          env: prod
    databases:
      hosts:
        db1.lab.local:
          ansible_host: 192.168.1.23
          env: prod
EOF
```

> **Replace IPs** with your actual managed node IPs. For a 2-node lab, use the same IP for multiple hosts or reduce the list.

```bash
sudo chown -R awx:awx /var/lib/awx/inventory_files
```

#### Step 7 — Create Inventory with File Source in AAP

1. **Resources --> Inventories --> Add --> Add inventory**
2. **Name:** `Lab-Dynamic-Inventory`
3. **Organization:** `Default`
4. Click **Save**
5. Click **Sources** tab
6. Click **Add**
7. Fill in:
   - **Name:** `File-Source`
   - **Source:** `File, directory or script`
   - **File:** Browse to `/var/lib/awx/inventory_files/dynamic_hosts.yml` (or type the path)
   - **Update on launch:** Check this so the inventory refreshes when a job uses it
8. Click **Save**
9. Click **Sync all** (or the sync icon next to the source)
10. After sync, check **Hosts** and **Groups** — they should be populated from the file

### Option B — Project-Sourced Dynamic Inventory (Script in Git)

If you use a Git project, you can place an inventory script in the repo that outputs JSON.

#### Step 8 — Create an Inventory Script (Alternative to Option A)

On your laptop or a dev machine, create a repo with:

**inventory/dynamic_inventory.py**

```python
#!/usr/bin/env python3
"""Simple dynamic inventory for AAP lab - outputs Ansible JSON inventory."""
import json
import os

# In production, this would call AWS API, Azure API, or a CMDB
# For lab, we use environment variables or a config file
hosts = {
    "webservers": ["web1.lab.local", "web2.lab.local"],
    "appservers": ["app1.lab.local"],
    "databases": ["db1.lab.local"],
}

# Build Ansible inventory structure
inventory = {
    "_meta": {"hostvars": {}},
    "all": {"children": ["webservers", "appservers", "databases"]},
    "webservers": {"hosts": hosts["webservers"]},
    "appservers": {"hosts": hosts["appservers"]},
    "databases": {"hosts": hosts["databases"]},
}

# Add host vars (replace with your IPs)
ip_map = {
    "web1.lab.local": "192.168.1.20",
    "web2.lab.local": "192.168.1.21",
    "app1.lab.local": "192.168.1.22",
    "db1.lab.local": "192.168.1.23",
}
for h in sum(hosts.values(), []):
    inventory["_meta"]["hostvars"][h] = {"ansible_host": ip_map.get(h, h)}

print(json.dumps(inventory, indent=2))
```

Make it executable: `chmod +x inventory/dynamic_inventory.py`

Push to GitHub/GitLab. Then in AAP:

1. Create a **Project** pointing to this repo (with SCM credential if private)
2. Create inventory `Lab-Script-Inventory`
3. Add **Source:** `Sourced from a project`
4. **Project:** your repo
5. **Script:** `inventory/dynamic_inventory.py`
6. **Update on launch:** Checked
7. Sync

### Option C — AWS EC2 Dynamic Inventory (If You Have AWS)

1. Create inventory `Lab-AWS-Inventory`
2. Add **Source:** `Amazon ec2`
3. **Credential:** Create an **Amazon Web Services** credential with Access Key and Secret Key
4. **Regions:** `us-east-1` (or your region)
5. **Instance filters** (optional): `tag:Environment=prod`
6. **Update on launch:** Checked
7. Sync — AAP will pull all EC2 instances (or filtered) as hosts

---

## Part 4 — Hands-On: Smart Inventory (20 minutes)

### Step 9 — Create a Smart Inventory

1. **Resources --> Inventories --> Add --> Add smart inventory**
2. **Name:** `Lab-Prod-Web-Only`
3. **Organization:** `Default`
4. **Inventory:** `Lab-Static-Inventory` (or `Lab-Dynamic-Inventory`)
5. **Smart host filter:** Use the query builder or enter:

```
groups__name=webservers
```

This shows only hosts in the `webservers` group.

6. **Limit:** (optional) Leave blank for no limit
7. Click **Save**
8. Open the smart inventory — **Hosts** tab shows only filtered hosts

### Step 10 — Create Another Smart Inventory (Multi-Condition)

1. **Add --> Add smart inventory**
2. **Name:** `Lab-Prod-All-Tiers`
3. **Inventory:** `Lab-Static-Inventory`
4. **Smart host filter:**

```
groups__name__in=webservers,appservers,databases
```

Or, if using a `prod` parent group:

```
groups__name=prod
```

5. Save and verify the host list.

---

## Part 5 — Hands-On: Credentials Management (50 minutes)

### Step 11 — Create Machine Credential (SSH Key)

1. **Resources --> Credentials --> Add**
2. Fill in:
   - **Name:** `SSH-Key-WebServers`
   - **Organization:** `Default`
   - **Credential Type:** `Machine`
   - **Username:** `ec2-user` or `root` (match your nodes)
   - **SSH Private Key:** Paste your private key content
   - **Privilege Escalation:** Enable if you need sudo
3. Click **Save**

### Step 12 — Create Source Control Credential (GitHub/GitLab)

1. **Resources --> Credentials --> Add**
2. Fill in:
   - **Name:** `GitHub-Personal-Access-Token`
   - **Organization:** `Default`
   - **Credential Type:** `Source Control`
   - **Source Control Type:** `Git`
   - **Username:** your GitHub username (or leave blank for token-only)
   - **Password:** Your Personal Access Token (not your GitHub password)
   - **Project:** (optional) Leave blank for org-wide use
3. Click **Save**

> **Getting a GitHub PAT:** GitHub → Settings → Developer settings → Personal access tokens → Generate new token. Scopes: `repo`, `read:org` (if using private org repos).

### Step 13 — Create Ansible Vault Credential

1. **Resources --> Credentials --> Add**
2. Fill in:
   - **Name:** `Vault-Lab-Secrets`
   - **Organization:** `Default`
   - **Credential Type:** `Ansible Vault`
   - **Vault Password:** A password you will use to encrypt/decrypt vault files
3. Click **Save**

> **Usage:** When a playbook or variable file is encrypted with `ansible-vault encrypt`, the job template must have this credential so AAP can decrypt it at runtime.

### Step 14 — Create AWS Credential (Optional — If You Have AWS)

1. **Resources --> Credentials --> Add**
2. Fill in:
   - **Name:** `AWS-Lab-Access`
   - **Organization:** `Default`
   - **Credential Type:** `Amazon Web Services`
   - **Access Key:** Your AWS access key ID
   - **Secret Key:** Your AWS secret access key
   - **Region:** `us-east-1` (or your region)
3. Click **Save**

### Step 15 — Create Network Credential (For Network Devices)

1. **Resources --> Credentials --> Add**
2. Fill in:
   - **Name:** `Network-Cisco-Lab`
   - **Organization:** `Default`
   - **Credential Type:** `Network`
   - **Username:** `admin`
   - **Password:** device password
   - **Authorize:** Enable and set **Authorize password** if the device uses enable secret
3. Click **Save**

> **Note:** You need actual network devices to use this. For lab awareness only, you can create the credential and not use it.

---

## Part 6 — Hands-On: Git Integration and Project Setup (30 minutes)

### Step 16 — Create a Project Linked to Git

1. **Resources --> Projects --> Add**
2. Fill in:
   - **Name:** `Lab-Git-Project`
   - **Organization:** `Default`
   - **Source Control Type:** `Git`
   - **Source Control URL:** `https://github.com/your-org/your-repo.git` (use a real repo)
   - **Source Control Credential:** `GitHub-Personal-Access-Token`
   - **Update on launch:** Checked (syncs before each job that uses this project)
3. Click **Save**
4. Click the **Sync** icon to pull the repo
5. Verify **Status** shows a green check and **Revision** shows the latest commit

### Step 17 — Create a Project with Manual (Local) Source (Fallback)

If you do not have a Git repo, use the manual project from Lab 02:

1. Ensure `/var/lib/awx/projects/lab-playbooks` exists with playbooks
2. **Resources --> Projects --> Add**
3. **Name:** `Lab-Manual-Project`
4. **Source Control Type:** `Manual`
5. **Playbook Directory:** `lab-playbooks`
6. Save

---

## Part 7 — Hands-On: Notification Integrations (35 minutes)

### Step 18 — Configure Slack Notification

1. **Create a Slack Incoming Webhook** (if you have Slack):
   - In Slack: Channel → Integrations → Incoming Webhooks → Add to Slack
   - Choose a channel, copy the webhook URL (e.g., `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXX`)

2. In AAP: **Administration --> Notification templates**
3. Click **Add**
4. Fill in:
   - **Name:** `Slack-Job-Status`
   - **Organization:** `Default`
   - **Notification type:** `Slack`
   - **Slack token** or **Slack webhook URL:** Paste your webhook URL
   - **Channels:** `#automation` (or your channel)
   - **Messages:** Customize the success/failure messages or use defaults
5. Click **Save**

6. **Attach to a Job Template:**
   - **Resources --> Templates** → Open `Ping-Lab` (or any template)
   - **Notifications** section: Add **Slack-Job-Status** for **On success** and **On failure**
   - Save

7. **Launch** the job — you should receive a Slack message when the job completes.

### Step 19 — Configure Email Notification

1. **Administration --> Settings --> System** (or **Jobs** settings)
2. Ensure **SMTP** is configured:
   - **Administration --> Settings --> System**
   - **SMTP Server:** Your SMTP host (e.g., `smtp.gmail.com`)
   - **SMTP Port:** 587 (TLS) or 465 (SSL)
   - **SMTP Username / Password:** If required
   - **From email:** `aap@yourdomain.com`

3. **Administration --> Notification templates --> Add**
4. Fill in:
   - **Name:** `Email-Job-Status`
   - **Notification type:** `Email`
   - **Recipients:** `admin@yourdomain.com`
5. Save

6. Attach to a job template and test.

> **Lab note:** If you do not have SMTP access, skip email and use Slack or webhook only.

### Step 20 — Configure Webhook Notification (Outbound)

A webhook sends an HTTP POST to a URL when a job completes. Useful for custom integrations.

1. **Administration --> Notification templates --> Add**
2. Fill in:
   - **Name:** `Webhook-On-Failure`
   - **Notification type:** `Webhook`
   - **Target URL:** `https://your-server.com/webhook` (use a request bin for testing: https://webhook.site)
   - **HTTP Headers (optional):** `Content-Type: application/json`
3. Save

4. Attach to a job template for **On failure**
5. Launch a job that fails (e.g., wrong inventory) — the webhook URL will receive a POST with job metadata

### Step 21 — Test Notifications End-to-End

1. Create a job template that runs a simple playbook (e.g., `ping.yml`)
2. Add **Slack-Job-Status** (and optionally **Email-Job-Status**) to **On success** and **On failure**
3. **Launch** the job
4. Verify you receive a Slack (and/or email) notification
5. Modify the playbook to force a failure (e.g., typo in module name), launch again
6. Verify you receive a failure notification

---

## Part 8 — Real-Time Use Case Discussion (20 minutes)

### Case Study: E-Commerce Company — Multi-Cloud Credential Management

**Background:**

An e-commerce company runs 3,000+ servers across AWS, Azure, and on-premises data centers. They use AAP for configuration management, patching, and deployment automation.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Thousands of servers | Manual credential management does not scale |
| Multiple clouds | Different credential types (AWS keys, Azure service principals, SSH keys) |
| Compliance (PCI-DSS) | Credentials must be rotated, audited, and never exposed |
| Multiple teams | Dev, QA, Ops need different access levels |
| Third-party integrations | Payment gateways, CDNs require API keys |

**Solution Architecture:**

| Component | Implementation |
|-----------|-----------------|
| **Inventories** | Dynamic inventories per cloud (AWS EC2, Azure RM) + static for on-prem. Smart inventories for prod vs. nonprod. |
| **Credentials** | Separate Machine credentials per environment (prod-ssh, nonprod-ssh). AWS/Azure credentials scoped by IAM role with least privilege. |
| **Credential scoping** | RBAC: Only Ops team can use prod credentials. Dev team uses nonprod only. |
| **External secrets** | HashiCorp Vault for payment gateway API keys. AAP fetches at runtime via Vault credential. |
| **Rotation** | Vault handles rotation; AAP always gets current secret. SSH keys rotated quarterly; old keys revoked in AAP. |
| **Audit** | All credential usage logged. Job templates show which credential was used. Integration with SIEM for alerting. |

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Credential sprawl | 50+ shared passwords in spreadsheets | Centralized in AAP + Vault |
| Rotation time | 2 weeks (manual) | Automated via Vault |
| Compliance audit | Failed (no audit trail) | Passed (full traceability) |
| Incident response | 4 hours to rotate compromised key | 15 minutes (Vault + AAP) |

**Key takeaway:** Inventories define *what* to manage; credentials define *how* to access it. Together with RBAC and external secret managers, they form the security backbone of enterprise automation.

---

## Troubleshooting

### Inventory Sync Fails

| Symptom | Check |
|---------|-------|
| File source not found | Verify path: `/var/lib/awx/inventory_files/` and `awx` user can read |
| Script source fails | Run script manually: `python3 inventory/dynamic_inventory.py` — must output valid JSON |
| AWS source fails | Verify AWS credential; check IAM permissions for `ec2:DescribeInstances` |
| Permission denied | Ensure `awx` owns files: `sudo chown -R awx:awx /var/lib/awx/inventory_files` |

### Credential Test Fails

| Credential Type | Common Issues |
|-----------------|---------------|
| Machine | Wrong username; key format (must be PEM); key permissions on source |
| Source Control | Invalid token; wrong URL; repo is private and token lacks `repo` scope |
| AWS | Access key expired; wrong region; IAM policy too restrictive |
| Vault | Wrong vault password; encrypted file not used in playbook |

### Notifications Not Received

| Issue | Fix |
|-------|-----|
| Slack: No message | Verify webhook URL; check channel exists; ensure template is attached to job |
| Email: No message | Verify SMTP settings; check spam folder; test with a simple job |
| Webhook: No POST | Verify URL is reachable from AAP; check firewall; use webhook.site for testing |

### Smart Inventory Shows No Hosts

- Verify the **Source inventory** has hosts
- Check **Smart host filter** syntax (use query builder for complex filters)
- Ensure group names match exactly (case-sensitive)

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Static inventory | Manually maintained host list; groups and variables in UI |
| Dynamic inventory | File, script, or cloud source; syncs automatically |
| Smart inventory | Filtered view of existing inventory; no host duplication |
| Host/group variables | Variable precedence; host overrides group |
| Machine credential | SSH/WinRM access; key or password |
| Source Control credential | Git authentication; token or SSH key |
| Vault credential | Decrypt ansible-vault encrypted content |
| Cloud credentials | AWS, Azure, GCP for dynamic inventory and modules |
| Notification templates | Slack, Email, Webhook; attach to job templates |
| External secret management | When to use Vault, CyberArk, Azure Key Vault |

---

## Lab Completion Checklist

- [ ] Static inventory created with groups (webservers, appservers, databases, prod)
- [ ] Hosts added with host variables
- [ ] Group variables set
- [ ] Dynamic inventory created (file-based or script-based)
- [ ] Inventory source synced successfully
- [ ] Smart inventory created with filter
- [ ] Machine credential created and tested
- [ ] Source Control credential created (if using Git)
- [ ] Ansible Vault credential created
- [ ] Project linked to Git (or manual) and synced
- [ ] Slack (or Email) notification template created and attached to a job template
- [ ] Webhook notification tested (e.g., via webhook.site)
- [ ] Job launched and notification received
- [ ] Can explain the e-commerce multi-cloud credential use case

---

## Quick Reference — Inventory Source Types

| Source | Use Case |
|--------|----------|
| Manual | Add hosts/groups in UI |
| File, directory or script | Local file or script path |
| Sourced from a project | Script in Git repo |
| Amazon ec2 | AWS EC2 instances |
| Microsoft Azure | Azure VMs |
| Google Compute Engine | GCP instances |
| Red Hat Satellite 6 | Satellite hosts |
| Red Hat Virtualization | RHV VMs |
| VMware vCenter | vSphere VMs |
| OpenStack | OpenStack instances |

---

## Quick Reference — Credential Types

| Type | Key Fields |
|------|------------|
| Machine | Username, SSH Private Key or Password, Port |
| Source Control | URL, Username, Password/Token |
| Amazon Web Services | Access Key, Secret Key, Region |
| Microsoft Azure | Subscription ID, Tenant ID, Client ID, Secret |
| Ansible Vault | Vault Password |
| Network | Username, Password, Authorize (enable secret) |
| HashiCorp Vault | Vault address, Token or Role ID/Secret ID |

---

*End of Lab 04*
