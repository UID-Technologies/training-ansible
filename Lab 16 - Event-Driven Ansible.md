# Lab 16 — Event-Driven Ansible

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate to Advanced (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 35% / Hands-on: 65%)                                      |
| **Outcome**  | Understand Event-Driven Ansible (EDA) architecture; write rulebooks with sources, conditions, and actions; use webhook, log, and Alertmanager event sources; build decision environments; integrate EDA with Automation Controller |
| **Platform** | Red Hat Ansible Automation Platform **2.4** — Event-Driven Ansible, Automation Controller |
| **Prerequisite** | Labs 02, 04, 05, 07 completed — AAP 2.4 installed, project with playbooks, job templates. EDA component installed (standalone or with AAP). |

---

## AAP 2.4 and Event-Driven Ansible Reference

| Component | Purpose |
|-----------|---------|
| **Event-Driven Ansible (EDA)** | Reactive automation — responds to events in real time |
| **ansible-rulebook** | CLI to run rulebooks |
| **Rulebook** | YAML file defining event sources, conditions, and actions |
| **Decision Environment** | Container image for running rulebooks (like EE for playbooks) |
| **Automation Controller** | Can be triggered by EDA via `run_job_template` action |

> **Note:** EDA can run standalone (ansible-rulebook CLI) or integrated with AAP. AAP 2.4 includes EDA as an optional component. This lab covers both standalone and Controller integration.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         EVENT SOURCES                                                │
│  - Webhook (HTTP POST)  - Log file  - Alertmanager  - Kafka (optional)               │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ Events
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN ANSIBLE (ansible-rulebook)                           │
│                                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                           │
│  │  RULEBOOK    │    │  CONDITIONS  │    │  ACTIONS     │                           │
│  │  - sources   │───▶│  - match     │───▶│  - run_      │                           │
│  │  - rules     │    │  - filter    │    │    playbook  │                           │
│  │              │    │              │    │  - run_job_ │                           │
│  │              │    │              │    │    template  │                           │
│  └──────────────┘    └──────────────┘    └──────┬──────┘                           │
└──────────────────────────────────────────────────┼───────────────────────────────────┘
                                                  │
                    ┌─────────────────────────────┼─────────────────────────────┐
                    ▼                             ▼                             ▼
┌─────────────────────────┐      ┌─────────────────────────┐      ┌─────────────────────────┐
│  LOCAL PLAYBOOK         │      │  AAP CONTROLLER         │      │  MANAGED NODES          │
│  (run_playbook)         │      │  (run_job_template)     │      │  web1, app1, db1        │
└─────────────────────────┘      └─────────────────────────┘      └─────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| EDA | Installed (standalone or with AAP) — `ansible-rulebook` available |
| Python 3.9+ | For ansible-rulebook |
| Collections | `ansible.eda` (or `eda.builtin` for built-in sources) |
| Project | Playbooks for remediation (e.g., disk cleanup, service restart) |
| Optional | Podman/Docker for decision environments |

---

## Verify Environment Before Starting

```bash
ansible-rulebook --version
ansible --version
ansible-galaxy collection list | grep -E "eda|ansible.eda"
```

---

## Part 1 — Theory: Event-Driven Automation, Rulebooks, Sources, Writing, Decision Environments, Integration (84 minutes)

### Section 15.1 — Introduction to Event-Driven Automation

#### What Is Event-Driven Ansible (EDA)?

**Event-Driven Ansible** enables **reactive automation** — instead of running automation on a schedule or manually, it responds to **events** as they happen.

| Traditional Automation | Event-Driven Automation |
|------------------------|-------------------------|
| Run playbook at 2 AM (schedule) | Run playbook when alert fires |
| Manual trigger | Automatic trigger on event |
| Poll-based | Push-based (event-driven) |

#### Traditional vs. Event-Driven Automation

| Aspect | Traditional | Event-Driven |
|--------|-------------|--------------|
| **Trigger** | Schedule, manual, API call | Event from monitoring, webhook, log |
| **Latency** | Minutes to hours | Seconds |
| **Use case** | Proactive maintenance | Reactive remediation |
| **Example** | Nightly patching | Restart service when crash detected |

#### Use Cases for Reactive Automation

