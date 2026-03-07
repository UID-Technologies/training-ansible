# Lab 17 — Manage Automation Execution Environments

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 4 hours (Theory: 30% / Hands-on: 70%)                                     |
| **Outcome**  | Understand execution environments (EE); build custom EEs with ansible-builder; add collections, Python packages, and system packages; test with ansible-navigator; publish to Private Automation Hub; assign EEs to organizations and job templates in AAP |
| **Platform** | Red Hat Ansible Automation Platform **2.4** — Automation Controller, Private Automation Hub |
| **Prerequisite** | Labs 02, 04, 05, 09 completed — AAP 2.4 installed, project with playbooks, Private Hub (optional). Podman or Docker for building. |

---

## AAP 2.4 UI Navigation Reference

| Menu | Location | Contains |
|------|----------|----------|
| **Administration** | Left panel | Execution Environments |
| **Execution Environments** | Administration → Execution Environments | List, add, edit EEs |
| **Organizations** | Access → Organizations | EE assignment per org |
| **Templates** | Resources → Templates | EE assignment per job template |
| **Projects** | Resources → Projects | Project-level EE (optional) |

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR WORKSTATION                                             │
│                         - ansible-builder                                           │
│                         - ansible-navigator                                         │
│                         - Podman / Docker                                           │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                        │ Build & Push
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    EXECUTION ENVIRONMENT BUILD FLOW                                  │
│                                                                                      │
│  execution-environment.yml  ──▶  ansible-builder build  ──▶  Container Image        │
│  - base image                    - collections              - ee-network:1.0        │
│  - collections                   - python packages           - ee-cloud:1.0          │
│  - python packages               - system packages                                   │
│  - system packages                                                                  │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ Push
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    PRIVATE AUTOMATION HUB / REGISTRY                                 │
│                    - ee-network:1.0                                                  │
│                    - ee-cloud:1.0                                                    │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ Pull
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    AAP CONTROLLER (aap1.lab.local)                                   │
│                    - Job templates use EE                                           │
│                    - Organizations have default EE                                  │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │ Run jobs
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    MANAGED NODES (web1, app1, db1)                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Podman or Docker | For building EE images |
| ansible-builder | `pip install ansible-builder` or from AAP installer |
| ansible-navigator | For local EE testing |
| Private Automation Hub | Optional — for publishing (Lab 09) |
| Red Hat account | For registry.redhat.io base images (if used) |

---

## Verify Environment Before Starting

```bash
ansible-builder --version
ansible-navigator --version
podman --version
# or: docker --version
```

---

## Part 1 — Theory: Execution Environments, Components, Building, Testing, Publishing, Managing (72 minutes)

### Section 16.1 — Understanding Execution Environments

#### What Are Execution Environments?

**Execution Environments (EE)** are **container images** that provide a consistent, isolated runtime for Ansible automation. Each job runs inside an EE container on an execution node.

| Component | Purpose |
|-----------|---------|
| **Base image** | OS layer (e.g., UBI, RHEL) |
| **Ansible Core** | Ansible engine |
| **Collections** | Modules, roles, plugins |
| **Python packages** | boto3, azure-identity, etc. |
| **System packages** | gcc, openssh, etc. |

#### Why Use Containers for Automation?

| Benefit | Description |
|---------|-------------|
| **Isolation** | No dependency conflicts between jobs |
| **Reproducibility** | Same image = same behavior everywhere |
| **Portability** | Run on any container host |
| **Scalability** | Horizontal scaling with execution nodes |
| **Security** | Minimal attack surface; no host pollution |

#### Differences from Legacy Virtual Environments

| Aspect | Virtualenv (legacy) | Execution Environment |
|--------|---------------------|------------------------|
| **Scope** | Python only | Full runtime (OS + Python + Ansible) |
| **Format** | Directory on disk | Container image |
| **Isolation** | Process-level | Container-level |
| **Distribution** | Copy directory | Push to registry, pull on demand |

