# Lab 08 — Build and Manage Job Workflows (Approvals, Recovery, Orchestration)

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate to Advanced (Ops / DevOps / Platform Engineers)               |
| **Duration** | 5 hours (Theory: 20% / Hands-on: 80%)                                     |
| **Outcome**  | Build workflow templates with job nodes, approval gates, project/inventory syncs; implement convergence, failure handling, artifact passing with set_stats; design multi-tier deployment and recovery workflows |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Labs 02, 04, 05, 07 completed — AAP 2.4 installed, inventories, projects, job templates, machine credential configured. |

---

## AAP 2.4 UI Navigation Reference

| Menu | Location | Contains |
|------|----------|----------|
| **Resources** | Left navigation panel | Hosts, Inventories, Projects, Credentials, **Templates** |
| **Templates** | Resources → Templates | Job Templates, **Workflow Job Templates** |
| **Workflow Visualizer** | Workflow template → Visualizer tab or icon | Graph editor for building workflows |

> **Note:** Workflow templates have a **Visualizer** icon. Use it to add nodes (job templates, approval, project sync, inventory sync) and connect them with success/failure/always paths.

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
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  WORKFLOW TEMPLATES                                                          │   │
│  │  - Deploy-Stack-Basic (sequential: DB → App → LB)                            │   │
│  │  - Deploy-Stack-Approval (with approval gate before prod)                    │   │
│  │  - Deploy-Stack-Recovery (failure paths, notifications)                      │   │
│  │  - Deploy-Stack-Artifacts (set_stats between nodes)                           │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Job Templates: Deploy-DB, Deploy-App, Deploy-LB, Ping-Lab, Project-Sync-Demo         │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ SSH (22)
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │ db1.lab.local │  │ app1.lab.local│  │ web1.lab.local │
            │ [databases]   │  │ [appservers]  │  │ [webservers]   │
            └───────────────┘  └───────────────┘  └───────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` with playbooks in `lab-playbooks` |
| Inventory | `Lab-Static-Inventory` with groups (webservers, appservers, databases) and hosts |
| Job Templates | At least `Ping-Lab`; we will create `Deploy-DB`, `Deploy-App`, `Deploy-LB` in this lab |
| Credential | Machine credential (e.g., `SSH-Key-Lab`) |

> **Minimal lab:** If you have only 2–3 nodes, assign the same host to multiple groups (e.g., `node1` in webservers, appservers, and databases). The playbooks target groups; hosts in those groups will run the tasks.

---

## Verify AAP 2.4 Before Starting

1. Log in to the Automation Controller.
2. **Resources → Templates** — confirm you see Job Templates and Workflow Job Templates.
3. Ensure at least one job template exists (e.g., `Ping-Lab` from Lab 07).

---

## Part 1 — Theory: Understanding Workflows (60 minutes)

### Section 7.1 — Introduction to Workflows

#### What Are Workflows and When to Use Them?

A **workflow** (workflow job template) links multiple automation steps into a single unit. Each step can be:

| Node Type | Purpose |
|-----------|---------|
| **Job Template** | Run a playbook |
| **Workflow Job Template** | Run another workflow (nested) |
| **Project Sync** | Sync project from Git/SCM |
| **Inventory Sync** | Sync inventory from source |
| **Approval** | Pause for human approval |

**When to use workflows:**

| Scenario | Use Workflow |
|----------|--------------|
| Sequential deployment (DB → App → LB) | Yes |
| Parallel tasks (deploy to region A and B, then converge) | Yes |
| Human approval before production changes | Yes |
| Sync project before running playbook | Yes (project sync node) |
| Single playbook run | Job template only |

#### Benefits of Workflow Automation

| Benefit | Description |
|---------|-------------|
| **Single unit of tracking** | All steps tracked as one workflow job; audit trail for entire release |
| **Reusable orchestration** | Same workflow for dev, staging, prod with different inventories |
| **Failure isolation** | Route failures to recovery steps (notify, rollback, retry) |
| **Convergence** | Wait for multiple parallel jobs before proceeding |
| **Approval gates** | Compliance: require sign-off before production |

#### Real-World Workflow Scenarios

| Scenario | Workflow Structure |
|----------|-------------------|
| **App deployment** | Project Sync → Deploy DB → Deploy App → Deploy LB |
| **Blue-green** | Deploy to green → Smoke test → Approval → Switch traffic |
| **Rolling update** | Deploy batch 1 → Deploy batch 2 → Deploy batch 3 (or parallel slices) |
| **Compliance** | Scan → Report → Approval → Remediate |
| **Backup & restore** | Backup → Verify → (on failure) Alert |

#### Understanding the Workflow Visualizer

The **Workflow Visualizer** is a graph editor:

- **Nodes** — rectangles (job template, approval, sync)
- **Edges** — lines connecting nodes with **On Success**, **On Failure**, or **Always**
- **Root node** — first node (default: Always)
- **Convergence** — when multiple nodes feed into one: **All** (all must succeed) or **Any** (one suffices)

---

### Section 7.2 — Building Basic Workflows

#### Creating a Workflow Template

1. **Resources → Templates → Add → Add workflow template**
2. **Name**, **Organization**, **Inventory** (optional)
3. **Save** — the Workflow Visualizer opens

#### Adding Job Template Nodes

1. Click **Start** (or + on a node)
2. Select **Job Template** (or **Template**)
3. Choose a job template from the list
4. Select **On Success**, **On Failure**, or **Always** for the edge
5. Click **Save** or **Select**

#### Connecting Nodes with Success/Failure/Always Paths

| Edge Type | When the next node runs |
|-----------|-------------------------|
| **On Success** | Parent job completed successfully |
| **On Failure** | Parent job failed |
| **Always** | Regardless of success or failure |

#### Understanding Convergence and Divergence

| Pattern | Description |
|---------|-------------|
| **Divergence** | One node has multiple children — parallel execution |
| **Convergence** | Multiple nodes feed into one — child waits for parents |
| **Convergence: All** | All parents must complete as specified (e.g., all success) |
| **Convergence: Any** | Any parent meeting the edge condition triggers the child |

---

### Section 7.3 — Advanced Workflow Features

#### Approval Nodes: Adding Manual Gates

- **Approval** node pauses the workflow until a user approves or denies.
- **Approvers:** Users with Approve permission, org admins, or users with Execute on the workflow.
- **Timeout:** Optional — if not approved within the time, the node times out (treated as failure).
- **Paths:** On Success → approved; On Failure → denied or timed out.

#### Inventory Sync and Project Sync Nodes

| Node | Purpose |
|------|---------|
| **Project Sync** | Sync project from Git before downstream jobs use latest code |
| **Inventory Sync** | Sync inventory source (e.g., EC2, file) before downstream jobs |

#### Workflow Job Templates as Nodes

- A workflow can include another workflow as a node (nested workflows).
- Nested workflow inherits variables if prompted or from survey.

#### Passing Variables Between Nodes

- Use **set_stats** in a playbook to store data in the workflow **artifacts** dictionary.
- Downstream jobs receive artifacts as extra variables.
- **Important:** Use `per_host: false` (default) so data is aggregated, not per-host.

#### Using set_stats for Artifact Sharing

```yaml
- name: Store result for downstream
  ansible.builtin.set_stats:
    data:
      deploy_version: "{{ app_version }}"
      db_host: "{{ groups['databases'][0] }}"
