# Lab 15 — Create, Manage, and Publish Ansible Content Collections

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate to Advanced (Ops / DevOps / Platform Engineers)               |
| **Duration** | 5 hours (Theory: 30% / Hands-on: 70%)                                     |
| **Outcome**  | Understand Ansible collections (structure, namespaces, benefits); create a collection with roles and plugins; document and test; build tarballs; publish to Private Automation Hub; consume collections in playbooks via requirements.yml and FQCN |
| **Platform** | Red Hat Ansible Automation Platform **2.4** — Automation Controller, Private Automation Hub (optional) |
| **Prerequisite** | Labs 02, 04, 05, 09 completed — AAP 2.4 installed, project with playbooks, Red Hat account. Lab 09 recommended for Hub concepts. |

---

## AAP 2.4 and Automation Hub Reference

| Component | Purpose |
|-----------|---------|
| **Automation Controller** | Run playbooks; projects pull from Git; job templates use collections |
| **Private Automation Hub** | Host custom collections; publish and consume internally |
| **Public Automation Hub** | console.redhat.com — certified/validated collections |
| **Ansible Galaxy** | galaxy.ansible.com — community collections |

> **Note:** This lab uses `ansible-galaxy` CLI for collection development. Publishing to Private Hub requires Hub to be installed (Lab 09). If Hub is not available, you can build and install the collection locally.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / WORKSTATION                                    │
│                         - ansible-galaxy CLI                                         │
│                         - Code editor, Git                                           │
│                         - Browser: https://aap1.lab.local, https://hub.lab.local    │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
│  COLLECTION       │      │  AAP CONTROLLER    │      │  PRIVATE HUB       │
│  (local dev)      │      │  (aap1.lab.local)  │      │  (hub.lab.local)   │
│  - galaxy init    │      │  - Projects        │      │  - Upload .tar.gz  │
│  - roles/         │      │  - requirements.yml│      │  - Namespace       │
│  - plugins/       │      │  - Job templates   │      │  - Versioning     │
│  - galaxy build   │      │  - FQCN in playbooks│     │  - Distribution   │
└───────────────────┘      └───────────────────┘      └───────────────────┘
                                       │
                                       ▼
                            ┌───────────────────┐
                            │  MANAGED NODES    │
                            │  web1, app1, db1  │
                            └───────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Ansible Core | Installed on workstation (for ansible-galaxy, ansible-test) |
| Python 3.8+ | Required for collection development |
| Git | For version control (optional but recommended) |
| Red Hat account | For Public Hub; namespace for Galaxy |
| Optional | Private Automation Hub (Lab 09) for publishing |

---

## Verify Environment Before Starting

```bash
ansible-galaxy --version
ansible --version
python3 --version
```

---

## Part 1 — Theory: Collections, Structure, Creation, Documentation, Publishing, Consumption (90 minutes)

### Section 14.1 — Understanding Ansible Collections

#### What Are Ansible Collections?

**Collections** are the standard way to package and distribute Ansible content:

| Component | Purpose |
|-----------|---------|
| **Modules** | Reusable automation units (e.g., `my_collection.my_module`) |
| **Roles** | Pre-built playbooks and tasks |
| **Plugins** | Filters, lookups, inventory, connection plugins |
| **Playbooks** | Example or reusable playbooks |

#### Evolution from Roles to Collections

| Era | Format | Limitation |
|-----|--------|------------|
| **Pre-collections** | Standalone roles (Ansible Galaxy) | No modules; role-only; naming conflicts |
| **Collections** | Namespace + name (e.g., `acme.deploy`) | Modules, roles, plugins; versioned; dependencies |

#### Collection Structure Explained

```
my_namespace.my_collection/
├── galaxy.yml           # Metadata, version, dependencies
├── README.md
├── meta/
│   └── runtime.yml      # Python/Ansible version requirements
├── roles/
│   └── my_role/
├── plugins/
│   ├── modules/
│   ├── filter_plugins/
│   └── lookup_plugins/
├── playbooks/
└── docs/
```

#### Benefits of Using Collections

| Benefit | Description |
|---------|-------------|
| **Reusability** | Share across teams and projects |
| **Versioning** | Semantic versioning (1.0.0, 1.1.0) |
| **Dependencies** | Declare other collections |
| **Namespace** | Avoid naming conflicts |
| **Distribution** | Hub, Galaxy, or internal repo |

