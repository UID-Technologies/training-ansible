# Lab 04 — Manage Inventories, Credentials/Secrets & Integrations

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 25% / Hands-on: 75%)                                     |
| **Outcome**  | Create and manage static and dynamic inventories, configure host/group variables, build smart inventories, set up multiple credential types, integrate with Git and notification services, and understand external secret management |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)         |
| **Prerequisite** | Lab 02 completed — AAP 2.4 installed with web UI access. Lab 03 optional (multi-node setup). |

---

## AAP 2.4 UI Navigation Reference

Before starting, familiarize yourself with the **Automation Controller** UI structure in AAP 2.4:

| Menu | Location | Contains |
|------|----------|----------|
| **Resources** | Left navigation panel | Hosts, Inventories, Projects, Credentials, Templates |
| **Access** | Left navigation panel | Teams, Users, Organizations |
| **Administration** | Left navigation panel | Notifications, Instances, Instance Groups, Execution Environments, etc. |
| **Settings** | Left navigation panel | System configuration, Authentication, Jobs, License |

> **Note:** In AAP 2.4, "Templates" refers to Job Templates and Workflow Job Templates. "Notifications" are under **Administration**, not a separate top-level menu.

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
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Managed Nodes | 2–4 Linux VMs (RHEL, Ubuntu, AlmaLinux) — can reuse nodes from Lab 02 |
| Network | AAP can SSH to all managed nodes |
| Optional | GitHub/GitLab account for SCM integration; Slack workspace for notifications |
| Optional | AWS account for EC2 dynamic inventory demo |

---

## Verify AAP 2.4 Before Starting

**How to check your version:**

1. Log in to the Automation Controller.
2. Click the **?** (help) icon or your **user profile** in the top-right.
3. Select **About** — the version (e.g., 4.4.x for AAP 2.4) is displayed.

Alternatively: **Settings** (left panel) → **System** → **Tower** or **Controller** section shows version info.

> **Note:** AAP 2.4 uses Automation Controller 4.4.x. The UI has **Resources**, **Access**, **Administration**, and **Settings** in the left navigation. If you see a different structure (e.g., Platform-level "Access Management"), you may be on AAP 2.5 — adjust navigation paths accordingly.

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

**Purpose:** A static inventory is created and maintained manually in the UI. It is the simplest way to define your managed hosts.

**Detailed steps:**

1. **Log in** to the Automation Controller: Open a browser and go to `https://aap1.lab.local` (or your AAP server IP). Accept the self-signed certificate warning if prompted.

2. **Navigate to Inventories:** In the left navigation panel, click **Resources** to expand it (if collapsed), then click **Inventories**. You will see a list of existing inventories (e.g., `Default` from Lab 02).

3. **Add a new inventory:** Click the **Add** button (top right). From the dropdown, select **Add inventory**.  
   > **Why this matters:** In AAP 2.4, "Add" may show multiple options. Choose **Add inventory** for a standard (non-smart, non-constructed) inventory.

4. **Fill in the required fields:**
   - **Name:** `Lab-Static-Inventory` — This is the display name for your inventory.
   - **Organization:** Select `Default` from the dropdown. All resources in AAP 2.4 belong to an organization.
   - **Description:** `Static inventory for lab - webservers, appservers, databases` (optional but recommended for documentation).

5. **Save:** Click the blue **Save** button at the bottom of the form. The inventory details page will open. In the **classic UI**, you should see tabs such as **Details**, **Hosts**, **Groups**, **Sources**, **Access**. If you don't see **Groups**, use **Actions** → **Create a new Group** instead.

---

### Step 2 — Create Groups

**Purpose:** Groups organize hosts logically (e.g., by tier or environment). You can run playbooks against specific groups.

**Detailed steps:**

1. **Open the inventory:** From the Inventories list, click on `Lab-Static-Inventory` to open it. You must be on the **inventory details page** (not the list view).

