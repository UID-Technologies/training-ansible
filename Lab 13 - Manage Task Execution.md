# Lab 13 — Manage Task Execution

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 3 hours (Theory: 30% / Hands-on: 70%)                                     |
| **Outcome**  | Understand how Ansible executes tasks (forks, connection types, serial vs. parallel); configure AAP execution control (instance capacity, concurrent limits, instance groups, queue); optimize performance (forks, fact gathering, pipelining, async); troubleshoot execution issues; apply execution environment considerations |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller) — Classic and New UI |
| **Prerequisite** | Labs 02, 03, 04, 05, 07 completed — AAP 2.4 installed, multi-node setup (optional), inventory, project, job templates. Lab 03 recommended for instance group exercises. |

---

## AAP 2.4 UI Navigation Reference (Classic and New UI)

Before starting, familiarize yourself with the **Automation Controller** UI structure in AAP 2.4:

| Menu | Classic UI Location | New UI (Preview) Location |
|------|----------------------|---------------------------|
| **Resources** | Left navigation panel | Left sidebar — Templates, Inventories, Hosts |
| **Administration** | Left panel | Settings gear or Administration |
| **Instances** | Administration → Instances | Administration → Instances |
| **Instance Groups** | Administration → Instance Groups | Administration → Instance Groups |
| **Execution Environments** | Administration → Execution Environments | Administration → Execution Environments |
| **Settings** | Left panel — System, Jobs | System configuration, Job settings |
| **Templates** | Resources → Templates | Resources → Templates |

> **Note:** AAP 2.4 offers a **Preview of New User Interface** (Settings → System → Miscellaneous). This lab covers both UIs; adjust paths if your environment uses the new UI preview.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                         │
│                         https://aap1.lab.local                                        │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
                                       │ HTTPS (443)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                       │
│                                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │  EXECUTION       │  │  INSTANCE        │  │  EXECUTION       │  │  JOB        │ │
│  │  CONTROL         │  │  GROUPS          │  │  ENVIRONMENTS    │  │  QUEUE      │ │
│  │  - Forks         │  │  - dev-workers   │  │  - Default EE    │  │  - Capacity │ │
│  │  - Capacity      │  │  - prod-workers  │  │  - Custom EE     │  │  - Limits   │ │
│  │  - Slots         │  │  - Assignment    │  │  - Python deps   │  │  - Priority │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  └─────────────┘ │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
                                       │ SSH (22) / WinRM (5985-5986)
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │ web1.lab.local│  │ app1.lab.local│  │ db1.lab.local  │
            │ [webservers]  │  │ [appservers]  │  │ [databases]   │
            └───────────────┘  └───────────────┘  └───────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` with playbooks in `lab-playbooks` |
| Inventory | `Lab-Static-Inventory` with 2+ hosts (webservers, appservers, databases) |
| Job Templates | At least `Ping-Lab`; `Deploy-App-Survey` or similar |
| Credential | Machine credential (SSH key) |
| Managed Nodes | 2–4 Linux VMs reachable via SSH |
| Optional | Multi-node AAP (Lab 03) for instance group exercises |

---

## Verify AAP 2.4 Before Starting

1. Log in to the Automation Controller: `https://aap1.lab.local`
2. Click **?** or **user profile** → **About** — version 4.4.x (AAP 2.4)
3. **Administration → Instances** — note your instance(s) and capacity
4. **Administration → Instance Groups** — note existing groups (default, controlplane)
5. Ensure at least one job template runs successfully

---

## Part 1 — Theory: Task Execution, Control, Optimization, Troubleshooting, and EEs (54 minutes)

### Section 12.1 — Understanding Task Execution

#### How Ansible Executes Tasks

Ansible executes tasks in sequence across hosts. For each task:

1. **Connect** — Establish connection (SSH, WinRM, local)
2. **Execute** — Run the module on the target host
3. **Return** — Collect result and update host state
4. **Next** — Move to next task (or next host)

