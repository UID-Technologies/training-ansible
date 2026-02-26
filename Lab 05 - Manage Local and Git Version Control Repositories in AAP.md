# Lab 05 — Manage Local and Git/Version Control Repositories in AAP

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform Engineers)               |
| **Duration** | 3 hours (Theory: 30% / Hands-on: 70%)                                     |
| **Outcome**  | Understand version control and Git basics, create and organize automation content in Git, connect AAP to GitHub/GitLab, configure branch selection and project sync, and set up webhooks for automatic synchronization |
| **Platform** | Red Hat Ansible Automation Platform 2.4 / 2.5                              |
| **Prerequisite** | Lab 02 completed — AAP installed. Lab 04 recommended (credentials and projects basics). |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / WORKSTATION                                    │
│                         - Git client (git)                                           │
│                         - Code editor (VS Code, etc.)                                │
│                         - Browser: https://aap1.lab.local                            │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          │ Git push (HTTPS/SSH)
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    GITHUB / GITLAB (Cloud or Self-Hosted)                            │
│                                                                                      │
│  Repository: ansible-automation-lab                                                 │
│  Branches: main, develop, staging, production                                        │
│  Content: playbooks/, roles/, group_vars/, requirements.yml                           │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          │ Project clone (Git)
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE AUTOMATION PLATFORM (aap1.lab.local)                      │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐
│  │  PROJECTS                                                                     │
│  │  - Lab-Git-Project (main branch)                                               │
│  │  - Lab-Git-Project-Staging (staging branch)                                   │
│  │  - Lab-Manual-Project (local /var/lib/awx/projects/)                          │
│  │                                                                               │
│  │  Sync: Manual, Update on launch, Webhook-triggered                             │
│  └──────────────────────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) |
| Managed Nodes | 1–2 Linux VMs for testing playbooks (from Lab 02 or 04) |
| GitHub or GitLab account | Free tier is sufficient |
| Git on your laptop | `git` command-line tool installed |
| SCM Credential | Created in Lab 04 (if not, you will create one in this lab) |

---

## Part 1 — Theory: Version Control and Projects (55 minutes)

### Section 1 — Introduction to Version Control

#### What Is Version Control and Why It Matters

**Version control** is a system that tracks changes to files over time. Every change is stored with metadata (who, when, why), enabling:

| Benefit | Description |
|---------|-------------|
| **History** | See what changed, when, and by whom |
| **Rollback** | Revert to a previous state if something breaks |
| **Collaboration** | Multiple people work on the same codebase without overwriting each other |
| **Audit trail** | Every change is tracked for compliance and troubleshooting |
| **Branching** | Test changes in isolation before merging to production |

For automation code, version control is essential — you cannot rely on "the last working copy" when managing hundreds of servers.

#### Introduction to Git Basics for Automation

**Git** is the most widely used version control system. Key concepts:

| Concept | Description |
|---------|-------------|
| **Repository (repo)** | A folder containing your project files and Git metadata (`.git`) |
| **Commit** | A snapshot of the project at a point in time — like a "save point" |
| **Branch** | A parallel line of development — e.g., `main` (production), `develop` (new features) |
| **Tag** | A named pointer to a specific commit — e.g., `v1.0.0` for releases |
| **Clone** | Copy a remote repository to your local machine |
| **Push** | Send your local commits to the remote repository |
| **Pull** | Fetch and merge changes from the remote repository |

#### Benefits of Storing Automation Code in Git

| Benefit | Impact |
|---------|--------|
| **Single source of truth** | AAP pulls from Git — no manual file copies |
| **Audit compliance** | Every change is logged with author and timestamp |
| **Environment promotion** | Use branches for dev → staging → production |
| **Code review** | Pull requests enable peer review before merge |
| **Disaster recovery** | Repo can be cloned from any backup location |
| **CI/CD integration** | Pipelines trigger on Git events (push, merge) |

#### Git Terminology Quick Reference

| Term | Meaning |
|------|---------|
| **Repository** | The project container — local or remote |
| **Commit** | A saved change with a message and hash (e.g., `a1b2c3d`) |
| **Branch** | A movable pointer to commits — `main` is the default |
| **Tag** | Immutable pointer to a commit — used for releases |
| **Remote** | The server hosting the repo (e.g., `origin` → GitHub) |
| **Merge** | Combine changes from one branch into another |
| **Merge conflict** | Two branches changed the same lines — must be resolved manually |

