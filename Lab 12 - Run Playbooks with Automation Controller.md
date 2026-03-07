# Lab 12 — Run Playbooks with Automation Controller

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 3 hours (Theory: 25% / Hands-on: 75%)                                      |
| **Outcome**  | Launch jobs via UI, surveys, and API; create scheduled jobs (one-time and recurring); trigger jobs via webhooks and provisioning callbacks; monitor execution in real-time; troubleshoot failures; work with job facts, artifacts, and fact caching |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller) — Classic and New UI |
| **Prerequisite** | Labs 02, 04, 05, 07 completed — AAP 2.4 installed, inventory with hosts, project with playbooks, job templates configured. |

---

## AAP 2.4 UI Navigation Reference (Classic and New UI)

Before starting, familiarize yourself with the **Automation Controller** UI structure in AAP 2.4:

| Menu | Classic UI Location | New UI (Preview) Location |
|------|----------------------|---------------------------|
| **Resources** | Left navigation panel | Left sidebar — Templates, Inventories, Projects, Credentials |
| **Templates** | Resources → Templates | Resources → Templates (Job / Workflow) |
| **Jobs** | Resources → Jobs (or Dashboard) | Jobs list — accessible from Templates or top-level |
| **Schedules** | Template → Schedules tab, or Administration → Schedules | Template details → Schedules; or Schedules under Resources |
| **Administration** | Left panel — Notifications, Instances, etc. | Settings gear or Administration section |
| **Settings** | Left panel — System, Jobs, License | System configuration, Job settings |

> **Note:** AAP 2.4 offers a **Preview of New User Interface** (Settings → System → Miscellaneous). The new UI has a refreshed layout with improved navigation. This lab covers both; adjust paths if your environment uses the new UI preview. Key concepts (Launch, Schedules, Jobs, API) remain the same.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER / API CLIENT                           │
│                         https://aap1.lab.local                                        │
│                         + curl / Postman (for API exercises)                          │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
                                       │ HTTPS (443) / REST API
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                       │
│                                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │  JOB LAUNCH       │  │  SCHEDULES       │  │  API / WEBHOOK   │  │  JOB OUTPUT │ │
│  │  - Manual (UI)    │  │  - One-time      │  │  - REST API      │  │  - Real-time│ │
│  │  - Survey input   │  │  - Recurring     │  │  - Webhook URL   │  │  - Facts    │ │
│  │  - Extra vars     │  │  - Cron syntax   │  │  - Callback URL  │  │  - Artifacts│ │
│  │  - Limit / Tags   │  │  - Timezone      │  │                  │  │  - History  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  └─────────────┘ │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
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
| Job Templates | At least `Ping-Lab`; `Deploy-App-Survey` (from Lab 07) recommended |
| Credential | Machine credential (SSH key) — e.g., `SSH-Key-Lab` |
| Managed Nodes | 2–4 Linux VMs reachable via SSH |
| API Access | User with Execute permission on job templates (for API exercises) |

---

## Verify AAP 2.4 Before Starting

1. Log in to the Automation Controller: `https://aap1.lab.local`
2. Click **?** (help) or your **user profile** → **About** — version should be 4.4.x (AAP 2.4)
3. Confirm **Resources → Templates** shows job templates
4. Ensure at least one job template (e.g., `Ping-Lab`) exists and can be launched successfully

---

## Part 1 — Theory: Launching Jobs, Scheduling, API, Monitoring, and Artifacts (45 minutes)

### Section 11.1 — Launching Jobs in AAP

#### Manual Job Launches from the UI

The primary way to run automation in AAP is to **launch** a job template from the web UI:

| Action | Location |
|--------|----------|
| **Launch** | Template list — click the **Launch** (rocket) icon next to a job template |
| **Launch with prompts** | If "Prompt on Launch" is enabled, a form appears before execution |
| **Job output** | **Resources → Jobs** (or **Jobs** in new UI) — click a job to view output |

#### Providing Runtime Input via Surveys

**Surveys** are custom questions shown at launch time. Answers are passed as **extra variables** to the playbook.