```

Downstream playbook uses `{{ deploy_version }}` and `{{ db_host }}` as extra vars.

---

### Section 7.4 — Error Handling and Recovery

#### Designing Failure Handling in Workflows

| Strategy | Implementation |
|----------|----------------|
| **Notify on failure** | On Failure → Job template that sends Slack/email |
| **Rollback** | On Failure → Rollback job template |
| **Retry** | On Failure → Same job template (with retry limit) |
| **Skip and continue** | On Failure → Next step (e.g., Always path) |

#### Creating Recovery Paths

- Add **On Failure** edges to recovery nodes.
- Recovery node can: send notification, run rollback playbook, create ticket.

#### Implementing Retry Logic

- Workflows do not have built-in retry. Use a job template that retries internally, or add a manual "Retry" workflow that re-launches the failed job.

#### Notification Integration in Workflows

- Attach notification templates to the **workflow job template** (same as job templates).
- Notifications fire for workflow start, success, failure.

#### Debugging Workflow Failures

- **Jobs** tab on the workflow — expand to see each node's job.
- Click a failed node to view its job output.
- Check **Activity Stream** for approval events.

---

### Section 7.5 — Complex Orchestration Patterns

#### Multi-Tier Application Deployment Workflows

```
Project Sync → Deploy DB → Deploy App → Deploy LB → Notify
     ↓              ↓            ↓            ↓
  (On Failure)  (On Failure) (On Failure) (On Failure)
     ↓              ↓            ↓            ↓
  Alert         Rollback DB   Rollback App  Rollback LB
