# Lab 01 — Install Ansible Automation Platform (Standalone) + Initial Setup + UI Tour

| Field        | Value                                              |
|--------------|----------------------------------------------------|
| **Audience** | Beginner to Intermediate (Ops / DevOps / Platform) |
| **Duration** | 3–4 hours                                          |
| **Outcome**  | AAP installed on one host, web UI accessible, inventory + credentials created, first job executed |
| **Platform** | Red Hat Ansible Automation Platform 2.4 / 2.5      |

---

## Section 1 — Lab Architecture (What You Are Building)

### 1.1 What "Standalone" Means

Everything runs on **one server** (ideal for labs and proof-of-concept):

| Component                | Purpose                          |
|--------------------------|----------------------------------|
| Automation Controller    | Web UI, REST API, scheduler, RBAC |
| Execution Environment    | Runs jobs locally on the same host |
| PostgreSQL               | Local database                   |
| Nginx                    | HTTPS reverse-proxy for the UI   |

> **Note:** Automation Hub and Event-Driven Ansible (EDA) are optional and not covered in this lab.

### 1.2 Mapping to Data Center / Host / Network Concepts

| Concept       | What It Maps To in This Lab                                        |
|---------------|--------------------------------------------------------------------|
| Data Center   | Your training network, cloud VPC/VNet, DNS zone, and firewall rules |
| Host          | The AAP server VM (single node) + managed target nodes             |
| Network       | Ports, IP addressing, DNS resolution, and routing between hosts    |

---

## Section 2 — Prerequisites

### 2.1 Infrastructure Requirements

**AAP Node (Single VM):**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| vCPU     | 4       | 8           |
| RAM      | 16 GB   | 32 GB       |
| Disk     | 80 GB   | 100 GB      |
| OS       | RHEL 8 or RHEL 9 (required — per Red Hat support matrix) | |

**Managed Nodes (2 Linux VMs recommended for testing):**

- Ubuntu, RHEL, CentOS Stream, AlmaLinux, or Rocky Linux
- SSH reachable from the AAP node

### 2.2 Network Requirements

| Direction                        | Port     | Protocol | Purpose                |
|----------------------------------|----------|----------|------------------------|
| Laptop/Workstation --> AAP Server | TCP 443  | HTTPS    | Web UI and API access  |
| AAP Server --> Managed Nodes     | TCP 22   | SSH      | Ansible job execution  |
| AAP Server (local only)         | TCP 5432 | PostgreSQL | Database (bound to localhost) |

### 2.3 Required Accounts and Files

