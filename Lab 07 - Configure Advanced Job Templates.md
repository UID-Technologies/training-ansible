# Lab 07 — Configure Advanced Job Templates

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 3 hours (Theory: 25% / Hands-on: 75%)                                      |
| **Outcome**  | Create and configure job templates with surveys, limit patterns, tags, job slicing, fact cache; understand workflow vs. job templates; implement check mode, callback, and timeout settings |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Labs 02, 04, 05 completed — AAP 2.4 installed, inventory with hosts, project with playbooks, machine credential configured. |

---

## AAP 2.4 UI Navigation Reference

| Menu | Location | Contains |
|------|----------|----------|
| **Resources** | Left navigation panel | Hosts, Inventories, Projects, Credentials, **Templates** |
| **Administration** | Left navigation panel | Notifications, Instances, Instance Groups, Execution Environments |
| **Settings** | Left navigation panel | System, Jobs, License |

> **Note:** In AAP 2.4, **Templates** includes both **Job Templates** and **Workflow Job Templates**. Job templates run a single playbook; workflow templates orchestrate multiple job templates or workflows.

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
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                       │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  JOB TEMPLATES                                                                │   │
│  │  - Ping-Lab (basic)                                                           │   │
│  │  - Deploy-App (survey, tags, limit)                                            │   │
│  │  - Patch-Hosts (job slicing, fact cache)                                       │   │
│  │  - Provision-VM (callback template)                                             │   │
│  │  - Dry-Run-Deploy (check mode)                                                 │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Projects: Lab-Project (lab-playbooks) | Inventories: Lab-Static-Inventory            │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ SSH (22)
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │ web1.lab.local│  │ app1.lab.local│  │ db1.lab.local  │
            │ [webservers]  │  │ [appservers]  │  │ [databases]    │
            └───────────────┘  └───────────────┘  └───────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` (or `Lab-Manual-Project`) with playbooks in `lab-playbooks` |
| Inventory | `Lab-Static-Inventory` with groups (webservers, appservers, databases) and hosts |
| Credential | Machine credential (SSH key) — e.g., `SSH-Key-Lab` |
| Managed Nodes | 2–4 Linux VMs reachable via SSH |

---

## Verify AAP 2.4 Before Starting

1. Log in to the Automation Controller.
2. Click **?** (help) or your **user profile** → **About** — version should be 4.4.x (AAP 2.4).
3. Confirm **Resources → Templates** shows job and workflow templates.

---

## Part 1 — Theory: Understanding Job Templates (45 minutes)

### Section 6.1 — Understanding Job Templates

#### What Is a Job Template?

A **job template** is a reusable definition that combines:

| Component | Purpose |
|-----------|---------|
| **Playbook** | The Ansible playbook file to run (from a Project) |
| **Inventory** | The set of hosts to target |
| **Credential** | How to authenticate (SSH, cloud, vault) |
| **Options** | Limit, tags, extra variables, verbosity, timeout, etc. |

Instead of running `ansible-playbook -i inventory playbook.yml` manually, users click **Launch** in the UI. AAP records who ran what, when, and with what parameters.

#### Components of a Job Template

| Field | Description |
|-------|-------------|
| **Name** | Display name (e.g., `Deploy-WebApp-Prod`) |
| **Job Type** | Run, Check, or Scan |
| **Inventory** | Required — which hosts to target |
| **Project** | Source of playbooks |
| **Playbook** | Path to the playbook file (e.g., `deploy.yml`) |
| **Credentials** | Machine, vault, cloud — can add multiple |
| **Execution Environment** | Container image for Ansible (default or custom) |
| **Forks** | Max parallel host connections (default: 5) |
| **Limit** | Restrict to a subset of inventory (e.g., `webservers`) |
| **Tags** | Run only tasks with these tags |
| **Skip Tags** | Skip tasks with these tags |
| **Extra Variables** | Key-value overrides passed to the playbook |
| **Survey** | Prompts shown at launch to collect user input |
| **Job Slicing** | Split inventory across multiple parallel jobs |
| **Fact Cache** | Cache gathered facts for reuse |
| **Timeout** | Kill job after N seconds |
| **Verbosity** | 0–4 for debug output |

#### Difference Between Job Templates and Workflow Templates

| Aspect | Job Template | Workflow Job Template |
|--------|--------------|------------------------|
| **Runs** | One playbook | Multiple job templates or workflows in a graph |
| **Use case** | Single automation task | Multi-step orchestration (provision → configure → deploy) |
| **Convergence** | N/A | Can wait for multiple nodes before proceeding |
| **Approval nodes** | N/A | Can pause for human approval |
| **Inventory override** | Per template | Can override per node in workflow |

#### When to Use Each Type

| Scenario | Use |
|----------|-----|
| Run a playbook on hosts | **Job Template** |
| Provision VM → configure → deploy app (sequence) | **Workflow Template** |
| Run playbook A and B in parallel, then C | **Workflow Template** |
| Pause for approval before production deploy | **Workflow Template** with approval node |
| Self-service: user picks environment and app | **Job Template** with **Survey** |

---

### Section 6.2 — Creating Basic Job Templates

#### Selecting Playbooks for Templates

- Playbooks come from a **Project**. The project must be synced (green check).
- **Playbook** field shows files in the project root (e.g., `ping.yml`, `deploy.yml`).
- Use subdirectories: `playbooks/deploy.yml` if your project has that structure.

#### Choosing Inventories and Credentials

| Resource | Rule |
|----------|------|
| **Inventory** | Required. Must have at least one host (or use limit to target specific groups). |
| **Machine Credential** | Required for SSH/WinRM to managed hosts. |
| **Vault Credential** | Add if playbook uses `ansible-vault` encrypted vars. |
| **Source Control** | Project sync uses it; job execution does not need it. |

#### Understanding Prompts and Surveys

**Prompt on Launch** — When enabled, the user is asked at launch time for:

| Prompt | Purpose |
|--------|---------|
| **Inventory** | Override which inventory to use |
| **Credentials** | Override credentials |
| **Limit** | Restrict to hosts (e.g., `web1.lab.local` or `webservers`) |
| **Tags** | Run only tasks with these tags |
| **Skip Tags** | Skip tasks with these tags |
| **Extra Variables** | Key-value overrides |
| **Job Slice Count** | Number of slices (if job slicing enabled) |
| **Labels** | Attach labels for filtering |

**Surveys** — Custom questions (text, number, dropdown, multi-select) that become **extra variables** passed to the playbook. Use for self-service: e.g., "Which environment?" or "How many instances?"

#### Extra Variables and Their Uses

| Use Case | Example |
|----------|---------|
| Override defaults | `app_version: 2.1.0` |
| Environment selection | `env: prod` |
| Feature flags | `enable_debug: true` |
| Survey input | User selects "prod" → `env: prod` passed as extra var |

Extra variables use **YAML or JSON** syntax. Host/group vars take precedence over extra vars unless you use `-e` at CLI; in AAP, extra vars from the template or survey override inventory vars for that run.

#### Privilege Escalation Settings

| Option | Purpose |
|--------|---------|
| **Privilege Escalation** | Enable to use `become` (e.g., `sudo`) |
| **Privilege Escalation Method** | `sudo`, `su`, `pbrun`, etc. |
| **Privilege Escalation Username** | User to become (e.g., `root`) |
| **Credential** | Machine credential can include "Privilege Escalation" password |

---

### Section 6.3 — Advanced Template Features

#### Surveys: Collecting Runtime Input from Users

A **survey** is a form shown when the user clicks **Launch**. Each question maps to an extra variable.

| Survey Type | Use Case |
|-------------|----------|
| **Text** | Free-form input (e.g., instance name) |
| **Textarea** | Multi-line (e.g., config snippet) |
| **Password** | Secret (masked) |
| **Integer** | Number (e.g., port, count) |
| **Float** | Decimal |
| **Multiple choice** | Single selection from list |
| **Multiple select** | Multiple selections |

Survey answers are passed as `extra_vars` to the playbook. Use `{{ variable_name }}` in playbooks.

#### Limit Patterns: Targeting Specific Hosts

The **Limit** field restricts execution to a subset of the inventory:

| Limit | Effect |
|-------|--------|
| `web1.lab.local` | Only that host |
| `webservers` | All hosts in group `webservers` |
| `webservers:!db1` | webservers except db1 |
| `prod:&webservers` | Hosts in both prod and webservers |

Enable **Prompt on Launch** for Limit to let users choose at runtime.

#### Tags: Running Specific Parts of Playbooks

Ansible tasks can have `tags:`. Use **Tags** or **Skip Tags** to run only certain tasks:

| Tags Field | Effect |
|------------|--------|
| `install` | Run only tasks tagged `install` |
| `install,config` | Run tasks with `install` OR `config` |
| (empty) | Run all tasks |

| Skip Tags | Effect |
|-----------|--------|
| `never` | Skip tasks tagged `never` (e.g., dangerous tasks) |
| `backup` | Skip backup tasks for a quick run |

#### Job Slicing: Parallel Execution for Large Inventories

**Job slicing** splits the inventory into N chunks and runs N parallel jobs (one per chunk). Use for large inventories (e.g., 1000+ hosts) to distribute work across controller nodes.

| Setting | Description |
|---------|-------------|
| **Job Slice Count** | Number of slices (e.g., 4 → 4 parallel jobs) |
| **Consideration** | Slice count ≤ number of controller nodes for best parallelism |
| **Result** | A workflow job is created; each slice runs a subset of hosts |

#### Concurrent Jobs and Job Limits

| Setting | Purpose |
|---------|---------|
| **Allow Simultaneous Jobs** | Multiple users can launch the same template at once (e.g., dev and prod in parallel). Enable in **Options** tab. |
| **Job concurrency** | System-wide or org-level limit on concurrent jobs (Settings → Jobs) |

#### Enabling Fact Cache

When **Use Fact Cache** is enabled:

- At the end of a job, AAP stores gathered facts in the database.
- Subsequent jobs can use `gather_facts: false` and read from cache via the fact cache plugin.
- Reduces redundant `setup` module runs across many jobs.

---

### Section 6.4 — Template Types and Options

#### Run Templates

- **Job Type: Run** — Normal playbook execution. Default.

#### Check Mode (Dry-Run) Templates

- **Job Type: Check** — Runs with `--check` (dry-run). No changes made; shows what *would* change.
- Use for: validating before real deploy, compliance checks.

#### Callback Templates for Provisioning

- **Job Type: Run** with **Enable Provisioning Callbacks**.
- New hosts can **request** configuration by calling a unique URL (with a host config key).
- Use for: auto-scaling, PXE-booted VMs that self-register and get configured.

#### Timeout Settings

- **Job Timeout** — Kill the job after N seconds (0 = no limit).
- Prevents runaway jobs.

#### Verbosity Levels for Troubleshooting

| Level | Flag | Use |
|-------|------|-----|
| 0 | (default) | Normal output |
| 1 | `-v` | Verbose |
| 2 | `-vv` | More verbose |
| 3 | `-vvv` | Include connection debugging |
| 4 | `-vvvv` | Full debug |

Enable **Prompt on Launch** for Verbosity to let users choose when debugging.

---

## Part 2 — Hands-On: Prepare Playbooks (20 minutes)

### Step 1 — Create Playbooks with Tags and Variables

**Purpose:** Playbooks must support tags and variables for the advanced template exercises.

**Detailed steps:**

1. **SSH to the AAP server** and ensure the project directory exists:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-playbooks
```