2. **Go to groups:**
   - **Classic UI:** Click the **Groups** tab. Initially it will be empty.
   - **If you don't see a Groups tab:** Use **Actions** (top right) → **Create a new Group**. This opens the same group creation form.
   - **New UI Preview:** If you enabled "Preview of New User Interface" (Settings → System → Miscellaneous), the layout may differ. Use the classic UI for this lab, or look for an equivalent **Groups** or **Create Group** option in the new interface.

   > **Note:** Groups are only available for **standard** inventories. Smart Inventories and Constructed Inventories do not have a Groups tab.

3. **Add the first group:** Click the **Add** button (or use Actions → Create a new Group). In the **Create new group** form:
   - **Name:** `webservers` (required)
   - **Description:** `Web tier hosts` (optional)
   - Click **Save**.

4. **Repeat for the remaining groups:** Click **Add** again and create:
   - `appservers` — Application tier hosts
   - `databases` — Database tier hosts
   - `prod` — Production environment (parent group)

5. **Create the parent-child relationship:** Click on the `prod` group to edit it. In the **Parent Groups** or **Child Groups** section (depending on UI layout), add `webservers`, `appservers`, and `databases` as **child groups** of `prod`.  
   > **Why:** When you run a playbook against `prod`, it will target all hosts in webservers, appservers, and databases. Save the group.

---

### Step 3 — Add Hosts to Groups

**Purpose:** Hosts are the actual targets (servers) that Ansible will manage. Each host must be assigned to at least one group.

**Detailed steps:**

1. **Go to the Hosts tab:** Click the **Hosts** tab in `Lab-Static-Inventory`.

2. **Add the first host:** Click **Add**. In the form:
   - **Name:** `web1.lab.local` (or `node1.lab.local` if you have only 2 nodes). This is the Ansible host identifier.
   - **Groups:** Select `webservers` and `prod` from the groups list. You can select multiple groups.
   - Click **Save**.

3. **Add more hosts:** Repeat for:
   - `app1.lab.local` — Groups: `appservers`, `prod`, or use the same host as web1 in a minimal lab.
   - `db1.lab.local` — Groups: `databases`, `prod`.

4. **Set host variables (if needed):** Click on a host name (e.g., `web1.lab.local`). Go to the **Variables** tab. Click the edit (pencil) icon and add:

```yaml
---
ansible_host: 192.168.1.20
env: prod
```

   > **Explanation:** `ansible_host` tells Ansible which IP or hostname to connect to. If the host name is already resolvable (e.g., in `/etc/hosts`), you can omit it. Use your actual managed node IP.

5. **Minimal lab:** If you have only 2 nodes, add `node1` and `node2` and assign them to multiple groups. For example, `node1` can be in both `webservers` and `prod`.

---

### Step 4 — Set Group Variables

**Purpose:** Group variables apply to all hosts in that group. Useful for shared settings (e.g., `ansible_user`, `env`).

**Detailed steps:**

1. **Go to the Groups tab:** Click the **Groups** tab. Click on the `webservers` group.

2. **Open Variables:** Click the **Variables** tab. Click the edit (pencil) icon to modify.

3. **Add variables in YAML format:**

```yaml
---
env: prod
tier: web
```

   Click **Save**.

4. **Repeat for other groups:** Edit `appservers` and add `tier: app`; edit `databases` and add `tier: db`.

---

### Step 5 — Verify Variable Precedence

**Purpose:** Host variables override group variables. This is a key concept in Ansible.

**Detailed steps:**

1. **Edit a host:** Click the **Hosts** tab, then click on `web1.lab.local` (or your first host).

2. **Add a host-specific variable:** In the **Variables** tab, add:

```yaml
---
custom_http_port: 8080
```

3. **Save.** When a playbook uses `{{ custom_http_port }}` on this host, it will get `8080`. Other hosts in `webservers` without this variable would need a default or would fail if the variable is required.