| Survey Type | Use Case |
|-------------|----------|
| Text | Free-form input (e.g., version number, hostname) |
| Multiple choice | Single selection (e.g., environment: dev/staging/prod) |
| Password | Secret input (masked) |
| Integer | Numeric value (e.g., instance count) |

#### Using Extra Variables at Launch

- **Extra Variables** override or supplement playbook variables
- Can be set in the template (default) or provided at launch (if "Prompt on Launch" enabled)
- Format: YAML or JSON (e.g., `app_version: "2.0"` or `{"env": "prod"}`)

#### Selecting Limits and Tags at Runtime

| Prompt | Purpose |
|--------|---------|
| **Limit** | Restrict execution to a subset of inventory (e.g., `webservers`, `web1.lab.local`) |
| **Tags** | Run only tasks with these tags (e.g., `install`, `config`) |
| **Skip Tags** | Skip tasks with these tags (e.g., `backup`) |

#### Different Launch Methods Explained

| Method | Description |
|--------|-------------|
| **UI — Launch** | Click Launch; optional survey/prompts; job runs immediately |
| **UI — Schedule** | Job runs at a scheduled time (one-time or recurring) |
| **API — POST** | `POST /api/v2/job_templates/<id>/launch/` — programmatic launch |
| **Webhook** | External system sends HTTP POST to webhook URL — triggers launch |
| **Provisioning Callback** | New host calls callback URL with host config key — adds host and launches job |

---

### Section 11.2 — Job Scheduling

#### Why Schedule Jobs?

| Use Case | Example |
|----------|---------|
| **Maintenance windows** | Nightly security patching at 2:00 AM |
| **Compliance** | Weekly configuration drift scan |
| **Backups** | Daily backup at midnight |
| **Reporting** | Monthly inventory report on the 1st |
| **Resource optimization** | Run heavy jobs during off-peak hours |

#### Creating One-Time Schedules

- **One-time** — Job runs once at a specified date and time
- Use for: ad-hoc deferred runs, maintenance events, one-off deployments

#### Creating Recurring Schedules

- **Recurring** — Job runs on a repeating pattern
- Options: Every X minutes, hourly, daily, weekly, monthly, or custom cron

#### Understanding Cron-Like Syntax

AAP uses **cron**-style fields for recurring schedules:

| Field | Values | Example |
|-------|--------|---------|
| Minute | 0–59 | `0` = at minute 0 |
| Hour | 0–23 | `2` = 2:00 AM |
| Day of month | 1–31 | `*` = every day |
| Month | 1–12 | `*` = every month |
| Day of week | 0–6 (Sun–Sat) | `0` = Sunday |

**Examples:**

| Schedule | Cron Expression | Meaning |
|----------|-----------------|---------|
| Daily at 2:00 AM | `0 2 * * *` | Every day at 2 AM |
| Every Monday at 3:00 AM | `0 3 * * 1` | Weekly maintenance |
| Every 15 minutes | `*/15 * * * *` | High-frequency checks |
| 1st of month at midnight | `0 0 1 * *` | Monthly report |

#### Timezone Considerations

- Schedules use the **Controller timezone** (Settings → System → Localization)
- Ensure timezone matches your maintenance window expectations
- Use UTC for distributed teams to avoid confusion

#### Managing Schedule Conflicts

| Scenario | Behavior |
|----------|----------|
| **Same template, overlapping runs** | By default, a new job queues if the previous is still running |
| **Concurrent jobs** | Enable "Allow Simultaneous Jobs" on the template to run parallel instances |
| **Resource contention** | Use instance groups to isolate scheduled jobs from ad-hoc jobs |

---

### Section 11.3 — API and Webhook Triggers

#### Introduction to AAP REST API

AAP exposes a **REST API** for all operations. Use it for:

- CI/CD pipeline integration (Jenkins, GitLab CI, GitHub Actions)
- Custom dashboards and reporting
- Automated job launches from external systems
- Infrastructure-as-code (Terraform, etc.)

**Base URL:** `https://aap1.lab.local/api/v2/`

