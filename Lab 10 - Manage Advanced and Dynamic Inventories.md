# Lab 10 — Manage Advanced and Dynamic Inventories

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate to Advanced (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 25% / Hands-on: 75%)                                      |
| **Outcome**  | Configure dynamic inventories (project-sourced, AWS EC2, Azure), create smart inventories with filters, manage combined sources, understand update on launch vs. scheduled sync, organize large-scale inventories |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Labs 02, 04 completed — AAP 2.4 installed, inventory and credential basics. Optional: AWS or Azure account for cloud exercises. |

---

## AAP 2.4 Inventory Source Types

| Source | Use Case |
|--------|----------|
| **Sourcing from a Project** | File or script in a Project (Git or Manual) |
| **Amazon ec2** | AWS EC2 instances |
| **Microsoft Azure Resource Manager** | Azure VMs |
| **Google Compute Engine** | GCP instances |
| **VMware vCenter** | vSphere VMs |
| **Red Hat Satellite 6** | Satellite hosts |
| **Red Hat Virtualization** | RHV VMs |
| **OpenStack** | OpenStack instances |
| **Red Hat Insights** | Insights inventory |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                        │
│                         https://aap1.lab.local                                       │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│  INVENTORY SOURCES      │  │  SMART INVENTORIES       │  │  COMBINED INVENTORY     │
│  - Project (YAML/script)│  │  - Prod-Web-Only         │  │  - Multi-source         │
│  - AWS EC2 (optional)   │  │  - Prod-All-Tiers        │  │  - Static + Dynamic     │
│  - Azure RM (optional)  │  │  - Region filters        │  │  - Variable precedence  │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Lab-Static-Inventory | From Lab 04 with groups (webservers, appservers, databases) and hosts |
| Project | Lab-Inventory-Project or Lab-Project (from Lab 04/05) |
| Optional | AWS account (IAM user with EC2 read access) for AWS exercise |
| Optional | Azure subscription (service principal) for Azure exercise |

---

## Part 1 — Theory: Dynamic Inventory Concepts (60 minutes)

### Section 9.1 — Dynamic Inventory Concepts

#### What Are Dynamic Inventories?

A **dynamic inventory** is populated automatically from an external source at sync time. Instead of manually maintaining a list of hosts, AAP queries an API, script, or file and builds the inventory.

| Aspect | Static | Dynamic |
|--------|--------|---------|
| **Source** | Manual entry in UI | Cloud API, CMDB, script, file |
| **Update** | Manual add/remove | Sync (manual, scheduled, or on launch) |
| **Scale** | Small (tens) | Large (thousands) |
| **Change frequency** | Low | High (auto-scaling, provisioning) |

#### When to Use Dynamic vs. Static Inventories

| Scenario | Recommendation |
|----------|----------------|
| Lab, POC, < 20 hosts | Static |
| Cloud (AWS, Azure, GCP) | Dynamic |
| VMware vCenter | Dynamic |
| Red Hat Satellite | Dynamic |
| CMDB (ServiceNow) | Dynamic (script/API) |
| Air-gapped, fixed fleet | Static or project-sourced file |
| Hybrid (cloud + on-prem) | Combined: multiple sources in one inventory |

#### How Dynamic Inventories Work

1. **Inventory source** — Configured in AAP (source type, credential, options)
2. **Sync** — AAP runs the inventory plugin or script
3. **Output** — Plugin/script returns JSON (Ansible inventory format)
4. **Import** — AAP parses and stores hosts/groups/variables
5. **Jobs** — Job templates use the inventory as usual

#### Inventory Plugins Overview

| Plugin | Collection | Purpose |
|--------|-------------|---------|
| `amazon.aws.aws_ec2` | amazon.aws | AWS EC2 instances |
| `azure.azcollection.azure_rm` | azure.azcollection | Azure VMs |
| `google.cloud.gcp_compute` | google.cloud | GCP instances |
| `community.vmware.vmware_vm_inventory` | community.vmware | VMware vCenter |
| `theforeman.foreman.foreman` | theforeman.foreman | Red Hat Satellite |
| `constructed` | ansible.builtin | Build inventory from other sources (AAP 2.4+) |

#### Cache Management Basics