---

## Part 3 — Hands-On: Dynamic Inventory (45 minutes)

### Option A — Project-Sourced Dynamic Inventory (No Cloud Required)

**AAP 2.4 behavior:** In AAP 2.4, dynamic inventory from a file or script must come from a **Project** (SCM or Manual). There is no "local file path" source. We will use a **Manual project** with an inventory file in `/var/lib/awx/projects/`.

#### Step 6 — Create an Inventory File in a Project Directory on the AAP Server

**Purpose:** The inventory file will be stored in a project directory. AAP will sync it via the "Sourcing from a Project" inventory source.

**Detailed steps:**

1. **SSH to the AAP server** and create a project directory with an inventory file:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-inventory-project
sudo tee /var/lib/awx/projects/lab-inventory-project/dynamic_hosts.yml > /dev/null << 'EOF'
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

2. **Replace IPs** with your actual managed node IPs. For a 2-node lab, use the same IP for multiple hosts or reduce the list.

3. **Set ownership** so the `awx` user can read the files:

```bash
sudo chown -R awx:awx /var/lib/awx/projects/lab-inventory-project
```

#### Step 7 — Create a Manual Project and Inventory with "Sourcing from a Project"

**Purpose:** AAP 2.4 requires a Project to source inventory from. We create a Manual project pointing to the directory, then an inventory that sources from that project.

**Detailed steps:**

1. **Create the Manual project (if not exists):**  
   - **Resources --> Projects --> Add**  
   - **Name:** `Lab-Inventory-Project`  
   - **Organization:** `Default`  
   - **Source Control Type:** `Manual`  
   - **Playbook Directory:** Select or type `lab-inventory-project` (the folder name under `/var/lib/awx/projects/`).  
   - Click **Save**.  
   - Click the **Sync** (refresh) icon to ensure the project has the latest content.

2. **Create the inventory:**  
   - **Resources --> Inventories --> Add --> Add inventory**  
   - **Name:** `Lab-Dynamic-Inventory`  
   - **Organization:** `Default`  
   - Click **Save**.

3. **Add the inventory source:**  
   - Open `Lab-Dynamic-Inventory` and click the **Sources** tab.  
   - Click **Add**. The **Create Source** window opens.

4. **Configure the source:**  
   - **Name:** `Project-Source`  
   - **Source:** Select **Sourcing from a Project** from the dropdown.  
   - **Project:** Click the search icon and select `Lab-Inventory-Project`.  
   - **Inventory File:** Type `dynamic_hosts.yml` (the file name in the project). You can also use the dropdown to select it if it appears after a project sync.  
   - **Update on Launch:** Check this — the inventory will refresh when a job using it is launched.  
   - Click **Save**.

5. **Sync the inventory:** Click **Sync all** (or the sync icon next to the source). Wait for the sync to complete (green check).

6. **Verify:** Click the **Hosts** and **Groups** tabs — they should be populated from the YAML file.

### Option B — Project-Sourced Dynamic Inventory (Script in Git)

If you use a Git project, you can place an inventory script in the repo that outputs JSON.

#### Step 8 — Create an Inventory Script (Alternative to Option A)

**Purpose:** A Python script that outputs Ansible JSON inventory format. AAP runs this script when syncing.

**Detailed steps:**

1. **On your laptop or dev machine**, create a Git repo with:

**inventory/dynamic_inventory.py**