2. **Create a multi-task playbook with tags** (`deploy_app.yml`):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/deploy_app.yml > /dev/null << 'EOF'
---
- name: Deploy application (tagged tasks)
  hosts: all
  gather_facts: true
  vars:
    app_version: "1.0"
    app_env: "dev"
  tasks:
    - name: Create app directory
      ansible.builtin.file:
        path: "/opt/app"
        state: directory
        mode: "0755"
      tags: install

    - name: Install package (simulated)
      ansible.builtin.debug:
        msg: "Installing app version {{ app_version }} for env {{ app_env }}"
      tags: install

    - name: Configure application
      ansible.builtin.copy:
        dest: "/opt/app/config.yml"
        content: |
          version: {{ app_version }}
          env: {{ app_env }}
        mode: "0644"
      tags: config

    - name: Start service (simulated)
      ansible.builtin.debug:
        msg: "Starting app service"
      tags: service

    - name: Verify deployment
      ansible.builtin.debug:
        msg: "Deployment complete for {{ inventory_hostname }}"
      tags: always
EOF
```

3. **Create a fact-gathering playbook for fact cache** (`gather_and_report.yml`):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/gather_and_report.yml > /dev/null << 'EOF'
---
- name: Gather facts and report
  hosts: all
  gather_facts: true
  tasks:
    - name: Display host info
      ansible.builtin.debug:
        msg: "Host {{ inventory_hostname }} - OS: {{ ansible_os_family }} - Mem: {{ ansible_memtotal_mb }}MB"
EOF
```

