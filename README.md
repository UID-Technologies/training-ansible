# Ansible Automation Platform — Training Course Curriculum

A comprehensive, hands-on training curriculum covering **Ansible Core**, **Ansible Automation Platform (AAP) 2.4**, and enterprise automation best practices. This course is designed for System Administrators, DevOps Engineers, and Platform Engineers.

---

## Course Overview

| Attribute | Description |
|-----------|-------------|
| **Platform** | Red Hat Ansible Automation Platform 2.4 |
| **Total Labs** | 19 (Lab 00 through Lab 18) |
| **Approx. Duration** | 60+ hours (theory + hands-on) |
| **Audience** | Beginner to Advanced |
| **Format** | Theory + Hands-on exercises |

---

## Prerequisites

- Basic Linux/Unix command-line experience
- Understanding of SSH and network concepts
- (Optional) Familiarity with YAML
- Red Hat account for AAP subscription (Labs 02+)

---

## Course Curriculum

### Lab 00 — Introduction to Automation and Ansible
**Duration:** 2.5–3 hours | **Type:** Theory  
**File:** `Lab 00 - Introduction to Ansible.md`

| Topics Covered |
|----------------|
| IT automation and why it matters |
| Infrastructure as Code (IaC) |
| Evolution: manual → shell scripts → configuration management |
| Ansible architecture and use cases |
| Ansible Core vs. Ansible Automation Platform |
| AAP components: Controller, Hub, Execution Environments, Event-Driven Ansible |
| When to use Core vs. AAP |

---

### Lab 01 — Install Ansible Core and Deploy Nginx Server
**Duration:** 1.5–2 hours | **Type:** Hands-on  
**File:** `Lab 01 - Install Ansible Core and Deploy Nginx Server.md`

| Topics Covered |
|----------------|
| Install Ansible Core on RHEL |
| SSH key-based authentication |
| Project structure (ansible.cfg, inventory) |
| Writing playbooks (dnf, apt, service, firewalld) |
| Idempotency and facts |
| Deploy custom web page |

---

### Lab 02 — Install Ansible Automation Platform (Standalone)
**Duration:** 3–4 hours | **Type:** Hands-on  
**File:** `Lab 02 - Install Ansible Automation Platform - Standalone.md`

| Topics Covered |
|----------------|
| AAP standalone architecture |
| Prepare RHEL server, register, enable repos |
| Configure installer inventory |
| Run AAP setup, subscription activation |
| UI tour (Dashboard, Projects, Inventories, Credentials, Templates) |
| Create inventory, credential, project, job template |
| First job execution |

---

### Lab 03 — Add, Remove, and Manage AAP Nodes
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 03 - Add Remove and Manage AAP Nodes.md`

| Topics Covered |
|----------------|
| Node types: control, execution, hybrid, hop |
| Receptor mesh and capacity |
| Add execution nodes to AAP |
| Instance groups for workload isolation |
| Pinning job templates to instance groups |
| Remove and decommission nodes |

---

### Lab 04 — Manage Inventories, Credentials and Integrations
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 04 - Manage Inventories Credentials and Integrations.md`

| Topics Covered |
|----------------|
| Static vs. dynamic inventories |
| Host/group variables, variable precedence |
| Smart inventories |
| Credential types: Machine, SCM, Vault, AWS, Azure, Network |
| External secret management overview |
| Git integration, notification templates (Slack, Email, Webhook) |

---

### Lab 05 — Manage Local and Git Version Control Repositories
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 05 - Manage Local and Git Version Control Repositories in AAP.md`

| Topics Covered |
|----------------|
| Version control and Git basics |
| Projects: Manual vs. Git/SCM |
| Directory structure (playbooks, roles, group_vars) |
| requirements.yml for collections |
| Connect AAP to Git, branch selection |
| Webhook for automatic sync |
| Update on launch |

---

### Lab 06 — Manage Organizations, Projects, Teams, and RBAC
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 06 - Manage Organizations Projects Teams and RBAC.md`

| Topics Covered |
|----------------|
| Organizations and multi-tenancy |
| Users and teams |
| RBAC: roles (Admin, Member, Execute, Use, Read) |
| Object-level permissions |
| Team-based access control |
| Testing RBAC scenarios |

---