```

#### Rolling Update Patterns

- Slice inventory into batches (e.g., 10% at a time).
- Job template with limit: `webservers[0:2]`, then `webservers[2:4]`, etc.
- Or use **Job Slicing** on a job template within the workflow.

#### Blue-Green Deployment Workflows

1. Deploy to green environment
2. Smoke test
3. Approval node
4. Switch load balancer to green
5. (On Failure from approval) Keep blue active, notify

#### Compliance and Validation Workflows

1. Scan/audit job
2. Report job (set_stats with findings)
3. Approval (review findings)
4. Remediate job (on success)

#### Backup and Disaster Recovery Workflows

1. Backup job
2. Verify backup job
3. (On Failure) Alert and optionally run restore test

---

## Part 2 — Hands-On: Prepare Job Templates and Playbooks (30 minutes)

### Step 1 — Create Playbooks for Multi-Tier Deployment

**Purpose:** Playbooks that simulate DB, App, and LB deployment for workflow exercises.

**Detailed steps:**

1. **SSH to the AAP server** and create the playbooks:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-playbooks
```

2. **Create `deploy_db.yml`:**

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/deploy_db.yml > /dev/null << 'EOF'
---
- name: Deploy database tier (simulated)
  hosts: databases
  gather_facts: true
  vars:
    db_version: "{{ db_version | default('14') }}"
  tasks:
    - name: Create DB data directory
      ansible.builtin.file:
        path: "/opt/db/data"
        state: directory
        mode: "0755"
    - name: Report DB deployment
      ansible.builtin.debug:
        msg: "DB tier deployed, version {{ db_version }}"
EOF
```

3. **Create `deploy_app.yml`** (simplified for workflow):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/deploy_app.yml > /dev/null << 'EOF'
---
- name: Deploy application tier
  hosts: appservers
  gather_facts: true
  vars:
    app_version: "{{ app_version | default('1.0') }}"
  tasks:
    - name: Create app directory
      ansible.builtin.file:
        path: "/opt/app"
        state: directory
        mode: "0755"
    - name: Report app deployment
      ansible.builtin.debug:
        msg: "App tier deployed, version {{ app_version }}"
EOF
```

4. **Create `deploy_lb.yml`:**

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/deploy_lb.yml > /dev/null << 'EOF'
---
- name: Deploy load balancer config (simulated)
  hosts: webservers
  gather_facts: false
  tasks:
    - name: Report LB config
      ansible.builtin.debug:
        msg: "LB config applied to {{ inventory_hostname }}"
EOF
```

5. **Create `set_stats_producer.yml`** (for artifact passing):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/set_stats_producer.yml > /dev/null << 'EOF'
---
- name: Produce artifacts for downstream
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Set artifact for downstream
      ansible.builtin.set_stats:
        data:
          deploy_id: "{{ 1000 | random }}"
          deploy_time: "{{ ansible_date_time.iso8601 }}"
EOF
```

6. **Create `set_stats_consumer.yml`:**

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/set_stats_consumer.yml > /dev/null << 'EOF'
---
- name: Consume artifacts from upstream
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Display artifact
      ansible.builtin.debug:
        msg: "Received deploy_id={{ deploy_id | default('N/A') }}, deploy_time={{ deploy_time | default('N/A') }}"
