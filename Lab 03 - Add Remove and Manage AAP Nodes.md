# Lab 03 — Add, Remove, and Manage Ansible Automation Platform Nodes

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 3 hours (Theory: 30% / Hands-on: 70%)                                     |
| **Outcome**  | Understand AAP node types and architecture, add execution nodes to a running AAP installation, create and manage instance groups, test job distribution across nodes, and remove/decommission nodes |
| **Platform** | Red Hat Ansible Automation Platform 2.4 / 2.5                              |
| **Prerequisite** | Lab 02 completed — a working standalone AAP installation with web UI access |

---

## Lab Topology

```
                         ┌───────────────────────────────────┐
                         │         YOUR LAPTOP / BROWSER     │
                         │         https://aap1.lab.local    │
                         └──────────────┬────────────────────┘
                                        │ HTTPS (443)
                                        ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          CONTROL PLANE                                     │
│                                                                            │
│   ┌─────────────────────────────────┐                                      │
│   │   aap1.lab.local                │                                      │
│   │   Role: Control Node            │                                      │
│   │   - Automation Controller       │                                      │
│   │   - Web UI / API / Scheduler    │                                      │
│   │   - RBAC / Credential Vault     │                                      │
│   │   - PostgreSQL (local)          │                                      │
│   └────────────┬────────────────────┘                                      │
│                │ Receptor mesh (TCP 27199)                                  │
└────────────────┼───────────────────────────────────────────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
        ▼                  ▼
┌───────────────┐   ┌───────────────┐
│ exec1.lab.local│   │ exec2.lab.local│
│               │   │               │
│ Role:         │   │ Role:         │
│ Execution Node│   │ Execution Node│
│               │   │               │
│ Instance Group:│   │ Instance Group:│
│ "dev-workers" │   │ "prod-workers"│
└───────┬───────┘   └───────┬───────┘
        │ SSH (22)          │ SSH (22)
        ▼                   ▼
  ┌───────────┐       ┌───────────┐
  │ Dev Hosts │       │ Prod Hosts│
  └───────────┘       └───────────┘
```

> **Note:** Replace hostnames and IPs with your actual lab values throughout this document.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Control Node | AAP installed and running from Lab 02 (`aap1.lab.local`) |
| Execution Node VMs | 2 new VMs running **RHEL 8 or RHEL 9** — these will become execution nodes |
| Network | All VMs can reach each other; Receptor port TCP 27199 open between control and execution nodes |
| Access | `root` or `sudo` privileges on all VMs |
| Subscription | Red Hat subscription attached to the new VMs (same as Lab 02) |

**VM Sizing for Execution Nodes:**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| vCPU     | 2       | 4           |
| RAM      | 4 GB    | 8 GB        |
| Disk     | 40 GB   | 60 GB       |
| OS       | RHEL 8 or RHEL 9 | |

---

## Part 1 — Theory: Understanding Node Architecture (30 minutes)

### Section 1 — What Are Nodes in AAP?

In a standalone AAP installation (Lab 02), everything runs on a single server — the Controller, the database, and job execution all share one machine. This works for labs and small environments, but in production you need to **scale out** by adding dedicated nodes.

A **node** in AAP is any server that participates in the automation platform. Nodes are connected via a **Receptor mesh** — a lightweight, encrypted overlay network that routes automation traffic between them.

### Section 2 — Node Types Explained

AAP defines four types of nodes. Understanding when to use each one is critical for designing a scalable platform.

| Node Type       | What It Does                                       | Runs Controller? | Runs Jobs? |
|-----------------|----------------------------------------------------|-------------------|------------|
| Control Node    | Hosts the Controller (UI, API, scheduler, RBAC). Makes decisions about what runs where. | Yes | No (by default) |
| Execution Node  | Dedicated worker that runs playbooks and automation jobs. | No | Yes |
| Hybrid Node     | Combines both roles — runs Controller services AND executes jobs. | Yes | Yes |
| Hop Node        | Routes traffic between isolated networks. Does not run Controller or jobs. | No | No |

#### Control Node (Control Plane)

The control node is **where decisions are made**:

- Hosts the web UI and REST API
- Runs the scheduler that decides when and where jobs execute
- Enforces RBAC and credential management
- Stores job results and audit logs
- Does **not** execute automation jobs in a multi-node setup (offloads to execution nodes)

> **Analogy:** The control node is the "brain" — it plans and delegates, but does not do the physical work.

#### Execution Node (Execution Plane)

The execution node is **where automation actually runs**:

- Receives job instructions from the control node via the Receptor mesh
- Runs playbooks inside Execution Environments (containers)
- Returns results back to the control node
- Can be scaled horizontally — add more execution nodes to handle more concurrent jobs

> **Analogy:** Execution nodes are the "hands" — they carry out the instructions the brain sends.

#### Hybrid Node

A hybrid node does **both** — it runs Controller services and executes jobs on the same machine:

- Used in smaller environments where a dedicated split is not justified
- The standalone install from Lab 02 is effectively a hybrid node
- Not recommended for large-scale production (resource contention between Controller and jobs)

#### Hop Node

A hop node is a **network relay**:

- Does not run Controller services or jobs
- Forwards Receptor mesh traffic between networks that cannot directly communicate
- Used when execution nodes are in an isolated network (DMZ, air-gapped segment)

```
Control Node ──▶ Hop Node (DMZ) ──▶ Execution Node (isolated network)
```

### Section 3 — When to Add More Nodes (Scaling Considerations)

| Symptom                                    | Solution                                  |
|--------------------------------------------|-------------------------------------------|
| Jobs are queuing for a long time           | Add more execution nodes                  |
| Controller UI is slow during heavy job runs | Move job execution to dedicated execution nodes |
| Need to run automation in an isolated network | Add a hop node + execution nodes in that network |
| Different teams need isolated job capacity | Create separate instance groups with dedicated execution nodes |
| Single point of failure concerns           | Add a second control node (HA) or hybrid nodes |

### Section 4 — Capacity Planning Basics

Each node has a **capacity** — the number of job forks it can run concurrently. Capacity is calculated based on CPU and memory:

| Factor    | Impact on Capacity                             |
|-----------|-------------------------------------------------|
| CPU cores | More cores = more concurrent forks              |
| RAM       | More memory = more concurrent forks             |
| Job size  | Complex playbooks with many tasks consume more resources |

**Rule of thumb for labs:**

| Node Specs       | Approximate Concurrent Forks |
|------------------|------------------------------|
| 2 vCPU / 4 GB   | ~10–20 forks                 |
| 4 vCPU / 8 GB   | ~30–50 forks                 |
| 8 vCPU / 16 GB  | ~80–100 forks                |

> **Key concept:** When total job demand exceeds total cluster capacity, jobs queue. Adding execution nodes increases total capacity.

---

## Part 2 — Hands-On: Prepare Execution Node VMs (20 minutes)

Perform the following steps on **each** execution node VM (`exec1.lab.local` and `exec2.lab.local`).

### Step 1 — Set Hostname

On `exec1`:

```bash
sudo hostnamectl set-hostname exec1.lab.local
```

On `exec2`:

```bash
sudo hostnamectl set-hostname exec2.lab.local
```

### Step 2 — Configure Host Resolution

On **all three machines** (control node + both execution nodes), add entries to `/etc/hosts`:

```bash
sudo vi /etc/hosts
```

Add:

```text
192.168.1.10   aap1.lab.local aap1
192.168.1.30   exec1.lab.local exec1
192.168.1.31   exec2.lab.local exec2
```

> Replace the IPs with your actual lab addresses.

Verify resolution from each machine:

```bash
getent hosts aap1.lab.local
getent hosts exec1.lab.local
getent hosts exec2.lab.local
```

### Step 3 — Register and Enable Repositories

On each execution node:

```bash
sudo subscription-manager register --username <RED_HAT_USERNAME> --password <RED_HAT_PASSWORD>
sudo subscription-manager attach --auto
```

Enable the required repos (for RHEL 9):

```bash
sudo subscription-manager repos \
  --enable ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms \
  --enable rhel-9-for-x86_64-baseos-rpms \
  --enable rhel-9-for-x86_64-appstream-rpms
```

Verify:

```bash
sudo dnf repolist
```

### Step 4 — Install Base Packages and Configure Time Sync