- **Fact cache** — Stores `setup` module output; reduces redundant fact gathering
- **Inventory cache** — Some plugins support caching (e.g., `cache: yes` in plugin config)
- **Update on launch** — Syncs inventory before job; ensures fresh data
- **Scheduled sync** — Run sync on a schedule (e.g., hourly) for predictable updates

---

### Section 9.2 — Cloud-Based Dynamic Inventories

#### AWS EC2 Dynamic Inventory

| Component | Details |
|-----------|---------|
| **Credential** | Amazon Web Services — Access Key, Secret Key, Region |
| **IAM permissions** | `ec2:DescribeInstances`, `ec2:DescribeTags`, etc. (e.g., `AmazonEC2ReadOnlyAccess`) |
| **Source** | Amazon ec2 |

**Filtering and grouping:**

- **Source variables** (YAML) control grouping and filtering:
  - `keyed_groups` — Create groups by region, tag, instance type
  - `hostnames` — Use tag or attribute as hostname
  - `filters` — Filter instances (e.g., `instance-state-name: running`)

**Example source variables:**

```yaml
keyed_groups:
  - key: placement.region
    prefix: aws_region
  - key: tags.Environment
    prefix: env
filters:
  instance-state-name: running
```

**Common use cases:** Patch management, config drift, security compliance, multi-region deployment

#### Microsoft Azure Dynamic Inventory

| Component | Details |
|-----------|---------|
| **Credential** | Microsoft Azure Resource Manager — Subscription ID, Tenant ID, Client ID, Client Secret |
| **Service principal** | Create in Azure AD → App registration → Certificates & secrets |
| **Source** | Microsoft Azure Resource Manager |

**Resource group and tag-based organization:**

- Groups by `resource_group`, `location`, `tags`
- Filter by `tags.Environment=prod`, `power_state=running`

**Common use cases:** Azure VM management, hybrid cloud, tag-based targeting

#### Google Cloud Platform Inventory Basics

| Component | Details |
|-----------|---------|
| **Credential** | Google Compute Engine — Service account JSON |
| **Source** | Google Compute Engine |
| **Grouping** | By zone, network, labels |

#### VMware vCenter Inventory Overview

| Component | Details |
|-----------|---------|
| **Credential** | VMware vCenter — Host, Username, Password |
| **Source** | VMware vCenter |
| **Grouping** | By datacenter, cluster, folder, custom attributes |

---

### Section 9.3 — Other Inventory Sources

#### Red Hat Satellite Integration

- **Source:** Red Hat Satellite 6
- **Credential:** Red Hat Satellite 6 (URL, username, password)
- **Content:** Hosts from Satellite; groups by host group, location, organization
- **Use case:** Patch management, compliance, lifecycle management

#### ServiceNow CMDB Integration

- **Source:** Custom script or constructed inventory
- **Approach:** Script calls ServiceNow API, returns Ansible JSON inventory
- **Use case:** Single source of truth from CMDB; approval workflows

#### Understanding Inventory Source Updates

| Option | When Sync Runs |
|--------|----------------|
| **Manual** | User clicks Sync |
| **Update on launch** | Before each job that uses the inventory |
| **Scheduled** | On defined schedule (e.g., every hour) |

**Best practice:** Use **Update on launch** for frequently changing cloud inventories; use **Scheduled** for predictable sync and reduced API calls.

---

### Section 9.4 — Smart Inventories

#### What Are Smart Inventories?

A **smart inventory** is a **filtered view** of hosts from existing inventories in the same organization. No hosts are duplicated — it is a dynamic query.

| Aspect | Standard Inventory | Smart Inventory |
|--------|-------------------|-----------------|
| **Hosts** | From sources | Filtered from other inventories |
| **Groups** | Configurable | Not configurable (filter-based) |
| **Use case** | Primary inventory | Cross-inventory views (e.g., "all prod") |

> **Note:** Smart Inventories are deprecated in favor of **Constructed Inventories** in future AAP releases. Both are available in AAP 2.4.

#### Creating Smart Inventories with Filters

1. **Resources → Inventories → Add → Add smart inventory**
2. **Name**, **Organization**
3. **Smart host filter** — Django-style query or query builder
4. **Source inventories** — Which inventories to filter from (org-wide or specific)