EOF
```

7. **Set ownership and sync project:**

```bash
sudo chown -R awx:awx /var/lib/awx/projects/lab-playbooks
```

8. **In AAP:** **Resources → Projects** → open `Lab-Project` → **Sync**. Verify all playbooks appear.

### Step 2 — Create Job Templates for the Workflow

**Purpose:** Job templates that the workflow will use as nodes.

**Detailed steps:**

1. **Resources → Templates → Add → Add job template**
   - **Name:** `Deploy-DB`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `deploy_db.yml`
   - **Credentials:** Machine credential
   - **Save**

2. **Create `Deploy-App`:**
   - Same as above, **Playbook:** `deploy_app.yml`

3. **Create `Deploy-LB`:**
   - Same as above, **Playbook:** `deploy_lb.yml`

4. **Create `Set-Stats-Producer`:**
   - **Inventory:** Create/use a minimal inventory with `localhost` or use `Lab-Static-Inventory` (playbook uses localhost)
   - **Playbook:** `set_stats_producer.yml`
   - **Credentials:** Not required for localhost (or use a minimal credential)

5. **Create `Set-Stats-Consumer`:**
   - Same inventory, **Playbook:** `set_stats_consumer.yml`

> **Note:** For `hosts: localhost` playbooks, the job runs on the controller. Use `Lab-Static-Inventory` or any inventory with at least one host — the playbook targets localhost with `connection: local`.

---

## Part 3 — Hands-On: Basic Workflow (40 minutes)

### Step 3 — Create a Sequential Workflow

**Purpose:** Build a workflow that runs DB → App → LB in sequence.

**Detailed steps:**

1. **Resources → Templates → Add → Add workflow template**
   - **Name:** `Deploy-Stack-Basic`
   - **Organization:** `Default`
   - **Inventory:** `Lab-Static-Inventory`
   - **Save**

2. **Workflow Visualizer** opens. Click **Start**.

3. **Add first node:** Select **Template** → choose `Deploy-DB` → **On Success** → **Save**.

4. **Add second node:** Hover over the `Deploy-DB` node → click **+** → **Template** → `Deploy-App` → **On Success** → **Save**.

5. **Add third node:** Hover over `Deploy-App` → **+** → **Template** → `Deploy-LB` → **On Success** → **Save**.

6. **Save** the workflow (click **Save** in the visualizer).

7. **Launch** the workflow. Verify all three jobs run in sequence and complete successfully.

### Step 4 — Add Convergence (Parallel Then Converge)

**Purpose:** Run two jobs in parallel, then a third after both complete.

**Detailed steps:**

1. **Create a new workflow:** `Deploy-Stack-Convergence`
2. **Add first root node:** Click **Start** → **Template** → `Deploy-DB` → **On Success** → **Save**
3. **Add sibling from Start:** Hover over the **Start** node → click **+** → **Template** → `Deploy-App` → **On Success** → **Save**. Now `Deploy-DB` and `Deploy-App` are siblings (both from Start).
4. **Add convergence node:** Hover over `Deploy-DB` → **+** → **Template** → `Deploy-LB` → **On Success** → **Save**. Then hover over `Deploy-App` → **+** (link icon) → link to existing `Deploy-LB` → **On Success**.
5. **Set convergence:** Click the `Deploy-LB` node → in the pane, set **Convergence** to **All** (both parents must succeed before Deploy-LB runs).
6. **Save** and **Launch**. Verify `Deploy-DB` and `Deploy-App` run in parallel, then `Deploy-LB` runs after both complete.

---

## Part 4 — Hands-On: Approval Node (35 minutes)

### Step 5 — Add an Approval Gate

**Purpose:** Pause the workflow for human approval before deploying to production.

**Detailed steps:**

1. **Edit** `Deploy-Stack-Basic` (or create `Deploy-Stack-Approval`).
2. **Workflow Visualizer** — we will insert an Approval node between `Deploy-App` and `Deploy-LB`.
3. **Insert node:** Hover over the line between `Deploy-App` and `Deploy-LB` → click **+**.
4. **Select Approval** node.
5. **Configure:**
   - **Name:** `Prod-Approval`
   - **Description:** `Approve before deploying to production LB`
   - **Timeout:** `600` (10 minutes) or leave empty for no timeout
6. **Save**.
7. **Launch** the workflow. It will pause at the Approval node.
8. **Approve:** Go to **Jobs** → open the workflow job → click the **Approval** node → **Approve** (or **Deny**). If you approve, the workflow continues to `Deploy-LB`.
9. **Test deny:** Launch again, and **Deny** at the approval — the workflow should follow the **On Failure** path (if configured) or stop.

---

## Part 5 — Hands-On: Project Sync and Inventory Sync (25 minutes)

### Step 6 — Add Project Sync Node

**Purpose:** Sync the project from Git before running deployment jobs.

**Detailed steps:**

1. **Create workflow** `Deploy-Stack-With-Sync`.
2. **Add root node:** Select **Project Sync** → choose your project (e.g., `Lab-Project`).
3. **Add child:** **On Success** → `Deploy-DB`.
4. **Continue:** `Deploy-DB` → `Deploy-App` → `Deploy-LB`.
5. **Save** and **Launch**. The project sync runs first; on success, the deployment jobs run.

### Step 7 — Add Inventory Sync Node (If You Have Dynamic Inventory)

**Purpose:** Sync inventory before jobs run (e.g., pull latest from EC2).

**Detailed steps:**

1. If you have an inventory with a **source** (e.g., Sourcing from a Project, AWS EC2), add an **Inventory Sync** node.
2. Select the inventory source to sync.
3. Connect **On Success** to the first deployment job.
4. **Save** and **Launch**.

> **Lab note:** If you only have a static inventory, skip this step or use a project-sourced inventory from Lab 04.

---

## Part 6 — Hands-On: set_stats and Artifact Passing (30 minutes)

### Step 8 — Pass Variables Between Nodes with set_stats

**Purpose:** Use set_stats in one job and consume the artifact in a downstream job.

**Detailed steps:**

1. **Verify set_stats playbook:** The `set_stats` module stores data in workflow artifacts. Use `per_host: false` (default) so data is aggregated for downstream. The playbook from Step 1 should work as-is.

2. **Create workflow** `Deploy-Stack-Artifacts`.
3. **Add nodes:** `Set-Stats-Producer` (root) → **On Success** → `Set-Stats-Consumer`.
4. **Save** and **Launch**.
5. **Verify** the consumer job output shows `deploy_id` and `deploy_time` from the producer.

> **Note:** Ensure the inventory for both job templates allows the playbook to run. For `localhost` playbooks, the inventory must include a host; the playbook targets `localhost` with `connection: local`, so any host works as long as the job runs on the controller.

---

## Part 7 — Hands-On: Error Handling and Recovery (40 minutes)

### Step 9 — Add Failure Paths

**Purpose:** Route failures to a notification or recovery step.

**Detailed steps:**

1. **Create a simple "Notify on Failure" job template:** Use a playbook that only runs `debug` or a real notification. For simplicity, create `Notify-Failure` with a playbook:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/notify_failure.yml > /dev/null << 'EOF'
---
- name: Notify on failure (simulated)
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Log failure
      ansible.builtin.debug:
        msg: "Workflow step failed - would send alert"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/notify_failure.yml
```