#### Namespaces and Naming Conventions

| Convention | Example |
|------------|---------|
| **Namespace** | Lowercase; organization or product (e.g., `acme`, `redhat`) |
| **Collection name** | Lowercase; descriptive (e.g., `deploy`, `rhel`) |
| **FQCN** | `namespace.collection.module` or `namespace.collection.role` |

---

### Section 14.2 — Collection Structure Deep Dive

#### Recommended Directory Layout

```
acme.deploy/
├── galaxy.yml
├── README.md
├── meta/
│   └── runtime.yml
├── roles/
│   └── app_deploy/
│       ├── tasks/
│       ├── defaults/
│       ├── handlers/
│       └── meta/
├── plugins/
│   ├── modules/
│   │   └── my_module.py
│   ├── filter_plugins/
│   └── lookup_plugins/
├── playbooks/
│   └── deploy.yml
└── docs/
```

#### galaxy.yml Configuration File

```yaml
---
namespace: acme
name: deploy
version: 1.0.0
readme: README.md
authors:
  - Your Name <you@example.com>
description: Application deployment collection
license: MIT
tags:
  - deploy
  - application
repository: https://github.com/acme/ansible-deploy
documentation: https://github.com/acme/ansible-deploy/blob/main/README.md
```

#### README and Documentation Requirements