```bash
sudo dnf update -y
sudo dnf install -y vim curl wget tar unzip firewalld
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
```

### Step 5 — Open the Receptor Port in the Firewall

The Receptor mesh uses **TCP port 27199** for communication between nodes:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port=27199/tcp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-all
```

### Step 6 — Verify Network Connectivity

From the control node (`aap1.lab.local`), test connectivity to both execution nodes:

```bash
ping -c 3 exec1.lab.local
ping -c 3 exec2.lab.local
ssh root@exec1.lab.local "hostname"
ssh root@exec2.lab.local "hostname"
```

All commands must succeed before continuing.

---

## Part 3 — Hands-On: Add Execution Nodes to AAP (30 minutes)

### Step 7 — Update the AAP Installer Inventory

Go back to the AAP control node and navigate to the installer directory:

```bash
ssh root@aap1.lab.local
cd /tmp/ansible-automation-platform-setup-bundle-*/
```

Edit the inventory file:

```bash
vi inventory
```

Update it to include the execution nodes:

```ini
[automationcontroller]
aap1.lab.local ansible_connection=local

[execution_nodes]
exec1.lab.local
exec2.lab.local

[database]
aap1.lab.local ansible_connection=local

[all:vars]
admin_password='StrongAdminPassword@123'

pg_host='aap1.lab.local'
pg_port=5432
pg_database='awx'
pg_username='awx'
pg_password='StrongDBPassword@123'
pg_sslmode='prefer'

registry_url='registry.redhat.io'
registry_username='<RED_HAT_USERNAME>'
registry_password='<RED_HAT_PASSWORD>'

receptor_listener_port=27199
```

### Understanding the Inventory Changes

| Section               | What Changed                                                 |
|-----------------------|--------------------------------------------------------------|
| `[execution_nodes]`   | New section — declares which hosts will be execution nodes   |
| `exec1.lab.local`     | Will be configured as an execution node and join the Receptor mesh |
| `exec2.lab.local`     | Same — second execution node                                |
| `receptor_listener_port` | Defines the port used for the Receptor mesh (default 27199) |

> **Important:** The `[automationcontroller]` section still lists only the control node. Execution nodes do **not** run Controller services.

### Step 8 — Configure SSH Access from Control Node to Execution Nodes

The installer connects to execution nodes via SSH. Ensure key-based access is configured:

```bash
ssh-copy-id root@exec1.lab.local
ssh-copy-id root@exec2.lab.local
```

Verify passwordless SSH:

```bash
ssh root@exec1.lab.local "hostname"
ssh root@exec2.lab.local "hostname"
```

### Step 9 — Run the Installer

```bash
sudo ./setup.sh
```

The installer will:

1. Configure the Receptor mesh on the control node
2. SSH into each execution node and install Receptor + Execution Environment runtime
3. Register the execution nodes with the control node
4. Pull EE container images onto the execution nodes

> **Expected duration:** 10–20 minutes.

Watch for the `PLAY RECAP` at the end:

```text
PLAY RECAP *********************************************************************
aap1.lab.local    : ok=150  changed=12  unreachable=0  failed=0  ...
exec1.lab.local   : ok=45   changed=30  unreachable=0  failed=0  ...
exec2.lab.local   : ok=45   changed=30  unreachable=0  failed=0  ...
```

All hosts must show `failed=0`.

### Step 10 — Verify Nodes in the Web UI

1. Log in to the AAP web UI: `https://aap1.lab.local`
2. Navigate to **Administration --> Instances**
3. You should see all three nodes listed:

| Instance Name    | Node Type  | Status  |
|------------------|------------|---------|
| aap1.lab.local   | Control    | Healthy |
| exec1.lab.local  | Execution  | Healthy |
| exec2.lab.local  | Execution  | Healthy |

If any node shows **Error** or **Unavailable**, see the Troubleshooting section at the end.

### Step 11 — Verify the Receptor Mesh from CLI

On the control node:

```bash
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock status
```

You should see all nodes in the mesh and their connection status.

To check the status of individual nodes:

```bash
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping exec1.lab.local
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping exec2.lab.local
```

Expected output: a response time in milliseconds confirming connectivity.

---

