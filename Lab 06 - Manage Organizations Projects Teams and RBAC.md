# Lab 06 — Manage Organizations, Projects, Teams, and RBAC

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 4 hours (Theory: 35% / Hands-on: 65%)                                     |
| **Outcome**  | Create organizations, users, and teams; implement RBAC policies; grant object-level permissions; test access control scenarios; understand authentication options |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller)        |
| **Prerequisite** | Lab 02 completed — AAP 2.4 installed with web UI access. Labs 04–05 recommended (inventories, projects, job templates). |

---

## AAP 2.4 UI Navigation Reference

Before starting, familiarize yourself with the **Automation Controller** UI structure in AAP 2.4:

| Menu | Location | Contains |
|------|----------|----------|
| **Resources** | Left navigation panel | Hosts, Inventories, Projects, Credentials, Templates |
| **Access** | Left navigation panel | Organizations, Users, Teams |
| **Administration** | Left navigation panel | Notifications, Instances, Instance Groups, Execution Environments, etc. |
| **Settings** | Left navigation panel | System configuration, Authentication, Jobs, License |

> **Note:** In AAP 2.4, **Access** contains Organizations, Users, and Teams. Some versions may show "Access Management" — navigate to Organizations, Users, and Teams from the left panel.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / BROWSER                                        │
│                         https://aap1.lab.local                                       │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
                                      │ HTTPS (443)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                      │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  ORGANIZATIONS (Multi-tenancy)                                               │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                          │   │
│  │  │  Default    │  │  Org-Dev    │  │  Org-Prod   │                          │   │
│  │  │  (existing) │  │  (new)      │  │  (new)     │                          │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                          │   │
│  │         │                │                │                                   │   │
│  │  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐                          │   │
│  │  │   Teams     │  │   Teams     │  │   Teams     │                          │   │
│  │  │  - Ops      │  │  - Devs     │  │  - Ops      │                          │   │
│  │  │  - Auditors │  │  - Testers  │  │  - Auditors │                          │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  RBAC: Users → Teams → Roles → Resources (Projects, Inventories, Templates)          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

> **Note:** In AAP 2.4, organizations scope all resources. Users must be in an organization to access resources. Teams belong to one organization. RBAC roles are granted per resource (project, inventory, template, credential).

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Admin access | Logged in as `admin` (or equivalent System Administrator) |
| Existing resources | At least one inventory, project, and job template from previous labs |

---

## Verify AAP 2.4 Before Starting

**How to check your version:**

1. Log in to the Automation Controller.
2. Click the **?** (help) icon or your **user profile** in the top-right.
3. Select **About** — the version (e.g., 4.4.x for AAP 2.4) is displayed.

Alternatively: **Settings** (left panel) → **System** → **Tower** or **Controller** section shows version info.

> **Note:** AAP 2.4 uses Automation Controller 4.4.x. The UI has **Resources**, **Access**, **Administration**, and **Settings** in the left navigation. If you see "Access Management" instead of "Access", the Organizations/Users/Teams paths may be under that menu.

---

## Part 1 — Theory: Organizational Structure and RBAC (85 minutes)

### Section 1 — Understanding Organizational Structure

#### What Are Organizations in AAP?

An **organization** is the top-level container in AAP. It groups:

- **Users** — who can access the platform
- **Teams** — logical groups of users with shared permissions
- **Resources** — inventories, projects, credentials, job templates (scoped to the organization)

Everything in AAP belongs to an organization. The **Default** organization is created automatically during installation.

#### Why Use Organizations? (Multi-Tenancy Explained)

| Scenario | How Organizations Help |
|----------|-------------------------|
| **Multi-tenant** | Separate customers, business units, or departments — each has its own org |
| **Isolation** | Users in Org A cannot see or access resources in Org B |
| **Governance** | Different execution environments, instance groups, or host limits per org |
| **Compliance** | Audit boundaries — e.g., PCI scope in one org, general IT in another |

#### Real-World Organizational Structure Examples

| Structure | Organizations | Use Case |
|-----------|----------------|----------|
| **Single org** | Default only | Small company, one team |
| **By department** | IT-Ops, Development, Security | Medium company, role separation |
| **By environment** | Dev, Staging, Production | Strict promotion workflow |
| **By customer** | Customer-A, Customer-B, Customer-C | Managed service provider (MSP) |
| **By geography** | US-East, EMEA, APAC | Regional autonomy |