---

### Section 2 — Understanding Projects in AAP

#### What Is a Project in Ansible Automation Platform?

A **project** is the link between AAP and your automation content source. It defines:

- **Where** the content comes from (Git URL, local path, or manual directory)
- **What** branch or revision to use
- **How** and **when** to update (sync)

When a job template runs, AAP uses the project's content (playbooks, roles, variables) to execute the playbook.

#### Manual Projects vs. SCM Projects

| Type | Source | Use Case |
|------|--------|----------|
| **Manual** | Local directory on the AAP server (`/var/lib/awx/projects/`) | Labs, quick tests, content that doesn't change often |
| **Git** | Remote Git repository (GitHub, GitLab, Bitbucket, internal Git) | Production — version control, collaboration, audit |
| **Subversion (SVN)** | SVN repository | Legacy environments — less common than Git |

#### How AAP Retrieves Automation Content

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────┐
│  Git Repository │     │  AAP Project    │     │  Job Template Execution  │
│  (GitHub)       │────▶│  Clone/Pull    │────▶│  Uses playbooks from    │
│                 │     │  to local disk  │     │  project directory      │
│  main branch    │     │  /var/lib/awx/   │     │  (e.g., ping.yml)       │
│  playbooks/     │     │  projects/     │     │                          │
└─────────────────┘     └─────────────────┘     └─────────────────────────┘
```

1. AAP clones the repository (or pulls updates) to a local directory
2. The directory is owned by the `awx` user
3. When a job runs, the execution node reads playbooks from this directory
4. Each project has its own directory

#### Project Synchronization Concepts

| Sync Trigger | When It Happens |
|--------------|-----------------|
| **Manual** | User clicks the **Sync** button in the UI |
| **Update on launch** | Automatically syncs when a job template using this project is launched |
| **Update on project update** | Syncs when the project itself is updated (e.g., branch changed) |
| **Schedule** | Configure a schedule to sync periodically (e.g., every hour) |
| **Webhook** | External system (GitHub, GitLab) sends HTTP POST to AAP when code is pushed — triggers sync |

---

### Section 3 — Organizing Automation Content

#### Recommended Directory Structure

```
ansible-automation-repo/
├── playbooks/              # Top-level playbooks
│   ├── site.yml            # Main orchestration playbook
│   ├── deploy.yml
│   ├── patch.yml
│   └── network/
│       ├── config.yml
│       └── backup.yml
├── roles/                  # Reusable roles (or use collections)
│   ├── common/
│   ├── nginx/
│   └── postgresql/
├── group_vars/             # Group-based variables
│   ├── all/
│   ├── webservers/
│   └── prod/
├── host_vars/              # Host-specific variables (optional)
├── inventory/              # Static inventory files (optional)
├── requirements.yml        # Ansible Galaxy / collections dependencies
├── ansible.cfg             # Optional project-level config
└── README.md
```

#### Where to Place Roles and Variables

| Content | Location | Purpose |
|---------|----------|---------|
| **Playbooks** | `playbooks/` or repo root | Entry points for automation |
| **Roles** | `roles/` | Reusable, modular automation units |
| **Group variables** | `group_vars/<group_name>/` | Variables for inventory groups |
| **Host variables** | `host_vars/<hostname>/` | Host-specific overrides |
| **Collections** | `requirements.yml` | External dependencies (e.g., `amazon.aws`) |

#### Understanding requirements.yml

The `requirements.yml` file declares dependencies for **collections** and **roles** from Ansible Galaxy or Automation Hub:

```yaml
---
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: ansible.posix
    version: ">=1.5.0"

roles:
  - name: geerlingguy.nginx
    version: "3.1.0"
