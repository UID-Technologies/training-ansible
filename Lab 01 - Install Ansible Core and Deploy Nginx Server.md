# Lab 01 — Install Ansible Core and Deploy an Nginx Web Server

| Field        | Value                                                        |
|--------------|--------------------------------------------------------------|
| **Audience** | Beginner (System Admins / DevOps Engineers / IT Operations)  |
| **Duration** | 1.5–2 hours                                                  |
| **Outcome**  | Ansible Core installed on a control node, SSH connectivity established to a managed node, Nginx installed via playbook, custom web page deployed and verified |

---

## Lab Topology

```
┌─────────────────────────┐         SSH (port 22)        ┌─────────────────────────┐
│    CONTROL NODE         │ ──────────────────────────▶   │    MANAGED NODE          │
│                         │                               │                         │
│  OS: RHEL 8/9           │                               │  OS: RHEL / Ubuntu /    │
│  Role: Run Ansible      │                               │       AlmaLinux / Rocky  │
│  Software: Ansible Core │                               │  Role: Web Server        │
│  IP: 192.168.1.10       │                               │  Software: Nginx         │
│                         │                               │  IP: 192.168.1.20        │
└─────────────────────────┘                               └─────────────────────────┘
        ▲                                                          ▲
        │              Your Laptop / Browser                       │
        │              http://192.168.1.20                         │
        └──────────────────────────────────────────────────────────┘
```

> **Note:** Replace the IP addresses above with your actual lab IPs throughout this document.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Control Node | 1 VM running **RHEL 8 or RHEL 9**, with internet access |
| Managed Node | 1 VM running any supported Linux (RHEL, Ubuntu, AlmaLinux, Rocky) |
| Network | Both VMs can reach each other on the network |
| Access | You have `root` or `sudo` privileges on both VMs |
| Ports | SSH (TCP 22) open between control and managed node; HTTP (TCP 80) open on managed node |

---

## Step 1 — Verify the Lab Environment

Before installing anything, confirm both machines are reachable and ready.

### 1.1 Log In to the Control Node

SSH into your control node from your laptop:

```bash
ssh user@192.168.1.10
```

### 1.2 Verify Operating System

```bash
cat /etc/redhat-release
```

Expected output (example):

```
Red Hat Enterprise Linux release 9.3 (Plow)
```

### 1.3 Verify Network Connectivity to the Managed Node

```bash
ping -c 3 192.168.1.20
```

You should get replies. If not, fix networking before continuing.

---

## Step 2 — Install Ansible Core on the Control Node

Ansible only needs to be installed on the **control node**. The managed node does not need Ansible — it only needs Python and SSH.

### 2.1 Update the System

```bash
sudo dnf update -y
```

### 2.2 Install Ansible Core

**Option A — From the default RHEL AppStream repository (simplest):**

```bash
sudo dnf install -y ansible-core
```

**Option B — Using `pip` (if you need the latest version):**

```bash
sudo dnf install -y python3 python3-pip
pip3 install --user ansible-core
```

> **Recommendation for this lab:** Use Option A. It integrates cleanly with RHEL and is easier to manage.

### 2.3 Verify the Installation

```bash
ansible --version
```

Expected output (version may vary):

```
ansible [core 2.14.x]
  config file = None
  configured module search path = ...
  ansible python module location = ...
  ...
```

Confirm Ansible can reach itself:

```bash
ansible localhost -m ping
```

Expected output:

```
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you see `SUCCESS` and `pong`, Ansible Core is working.

---

## Step 3 — Set Up SSH Key-Based Authentication

Ansible connects to managed nodes over **SSH**. Password-based login works, but key-based authentication is the standard practice because it is more secure and allows automation without interactive password prompts.

### 3.1 Generate an SSH Key Pair on the Control Node

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

| Flag | Meaning |
|------|---------|
| `-t rsa` | Key type: RSA |
| `-b 4096` | Key size: 4096 bits |
| `-f ~/.ssh/id_rsa` | File path for the key |
| `-N ""` | No passphrase (acceptable for labs) |

### 3.2 Copy the Public Key to the Managed Node

```bash
ssh-copy-id user@192.168.1.20
```

You will be prompted for the managed node's password **one last time**. After this, all future connections will use the key.

> **Replace `user`** with the actual username on the managed node (e.g., `root`, `ansible`, `adminuser`).

### 3.3 Test Passwordless SSH

```bash
ssh user@192.168.1.20
```

You should log in **without being asked for a password**. Type `exit` to return to the control node.

### 3.4 Verify Python on the Managed Node

Ansible requires Python on the managed node. While still logged into the managed node (or via SSH):

```bash
ssh user@192.168.1.20 "python3 --version"
```

If Python is not installed on the managed node:

```bash
# For RHEL-based managed nodes
ssh user@192.168.1.20 "sudo dnf install -y python3"