#### Planning Your Organization Structure

| Question | Consideration |
|----------|---------------|
| Who needs to be isolated from whom? | Create separate orgs for isolation |
| Who shares resources? | Keep them in the same org |
| How many orgs can your license support? | Check subscription (Self-support = 1 org) |
| Do you need org-level admins? | Assign Organization Administrators |

---

### Section 2 — User and Team Management Basics

#### Creating Users in AAP

| User Type | Capabilities |
|-----------|--------------|
| **Normal** | Access only to resources where they have roles — default for most users |
| **System Administrator** | Full access to entire AAP — equivalent to superuser |
| **System Auditor** | Read-only access to everything — for compliance and audit |

Users are created at the platform level and then **added to organizations**. A user can belong to multiple organizations.

#### Understanding Teams and Their Purpose

A **team** is a group of users within an organization. Benefits:

| Benefit | Description |
|---------|-------------|
| **Bulk permission assignment** | Grant roles to a team instead of each user individually |
| **Easier maintenance** | Add/remove users from team; permissions apply automatically |
| **Logical grouping** | Reflect real-world structure (DevOps team, Network team) |
| **Credential sharing** | Assign "Use" on a credential to a team — all members can use it |

Teams belong to **one organization**. Users in a team inherit the team's roles on resources.

#### Assigning Users to Teams

1. Create the team in an organization
2. Add users to the team (Users tab)
3. Grant roles to the team (Roles tab) — on projects, inventories, job templates, etc.
4. Team members automatically get those permissions

#### Team-Based Access Control Benefits

| Without Teams | With Teams |
|---------------|------------|
| Grant Execute on 10 job templates to 5 users = 50 assignments | Grant Execute to "Ops Team" on 10 templates = 10 assignments |
| New user joins → add to team → gets all team permissions | Same |
| User leaves → remove from team → loses permissions | Same |

---

### Section 3 — Authentication Methods (Overview)

| Method | Description | Use Case |
|--------|-------------|----------|
| **Local** | Users created in AAP; password stored in AAP database | Labs, small deployments |
| **LDAP / Active Directory** | AAP authenticates against LDAP/AD | Enterprise — central identity |
| **SAML / SSO** | Identity provider (IdP) — Okta, Azure AD, Keycloak | Enterprise — single sign-on |
| **Social (OAuth)** | GitHub, Google, etc. | Developer-friendly environments |

> **Lab note:** This lab uses **local authentication**. LDAP, SAML, and social auth require external IdP configuration — covered conceptually only.

---

### Section 4 — Introduction to RBAC (Role-Based Access Control)

#### What Is RBAC and Why It's Important

**RBAC** means users get permissions based on **roles** assigned to them (or their teams), not on individual "allow/deny" flags. Benefits:

| Benefit | Description |
|---------|-------------|
| **Least privilege** | Users get only what they need |
| **Auditability** | Clear record of who has what access |
| **Scalability** | Add users to teams; permissions flow automatically |
| **Consistency** | Same role = same permissions across resources |

#### Understanding Permissions in AAP

Permissions are granted by assigning **roles** to **users** or **teams** on **resources**.

```
User/Team  +  Role  +  Resource  =  Permission
   │           │          │
   │           │          └── Project, Inventory, Job Template, Credential, etc.
   │           └── Admin, Execute, Read, Member, Use, Update, etc.
   └── alice, Ops-Team, etc.
```

#### Built-in Roles Explained

**Organization-level roles:**

| Role | Capabilities |
|------|--------------|
| **Organization Admin** | Full control of the org — create teams, manage users, assign roles |
| **Organization Member** | Member of org; needs additional resource-level roles to do work |
| **Organization Auditor** | Read-only access to org resources |

**Resource-level roles (examples for Job Template, Project, Inventory):**

| Role | Job Template | Project | Inventory | Credential |
|------|--------------|---------|-----------|------------|
| **Admin** | Full control | Full control | Full control | Full control |
| **Execute** | Run jobs | — | — | — |
| **Read** | View only | View only | View only | — |
| **Member** | — | Edit, sync | Edit hosts | — |
| **Use** | — | — | Run ad hoc | Use in jobs |
| **Update** | — | Sync project | — | — |
| **Ad Hoc** | — | — | Run ad hoc commands | — |