| Phase | What Happens |
|-------|--------------|
| **Gathering Facts** | `setup` module runs on each host (unless `gather_facts: false`) |
| **Task Loop** | For each task in the playbook, Ansible runs it on all hosts (or per host) |
| **Handlers** | Run at end of play if notified |

#### Forks: Parallel Execution Explained

**Forks** — the number of hosts Ansible manages simultaneously.

| Forks | Behavior |
|-------|----------|
| **1** | One host at a time — sequential, slow |
| **5** | Default — 5 hosts in parallel |
| **10** | 10 hosts in parallel — faster for large inventories |
| **50** | 50 hosts — higher throughput; more CPU/memory on control node |

**Rule of thumb:** `forks = min(host_count, available_capacity, safe_limit)`. Too high can overload the control node or network.

#### Connection Types (SSH, WinRM, Local)

| Connection | Protocol | Use Case |
|------------|----------|---------|
| **SSH** | TCP 22 | Linux, Unix, network devices |
| **WinRM** | TCP 5985 (HTTP), 5986 (HTTPS) | Windows |
| **Local** | N/A | Run on control node (localhost) |

Connection is set in inventory (`ansible_connection`) or credential.

#### Understanding Task Output

| Output Element | Meaning |
|----------------|---------|
| **ok** | Task succeeded; no change needed |
| **changed** | Task succeeded; state was modified |
| **skipped** | Task skipped (e.g., `when` condition false) |
| **failed** | Task failed — play stops (unless `ignore_errors`) |
| **unreachable** | Could not connect to host |

#### Serial vs. Parallel Execution Strategies

| Strategy | How | Use Case |
|----------|-----|----------|
| **Parallel (default)** | All hosts run each task together (up to forks limit) | Most playbooks |
| **Serial** | Hosts in batches (e.g., `serial: 2` = 2 at a time) | Rolling updates, zero-downtime deploys |

```yaml
- name: Rolling update
  hosts: webservers
  serial: 2
  tasks:
    - name: Restart app
      ansible.builtin.service:
        name: myapp
        state: restarted
```

---

### Section 12.2 — Execution Control in AAP

#### Instance Capacity and Job Slots

- Each **instance** (control or execution node) has a **capacity** — max concurrent forks
- Capacity ≈ based on CPU and RAM (e.g., 4 vCPU / 8 GB ≈ 30–50 forks)
- **Job slot** — one fork consumed per host being managed
- When capacity is full, new jobs **queue**

#### Concurrent Job Limits

| Setting | Location | Purpose |
|---------|----------|---------|
| **System job limit** | Settings → Jobs | Max concurrent jobs across entire AAP |
| **Org job limit** | Organization settings | Max concurrent jobs per organization |
| **Allow Simultaneous Jobs** | Job template | Allow multiple runs of same template at once |

#### Instance Group Assignment for Jobs

- **Instance group** — logical group of instances (e.g., `dev-workers`, `prod-workers`)
- **Job template** → **Instance Groups** — pin template to specific group
- Jobs run only on instances in the assigned group(s)
- Use for: workload isolation, compliance, capacity reservation

#### Queue Management

| State | Meaning |
|-------|---------|
| **Pending** | Job waiting for an instance with capacity |
| **Running** | Job executing on an instance |
| **Queued** | More jobs than capacity — they wait |

**Queue order:** FIFO by default; can use job prioritization (if configured).

#### Job Prioritization Basics

- Jobs can have a **priority** (higher = run sooner)
- Set at launch or in template
- Useful for: critical patching before routine scans

---

### Section 12.3 — Performance Optimization for Beginners

#### Optimizing Fork Values for Your Environment

| Factor | Recommendation |
|--------|-----------------|
| **Small inventory (< 20)** | Default 5–10 forks |
| **Medium (20–100)** | 20–50 forks; monitor CPU/memory |
| **Large (100+)** | 50–100; use execution nodes to scale |
| **Network latency** | Lower forks if connections are slow |

