# Lab 18 — AAP Best Practices

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate to Advanced (Ops / DevOps / Platform Engineers)               |
| **Duration** | 3 hours (Theory: 50% / Hands-on: 50%)                                      |
| **Outcome**  | Apply architecture, development, security, operational, and content best practices; implement RBAC, monitoring, content organization, and security hardening; complete a capstone project incorporating lessons from all modules |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Labs 02–17 completed — comprehensive AAP knowledge. This lab serves as a capstone and best-practices consolidation. |

---

## AAP 2.4 UI Navigation Reference

| Menu | Location | Contains |
|------|----------|----------|
| **Resources** | Left panel | Inventories, Projects, Credentials, Templates |
| **Access** | Left panel | Organizations, Users, Teams |
| **Administration** | Left panel | Notifications, Instances, Execution Environments |
| **Settings** | Left panel | System, Jobs, License, Authentication |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    AAP BEST PRACTICES — END-TO-END VIEW                              │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  ARCHITECTURE│  │  DEVELOPMENT │  │  SECURITY   │  │  OPERATIONS  │              │
│  │  - Capacity  │  │  - Git flow  │  │  - RBAC     │  │  - Monitoring│              │
│  │  - HA/DR     │  │  - Testing  │  │  - Vault    │  │  - Alerting  │              │
│  │  - Network   │  │  - Review   │  │  - Audit    │  │  - Backup    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                               │
│  │  CONTENT     │  │  SCALABILITY│  │  PITFALLS   │                               │
│  │  - Collections│  │  - Growth   │  │  - Avoid    │                               │
│  │  - Roles    │  │  - Nodes    │  │  - Lessons   │                               │
│  └──────────────┘  └──────────────┘  └──────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 — **AAP 2.4** |
| Completed Labs | 02–17 (or equivalent knowledge) |
| Resources | Inventories, projects, job templates, credentials from prior labs |
| Admin access | For RBAC, settings, and configuration |

---

## Part 1 — Theory: Best Practices Across All Domains (90 minutes)

### Section 17.1 — Architecture and Planning Best Practices

#### Capacity Planning Guidelines

| Factor | Guideline |
|--------|-----------|
| **Control node** | 4–8 vCPU, 16–32 GB RAM for small; 8+ vCPU, 32+ GB for medium |
| **Execution nodes** | 2–4 vCPU, 4–8 GB RAM per node; scale horizontally |
| **Concurrent jobs** | Plan for peak load (e.g., 2x daily average) |
| **Database** | PostgreSQL; size based on job retention |

#### When to Scale Horizontally

| Trigger | Action |
|--------|--------|
| Jobs queuing > 5 min | Add execution nodes |
| Controller CPU > 70% | Add execution nodes; move jobs off controller |
| Single point of failure | Add HA control nodes (AAP HA) |

#### High Availability Considerations (Introduction)

- **Controller HA** — Multiple controller nodes; shared database
- **Database** — PostgreSQL HA (replication, failover)
- **Execution nodes** — Stateless; add/remove as needed

#### Disaster Recovery Basics

| Component | DR Strategy |
|-----------|-------------|
| **Database** | Regular backups; test restore |
| **Projects** | Git is source of truth; re-sync |
| **Credentials** | Backup credential store; document rotation |
| **Config** | Infrastructure as Code; version control |

#### Network Design Recommendations

| Recommendation | Purpose |
|----------------|---------|
| **Segment AAP** | Isolate from general user network |
| **Receptor port** | TCP 27199 between controller and execution nodes |
| **Database** | Local or dedicated VLAN; restrict access |
| **Outbound** | Allow Galaxy/Hub, Git, managed nodes |

#### Database Maintenance and Backup Strategies

| Task | Frequency |
|------|-----------|
| **Backup** | Daily (at minimum) |
| **Vacuum** | Weekly or as needed |
| **Job retention** | Configure in Settings → Jobs |
| **Test restore** | Quarterly |

---