#### Host Filter Syntax

| Filter | Meaning |
|--------|---------|
| `groups__name=webservers` | Hosts in group `webservers` |
| `groups__name__contains=prod` | Hosts in groups whose name contains `prod` |
| `groups__name__in=webservers,appservers` | Hosts in any of these groups |
| `variables__region=us-east-1` | Hosts with variable `region=us-east-1` |
| `variables__env__isnull=false` | Hosts with `env` variable set |
| `name__startswith=web` | Host names starting with `web` |
| `name__icontains=prod` | Host names containing `prod` (case-insensitive) |

**Combining conditions:** Use `and` / `or` (e.g., `groups__name=webservers and variables__env=prod`)

#### Use Cases for Smart Inventories

| Use Case | Filter Example |
|----------|----------------|
| All production hosts | `groups__name__contains=prod` |
| Web tier only | `groups__name=webservers` |
| Hosts in a region | `variables__region=us-east-1` |
| Hosts with a tag | `variables__env=prod` |
| Cross-inventory view | Filter from multiple inventories |

#### Performance Considerations

- Smart inventory is a **query** — no sync; evaluated at job launch
- Large host counts (10,000+) may slow filter evaluation
- Prefer specific filters over broad ones
- Consider **Constructed Inventory** for complex logic (Jinja2, `limit`)

---

### Section 9.5 — Managing Complex Inventories

#### Combining Multiple Inventory Sources

- One **inventory** can have **multiple sources**
- Each source syncs independently; hosts/groups are merged
- **Same host** in multiple sources — AAP merges variables (precedence applies)
- Use **Source variables** to set `group_by` or `keyed_groups` for consistent naming

#### Host and Group Variables in Dynamic Inventories

| Source | How Variables Are Set |
|--------|------------------------|
| **Cloud plugins** | From instance metadata (tags, attributes) |
| **Script** | In `_meta.hostvars` or group vars in JSON |
| **YAML file** | In `host_vars` / `group_vars` or inline |

**Example (AWS):** Tag `Environment=prod` → variable `Environment` or group `env_prod`

#### Variable Precedence with Multiple Sources

When the same host appears in multiple sources or has variables at multiple levels:

1. **Host vars** (most specific) override **group vars**
2. **Child group** vars override **parent group** vars
3. **Extra vars** (job template) override inventory vars
4. **Last sync wins** for same-level vars from different sources (undefined order)

#### Organizing Large-Scale Inventories

| Strategy | Description |
|----------|-------------|
| **Keyed groups** | Auto-create groups by region, tag, type |
| **Smart inventories** | Filter views (prod only, region only) |
| **Limit** | Use limit in job template to target subset |
| **Job slicing** | Split large inventory across parallel jobs |
| **Multiple inventories** | Separate by environment (dev, prod) or cloud |

---

## Part 2 — Hands-On: Project-Sourced Dynamic Inventory (30 minutes)

### Step 1 — Create YAML Inventory File in Project

**Purpose:** Use a YAML file as dynamic inventory source (no cloud required).

**Detailed steps:**

1. **SSH to AAP server** and create the inventory file:

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
          region: us-east-1
        web2.lab.local:
          ansible_host: 192.168.1.21
          env: prod
          region: us-east-1
    appservers:
      hosts:
        app1.lab.local:
          ansible_host: 192.168.1.22
          env: prod
          region: us-east-1
    databases:
      hosts:
        db1.lab.local:
          ansible_host: 192.168.1.23
          env: prod
          region: us-east-1
    prod:
      children:
        webservers: {}
        appservers: {}
        databases: {}