**Where to set:** Job template → **Forks** field (or `ansible.cfg` for CLI).

#### Fact Gathering Optimization

| Option | Effect |
|--------|--------|
| **gather_facts: false** | Skip setup — faster if facts not needed |
| **Fact cache** | Store facts; reuse in later jobs (enable on template) |
| **gather_subset** | `gather_subset: min` — minimal facts only |

#### Using Pipelining for SSH Performance

- **Pipelining** — reduces SSH round-trips by piping Python to remote
- Set in `ansible.cfg`: `pipelining = True`
- **Requirement:** `requiretty` disabled in sudoers on managed nodes
- **Benefit:** 20–40% faster for SSH-based playbooks

#### Understanding When to Use Async Tasks

- **Async** — run long task in background; poll for completion
- Use for: package installs, reboots, long-running scripts
- Avoid for: quick tasks (overhead not worth it)

```yaml
- name: Long-running task
  ansible.builtin.command: /opt/script.sh
  async: 3600
  poll: 10
```

#### Connection Persistence and Pooling

- **SSH multiplexing** — reuse one SSH connection for multiple tasks
- Controlled by `ControlPersist` in SSH config
- Ansible uses it by default when supported
- Reduces connection overhead for many hosts

---

### Section 12.4 — Troubleshooting Execution Issues

#### Enabling Verbose Output for Debugging

| Verbosity | Flag | Use |
|-----------|------|-----|
| 0 | (default) | Normal output |
| 1 | `-v` | Verbose |
| 2 | `-vv` | More verbose |
| 3 | `-vvv` | Connection debugging |
| 4 | `-vvvv` | Full debug |

**In AAP:** Enable "Prompt on Launch" for Verbosity on job template.

#### Common Error Messages and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `UNREACHABLE!` | SSH/WinRM connection failed | Check credential, network, firewall |
| `Permission denied (publickey)` | Wrong key or user | Verify Machine credential |
| `Module not found` | Missing collection or EE | Install collection; use correct EE |
| `No module named 'X'` | Python dependency missing | Add to EE or install on node |
| `Host key verification failed` | First-time SSH | Add to known_hosts; or `host_key_checking=False` (lab only) |

#### Connection Timeout Issues

| Symptom | Fix |
|---------|-----|
| Jobs hang at "Gathering Facts" | Increase timeout in credential or `ansible.cfg` |
| Intermittent timeouts | Check network; reduce forks |
| `Connection timed out` | Firewall, wrong IP, SSH not listening |

#### Permission Denied Errors

| Type | Fix |
|------|-----|
| SSH permission denied | Correct user, key, sudoers |
| Sudo password | Add privilege escalation to credential |
| File permission | Check ownership, mode on managed node |

#### Module Execution Failures

| Cause | Fix |
|------|-----|
| Wrong module name | Use FQCN: `ansible.builtin.copy` |
| Missing parameter | Check module docs |
| Python version | EE or node Python mismatch |

#### Using Check Mode to Validate Changes

- **Check mode** (`--check`) — dry run; no changes made
- In AAP: Job template → **Job Type: Check**
- Use to validate before real run

---

### Section 12.5 — Execution Environment Considerations

#### How Execution Environments Affect Task Execution

- **Execution Environment (EE)** — container image with Ansible Core, collections, Python libs
- Job runs **inside** the EE container on an execution node
- EE determines: which modules/collections exist, Python version, dependencies

#### Python and Dependency Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `Module X not found` | Collection not in EE | Add collection to EE; rebuild |
| `No module named 'boto3'` | Python package missing | Add to EE `requirements.txt`; rebuild |
| Python 2 vs 3 | Wrong interpreter | Use EE with correct Python |

#### Switching Execution Environments for Compatibility

- Job template → **Execution Environment** — select different EE
- Use case: one playbook needs `amazon.aws`, another needs `azure.azcollection`
- Create purpose-specific EEs (e.g., `ee-aws`, `ee-azure`)