> **Note:** Role names and availability vary by resource type. The UI shows the exact roles for each resource.

#### Object-Level Permissions

Permissions are **object-level** — you grant a role on a **specific** project, inventory, or job template, not globally.

| Scope | Example |
|-------|---------|
| **Organization** | User is Member of Default org |
| **Project** | Team "Devs" has **Member** on "Lab-Git-Project" |
| **Inventory** | Team "Ops" has **Use** on "Lab-Inventory" |
| **Job Template** | User "alice" has **Execute** on "Ping-Lab" |
| **Credential** | Team "Ops" has **Use** on "SSH-Key-Lab" |

#### Common RBAC Patterns for Beginners

| Pattern | Who | Roles | Result |
|---------|-----|-------|--------|
| **Developer** | Dev team | Member on projects; Read on inventories; Execute on dev job templates | Can edit playbooks, run dev jobs, view hosts |
| **Operator** | Ops team | Use on credentials; Execute on job templates; Use on inventories | Can run jobs, cannot edit playbooks or credentials |
| **Auditor** | Audit team | Read on all resources (or System Auditor) | Can view everything, cannot change anything |
| **Admin** | Org admin | Organization Admin | Full control within the org |

---

### Section 5 — Implementing Access Control (Theory)

#### Granting Permissions — Where to Start

1. **Organization first** — User must be in an org (as Member or Admin)
2. **Resource access** — Grant roles on specific projects, inventories, templates, credentials
3. **Credential access** — Job templates need credentials; users/teams need **Use** on those credentials to run jobs

#### Permission Flow for Running a Job

To run a job template, a user needs:

| Resource | Required Role |
|----------|---------------|
| Organization | Member (or Admin) |
| Project | Read (to see playbook) |
| Inventory | Use (to target hosts) |
| Credential | Use (to connect to hosts) |
| Job Template | Execute |

If any of these is missing, the job launch will fail or the user will not see the template.

---

## Part 2 — Hands-On: Create Organizations and Users (40 minutes)

### Step 1 — Create a New Organization

**Purpose:** Create an organization to scope users, teams, and resources for multi-tenancy or environment separation.

**Detailed steps:**

1. Log in to AAP as `admin`
2. Navigate to **Access** → **Organizations** (or **Access Management** → **Organizations** in some versions)
3. Click **Create organization**
4. Fill in:
   - **Name:** `Org-Dev`
   - **Description:** `Development organization for lab`
5. Click **Create organization**

   > **Note:** If you have a Self-support license, you may only have the Default organization. In that case, perform all exercises within **Default** — create teams and users there instead of a new org.

### Step 2 — Create a Second Organization (Optional)

**Purpose:** Create a second organization (e.g., for production) to demonstrate org isolation.

**Detailed steps:**

1. **Access** → **Organizations** → **Create organization** (or **Add**)
2. **Name:** `Org-Prod`
3. **Description:** `Production organization for lab`
4. Click **Create organization**

### Step 3 — Create Users

**Purpose:** Create users that will be assigned to teams and granted roles for RBAC testing.

**Detailed steps:**

1. Navigate to **Access** → **Users**
2. Click **Create user**
3. Create the following users (repeat for each):

| Username | First Name | Last Name | Email | User Type | Organization |
|----------|------------|------------|-------|-----------|--------------|
| `dev_alice` | Alice | Developer | alice@lab.local | Normal | Org-Dev (or Default) |
| `ops_bob` | Bob | Operator | bob@lab.local | Normal | Org-Dev (or Default) |
| `audit_carol` | Carol | Auditor | carol@lab.local | Normal | Org-Dev (or Default) |

4. Set a password for each user (e.g., `Password123!`)
5. Click **Create user** for each

### Step 4 — Add Users to an Organization

**Purpose:** Add users to an organization so they can access resources scoped to that org.

**Detailed steps:**

1. **Access** → **Organizations** → Click **Org-Dev** (or **Default**)
2. Click the **Users** tab
3. Click **Add users**
4. Select `dev_alice`, `ops_bob`, `audit_carol`
5. Click **Next**
6. Under **Roles**, select **Member** for each (or leave default)
7. Click **Finish**

---

## Part 3 — Hands-On: Create Teams and Assign Users (35 minutes)

### Step 5 — Create Teams

**Purpose:** Create teams to group users and assign permissions in bulk (Developer, Operator, Auditor patterns).