| Use Case | Event Source | Action |
|----------|--------------|--------|
| High disk usage | Monitoring (Prometheus, Zabbix) | Run cleanup playbook |
| Application crash | Health check failure | Restart service, notify team |
| Security alert | SIEM / IDS | Isolate host, block IP |
| Cloud scaling | Cloud metric threshold | Scale VMs or containers |
| ITSM | ServiceNow incident | Create ticket, run remediation |

#### EDA Architecture and Components

| Component | Purpose |
|-----------|---------|
| **Event source** | Produces events (webhook, Kafka, log, Alertmanager) |
| **Rulebook** | Defines rules: when condition matches, run action |
| **ansible-rulebook** | Runtime engine — runs rulebook, listens for events |
| **Decision environment** | Container with ansible-rulebook, collections, deps |
| **Action** | run_playbook, run_job_template, run_module, etc. |

#### When to Use EDA

| Use EDA When | Use Traditional When |
|--------------|------------------------|
| Fast response needed | Scheduled maintenance |
| Events from monitoring/alerting | Manual or CI/CD trigger |
| Self-healing / auto-remediation | Predictable workflows |
| Real-time compliance | Batch processing |

---

### Section 15.2 — Understanding Rulebooks

#### What Is a Rulebook?

A **rulebook** is a YAML file that defines:

1. **Event sources** — where events come from
2. **Rules** — conditions to match events
3. **Actions** — what to do when a rule matches

#### Rulebook Structure and Syntax

```yaml
---
- name: Rulebook name
  hosts: all
  sources:
    - event_source_plugin:
        param: value
  rules:
    - name: Rule name
      condition: event.field == "value"
      action:
        run_playbook:
          name: playbook.yml
```

#### Event Sources

- Plugins that generate or receive events
- Examples: `eda.builtin.webhook`, `eda.builtin.range`, `ansible.eda.alertmanager`, `ansible.eda.file`, `ansible.eda.kafka`

#### Conditions and Matching

- **condition** — Python expression evaluated against each event
- Access event data: `event.payload`, `event.alert`, `event.i`, etc.
- Examples: `event.payload.message == "hello"`, `event.alert.status == "firing"`

#### Actions to Take When Events Match

| Action | Purpose |
|--------|---------|
| `run_playbook` | Execute Ansible playbook locally |
| `run_job_template` | Launch AAP job template |
| `run_workflow_template` | Launch AAP workflow template |
| `run_module` | Run single Ansible module |
| `print_event` | Debug — print event |
| `set_fact` / `retract_fact` | Manage facts |
| `post_event` | Send event to another source |

---

### Section 15.3 — Event Sources

#### Webhook Event Sources

- **eda.builtin.webhook** — HTTP server; receives POST requests (built-in)
- Events: JSON payload in request body
- Use: Custom integrations, CI/CD, scripts

```yaml
sources:
  - eda.builtin.webhook:
      host: 0.0.0.0
      port: 5000
```

#### Kafka and Message Queue Sources

- **ansible.eda.kafka** — Consume from Kafka topics
- Use: High-throughput event streams, microservices

#### File and Log Monitoring

- **ansible.eda.file** — Watch file for changes (e.g., log file)
- **ansible.eda.log** — Tail log file; emit events on new lines
- Use: Log-based triggers (e.g., error pattern in log)

#### Alertmanager Integration

- **ansible.eda.alertmanager** — Receive Prometheus Alertmanager webhooks
- Events: `event.alert` with labels, status, annotations
- Use: Monitoring-driven remediation

```yaml
sources:
  - ansible.eda.alertmanager:
      host: 0.0.0.0
      port: 8000
```

#### Custom Event Sources (Overview)

- Implement custom source plugin in Python
- Register in collection; use in rulebook like built-in sources

---

### Section 15.4 — Writing Rulebooks

#### Creating Simple Rulebooks

- Start with `eda.builtin.range` for testing (generates numeric events)
- Add webhook for real-world testing

#### Defining Event Source Connections

- Each source has its own config (host, port, topic, path)
- Multiple sources can be listed; events from any can trigger rules

#### Writing Condition Matching Rules

- Use `event` object — structure depends on source
- Webhook: `event.payload.<key>`
- Alertmanager: `event.alert.labels.<label>`, `event.alert.status`
- Logical operators: `and`, `or`, `not`

#### Triggering Job Templates as Actions

```yaml
action:
  run_job_template:
    name: "Remediate-Disk-Space"
    organization: "Default"
    extra_vars:
      target_host: "{{ event.payload.host }}"
```