### Section 17.2 — Development and Content Best Practices

#### Organizing Playbooks and Roles Effectively

```
project/
├── playbooks/
│   ├── site.yml           # Main entry
│   ├── deploy.yml
│   ├── patch.yml
│   └── network/
├── roles/
│   ├── common/
│   ├── nginx/
│   └── postgresql/
├── group_vars/
├── host_vars/
├── requirements.yml
└── README.md
```

#### Naming Conventions for Clarity

| Resource | Convention | Example |
|----------|------------|---------|
| **Playbooks** | `verb_noun.yml` | `deploy_app.yml`, `patch_servers.yml` |
| **Roles** | `noun` or `purpose` | `nginx`, `common`, `hardening` |
| **Templates** | `Purpose-Env` | `Deploy-WebApp-Prod`, `Patch-All-Dev` |
| **Inventories** | `Env-Tier` | `Prod-Web`, `Dev-All` |

#### Version Control Workflow Recommendations

| Branch | Purpose |
|--------|---------|
| `main` | Production-ready |
| `develop` | Integration |
| `feature/*` | New features |
| `hotfix/*` | Urgent fixes |

#### Testing Before Production (Dev → Staging → Prod)

| Stage | Inventory | Validation |
|-------|-----------|------------|
| **Dev** | Dev hosts | Functional test |
| **Staging** | Staging hosts | Integration, performance |
| **Prod** | Prod hosts | After approval |

#### Code Review Processes

- Pull requests required for `main`
- Review checklist: syntax, idempotency, error handling, docs
- Automated: ansible-lint, molecule (optional)

#### Documentation Standards

- **README.md** — Purpose, usage, vars
- **Role README** — Vars, examples, dependencies
- **Playbook comments** — Complex logic explained

---

### Section 17.3 — Security Best Practices

#### Credential Management Principles

| Principle | Implementation |
|-----------|----------------|
| **Never in playbooks** | Use AAP credentials; reference by name |
| **Least privilege** | Separate creds per env (dev vs prod) |
| **No shared passwords** | Individual or service accounts |
| **Encryption** | AAP encrypts at rest; use Vault for files |

#### Regular Credential Rotation

| Schedule | Action |
|----------|--------|
| **Quarterly** | Rotate SSH keys, API tokens |
| **On personnel change** | Revoke and rotate |
| **On compromise** | Immediate rotation |

#### Implementing Least-Privilege RBAC

| Role | Access |
|------|--------|
| **Developer** | Member on projects; Execute on dev templates |
| **Operator** | Execute on templates; Use on creds/inventory |
| **Auditor** | Read on all |
| **Admin** | Full within org |

#### Audit Logging and Compliance

- **Activity Stream** — All AAP actions logged
- **Job output** — Retained per policy
- **Integration** — SIEM, log aggregation for compliance (PCI, HIPAA, SOC2)

#### Network Segmentation for AAP

| Segment | Access |
|---------|--------|
| **AAP management** | Ops team only |
| **Execution → managed** | SSH/WinRM outbound |
| **AAP → Hub/Galaxy** | Outbound HTTPS |

#### Securing Sensitive Data with Vault

- **Ansible Vault** — Encrypt vars, files
- **Vault credential** — In AAP for decryption
- **External secrets** — HashiCorp Vault, CyberArk for high-security

#### Regular Security Updates

- **AAP** — Apply patches per Red Hat advisories
- **Base images** — Rebuild EEs with updated base
- **Collections** — Update; test; promote

---

### Section 17.4 — Operational Best Practices

#### Monitoring AAP Health

| Metric | Tool / Location |
|--------|-----------------|
| **Service status** | systemctl, AAP health check |
| **Job success rate** | Dashboard, Reports |
| **Instance capacity** | Administration → Instances |
| **Database** | PostgreSQL metrics |

#### Setting Up Alerting for Critical Issues

| Alert | Trigger | Action |
|-------|---------|--------|
| **Job failure** | Template fails | Slack, email, PagerDuty |
| **Instance down** | Node unhealthy | Notify ops |
| **Queue backup** | Jobs pending > N min | Investigate capacity |