```

When AAP syncs a project:

1. If `requirements.yml` exists, AAP installs the listed collections/roles
2. They become available to playbooks in that project
3. Without this file, only built-in and pre-installed content is available

#### Best Practices for Content Organization

| Practice | Description |
|----------|-------------|
| **One repo per logical domain** | e.g., `infra-automation`, `network-automation`, `app-deployment` |
| **Use branches for environments** | `develop`, `staging`, `production` — each branch maps to an AAP project |
| **Keep playbooks small** | Orchestrate with `site.yml`; put logic in roles |
| **Document in README** | Explain structure, how to run, prerequisites |
| **Use .gitignore** | Exclude secrets, `*.retry`, `*.pyc` |
| **Pin collection versions** | Use `version: ">=2.0.0"` or exact version in `requirements.yml` |

---

## Part 2 — Hands-On: Create a Git Repository with Automation Content (40 minutes)

### Step 1 — Create a GitHub Repository

1. Log in to [GitHub](https://github.com) (or GitHub Enterprise)
2. Click **New repository**
3. **Repository name:** `ansible-automation-lab`
4. **Visibility:** Private (recommended for lab) or Public
5. **Initialize with:** Do NOT add README (we will add files manually)
6. Click **Create repository**

### Step 2 — Create the Local Directory Structure

On your laptop or workstation:

```bash
mkdir -p ansible-automation-lab/{playbooks,roles/common/tasks,group_vars/all,inventory}
cd ansible-automation-lab
```

### Step 3 — Create a Simple Playbook

```bash
cat > playbooks/ping.yml << 'EOF'
---
- name: Ping all hosts
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping
      ansible.builtin.ping:
EOF
```

### Step 4 — Create a Site Playbook

```bash
cat > playbooks/site.yml << 'EOF'
---
- name: Main site playbook
  hosts: all
  gather_facts: true
  tasks:
    - name: Display hostname
      ansible.builtin.debug:
        msg: "Host {{ inventory_hostname }} is running"
EOF
```

### Step 5 — Create Group Variables

```bash
cat > group_vars/all/all.yml << 'EOF'
---
# Global variables for all hosts
ansible_user: ec2-user
env: lab
EOF
```

### Step 6 — Create requirements.yml

```bash
cat > requirements.yml << 'EOF'
---
collections:
  - name: ansible.posix
    version: ">=1.5.0"
EOF
```

### Step 7 — Create a README

```bash
cat > README.md << 'EOF'
# Ansible Automation Lab

Repository for AAP Lab 05 - Version Control and Projects.

## Structure

- `playbooks/` - Playbooks
- `roles/` - Reusable roles
- `group_vars/` - Group variables
- `requirements.yml` - Collection dependencies

## Branches

- `main` - Production-ready
- `develop` - Development
- `staging` - Staging environment
EOF
```

### Step 8 — Create .gitignore

```bash
cat > .gitignore << 'EOF'
*.retry
*.pyc
__pycache__/
.vault_pass
*.log
EOF
```

### Step 9 — Initialize Git and Push to GitHub

```bash
cd ansible-automation-lab
git init
git add .
git commit -m "Initial commit - Lab 05 automation content"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/ansible-automation-lab.git
git push -u origin main
```

> **Replace** `YOUR_USERNAME` with your GitHub username. If the repo is private, you will be prompted for credentials (use a Personal Access Token, not your password).

---

## Part 3 — Hands-On: Connect AAP to the Git Repository (45 minutes)

### Step 10 — Create or Verify SCM Credential

1. Log in to AAP: `https://aap1.lab.local`
2. Navigate to **Resources --> Credentials**
3. If you do not have a GitHub credential from Lab 04, click **Add**:
   - **Name:** `GitHub-Personal-Access-Token`
   - **Organization:** `Default`
   - **Credential Type:** `Source Control`
   - **Source Control Type:** `Git`
   - **Username:** your GitHub username (or leave blank)
   - **Password:** Your Personal Access Token (GitHub → Settings → Developer settings → Personal access tokens)
4. Click **Save**

### Step 11 — Create a Project from Git

1. Navigate to **Resources --> Projects**
2. Click **Add**
3. Fill in:
   - **Name:** `Lab-Git-Project`
   - **Organization:** `Default`
   - **Source Control Type:** `Git`
   - **Source Control URL:** `https://github.com/YOUR_USERNAME/ansible-automation-lab.git`
   - **Source Control Credential:** `GitHub-Personal-Access-Token`
   - **Source Control Branch/Tag/Commit:** `main` (leave blank for default branch)
   - **Update on launch:** Checked
   - **Clean:** Unchecked (for now)
   - **Delete on update:** Unchecked
4. Click **Save**

### Step 12 — Sync the Project

1. Click the **Sync** icon (circular arrows) next to the project
2. Wait for the sync to complete — status should show a green check
3. Click on the project name
4. Verify **Revision** shows the latest commit hash
5. Check **Playbooks** — you should see `ping.yml` and `site.yml` listed