- Requires Controller connection (env vars or credential)
- `ANSIBLE_CONTROLLER_URL`, `ANSIBLE_CONTROLLER_TOKEN` (or OAuth)

#### Variables in Rulebooks

- Use `{{ event.payload.key }}` in action parameters
- Pass event data to playbook as extra_vars

#### Testing and Debugging Rulebooks

- `ansible-rulebook --rulebook rulebook.yml -i inventory.yml --verbose`
- Use `print_event` action to inspect event structure
- Test webhook with `curl`

---

### Section 15.5 — Decision Environments

#### What Are Decision Environments?

**Decision environments** are container images for running rulebooks. They include:

- ansible-rulebook
- ansible-core
- ansible.eda collection
- Python dependencies
- Optional: custom collections

#### Building Decision Environments

- Use **ansible-builder** with `execution-environment.yml`
- Base: `quay.io/ansible/ansible-rulebook:latest` or Red Hat decision environment base
- Add collections, Python packages as needed

#### Required Dependencies for EDA

- ansible
- ansible-rulebook
- ansible-runner
- ansible.eda collection

#### Publishing to Automation Hub

- Build image; push to container registry
- Add to Private Hub as decision environment
- AAP references it when activating rulebooks

---

### Section 15.6 — Integrating EDA with Automation Controller

#### Activating Rulebooks in AAP

- **Resources → Rulebooks** (or Event-Driven Ansible section)
- Create rulebook; upload or link to project
- Assign decision environment
- Activate — rulebook runs as a service

#### Triggering Workflows from Events

- Use `run_workflow_template` action
- Pass extra vars from event to workflow

#### Credential Management for EDA

- Controller token or OAuth for `run_job_template` / `run_workflow_template`
- Store in environment or credential file

#### Monitoring Active Rulebooks

- AAP UI: Rulebook instances, status, logs
- View which rulebooks are running; restart if needed

---

## Part 2 — Hands-On: Install EDA and Create Webhook Rulebook (40 minutes)

### Step 1 — Install Event-Driven Ansible

**Purpose:** Install ansible-rulebook and required collections.

**Detailed steps:**

1. Install via pip (on control node or workstation):

```bash
pip3 install ansible ansible-rulebook ansible-runner
```

2. Install EDA collection:

```bash
ansible-galaxy collection install ansible.eda
```

3. Verify:

```bash
ansible-rulebook --version
ansible-galaxy collection list ansible.eda
```

**Verify:** ansible-rulebook runs; ansible.eda is installed.

---

### Step 2 — Create a Simple Range-Based Rulebook

**Purpose:** Test EDA with built-in range source.

**Detailed steps:**

1. Create `inventory_eda.yml`:

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
```

2. Create `playbooks/hello_remediation.yml`:

```yaml
---
- name: Hello remediation
  hosts: localhost
  connection: local
  tasks:
    - name: Say hello
      ansible.builtin.debug:
        msg: "Event-driven remediation triggered!"
```

3. Create `rulebooks/range_rulebook.yml`:

```yaml
---
- name: Range test rulebook
  hosts: localhost
  sources:
    - eda.builtin.range:
        limit: 5
  rules:
    - name: Trigger on i equals 2
      condition: event.i == 2
      action:
        run_playbook:
          name: playbooks/hello_remediation.yml
```

4. Run:

```bash
ansible-rulebook -i inventory_eda.yml --rulebook rulebooks/range_rulebook.yml
```

**Verify:** Playbook runs when `event.i == 2`.

---

### Step 3 — Create Webhook-Triggered Rulebook

**Purpose:** Accept HTTP webhook and trigger playbook on matching payload.

**Detailed steps:**

1. Create `rulebooks/webhook_rulebook.yml`:

```yaml
---
- name: Webhook-triggered remediation
  hosts: localhost
  sources:
    - eda.builtin.webhook:
        host: 0.0.0.0
        port: 5000
  rules:
    - name: Remediate on disk alert
      condition: event.payload.alert_type == "high_disk"
      action:
        run_playbook:
          name: playbooks/disk_cleanup.yml
          extra_vars:
            target_host: "{{ event.payload.host }}"
```

2. Create `playbooks/disk_cleanup.yml`:

```yaml
---
- name: Disk cleanup remediation
  hosts: localhost
  connection: local
  vars:
    target_host: "{{ target_host | default('localhost') }}"
  tasks:
    - name: Simulate disk cleanup
      ansible.builtin.debug:
        msg: "Running disk cleanup for {{ target_host }}"
    - name: Clean old logs (simulated)
      ansible.builtin.debug:
        msg: "Would clean /var/log on {{ target_host }}"