EOF
sudo chown -R awx:awx /var/lib/awx/projects/lab-inventory-project
```

2. **Replace IPs** with your actual managed node IPs. For a minimal lab, use the same host in multiple groups.

### Step 2 — Create Inventory with Project Source

**Purpose:** Configure AAP to use the YAML file as an inventory source.

**Detailed steps:**

1. **Resources → Projects** — Ensure `Lab-Inventory-Project` exists with Playbook Directory `lab-inventory-project`. Sync if needed.
2. **Resources → Inventories → Add → Add inventory**
   - **Name:** `Lab-Dynamic-Inventory`
   - **Organization:** `Default`
   - **Save**
3. **Sources tab → Add**
   - **Name:** `Project-Source`
   - **Source:** `Sourcing from a Project`
   - **Project:** `Lab-Inventory-Project`
   - **Inventory File:** `dynamic_hosts.yml`
   - **Update on Launch:** Check this
   - **Save**
4. **Sync** — Click **Sync all**. Verify hosts and groups appear.

### Step 3 — Configure Update on Launch and Schedule

**Purpose:** Understand sync triggers.

**Detailed steps:**

1. **Edit** the inventory source `Project-Source`
2. **Update on Launch** — Already checked; inventory syncs before each job
3. **Schedules tab** (on the source or inventory) — Add a schedule to sync every hour (optional):
   - **Resources → Inventories** → `Lab-Dynamic-Inventory` → **Sources** → click source → **Schedules** tab
   - **Add** — Frequency: Hourly, Run at: :00
4. **Save**

---

## Part 3 — Hands-On: AWS EC2 Dynamic Inventory (Optional, 40 minutes)

> **Prerequisite:** AWS account with IAM user; Access Key and Secret Key; `AmazonEC2ReadOnlyAccess` policy (or equivalent).

### Step 4 — Create AWS Credential

**Purpose:** Allow AAP to query EC2 API.

**Detailed steps:**

1. **Resources → Credentials → Add**
2. **Credential type:** Amazon Web Services
3. **Name:** `AWS-EC2-Inventory`
4. **Access Key:** Your IAM access key
5. **Secret Key:** Your IAM secret key
6. **Region:** e.g., `us-east-1`
7. **Save**

### Step 5 — Create AWS EC2 Inventory Source

**Purpose:** Populate inventory from EC2 instances.

**Detailed steps:**

1. **Ensure `amazon.aws` collection** is installed (e.g., via project `collections/requirements.yml` or default Execution Environment). The `amazon.aws.aws_ec2` plugin is required.
2. **Resources → Inventories → Add → Add inventory**
   - **Name:** `Lab-AWS-Inventory`
   - **Organization:** `Default`
   - **Save**
2. **Sources tab → Add**
   - **Name:** `EC2-Source`
   - **Source:** `Amazon ec2`
   - **Credential:** `AWS-EC2-Inventory`
   - **Execution Environment:** Default (or one with `amazon.aws` collection)
   - **Source variables** (optional, for grouping):

```yaml
keyed_groups:
  - key: placement.region
    prefix: aws_region
  - key: tags.Environment
    prefix: env
filters:
  instance-state-name: running
```

   - **Update on Launch:** Check
   - **Save**
3. **Sync** — Wait for sync to complete. Verify EC2 instances appear as hosts.

---

## Part 4 — Hands-On: Azure Dynamic Inventory (Optional, 40 minutes)

> **Prerequisite:** Azure subscription; Service Principal (App registration) with Reader role on subscription or resource group.

### Step 6 — Create Azure Credential

**Purpose:** Allow AAP to query Azure Resource Manager.

**Detailed steps:**

1. **Resources → Credentials → Add**
2. **Credential type:** Microsoft Azure Resource Manager
3. **Name:** `Azure-Inventory`
4. **Subscription ID:** Your Azure subscription ID
5. **Tenant ID:** Your Azure AD tenant ID
6. **Client ID:** Service principal application (client) ID
7. **Client Secret:** Service principal secret
8. **Save**

### Step 7 — Create Azure RM Inventory Source

**Purpose:** Populate inventory from Azure VMs.

**Detailed steps:**

1. **Ensure `azure.azcollection` collection** is installed. The `azure.azcollection.azure_rm` plugin is required.
2. **Resources → Inventories → Add → Add inventory**
   - **Name:** `Lab-Azure-Inventory`
   - **Organization:** `Default`
   - **Save**
2. **Sources tab → Add**
   - **Name:** `Azure-Source`
   - **Source:** `Microsoft Azure Resource Manager`
   - **Credential:** `Azure-Inventory`
   - **Execution Environment:** Default (or one with `azure.azcollection`)
   - **Source variables** (optional):

```yaml
keyed_groups:
  - key: resource_group
    prefix: rg
  - key: location
    prefix: azure_region