#### Performance Tuning Guidelines

| Area | Tuning |
|------|--------|
| **Forks** | Match inventory size; avoid overload |
| **Fact cache** | Enable for repeated fact gathering |
| **Project sync** | Update on launch vs scheduled |
| **Database** | Connection pooling, vacuum |

#### Job Design Patterns for Reliability

| Pattern | Use |
|---------|-----|
| **Idempotent** | Safe re-runs |
| **Check mode** | Validate before apply |
| **Tags** | Partial runs for troubleshooting |
| **Handlers** | Restart services only when needed |
| **Blocks** | Error handling with rescue |

#### Inventory Organization Strategies

| Strategy | Example |
|----------|---------|
| **By environment** | prod, staging, dev |
| **By tier** | webservers, appservers, databases |
| **By region** | us-east, eu-west |
| **Dynamic** | EC2, Azure for cloud |

#### Change Management Processes

- **Approval workflows** — For prod templates
- **Staging validation** — Before prod
- **Rollback plan** — Document and test
- **Change window** — Schedule high-risk changes

#### Backup and Restore Procedures

| Component | Backup | Restore |
|-----------|--------|---------|
| **Database** | pg_dump daily | pg_restore |
| **Projects** | Git (source) | Re-sync |
| **Config** | Export; IaC | Re-apply |

---

### Section 17.5 — Content Organization Best Practices

#### Collections vs. Roles: When to Use Each

| Use | Choice |
|-----|--------|
| **Reusable, versioned, shared** | Collection |
| **Project-specific, simple** | Role in project |
| **Modules/plugins** | Collection |
| **Quick prototype** | Role |

#### Building Reusable Content

- **Parameterize** — Vars for all configurable values
- **Defaults** — Sensible defaults in role defaults
- **Document** — README with examples
- **Test** — Molecule, CI

#### Namespace Organization

| Namespace | Content |
|-----------|---------|
| `acme.network` | Network automation |
| `acme.cloud` | Cloud provisioning |
| `acme.security` | Hardening, compliance |

#### Managing Dependencies Effectively

- **requirements.yml** — Pin versions
- **meta/runtime.yml** — Collection dependencies
- **Avoid circular** — Design dependency graph

#### Versioning Strategies

| Strategy | Use |
|----------|-----|
| **Semantic** | 1.0.0, 1.1.0 |
| **CalVer** | 2024.03 |
| **Pinned** | Exact version in production |

#### Sharing Content Across Teams

- **Private Hub** — Internal collections
- **Git** — Playbooks, roles
- **Documentation** — Central wiki, README

---

### Section 17.6 — Scalability and Growth

#### Planning for Future Growth

| Factor | Plan |
|--------|------|
| **Host count** | 2x in 12 months? |
| **Teams** | New orgs, instance groups |
| **Job volume** | Peak vs average |
| **Geographic** | Regional execution nodes |

#### When to Add More Nodes

| Signal | Action |
|--------|--------|
| **Queue time > 5 min** | Add execution nodes |
| **Controller overloaded** | Move jobs to execution nodes |
| **Regional latency** | Add nodes in region |
| **Team isolation** | Instance groups |

#### Optimizing Job Execution at Scale

| Optimization | Impact |
|--------------|--------|
| **Forks** | Tune per job type |
| **Fact cache** | Reduce setup time |
| **Job slicing** | Parallelize large inventories |
| **Async** | Long-running tasks |

#### Managing Large Inventories Efficiently

| Approach | Use |
|----------|-----|
| **Dynamic** | Cloud, CMDB |
| **Smart inventory** | Filtered views |
| **Limit** | Target subsets |
| **Serial** | Rolling updates |

#### Performance Testing Strategies

- **Load test** — Simulate N concurrent jobs
- **Inventory scale** — Test with 100, 1000 hosts
- **Baseline** — Measure before/after changes