## Part 4 — Hands-On: Monitoring Node Health and Capacity (15 minutes)

### Step 12 — View Node Details in the UI

1. Navigate to **Administration --> Instances**
2. Click on any node (e.g., `exec1.lab.local`)
3. Review the following information:

| Field           | What It Shows                                        |
|-----------------|------------------------------------------------------|
| Node Type       | Control, Execution, or Hybrid                        |
| Status          | Healthy, Unavailable, or Error                       |
| Capacity        | Total forks this node can handle                     |
| Used Capacity   | Forks currently in use by running jobs               |
| Running Jobs    | Number of jobs currently executing on this node      |
| Last Health Check | When the node was last checked                     |

### Step 13 — Understand Job Distribution

When you launch a job, the Controller scheduler decides which execution node runs it based on:

| Factor          | How It Affects Scheduling                           |
|-----------------|-----------------------------------------------------|
| Available capacity | Nodes with more free capacity get jobs first      |
| Instance group    | If the job template is pinned to an instance group, only nodes in that group are considered |
| Node health       | Unhealthy nodes are skipped                        |

### Step 14 — Test Job Execution Across Nodes

1. Navigate to **Resources --> Templates**
2. Open your `Ping-Lab` job template from Lab 02 (or create a new one)
3. Launch the job
4. After the job completes, click on the job details
5. Look at the **Execution Node** field — it should show one of the execution nodes (`exec1.lab.local` or `exec2.lab.local`) instead of the control node

Run the job several times and observe which execution node picks it up. The scheduler distributes jobs based on available capacity.

---

## Part 5 — Hands-On: Instance Groups (40 minutes)

### Section 5 — What Are Instance Groups and Why Use Them?

An **instance group** is a logical grouping of nodes. It allows you to control **which nodes run which jobs**.

| Without Instance Groups                    | With Instance Groups                         |
|--------------------------------------------|----------------------------------------------|
| All jobs run on any available node          | Jobs run only on nodes assigned to the group |
| No separation between Dev and Prod workloads | Dev jobs run on Dev nodes, Prod on Prod nodes |
| A runaway Dev job could consume Prod capacity | Workloads are isolated by group            |
| No compliance boundary                      | Meets compliance requirements for separation |

**Common instance group patterns:**

| Instance Group   | Nodes Assigned       | Purpose                                  |
|------------------|----------------------|------------------------------------------|
| `default`        | All nodes            | Catch-all group (always exists)          |
| `dev-workers`    | exec1.lab.local      | Development and testing automation       |
| `prod-workers`   | exec2.lab.local      | Production automation (isolated)         |
| `network-team`   | exec3 (future)       | Network automation with specialized EEs  |

### Step 15 — Create the "dev-workers" Instance Group

1. Navigate to **Administration --> Instance Groups**
2. Click **Add**
3. Fill in:
   - **Name:** `dev-workers`
4. Click **Save**
5. Click the **Instances** tab
6. Click **Associate**
7. Select `exec1.lab.local` and click **Save**

### Step 16 — Create the "prod-workers" Instance Group

1. Navigate to **Administration --> Instance Groups**
2. Click **Add**
3. Fill in:
   - **Name:** `prod-workers`
4. Click **Save**
5. Click the **Instances** tab
6. Click **Associate**
7. Select `exec2.lab.local` and click **Save**

### Step 17 — Verify Instance Groups

Navigate to **Administration --> Instance Groups**. You should now see:

| Instance Group   | Nodes              |
|------------------|--------------------|
| default          | aap1, exec1, exec2 |
| controlplane     | aap1               |
| dev-workers      | exec1              |
| prod-workers     | exec2              |

### Step 18 — Pin a Job Template to an Instance Group

This ensures jobs for a specific template always run on a specific set of nodes.

1. Navigate to **Resources --> Templates**
2. Open your `Ping-Lab` template (or create a new one)
3. In the **Instance Groups** field, select `dev-workers`
4. Click **Save**
5. **Launch** the job

After the job completes, verify:
- The **Execution Node** field shows `exec1.lab.local`
- The job did **not** run on `exec2.lab.local`

### Step 19 — Test Isolation Between Instance Groups

Create a second job template to verify isolation:

1. Navigate to **Resources --> Templates**
2. Click **Add --> Add job template**
3. Fill in:
   - **Name:** `Ping-Prod`
   - **Inventory:** `Lab-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `ping.yml`
   - **Credential:** `SSH-Key-Lab`
   - **Instance Groups:** `prod-workers`
4. Click **Save**
5. **Launch** the job

Verify:
- The **Execution Node** shows `exec2.lab.local`
- The `dev-workers` node (`exec1`) was **not** used

This confirms that instance groups correctly isolate workloads to their assigned nodes.

---

## Part 6 — Hands-On: Removing and Decommissioning Nodes (20 minutes)

### Step 20 — Remove a Node from an Instance Group

To remove a node from a specific instance group (without removing it from AAP entirely):

1. Navigate to **Administration --> Instance Groups**
2. Click on `dev-workers`
3. Click the **Instances** tab
4. Click the disassociate (remove) button next to `exec1.lab.local`
5. Confirm the action

The node is no longer part of `dev-workers`, but it remains in the `default` group and is still available for other jobs.

### Step 21 — Decommission a Node Entirely

To fully remove an execution node from AAP:

**Step A — Remove from all instance groups**

Follow Step 20 for every instance group the node belongs to.

**Step B — Update the installer inventory**

On the control node, edit the installer inventory:

```bash
cd /tmp/ansible-automation-platform-setup-bundle-*/
vi inventory
```

Remove the node from `[execution_nodes]`:

```ini
[execution_nodes]
exec2.lab.local
# exec1.lab.local has been removed
```

**Step C — Run the installer to apply the change**

```bash
sudo ./setup.sh
```

**Step D — Verify in the web UI**

Navigate to **Administration --> Instances**. The removed node should no longer appear.

**Step E — Clean up the decommissioned VM (optional)**

On the former execution node, if you want to uninstall the Receptor and AAP components:

```bash
sudo dnf remove -y receptor ansible-automation-platform-*
sudo rm -rf /var/run/awx-receptor
sudo rm -rf /etc/receptor
```

### Step 22 — Re-add the Node (Undo the Removal)

To add the node back, simply:

1. Re-add it to `[execution_nodes]` in the inventory file
2. Re-run `sudo ./setup.sh`
3. Verify it appears in the UI under **Administration --> Instances**

> **For this lab:** Re-add `exec1.lab.local` so you have both execution nodes available for subsequent exercises.

---

## Part 7 — Real-Time Use Case Discussion (15 minutes)

### Case Study: Healthcare Organization — Compliance-Driven Instance Group Separation

**Background:**

A healthcare organization manages 800+ servers across two environments:

| Environment     | Servers | Compliance Requirement                       |
|-----------------|---------|----------------------------------------------|
| Production      | 500     | HIPAA — strict change control, audit logging |
| Non-Production  | 300     | Standard — faster iteration, less restriction |

**Problem:**

With a single AAP installation and no instance groups, a developer testing a playbook in the non-production environment could accidentally consume all execution capacity, causing production patch jobs to queue and miss their maintenance window.

**Solution using Instance Groups:**

| Instance Group      | Nodes Assigned       | Policies                                     |
|---------------------|----------------------|----------------------------------------------|
| `prod-automation`   | 4 dedicated exec nodes | Reserved for production jobs only. RBAC restricts access to the Ops team. All jobs logged for HIPAA audit. |
| `nonprod-automation`| 2 dedicated exec nodes | Used by Dev and QA teams. No impact on production capacity. |
| `security-scans`    | 1 dedicated exec node  | Runs CIS benchmark scans on a schedule. Isolated from application workloads. |

**Results:**

| Metric                        | Before Instance Groups | After Instance Groups  |
|-------------------------------|------------------------|------------------------|
| Production job queue time     | Up to 45 minutes       | Under 2 minutes        |
| Cross-environment interference | Frequent               | Zero                   |
| HIPAA audit compliance        | Partial                | Full                   |
| Team satisfaction              | Low (job contention)   | High (dedicated capacity) |

**Key takeaway:** Instance groups provide workload isolation, predictable capacity, and compliance boundaries — all without needing separate AAP installations.

---

## Troubleshooting

### Node Shows "Unavailable" in the UI

```bash
# On the control node — check Receptor mesh status
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock status