**Detailed steps:**

1. Navigate to **Access** → **Teams**
2. Click **Create team**
3. Create:

| Team Name | Organization | Description |
|-----------|--------------|-------------|
| `Dev-Team` | Org-Dev (or Default) | Developers who edit playbooks |
| `Ops-Team` | Org-Dev (or Default) | Operators who run jobs |
| `Audit-Team` | Org-Dev (or Default) | Auditors with read-only access |

4. Click **Create team** for each

### Step 6 — Add Users to Teams

**Purpose:** Assign users to teams so they inherit the team's roles on resources.

**Detailed steps:**

1. Open **Dev-Team**
2. Click **Users** tab → **Add users**
3. Select `dev_alice` → **Add**
4. Open **Ops-Team** → Add `ops_bob`
5. Open **Audit-Team** → Add `audit_carol`

### Step 7 — Verify Team Membership

**Purpose:** Confirm users are correctly assigned to their teams before testing RBAC.

**Detailed steps:**

1. **Access** → **Users** → Click `dev_alice`
2. Check **Teams** tab — should show **Dev-Team**
3. Repeat for `ops_bob` and `audit_carol`

---

## Part 4 — Hands-On: Grant Resource-Level Permissions (50 minutes)

### Step 8 — Grant Permissions to Dev-Team (Developer Pattern)

**Purpose:** Grant Dev-Team Member on project, Execute on template, Use on inventory and credential — developer pattern.

**Detailed steps:**

1. **Access** → **Teams** → **Dev-Team**
2. Click **Roles** tab
3. Click **Add roles**
4. Select **Resource type:** **Projects**
5. Click **Next**
6. Select your project (e.g., `Lab-Git-Project` or `Lab-Project`)
7. Click **Next**
8. Select role **Member** (allows edit, sync)
9. Click **Finish**
10. Repeat to add:
    - **Job Templates** → Select `Ping-Lab` (or your template) → Role **Execute**
    - **Inventories** → Select `Lab-Inventory` → Role **Use**
    - **Credentials** → Select `SSH-Key-Lab` → Role **Use**

### Step 9 — Grant Permissions to Ops-Team (Operator Pattern)

**Purpose:** Grant Ops-Team Execute on template, Use on inventory and credential — operator pattern (run only, no edit).

**Detailed steps:**

1. **Access** → **Teams** → **Ops-Team**
2. **Roles** tab → **Add roles**
3. Add:
    - **Job Templates** → `Ping-Lab` → **Execute**
    - **Inventories** → `Lab-Inventory` → **Use**
    - **Credentials** → `SSH-Key-Lab` → **Use**
4. Do **not** grant Member on projects — operators only run, not edit

### Step 10 — Grant Permissions to Audit-Team (Auditor Pattern)

**Purpose:** Grant Audit-Team Read on project, inventory, template — auditor pattern (view only, no execute).

**Detailed steps:**

1. **Access** → **Teams** → **Audit-Team**
2. **Roles** tab → **Add roles**
3. Add **Read** on:
    - **Projects** → `Lab-Git-Project`
    - **Inventories** → `Lab-Inventory`
    - **Job Templates** → `Ping-Lab`
    - **Credentials** → (optional) **Read** if you want them to see credential metadata (not the secret)

### Step 11 — Grant Permissions via User (Alternative)

**Purpose:** Demonstrate that roles can be granted directly to users (alternative to team-based assignment).

**Detailed steps:**

1. **Access** → **Users** → Click `dev_alice`
2. **Roles** tab → **Add roles**
3. Select resource type, resource, and role — same as for teams

---

## Part 5 — Hands-On: Test and Verify Permissions (40 minutes)

### Step 12 — Test as Developer (dev_alice)

**Purpose:** Verify dev_alice can edit projects, launch jobs, and use inventories/credentials as expected for the developer role.

**Detailed steps:**

1. Log out from `admin`
2. Log in as `dev_alice` with the password you set
3. Verify:
   - **Resources** → **Projects** — Can see and edit the project (sync, etc.)
   - **Resources** → **Inventories** — Can see the inventory
   - **Resources** → **Templates** — Can see and **Launch** the job template
4. **Launch** the `Ping-Lab` job — it should succeed
5. Try editing a playbook in the project (if Git) or project settings — should work

### Step 13 — Test as Operator (ops_bob)