#### Benefits of Execution Environments

| Benefit | Impact |
|---------|--------|
| **Consistency** | Dev, staging, prod use same EE |
| **Version pinning** | Pin ansible-core, collections |
| **Team specialization** | Network EE, cloud EE, security EE |
| **No "works on my machine"** | Identical runtime everywhere |

#### Base Images and Layering Concepts

- **Base image** — Minimal OS (e.g., `ubi9`, `ee-minimal-rhel9`)
- **Layers** — Each build step adds a layer
- **Caching** — Unchanged layers reused for faster rebuilds

---

### Section 16.2 — Execution Environment Components

#### Container Image Structure

```
Layer 4: Application (ansible, collections, playbooks)
Layer 3: Python packages (boto3, etc.)
Layer 2: System packages (from bindep.txt)
Layer 1: Ansible Core + Runner
Layer 0: Base image (UBI/RHEL)
```

#### Ansible Version and Collections

- **ansible-core** — Core engine; specify version for reproducibility
- **Collections** — From Galaxy, Hub, or requirements.yml
- **ansible-runner** — Used by AAP to run playbooks inside EE

#### Python Packages and Dependencies

- Cloud SDKs: boto3, azure-identity, google-cloud-*
- Utilities: netaddr, jmespath, passlib
- Add via `dependencies.python` in EE definition

#### System Packages (RPMs/APT)

- Compilers, libraries, CLI tools
- Add via `bindep.txt` (or `dependencies.system` list in v1)
- **Note:** ansible-builder uses **RPM-based** base images (dnf); Debian/Ubuntu base not supported

#### Understanding Base Images

| Base Image | Use Case |
|------------|----------|
| `registry.redhat.io/.../ee-minimal-rhel9` | Red Hat supported; requires subscription |
| `quay.io/ansible/ansible-runner` | Community; general purpose |
| `docker.io/redhat/ubi9` | UBI — free, RHEL-compatible |
| `quay.io/rockylinux/rockylinux:9` | Rocky Linux base |

---

### Section 16.3 — Building Execution Environments

#### Introduction to ansible-builder

- **ansible-builder** — CLI tool to build EE images
- Reads `execution-environment.yml` (or `-f` for custom path)
- Produces Containerfile/Dockerfile and build context
- Builds image with Podman or Docker

#### execution-environment.yml File Structure

**Version 3** (ansible-builder 3.x):

```yaml
---
version: 3
images:
  base_image:
    name: docker.io/redhat/ubi9:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python:
    - boto3
    - netaddr
  system: bindep.txt
```

**Version 1** (simpler; older ansible-builder):

```yaml
---
version: 1
dependencies:
  galaxy:
    collections:
      - name: ansible.posix
      - name: amazon.aws
  python:
    - boto3
  system:
    - gcc
```

#### Adding Ansible Collections to EE

- Create `requirements.yml` with collections
- Reference in EE: `galaxy: requirements.yml`
- Or inline in version 1: `galaxy.collections`

#### Adding Python Packages to EE

- List in `dependencies.python`
- Or use `requirements.txt` and reference path

#### Adding System Packages to EE

- Create `bindep.txt` with package names (one per line)
- Reference: `system: bindep.txt`
- Format: `package-name [platform]` (e.g., `gcc` or `openssh-clients`)

#### Building Your First Execution Environment

```bash
ansible-builder build -t my-ee:1.0 -f execution-environment.yml
```

---

### Section 16.4 — Testing Execution Environments

#### Using ansible-navigator for Local Testing

- **ansible-navigator** — CLI to run playbooks in EE locally
- `ansible-navigator run playbook.yml -ee my-ee:1.0 -i inventory`
- Verifies EE has required collections and dependencies

#### Verifying Installed Collections

```bash
ansible-navigator exec -ee my-ee:1.0 -- ansible-galaxy collection list
```

#### Testing Playbooks in Execution Environments