2. **Create job template** `Notify-Failure` with playbook `notify_failure.yml`.
3. **Edit** `Deploy-Stack-Basic` (or create `Deploy-Stack-Recovery`).
4. **Add On Failure edge:** From `Deploy-DB` → `Notify-Failure` (On Failure). From `Deploy-App` → `Notify-Failure` (On Failure). From `Deploy-LB` → `Notify-Failure` (On Failure).
5. **Save**.
6. **Test:** Temporarily break `Deploy-DB` (e.g., change playbook to fail) or use a job template that fails. Launch the workflow — it should route to `Notify-Failure` on failure.

### Step 10 — Attach Notifications to Workflow

**Purpose:** Send Slack/email when workflow completes or fails.

**Detailed steps:**

1. **Edit** `Deploy-Stack-Basic` → **Notifications** tab.
2. **Add** a notification template (e.g., `Slack-Job-Status` from Lab 04) for **On Success** and **On Error**.
3. **Save**.
4. **Launch** — verify notification is sent when the workflow completes.

---

## Part 8 — Hands-On: Complex Orchestration Pattern (30 minutes)

### Step 11 — Build Multi-Tier Deployment with Approval

**Purpose:** Implement the retail use case pattern: DB → App → Approval → LB.

**Detailed steps:**

1. **Create workflow** `Deploy-Stack-Production`.
2. **Structure:**
   - Root: `Deploy-DB` (On Success)
   - `Deploy-DB` → `Deploy-App` (On Success)
   - `Deploy-App` → **Approval** node "Production Approval" (On Success)
   - **Approval** → `Deploy-LB` (On Success)
   - **Approval** → `Notify-Failure` (On Failure) — if denied or timed out
   - `Deploy-DB` / `Deploy-App` / `Deploy-LB` → `Notify-Failure` (On Failure) — optional
3. **Save** and **Launch**.
4. **Verify** the workflow pauses at approval; approve to continue to LB deployment.

---

## Part 9 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Retail Company — Complete Application Stack Deployment

**Background:**

A retail company deploys a full application stack: database → application servers → load balancers → monitoring. Production changes require approval from the platform team.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Manual deployment order | Human error; wrong sequence causes outages |
| No approval for prod | Uncontrolled changes; compliance gaps |
| Failure handling | Failures go unnoticed; no automatic rollback |
| Multi-team coordination | DB team, app team, infra team — handoffs are slow |

**Solution: Workflow with Approval Gates**