```

3. Run rulebook in one terminal:

```bash
ansible-rulebook -i inventory_eda.yml --rulebook rulebooks/webhook_rulebook.yml --verbose
```

4. In another terminal, trigger webhook:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"alert_type": "high_disk", "host": "web1.lab.local"}' \
  http://localhost:5000/endpoint
```

**Verify:** Playbook runs when webhook payload matches condition.

---

### Step 4 — Test with Non-Matching Payload

**Purpose:** Confirm rule only fires when condition matches.

**Detailed steps:**

1. With rulebook running, send non-matching payload:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"alert_type": "high_cpu", "host": "web1"}' \
  http://localhost:5000/endpoint
```

2. No playbook should run (condition not met)
3. Send matching payload again — playbook runs

**Verify:** Rule is selective; only matching events trigger action.

---

## Part 3 — Hands-On: Log Monitoring Rulebook (35 minutes)

### Step 5 — Create Log-Based Rulebook

**Purpose:** Trigger remediation when log file contains error pattern.

**Detailed steps:**

1. Create a sample log file:

```bash
mkdir -p /tmp/eda-lab
echo "2024-01-15 10:00:00 INFO Application started" > /tmp/eda-lab/app.log
```

2. Create `rulebooks/log_rulebook.yml`:

```yaml
---
- name: Log monitoring rulebook
  hosts: localhost
  sources:
    - ansible.eda.file:
        path: /tmp/eda-lab/app.log
        path_ignore_regex: []
  rules:
    - name: React to ERROR in log
      condition: '"ERROR" in event.message'
      action:
        run_playbook:
          name: playbooks/hello_remediation.yml
```

**Note:** `ansible.eda.file` may emit events on file change. For log tailing, `ansible.eda.log` (if available) or a custom source may be used. If `ansible.eda.file` structure differs, adjust condition (e.g., `event.content` or `event.data`). Check collection docs for exact event structure.

3. **Alternative — use range + manual test:** If log source is complex, use webhook for lab and document log pattern for production.

4. Run rulebook; append ERROR to log: `echo "2024-01-15 10:01:00 ERROR Something failed" >> /tmp/eda-lab/app.log`

**Verify:** Understand log-based event pattern (adjust per actual plugin behavior).

---

### Step 6 — Create Alertmanager-Style Rulebook

**Purpose:** Simulate Alertmanager webhook format.

**Detailed steps:**

1. Create `rulebooks/alertmanager_rulebook.yml`:

```yaml
---
- name: Alertmanager-style rulebook
  hosts: localhost
  sources:
    - eda.builtin.webhook:
        host: 0.0.0.0
        port: 8000
  rules:
    - name: Restart service on firing alert
      condition: >
        event.payload.get("status") == "firing" and
        event.payload.get("alerts", [{}])[0].get("labels", {}).get("alertname") == "ServiceDown"
      action:
        run_playbook:
          name: playbooks/restart_service.yml
          extra_vars:
            service_name: "{{ event.payload.alerts[0].labels.service | default('myapp') }}"
```

2. Create `playbooks/restart_service.yml`:

```yaml
---
- name: Restart service remediation
  hosts: localhost
  connection: local
  vars:
    service_name: "{{ service_name | default('myapp') }}"
  tasks:
    - name: Restart service (simulated)
      ansible.builtin.debug:
        msg: "Would restart service {{ service_name }}"
```

3. Run rulebook; trigger with Alertmanager-like payload:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"status":"firing","alerts":[{"labels":{"alertname":"ServiceDown","service":"nginx"}}]}' \
  http://localhost:8000/endpoint
```

**Verify:** Playbook runs with Alertmanager-style payload.

---

## Part 4 — Hands-On: Integrate with Automation Controller (40 minutes)

### Step 7 — Create Remediation Job Template in AAP

**Purpose:** Job template for EDA to trigger.

**Detailed steps:**

1. In AAP, ensure playbook `disk_cleanup.yml` (or similar) exists in project
2. Create job template `Remediate-Disk-Cleanup`:
   - **Playbook:** disk_cleanup.yml (or your remediation playbook)
   - **Inventory:** Lab-Static-Inventory
   - **Credential:** Machine credential
