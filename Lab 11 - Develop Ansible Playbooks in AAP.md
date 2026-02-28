# Lab 11 — Develop Ansible Playbooks in AAP

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 30% / Hands-on: 70%)                                      |
| **Outcome**  | Write playbooks from scratch; use variables, facts, loops, conditionals; apply best practices (idempotency, handlers, error handling); use block/rescue, delegation, run_once; integrate with AAP (surveys, extra vars, Vault, artifacts) |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Labs 02, 04, 05, 07 completed — AAP 2.4 installed, project, inventory, job template, machine credential. |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                        │
│                         https://aap1.lab.local                                       │
│                         + Code editor (VS Code, etc.)                                │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│  AAP CONTROLLER         │  │  PROJECT (lab-playbooks)│  │  MANAGED NODES           │
│  - Job templates        │  │  - playbooks/*.yml      │  │  - web1, app1, db1       │
│  - Surveys, extra vars  │  │  - group_vars/         │  │  - Lab-Static-Inventory   │
│  - Vault credentials    │  │  - files/, templates/  │  │                          │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` with `lab-playbooks` (from Lab 04/05) |
| Inventory | `Lab-Static-Inventory` with hosts (webservers, appservers, databases) |
| Credential | Machine credential (SSH key) |
| Managed Nodes | 2–4 Linux VMs reachable via SSH |

---

## Part 1 — Theory: Ansible Playbook Fundamentals (45 minutes)

### Section 10.1 — Ansible Playbook Fundamentals

#### What Is a Playbook?

A **playbook** is a YAML file that defines automation: which hosts to target, what tasks to run, and in what order. It is the primary unit of automation in Ansible and AAP.

| Concept | Description |
|---------|-------------|
| **Play** | A set of tasks run on a set of hosts |
| **Task** | A single unit of work (calls a module) |
| **Module** | Reusable code that performs an action (e.g., copy file, install package) |

#### YAML Syntax Basics for Beginners

| Rule | Example |
|------|---------|
| **Indentation** | Use spaces (2 or 4); no tabs |
| **Key-value** | `key: value` |
| **Lists** | `- item1` or `[item1, item2]` |
| **Strings** | `"quoted"` or `unquoted`; use `\|` for multiline |
| **Comments** | `# comment` |

#### Playbook Structure: Plays and Tasks

```yaml
---
- name: First play
  hosts: webservers
  tasks:
    - name: Task 1
      ansible.builtin.debug:
        msg: "Hello"

- name: Second play
  hosts: databases
  tasks:
    - name: Task 2
      ansible.builtin.ping:
```

#### Understanding Modules

- **FQCN:** `ansible.builtin.copy` (collection.module)
- **Parameters:** Passed as key-value under the module name
- **Idempotent:** Running twice should produce the same result (no unnecessary changes)

#### Common Modules for Beginners

| Module | Purpose |
|--------|---------|
| `ansible.builtin.copy` | Copy file; create from content |
| `ansible.builtin.file` | Create/delete files, directories; set permissions |
| `ansible.builtin.service` | Start, stop, enable services |
| `ansible.builtin.package` | Install packages (auto-detects dnf/apt) |
| `ansible.builtin.dnf` / `apt` | Package management (RHEL / Debian) |
| `ansible.builtin.user` | Create, modify, remove users |
| `ansible.builtin.template` | Render Jinja2 template |
| `ansible.builtin.debug` | Print messages (for learning) |

#### Writing Your First Playbook

```yaml
---
- name: My first playbook
  hosts: all
  become: true
  tasks:
    - name: Ensure directory exists
      ansible.builtin.file:
        path: /opt/myapp
        state: directory
        mode: "0755"
    - name: Display hostname
      ansible.builtin.debug:
        msg: "Running on {{ inventory_hostname }}"
```

---

### Section 10.2 — Variables and Facts

#### What Are Variables?

Variables store values used in playbooks: `{{ variable_name }}`. They can come from inventory, play `vars`, `vars_files`, `set_fact`, or `register`.

#### Defining and Using Variables

| Location | Example |
|----------|---------|
| **Play vars** | `vars: { app_port: 8080 }` |
| **Inventory** | Host/group variables |
| **Extra vars** | Job template or `-e` at CLI |
| **Facts** | `{{ ansible_hostname }}` |

#### Variable Naming Conventions

- Use lowercase, underscores: `app_version`, `db_host`
- Avoid names starting with `ansible_` (reserved for facts)

#### Facts: Automatically Gathered Information

- **gather_facts: true** (default) runs `setup` module
- Facts: `ansible_hostname`, `ansible_os_family`, `ansible_distribution_major_version`, `ansible_memtotal_mb`, etc.
- Use: `{{ ansible_hostname }}`, `{{ ansible_os_family }}`

#### Registering Task Outputs

```yaml
- name: Get disk usage
  ansible.builtin.shell: df -h /
  register: disk_result
- name: Display usage
  ansible.builtin.debug:
    msg: "{{ disk_result.stdout }}"
```

#### Using Variables in Playbooks

- **Jinja2:** `{{ var }}`, `{{ var | default('fallback') }}`, `{{ list | join(',') }}`
- **Conditionals:** `when: var == 'value'`

---

### Section 10.3 — Playbook Best Practices for AAP

#### Idempotency Explained (Why It Matters)

- **Idempotent:** Running the playbook multiple times produces the same end state
- Use `state: present` (not `state: latest` for packages if you want predictability)
- Avoid `shell`/`command` when a declarative module exists
- **Why:** Safe re-runs; no unintended side effects; check mode works

#### Using Check Mode for Testing

- **Check mode** (`--check`): Simulates changes without applying them
- Use before production runs
- In AAP: Create a job template with **Job Type: Check**

#### Error Handling with ignore_errors and failed_when

| Option | Purpose |
|--------|---------|
| `ignore_errors: true` | Continue playbook even if task fails |
| `failed_when: condition` | Fail only when condition is true |
| `changed_when: condition` | Report "changed" only when condition is true |

#### Handlers for Service Restarts

- **Handlers** run once at end of play, only if notified
- Use for service restarts after config changes

```yaml
handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted

tasks:
  - name: Update nginx config
    ansible.builtin.copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx
```

#### Using when Conditionals

```yaml
- name: Install on RHEL
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

- name: Install on Debian
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"
```

#### Loops for Repetitive Tasks

```yaml
- name: Create users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - charlie
```

---

### Section 10.4 — Advanced Playbook Constructs

#### Block, Rescue, and Always for Error Handling

```yaml
- block:
    - name: Risky task
      ansible.builtin.shell: /opt/risky-script.sh
  rescue:
    - name: On failure
      ansible.builtin.debug:
        msg: "Task failed, running recovery"
  always:
    - name: Always run
      ansible.builtin.debug:
        msg: "Cleanup"
```

#### Delegation: Running Tasks on Different Hosts

- **delegate_to:** Run task on a different host (e.g., controller, load balancer)

```yaml
- name: Add host to load balancer
  ansible.builtin.uri:
    url: "http://lb.local/add/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost
```

#### Running Tasks Once with run_once

- **run_once: true** — Run on one host only (e.g., primary DB)

```yaml
- name: Initialize database (once)
  ansible.builtin.mysql_db:
    name: myapp
    state: present
  run_once: true
```

#### Asynchronous Tasks for Long-Running Operations

```yaml
- name: Long-running task
  ansible.builtin.command: /opt/long-script.sh
  async: 3600
  poll: 60
```

#### Serial Execution for Controlled Rollouts

```yaml
- name: Rolling update
  hosts: webservers
  serial: 2
  tasks:
    - name: Deploy
      ansible.builtin.copy:
        src: app.war
        dest: /opt/app/
```

---

### Section 10.5 — Working with AAP-Specific Features

#### Using Survey Variables in Playbooks

- Survey answers become **extra variables**
- Use in playbook: `{{ app_version }}`, `{{ env }}`
- Define in job template **Survey** tab

#### Extra Variables from Job Templates

- **Extra Variables** in job template override inventory vars
- Pass at launch (Prompt on Launch) or set in template
- Use: `{{ my_var }}` in playbook

#### Using Ansible Vault for Sensitive Data

- Encrypt files: `ansible-vault encrypt secrets.yml`
- In AAP: Add **Ansible Vault** credential with vault password
- Attach to job template; AAP decrypts at runtime

#### Artifacts for Workflow Communication

- Use **set_stats** in playbook to pass data to downstream workflow nodes
- `ansible.builtin.set_stats: data: { key: value }`
- Downstream jobs receive as extra vars

#### Debugging Playbooks in AAP

- **Verbosity:** Enable Prompt on Launch for Verbosity (0–4)
- **Check mode:** Run as Check job type
- **Limit:** Target one host for faster iteration
- **Job output:** View full task output, including `msg` from debug

---

## Part 2 — Hands-On: Your First Playbook (25 minutes)

### Step 1 — Create a Simple Playbook

**Purpose:** Write a minimal playbook and run it via AAP.

**Detailed steps:**

1. **SSH to AAP server** (or use Git and push to project):

```bash
sudo mkdir -p /var/lib/awx/projects/lab-playbooks
sudo tee /var/lib/awx/projects/lab-playbooks/first_playbook.yml > /dev/null << 'EOF'
---
- name: My first playbook
  hosts: all
  gather_facts: true
  tasks:
    - name: Display host info
      ansible.builtin.debug:
        msg: "Hello from {{ inventory_hostname }} ({{ ansible_os_family }})"
    - name: Ensure directory exists
      ansible.builtin.file:
        path: /opt/lab-app
        state: directory
        mode: "0755"
EOF
sudo chown -R awx:awx /var/lib/awx/projects/lab-playbooks
```

2. **Sync project** in AAP: **Resources → Projects** → `Lab-Project` → **Sync**
3. **Create job template:** **Resources → Templates** → **Add** → **Add job template**
   - **Name:** `First-Playbook`
   - **Inventory:** `Lab-Static-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `first_playbook.yml`
   - **Credentials:** Machine credential
4. **Launch** and verify success.

---

## Part 3 — Hands-On: Variables, Facts, and Loops (35 minutes)

### Step 2 — Use Variables and Facts

**Purpose:** Define variables and use facts in a playbook.

**Detailed steps:**

1. **Create** `variables_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/variables_playbook.yml > /dev/null << 'EOF'
---
- name: Variables and facts demo
  hosts: all
  gather_facts: true
  vars:
    app_user: appadmin
    app_dir: /opt/myapp
  tasks:
    - name: Display variables
      ansible.builtin.debug:
        msg: "User={{ app_user }}, Dir={{ app_dir }}, Host={{ inventory_hostname }}"
    - name: Display facts
      ansible.builtin.debug:
        msg: "OS={{ ansible_os_family }}, Mem={{ ansible_memtotal_mb }}MB"
    - name: Create app directory
      ansible.builtin.file:
        path: "{{ app_dir }}"
        state: directory
        mode: "0755"
    - name: Create user
      ansible.builtin.user:
        name: "{{ app_user }}"
        state: present
        system: true
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/variables_playbook.yml
```

2. **Sync** and create job template `Variables-Playbook`. **Launch**.

### Step 3 — Register Output and Use in Tasks

**Purpose:** Capture task output and use it later.

**Detailed steps:**

1. **Create** `register_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/register_playbook.yml > /dev/null << 'EOF'
---
- name: Register and use output
  hosts: all
  gather_facts: true
  tasks:
    - name: Get hostname
      ansible.builtin.command: hostname
      register: hostname_result
    - name: Display hostname
      ansible.builtin.debug:
        msg: "Hostname is {{ hostname_result.stdout }}"
    - name: Get disk usage
      ansible.builtin.shell: df -h / | tail -1 | awk '{print $5}'
      register: disk_result
    - name: Display disk usage
      ansible.builtin.debug:
        msg: "Root disk {{ disk_result.stdout }} used"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/register_playbook.yml
```

2. **Sync**, create template, **Launch**.

### Step 4 — Use Loops

**Purpose:** Iterate over a list of items.

**Detailed steps:**

1. **Create** `loops_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/loops_playbook.yml > /dev/null << 'EOF'
---
- name: Loops demo
  hosts: all
  become: true
  vars:
    packages:
      - vim
      - htop
      - curl
  tasks:
    - name: Install packages
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
    - name: Create directories
      ansible.builtin.file:
        path: "/opt/{{ item }}"
        state: directory
        mode: "0755"
      loop:
        - data
        - logs
        - config
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/loops_playbook.yml
```

2. **Sync**, create template, **Launch**.

---

## Part 4 — Hands-On: Best Practices (40 minutes)

### Step 5 — Use when Conditionals and Handlers

**Purpose:** Conditional execution and handler notification.

**Detailed steps:**

1. **Create** `nginx_install.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/nginx_install.yml > /dev/null << 'EOF'
---
- name: Install and configure Nginx
  hosts: webservers
  become: true
  vars:
    nginx_port: 80
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
  tasks:
    - name: Install Nginx (RHEL)
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"
    - name: Install Nginx (Debian)
      ansible.builtin.apt:
        name: nginx
        state: present
      when: ansible_os_family == "Debian"
    - name: Ensure Nginx is started
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
    - name: Create custom index
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: |
          <h1>Hello from {{ inventory_hostname }}</h1>
      notify: Restart nginx
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/nginx_install.yml
```

2. **Sync**, create template targeting `webservers`, **Launch**.

### Step 6 — Use ignore_errors and failed_when

**Purpose:** Control task failure behavior.

**Detailed steps:**

1. **Create** `error_handling.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/error_handling.yml > /dev/null << 'EOF'
---
- name: Error handling demo
  hosts: all
  tasks:
    - name: Task that may fail (ignore)
      ansible.builtin.command: /bin/false
      ignore_errors: true
    - name: Check file exists
      ansible.builtin.stat:
        path: /etc/hostname
      register: hostname_stat
    - name: Fail only if file missing
      ansible.builtin.fail:
        msg: "hostname file not found"
      when: not hostname_stat.stat.exists
    - name: Always runs
      ansible.builtin.debug:
        msg: "Error handling demo complete"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/error_handling.yml
```

2. **Sync**, create template, **Launch** — playbook should complete despite first task failing.

### Step 7 — Run in Check Mode

**Purpose:** Validate playbook without making changes.

**Detailed steps:**

1. **Create job template** `Nginx-Check` — same as Nginx install but **Job Type: Check**
2. **Launch** — tasks show "would change" instead of actually changing; no modifications on hosts.

---

## Part 5 — Hands-On: Advanced Constructs (35 minutes)

### Step 8 — Use Block, Rescue, Always

**Purpose:** Structured error handling.

**Detailed steps:**

1. **Create** `block_rescue.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/block_rescue.yml > /dev/null << 'EOF'
---
- name: Block, rescue, always demo
  hosts: all
  tasks:
    - block:
        - name: Create temp file
          ansible.builtin.tempfile:
            state: file
          register: tmpfile
        - name: Simulate failure
          ansible.builtin.command: /bin/false
      rescue:
        - name: Rescue - log failure
          ansible.builtin.debug:
            msg: "Block failed, running rescue"
      always:
        - name: Always - cleanup message
          ansible.builtin.debug:
            msg: "Always block ran"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/block_rescue.yml
```

2. **Sync**, create template, **Launch**.

### Step 9 — Use run_once and delegate_to

**Purpose:** Run task on one host or different host.

**Detailed steps:**

1. **Create** `delegate_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/delegate_playbook.yml > /dev/null << 'EOF'
---
- name: Delegate and run_once demo
  hosts: all
  tasks:
    - name: Run on controller (delegate_to)
      ansible.builtin.debug:
        msg: "This ran on {{ inventory_hostname }} (delegated to controller)"
      delegate_to: localhost
    - name: Run once (first host only)
      ansible.builtin.debug:
        msg: "This ran once on {{ inventory_hostname }}"
      run_once: true
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/delegate_playbook.yml
```

2. **Sync**, create template, **Launch** — observe delegate and run_once behavior.

### Step 10 — Use Serial for Rolling Update

**Purpose:** Control parallelism (e.g., 2 hosts at a time).

**Detailed steps:**

1. **Create** `serial_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/serial_playbook.yml > /dev/null << 'EOF'
---
- name: Serial execution demo
  hosts: all
  serial: 2
  tasks:
    - name: Simulate update
      ansible.builtin.debug:
        msg: "Updating {{ inventory_hostname }}"
    - name: Pause between batches
      ansible.builtin.pause:
        seconds: 2
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/serial_playbook.yml
```

2. **Sync**, create template, **Launch** — with 4 hosts, 2 run first, then next 2.

---

## Part 6 — Hands-On: AAP-Specific Features (35 minutes)

### Step 11 — Use Survey Variables in Playbook

**Purpose:** Pass user input from job template survey to playbook.

**Detailed steps:**

1. **Create** `survey_playbook.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/survey_playbook.yml > /dev/null << 'EOF'
---
- name: Survey variables demo
  hosts: all
  vars:
    app_version: "{{ app_version | default('1.0') }}"
    env: "{{ env | default('dev') }}"
  tasks:
    - name: Display survey values
      ansible.builtin.debug:
        msg: "Deploying version {{ app_version }} to {{ env }}"
    - name: Create env file
      ansible.builtin.copy:
        dest: /tmp/deploy-info.txt
        content: |
          version={{ app_version }}
          env={{ env }}
          host={{ inventory_hostname }}
        mode: "0644"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/survey_playbook.yml
```

2. **Create job template** `Survey-Playbook` with this playbook
3. **Survey tab** — Enable survey; add:
   - **app_version** (Text, default: 1.0)
   - **env** (Multiple choice: dev, staging, prod)
4. **Launch** — fill survey; verify playbook receives values.

### Step 12 — Use set_stats for Workflow Artifacts

**Purpose:** Pass data to downstream workflow nodes.

**Detailed steps:**

1. **Create** `set_stats_producer.yml`:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/set_stats_producer.yml > /dev/null << 'EOF'
---
- name: Produce workflow artifact
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Set artifact for downstream
      ansible.builtin.set_stats:
        data:
          deploy_id: "{{ 10000 | random }}"
          deploy_time: "{{ ansible_date_time.iso8601 }}"
EOF
sudo chown awx:awx /var/lib/awx/projects/lab-playbooks/set_stats_producer.yml
```

2. Use in a workflow (see Lab 08) — downstream job receives `deploy_id` and `deploy_time`.

### Step 13 — Debug Playbook in AAP

**Purpose:** Use verbosity and limit for troubleshooting.

**Detailed steps:**

1. **Edit** a job template → **Options** → enable **Prompt on Launch** for **Limit** and **Verbosity**
2. **Launch** — set Limit to one host (e.g., `web1.lab.local`), Verbosity to 2
3. **Review** job output — more detailed Ansible output for debugging

---

## Part 7 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Web Hosting Company — Standardized Customer Onboarding

**Background:**

A web hosting company onboarded new customers manually: server setup, application deployment, monitoring configuration. Each onboarding took ~2 days and was error-prone.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Manual process | 2 days per customer; inconsistent setup |
| No standardization | Different admins did it differently |
| Errors | Wrong configs; security gaps |
| Scale | Cannot onboard 10+ customers/week |

**Solution: Standardized Playbooks in AAP**

| Component | Implementation |
|-----------|----------------|
| **Server setup playbook** | Create user, install packages, configure SSH, firewall |
| **Application deployment** | Deploy from Git; configure app; start service |
| **Monitoring playbook** | Install agent; configure alerts; register with monitoring |
| **Workflow** | Server setup → App deploy → Monitoring (with approval between) |
| **Survey** | Customer name, domain, app type, environment (dev/staging/prod) |
| **Variables** | `customer_name`, `domain`, `app_type` from survey |
| **Idempotency** | All playbooks idempotent; safe to re-run |

**Playbook structure:**

```yaml
# server_setup.yml
- hosts: "{{ target_hosts | default('all') }}"
  vars:
    customer: "{{ customer_name }}"
    app_type: "{{ app_type | default('webapp') }}"
  tasks:
    - name: Create customer user
      ansible.builtin.user: ...
    - name: Install base packages
      ansible.builtin.package: ...
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Onboarding time | 2 days | 30 minutes |
| Consistency | Low | High (playbook-driven) |
| Errors | Frequent | Rare (validated playbooks) |
| Capacity | 2–3/week | 15+/week |

**Key takeaway:** Standardized playbooks with surveys enable self-service, repeatable onboarding. Idempotency and error handling make them safe for production.

---

## Troubleshooting

### Playbook Fails — "module not found"

| Symptom | Check |
|---------|-------|
| Collection missing | Add to `collections/requirements.yml`; sync project |
| Wrong FQCN | Use `ansible.builtin.module` or `collection.module` |

### Survey Variable Undefined

| Symptom | Check |
|---------|-------|
| Variable not passed | Use `{{ var | default('value') }}` for optional vars |
| Survey not shown | Enable survey; add questions; save |
| Wrong name | Survey "Answer Variable Name" must match playbook |

### Handler Not Firing

| Symptom | Check |
|---------|-------|
| Handler not notified | Task must have `notify: HandlerName` |
| No change | Handler runs only if notifying task reports "changed" |
| Flush handlers | Use `meta: flush_handlers` to run handlers mid-play |

### Check Mode Shows "Would" but Job Type Wrong

- Ensure **Job Type** is **Check**, not Run.

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Playbook structure | Plays, tasks, modules |
| Variables | vars, facts, register, extra vars |
| Loops | `loop`, `with_items` |
| Conditionals | `when` |
| Handlers | `notify`, `handlers` |
| Error handling | `ignore_errors`, `failed_when`, block/rescue |
| Delegation | `delegate_to`, `run_once` |
| Serial | Rolling updates |
| Survey | Pass user input to playbook |
| set_stats | Workflow artifacts |

---

## Lab Completion Checklist

- [ ] First playbook created and run via AAP
- [ ] Variables and facts playbook created
- [ ] Register and loops playbooks created
- [ ] Nginx install with handlers and when
- [ ] Error handling playbook (ignore_errors, failed_when)
- [ ] Check mode job template created and run
- [ ] Block/rescue/always playbook created
- [ ] delegate_to and run_once playbook created
- [ ] Serial playbook created
- [ ] Survey playbook with job template survey
- [ ] set_stats playbook for workflow
- [ ] Can explain the web hosting use case

---

## Quick Reference — Common Modules

| Module | Purpose |
|--------|---------|
| `ansible.builtin.copy` | Copy file; create from content |
| `ansible.builtin.file` | Directory, file, permissions |
| `ansible.builtin.template` | Jinja2 template |
| `ansible.builtin.service` | Start, stop, restart service |
| `ansible.builtin.package` | Install package (generic) |
| `ansible.builtin.user` | Create/modify user |
| `ansible.builtin.debug` | Print message |
| `ansible.builtin.set_fact` | Set variable |
| `ansible.builtin.set_stats` | Workflow artifact |

---

*End of Lab 11*