```python
#!/usr/bin/env python3
"""Simple dynamic inventory for AAP lab - outputs Ansible JSON inventory."""
import json

hosts = {
    "webservers": ["web1.lab.local", "web2.lab.local"],
    "appservers": ["app1.lab.local"],
    "databases": ["db1.lab.local"],
}

inventory = {
    "_meta": {"hostvars": {}},
    "all": {"children": ["webservers", "appservers", "databases"]},
    "webservers": {"hosts": hosts["webservers"]},
    "appservers": {"hosts": hosts["appservers"]},
    "databases": {"hosts": hosts["databases"]},
}

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

2. **Make it executable:** `chmod +x inventory/dynamic_inventory.py`  
3. **Push to GitHub/GitLab.**

4. **In AAP 2.4:**
   - Create a **Project** (Resources --> Projects --> Add) pointing to the repo, with SCM credential if private.
   - Create inventory `Lab-Script-Inventory` (Resources --> Inventories --> Add).
   - Open it, go to **Sources** tab, click **Add**.
   - **Source:** Select **Sourcing from a Project**.
   - **Project:** Select your Git project.
   - **Inventory File:** Enter `inventory/dynamic_inventory.py` (the path to the script in the repo). The script must be executable in Git.
   - **Update on launch:** Checked.
   - Save and click **Sync all**.

### Option C — AWS EC2 Dynamic Inventory (If You Have AWS)

**Purpose:** Pull EC2 instances automatically from AWS as inventory hosts.

**Detailed steps:**

1. **Create an AWS credential** (Step 14 above) with Access Key and Secret Key.
2. **Create inventory:** **Resources --> Inventories --> Add** → Name: `Lab-AWS-Inventory`, Organization: `Default` → Save.
3. **Add source:** **Sources** tab → **Add**
4. **Source:** Select **Amazon ec2** (or **Amazon Web Services EC2** — exact label may vary in AAP 2.4).
5. **Credential:** Select your AWS credential.
6. **Regions:** `us-east-1` (or your region). Add multiple regions if needed.
7. **Instance filters (optional):** e.g., `tag:Environment=prod` to filter by tag.
8. **Update on launch:** Checked.
9. **Save** and click **Sync all**. AAP will call the AWS API and populate hosts from EC2.

---

## Part 4 — Hands-On: Smart Inventory (20 minutes)

> **AAP 2.4 note:** Smart Inventories are deprecated in favor of Constructed Inventories in future releases. For this lab we use Smart Inventory as it is still available and simpler for beginners. Constructed Inventories use `source_vars` and `limit` with Jinja2.

### Step 9 — Create a Smart Inventory

**Purpose:** A Smart Inventory is a filtered view of hosts from existing inventories. Hosts are filtered by a search (e.g., group name, variables). No hosts are duplicated — it is a dynamic query.

**Detailed steps:**

1. **Navigate:** **Resources --> Inventories**
2. **Add smart inventory:** Click **Add**, then select **Add smart inventory** from the dropdown.
3. **Fill in the form:**
   - **Name:** `Lab-Prod-Web-Only`
   - **Organization:** `Default`
   - **Smart host filter:** Click the search icon to open the filter builder. Change the search type to **Advanced** if needed. Enter a filter such as:
     - **Key:** `groups__name` (or use the query builder)
     - **Value:** `webservers`
   - Or type directly: `groups__name=webservers`
   - **Source inventory:** The Smart Inventory filters hosts from inventories in the same organization. Ensure `Lab-Static-Inventory` has hosts. The filter applies to all inventories in the org by default, or you may need to specify the source (check AAP 2.4 UI for "Source" or "Input inventories").
4. **Save** and open the smart inventory. The **Hosts** tab shows only hosts matching the filter (i.e., in the `webservers` group).

### Step 10 — Create Another Smart Inventory (Multi-Condition)

**Purpose:** Filter hosts that belong to multiple groups.

**Detailed steps:**

1. **Add --> Add smart inventory**
2. **Name:** `Lab-Prod-All-Tiers`
3. **Smart host filter:** Use one of:
   - `groups__name__in=webservers,appservers,databases` (hosts in any of these groups)
   - `groups__name=prod` (if using a `prod` parent group that contains all tiers)
4. **Save** and verify the host list in the **Hosts** tab.

---

## Part 5 — Hands-On: Credentials Management (50 minutes)

### Step 11 — Create Machine Credential (SSH Key)

**Purpose:** Machine credentials allow AAP to SSH into managed hosts. Use an SSH key for passwordless, secure access.

**Detailed steps:**

1. **Navigate:** **Resources --> Credentials**
2. **Add credential:** Click **Add**
3. **Select credential type:** In the **Credential Type** dropdown, select **Machine**
4. **Fill in the form:**
   - **Name:** `SSH-Key-WebServers` — A descriptive name for this credential
   - **Organization:** `Default`
   - **Username:** `ec2-user` (Amazon Linux/RHEL on EC2), `root`, or `ubuntu` — must match the user on your managed nodes
   - **SSH Private Key:** Paste the contents of your private key file (e.g., `~/.ssh/id_rsa`). The key must be in PEM format. Do not paste the public key.
   - **Privilege Escalation:** Expand this section and check **Enable** if your playbooks need to run commands as root via `sudo`. If enabled, you may need to set **Privilege Escalation Username** to `root` and **Privilege Escalation Password** if sudo requires a password (or ensure passwordless sudo on nodes).
5. **Save** — The credential is stored encrypted. It will appear in the Credentials list.

---

### Step 12 — Create Source Control Credential (GitHub/GitLab)

**Purpose:** Authenticates AAP to clone private Git repositories. Use a Personal Access Token (PAT), not your account password.

**Detailed steps:**

1. **Create a GitHub PAT (if you don't have one):**  
   - GitHub → Your profile → **Settings** → **Developer settings** → **Personal access tokens** → **Generate new token (classic)**  
   - Scopes: `repo`, `read:org` (for private org repos)  
   - Copy the token — you won't see it again.

2. **In AAP 2.4:** **Resources --> Credentials --> Add**
3. **Credential Type:** Select **Source Control**
4. **Fill in:**
   - **Name:** `GitHub-Personal-Access-Token`
   - **Organization:** `Default`
   - **Source Control Type:** `Git`
   - **Username:** Your GitHub username (optional for token-only auth)
   - **Password:** Paste your **Personal Access Token** (not your GitHub password)
   - **Project:** Leave blank for org-wide use
5. **Save**

---

### Step 13 — Create Ansible Vault Credential

**Purpose:** Decrypts files encrypted with `ansible-vault encrypt`. Required when playbooks or variables use vault encryption.

**Detailed steps:**

1. **Resources --> Credentials --> Add**
2. **Credential Type:** Select **Ansible Vault**
3. **Fill in:**
   - **Name:** `Vault-Lab-Secrets`
   - **Organization:** `Default`
   - **Vault Password:** The password you use with `ansible-vault encrypt/decrypt`
4. **Save**

> **Usage:** When a job template runs a playbook that includes vault-encrypted content, add this credential to the template. AAP will use it to decrypt at runtime.

---

### Step 14 — Create AWS Credential (Optional — If You Have AWS)

**Purpose:** Allows AAP to call AWS APIs (e.g., for EC2 dynamic inventory or cloud modules).

**Detailed steps:**

1. **Resources --> Credentials --> Add**
2. **Credential Type:** Select **Amazon Web Services**
3. **Fill in:**
   - **Name:** `AWS-Lab-Access`
   - **Organization:** `Default`
   - **Access Key:** AWS access key ID
   - **Secret Key:** AWS secret access key
   - **Region:** `us-east-1` (or your region)
4. **Save**

---

### Step 15 — Create Network Credential (For Network Devices)

**Purpose:** Used for network devices (Cisco, Juniper) that require enable/privileged mode.

**Detailed steps:**

1. **Resources --> Credentials --> Add**
2. **Credential Type:** Select **Network**
3. **Fill in:**
   - **Name:** `Network-Cisco-Lab`
   - **Organization:** `Default`
   - **Username:** `admin`
   - **Password:** Device login password
   - **Authorize:** Enable, and set **Authorize password** (enable secret) if the device uses it
4. **Save**

> **Note:** For lab awareness only unless you have network devices to test.

---

## Part 6 — Hands-On: Git Integration and Project Setup (30 minutes)

### Step 16 — Create a Project Linked to Git

**Purpose:** A Project links AAP to a Git repository. Playbooks are pulled from the repo and stored locally for job execution.

**Detailed steps:**

1. **Navigate:** **Resources --> Projects**
2. **Add project:** Click **Add**
3. **Fill in the form:**
   - **Name:** `Lab-Git-Project`
   - **Organization:** `Default`
   - **Source Control Type:** Select **Git** from the dropdown
   - **Source Control URL:** `https://github.com/your-username/your-repo.git` — use your actual repo URL
   - **Source Control Credential:** Select `GitHub-Personal-Access-Token` (required for private repos)
   - **Source Control Branch/Tag/Commit:** Leave blank to use the default branch (usually `main`)
   - **Update on launch:** Check this — AAP will sync the project before each job that uses it