---

## Part 2 — Hands-On: Understanding Forks and Task Output (25 minutes)

### Step 1 — Observe Default Fork Behavior

**Purpose:** See how forks affect parallel execution.

**Detailed steps:**

1. Create a playbook that runs a simple task on all hosts:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/fork_demo.yml > /dev/null << 'EOF'
---
- name: Fork demo - run on all hosts
  hosts: all
  gather_facts: false
  tasks:
    - name: Simulate work
      ansible.builtin.shell: sleep 2 && echo "Done on {{ inventory_hostname }}"
    - name: Second task
      ansible.builtin.debug:
        msg: "Task 2 on {{ inventory_hostname }}"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/fork_demo.yml
```

2. Create job template `Fork-Demo`:
   - **Playbook:** `fork_demo.yml`
   - **Forks:** `5` (default)
   - **Inventory:** `Lab-Static-Inventory`
   - **Credential:** Machine credential
3. **Save** and **Launch**
4. Observe output — hosts run in parallel (up to 5); note elapsed time

**Verify:** Job completes; output shows parallel execution across hosts.

---

### Step 2 — Compare Forks 1 vs. 10

**Purpose:** Measure impact of fork value on execution time.

**Detailed steps:**

1. Edit `Fork-Demo` → set **Forks** to `1`
2. **Launch** — note **Elapsed** time when job completes
3. Edit **Forks** to `10` (or your host count)
4. **Launch** again — note **Elapsed** time
5. Compare: forks=1 should be slower (sequential); forks=10 faster (parallel)

**Verify:** Higher forks reduce elapsed time for multi-host playbooks.

---

### Step 3 — Use Serial for Rolling Execution

**Purpose:** Run tasks in batches using `serial`.

**Detailed steps:**

1. Create playbook `serial_demo.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/serial_demo.yml > /dev/null << 'EOF'
---
- name: Serial demo - 2 hosts at a time
  hosts: all
  gather_facts: false
  serial: 2
  tasks:
    - name: Batch task
      ansible.builtin.debug:
        msg: "Running on {{ inventory_hostname }} in batch"
    - name: Pause between batches
      ansible.builtin.pause:
        seconds: 2
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/serial_demo.yml
```

2. Create job template `Serial-Demo` with this playbook
3. **Launch** — observe output: hosts run in batches of 2

**Verify:** Tasks run in batches; `serial: 2` limits parallelism per task.

---

## Part 3 — Hands-On: Execution Control in AAP (30 minutes)

### Step 4 — View Instance Capacity and Used Capacity

**Purpose:** Understand how AAP tracks execution capacity.

**Detailed steps:**

1. Navigate to **Administration → Instances**
2. Click your instance (e.g., `aap1.lab.local`)
3. Note:
   - **Capacity** — total forks
   - **Used Capacity** — forks in use (0 when idle)
   - **Running Jobs** — number of active jobs
4. Launch a job — refresh the instance view; **Used Capacity** increases
5. When job completes — **Used Capacity** returns to 0

**Verify:** You understand capacity and how jobs consume it.

---

### Step 5 — Assign Job Template to Instance Group

**Purpose:** Pin a job template to a specific instance group.

**Detailed steps:**

1. If you have multiple instances (Lab 03): **Administration → Instance Groups**
2. Note groups: `default`, `controlplane`, and any custom (e.g., `dev-workers`)
3. Open a job template (e.g., `Ping-Lab`)
4. Find **Instance Groups** (or **Instances** in some UIs)
5. Add `default` or a custom group — job will run only on instances in that group
6. **Save** and **Launch** — job runs on the assigned instance(s)

**Minimal lab (single instance):** All groups include the control node; job still runs. The exercise demonstrates where to configure it.

**Verify:** Instance group is assigned; job runs successfully.

---

### Step 6 — Test Concurrent Job Limits

**Purpose:** Observe job queuing when capacity is exceeded.

**Detailed steps:**

1. Create a long-running playbook (e.g., `pause: seconds=120`)
2. Create template `Long-Run-Lab` — do **not** enable "Allow Simultaneous Jobs"
3. **Launch** the job — it starts running
4. While it runs, **Launch** again — second job should **queue** (Pending)
5. When first job completes, second starts
6. **Optional:** Enable "Allow Simultaneous Jobs" — both can run if capacity allows

**Verify:** You understand queuing when capacity or template limits apply.

---

## Part 4 — Hands-On: Performance Optimization (30 minutes)

### Step 7 — Optimize Fork Value for Your Inventory

**Purpose:** Set forks based on host count and capacity.

**Detailed steps:**

1. Count hosts in your inventory: **Resources → Inventories** → open inventory → **Hosts** tab
2. Edit a job template (e.g., `Ping-Lab`)
3. Set **Forks** to match or slightly exceed host count (e.g., 10 for 4 hosts)
4. **Save** and **Launch**
5. Compare elapsed time to previous runs (if you have baseline)

**Verify:** Forks is set appropriately for your environment.

---

### Step 8 — Disable Fact Gathering for Speed

**Purpose:** Skip fact gathering when not needed.

**Detailed steps:**

1. Create playbook `no_facts.yml`:

```yaml
---
- name: No facts - fast run
  hosts: all
  gather_facts: false
  tasks:
    - name: Quick task
      ansible.builtin.ping:
```

2. Create template `No-Facts-Demo` with this playbook
3. **Launch** — job should complete quickly (no setup phase)
4. Compare to a template that uses `gather_facts: true`

**Verify:** `gather_facts: false` reduces job time when facts are not used.

---

### Step 9 — Enable Pipelining (ansible.cfg)

**Purpose:** Improve SSH performance with pipelining.

**Detailed steps:**

1. SSH to AAP server (or control node if using CLI)
2. Edit project `ansible.cfg` or create one in project directory:

```ini
[defaults]
pipelining = True
```

3. **Important:** On managed nodes, ensure `requiretty` is disabled for sudo:

```bash
# On managed node
sudo visudo
# Ensure: Defaults !requiretty  or  user ALL=(ALL) NOPASSWD: ALL
```

4. Run a job that uses SSH — observe potential speed improvement

**Verify:** Pipelining is enabled; jobs run (and may run faster).

---

### Step 10 — Use Async Task for Long-Running Operation

**Purpose:** Run a long task asynchronously.

**Detailed steps:**

1. Create playbook `async_demo.yml`:

```yaml
---
- name: Async task demo
  hosts: all
  gather_facts: false
  tasks:
    - name: Long async task
      ansible.builtin.command: sleep 15
      async: 30
      poll: 5
      register: async_result
    - name: Show result
      ansible.builtin.debug:
        msg: "Async job completed: {{ async_result.finished }}"