```bash
ansible-navigator run playbook.yml -ee my-ee:1.0 -i inventory --mode stdout
```

#### Troubleshooting Build Issues

| Issue | Fix |
|-------|-----|
| Base image pull fails | Check registry auth; use alternative base |
| Collection not found | Verify Galaxy/Hub URL; check requirements.yml |
| Python package conflict | Pin versions; use exclude |
| System package missing | Add to bindep.txt; ensure base is RPM-based |

#### Dependency Conflict Resolution

- Pin versions: `ansible-core==2.14.4`, `boto3==1.28.0`
- Use `exclude` to remove conflicting packages
- Test in minimal EE first

---

### Section 16.5 — Publishing and Using Execution Environments

#### Container Registry Concepts

- **Registry** — Stores container images (e.g., quay.io, registry.redhat.io)
- **Repository** — Namespace/name (e.g., `myorg/ee-network`)
- **Tag** — Version (e.g., `1.0`, `latest`)

#### Pushing to Private Automation Hub

- Hub has built-in container registry
- Push: `podman push localhost/my-ee:1.0 hub.lab.local/namespace/my-ee:1.0`
- Or use Hub's "Execution Environments" → "Add" → upload/import

#### Image Tagging Best Practices

| Tag | Use |
|-----|-----|
| `1.0.0` | Semantic version |
| `1.0` | Minor version |
| `latest` | Avoid in production |
| `main` | Git branch alignment |

#### Version Management Strategies

| Strategy | Example | Use Case |
|----------|---------|----------|
| Semantic | `1.0.0`, `1.1.0` | Production |
| Date | `20240307` | Daily builds |
| Git SHA | `abc1234` | Traceability |

#### Assigning EE to Organizations and Templates

- **Organization** — Default EE for all templates in org
- **Job template** — Override EE per template
- **Project** — Optional project-level EE (AAP 2.5+)

---

### Section 16.6 — Managing Execution Environments in AAP

#### Global Execution Environment Settings

- **Administration → Execution Environments** — Add, edit, delete
- **Settings → Jobs** — Default EE for jobs (if no org/template override)

#### Project-Specific Execution Environments

- Project → **Execution Environment** — Override for playbooks in project
- Use when project needs specific collections

#### Template-Specific Execution Environments

- Job template → **Execution Environment** — Select EE for this template
- Most common override — each template can use different EE

#### Execution Environment Pull Policies

| Policy | Behavior |
|--------|----------|
| **Always** | Pull every time (get latest tag) |
| **Missing** | Pull only if not present |
| **Never** | Use cached only; fail if missing |

#### Troubleshooting Execution Environment Issues

| Issue | Fix |
|-------|-----|
| Job fails "image not found" | Verify image name; check pull policy; registry auth |
| "Module not found" | EE missing collection; rebuild with collection |
| "No module named X" | EE missing Python package; add to EE |
| Pull fails | Check registry credential; network; image exists |

---

## Part 2 — Hands-On: Create execution-environment.yml (30 minutes)

### Step 1 — Create Minimal EE Definition

**Purpose:** Create a basic execution-environment.yml.

**Detailed steps:**

1. Create working directory:

```bash
mkdir -p ~/ee-lab
cd ~/ee-lab
```

2. Create `execution-environment.yml` (version 3):

```yaml
---
version: 3
images:
  base_image:
    name: docker.io/redhat/ubi9:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python:
    - netaddr
```

3. Create `requirements.yml`:

```yaml
---
collections:
  - name: ansible.posix
    version: ">=1.0.0"
  - name: ansible.utils
    version: ">=2.0.0"
```

4. Save both files.

**Verify:** Files exist; YAML is valid.

---

### Step 2 — Add System Packages (bindep.txt)

**Purpose:** Add system-level dependencies.

**Detailed steps:**

1. Create `bindep.txt`:

```
gcc
openssh-clients
```

2. Update `execution-environment.yml` — add under dependencies:

```yaml
  system: bindep.txt
```

3. Save.

**Verify:** bindep.txt exists; EE file references it.

---

### Step 3 — Create Network-Specific EE Definition

**Purpose:** EE for network automation with network collections.

**Detailed steps:**

1. Create `execution-environment-network.yml`:

```yaml
---
version: 3
images:
  base_image:
    name: docker.io/redhat/ubi9:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements-network.yml
  python:
    - netaddr
    - paramiko
  system: bindep.txt
```

2. Create `requirements-network.yml`:

```yaml
---
collections:
  - name: ansible.posix
    version: ">=1.0.0"
  - name: ansible.utils
    version: ">=2.0.0"
  - name: ansible.netcommon
    version: ">=4.0.0"
```

**Verify:** Network EE definition is complete.

---

## Part 3 — Hands-On: Build Execution Environments (40 minutes)

### Step 4 — Build Your First EE

**Purpose:** Build EE image with ansible-builder.

**Detailed steps:**

1. Ensure Podman (or Docker) is running
2. Build:

```bash
cd ~/ee-lab
ansible-builder build -t ee-lab:1.0 -f execution-environment.yml
```

3. Verify image exists:

```bash
podman images | grep ee-lab
# or: docker images | grep ee-lab
```

**Verify:** Image `ee-lab:1.0` is built successfully.

---

### Step 5 — Build Network EE

**Purpose:** Build network-specific EE.

**Detailed steps:**

1. Build:

```bash
ansible-builder build -t ee-network:1.0 -f execution-environment-network.yml
```

2. Verify:

```bash
podman images | grep ee-network
```

**Verify:** Image `ee-network:1.0` exists.

---

### Step 6 — Create Cloud EE with Multiple Cloud SDKs

**Purpose:** EE with AWS and Azure collections and Python packages.

**Detailed steps:**

1. Create `requirements-cloud.yml`:

```yaml
---
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: azure.azcollection
    version: ">=2.0.0"
  - name: ansible.posix
    version: ">=1.0.0"
```

2. Create `execution-environment-cloud.yml`:

```yaml
---
version: 3
images:
  base_image:
    name: docker.io/redhat/ubi9:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements-cloud.yml
  python:
    - boto3
    - botocore
    - azure-identity
    - azure-mgmt-compute
```

3. Build:

```bash
ansible-builder build -t ee-cloud:1.0 -f execution-environment-cloud.yml
```

**Lab note:** Cloud EE build may take longer due to large Python packages. If build fails (e.g., network), use minimal EE for remaining steps.

**Verify:** Image `ee-cloud:1.0` builds (or document any failures for troubleshooting).

---

## Part 4 — Hands-On: Test Execution Environments (35 minutes)

### Step 7 — Use ansible-navigator to Test EE

**Purpose:** Run playbook inside EE locally.

**Detailed steps:**

1. Create simple playbook `test_ee.yml`:

```yaml
---
- name: Test EE
  hosts: localhost
  connection: local
  gather_facts: true
  tasks:
    - name: Check ansible version
      ansible.builtin.debug:
        msg: "Ansible {{ ansible_version.full }}"
    - name: Use ansible.utils (from EE)
      ansible.builtin.debug:
        msg: "ansible.utils available - ipaddr filter works"
```