### Lab 07 — Configure Advanced Job Templates
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 07 - Configure Advanced Job Templates.md`

| Topics Covered |
|----------------|
| Job template components |
| Surveys and runtime input |
| Limit patterns, tags, skip tags |
| Extra variables |
| Job slicing, fact cache |
| Check mode, provisioning callbacks |
| Timeout, verbosity, concurrent jobs |

---

### Lab 08 — Build and Manage Job Workflows
**Duration:** 5 hours | **Type:** Theory + Hands-on  
**File:** `Lab 08 - Build and Manage Job Workflows.md`

| Topics Covered |
|----------------|
| Workflow templates vs. job templates |
| Workflow Visualizer |
| Job nodes, approval nodes, project/inventory sync |
| Success/failure/always paths |
| Convergence and divergence |
| set_stats and artifact passing |
| Failure handling and recovery |

---

### Lab 09 — Ansible Automation Hub
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 09 - Ansible Automation Hub.md`

| Topics Covered |
|----------------|
| Public vs. Private Automation Hub |
| Certified and validated content |
| Browse collections |
| Configure Hub as content source in Controller |
| Galaxy credential |
| Private Hub setup (optional) |
| Execution environment images in Hub |

---

### Lab 10 — Manage Advanced and Dynamic Inventories
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 10 - Manage Advanced and Dynamic Inventories.md`

| Topics Covered |
|----------------|
| Dynamic inventory sources |
| Project-sourced, AWS EC2, Azure RM |
| Smart inventories and filters |
| Combined inventory sources |
| Update on launch vs. scheduled sync |
| Large-scale inventory organization |

---

### Lab 11 — Develop Ansible Playbooks in AAP
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 11 - Develop Ansible Playbooks in AAP.md`

| Topics Covered |
|----------------|
| Playbook structure (plays, tasks) |
| Variables, facts, registers |
| Loops, conditionals (when) |
| Handlers, block/rescue/always |
| Idempotency, error handling |
| delegation, run_once |
| Integration with AAP (surveys, extra vars, Vault, artifacts) |

---

### Lab 12 — Run Playbooks with Automation Controller
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 12 - Run Playbooks with Automation Controller.md`

| Topics Covered |
|----------------|
| Manual job launches, surveys, extra variables |
| Limit and tags at runtime |
| Job scheduling (one-time, recurring, cron) |
| REST API job triggers |
| Webhook and provisioning callbacks |
| Monitoring job execution, cancel, troubleshoot |
| Job facts, artifacts, fact caching |

---

### Lab 13 — Manage Task Execution
**Duration:** 3 hours | **Type:** Theory + Hands-on  
**File:** `Lab 13 - Manage Task Execution.md`

| Topics Covered |
|----------------|
| How Ansible executes tasks |
| Forks, connection types (SSH, WinRM, local) |
| Serial vs. parallel execution |
| Instance capacity, job slots, queue |
| Instance group assignment |
| Performance: forks, fact gathering, pipelining, async |
| Troubleshooting execution issues |
| Execution environment considerations |

---

### Lab 14 — Transform Data with Filters and Plug-ins
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 14 - Transform Data with Filters and Plug-ins.md`

| Topics Covered |
|----------------|
| Jinja2 templating |
| String, list, dict filters |
| default, sum, min, max, strftime |
| ipaddr, to_json, from_json, regex |
| Lookup plugins: file, env, password, template, URL, CSV |
| Custom filter plugins |

---

### Lab 15 — Create, Manage, and Publish Ansible Content Collections
**Duration:** 5 hours | **Type:** Theory + Hands-on  
**File:** `Lab 15 - Create Manage and Publish Ansible Content Collections.md`

| Topics Covered |
|----------------|
| Collection structure (galaxy.yml, roles, plugins) |
| ansible-galaxy collection init |
| Adding roles and filter plugins |
| meta/runtime.yml dependencies |
| Building and publishing to Private Hub |
| requirements.yml, FQCN |
| Version pinning |

---

### Lab 16 — Event-Driven Ansible
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 16 - Event-Driven Ansible.md`

| Topics Covered |
|----------------|
| Event-Driven Ansible (EDA) architecture |
| Rulebooks: sources, rules, conditions, actions |
| Event sources: webhook, range, Alertmanager, file |
| Writing rulebooks |
| run_playbook, run_job_template actions |
| Decision environments |
| Integrating EDA with Automation Controller |

---

### Lab 17 — Manage Automation Execution Environments
**Duration:** 4 hours | **Type:** Theory + Hands-on  
**File:** `Lab 17 - Manage Automation Execution Environments.md`

| Topics Covered |
|----------------|
| Execution environment concepts |
| ansible-builder, execution-environment.yml |
| Adding collections, Python packages, system packages |
| bindep.txt |
| Building EE images |
| ansible-navigator for testing |
| Publishing to Hub, assigning to templates and organizations |

---

### Lab 18 — AAP Best Practices
**Duration:** 3 hours | **Type:** Theory + Hands-on (Capstone)  
**File:** `Lab 18 - AAP Best Practices.md`

| Topics Covered |
|----------------|
| Architecture and planning (capacity, HA, DR, network) |
| Development best practices (Git flow, testing, code review) |
| Security (RBAC, credentials, Vault, audit) |
| Operational (monitoring, alerting, backup) |
| Content organization (collections vs. roles) |
| Scalability and growth |
| Common pitfalls to avoid |
| **Capstone:** End-to-end automation solution |

---

## Learning Path

```
Lab 00 (Theory) ──▶ Lab 01 (Ansible Core) ──▶ Lab 02 (AAP Install)
                                                    │
                                                    ▼
