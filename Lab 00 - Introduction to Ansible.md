# Lab 00 — Introduction to Automation and Ansible

| Field        | Value                                                              |
|--------------|--------------------------------------------------------------------|
| **Audience** | Beginner (System Admins / DevOps Engineers / IT Operations)        |
| **Duration** | 2.5–3 hours (instructor-led theory session)                       |
| **Outcome**  | Understand IT automation, Infrastructure as Code, Ansible architecture, Ansible Automation Platform components, and when to use Ansible Core vs. AAP |

---

## Section 1 — What Is IT Automation and Why It Matters

### What Is IT Automation?

**IT Automation** is the use of software tools and scripts to automatically perform repetitive IT tasks such as:

- Server provisioning
- Application deployment
- Configuration management
- Security patching
- Network setup
- Cloud resource management

Instead of manually logging into servers and running commands, automation executes predefined instructions consistently across any number of machines.

### Why IT Automation Matters (Enterprise View)

| Manual IT                | Automated IT                    |
|--------------------------|---------------------------------|
| Error-prone              | Consistent and repeatable       |
| Slow deployments         | Faster releases                 |
| Hard to scale            | Easily scalable                 |
| Depends on individuals   | Documented and version-controlled |

### Business Impact

- Faster time to market
- Reduced operational cost
- Improved reliability
- DevOps enablement
- Better compliance and auditability

> For large enterprise systems — multi-cloud, AKS/EKS, microservices environments — automation is non-negotiable.

---

## Section 2 — Introduction to Infrastructure as Code (IaC)

### What Is Infrastructure as Code?

**Infrastructure as Code (IaC)** means managing infrastructure (servers, networks, cloud resources) using code instead of manual processes.

**Without IaC (manual):**

> "Create 3 Ubuntu VMs, install Nginx, configure firewall"

**With IaC (Ansible playbook):**

```yaml
- name: Install Nginx
  hosts: webservers
  tasks:
    - name: Install package
      apt:
        name: nginx
        state: present
```

The code above achieves the same result — but it is repeatable, versionable, and auditable.

### Benefits of IaC

| Benefit                    | Description                                                |
|----------------------------|------------------------------------------------------------|
| Version control            | Store infrastructure definitions in Git                    |
| Repeatable environments    | Create identical Dev / Test / Prod environments            |
| Consistency                | Eliminate configuration drift between environments         |
| Disaster recovery          | Rebuild infrastructure from code in minutes                |
| Audit-friendly             | Every change is tracked in version control history         |

### Types of IaC Tools

| Tool Category            | Examples                  | Purpose                          |
|--------------------------|---------------------------|----------------------------------|
| Provisioning             | Terraform, CloudFormation | Create cloud resources           |
| Configuration Management | Ansible, Chef, Puppet     | Configure and maintain servers   |
| Image-based              | Packer                    | Build pre-configured VM images   |
| Container Orchestration  | Kubernetes                | Manage containerized workloads   |

---

## Section 3 — Evolution: Manual Configuration to Automation

### Phase 1 — Manual Configuration

The traditional approach:

- SSH into each server individually
- Install packages manually
- Edit config files by hand
- Hard to track what changed and when

**Problems:**

- No documentation trail
- Configuration drift (servers become "snowflakes")
- Environment mismatch between Dev, Test, and Prod

### Phase 2 — Shell Scripts

A step forward:

- Bash scripts (Linux) or PowerShell scripts (Windows)
- Some level of automation and repeatability

**Problems:**