2. Create `inventory_ee.yml`:

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
```

3. Run with ansible-navigator:

```bash
ansible-navigator run test_ee.yml -i inventory_ee.yml -ee ee-lab:1.0 --mode stdout
```

**Verify:** Playbook runs inside EE; collections are available.

---

### Step 8 — Verify Installed Collections in EE

**Purpose:** List collections inside EE.

**Detailed steps:**

1. Run:

```bash
ansible-navigator exec -ee ee-lab:1.0 -- ansible-galaxy collection list
```

2. Confirm `ansible.posix`, `ansible.utils` appear

**Verify:** Collections are installed in EE.

---

### Step 9 — Troubleshoot Missing Collection

**Purpose:** Simulate and fix missing collection.

**Detailed steps:**

1. Create playbook that uses a collection NOT in EE (e.g., `community.docker` if not in requirements)
2. Run — expect failure
3. Add collection to requirements.yml; rebuild EE
4. Run again — success

**Verify:** You understand how to add collections and rebuild.

---

## Part 5 — Hands-On: Publish and Use in AAP (40 minutes)

### Step 10 — Tag and Push to Registry

**Purpose:** Push EE to container registry (Private Hub or local registry).

**Detailed steps:**

1. Tag for registry:

```bash
podman tag ee-lab:1.0 hub.lab.local/ee-lab:1.0
# Or for default registry: quay.io/myuser/ee-lab:1.0
```

2. Log in to registry (if required):

```bash
podman login hub.lab.local
```

3. Push:

```bash
podman push hub.lab.local/ee-lab:1.0
```

**Lab note:** If Private Hub is not available, use local registry or skip push; add EE manually in AAP using local image.

**Verify:** Image is in registry.

---

### Step 11 — Add EE to AAP

**Purpose:** Register EE in Automation Controller.

**Detailed steps:**

1. Log in to AAP: `https://aap1.lab.local`
2. Navigate to **Administration → Execution Environments**
3. Click **Add**
4. Fill in:
   - **Name:** `ee-lab-1.0`
   - **Image:** `hub.lab.local/ee-lab:1.0` (or `localhost/ee-lab:1.0` for local)
   - **Pull:** Always or Missing
   - **Registry credential:** Add if registry requires auth
5. **Save**

**Verify:** EE appears in list; status shows as available (after pull).

---

### Step 12 — Assign EE to Job Template

**Purpose:** Use custom EE for a job template.

**Detailed steps:**

1. Navigate to **Resources → Templates**
2. Open a job template (e.g., `Ping-Lab` or create one)
3. Find **Execution Environment**
4. Select `ee-lab-1.0` (or your custom EE)
5. **Save**
6. **Launch** the job
7. Verify job runs successfully with custom EE

**Verify:** Job uses custom EE; completes successfully.

---

### Step 13 — Assign EE to Organization

**Purpose:** Set default EE for an organization.

**Detailed steps:**

1. Navigate to **Access → Organizations**
2. Open **Default** (or your org)
3. Find **Execution Environment** (or **Default Execution Environment**)
4. Select `ee-lab-1.0`
5. **Save**
6. New templates in this org default to this EE

**Verify:** Org has default EE set.

---

## Part 6 — Real-Time Use Case Discussion (25 minutes)

### Case Study: Financial Services — Specialized Execution Environments

**Background:**

A financial services company runs automation for network devices (routers, firewalls), cloud resources (AWS, Azure), and security compliance. Each domain has different Ansible collections and Python dependencies. A single EE caused conflicts (e.g., network collections vs. cloud SDK versions).

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| Mixed dependencies | Network + cloud in one EE = version conflicts |
| Large image size | Single EE with all collections = slow pull, bloat |
| Team ownership | Network team vs. cloud team needed different content |
| Compliance | Audit required knowing exactly which EE ran which job |

**Solution: Specialized Execution Environments**

#### 1. Network Automation EE

| Component | Content |
|-----------|---------|
| **Collections** | ansible.netcommon, cisco.ios, junipernetworks.junos, arista.eos |
| **Python** | netaddr, paramiko, ncclient |
| **System** | openssh-clients, libffi |
| **Use** | Job templates for network config, backup, compliance |

#### 2. Cloud Automation EE

| Component | Content |
|-----------|---------|
| **Collections** | amazon.aws, azure.azcollection, google.cloud |
| **Python** | boto3, azure-identity, google-auth |
| **System** | (minimal) |
| **Use** | Job templates for EC2, Azure VM, GCP provisioning |

#### 3. Security Compliance EE