| Component | Implementation |
|-----------|----------------|
| **Project Sync** | First node — always pull latest code before deploy |
| **Deploy DB** | Migration playbook; schema updates |
| **Deploy App** | Application deployment; config from survey |
| **Approval** | "Production deployment — approve to proceed to LB" — 24h timeout |
| **Deploy LB** | Update load balancer config; switch traffic |
| **On Failure** | Notify Slack; create Jira ticket; optional rollback job |
| **Notifications** | Workflow-level: start, success, failure |

**Workflow Structure:**

```
Project Sync
    ↓ (On Success)
Deploy DB
    ↓ (On Success)     ↓ (On Failure)
Deploy App         →  Notify + Rollback DB
    ↓ (On Success)     ↓ (On Failure)
Approval           →  Notify (denied/timeout)
    ↓ (On Success)
Deploy LB
    ↓ (On Success)     ↓ (On Failure)
Notify Success      →  Notify + Rollback
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Deployment time | 4–6 hours (manual) | 30 minutes (automated) |
| Production incidents | 2–3/month | <1/month |
| Approval compliance | 60% | 100% |
| Rollback time | 2 hours | 15 minutes |

**Key takeaway:** Workflows encode best practices (order, approval, failure handling) into repeatable automation. Approval nodes provide governance without blocking speed for non-prod environments.

---

## Troubleshooting

### Workflow Visualizer: Cannot Add Node

| Symptom | Check |
|---------|-------|
| Job template grayed out | Job template may require prompted inventory/credential with no default |
| "Credential required" | Ensure job template has a default credential |
| "Select" disabled | Provide required survey defaults or prompt values at node level |

### Approval Node: No One Can Approve

| Symptom | Check |
|---------|-------|
| User cannot approve | Grant **Approve** permission on the workflow template (Access tab) |
| Org admin | Org admins can approve by default |
| Execute permission | User needs Execute on workflow to launch; Approve is separate |

### set_stats Not Passed to Downstream

| Symptom | Check |
|---------|-------|
| Variable undefined | Use `per_host: false` (default); ensure unique keys in convergence |
| Wrong variable name | Downstream uses exact key from `data:` in set_stats |
| Convergence | In convergence, set_stats from multiple parents merge; use unique keys |

### Workflow Fails at First Node

| Symptom | Check |
|---------|-------|
| Job fails | Click the failed node → view job output |
| Inventory empty | Ensure inventory has hosts for the job template's target (e.g., `databases` group) |
| Credential | Machine credential must work for target hosts |

### Convergence Node Never Runs

| Symptom | Check |
|---------|-------|
| "All" convergence | All parents must complete with the edge type (e.g., all On Success) |
| "Any" convergence | At least one parent must meet the condition |
| Parent failed | If one parent fails and edge is On Success, that branch does not trigger success path |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Workflow template | Graph of nodes (job, approval, sync) |
| Edge types | On Success, On Failure, Always |
| Convergence | Multiple parents → one child; All vs. Any |
| Divergence | One parent → multiple children (parallel) |
| Approval node | Manual gate; timeout; Approve/Deny |
| Project sync | Sync project before jobs |
| Inventory sync | Sync inventory before jobs |
| set_stats | Pass artifacts to downstream jobs |
| Failure handling | On Failure edges to recovery/notify |
| Notifications | Attach to workflow template |

---

## Lab Completion Checklist

- [ ] Playbooks created (deploy_db, deploy_app, deploy_lb, set_stats producer/consumer)
- [ ] Job templates created (Deploy-DB, Deploy-App, Deploy-LB, Set-Stats-Producer, Set-Stats-Consumer)
- [ ] Basic sequential workflow created and tested
- [ ] Convergence workflow (parallel → converge) created and tested
- [ ] Approval node added and tested (approve and deny)
- [ ] Project sync node added to workflow
- [ ] set_stats artifact passing tested
- [ ] Failure path (On Failure → Notify) implemented
- [ ] Notifications attached to workflow
- [ ] Multi-tier deployment with approval workflow built
- [ ] Can explain the retail company use case

---

## Quick Reference — Workflow Node Types (AAP 2.4)

| Node | Purpose |
|------|---------|
| Job Template | Run playbook |
| Workflow Job Template | Nested workflow |
| Project Sync | Sync project from SCM |
| Inventory Sync | Sync inventory source |
| Approval | Pause for human approval |

---

## Quick Reference — Edge Types

| Edge | When child runs |
|------|-----------------|
| On Success | Parent succeeded |
| On Failure | Parent failed |
| Always | Regardless of outcome |

---

*End of Lab 08*