4. **Save**
5. **Sync the project:** Click the **Sync** (circular arrows) icon next to the project. Wait for the sync to complete.
6. **Verify:** **Status** should show a green check. **Revision** shows the latest commit hash. **Playbooks** list should show your playbook files.

---

### Step 17 — Create a Project with Manual (Local) Source (Fallback)

**Purpose:** For labs without Git, a Manual project uses a directory on the AAP server.

**Detailed steps:**

1. **Ensure the directory exists** on the AAP server (from Lab 02):
   ```bash
   sudo ls /var/lib/awx/projects/lab-playbooks
   ```
   If it doesn't exist, create it and add a playbook (see Lab 02).

2. **In AAP 2.4:** **Resources --> Projects --> Add**
3. **Fill in:**
   - **Name:** `Lab-Manual-Project`
   - **Organization:** `Default`
   - **Source Control Type:** Select **Manual**
   - **Playbook Directory:** Select `lab-playbooks` from the dropdown (it lists directories under `/var/lib/awx/projects/`)
4. **Save**

---

## Part 7 — Hands-On: Notification Integrations (35 minutes)

> **AAP 2.4 navigation:** Notifications are under **Administration --> Notifications**. Each notification type (Email, Slack, Webhook) is a **Notification Template**. You create the template, then attach it to Job Templates, Projects, or Organizations.