3. **Save**

**Verify:** Job template exists and runs successfully when launched manually.

---

### Step 8 — Create Rulebook with run_job_template Action

**Purpose:** Trigger AAP job template from EDA.

**Detailed steps:**

1. Obtain AAP API token: User Details → Tokens → Create
2. Create `rulebooks/controller_rulebook.yml`:

```yaml
---
- name: Trigger Controller job from webhook
  hosts: localhost
  sources:
    - eda.builtin.webhook:
        host: 0.0.0.0
        port: 6000
  rules:
    - name: Run remediation job template
      condition: event.payload.action == "remediate"
      action:
        run_job_template:
          name: "Remediate-Disk-Cleanup"
          organization: "Default"
```

3. Set environment variables before running:

```bash
export ANSIBLE_CONTROLLER_URL="https://aap1.lab.local"
export ANSIBLE_CONTROLLER_TOKEN="your-api-token"
export ANSIBLE_CONTROLLER_VERIFY_SSL="false"
```

4. Run rulebook:

```bash
ansible-rulebook -i inventory_eda.yml --rulebook rulebooks/controller_rulebook.yml --verbose
```

5. Trigger webhook:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"action": "remediate"}' \
  http://localhost:6000/endpoint
```

**Verify:** AAP job template launches when webhook received.

---

### Step 9 — Activate Rulebook in AAP (If EDA Controller Integration Available)

**Purpose:** Run rulebook as managed service in AAP.

**Detailed steps:**

1. In AAP 2.4, navigate to **Event-Driven Ansible** or **Rulebooks** (if available)
2. **Add** rulebook:
   - **Name:** Lab-Webhook-Rulebook
   - **Source:** Project (upload rulebook) or paste content
   - **Decision Environment:** Default or custom
   - **Credential:** Controller credential for run_job_template
3. **Activate** the rulebook
4. Rulebook runs as a service; listen for events

**Lab note:** EDA Controller integration may require AAP 2.4 with EDA add-on. If not available, standalone ansible-rulebook (Steps 1–8) demonstrates concepts.

**Verify:** Rulebook is active in AAP (if supported).

---

## Part 5 — Hands-On: Decision Environment (30 minutes)

### Step 10 — Build a Decision Environment

**Purpose:** Create container image for running rulebooks.

**Detailed steps:**

1. Create `execution-environment.yml`:

```yaml
---
version: 3
dependencies:
  galaxy:
    collections:
      - name: ansible.eda
        version: ">=1.0.0"
  python: |
    ansible-rulebook
    ansible-runner
```

2. Build (requires ansible-builder, podman/docker):

```bash
ansible-builder build -t eda-lab-de:1.0 -f execution-environment.yml
```

3. Run rulebook in container:

```bash
podman run --rm -v $(pwd):/rules \
  -e ANSIBLE_CONTROLLER_URL -e ANSIBLE_CONTROLLER_TOKEN \
  eda-lab-de:1.0 \
  ansible-rulebook -i /rules/inventory_eda.yml --rulebook /rules/rulebooks/webhook_rulebook.yml