- **Red Hat account** with an active Ansible Automation Platform subscription
- **AAP installer bundle** (`.tar.gz`) downloaded from the [Red Hat Customer Portal](https://access.redhat.com/downloads/content/480)
- **Subscription manifest** (`.zip`) exported from [Red Hat Subscription Management](https://access.redhat.com/management/subscription_allocations) — needed after install to activate the Controller

### 2.4 Prerequisite Checkpoint

Before proceeding, verify all of the following:

```bash
# From your laptop — can you SSH into the AAP VM?
ssh <user>@<AAP_SERVER_IP>

# From the AAP VM — can you reach managed nodes?
ping node1.lab.local
ssh user@node1.lab.local
```

If any of these fail, fix networking/firewall before continuing.

---

## Section 3 — Prepare the AAP Server VM

### 3.1 Set Hostname

```bash
sudo hostnamectl set-hostname aap1.lab.local
```

If DNS is not configured in your lab, add entries to `/etc/hosts`:

```bash
sudo vi /etc/hosts
```

Add lines like:

```text
192.168.1.10   aap1.lab.local aap1
192.168.1.20   node1.lab.local node1
192.168.1.21   node2.lab.local node2
```

> **Why this matters:** AAP generates URLs, TLS certificates, and callback addresses that depend on a stable, resolvable hostname. A wrong or unresolvable hostname is the #1 cause of install failures.

### 3.2 Update the OS and Install Basic Tools

```bash
sudo dnf update -y
sudo dnf install -y vim curl wget tar unzip firewalld git
```

### 3.3 Configure Time Synchronization

TLS certificates and authentication tokens depend on accurate time.

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
timedatectl
```

Verify the clock is synchronized:

```bash
chronyc tracking
```

### 3.4 Verify SELinux Mode

AAP is designed to work with SELinux in **enforcing** mode. Do **not** disable SELinux.

```bash
getenforce
```

Expected output: `Enforcing`

If it shows `Disabled` or `Permissive`, set it to enforcing:

```bash
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

> **Important:** If SELinux was `Disabled`, a reboot is required for the change to take effect.

---

## Section 4 — Register System and Enable Repositories

### 4.1 Register with Red Hat Subscription Manager

```bash
sudo subscription-manager register --username <RED_HAT_USERNAME> --password <RED_HAT_PASSWORD>
sudo subscription-manager attach --auto
```

Verify registration:

```bash
sudo subscription-manager status
```

### 4.2 Enable Required Repositories

For **RHEL 9** with **AAP 2.4**:

```bash
sudo subscription-manager repos \
  --enable ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms \
  --enable rhel-9-for-x86_64-baseos-rpms \
  --enable rhel-9-for-x86_64-appstream-rpms
```

For **RHEL 8** with **AAP 2.4**, replace `rhel-9` with `rhel-8` in each repo name.

> **Tip:** If using AAP 2.5, replace `2.4` with `2.5` in the repo name.

### 4.3 Verify Repositories

```bash
sudo dnf repolist
```

You should see the three repos listed as enabled.

---

## Section 5 — Download and Extract the AAP Installer

### 5.1 Download the Installer Bundle

Download the **Setup Bundle** from the [Red Hat Customer Portal — AAP Downloads](https://access.redhat.com/downloads/content/480).

Choose the bundle matching your RHEL version (e.g., `ansible-automation-platform-setup-bundle-2.4-*-x86_64.tar.gz`).

Transfer it to the AAP server using `scp` or direct download:

```bash
scp ansible-automation-platform-setup-bundle-*.tar.gz user@aap1.lab.local:/tmp/
```

or

```bash
scp -i /path/to/your-key.pem ansible-automation-platform-setup-bundle-*.tar.gz ec2-user@<EC2_HOST>:/tmp/
```

or

```bash
cd /tmp
curl -O "https://your-internal-server/bundle.tar.gz"
# or
wget "https://your-internal-server/bundle.tar.gz"
```


### 5.2 Extract the Installer

```bash
cd /tmp
tar -xvf ansible-automation-platform-setup-bundle-*.tar.gz
cd ansible-automation-platform-setup-bundle-*/
```

### 5.3 Verify Installer Contents

```bash
ls -la
```

You should see at least:

| File/Directory | Purpose                          |
|----------------|----------------------------------|
| `inventory`    | Defines which hosts get which roles |
| `setup.sh`     | Main installer script            |
| `collections/` | Bundled Ansible collections      |
| `bundle/`      | Bundled RPM packages (offline install) |

---

## Section 6 — Configure the Installer Inventory File

This is the **most important step**. The inventory file tells the installer what to install and where.

### 6.1 Edit the Inventory File

```bash
vi inventory
```

Replace the contents with the following standalone configuration:

```ini
[automationcontroller]
aap1.lab.local ansible_connection=local

[database]
aap1.lab.local ansible_connection=local

[all:vars]
admin_password='StrongAdminPassword@123'

pg_host='aap1.lab.local'
pg_port=5432
pg_database='awx'
pg_username='awx'
pg_password='StrongDBPassword@123'
pg_sslmode='prefer'

# --- Container Registry for Execution Environments ---
# Required for pulling default EE images during install.
# Use your Red Hat registry credentials (same as Red Hat account).
registry_url='registry.redhat.io'
registry_username='<RED_HAT_USERNAME>'
registry_password='<RED_HAT_PASSWORD>'
```

> **Security note:** In production, use Ansible Vault to encrypt passwords in the inventory file. For this lab, plaintext is acceptable.

### 6.2 Understand the Inventory Sections

| Section                  | Meaning                                          |
|--------------------------|--------------------------------------------------|
| `[automationcontroller]` | Host(s) where the Controller (UI/API) is installed |
| `[database]`             | Host where PostgreSQL runs                       |
| `[all:vars]`             | Variables applied to all hosts                   |
| `ansible_connection=local` | Install locally (no SSH to self needed)        |

### 6.3 Verify Hostname Resolution

```bash
getent hosts aap1.lab.local
```

This must return the correct IP. If not, fix `/etc/hosts` or DNS before continuing.

---

## Section 7 — Run the Installer

### 7.1 Execute the Setup Script

From inside the extracted installer directory:

```bash
sudo ./setup.sh
```

The installer will:

1. Install and configure PostgreSQL
2. Install the Automation Controller application
3. Configure Nginx as an HTTPS reverse-proxy
4. Pull Execution Environment container images
5. Start all services

> **Expected duration:** 15–30 minutes depending on hardware and network speed.

### 7.2 Monitor the Installation

Watch for `PLAY RECAP` at the end. A successful install shows `failed=0` for all hosts:

```text
PLAY RECAP *********************************************************************
aap1.lab.local   : ok=150  changed=75  unreachable=0  failed=0  ...
```

If there are failures, scroll up to find the first red error message.

### 7.3 Verify Services Are Running

```bash
sudo systemctl status automation-controller
sudo systemctl status postgresql
sudo systemctl status nginx
```

All three should show `active (running)`.

---

## Section 8 — Configure Firewall and Verify Network Access

### 8.1 Enable Firewall and Allow HTTPS

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### 8.2 Verify Ports Are Listening

```bash
sudo ss -tulpn | grep -E ':(443|5432)'
```

You should see Nginx listening on port 443.

### 8.3 Test Browser Access

From your laptop browser, navigate to:

```text
https://aap1.lab.local
```

or

```text
https://<AAP_SERVER_IP>
```

> **Note:** You will see a certificate warning because the installer generates a self-signed TLS certificate. Accept the warning to proceed (this is expected in a lab).

If the page does not load, see **Section 14 — Troubleshooting**.

---

## Section 9 — First Login and Subscription Activation

### 9.1 Log In to the Web UI

- **Username:** `admin`
- **Password:** the `admin_password` you set in the inventory file (e.g., `StrongAdminPassword@123`)

### 9.2 Upload the Subscription Manifest

On first login, AAP will prompt you to activate your subscription.

1. If you have not already, go to [Red Hat Subscription Allocations](https://access.redhat.com/management/subscription_allocations)
2. Create a new allocation (type: **Satellite**), add your AAP subscription entitlements, and **Export Manifest** (downloads a `.zip` file)
3. In the AAP UI, click **Browse** and upload the manifest `.zip` file
4. Click **Next**, accept the license agreement, and click **Submit**

> **Why this is required:** Without a valid subscription manifest, the Controller operates in a limited trial mode and some features are restricted.

### 9.3 Verify Activation

After uploading, the Dashboard should load without subscription warnings. You should see the main navigation with Dashboard, Projects, Inventories, etc.

---

## Section 10 — UI Tour (Controller Web Interface)

Walk through each section of the UI with participants:

| UI Section         | What It Does                                                   |
|--------------------|----------------------------------------------------------------|
| **Dashboard**      | Overview of recent job activity, host counts, and job status   |
| **Projects**       | Source of playbooks — linked to Git repos or local directories |
| **Inventories**    | Lists of managed hosts, organized into groups                  |
| **Credentials**    | Stored secrets — SSH keys, passwords, vault tokens, cloud creds |
| **Templates**      | Reusable job definitions — combines playbook + inventory + credential |
| **Jobs**           | Audit log — who ran what, when, with full output logs          |
| **Users / Teams**  | RBAC — role-based access control for enterprise workflows      |
| **Settings**       | System configuration, authentication, logging, license info    |

---

## Section 11 — Add Managed Nodes (Inventory and Credentials)

### 11.1 Create an Inventory

1. Navigate to **Resources --> Inventories**
2. Click **Add --> Add inventory**
3. Fill in:
   - **Name:** `Lab-Inventory`
   - **Organization:** `Default`
4. Click **Save**

### 11.2 Add Hosts to the Inventory

1. Inside `Lab-Inventory`, click the **Hosts** tab
2. Click **Add**
3. Add each managed node:
   - **Name:** `node1.lab.local` (or use the IP address)
4. Repeat for `node2.lab.local`

> **Conceptual mapping:**
> - **Inventory** = your data center's target list
> - **Hosts** = individual servers or devices in your environment
> - **Groups** = logical groupings (network zones, app tiers, locations)

### 11.3 Create a Machine Credential (SSH)

1. Navigate to **Resources --> Credentials**
2. Click **Add**
3. Fill in:
   - **Name:** `SSH-Key-Lab`
   - **Organization:** `Default`
   - **Credential Type:** `Machine`
   - **Username:** `ubuntu`, `ec2-user`, or `root` (match your managed nodes)
   - **SSH Private Key:** paste the contents of your private key, **or** set a **Password** if using password-based authentication
4. Click **Save**

> **Key concept:** Credentials are how AAP securely crosses the network boundary to reach managed hosts. They are stored encrypted and never exposed in job logs.

---

## Section 12 — Create a Project (Get Playbooks into AAP)

### Option A — Manual Project Directory (Simplest for Lab)

Create a playbook directory on the AAP server:

```bash
sudo mkdir -p /var/lib/awx/projects/lab-playbooks
```

Create a simple ping playbook:

```bash
sudo tee /var/lib/awx/projects/lab-playbooks/ping.yml > /dev/null << 'EOF'
---
- name: Ping all hosts
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping
      ansible.builtin.ping:
EOF
```

Set correct ownership so the `awx` service user can read the files:

```bash
sudo chown -R awx:awx /var/lib/awx/projects/lab-playbooks
```

In the UI:

1. Navigate to **Resources --> Projects**
2. Click **Add**
3. Fill in:
   - **Name:** `Lab-Project`
   - **Organization:** `Default`
   - **Source Control Type:** `Manual`
   - **Playbook Directory:** select `lab-playbooks` from the dropdown
4. Click **Save**

### Option B — Git Repository (Recommended for Real Use)

If you have a Git repository (GitHub, GitLab, etc.):

1. Navigate to **Resources --> Projects**
2. Click **Add**
3. Fill in:
   - **Name:** `Lab-Project`
   - **Organization:** `Default`
   - **Source Control Type:** `Git`
   - **Source Control URL:** `https://github.com/your-org/your-repo.git`
   - **Source Control Credential:** add one if the repo is private
4. Click **Save**
5. Click the sync icon to pull the repository

> **Tip for trainers:** Option A is faster for labs. Option B teaches real-world SCM integration.

---

## Section 13 — Create a Job Template and Run Your First Job

### 13.1 Create the Job Template

1. Navigate to **Resources --> Templates**
2. Click **Add --> Add job template**
3. Fill in:
   - **Name:** `Ping-Lab`
   - **Inventory:** `Lab-Inventory`
   - **Project:** `Lab-Project`
   - **Playbook:** `ping.yml`
   - **Credential:** `SSH-Key-Lab`
4. Click **Save**

### 13.2 Launch the Job

1. Click **Launch** (rocket icon)
2. Watch the job output in real time

### 13.3 Verify Success

- The job status should show **Successful**
- Each host should return `pong` in the output:

```text
node1.lab.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If the job fails, see **Section 14 — Troubleshooting**.

---

## Section 14 — Troubleshooting (Common Lab Failures)

### Problem: Web UI Does Not Load

```bash
# Check if Nginx is running
sudo systemctl status nginx

# Check if port 443 is listening
sudo ss -tulpn | grep 443

# Check firewall rules
sudo firewall-cmd --list-all

# Check Controller service
sudo systemctl status automation-controller

# View Controller logs
sudo journalctl -u automation-controller --no-pager -n 50
```

### Problem: Jobs Fail with "UNREACHABLE"

```bash
# Test SSH connectivity manually from the AAP server
ssh -i /path/to/key user@node1.lab.local

# Verify the key has correct permissions
ls -la /path/to/key
# Should show: -rw------- (chmod 600)

# Check if port 22 is open on the managed node
ss -tulpn | grep 22
```

Common causes:
- Wrong username in the credential
- SSH key permissions too open (must be `600`)
- Port 22 blocked by firewall on the managed node
- Host key verification failed (first-time SSH)

### Problem: Hostname Not Resolving

```bash
getent hosts node1.lab.local
getent hosts aap1.lab.local
```

Fix by adding entries to `/etc/hosts` on the AAP server, or configure DNS.

### Problem: Installer Fails

```bash
# Re-run the installer with verbose output
sudo ./setup.sh -v

# Check the installer log
less /var/log/tower/setup-*.log
```

Common causes:
- Insufficient disk space (`df -h`)
- Hostname does not resolve (`getent hosts aap1.lab.local`)
- Missing or incorrect registry credentials in inventory
- SELinux set to disabled (must be enforcing or permissive)

---

## Section 15 — Learning Exercise: Data Center / Host / Network Mapping

Have participants fill in and discuss:

### A) Data Center View

| Question | Answer |
|----------|--------|
| What is your "data center" in this lab? | (VPC / VNet / Lab LAN / home network) |
| What provides name resolution? | (DNS server / `/etc/hosts`) |
| What enforces inbound traffic rules? | (Security group / firewall / iptables) |

### B) Host View

| Question | Answer |
|----------|--------|
| Which host is the control plane? | AAP server (`aap1.lab.local`) |
| Which hosts are targets? | Managed nodes (`node1`, `node2`) |
| What user does AAP connect as? | (The user configured in the Machine credential) |

### C) Network View

| Question | Answer |
|----------|--------|
| What ports are required? | 443 (UI), 22 (SSH to nodes) |
| What breaks if DNS is wrong? | UI URL fails, TLS errors, callbacks break |
| What breaks if firewall blocks 443? | Cannot access the web UI |
| What breaks if port 22 is blocked? | Jobs fail with UNREACHABLE |

---

## Section 16 — Lab Completion Checklist

Participants must demonstrate the following before the lab is considered complete:

- [ ] AAP is installed and running on a single node
- [ ] Web UI is accessible via HTTPS in a browser
- [ ] Subscription manifest has been uploaded and accepted
- [ ] An inventory has been created with at least 2 managed hosts
- [ ] A machine credential (SSH) has been created
- [ ] A project has been added with at least one playbook
- [ ] A job template has been created and executed successfully (status: Successful)
- [ ] Participant can explain:
  - The data center boundary in this lab
  - The role of the controller host vs. managed nodes
  - Which network ports are required and why

---

## Quick Reference — Key Commands

| Task | Command |
|------|---------|
| Check AAP service | `sudo systemctl status automation-controller` |
| Check PostgreSQL | `sudo systemctl status postgresql` |
| Check Nginx | `sudo systemctl status nginx` |
| View AAP logs | `sudo journalctl -u automation-controller -f` |
| Check listening ports | `sudo ss -tulpn \| grep -E ':(443\|5432)'` |
| Restart AAP | `sudo systemctl restart automation-controller` |
| Re-run installer | `sudo ./setup.sh` (from installer directory) |
| Check hostname resolution | `getent hosts aap1.lab.local` |
| Check firewall | `sudo firewall-cmd --list-all` |
| Check SELinux | `getenforce` |

---

*End of Lab 01*