### Step 18 — Configure Slack Notification

**Purpose:** Send job status (success/failure) to a Slack channel.

**Detailed steps:**

1. **Create a Slack Incoming Webhook:**
   - In Slack: Open a channel → click the channel name → **Integrations** → **Add an app** → search for **Incoming Webhooks** → **Add to Slack**
   - Choose the channel, click **Add Incoming Webhooks integration**
   - Copy the **Webhook URL** (e.g., `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXX`)

2. **In AAP 2.4:** Navigate to **Administration --> Notifications**
3. **Add notification template:** Click **Add**
4. **Fill in the form:**
   - **Name:** `Slack-Job-Status`
   - **Organization:** `Default`
   - **Notification type:** Select **Slack** from the dropdown
   - **Slack configuration:** AAP 2.4 may use either:
     - **Token:** A Slack Bot token (from api.slack.com/apps)
     - **Webhook URL:** Or a target URL field — paste your Incoming Webhook URL
   - **Channels:** `#automation` (or your channel name)
   - **Messages:** You can customize the **Started**, **Success**, and **Error** message bodies. Defaults work for most cases.
5. **Save**

6. **Attach to a Job Template:**
   - **Resources --> Templates** → Open your job template (e.g., `Ping-Lab`)
   - Scroll to the **Notifications** section
   - Click **Add** and select `Slack-Job-Status`
   - For **Started**, **Success**, and **Error** — enable the ones you want (e.g., Success and Error)
   - **Save**

7. **Launch** the job — you should receive a Slack message when the job completes.