| Component | Content |
|-----------|---------|
| **Collections** | ansible.posix, redhat.rhel_system_roles |
| **Python** | passlib, jmespath |
| **Use** | CIS benchmark, hardening playbooks |

#### 4. Version Management

- Tag EEs with semantic version: `ee-network:1.2.0`
- Pin job templates to specific EE version
- New EE version → test in dev → promote to prod

**Results:**

| Metric | Before | After |
|--------|--------|-------|
| Dependency conflicts | Frequent | Zero |
| Image size | 2.5 GB (monolithic) | 800 MB avg (specialized) |
| Job failures (missing module) | 15% | < 1% |
| Team autonomy | Low | High (own EE) |

**Key takeaway:** Specialized EEs per domain (network, cloud, security) eliminate conflicts, reduce image size, and give teams ownership of their automation runtime.

---

## Troubleshooting

### ansible-builder Build Fails

| Cause | Fix |
|-------|-----|
| Base image pull fails | Check registry auth; try alternative base (ubi9, rockylinux) |
| Non-RPM base | Use RPM-based image (dnf); Debian/Ubuntu not supported |
| Collection not found | Verify requirements.yml; check Galaxy/Hub URL |
| Python conflict | Pin versions; use exclude |

### EE Not Pulling in AAP

| Cause | Fix |
|-------|-----|
| Wrong image name | Use full path: `registry/namespace/image:tag` |
| No credential | Add registry credential for private registry |
| Network | Execution node must reach registry |
| Pull policy | Use "Always" to force pull |

### Job Fails with "Module not found"

| Cause | Fix |
|-------|-----|
| EE missing collection | Add to requirements.yml; rebuild EE |
| Wrong EE assigned | Verify template uses correct EE |
| FQCN typo | Use `namespace.collection.module` |

### ansible-navigator Cannot Find EE

| Cause | Fix |
|-------|-----|
| Image not built | Run ansible-builder build |
| Wrong tag | Use exact tag: `-ee ee-lab:1.0` |
| Podman/Docker | Ensure container runtime is running |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|-----------------|
| Execution Environment | Container image for Ansible runtime |
| ansible-builder | Build EE from execution-environment.yml |
| execution-environment.yml | version 3; images, dependencies |
| requirements.yml | Collections for EE |
| bindep.txt | System packages |
| ansible-navigator | Test EE locally |
| Registry | Push/pull container images |
| AAP EE assignment | Org, template, project level |
| Pull policy | Always, Missing, Never |
| Specialized EEs | Network, cloud, security per domain |

---

## Lab Completion Checklist

- [ ] Created execution-environment.yml (version 3)
- [ ] Created requirements.yml for collections
- [ ] Created bindep.txt for system packages
- [ ] Built EE with ansible-builder
- [ ] Built network-specific EE
- [ ] Built cloud EE (or attempted)
- [ ] Tested EE with ansible-navigator
- [ ] Verified collections in EE
- [ ] Pushed EE to registry (if available)
- [ ] Added EE to AAP
- [ ] Assigned EE to job template
- [ ] Assigned EE to organization
- [ ] Can explain financial services use case

---

## Quick Reference — execution-environment.yml (v3)

```yaml
---
version: 3
images:
  base_image:
    name: docker.io/redhat/ubi9:latest
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python:
    - boto3
  system: bindep.txt
```

---

## Quick Reference — ansible-builder Commands

| Command | Purpose |
|---------|---------|
| `ansible-builder build -t name:tag -f ee.yml` | Build EE |
| `ansible-builder create` | Create build context only |
| `ansible-builder prune` | Prune build cache |

---

## Quick Reference — ansible-navigator

| Command | Purpose |
|---------|---------|
| `ansible-navigator run playbook.yml -ee image:tag` | Run playbook in EE |
| `ansible-navigator exec -ee image:tag -- cmd` | Run command in EE |
| `ansible-navigator --help` | List options |

---

*End of Lab 17*