# Ping the execution node via Receptor
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping exec1.lab.local

# On the execution node — check if Receptor service is running
sudo systemctl status receptor

# Check Receptor logs
sudo journalctl -u receptor --no-pager -n 50
```

Common causes:
- Receptor service not running on the execution node
- Port 27199 blocked by firewall
- Hostname resolution failure between nodes

### Installer Fails When Adding Execution Nodes

```bash
# Verify SSH access from control to execution node
ssh root@exec1.lab.local "hostname"

# Check repos are enabled on the execution node
ssh root@exec1.lab.local "dnf repolist"

# Re-run installer with verbose output
sudo ./setup.sh -v
```

Common causes:
- SSH key not copied to the execution node
- Missing Red Hat subscription or repos on the execution node
- Insufficient disk space on the execution node (`df -h`)

### Jobs Not Running on Execution Nodes

```bash
# Verify nodes are in the correct instance group
# UI: Administration --> Instance Groups --> (group name) --> Instances tab

# Verify the job template has the correct instance group assigned
# UI: Resources --> Templates --> (template name) --> Instance Groups field

# Check node capacity
# UI: Administration --> Instances --> (node name) --> Capacity
```

Common causes:
- Job template is not assigned to any instance group (falls back to `default`)
- Execution node capacity is full (all forks in use)
- Node is in an error state (check health)

### Receptor Mesh Connectivity Test

Run from the control node to test end-to-end mesh health:

```bash
# List all known nodes in the mesh
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock status

# Test latency to each node
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping exec1.lab.local
sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping exec2.lab.local
```

---

## Key Concepts Covered in This Lab

| Concept            | What You Learned                                                 |
|--------------------|------------------------------------------------------------------|
| Control Node       | Hosts the Controller — makes scheduling and RBAC decisions       |
| Execution Node     | Dedicated worker — runs playbooks and automation jobs            |
| Hybrid Node        | Combines both roles — suitable for small environments            |
| Hop Node           | Network relay for isolated/air-gapped environments               |
| Receptor Mesh      | Encrypted overlay network connecting all AAP nodes               |
| Instance Group     | Logical grouping of nodes for workload isolation                 |
| Capacity           | A node's ability to run concurrent job forks (based on CPU/RAM)  |
| Job Distribution   | The scheduler assigns jobs based on capacity, health, and instance group |
| Node Decommission  | Remove a node from instance groups, update inventory, re-run installer |

---

## Lab Completion Checklist

- [ ] Can explain the difference between control, execution, hybrid, and hop nodes
- [ ] Two execution node VMs are prepared (hostname, repos, firewall, time sync)
- [ ] Execution nodes are added to AAP via the installer inventory
- [ ] Both execution nodes appear as "Healthy" in the web UI under **Administration --> Instances**
- [ ] Receptor mesh connectivity is verified (`receptorctl ping`)
- [ ] Instance group `dev-workers` is created with `exec1` assigned
- [ ] Instance group `prod-workers` is created with `exec2` assigned
- [ ] A job template pinned to `dev-workers` runs only on `exec1`
- [ ] A job template pinned to `prod-workers` runs only on `exec2`
- [ ] Demonstrated removing a node from an instance group
- [ ] Demonstrated the full node decommission process (and re-added the node)
- [ ] Can explain the healthcare compliance use case for instance group separation

---

## Quick Reference — Key Commands

| Task | Command |
|------|---------|
| Check Receptor service | `sudo systemctl status receptor` |
| Receptor mesh status | `sudo receptorctl --socket /var/run/awx-receptor/receptor.sock status` |
| Ping a node via Receptor | `sudo receptorctl --socket /var/run/awx-receptor/receptor.sock ping <node>` |
| Check Receptor logs | `sudo journalctl -u receptor -f` |
| Check Controller service | `sudo systemctl status automation-controller` |
| Verify firewall rules | `sudo firewall-cmd --list-all` |
| Check hostname resolution | `getent hosts <hostname>` |
| Re-run AAP installer | `sudo ./setup.sh` (from installer directory) |
| Check node capacity | Web UI: Administration --> Instances --> (node) |

---

*End of Lab 03*