---

### Step 19 — Configure Email Notification

**Purpose:** Send job status via email.

**Detailed steps:**

1. **Configure SMTP (if not done):**  
   - **Settings** (left panel) → **Jobs** or **System** tab  
   - Set **SMTP Host**, **SMTP Port** (587 or 465), **SMTP Username/Password** if required, **Sender email**

2. **Create the notification template:**  
   - **Administration --> Notifications --> Add**
   - **Name:** `Email-Job-Status`
   - **Organization:** `Default`
   - **Notification type:** Select **Email**
   - **Recipient list:** `admin@yourdomain.com` (comma-separated for multiple)
   - **Save**

3. **Attach to a job template** (same as Slack — add in the Notifications section).

> **Lab note:** If you don't have SMTP access, skip email and use Slack or webhook only.

---

### Step 20 — Configure Webhook Notification (Outbound)

**Purpose:** Send an HTTP POST to a URL when a job completes. Useful for CI/CD or custom integrations.

**Detailed steps:**

1. **Get a test URL:** Go to https://webhook.site and copy your unique URL.
2. **Administration --> Notifications --> Add**
3. **Fill in:**
   - **Name:** `Webhook-On-Failure`
   - **Organization:** `Default`
   - **Notification type:** Select **Webhook**
   - **Target URL:** Paste your webhook.site URL (or `https://your-server.com/webhook`)
   - **HTTP Method:** `POST`
   - **HTTP Headers (optional):** `{"Content-Type": "application/json"}`
4. **Save**
5. **Attach to a job template** for **On failure** (Error). Launch a job that fails — the URL will receive a POST with job metadata in JSON.

---

### Step 21 — Test Notifications End-to-End

1. Ensure a job template has **Slack-Job-Status** (or **Email-Job-Status**) attached for **On success** and **On failure**.
2. **Launch** the job — verify you receive a success notification.
3. Modify the playbook to force a failure (e.g., add a task with a typo: `ansible.builtin.pingg`), launch again.
4. Verify you receive a failure notification.

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
| Project source — file not found | Verify the project is synced; the inventory file path is relative to project root (e.g., `dynamic_hosts.yml` or `inventory/script.py`). Ensure `awx` owns `/var/lib/awx/projects/<project>/` |
| Script source fails | Run script manually: `python3 inventory/dynamic_inventory.py` — must output valid JSON. Script must be executable (`chmod +x`) in Git |
| AWS source fails | Verify AWS credential; check IAM permissions for `ec2:DescribeInstances` |
| Permission denied | Ensure `awx` owns project files: `sudo chown -R awx:awx /var/lib/awx/projects/lab-inventory-project` |

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

### Groups Tab Not Visible

- **Use Actions instead:** On the inventory details page, click **Actions** (top right) → **Create a new Group**. This opens the same form as the Groups tab.
- **Check inventory type:** Groups are only for **standard** inventories. Smart Inventories and Constructed Inventories do not have a Groups tab.
- **New UI Preview:** If "Preview of New User Interface" is enabled, the layout may differ. Use the classic UI (disable the preview in Settings → System → Miscellaneous) or look for equivalent group options in the new interface.

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

## Quick Reference — Inventory Source Types (AAP 2.4)

| Source | Use Case |
|--------|----------|
| Manual | Add hosts/groups in UI (no source) |
| Sourcing from a Project | File or script in a Project (Git or Manual) |
| Amazon ec2 | AWS EC2 instances |
| Microsoft Azure Resource Manager | Azure VMs |
| Google Compute Engine | GCP instances |
| Red Hat Satellite 6 | Satellite hosts |
| Red Hat Virtualization | RHV VMs |
| VMware vCenter | vSphere VMs |
| OpenStack | OpenStack instances |
| Red Hat Ansible Automation Platform | AAP instances |
| Red Hat Insights | Insights inventory |

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