**Purpose:** Verify ops_bob can launch jobs but cannot edit projects (operator pattern).

**Detailed steps:**

1. Log out, log in as `ops_bob`
2. Verify:
   - **Resources** → **Projects** — May see project but **cannot** edit (no Member role)
   - **Resources** → **Templates** — Can see and **Launch** the job template
3. **Launch** the job — it should succeed
4. Try to edit the project — should be denied or see read-only view

### Step 14 — Test as Auditor (audit_carol)

**Purpose:** Verify audit_carol has read-only access and cannot launch jobs (auditor pattern).

**Detailed steps:**

1. Log out, log in as `audit_carol`
2. Verify:
   - **Resources** → **Projects** — Can see project (read-only)
   - **Resources** → **Inventories** — Can see inventory
   - **Resources** → **Templates** — Can see template but **cannot** Launch (no Execute)
3. Try to **Launch** the job — should be denied or button hidden
4. Try to edit any resource — should be denied

### Step 15 — Test Permission Denial

**Purpose:** Confirm that users without system-level permissions are correctly denied access to admin areas.

**Detailed steps:**

1. As `ops_bob`, try to access **Access** → **Users** — should be restricted (normal users cannot manage users)
2. As `audit_carol`, try to create a new inventory — should be denied
3. As `dev_alice`, try to delete a credential — should be denied (unless granted Admin)

### Step 16 — Document Your RBAC Matrix

**Purpose:** Record the effective permissions for each role to validate your RBAC design and aid troubleshooting.

**Detailed steps:**

1. Create a simple table of who can do what:

| User | Project Edit | Job Launch | Inventory Edit | Credential View |
|------|--------------|------------|----------------|-----------------|
| dev_alice | Yes (Member) | Yes (Execute) | Use only | Use only |
| ops_bob | No | Yes (Execute) | Use only | Use only |
| audit_carol | No (Read) | No | No (Read) | Read (if granted) |

---

## Part 6 — Hands-On: Organization-Level Access (25 minutes)

### Step 17 — Add an Organization Administrator

**Purpose:** Grant dev_alice org-admin rights so she can manage teams, users, and roles within Org-Dev only.

**Detailed steps:**

1. Log back in as `admin`
2. **Access** → **Organizations** → **Org-Dev**
3. Click **Administrators** tab
4. Click **Add administrators**
5. Select `dev_alice`
6. Click **Add administrators**

7. Log in as `dev_alice` — she can now:
   - Create teams in Org-Dev
   - Add/remove users from Org-Dev
   - Manage roles within Org-Dev
   - She still cannot access other organizations or system settings

### Step 18 — Create Resources in a New Organization (If You Have Multiple Orgs)

**Purpose:** Demonstrate multi-org setup and that users can access resources from multiple orgs when assigned.

**Detailed steps:**

1. As `admin`, switch to **Org-Prod**
2. Create an inventory, project, and job template in Org-Prod
3. Add `ops_bob` to Org-Prod as Member
4. Grant Ops-Team (or `ops_bob`) Execute on the Org-Prod job template
5. Log in as `ops_bob` — he should see resources from **both** Org-Dev and Org-Prod (if he is in both), or only from orgs he belongs to

### Step 19 — Verify Isolation Between Organizations

**Purpose:** Confirm that users in one org cannot see resources from another org (org isolation).

**Detailed steps:**

1. Create a user `prod_only` in **Org-Prod** only
2. Log in as `prod_only` — should see only Org-Prod resources
3. Org-Dev resources should not be visible

---

## Part 7 — Real-Time Use Case Discussion (20 minutes)

### Case Study: Managed Service Provider — Customer Isolation with RBAC

**Background:**

An MSP manages automation for 50+ customers. Each customer has its own infrastructure (servers, networks) and must be isolated from others for security and compliance.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Customer A must not see Customer B's data | Regulatory requirement |
| MSP ops team must manage all customers | Centralized operations |
| Customer-specific credentials | Each customer has own SSH keys, cloud creds |
| Audit per customer | Compliance requires per-customer audit trail |

**Solution: Organizations + RBAC**

| Component | Implementation |
|-----------|-----------------|
| **Organizations** | One org per customer: `Customer-A`, `Customer-B`, ... |
| **Resources** | Each org has its own inventories, projects, credentials, templates |
| **Customer users** | Customers get Normal users in their org only — they see only their resources |
| **MSP Ops team** | One team `MSP-Ops` with users added to **each** customer org as Org Admin or with Execute on all templates |
| **Credentials** | Stored per org; MSP-Ops has Use on all; customers have Use only on their own |