---

### Section 17.7 — Common Pitfalls to Avoid

| Pitfall | Avoid By |
|---------|----------|
| **Over-complicating** | Start simple; add complexity when needed |
| **Poor error handling** | Use blocks, rescue, failed_when |
| **Neglecting documentation** | README, comments, runbooks |
| **Not testing before prod** | Dev → staging → prod |
| **Ignoring performance** | Tune forks, fact cache, slicing |
| **Security misconfigurations** | RBAC, credential rotation, audit |
| **No backup/DR** | Automated backups; test restore |
| **Monolithic playbooks** | Roles, includes, smaller units |

---

## Part 2 — Hands-On: Implement Best-Practice AAP Setup (90 minutes)

### Step 1 — Implement RBAC Best Practices

**Purpose:** Apply least-privilege RBAC.

**Detailed steps:**

1. Create teams: `Dev-Team`, `Ops-Team`, `Audit-Team`
2. Create users: `dev_user`, `ops_user`, `audit_user`
3. Assign roles:
   - **Dev-Team:** Member on projects; Execute on dev templates; Use on dev inventory/creds
   - **Ops-Team:** Execute on prod templates; Use on prod inventory/creds; no project edit
   - **Audit-Team:** Read on all resources
4. Test: Log in as each user; verify access matches design

**Verify:** Each role has correct access; no over-permission.

---

### Step 2 — Configure Monitoring and Alerting

**Purpose:** Set up notifications for critical issues.

**Detailed steps:**

1. **Administration → Notifications** — Create notification template:
   - **Name:** `Slack-Job-Failure`
   - **Type:** Slack (or Email if no Slack)
   - **Configure:** Webhook URL or SMTP
2. Attach to a job template: **On failure**
3. Create a job template that intentionally fails (e.g., typo in module)
4. Launch — verify notification is received

**Verify:** Failure triggers notification.

---

### Step 3 — Organize Content Structure

**Purpose:** Apply content organization best practices.

**Detailed steps:**

1. In your project (Git or manual), create structure:

```
playbooks/
├── site.yml
├── deploy.yml
├── patch.yml
group_vars/
├── all/
│   └── all.yml
├── prod/
└── dev/
roles/
README.md
requirements.yml
```

2. Add README.md with: purpose, usage, variable documentation
3. Add requirements.yml with pinned collections
4. Sync project in AAP

**Verify:** Structure is clear; project syncs successfully.

---

### Step 4 — Security Hardening

**Purpose:** Apply security best practices.

**Detailed steps:**

1. **Credentials:** Ensure no credentials in playbooks; all use AAP credential references
2. **RBAC:** Verify no overly broad permissions (e.g., System Admin for normal users)
3. **Vault:** If using encrypted vars, ensure Vault credential is configured
4. **Audit:** Review Activity Stream — confirm actions are logged
5. **Settings:** Check Session timeout, Password policy (if local auth)

**Verify:** Security posture improved; no credentials in code.

---

### Step 5 — Capstone Project: End-to-End Automation Solution

**Purpose:** Design and implement a complete automation solution using best practices.

**Detailed steps:**

1. **Design** (15 min):
   - Choose scenario: e.g., "Deploy and patch web application"
   - Define: inventory groups, playbooks, job templates, workflow
   - Plan: dev → staging → prod promotion
   - Document: architecture diagram, RBAC matrix

2. **Implement** (45 min):
   - Create inventory with groups (webservers, appservers)
   - Create playbook(s) with roles, handlers, error handling
   - Create job templates: Deploy-Dev, Deploy-Staging, Deploy-Prod
   - Create workflow: Project Sync → Deploy → Notify (optional)
   - Apply RBAC: Dev team → dev; Ops team → prod
   - Add notifications: On failure
   - Use execution environment (default or custom)
   - Add README and documentation

3. **Validate** (15 min):
   - Run Deploy-Dev — success
   - Run Deploy-Staging — success
   - Run Deploy-Prod (or simulate) — success
   - Test failure path — notification received
   - Verify RBAC — dev_user cannot run prod template