### Step 13 — Create a Job Template and Run a Playbook

1. Navigate to **Resources --> Templates**
2. Click **Add --> Add job template**
3. Fill in:
   - **Name:** `Ping-from-Git`
   - **Inventory:** `Lab-Inventory` (or your inventory from Lab 02/04)
   - **Project:** `Lab-Git-Project`
   - **Playbook:** `playbooks/ping.yml`
   - **Credential:** Your Machine credential
4. Click **Save**
5. Click **Launch**
6. Verify the job completes successfully — the playbook was pulled from Git

### Step 14 — Test "Update on Launch"

1. In your local repo, edit `playbooks/ping.yml` — add a small change (e.g., a comment)
2. Commit and push:

```bash
cd ansible-automation-lab
echo "# Updated from Lab 05" >> playbooks/ping.yml
git add playbooks/ping.yml
git commit -m "Add comment to ping playbook"
git push
```

3. In AAP, **Launch** the `Ping-from-Git` job again (do NOT manually sync)
4. AAP will sync the project automatically before the job runs
5. Verify the job completes — the updated content was pulled

---

## Part 4 — Hands-On: Branch Selection and Multiple Projects (35 minutes)

### Step 15 — Create the `develop` Branch

```bash
cd ansible-automation-lab
git checkout -b develop
git push -u origin develop
```

### Step 16 — Create the `staging` Branch

```bash
git checkout -b staging
git push -u origin staging
git checkout main
```

### Step 17 — Create a Project for the Staging Branch

1. **Resources --> Projects --> Add**
2. Fill in:
   - **Name:** `Lab-Git-Project-Staging`
   - **Organization:** `Default`
   - **Source Control Type:** `Git`
   - **Source Control URL:** `https://github.com/YOUR_USERNAME/ansible-automation-lab.git`
   - **Source Control Credential:** `GitHub-Personal-Access-Token`
   - **Source Control Branch/Tag/Commit:** `staging`
   - **Update on launch:** Checked
3. Click **Save**
4. Click **Sync** to pull the staging branch

### Step 18 — Create a Job Template for Staging

1. **Resources --> Templates --> Add --> Add job template**
2. Fill in:
   - **Name:** `Ping-Staging`
   - **Inventory:** `Lab-Inventory` (or a staging-specific inventory)
   - **Project:** `Lab-Git-Project-Staging`
   - **Playbook:** `playbooks/ping.yml`
   - **Credential:** Your Machine credential
3. Click **Save**

### Step 19 — Test Branch Isolation

1. On `develop` branch, make a change:

```bash
git checkout develop
echo "env: develop" >> group_vars/all/all.yml
git add group_vars/all/all.yml
git commit -m "Set env to develop"
git push
```

2. Merge `develop` into `staging` (optional — simulates promotion):

```bash
git checkout staging
git merge develop
git push
```

3. In AAP, **Launch** `Ping-Staging` — it uses content from the `staging` branch
4. **Launch** `Ping-from-Git` — it uses content from the `main` branch
5. Different branches = different content = different behavior per environment

---

## Part 5 — Hands-On: Webhook for Automatic Synchronization (30 minutes)

### Step 20 — Enable Webhook in AAP

1. Open the `Lab-Git-Project` project
2. Note the **Webhook URL** (or find it under **Details** section)
3. It looks like: `https://aap1.lab.local/api/v2/project_update/<ID>/`