```

   - **Update on Launch:** Check
   - **Save**
3. **Sync** — Wait for sync. Verify Azure VMs appear.

---

## Part 5 — Hands-On: Smart Inventories (35 minutes)

### Step 8 — Create Smart Inventory (Single Condition)

**Purpose:** Filter hosts by group.

**Detailed steps:**

1. **Resources → Inventories → Add → Add smart inventory**
2. **Name:** `Lab-Prod-Web-Only`
3. **Organization:** `Default`
4. **Smart host filter:** `groups__name=webservers`
5. **Source inventories:** Select `Lab-Static-Inventory` (or `Lab-Dynamic-Inventory`) — or leave default to use all org inventories
6. **Save**
7. **Verify:** Hosts tab shows only hosts in `webservers` group.

### Step 9 — Create Smart Inventory (Multi-Condition)

**Purpose:** Filter hosts by multiple criteria.

**Detailed steps:**

1. **Add → Add smart inventory**
2. **Name:** `Lab-Prod-All-Tiers`
3. **Smart host filter:** `groups__name__in=webservers,appservers,databases`
   - Or: `groups__name=prod` (if you have a `prod` parent group)
4. **Save**
5. **Verify:** Hosts tab shows all prod-tier hosts.

### Step 10 — Create Smart Inventory (Variable Filter)

**Purpose:** Filter by host variable.

**Detailed steps:**

1. **Add → Add smart inventory**
2. **Name:** `Lab-Region-US-East`
3. **Smart host filter:** `variables__region=us-east-1`
   - (Requires hosts to have `region` variable — add in `dynamic_hosts.yml` or host vars)
4. **Save**
5. **Verify:** Only hosts with `region: us-east-1` appear.

---

## Part 6 — Hands-On: Combined Inventory Sources (30 minutes)

### Step 11 — Add Multiple Sources to One Inventory

**Purpose:** Combine project-sourced and (optionally) cloud sources.

**Detailed steps:**

1. **Edit** `Lab-Dynamic-Inventory` (or create `Lab-Combined-Inventory`)
2. **Sources tab** — You already have `Project-Source`
3. **Add** a second source (if you have AWS/Azure from Steps 5/7):
   - **Name:** `EC2-Source` (or `Azure-Source`)
   - **Source:** Amazon ec2 (or Microsoft Azure Resource Manager)
   - **Credential:** AWS-EC2-Inventory (or Azure-Inventory)
   - **Save**
4. **Sync all** — Both sources sync; hosts from both appear in the inventory
5. **Verify:** Hosts tab shows hosts from all sources. Groups may differ per source; use consistent `keyed_groups` for unified grouping.

### Step 12 — Variable Precedence with Multiple Sources

**Purpose:** Understand how variables merge when the same host is in multiple sources.

**Detailed steps:**

1. If the same host (e.g., by `ansible_host` or name) appears in two sources with different variables, AAP merges them. **Host-level** vars typically override **group-level**.
2. **Test:** Add a host variable in one source (e.g., in `dynamic_hosts.yml`) and ensure the playbook receives it. Add a conflicting variable in another source — the last sync or more specific level wins (behavior can vary; document your setup).

---

## Part 7 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Multinational Corporation — 10,000+ Servers Across AWS, Azure, VMware

**Background:**

A multinational corporation runs 10,000+ servers across AWS, Azure, and on-premises VMware. Manual inventory management is impossible. They need automated, up-to-date inventories for patching, compliance, and deployment.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Scale | 10,000+ hosts; manual lists not feasible |
| Multi-cloud | AWS, Azure, VMware — different APIs and structures |
| Change rate | Auto-scaling; new VMs daily |
| Consistency | Same grouping logic across clouds |
| Performance | Sync and job execution must scale |

**Solution: Dynamic Inventories with Scheduled Updates**

| Component | Implementation |
|-----------|----------------|
| **AWS inventory** | Amazon ec2 source; keyed_groups by region, Environment tag; sync hourly |
| **Azure inventory** | Azure RM source; keyed_groups by resource_group, location; sync hourly |
| **VMware inventory** | VMware vCenter source; keyed_groups by cluster, folder; sync hourly |
| **Combined inventory** | One inventory with 3 sources; unified groups (e.g., `env_prod`, `region_us_east`) |
| **Smart inventories** | `Prod-All-Clouds` = filter `groups__name__contains=prod`; `Patch-Targets` = filter by `variables__patch_window` |
| **Update on launch** | Disabled for large inventories; use scheduled sync to avoid long job startup |
| **Job slicing** | For 10K hosts, use job slicing (e.g., 10 slices) for parallel execution |

**Workflow:**

```
Hourly schedule:
  AWS source sync  →  Merge
  Azure source sync →  Merge   →  Combined inventory (10,000+ hosts)
  VMware source sync → Merge