**Authentication:** Token (recommended) or Basic Auth

#### Triggering Jobs via API (Basic Examples)

**Launch a job template:**

```bash
curl -k -X POST \
  "https://aap1.lab.local/api/v2/job_templates/<TEMPLATE_ID>/launch/" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"env": "prod"}, "limit": "webservers"}'
```

**Get job status:**

```bash
curl -k -s "https://aap1.lab.local/api/v2/jobs/<JOB_ID>/" \
  -H "Authorization: Bearer <YOUR_TOKEN}"
```

#### Webhook Setup for External Triggers

- Each job template has a **Webhook URL** (or can be configured)
- External systems (GitHub, GitLab, Jenkins) send HTTP POST to trigger a launch
- **Webhook credential** (optional) — shared secret for verification
- **Webhook key** — passed as query parameter for authentication

#### Provisioning Callbacks for New Systems

- **Provisioning Callback** — allows a **new host** to request configuration
- Use case: Auto-scaling — new VM boots, runs a script that calls the callback URL
- AAP adds the host to inventory and launches the job
- **Host Config Key** — unique per template; passed in the callback request

---

### Section 11.4 — Monitoring Job Execution

#### Real-Time Job Output and Logs

- **Jobs** list shows all job runs (pending, running, successful, failed)
- Click a job to view **output** — updates in real time as the job runs
- Output is searchable and can be downloaded

#### Understanding Job Status

| Status | Meaning |
|--------|---------|
| **Pending** | Job is queued, waiting for an execution node |
| **Running** | Job is executing on an execution node |
| **Successful** | All tasks completed without failure |
| **Failed** | One or more tasks failed |
| **Canceled** | Job was manually canceled |
| **Error** | Job could not start (e.g., inventory empty, credential missing) |

#### Canceling Running Jobs

- Open the running job → click **Cancel** (or **Stop**)
- Ansible receives the cancel signal and stops after the current task
- Job status becomes **Canceled**

#### Job History and Retention

- **Jobs** list shows historical runs
- **Retention:** Configured in Settings → Jobs → Job Event Retention
- Old job data can be purged to manage database size

#### Troubleshooting Failed Jobs

| Step | Action |
|------|--------|
| 1 | Open the failed job; scroll to the first red (failed) task |
| 2 | Read the error message — often indicates the cause |
| 3 | Check **Standard Error** and **Standard Output** for the failing task |
| 4 | Verify: inventory hosts reachable, credential correct, playbook syntax |
| 5 | Re-run with **Verbosity** (Prompt on Launch) for more detail |
| 6 | Use **Check Mode** template to simulate without making changes |

---

### Section 11.5 — Job Results and Artifacts

#### Understanding Job Facts

- **Facts** — data gathered by the `setup` module (or `gather_facts: true`)
- Stored per host: `ansible_hostname`, `ansible_os_family`, `ansible_memtotal_mb`, etc.
- View: **Resources → Hosts** → select host → **Facts** tab

#### Setting and Using Artifacts

- **Artifacts** — custom data stored by a playbook for use by downstream jobs
- Use `ansible.builtin.set_stats` with `per_host: false` to add to workflow artifacts
- In workflows, downstream job template nodes receive artifacts as extra variables

#### Fact Caching Across Runs

- **Use Fact Cache** — enabled on job template
- At job end, AAP stores gathered facts in the database
- Subsequent jobs can use `gather_facts: false` and read from cache (via fact cache plugin)
- Reduces redundant `setup` runs on large inventories

#### Sharing Data Between Jobs in Workflows

| Method | Use Case |
|--------|----------|
| **set_stats** | Store `deploy_version`, `db_host` in workflow artifacts |
| **Extra vars** | Pass from workflow survey to job nodes |
| **Fact cache** | Reuse facts from a "gather facts" job in later jobs |

---

## Part 2 — Hands-On: Launching Jobs (30 minutes)

### Step 1 — Manual Job Launch from the UI

**Purpose:** Launch a job template manually and observe the job lifecycle.

**Detailed steps:**