Lab 03 (Nodes) ◀── Lab 04 (Inventories, Creds) ◀──┘
    │                    │
    └────────────────────┼────────────────────────┐
                         ▼                         ▼
Lab 05 (Git/Projects)  Lab 06 (RBAC)         Lab 07 (Job Templates)
    │                    │                         │
    └────────────────────┼─────────────────────────┘
                         ▼
Lab 08 (Workflows) ──▶ Lab 09 (Hub) ──▶ Lab 10 (Dynamic Inventories)
    │                        │
    └────────────────────────┼─────────────────────┐
                             ▼                       ▼
Lab 11 (Playbooks) ──▶ Lab 12 (Run Jobs) ──▶ Lab 13 (Task Execution)
    │                        │                       │
    └────────────────────────┼───────────────────────┘
                             ▼
Lab 14 (Filters) ──▶ Lab 15 (Collections) ──▶ Lab 16 (EDA)
    │                        │                       │
    └────────────────────────┼───────────────────────┘
                             ▼
                    Lab 17 (Execution Envs) ──▶ Lab 18 (Best Practices)
```

---

## Lab File Reference

| Lab | File Name |
|-----|-----------|
| 00 | `Lab 00 - Introduction to Ansible.md` |
| 01 | `Lab 01 - Install Ansible Core and Deploy Nginx Server.md` |
| 02 | `Lab 02 - Install Ansible Automation Platform - Standalone.md` |
| 03 | `Lab 03 - Add Remove and Manage AAP Nodes.md` |
| 04 | `Lab 04 - Manage Inventories Credentials and Integrations.md` |
| 05 | `Lab 05 - Manage Local and Git Version Control Repositories in AAP.md` |
| 06 | `Lab 06 - Manage Organizations Projects Teams and RBAC.md` |
| 07 | `Lab 07 - Configure Advanced Job Templates.md` |
| 08 | `Lab 08 - Build and Manage Job Workflows.md` |
| 09 | `Lab 09 - Ansible Automation Hub.md` |
| 10 | `Lab 10 - Manage Advanced and Dynamic Inventories.md` |
| 11 | `Lab 11 - Develop Ansible Playbooks in AAP.md` |
| 12 | `Lab 12 - Run Playbooks with Automation Controller.md` |
| 13 | `Lab 13 - Manage Task Execution.md` |
| 14 | `Lab 14 - Transform Data with Filters and Plug-ins.md` |
| 15 | `Lab 15 - Create Manage and Publish Ansible Content Collections.md` |
| 16 | `Lab 16 - Event-Driven Ansible.md` |
| 17 | `Lab 17 - Manage Automation Execution Environments.md` |
| 18 | `Lab 18 - AAP Best Practices.md` |

---

## Module-to-Lab Mapping

| Module | Lab | Focus |
|--------|-----|-------|
| Module 01 | Lab 00 | Introduction to automation |
| Module 02 | Lab 01 | Ansible Core basics |
| Module 03 | Lab 02 | AAP installation |
| Module 04 | Lab 03 | AAP node management |
| Module 05 | Lab 04 | Inventories and credentials |
| Module 06 | Lab 05 | Version control and projects |
| Module 07 | Lab 06 | Organizations and RBAC |
| Module 08 | Lab 07 | Advanced job templates |
| Module 09 | Lab 08 | Job workflows |
| Module 10 | Lab 09 | Automation Hub |
| Module 11 | Lab 10, 12 | Dynamic inventories; Running jobs |
| Module 12 | Lab 11, 13 | Playbooks; Task execution |
| Module 13 | Lab 14 | Filters and plugins |
| Module 14 | Lab 15 | Collections |
| Module 15 | Lab 16 | Event-Driven Ansible |
| Module 16 | Lab 17 | Execution environments |
| Module 17 | Lab 18 | Best practices (capstone) |

---

## Quick Stats

| Metric | Value |
|--------|-------|
| Total labs | 19 |
| Theory-only | 1 (Lab 00) |
| Hands-on | 18 |
| Estimated total duration | 60+ hours |
| Platform | Red Hat Ansible Automation Platform 2.4 |

---

*This curriculum is designed for instructor-led or self-paced training. Labs build on each other; complete prerequisites before starting each lab.*