```

2. Create template `Async-Demo` with this playbook
3. **Launch** — task runs in background; Ansible polls every 5 seconds
4. Job completes when async task finishes (within 30 seconds)

**Verify:** Async task runs; job completes successfully.

---

## Part 5 — Hands-On: Troubleshooting Execution Issues (25 minutes)

### Step 11 — Enable Verbose Output for Debugging

**Purpose:** Use verbosity to diagnose issues.

**Detailed steps:**

1. Edit a job template (e.g., `Ping-Lab`)
2. Enable **Prompt on Launch** for **Verbosity**
3. **Save**
4. **Launch** — when prompted, select **Verbosity 2** (or 3)
5. Run the job — output shows more detail (e.g., SSH connection info)

**Verify:** Verbose output appears in job log.

---

### Step 12 — Reproduce and Fix "Permission Denied" Error

**Purpose:** Troubleshoot a common execution failure.

**Detailed steps:**

1. Create a credential with wrong username (e.g., `wronguser` instead of actual SSH user)
2. Create a test template using this credential
3. **Launch** — job will fail with "Permission denied" or "UNREACHABLE"
4. Open job output — locate the error
5. Fix: use correct credential
6. Re-launch — job succeeds

**Verify:** You can identify and fix permission/credential errors.

---

### Step 13 — Reproduce and Fix Module Not Found Error

**Purpose:** Troubleshoot missing module/collection.

**Detailed steps:**

1. Create playbook with invalid module: `ansible.builtin.pingg` (typo)
2. Create template `Broken-Module-Demo`
3. **Launch** — job fails with "module not found"
4. Fix playbook: change to `ansible.builtin.ping`
5. Re-launch — job succeeds

**Verify:** You can diagnose and fix module errors.

---

### Step 14 — Use Check Mode to Validate Changes

**Purpose:** Run playbook in check mode without making changes.

**Detailed steps:**

1. Create a job template (or copy existing)
2. Set **Job Type** to **Check** (not Run)
3. **Save**
4. **Launch** — playbook runs in dry-run mode
5. Output shows "would create", "would change" instead of actual changes

**Verify:** Check mode runs without modifying hosts.

---

## Part 6 — Hands-On: Execution Environment Considerations (20 minutes)

### Step 15 — View and Select Execution Environment

**Purpose:** Understand how EE affects job execution.

**Detailed steps:**

1. Navigate to **Administration → Execution Environments**
2. Note available EEs (e.g., `Default execution environment`)
3. Open a job template
4. Find **Execution Environment** — note which EE is selected
5. **Optional:** If you have a custom EE (e.g., with `amazon.aws`), create a playbook that uses an AWS module and assign that EE to the template

**Verify:** You know where to view and change the execution environment.

---

### Step 16 — Identify Python/Collection Dependency Issue

**Purpose:** Recognize when EE lacks required content.

**Detailed steps:**

1. Create playbook that uses a collection not in default EE (e.g., `community.docker.docker_container` if not in default EE)
2. Create template with default EE
3. **Launch** — may fail with "module/collection not found"
4. **Fix:** Use an EE that includes the collection, or add collection to EE and rebuild

**Lab note:** If default EE has common collections, use a less common one (e.g., `community.general` subset) or document the pattern for real scenarios.

**Verify:** You understand that EE must contain required collections and Python packages.

---

## Part 7 — Real-Time Use Case Discussion (15 minutes)

### Case Study: Gaming Company — 5,000 Servers in Under 10 Minutes

**Background:**

A gaming company operates 5,000 game servers across multiple regions. They need to apply configuration updates (firewall rules, tuning parameters) during maintenance windows. Previously, playbooks took 45+ minutes, causing extended downtime.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| 5,000 hosts | Sequential execution would take hours |
| Short maintenance window | 15 minutes max |
| Network latency | Some regions have higher latency |
| Resource limits | Control node could not handle 5,000 parallel connections |

**Solution: Forks, Async, and Instance Groups**

#### 1. Fork Tuning

| Setting | Value | Rationale |
|---------|-------|-----------|
| **Forks** | 100 | Balance throughput vs. control node load |
| **Execution nodes** | 4 nodes, 50 forks each | Distribute load; total 200 concurrent |
| **Job slicing** | 25 slices (200 hosts each) | Each slice runs on one node; parallel slices |

#### 2. Async Tasks for Long Operations

- Config update tasks use `async: 600` and `poll: 10`
- Ansible does not block on each host; polls for completion
- Reduces wall-clock time when tasks have variable duration

#### 3. Fact Gathering Optimization

| Optimization | Effect |
|--------------|--------|
| **Fact cache** | First job gathers; subsequent jobs reuse |
| **gather_subset: min** | Only essential facts |
| **gather_facts: false** | For playbooks that do not need facts |

#### 4. Connection and SSH Tuning

- **Pipelining** enabled
- **SSH connection timeout** increased for high-latency regions
- **Forks** reduced for high-latency inventory groups

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Total time | 45+ minutes | Under 10 minutes |
| Maintenance window violations | Frequent | Zero |
| Failed runs (timeout) | 5–10% | < 1% |
| Ops team confidence | Low | High |

**Key takeaway:** Combining fork tuning, async tasks, fact optimization, and distributed execution (instance groups + job slicing) enables large-scale automation within tight time windows.

---

## Troubleshooting

### Jobs Queue Indefinitely

| Cause | Fix |
|-------|-----|
| No capacity | Add execution nodes; or reduce forks on running jobs |
| Instance unhealthy | Check **Administration → Instances**; restart receptor |
| Instance group misconfigured | Ensure template's instance group has healthy instances |

### High CPU/Memory on Control Node

| Cause | Fix |
|-------|-----|
| Forks too high | Reduce forks on job template |
| Too many concurrent jobs | Lower system/org job limits |
| Move execution to dedicated nodes | Use instance groups with execution nodes |

### "Module not found" in Job

| Cause | Fix |
|-------|-----|
| Collection not in EE | Add collection to EE; use ansible-builder to rebuild |
| Wrong EE selected | Select EE that has the collection |
| Typo in module name | Use FQCN: `collection.module` |

### Connection Timeout

| Cause | Fix |
|-------|-----|
| Network/firewall | Verify SSH/WinRM reachable |
| Slow hosts | Increase timeout in credential |
| Too many forks | Reduce forks to avoid connection storms |

### Pipelining Causes "sudo: sorry, you are not allowed"

| Cause | Fix |
|-------|-----|
| `requiretty` in sudoers | Add `Defaults !requiretty` or ensure NOPASSWD for user |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Forks | Parallel host execution; tune for inventory size |
| Connection types | SSH, WinRM, local |
| Serial | Batch execution for rolling updates |
| Instance capacity | Forks per instance; jobs consume capacity |
| Instance groups | Pin jobs to specific instances |
| Queue | Jobs wait when capacity is full |
| Fact optimization | gather_facts: false, fact cache, gather_subset |
| Pipelining | Reduces SSH round-trips |
| Async tasks | Background execution with poll |
| Verbosity | -v to -vvvv for debugging |
| Check mode | Dry run without changes |
| Execution Environment | Container with Ansible; affects modules/dependencies |

---

## Lab Completion Checklist

- [ ] Observed fork behavior (default and custom)
- [ ] Used serial for batch execution
- [ ] Viewed instance capacity and used capacity
- [ ] Assigned instance group to job template
- [ ] Tested concurrent job limits / queuing
- [ ] Optimized forks for inventory
- [ ] Disabled fact gathering for speed
- [ ] Enabled pipelining (if applicable)
- [ ] Used async task in playbook
- [ ] Enabled verbose output for debugging
- [ ] Troubleshot permission denied error
- [ ] Troubleshot module not found error
- [ ] Ran job in check mode
- [ ] Viewed and selected execution environment
- [ ] Can explain gaming company use case (5,000 servers in < 10 min)

---

## Quick Reference — Fork Guidelines

| Host Count | Suggested Forks |
|------------|-----------------|
| 1–10 | 5–10 |
| 10–50 | 10–25 |
| 50–200 | 25–50 |
| 200+ | 50–100; use execution nodes |

---

## Quick Reference — Common Execution Errors

| Error | Quick Fix |
|-------|-----------|
| UNREACHABLE | Check credential, network, firewall |
| Permission denied | Verify SSH user, key, sudoers |
| Module not found | Add collection to EE; use FQCN |
| Connection timeout | Increase timeout; reduce forks |
| No module named 'X' | Add Python package to EE |

---

## Quick Reference — Performance Tips

| Tip | Effect |
|-----|--------|
| Increase forks | Faster (up to capacity) |
| gather_facts: false | Skip setup when not needed |
| Fact cache | Reuse facts across jobs |
| Pipelining | Fewer SSH round-trips |
| Serial | Controlled rollout |
| Async | Non-blocking long tasks |

---

*End of Lab 13*