1. Log in to AAP: `https://aap1.lab.local`
2. Navigate to **Resources → Templates** (or **Templates** in new UI)
3. Locate `Ping-Lab` (or your basic job template)
4. Click the **Launch** (rocket) icon
5. If no prompts appear, the job launches immediately
6. You are redirected to the **Job** output view
7. Watch the output update in real time — each task appears as it runs
8. When complete, the status shows **Successful** (green)

**Verify:** Job status is **Successful**; output shows `pong` for each host.

---

### Step 2 — Launch with Survey Input

**Purpose:** Use a job template with a survey to provide runtime input.

**Detailed steps:**

1. If you have `Deploy-App-Survey` from Lab 07, open it. Otherwise, create a job template with a survey:
   - **Resources → Templates → Add → Add job template**
   - **Name:** `Deploy-App-Survey`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `deploy_app.yml` (create if missing — see Lab 07)
   - **Credential:** Machine credential
   - **Survey tab:** Enable survey; add questions: `app_version` (Text), `app_env` (Multiple choice: dev, staging, prod)
   - **Save**

2. Click **Launch**
3. The **Survey** form appears — enter `2.0` for version, select `prod` for environment
4. Click **Next** (or **Launch**)
5. Job runs with `app_version: "2.0"` and `app_env: "prod"` as extra variables

**Verify:** Job output shows the survey values in debug messages.

---

### Step 3 — Launch with Limit and Tags at Runtime

**Purpose:** Use "Prompt on Launch" for Limit and Tags to target specific hosts and tasks.

**Detailed steps:**

1. Edit `Deploy-App-Survey` (or `Ping-Lab`)
2. Open the **Options** (or **Prompt** / **Survey**) section
3. Enable **Prompt on Launch** for:
   - **Limit**
   - **Tags**
4. **Save**
5. Click **Launch**
6. When prompted:
   - **Limit:** Enter `webservers` (or a single host like `web1.lab.local`)
   - **Tags:** Enter `install` (to run only install-tagged tasks)
7. Launch the job

**Verify:** Only hosts matching the limit were targeted; only tasks with `install` tag ran.

---

### Step 4 — Launch with Extra Variables

**Purpose:** Pass extra variables at launch (when "Prompt on Launch" for Extra Variables is enabled).

**Detailed steps:**

1. Edit a job template that uses variables (e.g., `Deploy-App-Survey`)
2. Enable **Prompt on Launch** for **Extra Variables**
3. **Save**
4. Click **Launch**
5. In the **Extra Variables** field, enter (YAML format):

```yaml
app_version: "3.0"
app_env: "staging"
```

6. Launch the job

**Verify:** Playbook receives and uses the extra variables.

---

## Part 3 — Hands-On: Job Scheduling (35 minutes)

### Step 5 — Create a One-Time Schedule

**Purpose:** Schedule a job to run once at a future date and time.

**Detailed steps:**

1. Open `Ping-Lab` (or any job template)
2. Click the **Schedules** tab (or **Add schedule** in new UI)
3. Click **Add** (or **Create schedule**)
4. Fill in:
   - **Name:** `Ping-One-Time-Lab`
   - **Schedule type:** **One-time**
   - **Start date/time:** Select a time 5–10 minutes from now (for testing)
   - **Timezone:** Use Controller default or your timezone
5. **Save**
6. Wait for the scheduled time — the job will launch automatically
7. Check **Resources → Jobs** — a new job should appear and run

**Verify:** Job runs at the scheduled time without manual launch.

---

### Step 6 — Create a Recurring Schedule (Cron)

**Purpose:** Create a recurring schedule using cron syntax (e.g., daily at 2:00 AM).

**Detailed steps:**

1. Open `Ping-Lab` (or a template suitable for recurring runs)
2. **Schedules** tab → **Add**
3. Fill in:
   - **Name:** `Ping-Daily-Lab`
   - **Schedule type:** **Recurring**
   - **Cron expression:** `0 2 * * *` (daily at 2:00 AM)
   - **Timezone:** Your timezone
   - **Enabled:** Checked
4. **Save**