4. **Document** (15 min):
   - Update README with runbook
   - Document RBAC matrix
   - Document promotion process

**Verify:** End-to-end solution works; follows best practices.

---

## Part 3 — Real-Time Use Case Discussion (Integrated)

### Case Study: Global Enterprise — AAP Best Practices in Action

**Background:**

A global enterprise with 50+ IT teams, 10,000+ servers across 5 regions, and strict compliance requirements (SOX, PCI-DSS). Manual operations were slow, error-prone, and costly.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| 50+ teams | Siloed automation; no reuse |
| 10,000+ servers | Manual impossible; needed scale |
| Compliance | Audit trail; change control; least privilege |
| Cost | High ops cost; slow delivery |

**Solution: AAP Best Practices Implementation**

#### 1. Organization Structure for 50+ Teams

| Structure | Implementation |
|-----------|-----------------|
| **Organizations** | By business unit: BU-Finance, BU-Retail, BU-Cloud |
| **Teams** | Per org: Dev, Ops, Audit |
| **Instance groups** | Per environment: dev-workers, prod-workers |
| **RBAC** | Least privilege; approval for prod |

#### 2. Content Organization

| Practice | Implementation |
|----------|-----------------|
| **Collections** | Internal: acme.network, acme.cloud, acme.security |
| **Git workflow** | main, develop, feature branches |
| **Promotion** | Dev → Staging (auto) → Prod (approval) |
| **Versioning** | Semantic; pinned in prod |

#### 3. Security and Compliance

| Practice | Implementation |
|----------|-----------------|
| **Credentials** | Vault; rotation quarterly |
| **RBAC** | Role matrix per team |
| **Audit** | Activity Stream → SIEM |
| **Network** | AAP in dedicated segment |

#### 4. Operations

| Practice | Implementation |
|----------|-----------------|
| **Monitoring** | Health checks; job success rate |
| **Alerting** | Failure → PagerDuty; queue → Slack |
| **Backup** | Daily DB; tested restore |
| **DR** | Documented; tested annually |

**Results (18 months):**

| Metric | Before | After |
|--------|--------|-------|
| **Infrastructure automated** | 20% | 80% |
| **Automation success rate** | 85% | 99.9% |
| **Operational cost** | Baseline | -60% |
| **Deployment time** | Days | Hours |
| **Audit findings** | 15 | 0 |

**Key takeaway:** Consistent application of architecture, development, security, operational, and content best practices enabled scale, reliability, and cost reduction.

---

## Lab Completion Checklist

- [ ] Implemented RBAC with least privilege
- [ ] Configured monitoring/alerting for job failures
- [ ] Organized content structure (playbooks, roles, vars)
- [ ] Applied security hardening (creds, RBAC, audit)
- [ ] Completed capstone project
- [ ] Documented solution (README, RBAC matrix)
- [ ] Can explain global enterprise use case
- [ ] Can list common pitfalls to avoid

---

## Quick Reference — Best Practices Summary

| Domain | Key Practices |
|--------|---------------|
| **Architecture** | Capacity plan; scale horizontally; backup DB |
| **Development** | Git workflow; test dev→staging→prod; code review |
| **Security** | RBAC; credential rotation; Vault; audit |
| **Operations** | Monitor; alert; tune; backup/restore |
| **Content** | Collections for reuse; version; document |
| **Scalability** | Add nodes; optimize; dynamic inventory |
| **Pitfalls** | Avoid over-complexity; test; document |

---

## Quick Reference — Capstone Deliverables

| Deliverable | Content |
|-------------|---------|
| **Architecture** | Diagram; inventory; templates |
| **RBAC matrix** | User/team → permissions |
| **README** | Purpose; usage; vars |
| **Runbook** | How to run; troubleshoot |
| **Promotion** | Dev → staging → prod process |

---

*End of Lab 18 — AAP Best Practices and Capstone*