# For Ubuntu-based managed nodes
ssh user@192.168.1.20 "sudo apt update && sudo apt install -y python3"
```

---

## Step 4 — Create the Ansible Project Structure

A well-organized project makes your automation maintainable and reusable. We will create a dedicated project directory with all necessary files.

### 4.1 Create the Project Directory

On the control node:

```bash
mkdir -p ~/ansible-nginx-lab
cd ~/ansible-nginx-lab
```

### 4.2 Create the Ansible Configuration File

The `ansible.cfg` file defines default settings for your project so you don't have to pass flags every time.

```bash
vi ansible.cfg
```

Add the following content:

```ini
[defaults]
inventory = ./inventory
remote_user = user
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

**What each setting does:**

| Setting | Purpose |
|---------|---------|
| `inventory` | Points to the inventory file in the same directory |
| `remote_user` | The SSH user Ansible will connect as (change `user` to your actual username) |
| `host_key_checking = False` | Skips SSH host key prompts (lab only — enable in production) |
| `become = True` | Automatically escalate to root using `sudo` |
| `become_ask_pass = False` | Do not prompt for sudo password (requires passwordless sudo on managed node) |

> **Important:** Replace `user` in `remote_user` with the actual SSH username for your managed node.

### 4.3 Ensure Passwordless Sudo on the Managed Node

The managed node user must be able to run `sudo` without a password. If not already configured:

```bash
ssh user@192.168.1.20
sudo visudo
```

Add this line at the end (replace `user` with the actual username):

```
user ALL=(ALL) NOPASSWD: ALL
```

Save and exit (`Esc`, then `:wq`).

---

## Step 5 — Create the Inventory File

The inventory file tells Ansible which machines to manage and how to organize them.

### 5.1 Create the Inventory

On the control node, inside `~/ansible-nginx-lab`:

```bash
vi inventory
```

Add the following content:

```ini
[webservers]
web1 ansible_host=192.168.1.20
```

**What this means:**

| Element | Meaning |
|---------|---------|
| `[webservers]` | A group name — you can have multiple groups |
| `web1` | An alias for the managed host (used in logs and output) |
| `ansible_host=192.168.1.20` | The actual IP address to connect to |

### 5.2 Test the Inventory with Ansible

Verify Ansible can reach the managed node:

```bash
cd ~/ansible-nginx-lab
ansible all -m ping
```

Expected output:

```
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you see `SUCCESS`, Ansible can connect to your managed node over SSH and execute modules. You are ready to write playbooks.

If it fails, check:
- Is the IP address correct in the inventory?
- Can you SSH manually: `ssh user@192.168.1.20`?
- Is `remote_user` in `ansible.cfg` set to the correct username?

### 5.3 Gather Facts (Optional — Understanding Your Managed Node)

```bash
ansible web1 -m setup | head -40
```

This returns detailed information about the managed node (OS, IP, memory, CPU, etc.). Ansible uses these **facts** inside playbooks.

---

## Step 6 — Write the Nginx Installation Playbook

Now we write a playbook that installs Nginx and starts it on the managed node.

### 6.1 Create the Playbook File

```bash
vi install_nginx.yml
```

Add the following content:

```yaml
---
- name: Install and Start Nginx Web Server
  hosts: webservers
  become: true

  tasks:

    - name: Install Nginx (RHEL/CentOS/Alma/Rocky)
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install Nginx (Ubuntu/Debian)
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"

    - name: Start and enable Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Open HTTP port in firewall (RHEL/CentOS)
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
      when: ansible_os_family == "RedHat"
      ignore_errors: true