**Verify:** Schedule appears in the list. For testing, you can use `*/5 * * * *` (every 5 minutes) — run a few cycles, then disable or delete the schedule.

---

### Step 7 — Manage Schedule Conflicts

**Purpose:** Understand behavior when a scheduled job is still running and the next run is due.

**Detailed steps:**

1. Create a job template that runs a long playbook (e.g., add `pause: seconds=120` to a playbook)
2. Create a recurring schedule that runs every 1 minute: `* * * * *`
3. Launch the job manually (or wait for first schedule) — it will run for ~2 minutes
4. When the next minute triggers, observe: the new job will **queue** (pending) until the first completes
5. **Optional:** Enable **Allow Simultaneous Jobs** on the template — now both can run in parallel

**Verify:** You understand queuing vs. concurrent execution.

---

## Part 4 — Hands-On: API and Webhook Triggers (35 minutes)

### Step 8 — Obtain an API Token

**Purpose:** Create an API token for authentication.

**Detailed steps:**

1. Log in to AAP
2. Click your **username** (top right) → **User Details** (or **My Account**)
3. Click the **Tokens** tab
4. Click **Create** (or **Add**)
5. **Description:** `Lab-12-API-Token`
6. **Scope:** Optional — leave default
7. Click **Save**
8. **Copy the token** — it is shown only once; store it securely

**Verify:** You have a token (e.g., `a1b2c3d4e5f6...`).

---

### Step 9 — Launch a Job via REST API

**Purpose:** Trigger a job using the API instead of the UI.

**Detailed steps:**

1. Get your **Job Template ID**: **Resources → Templates** → open `Ping-Lab` → note the ID in the URL (e.g., `https://aap1.lab.local/#/templates/job_template/7` → ID is `7`)
2. From your laptop (or a machine with `curl`), run:

```bash
curl -k -X POST \
  "https://aap1.lab.local/api/v2/job_templates/<TEMPLATE_ID>/launch/" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Replace `<TEMPLATE_ID>` and `<YOUR_TOKEN>` with your values.

3. The response is JSON containing `job` (the new job ID)
4. Check **Resources → Jobs** — the job should appear and run

**With extra variables and limit:**

```bash
curl -k -X POST \
  "https://aap1.lab.local/api/v2/job_templates/<TEMPLATE_ID>/launch/" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"env": "prod"}, "limit": "webservers"}'
```

**Verify:** Job is created and runs successfully via API.

---

### Step 10 — Configure Webhook for External Triggers

**Purpose:** Set up a webhook URL so external systems can trigger job launches.

**Detailed steps:**

1. Open a job template (e.g., `Ping-Lab`)
2. Scroll to the **Webhook** section (or **Options** tab)
3. Note the **Webhook URL** — e.g., `https://aap1.lab.local/api/v2/job_templates/<ID>/launch/`
4. **Webhook credential** (optional): Create a credential of type "Webhook" with a shared secret; attach to template for verification
5. **Test the webhook** from command line:

```bash
curl -k -X POST "https://aap1.lab.local/api/v2/job_templates/<ID>/launch/" \
  -H "Content-Type: application/json" \
  -d '{}'
```

If the template does not require authentication for webhooks, this may work without a token (check your AAP configuration). Otherwise, include the webhook key:

```bash
curl -k -X POST "https://aap1.lab.local/api/v2/job_templates/<ID>/launch/?token=<WEBHOOK_KEY>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Verify:** Job launches when the webhook URL receives a POST.

---

### Step 11 — Enable Provisioning Callback (Conceptual Setup)

**Purpose:** Configure a job template for provisioning callbacks (new hosts request configuration).

**Detailed steps:**

1. Open a job template (e.g., `Ping-Lab` or create `Provision-New-Host`)
2. **Options** tab → check **Enable Provisioning Callbacks**
3. **Save**
4. Note the **Provisioning Callback URL** and **Host Config Key** displayed
5. **How it works:** A new host (e.g., from cloud init or kickstart) runs:

```bash
curl -k -X POST "https://aap1.lab.local/api/v2/job_templates/<ID>/callback/" \
  -H "Content-Type: application/json" \
  -d '{"host_config_key": "<HOST_CONFIG_KEY>"}'