4. **Ensure ping.yml exists** (from Lab 02):

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/ping.yml > /dev/null << 'EOF'
---
- name: Ping all hosts
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping
      ansible.builtin.ping:
EOF
```

5. **Set ownership:**

```bash
sudo chown -R awx:awx /var/lib/awx/projects/lab-playbooks
```

6. **Sync the project in AAP:** **Resources → Projects** → open `Lab-Project` (or `Lab-Manual-Project`) → click **Sync** (refresh icon). Verify `ping.yml`, `deploy_app.yml`, and `gather_and_report.yml` appear in the playbooks list.

---

## Part 3 — Hands-On: Basic Job Template (15 minutes)

### Step 2 — Create a Basic Job Template

**Purpose:** Build a job template with playbook, inventory, and credential.

**Detailed steps:**

1. **Navigate:** **Resources → Templates**
2. **Add:** Click **Add** → **Add job template**
3. **Fill in:**
   - **Name:** `Ping-Lab`
   - **Job Type:** `Run`
   - **Inventory:** Select `Lab-Static-Inventory` (or your inventory)
   - **Project:** Select `Lab-Project` (or `Lab-Manual-Project`)
   - **Playbook:** Select `ping.yml`
   - **Credentials:** Click the search icon, select your Machine credential (e.g., `SSH-Key-Lab`)
   - **Options:** Leave defaults
4. **Save**
5. **Launch** the job — click the rocket icon. Verify it completes successfully.

---

## Part 4 — Hands-On: Surveys and Extra Variables (25 minutes)

### Step 3 — Add a Survey to a Job Template

**Purpose:** Collect user input at launch and pass it as extra variables to the playbook.

**Detailed steps:**

1. **Create a new job template:** **Resources → Templates → Add → Add job template**
   - **Name:** `Deploy-App-Survey`
   - **Job Type:** `Run`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `deploy_app.yml`
   - **Credentials:** Your Machine credential
   - **Save**

2. **Add a survey:** Open `Deploy-App-Survey` → **Survey** tab → check **Enable Survey**
3. **Add survey questions:** Click **Add** for each:

   **Question 1:**
   - **Question:** `Application version`
   - **Description:** `Version to deploy (e.g., 1.0, 2.1)`
   - **Answer Variable Name:** `app_version`
   - **Answer Type:** `Text`
   - **Default:** `1.0`
   - **Required:** Yes

   **Question 2:**
   - **Question:** `Environment`
   - **Description:** `Target environment`
   - **Answer Variable Name:** `app_env`
   - **Answer Type:** `Multiple choice (single select)`
   - **Choices:** `dev` (one per line), then `staging`, then `prod`
   - **Default:** `dev`
   - **Required:** Yes

4. **Save** the survey.
5. **Launch** the job — you should see the survey form. Enter `2.0` for version, select `prod` for environment. Click **Next** → **Launch**.
6. **Verify** in the job output that the debug messages show `app_version: 2.0` and `app_env: prod`.

---

## Part 5 — Hands-On: Limit and Tags (25 minutes)

### Step 4 — Use Limit to Target Specific Hosts

**Purpose:** Restrict execution to a subset of the inventory.

**Detailed steps:**

1. **Edit** `Deploy-App-Survey` (or create a copy).
2. **Options** tab → enable **Prompt on Launch** for **Limit**.
3. **Save**
4. **Launch** — you will be prompted for **Limit**. Enter `webservers` (or a single host like `web1.lab.local`). Launch.
5. **Verify** only hosts matching the limit were targeted.

### Step 5 — Use Tags to Run Specific Tasks

**Purpose:** Run only a subset of tasks (e.g., install but not config).

**Detailed steps:**

1. **Edit** `Deploy-App-Survey` (or create `Deploy-App-Tags`).
2. **Options** tab → enable **Prompt on Launch** for **Tags**.
3. **Save**
4. **Launch** — when prompted for **Tags**, enter `install`. Launch.
5. **Verify** in the output: only "Create app directory" and "Install package" ran; "Configure application" and "Start service" were skipped.
6. **Launch again** with **Tags** = `config` — only the config task runs.
7. **Launch** with **Tags** empty — all tasks run.

---

## Part 6 — Hands-On: Job Slicing and Fact Cache (25 minutes)

### Step 6 — Configure Job Slicing

**Purpose:** Split a large inventory across multiple parallel jobs.

**Detailed steps:**

1. **Create a new job template:** **Resources → Templates → Add → Add job template**
   - **Name:** `Ping-Sliced`
   - **Job Type:** `Run`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `ping.yml`
   - **Credentials:** Machine credential
   - **Job Slice Count:** `2` (or 3 if you have 3+ hosts)
   - **Save**

2. **Launch** the job.
3. **Observe:** The job appears as a **Workflow** in the Jobs list. Click it — you will see 2 slice jobs running in parallel. Each targets a subset of hosts.
4. **Verify** all hosts received the ping across the slices.

> **Note:** With 2 hosts, each slice may get 1 host. With 4 hosts, each slice gets 2. Job slicing is most useful with 10+ hosts.

### Step 7 — Enable Fact Cache

**Purpose:** Cache gathered facts for reuse by other jobs.

**Detailed steps:**

1. **Create a job template:** **Resources → Templates → Add → Add job template**
   - **Name:** `Gather-Facts-Cache`
   - **Job Type:** `Run`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `gather_and_report.yml`
   - **Credentials:** Machine credential
   - **Options:** Check **Use Fact Cache**
   - **Save**

2. **Launch** the job. It gathers facts and, at the end, AAP stores them.
3. **Verify:** **Resources → Hosts** → click a host → **Facts** tab. You should see cached facts (or run the job again and check after completion).

> **Note:** Fact cache is written at job end. A follow-up job can use `gather_facts: false` and the fact cache plugin to read cached facts — useful for large inventories to avoid repeated setup.

---

## Part 7 — Hands-On: Check Mode and Callback (20 minutes)

### Step 8 — Create a Check Mode (Dry-Run) Template

**Purpose:** Run playbook in check mode to see what would change without making changes.

**Detailed steps:**

1. **Create a job template:** **Resources → Templates → Add → Add job template**
   - **Name:** `Deploy-App-Dry-Run`
   - **Job Type:** `Check` (select from dropdown — not Run)
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `deploy_app.yml`
   - **Credentials:** Machine credential
   - **Save**

2. **Launch** the job.
3. **Observe:** Tasks show "would create" or "would change" instead of actually creating files. No changes are made on the hosts.

### Step 9 — Understand Callback Templates (Conceptual)

**Purpose:** Callback templates allow new hosts to request configuration by calling a URL.

**How it works:**

1. Create a job template with **Enable Provisioning Callbacks** checked.
2. AAP generates a unique **Provisioning Callback URL** and **Host Config Key**.
3. A new host (e.g., from PXE boot or cloud init) runs a script that calls the URL with the key.
4. AAP adds the host to the inventory and launches the job.

**Lab note:** Full callback setup requires a host that can reach AAP and run the callback script. For this lab, we configure the option only:

1. **Edit** `Ping-Lab` (or create `Provision-Callback-Demo`).
2. **Options** tab → check **Enable Provisioning Callbacks**.
3. **Save**
4. **View** the **Provisioning Callback URL** and **Host Config Key** in the template details. In production, you would embed these in a kickstart or cloud-init script.

---

## Part 8 — Hands-On: Timeout and Verbosity (15 minutes)

### Step 10 — Set Job Timeout, Verbosity, and Concurrent Jobs

**Purpose:** Prevent runaway jobs, enable debug output, and allow multiple simultaneous launches.

**Detailed steps:**

1. **Edit** `Ping-Lab` (or any template).
2. **Job timeout:** Set **Job Timeout** to `120` (seconds). A job running longer than 2 minutes will be killed.
3. **Verbosity:** Enable **Prompt on Launch** for **Verbosity**. Save.
4. **Allow simultaneous jobs:** In the **Options** tab, check **Enable Concurrent Jobs**. This allows multiple users (or the same user) to launch this template at the same time without waiting for the previous job to finish.
5. **Launch** — when prompted, select **Verbosity 2 (more verbose)**. Launch.
6. **Observe** the job output — you will see more detailed Ansible output (`-vv`).

---

## Part 9 — Real-Time Use Case Discussion (20 minutes)

### Case Study: Cloud Services Company — Self-Service Infrastructure Provisioning

**Background:**

A cloud services company provides development teams with self-service infrastructure. Developers need to provision VMs, install apps, and configure environments without waiting for the platform team.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Manual provisioning | Slow; platform team bottleneck |
| Inconsistent configs | Each dev does it differently |
| Security and compliance | Uncontrolled changes; audit gaps |
| Scale | 50+ dev teams; cannot scale manually |

**Solution: Job Templates with Surveys**

| Component | Implementation |
|-----------|----------------|
| **Survey: Environment** | Dropdown: `dev`, `staging`, `prod` — passed as `env` extra var |
| **Survey: App Type** | Dropdown: `webapp`, `api`, `worker` — selects different playbooks or roles |
| **Survey: Instance Count** | Integer: 1–10 — used in dynamic inventory or loop |
| **Survey: Region** | Dropdown: `us-east-1`, `eu-west-1` — for cloud provisioning |
| **Limit** | Prompt on launch — dev can restrict to a test host first |
| **RBAC** | Only "Dev Team" has Execute on the template; no Edit |
| **Approval** | For `prod`, use a workflow with an approval node — platform team approves |

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Provisioning time | 2–5 days | 10 minutes |
| Platform team tickets | 200/week | 20/week |
| Config consistency | Low | High (playbook-driven) |
| Audit trail | Manual | Full (who, when, what in AAP) |

**Key takeaway:** Surveys turn complex playbooks into simple forms. Users provide only what they need to choose; the playbook handles the rest. Combined with RBAC and workflows, this enables secure self-service at scale.

---

## Troubleshooting

### Job Template Fails to Launch

| Symptom | Check |
|---------|-------|
| "Inventory required" | Select an inventory; ensure it has hosts |
| "Credential required" | Add Machine credential |
| "Project not synced" | Sync the project; fix SCM URL/credential if Git |
| "Playbook not found" | Verify playbook path; check project sync |

### Survey Variables Not Passed

| Symptom | Check |
|---------|-------|
| Variable undefined in playbook | Ensure Answer Variable Name matches `{{ var_name }}` in playbook |
| Survey not shown | Enable Survey; add at least one question; Save |
| Wrong type | Use correct type (e.g., integer for numbers) |

### Job Slicing Issues

| Symptom | Check |
|---------|-------|
| Slices fail with "no hosts" | Limit may exclude hosts from some slices; avoid limit with slicing or ensure limit spreads across slices |
| Too many slices | Keep slice count ≤ controller nodes; avoid thousands |
| Workflow not created | Job slice count must be > 1 |

### Check Mode Shows "Would" but Job Type Wrong

- Ensure **Job Type** is `Check`, not `Run`.

### Fact Cache Empty

- **Use Fact Cache** must be enabled on the template.
- Facts are written at job *end*; run a job that gathers facts first.
- Verify in **Resources → Hosts → [host] → Facts** tab.

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Job template | Playbook + inventory + credential + options |
| Workflow template | Multi-step orchestration; different use case |
| Survey | Runtime prompts → extra variables |
| Limit | Restrict to subset of inventory |
| Tags / Skip Tags | Run only specific tasks |
| Job slicing | Split inventory across parallel jobs |
| Fact cache | Store facts for reuse |
| Check mode | Dry-run; no changes |
| Callback template | Hosts request config via URL |
| Timeout | Kill long-running jobs |
| Verbosity | Debug output levels |

---

## Lab Completion Checklist

- [ ] Playbooks created with tags (`deploy_app.yml`, `gather_and_report.yml`)
- [ ] Basic job template created and launched successfully
- [ ] Survey added with at least 2 questions; variables passed to playbook
- [ ] Limit used (prompt on launch) to target specific hosts
- [ ] Tags used to run only `install` or `config` tasks
- [ ] Job slicing configured and tested (workflow with slice jobs)
- [ ] Fact cache enabled; facts visible on hosts
- [ ] Check mode template created and run (dry-run)
- [ ] Provisioning callback option reviewed
- [ ] Timeout and verbosity configured
- [ ] Can explain the cloud services self-service use case

---

## Quick Reference — Job Template Prompts on Launch (AAP 2.4)

| Prompt | Purpose |
|--------|---------|
| Inventory | Override inventory at launch |
| Credentials | Override credentials |
| Limit | Restrict to hosts (e.g., `webservers`) |
| Tags | Run only tagged tasks |
| Skip Tags | Skip tagged tasks |
| Extra Variables | Key-value overrides |
| Job Slice Count | Number of slices |
| Verbosity | 0–4 debug level |
| Labels | Attach labels |

---

## Quick Reference — Survey Answer Types

| Type | Use Case |
|------|----------|
| Text | Free-form string |
| Textarea | Multi-line |
| Password | Secret (masked) |
| Integer | Whole number |
| Float | Decimal |
| Multiple choice (single) | One of N options |
| Multiple select | Many of N options |

---

*End of Lab 07*