**Architecture:**

```
Org: Customer-A          Org: Customer-B          Org: Customer-C
├── Inventory (A hosts)   ├── Inventory (B hosts)  ├── Inventory (C hosts)
├── Project (A playbooks) ├── Project (B playbooks)├── Project (C playbooks)
├── Credentials (A)      ├── Credentials (B)      ├── Credentials (C)
├── Users: customer_a_1  ├── Users: customer_b_1  ├── Users: customer_b_1
└── MSP-Ops (Admin)      └── MSP-Ops (Admin)      └── MSP-Ops (Admin)
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Cross-customer visibility | Risk of accidental access | Zero — org isolation |
| MSP efficiency | 50 separate AAP installs | One AAP, 50 orgs |
| Customer self-service | None | Customers can run their own jobs (with limits) |
| Audit | Complex | Per-org job history, clear ownership |

**Key takeaway:** Organizations provide hard boundaries. RBAC within each org provides fine-grained control. Together they enable secure multi-tenant automation.

---

## Troubleshooting

### User Cannot See Job Template

| Cause | Fix |
|-------|-----|
| Not in organization | Add user to the org that owns the template |
| No Execute role | Grant Execute on the job template to user or their team |
| No Use on credential | Grant Use on the credential used by the template |
| No Use on inventory | Grant Use on the inventory used by the template |

### User Cannot Launch Job

| Cause | Fix |
|-------|-----|
| Missing credential access | Grant **Use** on the Machine credential |
| Missing inventory access | Grant **Use** on the inventory |
| Missing project access | Grant at least **Read** on the project |
| Template not visible | Grant **Execute** (or **Read** to see it) on the job template |

### User Sees "Permission Denied" on Resource

| Cause | Fix |
|-------|-----|
| Read-only role | User has Read, not Execute/Member/Admin |
| Wrong organization | Resource is in another org; user is not a member |
| Team not granted role | Add the role to the team, or grant directly to user |

### Cannot Create Organization

| Cause | Fix |
|-------|-----|
| Self-support license | Only Default org allowed; use teams within Default |
| Not system admin | Only System Administrators can create orgs |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Organization | Top-level container; isolates users, teams, resources |
| Multi-tenancy | Multiple orgs = multiple tenants (customers, departments) |
| User types | Normal, System Administrator, System Auditor |
| Team | Group of users; permissions granted to team flow to members |
| RBAC | Role-based access; assign roles on resources to users/teams |
| Organization Admin | Full control within an org |
| Member | Edit access (projects, inventories) |
| Execute | Run job templates |
| Use | Use credential or inventory in jobs |
| Read | View only |
| Object-level permissions | Roles are per resource, not global |

---

## Lab Completion Checklist

- [ ] New organization(s) created (or used Default)
- [ ] Users created (dev_alice, ops_bob, audit_carol)
- [ ] Users added to organization
- [ ] Teams created (Dev-Team, Ops-Team, Audit-Team)
- [ ] Users assigned to teams
- [ ] Dev-Team granted Member on project, Execute on template, Use on inventory and credential
- [ ] Ops-Team granted Execute on template, Use on inventory and credential
- [ ] Audit-Team granted Read on project, inventory, template
- [ ] Logged in as dev_alice — can edit project and launch job
- [ ] Logged in as ops_bob — can launch job, cannot edit project
- [ ] Logged in as audit_carol — can view, cannot launch or edit
- [ ] Organization Administrator assigned (dev_alice)
- [ ] Can explain the MSP use case for org-based customer isolation

---

## Quick Reference — Role Summary

| Role | Typical Use |
|------|-------------|
| **Admin** | Full control on a resource |
| **Member** | Edit (projects, inventories, teams) |
| **Execute** | Run job templates |
| **Read** | View only |
| **Use** | Use credential or inventory (no edit) |
| **Update** | Sync project |
| **Ad Hoc** | Run ad hoc commands on inventory |

---

## Quick Reference — Permission Requirements to Run a Job

| Resource | Minimum Role |
|----------|--------------|
| Organization | Member |
| Project | Read |
| Inventory | Use |
| Credential | Use |
| Job Template | Execute |

---

*End of Lab 06*