- Hard to maintain as complexity grows
- Not idempotent (running twice may cause errors or duplicates)
- Platform-specific (a Bash script won't work on Windows)

### Phase 3 — Configuration Management Tools

The modern approach:

- **Declarative** — you describe the desired state, the tool figures out how to get there
- **Idempotent** — running the same automation multiple times produces the same result
- **Centralized orchestration** — manage thousands of servers from a single control point
- **Agentless or agent-based** — depending on the tool

This is where **Ansible** enters the picture.

---

## Section 4 — What Is Ansible and Its Use Cases

### What Is Ansible?

**Ansible** is an open-source IT automation tool created by Michael DeHaan in 2012, now maintained by **Red Hat**. It automates:

- Configuration management
- Application deployment
- Cloud provisioning
- Network automation
- Security automation

**Key characteristics:**

| Characteristic | Detail                                           |
|----------------|--------------------------------------------------|
| Agentless      | Uses SSH (Linux) or WinRM (Windows) — no agent to install on managed nodes |
| YAML-based     | Playbooks are written in human-readable YAML     |
| Idempotent     | Safe to run repeatedly — only changes what needs changing |
| Easy to learn  | Minimal learning curve compared to Chef or Puppet |

### How Ansible Works

```
┌──────────────────┐        SSH / WinRM        ┌──────────────────┐
│   CONTROL NODE   │ ────────────────────────▶  │  MANAGED NODE(s) │
│                  │                            │                  │
│  - Ansible CLI   │   Pushes tasks via         │  - No agent      │
│  - Playbooks     │   modules over SSH         │  - Only Python   │
│  - Inventory     │                            │    + SSH needed   │
│  - Modules       │                            │                  │
└──────────────────┘                            └──────────────────┘
```

| Component      | Purpose                                                  |
|----------------|----------------------------------------------------------|
| Control Node   | The machine where Ansible is installed and playbooks run |
| Managed Nodes  | Target servers that Ansible configures                   |
| Inventory      | A file listing all managed hosts, organized into groups  |
| Playbook       | A YAML file defining automation tasks                    |
| Modules        | Pre-built units of automation (e.g., `dnf`, `copy`, `service`) |

### Common Use Cases

| Use Case               | Example                                     |
|------------------------|---------------------------------------------|
| Server Provisioning    | Install Apache/Nginx on 100 servers at once |
| Application Deployment | Deploy a .NET or Java app to Linux servers  |
| Cloud Automation       | Create Azure VMs, AWS EC2 instances         |
| Kubernetes Automation  | Deploy Helm charts, manage namespaces       |
| Network Automation     | Configure Cisco, Juniper, or Arista devices |
| Security Hardening     | Enforce firewall rules, apply patches       |

---

## Section 5 — Ansible Core vs. Ansible Automation Platform

### Ansible Core

The **open-source**, community-driven version.

**Includes:**

- Command-line interface (`ansible`, `ansible-playbook`, `ansible-galaxy`)
- Playbooks and roles
- Modules and plugins
- Inventory management
- Ansible Vault (secret encryption)

**Best for:**

- Individual developers and small teams
- Learning and experimentation
- Basic automation tasks
- Budget-constrained environments

### Ansible Automation Platform (AAP)

The **enterprise** version from **Red Hat** (commercially supported).

**Includes everything in Ansible Core, plus:**

- Web UI (Automation Controller)
- Role-based access control (RBAC)
- Workflow automation (chain multiple playbooks)
- REST API integration
- Centralized logging and audit trail
- Automation Hub (curated content collections)
- Execution Environments (containerized runtime)
- Enterprise support from Red Hat

### Side-by-Side Comparison

| Feature                | Ansible Core   | Ansible Automation Platform |
|------------------------|----------------|-----------------------------|
| CLI                    | Yes            | Yes                         |
| Playbooks and Roles    | Yes            | Yes                         |
| Web UI                 | No             | Yes                         |
| Role-Based Access (RBAC) | No           | Yes                         |
| Workflow Engine        | No             | Yes                         |
| REST API               | No             | Yes                         |
| Centralized Logging    | No             | Yes                         |
| Enterprise Support     | Community only | Red Hat supported           |
| Audit and Compliance   | Limited        | Advanced                    |
| Scheduling             | Cron (manual)  | Built-in scheduler          |

---

## Section 6 — When to Use Ansible Core vs. Ansible Automation Platform

### Use Ansible Core When:

- You are learning automation
- You are a small team (1–5 people)
- Automation tasks are simple and ad-hoc
- No need for RBAC or approval workflows
- Budget is limited

### Use Ansible Automation Platform When:

- You are in a large enterprise with multiple teams
- You need RBAC and approval workflows
- Audit logs and compliance are required
- You need CI/CD pipeline integration
- Centralized execution and scheduling are needed
- You manage hundreds or thousands of servers

### Real-World Enterprise Scenario

Consider a banking or healthcare environment:

| Factor                  | Reality                                              |
|-------------------------|------------------------------------------------------|
| Server count            | 500+ servers across multiple data centers            |
| Environments            | Separate Dev, QA, Staging, and Production            |
| Teams                   | Multiple teams (networking, security, app, platform) |
| Compliance              | Strict audit requirements (PCI-DSS, HIPAA)           |
| Change management       | Approval workflows required before any change        |

In this scenario, CLI-only Ansible Core is not enough. You need:

- **Approval workflows** — so changes go through proper review
- **Credential vault** — so secrets are stored centrally and securely
- **Centralized logging** — so every action is auditable
- **Role-based access** — so teams only access what they should
- **Scheduling** — so patch windows are automated
- **API-based execution** — so CI/CD pipelines can trigger automation

This is exactly what **Ansible Automation Platform** provides.

---

## Section 7 — Ansible Automation Platform Architecture Overview

**Ansible Automation Platform** (AAP) is the enterprise automation suite from **Red Hat**, built on top of Ansible Core. It transforms simple CLI-based automation into a centralized, scalable, auditable platform with role-based execution and API-driven workflows.

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Users / API / CI-CD Pipelines              │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   AUTOMATION CONTROLLER                       │
│          (Web UI, REST API, RBAC, Workflows,                 │
│           Scheduling, Credentials, Audit Logs)               │
└──────────┬───────────────────────────────┬───────────────────┘
           │                               │
           ▼                               ▼
┌─────────────────────┐     ┌─────────────────────────────────┐
│   AUTOMATION HUB    │     │   EXECUTION ENVIRONMENTS (EE)   │
│  (Collections,      │     │  (Containerized runtime with    │
│   Roles, Certified  │     │   ansible-core, Python libs,    │
│   Content)          │     │   collections, dependencies)    │
└─────────────────────┘     └──────────────┬──────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────┐
│              MANAGED INFRASTRUCTURE                           │
│   (Servers, Cloud Resources, Network Devices, Kubernetes)    │
└──────────────────────────────────────────────────────────────┘

           ┌─────────────────────────────────┐
           │     EVENT-DRIVEN ANSIBLE (EDA)  │
           │  (Listens for events, triggers  │
           │   automation reactively)        │
           └─────────────────────────────────┘
```

### Core Components at a Glance

| Component              | Purpose                                                    |
|------------------------|------------------------------------------------------------|
| Automation Controller  | Orchestration and control plane — the "brain" of AAP       |
| Automation Hub         | Content repository for collections, roles, and EE images   |
| Execution Environments | Containerized, portable runtime for running automation     |
| Event-Driven Ansible   | Reactive automation engine — responds to real-time events  |
| Private Automation Hub | Self-hosted hub for enterprise content governance          |

Each component is explained in detail in the following sections.

---

## Section 8 — Automation Controller (Formerly Ansible Tower)

### What It Is

**Automation Controller** (formerly known as **Ansible Tower**) is the **control plane** of the Ansible Automation Platform. It is the central place where all automation is managed, executed, and audited.

It provides:

- A **Web UI** for managing automation visually
- A **REST API** for integration with CI/CD pipelines and external tools
- **Role-Based Access Control (RBAC)** for team-level permissions
- **Workflow orchestration** to chain multiple playbooks into sequences
- **Job scheduling** with cron-like capabilities
- **Credential management** with encrypted storage
- **Audit logging** of every action and job execution

### What It Solves

| Without Controller (CLI Only)         | With Controller                           |
|---------------------------------------|-------------------------------------------|
| Anyone can run any playbook, anywhere | Centralized execution with permissions    |
| No governance or approval process     | Workflow approvals and review gates       |
| No centralized logging                | Full audit trail of every job run         |
| Credentials stored in plain files     | Encrypted credential vault                |
| No scheduling — manual or cron only   | Built-in scheduling with timezone support |
| No visibility across teams            | Dashboard showing all activity            |

### Key Features

| Feature        | Purpose                                                  |
|----------------|----------------------------------------------------------|
| Job Templates  | Reusable job definitions — playbook + inventory + credential |
| Projects       | Git-linked source of playbooks (auto-synced from SCM)    |
| Inventories    | Managed host lists with groups, variables, and dynamic sources |
| Credentials    | Securely stored SSH keys, API tokens, cloud credentials  |
| Workflows      | Multi-step automation — chain templates with conditions  |
| Schedules      | Time-based job execution (daily patches, weekly reports) |
| RBAC           | Control who can view, edit, or execute each resource     |
| Notifications  | Send alerts via email, Slack, webhook on job status      |

### Enterprise Example

Consider an organization with 500 Linux servers across Dev, QA, and Production environments, where the security team must approve changes before they reach Production:

- **RBAC** ensures only the DevOps team can deploy to Production
- **Workflows** route deployments through Dev --> QA --> approval gate --> Production
- **Credential vault** stores SSH keys encrypted — no one sees the raw key
- **Audit logs** record who ran what, when, and what changed

---

## Section 9 — Automation Hub

### What It Is

**Automation Hub** is the **content repository** for Ansible automation. It stores and distributes:

- **Collections** — packaged sets of modules, roles, and plugins
- **Roles** — reusable automation units
- **Certified content** — Red Hat and partner-tested automation
- **Execution Environment images** — container images for runtime

Think of it as an **enterprise app store for Ansible content** — curated, versioned, and governed.

### Types of Hubs

| Type                   | Description                                              |
|------------------------|----------------------------------------------------------|
| Public Automation Hub  | Hosted by Red Hat at `console.redhat.com` — certified and supported collections |
| Private Automation Hub | Self-hosted within your enterprise — internal content governance |
| Ansible Galaxy         | Community-hosted (open-source) — not certified, not governed |

### Why It Matters in Enterprise

In enterprise environments, you cannot use arbitrary open-source roles from the internet without review. Automation Hub provides:

| Need                     | How Automation Hub Addresses It                        |
|--------------------------|--------------------------------------------------------|
| Content governance       | Only approved collections are available to teams       |
| Version control          | Pin specific versions of collections for stability     |
| Security                 | Content is scanned and certified by Red Hat / partners |
| Certified integrations   | Pre-built collections for AWS, Azure, VMware, SAP, ServiceNow |
| Internal content sharing | Teams publish internal roles to Private Hub for reuse  |

### Example: Using Certified Collections

Instead of writing custom modules for cloud automation, teams use certified collections:

| Collection               | Purpose                          |
|--------------------------|----------------------------------|
| `amazon.aws`             | Manage AWS resources             |
| `azure.azcollection`     | Manage Azure resources           |
| `vmware.vmware_rest`     | Manage VMware vSphere            |
| `cisco.ios`              | Configure Cisco IOS devices      |
| `servicenow.itsm`        | Integrate with ServiceNow ITSM  |

These are maintained, tested, and supported — reducing the risk of using unvetted community code.

---

## Section 10 — Execution Environments (EE)

### What Are Execution Environments?

Execution Environments are **containerized runtime environments** for running Ansible automation. They package everything needed to execute a playbook into a single, portable container image.

### The Problem They Solve

| Before EEs (Traditional)                    | With EEs (Modern)                          |
|---------------------------------------------|--------------------------------------------|
| Python dependency conflicts on control node | All dependencies bundled in a container    |
| "Works on my machine" problems              | Identical runtime everywhere               |
| Different teams have different environments | Standardized, reproducible execution       |
| Manual dependency management                | Automated image builds                     |
| Hard to scale                               | Cloud-native, horizontally scalable        |

### What Is Inside an Execution Environment?

An EE container image includes:

| Component          | Purpose                                                  |
|--------------------|----------------------------------------------------------|
| `ansible-core`     | The Ansible engine itself                                |
| Python libraries   | All required Python packages (boto3, azure-cli, etc.)    |
| Collections        | Pre-installed Ansible collections for the use case       |
| System packages    | Any OS-level dependencies the modules need               |
| Custom scripts     | Organization-specific utilities if needed                |

### How EEs Are Built

EEs are built using **`ansible-builder`**, a tool that reads a definition file and produces an OCI-compatible container image:

```yaml
# execution-environment.yml (definition file)
version: 3
dependencies:
  galaxy:
    collections:
      - amazon.aws
      - azure.azcollection
  python:
    - boto3
    - azure-identity
  system:
    - gcc
```

The image is built with **Podman** or Docker and can be stored in any container registry (Quay, Private Hub, or a corporate registry).

### Enterprise Impact

Organizations can build purpose-specific EEs for different teams:

| EE Name               | Contains                             | Used By          |
|------------------------|--------------------------------------|------------------|
| `ee-cloud-aws`         | AWS collections + boto3              | Cloud team       |
| `ee-cloud-azure`       | Azure collections + azure-identity   | Cloud team       |
| `ee-network`           | Cisco, Juniper, Arista collections   | Network team     |
| `ee-security`          | CIS benchmarks, firewall modules     | Security team    |
| `ee-general`           | Common modules for server management | All teams        |

Each EE runs in isolation — no dependency conflicts, no drift, fully reproducible.

---

## Section 11 — Event-Driven Ansible (EDA)

### What Is Event-Driven Ansible?

Event-Driven Ansible enables **reactive automation** — instead of running automation on a schedule or manually, it responds to real-time events as they happen.

**Traditional automation:**

> "Run this playbook every 6 hours"

**Event-driven automation:**

> "When a monitoring alert fires, automatically run this playbook"

### How It Works

```
Event Source  ──▶  Rulebook  ──▶  Condition Match  ──▶  Action (Run Playbook)
```

| Step           | Description                                              |
|----------------|----------------------------------------------------------|
| Event Source   | Where events come from — webhooks, monitoring tools, message queues, cloud events |
| Rulebook       | A YAML file defining rules — "when X happens, do Y"     |
| Condition      | The matching logic — filter events by type, severity, etc. |
| Action         | What to do — run a playbook, call an API, send a notification |

### Example: Auto-Remediation

Scenario: A server's CPU exceeds 90%, and the monitoring tool sends an alert.

```yaml
# rulebook.yml
---
- name: Auto-remediate high CPU
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  rules:
    - name: Restart service on high CPU
      condition: event.payload.alert_name == "HighCPU"
      action:
        run_playbook:
          name: remediate_cpu.yml
```

What happens:

1. Monitoring tool (Prometheus, Zabbix, Datadog) detects CPU > 90%
2. It sends a webhook to Event-Driven Ansible on port 5000
3. EDA matches the event against the rulebook
4. EDA triggers `remediate_cpu.yml` which restarts the offending service
5. All of this happens automatically — no human intervention needed

### Enterprise Use Cases

| Use Case              | Trigger                   | Automated Response                     |
|-----------------------|---------------------------|----------------------------------------|
| Security incident     | IDS alert (compromised host) | Isolate host, block IP, open ticket |
| Infrastructure scaling | Cloud metric threshold     | Scale up VMs or containers            |
| ITSM integration      | Service failure alert      | Create ServiceNow incident automatically |
| Self-healing          | Application crash detected | Restart service, notify team via Slack |
| Compliance drift      | Configuration scan failure | Re-apply hardened configuration        |

---

## Section 12 — How All AAP Components Work Together

### End-to-End Workflow

Here is how the components connect in a real enterprise workflow:

```
Step 1:  Developer pushes code to Git repository
              │
              ▼
Step 2:  Automation Controller detects the change (SCM polling or webhook)
              │
              ▼
Step 3:  Controller pulls approved collections from Automation Hub
              │
              ▼
Step 4:  Controller selects the appropriate Execution Environment
              │
              ▼
Step 5:  EE runs the playbook against target infrastructure
              │
              ▼
Step 6:  Results are logged centrally in the Controller (audit trail)
              │
              ▼
Step 7:  If an event occurs (failure, alert) → Event-Driven Ansible
         triggers the next remediation action automatically
```

### Component Interaction Map

| Component              | Role in the Workflow                                      |
|------------------------|-----------------------------------------------------------|
| Automation Controller  | Orchestrates everything — decides what runs, where, when  |
| Automation Hub         | Supplies approved, versioned content (collections, roles) |
| Execution Environment  | Provides the isolated runtime to execute automation       |
| Event-Driven Ansible   | Listens for events and triggers automation reactively     |
| Managed Infrastructure | The target — servers, cloud, network devices, Kubernetes  |

### Real Enterprise Architecture Scenario

Consider a large financial institution:

| Factor         | Detail                                                    |
|----------------|-----------------------------------------------------------|
| Scale          | 1000+ servers across 3 data centers and 2 cloud providers |
| Teams          | 10 teams — networking, security, cloud, application, DBA  |
| Cloud          | Multi-cloud (AWS + Azure)                                 |
| Compliance     | PCI-DSS, SOX audit requirements                          |
| Change control | All changes require approval before reaching Production   |

How they use each AAP component:

| Component              | How It Is Used                                            |
|------------------------|-----------------------------------------------------------|
| Automation Hub         | Stores certified AWS, Azure, and VMware collections. Internal roles for CIS hardening are published to Private Hub. |
| Automation Controller  | RBAC ensures each team only accesses their resources. Workflows enforce Dev --> QA --> Approval --> Prod pipeline. |
| Execution Environments | Separate EEs for cloud, network, and security teams — no dependency conflicts. |
| Event-Driven Ansible   | Security alerts from the SIEM trigger automatic host isolation. Failed health checks trigger self-healing playbooks. |

Together, these components form a **complete enterprise automation ecosystem** — governed, scalable, auditable, and reactive.

---

## Summary

| Concept                   | Key Takeaway                                                |
|---------------------------|-------------------------------------------------------------|
| IT Automation             | Eliminates manual errors and enables scale                  |
| Infrastructure as Code    | Manage infrastructure as versionable, repeatable code       |
| Evolution                 | Manual --> Shell Scripts --> Configuration Management Tools  |
| Ansible                   | Agentless, YAML-based, idempotent automation tool           |
| Ansible Core              | Open-source CLI version — great for learning and small teams |
| Ansible Automation Platform | Enterprise platform with UI, RBAC, workflows, and audit   |
| When to Use AAP           | Large-scale, multi-team, compliance-driven environments     |
| Automation Controller     | The control plane — Web UI, API, RBAC, workflows, audit    |
| Automation Hub            | Content governance — certified collections and roles        |
| Execution Environments    | Containerized, portable, consistent runtime for automation  |
| Event-Driven Ansible      | Reactive automation triggered by real-time events           |

---

## What Comes Next

| Next Lab | Topic |
|----------|-------|
| **Lab 01** | Install Ansible Core on a RHEL machine, set up SSH, create a project, write playbooks, install Nginx, and deploy a custom web page |
| **Lab 02** | Install Ansible Automation Platform (standalone), configure the web UI, create inventories, credentials, projects, and run your first job template |

---

*End of Lab 00*