> **Note:** For webhooks to work, AAP must be reachable from the internet (or from GitHub's servers). In a lab environment, this may require a tunnel (ngrok, etc.) or a public IP. If your AAP is not reachable, skip webhook and use "Update on launch" or manual sync.

### Step 21 — Configure GitHub Webhook

1. In GitHub, open your repository
2. Settings --> Webhooks --> Add webhook
3. Fill in:
   - **Payload URL:** `https://aap1.lab.local/api/v2/project_update/<PROJECT_UPDATE_ID>/`

> **Important:** The correct AAP webhook URL format is typically:
> `https://<AAP_HOST>/api/v2/projects/<PROJECT_ID>/update/`
>
> Or with a webhook key:
> `https://<AAP_HOST>/api/v2/projects/<PROJECT_ID>/update/?token=<WEBHOOK_KEY>`

4. **Content type:** `application/json`
5. **Which events:** "Just the push event"
6. **Active:** Checked
7. Click **Add webhook**

### Step 22 — Verify Webhook (If AAP Is Reachable)

1. In your local repo, make a small change and push:

```bash
git checkout main
echo "# Webhook test" >> playbooks/ping.yml
git add playbooks/ping.yml
git commit -m "Webhook test"
git push
```

2. In GitHub, go to **Settings --> Webhooks** and click on your webhook
3. Check **Recent Deliveries** — you should see a successful delivery (200 response)
4. In AAP, open `Lab-Git-Project` and check **Recent project updates** — a sync should have been triggered by the webhook

### Step 23 — Alternative: Manual and Scheduled Sync

If webhooks are not feasible in your lab:

1. **Manual sync:** Click the Sync icon whenever you push changes
2. **Update on launch:** Already configured — every job launch syncs first
3. **Schedule:** Create a schedule (e.g., every 15 minutes) that runs a project update job

---

## Part 6 — Hands-On: Organize Content with Roles and requirements.yml (25 minutes)

### Step 24 — Create a Simple Role

```bash
cd ansible-automation-lab
mkdir -p roles/hello/tasks
cat > roles/hello/tasks/main.yml << 'EOF'
---
- name: Say hello
  ansible.builtin.debug:
    msg: "Hello from role {{ role_name }} on {{ inventory_hostname }}"
EOF
```

### Step 25 — Create a Playbook That Uses the Role

```bash
cat > playbooks/role-demo.yml << 'EOF'
---
- name: Demo role usage
  hosts: all
  gather_facts: false
  roles:
    - hello
EOF
```

### Step 26 — Update requirements.yml

```bash
cat > requirements.yml << 'EOF'
---
collections:
  - name: ansible.posix
    version: ">=1.5.0"
  - name: ansible.utils
    version: ">=2.0.0"
EOF
```

### Step 27 — Commit and Push

```bash
git add .
git commit -m "Add hello role and role-demo playbook"
git push
```

### Step 28 — Sync and Run in AAP

1. In AAP, sync `Lab-Git-Project`
2. Create a job template:
   - **Name:** `Role-Demo`
   - **Project:** `Lab-Git-Project`
   - **Playbook:** `playbooks/role-demo.yml`
   - **Inventory:** `Lab-Inventory`
   - **Credential:** Machine credential
3. Click **Save** and **Launch**
4. Verify the job runs — the role is executed and collections from `requirements.yml` are installed during sync

---

## Part 7 — Hands-On: Manual Project (Local Directory) (15 minutes)

### Step 29 — Create a Manual Project on AAP Server

SSH to the AAP server:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-manual
sudo tee /var/lib/awx/projects/lab-manual/ping.yml << 'EOF'
---
- name: Ping from manual project
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping
      ansible.builtin.ping:
EOF
sudo chown -R awx:awx /var/lib/awx/projects/lab-manual
```

### Step 30 — Create Manual Project in AAP UI

1. **Resources --> Projects --> Add**
2. Fill in:
   - **Name:** `Lab-Manual-Project`
   - **Organization:** `Default`
   - **Source Control Type:** `Manual`
   - **Playbook Directory:** `lab-manual` (select from dropdown)
3. Click **Save**

### Step 31 — Compare Manual vs. Git Project

| Aspect | Manual Project | Git Project |
|--------|----------------|-------------|
| **Source** | Local directory on AAP server | Remote Git repository |
| **Updates** | Manual file copy or SSH | Sync (manual, on launch, webhook) |
| **Version control** | None | Full Git history |
| **Use case** | Quick lab, one-off scripts | Production, collaboration |

---

## Part 8 — Real-Time Use Case Discussion (20 minutes)

### Case Study: Telecommunications Company — Git Branching for 1000+ Network Devices

**Background:**

A telecommunications company manages 1,000+ network devices (routers, switches, firewalls) across multiple regions. Automation code is used for configuration updates, compliance checks, and firmware upgrades.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Changes must be tested before production | A bad config can take down network segments |
| Multiple teams (network, security, NOC) | Need clear separation of changes |
| Audit requirements | Every change must be traceable to a commit and approval |
| Rollback capability | Must revert quickly if a deployment fails |

**Solution: Git Branching Strategy**

| Branch | Purpose | AAP Project | Inventory |
|--------|---------|-------------|-----------|
| `develop` | New feature development, testing | `Network-Automation-Dev` | Dev lab devices (50) |
| `staging` | Pre-production validation | `Network-Automation-Staging` | Staging devices (100) |
| `production` | Production-ready, approved code | `Network-Automation-Prod` | All production devices (1000+) |

**Workflow:**

```
Developer creates feature branch → merge to develop
    → AAP runs on dev lab → validate
    → Merge to staging → AAP runs on staging
    → Change advisory board approval
    → Merge to production → AAP runs on production
```

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Failed deployments | 5–10% (untested changes) | <1% (staged validation) |
| Rollback time | 2–4 hours (manual) | 15 minutes (revert commit + re-run) |
| Audit compliance | Partial (spreadsheets) | Full (Git history + AAP job logs) |
| Team collaboration | Conflicts, overwrites | Pull requests, code review |

**Key takeaway:** Git branches map to environments. AAP projects map to branches. Each environment gets the right code at the right time, with full traceability.

---

## Troubleshooting

### Project Sync Fails

| Symptom | Check |
|---------|-------|
| Authentication failed | Verify SCM credential; token may have expired |
| Repository not found | Check URL; ensure repo exists and is accessible |
| Permission denied | Token needs `repo` scope (GitHub) or `read_repository` (GitLab) |
| Branch not found | Verify branch name; default is `main` or `master` |
| Timeout | Large repo or slow network; increase sync timeout in project settings |

### Playbook Not Found After Sync

| Cause | Fix |
|-------|-----|
| Wrong path | Playbook path is relative to project root; use `playbooks/ping.yml` not `ping.yml` if in subfolder |
| Sync failed | Check project sync status; fix errors and re-sync |
| Branch mismatch | Ensure correct branch is selected; sync again |

### requirements.yml Not Installing Collections

| Cause | Fix |
|-------|-----|
| Wrong format | Ensure valid YAML; check collection names (e.g., `amazon.aws` not `ansible.amazon.aws`) |
| Network issues | Execution node needs internet to reach Galaxy/Hub |
| Version constraint | Try `version: ">=1.0.0"` or exact version |

### Webhook Not Triggering Sync

| Cause | Fix |
|-------|-----|
| AAP not reachable | GitHub must reach AAP; use ngrok or public IP for lab |
| Wrong URL | Use correct AAP webhook URL format from project details |
| Invalid payload | Ensure Content-Type is `application/json` |
| Firewall | Allow HTTPS (443) from GitHub IP ranges |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Version control | Git tracks changes; enables history, rollback, collaboration |
| Repository | Container for project files; local and remote |
| Branch | Parallel line of development; map to environments |
| Project | AAP's link to automation content source |
| Manual project | Local directory on AAP server |
| SCM project | Git (or SVN) repository — production standard |
| Source Control Branch | Branch/tag/commit to use in AAP |
| Update on launch | Automatic sync when job runs |
| Webhook | External trigger (e.g., Git push) to sync project |
| requirements.yml | Declares collections and roles for install during sync |
| Directory structure | playbooks/, roles/, group_vars/, inventory/ |

---

## Lab Completion Checklist

- [ ] Git repository created on GitHub with automation content
- [ ] Local directory structure created (playbooks, roles, group_vars)
- [ ] `requirements.yml` created with at least one collection
- [ ] SCM credential created and tested
- [ ] AAP project created from Git repository
- [ ] Project synced successfully
- [ ] Job template runs playbook from Git project
- [ ] "Update on launch" verified — push change, launch job, new content used
- [ ] `develop` and `staging` branches created
- [ ] Second AAP project created for `staging` branch
- [ ] Job template for staging branch runs successfully
- [ ] Webhook configured (if AAP is reachable) or alternative sync method documented
- [ ] Manual project created and compared to Git project
- [ ] Role created and used in a playbook
- [ ] Can explain the telecom Git branching use case

---

## Quick Reference — Project Source Types

| Type | Description |
|------|-------------|
| Manual | Local directory on AAP server |
| Git | GitHub, GitLab, Bitbucket, internal Git |
| Subversion | SVN repository |

---

## Quick Reference — Project Sync Options

| Option | Description |
|--------|-------------|
| Update on launch | Sync before each job that uses this project |
| Update on project update | Sync when project config changes |
| Clean | Delete local files before sync (fresh clone) |
| Delete on update | Remove project directory before sync |
| Schedule | Create a schedule to run project update |
| Webhook | Trigger sync via HTTP POST from external system |

---

*End of Lab 05*