```

### 6.2 Understand the Playbook

| Keyword | Purpose |
|---------|---------|
| `hosts: webservers` | Run this playbook against the `webservers` group from inventory |
| `become: true` | Execute tasks as root (via sudo) |
| `ansible.builtin.dnf` | Module to install packages on RHEL-based systems |
| `ansible.builtin.apt` | Module to install packages on Debian/Ubuntu systems |
| `when:` | Conditional — only run the task if the condition is true |
| `ansible_os_family` | An Ansible fact that identifies the OS type |
| `state: present` | Ensure the package is installed (do not reinstall if already there) |
| `state: started` | Ensure the service is running |
| `enabled: true` | Ensure the service starts automatically on boot |

> **Why two install tasks?** The playbook handles both RHEL and Ubuntu managed nodes. Ansible checks the OS family and runs only the matching task. This makes the playbook portable.

### 6.3 Install the Required Collection for Firewalld

The `ansible.posix.firewalld` module requires the `ansible.posix` collection:

```bash
ansible-galaxy collection install ansible.posix
```

---

## Step 7 — Run the Nginx Installation Playbook

### 7.1 Perform a Dry Run First (Check Mode)

Always validate before making changes:

```bash
ansible-playbook install_nginx.yml --check
```

This simulates the playbook without actually changing anything on the managed node. Review the output for errors.

### 7.2 Run the Playbook

```bash
ansible-playbook install_nginx.yml
```

Expected output:

```
PLAY [Install and Start Nginx Web Server] **************************************

TASK [Gathering Facts] *********************************************************
ok: [web1]

TASK [Install Nginx (RHEL/CentOS/Alma/Rocky)] *********************************
changed: [web1]

TASK [Install Nginx (Ubuntu/Debian)] *******************************************
skipping: [web1]

TASK [Start and enable Nginx service] ******************************************
changed: [web1]

TASK [Open HTTP port in firewall (RHEL/CentOS)] ********************************
changed: [web1]

PLAY RECAP *********************************************************************
web1                       : ok=4    changed=3    unreachable=0    failed=0
```

Key things to observe:
- `changed` means Ansible made a modification
- `skipping` means the `when` condition was false (expected if the managed node is RHEL)
- `failed=0` means everything succeeded

### 7.3 Verify Nginx Is Running on the Managed Node

```bash
ansible webservers -m shell -a "systemctl status nginx --no-pager"
```

### 7.4 Test from Your Browser

Open a browser on your laptop and go to:

```
http://192.168.1.20
```

You should see the **default Nginx welcome page**. This confirms Nginx is installed, running, and accessible over the network.

---

## Step 8 — Create and Deploy a Custom Web Page

Now we replace the default Nginx page with a custom HTML page using a second playbook.

### 8.1 Create the Web Page Template

First, create a `files` directory to store static files:

```bash
mkdir -p ~/ansible-nginx-lab/files
```

Create the custom HTML page:

```bash
vi ~/ansible-nginx-lab/files/index.html
```

Add the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ansible Lab - Web Server</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
            color: #ffffff;
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .container {
            text-align: center;
            padding: 60px 40px;
            background: rgba(255, 255, 255, 0.05);
            border-radius: 20px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            max-width: 700px;
        }
        h1 {
            font-size: 2.5em;
            margin-bottom: 10px;
            color: #e94560;
        }
        .subtitle {
            font-size: 1.2em;
            color: #a8a8b3;
            margin-bottom: 30px;
        }
        .info-box {
            background: rgba(233, 69, 96, 0.1);
            border-left: 4px solid #e94560;
            padding: 20px;
            margin: 20px 0;
            border-radius: 0 10px 10px 0;
            text-align: left;
        }
        .info-box p { margin: 8px 0; font-size: 1em; }
        .label { color: #e94560; font-weight: bold; }
        .footer {
            margin-top: 30px;
            font-size: 0.9em;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Ansible Lab - Success!</h1>
        <p class="subtitle">This page was deployed using Ansible Automation</p>
        <div class="info-box">
            <p><span class="label">Lab:</span> 01 - Install Ansible Core and Deploy Nginx</p>
            <p><span class="label">Managed By:</span> Ansible Core</p>
            <p><span class="label">Web Server:</span> Nginx</p>
            <p><span class="label">Deployed On:</span> MANAGED_HOSTNAME</p>
        </div>
        <p class="footer">Automation makes infrastructure consistent, repeatable, and reliable.</p>
    </div>
</body>
</html>
```