```

AAP adds the host to the template's inventory and launches the job.

**Lab note:** Full callback testing requires a host that can reach AAP and run the curl command. For this lab, configuring the option and understanding the URL/key is sufficient.

**Verify:** Provisioning Callback URL and Host Config Key are visible in the template.

---

## Part 5 — Hands-On: Monitoring Job Execution (25 minutes)

### Step 12 — Monitor Real-Time Job Output

**Purpose:** Observe job execution in real time and understand output structure.

**Detailed steps:**

1. Launch a job (e.g., `Ping-Lab`)
2. You are taken to the job output view
3. Observe:
   - **Header:** Job name, status, started time, elapsed time
   - **Output:** Play name, task names, per-host results (ok, changed, skipped, failed)
   - **Real-time updates:** Output appears as each task runs
4. Use **Search** (if available) to find specific text in the output
5. Use **Download** (if available) to save the full output

**Verify:** You can follow a job from start to finish in real time.

---

### Step 13 — Cancel a Running Job

**Purpose:** Cancel a job that is in progress.

**Detailed steps:**

1. Create a playbook with a long-running task (e.g., `pause: seconds=300`)
2. Add it to a job template (e.g., `Long-Running-Lab`)
3. Launch the job
4. While it is **Running**, click **Cancel** (or **Stop**)
5. Confirm the cancellation
6. Observe: the job stops; status becomes **Canceled**

**Verify:** Job is canceled and does not complete all tasks.

---

### Step 14 — Troubleshoot a Failed Job

**Purpose:** Diagnose and fix a failed job.

**Detailed steps:**

1. Create a playbook that intentionally fails (e.g., typo in module name: `ansible.builtin.pingg`)
2. Create a job template using this playbook
3. Launch the job — it will **Fail**
4. Open the job output
5. Locate the first **red** (failed) task
6. Read the error message — e.g., `module 'ansible.builtin.pingg' not found`
7. Fix the playbook (correct the typo)
8. Re-launch the job — it should succeed

**Verify:** You can identify the cause of failure and correct it.

---

### Step 15 — View Job History

**Purpose:** Navigate job history and understand retention.

**Detailed steps:**

1. Navigate to **Resources → Jobs** (or **Jobs** in new UI)
2. Review the list — columns typically include: Job Template, Status, Started, Elapsed, Launch Type (manual, schedule, callback, etc.)
3. Filter by status (e.g., Failed) if needed
4. Click a job to view details and output
5. **Settings → Jobs** — review **Job Event Retention** (if visible) to understand how long job data is kept

**Verify:** You can find past jobs and their outcomes.

---

## Part 6 — Hands-On: Job Results and Artifacts (25 minutes)

### Step 16 — Enable Fact Cache and View Host Facts

**Purpose:** Use fact caching and view stored facts on hosts.

**Detailed steps:**

1. Create a job template that gathers facts (e.g., uses `gather_facts: true`)
2. **Options** tab → check **Use Fact Cache**
3. **Save**
4. Launch the job — it gathers facts and, at completion, AAP stores them
5. Navigate to **Resources → Hosts**
6. Click a host that was in the job
7. Open the **Facts** tab — you should see cached facts (e.g., `ansible_hostname`, `ansible_os_family`)

**Verify:** Host facts are visible after a fact-cache-enabled job runs.

---

### Step 17 — Use set_stats for Artifacts in a Workflow

**Purpose:** Store data in workflow artifacts and use it in a downstream job.

**Detailed steps:**

1. Create a playbook `upstream.yml` that uses `set_stats`:

```yaml
---
- name: Upstream job - store artifact
  hosts: all
  gather_facts: false
  vars:
    deploy_version: "1.0"
  tasks:
    - name: Store artifact for downstream
      ansible.builtin.set_stats:
        data:
          deploy_version: "{{ deploy_version }}"
          db_host: "{{ groups['databases'][0] | default(inventory_hostname) }}"
        per_host: false