- **README.md** — Required; describes collection, installation, usage
- **docs/** — Optional; detailed docs
- **Module docstrings** — Required for modules; become docs

#### roles/ Directory

- Same structure as standalone roles
- Referenced as `acme.deploy.app_deploy`

#### plugins/ Directory

| Subdirectory | Contents |
|--------------|----------|
| `modules/` | Python module files |
| `filter_plugins/` | Custom filters |
| `lookup_plugins/` | Custom lookups |
| `inventory/` | Inventory plugins |
| `connection/` | Connection plugins |

#### playbooks/ Directory

- Example or reusable playbooks
- Can be run with `ansible-playbook acme.deploy.playbooks.deploy`

---

### Section 14.3 — Creating Your First Collection

#### Initializing a Collection with ansible-galaxy

```bash
ansible-galaxy collection init acme.deploy
```

Creates directory structure with `galaxy.yml`, `meta/runtime.yml`, placeholder dirs.

#### Adding Roles to a Collection

```bash
cd acme.deploy
ansible-galaxy role init roles.app_deploy
# Or manually create roles/app_deploy/tasks/main.yml, etc.
```

#### Adding Modules (Overview)

- Create Python file in `plugins/modules/my_module.py`
- Include docstring, `DOCUMENTATION`, `EXAMPLES`, `RETURN` blocks
- Implement `run_module()` or `AnsibleModule`

#### Adding Plugins to a Collection

- **Filter:** `plugins/filter_plugins/my_filter.py` with `FilterModule`
- **Lookup:** `plugins/lookup_plugins/my_lookup.py` with `LookupModule`

#### Managing Dependencies in meta/runtime.yml

```yaml
---
requires_ansible: ">=2.14.0"
dependencies:
  ansible.posix: ">=1.5.0"
  community.general: ">=5.0.0"
```

---

### Section 14.4 — Documentation and Testing

#### Documenting Your Collection

- **README.md** — Installation, quick start, role/module list
- **Module docstrings** — DOCUMENTATION, EXAMPLES, RETURN
- **Role README** — Role-specific vars, examples

#### README Best Practices

- Clear title and description
- Installation instructions
- Usage examples
- Requirements (Ansible version, collections)
- License

#### Module Documentation Standards

```python
DOCUMENTATION = r'''
---
module: my_module
short_description: Does something
description: Longer description.
options:
  param1:
    description: Parameter 1
    required: true
    type: str
'''

EXAMPLES = r'''
- name: Example
  acme.deploy.my_module:
    param1: value
'''

RETURN = r'''
result:
  description: Result
  type: str
  returned: always
'''
```

#### Testing Collections (Introduction)

- **ansible-test** — Ansible's test framework
- Run sanity: `ansible-test sanity`
- Run integration: `ansible-test integration` (requires VMs)

#### Using ansible-test (Basic Overview)

```bash
ansible-test sanity --docker
ansible-test integration my_module --docker
```

---

### Section 14.5 — Building and Publishing Collections

#### Building Collection Tarballs

```bash
ansible-galaxy collection build
```

Produces `acme-deploy-1.0.0.tar.gz` in current directory.

#### Version Management and Semantic Versioning

| Format | Meaning |
|--------|---------|
| **MAJOR.MINOR.PATCH** | 1.0.0 |
| **MAJOR** | Breaking changes |
| **MINOR** | New features, backward compatible |
| **PATCH** | Bug fixes |

#### Publishing to Private Automation Hub

1. Log in to Private Hub
2. Create namespace (if not exists)
3. **Collections** → **Upload** → select `.tar.gz`
4. Approve (if content approval enabled)
5. Collection available for install

#### Publishing to Ansible Galaxy (Overview)

1. Create Galaxy account; create namespace
2. `ansible-galaxy collection publish acme-deploy-1.0.0.tar.gz --api-key=<key>`
3. Or use GitHub integration for automated publishing

#### Updating and Maintaining Collections

- Bump version in `galaxy.yml`
- Rebuild and republish
- Use tags in Git for releases

---

### Section 14.6 — Using Collections in Playbooks

#### Installing Collections from Hub

```bash
ansible-galaxy collection install acme.deploy
# From Hub (with token):
ansible-galaxy collection install acme.deploy -p /path --server https://hub.lab.local/api/galaxy/
```

#### requirements.yml for Collections

```yaml
---
collections:
  - name: acme.deploy
    version: ">=1.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
```

#### Fully Qualified Collection Names (FQCN)

| Usage | Example |
|-------|---------|
| Role | `acme.deploy.app_deploy` |
| Module | `acme.deploy.my_module` |
| Filter | `{{ x \| acme.deploy.my_filter }}` |

#### Version Pinning Strategies

| Strategy | Example | Use Case |
|----------|---------|----------|
| Exact | `version: "1.0.0"` | Reproducible |
| Minimum | `version: ">=1.0.0"` | Get fixes |
| Range | `version: ">=1.0.0,<2.0.0"` | Avoid breaking |

#### Collection Dependencies

- Declared in `meta/runtime.yml`
- Installed automatically when installing the collection

---

## Part 2 — Hands-On: Create Collection Structure (40 minutes)

### Step 1 — Initialize a Collection

**Purpose:** Create a new collection using ansible-galaxy.

**Detailed steps:**

1. Create a working directory and initialize:

```bash
mkdir -p ~/ansible-collections-lab
cd ~/ansible-collections-lab
ansible-galaxy collection init acme.deploy
```

2. Inspect the structure:

```bash
tree acme/deploy
# Or: ls -la acme/deploy/
```

3. You should see: `galaxy.yml`, `meta/runtime.yml`, `plugins/`, `roles/`, etc.

**Verify:** Collection directory structure exists.

---

### Step 2 — Configure galaxy.yml

**Purpose:** Set metadata for the collection.

**Detailed steps:**

1. Edit `acme/deploy/galaxy.yml`:

```yaml
---
namespace: acme
name: deploy
version: 1.0.0
readme: README.md
authors:
  - Lab User <lab@example.com>
description: Application deployment collection for Lab 15
license: MIT
tags:
  - deploy
  - application
  - lab
repository: https://github.com/acme/ansible-deploy
```

2. Save the file.

**Verify:** galaxy.yml has correct namespace, name, version.

---

### Step 3 — Add a Role to the Collection

**Purpose:** Create a role inside the collection.

**Detailed steps:**

1. Create role structure:

```bash
mkdir -p acme/deploy/roles/app_deploy/{tasks,defaults,handlers,meta}
```

2. Create `acme/deploy/roles/app_deploy/tasks/main.yml`:

```yaml
---
- name: Create app directory
  ansible.builtin.file:
    path: "{{ app_install_dir | default('/opt/myapp') }}"
    state: directory
    mode: "0755"

- name: Deploy config file
  ansible.builtin.copy:
    dest: "{{ app_install_dir | default('/opt/myapp') }}/config.conf"
    content: |
      app_name={{ app_name | default('myapp') }}
      app_version={{ app_version | default('1.0') }}
      env={{ env | default('dev') }}
    mode: "0644"

- name: Display deployment info
  ansible.builtin.debug:
    msg: "Deployed {{ app_name | default('myapp') }} v{{ app_version | default('1.0') }} to {{ app_install_dir | default('/opt/myapp') }}"
```

3. Create `acme/deploy/roles/app_deploy/defaults/main.yml`:

```yaml
---
app_name: myapp
app_version: "1.0"
app_install_dir: /opt/myapp
env: dev
```

4. Create `acme/deploy/roles/app_deploy/meta/main.yml`:

```yaml
---
dependencies: []
```

**Verify:** Role structure is complete; tasks runnable.

---

### Step 4 — Add a Filter Plugin to the Collection

**Purpose:** Add a custom filter plugin.

**Detailed steps:**

1. Create `acme/deploy/plugins/filter_plugins/deploy_filters.py`:

```python
# Filter plugins for acme.deploy collection
def format_app_version(value, prefix='v'):
    """Format version string with optional prefix."""
    if not value:
        return ''
    return f"{prefix}{value}"

class FilterModule:
    def filters(self):
        return {
            'format_app_version': format_app_version,
        }
```

2. Save the file.

**Verify:** Filter plugin file exists.

---

### Step 5 — Configure meta/runtime.yml

**Purpose:** Set Ansible and collection dependencies.

**Detailed steps:**

1. Edit `acme/deploy/meta/runtime.yml`:

```yaml
---
requires_ansible: ">=2.14.0"
dependencies:
  ansible.posix: ">=1.0.0"
```

2. Save (adjust versions if needed).

**Verify:** runtime.yml has valid content.

---

## Part 3 — Hands-On: Documentation and Build (35 minutes)

### Step 6 — Create README.md

**Purpose:** Document the collection.

**Detailed steps:**

1. Create `acme/deploy/README.md`:

```markdown
# acme.deploy

Application deployment collection for Ansible.

## Installation

```bash
ansible-galaxy collection install acme.deploy
```

## Requirements

- Ansible >= 2.14
- ansible.posix >= 1.0.0

## Roles

### app_deploy

Deploys an application with configurable name, version, and directory.

**Variables:**
- `app_name` - Application name (default: myapp)
- `app_version` - Version (default: 1.0)
- `app_install_dir` - Install path (default: /opt/myapp)
- `env` - Environment (default: dev)

**Example:**
```yaml
- hosts: all
  roles:
    - role: acme.deploy.app_deploy
      vars:
        app_name: myapp
        app_version: "2.0"
        env: prod
```

## License

MIT
```

2. Save.

**Verify:** README is complete and readable.

---

### Step 7 — Build the Collection

**Purpose:** Create distributable tarball.

**Detailed steps:**

1. From the collection root (parent of `acme/`):

```bash
cd ~/ansible-collections-lab
ansible-galaxy collection build acme/deploy
```

2. Output: `acme-deploy-1.0.0.tar.gz` in current directory
3. Verify: `tar -tzf acme-deploy-1.0.0.tar.gz | head -20`

**Verify:** Tarball is created and contains expected files.

---

### Step 8 — Install and Test Collection Locally

**Purpose:** Install from tarball and run the role.

**Detailed steps:**

1. Install from tarball:

```bash
ansible-galaxy collection install acme-deploy-1.0.0.tar.gz -p ./collections --force
```

2. Create test playbook `test_deploy.yml`:

```yaml
---
- name: Test acme.deploy collection
  hosts: all
  gather_facts: true
  roles:
    - role: acme.deploy.app_deploy
      vars:
        app_name: labapp
        app_version: "1.0.0"
        env: lab
```

3. Create `ansible.cfg` to use local collections:

```ini
[defaults]
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections
```

4. Run: `ansible-playbook test_deploy.yml -i inventory` (use your inventory)

**Verify:** Role runs successfully; config file created on hosts.

---

## Part 4 — Hands-On: Publish to Private Automation Hub (40 minutes)

### Step 9 — Prepare for Publishing

**Purpose:** Ensure collection is ready for Hub upload.

**Detailed steps:**

1. Verify `galaxy.yml` has correct namespace
2. Ensure namespace exists in Private Hub (create via Hub UI if needed)
3. Rebuild: `ansible-galaxy collection build acme/deploy`

**Verify:** Tarball is ready.

---

### Step 10 — Upload to Private Automation Hub

**Purpose:** Publish collection to Private Hub.

**Detailed steps:**

1. Log in to Private Automation Hub: `https://hub.lab.local` (or your Hub URL)
2. Navigate to **Collections**
3. Select your namespace (e.g., `acme`)
4. Click **Upload** (or **Import**)
5. Select `acme-deploy-1.0.0.tar.gz`
6. Upload; wait for import
7. If content approval is enabled, approve the collection
8. Verify collection appears in the namespace

**Lab note:** If Private Hub is not available, skip to Step 11 and install from tarball or local path.

**Verify:** Collection visible in Hub under the namespace.

---

### Step 11 — Install Collection from Private Hub

**Purpose:** Install collection from Hub in AAP project.

**Detailed steps:**

1. Configure Galaxy credential for Private Hub (if not done):
   - **Resources → Credentials → Add**
   - **Credential Type:** Galaxy/Automation Hub
   - **URL:** `https://hub.lab.local/api/galaxy/` (or your Hub)
   - **Auth URL:** `https://hub.lab.local/api/galaxy/v3/auth/token/`
   - **Token:** Your Hub API token
2. In project's `requirements.yml`:

```yaml
---
collections:
  - name: acme.deploy
    version: "1.0.0"
    source: https://hub.lab.local/api/galaxy/
```

3. Add `requirements.yml` to project; sync project
4. AAP installs collection during project sync (if configured)

**Alternative (CLI):**

```bash
ansible-galaxy collection install acme.deploy -p ./collections \
  --server https://hub.lab.local/api/galaxy/ \
  --token <HUB_TOKEN>
```

**Verify:** Collection installs from Hub.

---

## Part 5 — Hands-On: Use Collection in Playbooks (30 minutes)

### Step 12 — Create requirements.yml

**Purpose:** Declare collection dependencies for a project.

**Detailed steps:**

1. Create `requirements.yml` in project root:

```yaml
---
collections:
  - name: acme.deploy
    version: "1.0.0"
  - name: ansible.posix
    version: ">=1.0.0"
```

2. Install: `ansible-galaxy collection install -r requirements.yml`

**Verify:** Collections installed.

---

### Step 13 — Use FQCN in Playbook

**Purpose:** Reference collection role and filter by FQCN.

**Detailed steps:**

1. Create playbook `use_collection.yml`:

```yaml
---
- name: Use acme.deploy collection
  hosts: all
  gather_facts: true
  vars:
    app_version: "2.0"
  tasks:
    - name: Use role via FQCN
      ansible.builtin.include_role:
        name: acme.deploy.app_deploy
      vars:
        app_name: prodapp
        app_version: "{{ app_version | acme.deploy.format_app_version }}"
        env: prod
```

2. Run the playbook

**Verify:** Role runs; custom filter formats version.

---

### Step 14 — Add Collection to AAP Project

**Purpose:** Use collection in AAP job templates.

**Detailed steps:**

1. Add `requirements.yml` to your Git project (or `lab-playbooks`)
2. Ensure project structure:

```
lab-playbooks/
├── requirements.yml
├── playbooks/
│   └── deploy_app.yml
└── ...
```

3. Sync project in AAP — collections install during sync (if `requirements.yml` present)
4. Create job template using playbook that uses `acme.deploy.app_deploy`
5. Launch job — verify it runs

**Verify:** Collection works in AAP.

---

## Part 6 — Hands-On: Version Bump and Rebuild (20 minutes)

### Step 15 — Update Version and Rebuild

**Purpose:** Practice version management and rebuild.

**Detailed steps:**

1. Edit `acme/deploy/galaxy.yml` — change `version: 1.0.0` to `version: 1.1.0`
2. Rebuild: `ansible-galaxy collection build acme/deploy`
3. New tarball: `acme-deploy-1.1.0.tar.gz`
4. Install new version: `ansible-galaxy collection install acme-deploy-1.1.0.tar.gz -p ./collections --force`

**Verify:** New version installs and works.

---

## Part 7 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Software Company — Reusable Deployment Collections

**Background:**

A software company has multiple products (WebApp, API, Worker) deployed across dev, staging, and production. Each product has similar but slightly different deployment steps. Previously, each team maintained separate playbooks — duplication, drift, and inconsistent versions.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Multiple products | WebApp, API, Worker each need deployment |
| Multiple environments | Dev, staging, prod — different configs |
| Multiple teams | Each team had own scripts |
| Version consistency | Prod ran v1.2 while staging had v1.5 |
| Reusability | 80% same logic, 20% product-specific |

**Solution: Collection-Based Deployment**

#### 1. Collection Structure

```
acme.deploy/
├── roles/
│   ├── common_setup/      # Base setup (all products)
│   ├── webapp_deploy/     # WebApp-specific
│   ├── api_deploy/        # API-specific
│   └── worker_deploy/     # Worker-specific
├── playbooks/
│   ├── deploy_webapp.yml
│   ├── deploy_api.yml
│   └── deploy_worker.yml
└── plugins/
    └── filter_plugins/
        └── env_config.py  # Env-specific defaults
```

#### 2. Single Source of Truth

- **group_vars/** — `dev.yml`, `staging.yml`, `prod.yml` with env-specific vars
- **Collection vars** — Product defaults in role defaults
- **Override at runtime** — Extra vars from AAP survey for version, env

#### 3. Version Pinning

```yaml
# requirements.yml (in each project)
collections:
  - name: acme.deploy
    version: "1.2.3"  # Pinned for reproducibility
```

- Dev project: `1.2.3` (or `>=1.2.0` for latest patch)
- Staging: `1.2.3` (same as prod candidate)
- Prod: `1.2.3` (pinned after validation)

#### 4. Publishing Workflow

1. Developer updates collection → PR
2. CI runs `ansible-test sanity`
3. Merge → build → publish to Private Hub as `1.2.4`
4. Staging project updates `requirements.yml` → `1.2.4`
5. Validate in staging
6. Prod project updates to `1.2.4` after approval

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Deployment scripts | 12 (per team/product) | 1 collection, 3 playbooks |
| Version drift | Frequent | Zero (pinned in requirements) |
| New env setup | 2 days | 30 minutes (install collection, set vars) |
| Consistency | Low | High (single collection) |

**Key takeaway:** Packaging automation as collections enables reuse across teams and environments, with version control and consistent deployment.

---

## Troubleshooting

### ansible-galaxy collection init Fails

| Cause | Fix |
|-------|-----|
| Invalid namespace/name | Use lowercase; format `namespace.name` |
| Path exists | Use new directory or remove existing |

### Build Fails

| Cause | Fix |
|-------|-----|
| Invalid galaxy.yml | Check YAML syntax; required fields |
| Missing files | Ensure roles/plugins have valid content |

### Install from Hub Fails

| Cause | Fix |
|-------|-----|
| Wrong URL | Use `/api/galaxy/` suffix |
| Auth | Verify token; check credential |
| Namespace | Ensure namespace exists in Hub |

### Role Not Found When Using FQCN

| Cause | Fix |
|-------|-----|
| Collection not installed | Install via requirements.yml or galaxy |
| Wrong FQCN | Use `namespace.collection.role_name` |
| collections_path | Check ansible.cfg or COLLECTIONS_PATHS |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|------------------|
| Collection | Packaged Ansible content (roles, modules, plugins) |
| galaxy.yml | Metadata, version, dependencies |
| meta/runtime.yml | Ansible version, collection dependencies |
| ansible-galaxy collection init | Create collection structure |
| ansible-galaxy collection build | Create tarball |
| Private Hub | Publish and distribute custom collections |
| requirements.yml | Declare and install collections |
| FQCN | namespace.collection.role or module |
| Version pinning | Exact, minimum, or range |
| Semantic versioning | MAJOR.MINOR.PATCH |

---

## Lab Completion Checklist

- [ ] Initialized collection with ansible-galaxy
- [ ] Configured galaxy.yml
- [ ] Added role to collection
- [ ] Added filter plugin to collection
- [ ] Configured meta/runtime.yml
- [ ] Created README.md
- [ ] Built collection tarball
- [ ] Installed collection from tarball
- [ ] Tested role locally
- [ ] Uploaded to Private Hub (if available)
- [ ] Installed collection from Hub
- [ ] Created requirements.yml
- [ ] Used FQCN in playbook
- [ ] Used collection in AAP project
- [ ] Bumped version and rebuilt
- [ ] Can explain software company use case

---

## Quick Reference — Collection Commands

| Command | Purpose |
|---------|---------|
| `ansible-galaxy collection init ns.name` | Create collection |
| `ansible-galaxy collection build path/` | Build tarball |
| `ansible-galaxy collection install ns.name` | Install from Galaxy/Hub |
| `ansible-galaxy collection install file.tar.gz` | Install from tarball |
| `ansible-galaxy collection install -r requirements.yml` | Install from requirements |

---

## Quick Reference — Collection Layout

```
namespace.collection/
├── galaxy.yml
├── README.md
├── meta/runtime.yml
├── roles/
├── plugins/{modules,filter_plugins,...}
├── playbooks/
└── docs/
```

---

*End of Lab 15*