### 8.2 Create the Deployment Playbook

```bash
vi ~/ansible-nginx-lab/deploy_website.yml
```

Add the following content:

```yaml
---
- name: Deploy Custom Web Page to Nginx
  hosts: webservers
  become: true

  tasks:

    - name: Determine Nginx web root path
      ansible.builtin.set_fact:
        nginx_web_root: >-
          {{ '/usr/share/nginx/html' if ansible_os_family == 'RedHat'
             else '/var/www/html' }}

    - name: Deploy custom index.html
      ansible.builtin.copy:
        src: files/index.html
        dest: "{{ nginx_web_root }}/index.html"
        owner: root
        group: root
        mode: '0644'

    - name: Replace placeholder with actual hostname
      ansible.builtin.replace:
        path: "{{ nginx_web_root }}/index.html"
        regexp: 'MANAGED_HOSTNAME'
        replace: "{{ ansible_hostname }}"

    - name: Ensure Nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
```

### 8.3 Understand the Playbook

| Task | What It Does |
|------|-------------|
| `set_fact` | Sets a variable for the web root path based on the OS family |
| `copy` | Copies the HTML file from the control node to the managed node |
| `replace` | Replaces the `MANAGED_HOSTNAME` placeholder with the actual hostname of the managed node |
| `service` | Ensures Nginx is running so the new page is served immediately |

---

## Step 9 — Run the Deployment Playbook

### 9.1 Run the Playbook

```bash
cd ~/ansible-nginx-lab
ansible-playbook deploy_website.yml
```

Expected output:

```
PLAY [Deploy Custom Web Page to Nginx] *****************************************

TASK [Gathering Facts] *********************************************************
ok: [web1]

TASK [Determine Nginx web root path] ******************************************
ok: [web1]

TASK [Deploy custom index.html] ************************************************
changed: [web1]

TASK [Replace placeholder with actual hostname] ********************************
changed: [web1]

TASK [Ensure Nginx is running] *************************************************
ok: [web1]

PLAY RECAP *********************************************************************
web1                       : ok=5    changed=2    unreachable=0    failed=0
```

### 9.2 Verify the Web Page in a Browser

Open your browser and go to:

```
http://192.168.1.20
```

You should see your **custom web page** with the red "Ansible Lab - Success!" heading and the server details. The hostname should show the actual managed node hostname instead of `MANAGED_HOSTNAME`.

### 9.3 Verify Using the Command Line (Alternative)

From the control node:

```bash
curl http://192.168.1.20
```

You should see the HTML content of your custom page.

---

## Step 10 — Create a Combined Playbook (Full Deployment in One Run)

In practice, you often combine all tasks into a single playbook with multiple plays or use a single play. Let us create one playbook that does everything.

### 10.1 Create the Site Playbook

```bash
vi ~/ansible-nginx-lab/site.yml
```

Add the following content:

```yaml
---
- name: Full Stack - Install Nginx and Deploy Web Page
  hosts: webservers
  become: true

  tasks:

    - name: Install Nginx (RHEL-based)
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install Nginx (Debian-based)
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"

    - name: Start and enable Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Open HTTP port in firewall (RHEL-based)
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
      when: ansible_os_family == "RedHat"
      ignore_errors: true

    - name: Set web root path based on OS
      ansible.builtin.set_fact:
        nginx_web_root: >-
          {{ '/usr/share/nginx/html' if ansible_os_family == 'RedHat'
             else '/var/www/html' }}

    - name: Deploy custom index.html
      ansible.builtin.copy:
        src: files/index.html
        dest: "{{ nginx_web_root }}/index.html"
        owner: root
        group: root
        mode: '0644'

    - name: Replace placeholder with actual hostname
      ansible.builtin.replace:
        path: "{{ nginx_web_root }}/index.html"
        regexp: 'MANAGED_HOSTNAME'
        replace: "{{ ansible_hostname }}"
```

### 10.2 Run the Combined Playbook

```bash
ansible-playbook site.yml
```

This single command installs Nginx, opens the firewall, deploys the web page, and inserts the hostname — all in one run.