```

2. Create a job template `Upstream-Artifact` using this playbook
3. Create a playbook `downstream.yml` that uses the artifact:

```yaml
---
- name: Downstream job - use artifact
  hosts: all
  gather_facts: false
  tasks:
    - name: Display artifact
      ansible.builtin.debug:
        msg: "Deploy version: {{ deploy_version | default('N/A') }}, DB host: {{ db_host | default('N/A') }}"
```

4. Create job template `Downstream-Artifact` using `downstream.yml`
5. Create a **Workflow Template**:
   - Add `Upstream-Artifact` as first node
   - Add `Downstream-Artifact` as second node (On Success)
6. Launch the workflow
7. In the downstream job output, verify `deploy_version` and `db_host` are passed from the upstream job

**Verify:** Artifacts flow from upstream to downstream in the workflow.

---

## Part 7 — Real-Time Use Case Discussion (20 minutes)

### Case Study: DevOps Team — Scheduled Patching, API Integration, and Auto-Scaling Callbacks

**Background:**

A DevOps team manages 500+ Linux servers across development, staging, and production. They use AAP for configuration management, security patching, and application deployment.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Security patches must run in maintenance windows | Manual runs are error-prone; timezone confusion |
| CI/CD pipeline must trigger deployments | Need API integration with Jenkins/GitLab |
| Auto-scaling adds new VMs frequently | New VMs must be configured automatically without manual inventory updates |

**Solution: Multi-Method Job Execution**

#### 1. Scheduled Jobs for Nightly Security Patching

| Component | Implementation |
|-----------|-----------------|
| **Job template** | `Security-Patch-All` — runs patching playbook |
| **Schedule** | Recurring: `0 2 * * *` (daily at 2:00 AM local time) |
| **Inventory** | Dynamic (EC2) or static with prod hosts |
| **Limit** | Optional: use `limit: prod` to patch only production |
| **Notifications** | On failure → Slack channel; On success → optional email digest |

**Results:**

| Metric | Before | After |
|--------|--------|------|
| Patch compliance | 60% (manual, inconsistent) | 98% (automated, auditable) |
| Maintenance window violations | Frequent | Zero (scheduled precisely) |
| Ops team effort | 10 hrs/week | 1 hr/week (review only) |

#### 2. API Triggers for CI/CD Integration

| Component | Implementation |
|-----------|-----------------|
| **Pipeline** | Jenkins/GitLab CI builds app, runs tests |
| **Trigger** | On successful build → `POST /api/v2/job_templates/<ID>/launch/` |
| **Payload** | `{"extra_vars": {"app_version": "{{ build_number }}", "env": "staging"}}` |
| **Authentication** | API token stored in CI/CD secrets |
| **Verification** | Pipeline polls job status via `GET /api/v2/jobs/<ID>/` until complete |

**Results:**

| Metric | Before | After |
|--------|--------|------|
| Deploy time (build → staging) | 45 minutes (manual) | 8 minutes (automated) |
| Failed deployments | 15% (human error) | 2% (automated, consistent) |
| Audit trail | Partial | Full (every deploy in AAP) |

#### 3. Provisioning Callbacks for Auto-Scaling

| Component | Implementation |
|-----------|-----------------|
| **Cloud** | AWS Auto Scaling Group adds instances |
| **User data** | Cloud-init script runs on new instance boot |
| **Callback** | Script calls `POST .../callback/` with host config key |
| **AAP** | Adds host to inventory; launches `Provision-New-Host` job |
| **Playbook** | Installs agent, configures app, joins cluster |

**Results:**

| Metric | Before | After |
|--------|--------|------|
| Time to configure new VM | 30 minutes (manual) | 5 minutes (automated) |
| Configuration drift | High (manual variance) | Low (playbook-driven) |
| Scale-up during traffic spike | Delayed | Automatic |

**Key takeaway:** Combining scheduled jobs, API triggers, and provisioning callbacks enables fully automated, auditable operations — from routine maintenance to CI/CD and auto-scaling.

---

## Troubleshooting

### Job Fails to Launch

| Symptom | Check |
|---------|-------|
| "Inventory required" | Ensure template has an inventory with at least one host |
| "Credential required" | Add Machine credential to template |
| "Project not synced" | Sync the project; fix Git URL/credential if applicable |
| "Permission denied" | User needs Execute on template, Use on credential and inventory |

### Schedule Does Not Run

| Symptom | Check |
|---------|-------|
| No job at scheduled time | Verify schedule is **Enabled**; check timezone |
| Wrong time | Controller timezone in Settings → System |
| Schedule disabled | Re-enable in Schedules tab |

### API Launch Returns 403

| Cause | Fix |
|-------|-----|
| Invalid or expired token | Create new token in User Details → Tokens |
| Insufficient permissions | User needs Execute on template, Use on credential/inventory |
| Wrong URL | Use `POST .../launch/` not `POST .../` |

### Webhook Does Not Trigger Job

| Cause | Fix |
|-------|-----|
| AAP not reachable | External system must reach AAP on HTTPS (443) |
| Wrong URL | Use exact webhook URL from template |
| Auth required | Include webhook key or credential as configured |

### Facts Not Appearing on Host

| Cause | Fix |
|-------|-----|
| Fact cache not enabled | Enable "Use Fact Cache" on template |
| Job did not gather facts | Ensure `gather_facts: true` (default) in playbook |
| Job failed before setup | Fix job failure; re-run |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Manual launch | Launch jobs from UI; optional survey and prompts |
| Survey | Runtime input → extra variables |
| Limit / Tags | Restrict hosts and tasks at launch |
| One-time schedule | Job runs once at specified time |
| Recurring schedule | Cron syntax; daily, weekly, custom |
| Timezone | Schedules use Controller timezone |
| REST API | POST to `/launch/` to trigger jobs programmatically |
| Webhook | External systems trigger jobs via HTTP POST |
| Provisioning callback | New hosts request config via callback URL |
| Job status | Pending, Running, Successful, Failed, Canceled |
| Job output | Real-time; searchable; downloadable |
| Cancel | Stop a running job |
| Fact cache | Store facts for reuse; view on Hosts → Facts |
| Artifacts | set_stats in workflows; pass data to downstream jobs |

---

## Lab Completion Checklist

- [ ] Launched a job manually from the UI
- [ ] Launched a job with survey input (version, environment)
- [ ] Launched with Limit and Tags at runtime
- [ ] Launched with extra variables at runtime
- [ ] Created a one-time schedule and verified it ran
- [ ] Created a recurring schedule (cron)
- [ ] Obtained API token
- [ ] Launched a job via REST API (curl)
- [ ] Configured webhook and tested trigger
- [ ] Enabled provisioning callback on a template
- [ ] Monitored real-time job output
- [ ] Canceled a running job
- [ ] Troubleshot a failed job and fixed it
- [ ] Viewed job history
- [ ] Enabled fact cache and viewed host facts
- [ ] Used set_stats/artifacts in a workflow
- [ ] Can explain the DevOps use case (scheduled patching, API, callbacks)

---

## Quick Reference — Launch Methods

| Method | How |
|--------|-----|
| UI | Templates → Launch (rocket icon) |
| Schedule | Template → Schedules → Add |
| API | `POST /api/v2/job_templates/<ID>/launch/` |
| Webhook | `POST <webhook_url>` from external system |
| Callback | New host: `POST .../callback/` with host_config_key |

---

## Quick Reference — Cron Examples

| Schedule | Expression |
|----------|------------|
| Daily 2 AM | `0 2 * * *` |
| Every 15 min | `*/15 * * * *` |
| Weekly Mon 3 AM | `0 3 * * 1` |
| 1st of month | `0 0 1 * *` |

---

## Quick Reference — Job Status Values

| Status | Meaning |
|--------|---------|
| Pending | Queued |
| Running | Executing |
| Successful | Completed OK |
| Failed | Task failure |
| Canceled | Stopped by user |
| Error | Could not start |

---

*End of Lab 12*