Job launch:
  Smart inventory "Prod-Web" (filter) → 2,000 hosts
  Job slicing (4) → 4 parallel jobs, 500 hosts each
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Inventory update | Manual, weekly | Automated, hourly |
| Time to add new VM | 1–2 days | Automatic (next sync) |
| Patch coverage | 60% (incomplete lists) | 98% |
| Compliance audit | Failed (stale data) | Passed (current inventory) |

**Key takeaway:** Dynamic inventories with **keyed_groups** and **smart inventories** enable scalable, multi-cloud automation. Schedule syncs for predictability; use job slicing for large runs.

---

## Troubleshooting

### Dynamic Inventory Sync Fails

| Symptom | Check |
|---------|-------|
| Project source — file not found | Verify project synced; path relative to project root; `awx` owns files |
| AWS — Access Denied | IAM policy (ec2:DescribeInstances, etc.); correct region |
| Azure — Auth failed | Service principal; Client ID/Secret; Reader role on subscription |
| Script source — non-zero exit | Run script manually; check JSON output format |

### Smart Inventory Shows No Hosts

| Symptom | Check |
|---------|-------|
| Empty result | Source inventories have hosts; filter syntax correct |
| Wrong hosts | Filter is case-sensitive; `groups__name` must match exactly |
| Syntax error | Use query builder for complex filters; check `__` (double underscore) |

### Combined Inventory — Duplicate or Missing Hosts

| Symptom | Check |
|---------|-------|
| Duplicates | Same host in multiple sources; AAP may merge or create duplicates by name |
| Missing | Sync each source; check source-specific filters |
| Variable conflicts | Document precedence; use unique variable names per source |

### Update on Launch — Job Slow to Start

| Symptom | Check |
|---------|-------|
| Long startup | Large inventory sync before job; consider scheduled sync, disable update on launch |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Dynamic vs. static | When to use each; sync triggers |
| Project-sourced | YAML/script in project; no cloud required |
| AWS EC2 | Credential, keyed_groups, filters |
| Azure RM | Service principal, keyed_groups |
| Smart inventory | Filter syntax; groups__name, variables__ |
| Combined sources | Multiple sources in one inventory |
| Update on launch | Sync before job; vs. scheduled |
| Variable precedence | Host > group; multiple sources |

---

## Lab Completion Checklist

- [ ] Project-sourced dynamic inventory created and synced
- [ ] Update on launch and schedule configured
- [ ] (Optional) AWS EC2 inventory source created and synced
- [ ] (Optional) Azure RM inventory source created and synced
- [ ] Smart inventory created (single condition)
- [ ] Smart inventory created (multi-condition)
- [ ] Smart inventory created (variable filter)
- [ ] Combined inventory with multiple sources (if cloud available)
- [ ] Can explain the multinational corporation use case

---

## Quick Reference — Smart Inventory Filter Syntax

| Pattern | Example |
|---------|---------|
| Exact match | `groups__name=webservers` |
| Contains | `groups__name__contains=prod` |
| In list | `groups__name__in=web,app,db` |
| Variable | `variables__region=us-east-1` |
| Variable set | `variables__env__isnull=false` |
| Host name | `name__startswith=web`, `name__icontains=prod` |
| Combine | `groups__name=webservers and variables__env=prod` |

---

## Quick Reference — AWS keyed_groups Example

```yaml
keyed_groups:
  - key: placement.region
    prefix: aws_region
  - key: tags.Environment
    prefix: env
  - key: tags.Application
    prefix: app
filters:
  instance-state-name: running
```

---

*End of Lab 10*