> **Key learning:** Running this playbook a second time should show `changed=0` for most tasks. This is called **idempotency** — Ansible only makes changes when the desired state differs from the current state.

---

## Step 11 — Verify Everything End-to-End

Run through this final verification checklist:

### 11.1 Verify Ansible Connectivity

```bash
ansible all -m ping
```

### 11.2 Verify Nginx Status on the Managed Node

```bash
ansible webservers -m shell -a "systemctl status nginx --no-pager"
```

### 11.3 Verify the Web Page

```bash
curl -s http://192.168.1.20 | grep "<title>"
```

Expected output:

```
    <title>Ansible Lab - Web Server</title>
```

### 11.4 Verify Idempotency

Run the site playbook again:

```bash
ansible-playbook site.yml
```

Observe that most tasks now show `ok` instead of `changed`. This proves idempotency.

---

## Step 12 — Review the Final Project Structure

Your completed project should look like this:

```
~/ansible-nginx-lab/
├── ansible.cfg              # Project-level Ansible configuration
├── inventory                # List of managed hosts
├── install_nginx.yml        # Playbook: install Nginx only
├── deploy_website.yml       # Playbook: deploy custom web page only
├── site.yml                 # Playbook: full stack (install + deploy)
└── files/
    └── index.html           # Custom web page to deploy
```

| File | Purpose |
|------|---------|
| `ansible.cfg` | Defines defaults (inventory path, SSH user, privilege escalation) |
| `inventory` | Declares which machines Ansible manages, organized in groups |
| `install_nginx.yml` | Installs Nginx and configures firewall |
| `deploy_website.yml` | Copies and customizes the web page |
| `site.yml` | Combined playbook — does both install and deploy |
| `files/index.html` | The static HTML file pushed to the managed node |

---

## Troubleshooting

### Ansible ping fails with "UNREACHABLE"

```bash
# Test SSH manually
ssh user@192.168.1.20

# Check the username in ansible.cfg matches the managed node user
grep remote_user ansible.cfg

# Check inventory IP is correct
cat inventory
```

### Playbook fails with "Permission denied"

Ensure passwordless sudo is configured on the managed node:

```bash
ssh user@192.168.1.20 "sudo whoami"
```

Expected output: `root`. If it prompts for a password, configure `NOPASSWD` in `/etc/sudoers` (see Step 4.3).

### Nginx page not loading in browser

```bash
# Is Nginx running?
ansible webservers -m shell -a "systemctl status nginx --no-pager"

# Is port 80 listening?
ansible webservers -m shell -a "ss -tlnp | grep :80"

# Is the firewall allowing HTTP?
ansible webservers -m shell -a "firewall-cmd --list-all"
```

### "ansible.posix.firewalld" module not found

Install the required collection:

```bash
ansible-galaxy collection install ansible.posix
```

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|-----------------|
| **Control Node** | The machine where Ansible is installed and playbooks are executed from |
| **Managed Node** | The target machine that Ansible configures remotely over SSH |
| **Inventory** | A file listing all managed hosts, organized into groups |
| **Playbook** | A YAML file defining the desired state of your infrastructure |
| **Task** | A single action within a playbook (install a package, copy a file, etc.) |
| **Module** | Built-in Ansible functions (`dnf`, `apt`, `copy`, `service`, `replace`) |
| **Fact** | System information Ansible gathers automatically (`ansible_os_family`, `ansible_hostname`) |
| **Idempotency** | Running a playbook multiple times produces the same result without side effects |
| **Privilege Escalation** | Using `become: true` to run tasks as root via sudo |

---

## Lab Completion Checklist

- [ ] Ansible Core is installed on the control node (`ansible --version` works)
- [ ] SSH key-based authentication is configured (passwordless SSH to managed node)
- [ ] Project structure is created with `ansible.cfg`, `inventory`, playbooks, and `files/`
- [ ] Inventory is verified (`ansible all -m ping` returns SUCCESS)
- [ ] Nginx is installed and running on the managed node
- [ ] Firewall allows HTTP traffic (port 80)
- [ ] Custom web page is accessible in a browser at `http://<managed-node-IP>`
- [ ] The `site.yml` combined playbook runs successfully
- [ ] Running the playbook a second time shows idempotency (minimal `changed` count)

---

*End of Lab 01*