```

**Lab note:** If ansible-builder or podman is not available, skip build and use system-installed ansible-rulebook.

**Verify:** Decision environment builds and runs rulebook (if tools available).

---

## Part 6 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Cloud Infrastructure Team — Auto-Remediation

**Background:**

A cloud infrastructure team manages 200+ VMs. Common issues: high disk usage (logs, temp files) and application crashes. Previously, on-call engineers received alerts and manually ran playbooks — slow and error-prone.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| High disk alerts | 20+ per week; manual cleanup took 30 min each |
| App crashes | Required manual SSH and restart |
| Alert fatigue | Engineers ignored non-critical alerts |
| Inconsistent remediation | Different engineers did different things |

**Solution: Event-Driven Ansible**

#### 1. High Disk Usage — Auto Cleanup

| Component | Implementation |
|-----------|-----------------|
| **Event source** | Prometheus Alertmanager webhook (alert: HighDiskUsage) |
| **Rulebook** | Match `event.alert.labels.alertname == "HighDiskUsage"` |
| **Action** | `run_job_template`: "Cleanup-Disk-Space" |
| **Extra vars** | `target_host: "{{ event.alert.labels.instance }}"` |
| **Playbook** | Cleans /var/log, /tmp; truncates large logs; notifies if still high |

**Flow:** Alertmanager fires → EDA receives → rule matches → AAP job runs → disk cleaned → next alert check passes.

#### 2. Application Crash — Auto Restart

| Component | Implementation |
|-----------|-----------------|
| **Event source** | Health check failure webhook (e.g., from load balancer or custom monitor) |
| **Rulebook** | Match `event.payload.service_down == true` |
| **Action** | `run_job_template`: "Restart-Application" |
| **Extra vars** | `service: "{{ event.payload.service }}"`, `host: "{{ event.payload.host }}"` |
| **Playbook** | Restarts service; verifies health; notifies on-call if restart fails |

**Flow:** Health check fails → webhook to EDA → job runs → service restarted → health check passes.

#### 3. Notify on-Call for Failures

- Playbook uses `fail` or `assert` when remediation fails
- AAP job fails → notification template (Slack, PagerDuty) fires
- On-call gets alert only when auto-remediation fails

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Disk alert resolution | 30 min (manual) | 2 min (automated) |
| App crash resolution | 15 min (manual) | 1 min (automated) |
| On-call pages (remediation) | 40/week | 5/week (only failures) |
| Consistency | Variable | 100% (playbook-driven) |

**Key takeaway:** EDA turns monitoring alerts into immediate, consistent remediation — reducing MTTR and on-call load while keeping humans in the loop for failures.

---

## Troubleshooting

### ansible-rulebook Not Found

| Cause | Fix |
|-------|-----|
| Not installed | `pip3 install ansible-rulebook` |
| Wrong path | Use full path or ensure in PATH |

### Webhook Not Triggering

| Cause | Fix |
|-------|-----|
| Wrong port | Check rulebook port; firewall |
| Wrong payload | Match condition exactly; use --verbose |
| Wrong endpoint | Default is `/endpoint` |

### run_job_template Fails

| Cause | Fix |
|-------|-----|
| No token | Set ANSIBLE_CONTROLLER_TOKEN |
| Wrong URL | Set ANSIBLE_CONTROLLER_URL |
| SSL verify | Set ANSIBLE_CONTROLLER_VERIFY_SSL=false for self-signed |
| Job template name | Must match exactly; check organization |

### Condition Never Matches

| Cause | Fix |
|-------|-----|
| Wrong event structure | Use print_event to inspect |
| Typo in field | Check event.payload vs event.alert |
| Type mismatch | Use `| string` or `| int` if needed |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|-----------------|
| EDA | Reactive automation; event-triggered actions |
| Rulebook | YAML with sources, rules, conditions, actions |
| Event sources | webhook, range, file, alertmanager, kafka |
| Condition | Python expression; event.field, event.payload |
| Actions | run_playbook, run_job_template, run_workflow_template |
| Decision environment | Container for running rulebooks |
| Controller integration | run_job_template with token/URL |
| Webhook | HTTP POST to trigger rules |

---

## Lab Completion Checklist

- [ ] Installed ansible-rulebook and ansible.eda
- [ ] Created range-based rulebook
- [ ] Created webhook-triggered rulebook
- [ ] Tested matching and non-matching payloads
- [ ] Created log or Alertmanager-style rulebook
- [ ] Created remediation job template in AAP
- [ ] Created rulebook with run_job_template
- [ ] Triggered AAP job from webhook
- [ ] Built decision environment (if tools available)
- [ ] Can explain cloud infra auto-remediation use case

---

## Quick Reference — Rulebook Structure

```yaml
---
- name: Name
  hosts: localhost
  sources:
    - plugin_name: {param: value}
  rules:
    - name: Rule name
      condition: event.field == "value"
      action:
        run_playbook:
          name: playbook.yml
```

---

## Quick Reference — Common Event Sources

| Source | Plugin | Use |
|--------|--------|-----|
| Webhook | eda.builtin.webhook | HTTP POST |
| Range | eda.builtin.range | Testing |
| Alertmanager | ansible.eda.alertmanager | Prometheus |
| File | ansible.eda.file | File changes |
| Kafka | ansible.eda.kafka | Message queue |

---

## Quick Reference — run_job_template Environment

| Variable | Purpose |
|----------|---------|
| ANSIBLE_CONTROLLER_URL | Controller URL |
| ANSIBLE_CONTROLLER_TOKEN | API token |
| ANSIBLE_CONTROLLER_VERIFY_SSL | false for self-signed |

---

*End of Lab 16*
