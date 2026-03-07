# Lab 14 — Transform Data with Filters and Plug-ins

| Field        | Value                                                                      |
|--------------|----------------------------------------------------------------------------|
| **Audience** | Intermediate (Ops / DevOps / Platform Engineers)                           |
| **Duration** | 4 hours (Theory: 30% / Hands-on: 70%)                                      |
| **Outcome**  | Use Jinja2 templating for configuration files; apply common filters (string, list, dict, default, math, date); use advanced filters (ipaddr, JSON/YAML, regex); work with lookup plugins (file, env, password, URL, CSV); create a simple custom filter plugin |
| **Platform** | Red Hat Ansible Automation Platform **2.4** (Automation Controller) — Classic and New UI |
| **Prerequisite** | Labs 02, 04, 05, 07, 11 completed — AAP 2.4 installed, project with playbooks, inventory, job template, machine credential. |

---

## AAP 2.4 UI Navigation Reference (Classic and New UI)

| Menu | Classic UI Location | New UI (Preview) Location |
|------|----------------------|---------------------------|
| **Resources** | Left navigation panel | Left sidebar — Templates, Inventories, Projects |
| **Projects** | Resources → Projects | Resources → Projects |
| **Templates** | Resources → Templates | Resources → Templates |
| **Credentials** | Resources → Credentials | Resources → Credentials |

> **Note:** This lab focuses on playbook development (templates, filters, lookups). Run playbooks via AAP job templates. AAP 2.4 UI paths apply when creating projects and launching jobs.

---

## Lab Topology

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YOUR LAPTOP / WORKSTATION                                    │
│                         - Code editor (VS Code, etc.)                                │
│                         - Browser: https://aap1.lab.local                            │
└──────────────────────────────────────┬──────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│  AAP / CONTROL NODE     │  │  PROJECT (lab-playbooks)│  │  MANAGED NODES           │
│  - Job templates        │  │  - playbooks/*.yml      │  │  - web1, app1, db1       │
│  - Projects             │  │  - templates/*.j2       │  │  - Lab-Static-Inventory  │
│                         │  │  - filter_plugins/      │  │                          │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘

Data Flow: Variables → Jinja2 Templates → Filters → Lookups → Rendered Config → Managed Hosts
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AAP Controller | Running from Lab 02 (`aap1.lab.local`) — **AAP 2.4** |
| Project | `Lab-Project` with `lab-playbooks` directory |
| Inventory | `Lab-Static-Inventory` with hosts (webservers, appservers, databases) |
| Credential | Machine credential (SSH key) |
| Managed Nodes | 2–4 Linux VMs reachable via SSH |
| Collections | `ansible.utils` (for ipaddr filter) — install via `ansible-galaxy collection install ansible.utils` |

---

## Verify Environment Before Starting

1. Log in to AAP: `https://aap1.lab.local`
2. Ensure `Lab-Project` is synced and contains playbooks
3. Install `ansible.utils` if not present: `ansible-galaxy collection install ansible.utils`
4. Create project directory structure: `templates/`, `filter_plugins/` (we will create in lab)

---

## Part 1 — Theory: Jinja2, Filters, Lookups, and Custom Plugins (72 minutes)

### Section 13.1 — Introduction to Jinja2 Templating

#### What Is Jinja2 and Why Use It?

**Jinja2** is a templating engine for Python. Ansible uses it to:

- **Variable substitution** — `{{ variable }}` replaced at runtime
- **Logic** — conditionals, loops, filters
- **Generate configs** — one template, many hosts; each gets host-specific output

| Without templates | With Jinja2 templates |
|-------------------|------------------------|
| One config file per host | One template for all hosts |
| Manual updates | Data-driven; change variable, regenerate |
| Error-prone | Consistent structure |

#### Basic Template Syntax

| Syntax | Purpose |
|--------|---------|
| `{{ variable }}` | Output variable (auto-escaped) |
| `{% if condition %}...{% endif %}` | Conditional block |
| `{% for item in list %}...{% endfor %}` | Loop |
| `{{ var \| filter }}` | Apply filter |
| `{# comment #}` | Template comment |

#### Using Templates in Playbooks

- **ansible.builtin.template** — render a `.j2` file and copy to host
- Template path: relative to playbook or `templates/` directory
- Variables from inventory, vars, facts available in template

#### Creating Configuration File Templates

- Store templates in `templates/` directory (Ansible convention)
- Use `.j2` extension (optional but conventional)
- Variables: `{{ hostname }}`, `{{ ip_address }}`, `{{ port }}`

#### Variable Substitution in Templates

```jinja2
# Example: nginx.conf.j2
server {
    listen {{ http_port | default(80) }};
    server_name {{ inventory_hostname }};
    root {{ web_root }};
}
```

---

### Section 13.2 — Common Jinja2 Filters

#### String Manipulation

| Filter | Example | Result |
|--------|---------|--------|
| `upper` | `"hello" \| upper` | `HELLO` |
| `lower` | `"HELLO" \| lower` | `hello` |
| `replace` | `"foo bar" \| replace(" ", "-")` | `foo-bar` |
| `trim` | `"  x  " \| trim` | `x` |
| `capitalize` | `"hello" \| capitalize` | `Hello` |

#### List Filters

| Filter | Example | Result |
|--------|---------|--------|
| `first` | `[1,2,3] \| first` | `1` |
| `last` | `[1,2,3] \| last` | `3` |
| `unique` | `[1,2,2,3] \| unique` | `[1,2,3]` |
| `sort` | `[3,1,2] \| sort` | `[1,2,3]` |
| `join` | `[1,2,3] \| join(',')` | `1,2,3` |

#### Dictionary Filters

| Filter | Example | Result |
|--------|---------|--------|
| `dict2items` | `{"a":1,"b":2} \| dict2items` | `[{"key":"a","value":1},...]` |
| `items2dict` | `[{"key":"a","value":1}] \| items2dict` | `{"a":1}` |

#### Default Values

| Filter | Example | Result |
|--------|---------|--------|
| `default` | `undefined_var \| default('fallback')` | `fallback` |
| `default` | `"" \| default('x')` | `x` (empty string replaced) |

#### Mathematical Operations

| Filter | Example | Result |
|--------|---------|--------|
| `sum` | `[1,2,3] \| sum` | `6` |
| `min` | `[3,1,2] \| min` | `1` |
| `max` | `[3,1,2] \| max` | `3` |

#### Date and Time Formatting

| Filter | Example | Result |
|--------|---------|--------|
| `strftime` | `ansible_date_time.epoch \| int \| strftime('%Y-%m-%d')` | `2025-03-07` |

---

### Section 13.3 — Advanced Filtering Techniques

#### IP Address and Network Filters (ipaddr)

- **Collection:** `ansible.utils`
- **Filter:** `ansible.utils.ipaddr` or `ipaddr`

| Use | Example |
|-----|---------|
| Validate IP | `"192.168.1.1" \| ansible.utils.ipaddr` |
| Get network | `"192.168.1.10/24" \| ansible.utils.ipaddr('network')` |
| Filter list | `ip_list \| ansible.utils.ipaddr` |

#### JSON and YAML Manipulation

| Filter | Example | Purpose |
|--------|---------|---------|
| `to_json` | `var \| to_json` | Convert to JSON string |
| `to_yaml` | `var \| to_yaml` | Convert to YAML string |
| `from_json` | `json_string \| from_json` | Parse JSON |
| `from_yaml` | `yaml_string \| from_yaml` | Parse YAML |

#### Combining Multiple Filters

- Chain with `|`: `{{ var | upper | replace("X","Y") }}`
- Order matters: left to right

#### Conditional Expressions in Filters

- `default` with condition: `{{ var | default('none') }}`
- Ternary: `{{ (x > 0) | ternary('yes', 'no') }}`

#### Regular Expressions with regex Filters

| Filter | Example |
|--------|---------|
| `regex_search` | `"host01" \| regex_search('\\d+')` → `01` |
| `regex_replace` | `"foo123" \| regex_replace('\\d+', '')` → `foo` |

---

### Section 13.4 — Lookup Plugins

#### What Are Lookup Plugins?

**Lookups** run on the control node and retrieve data from external sources. They are evaluated at runtime.

| Lookup | Purpose |
|--------|---------|
| `file` | Read file contents |
| `template` | Render template (returns string) |
| `env` | Environment variable |
| `password` | Generate/store password |
| `url` | Fetch URL content |
| `csvfile` | Read CSV file |
| `vars` | Variable from inventory |

#### File and Template Lookups

```yaml
- name: Read file
  ansible.builtin.debug:
    msg: "{{ lookup('file', '/etc/hostname') }}"

- name: Render template to variable
  ansible.builtin.set_fact:
    config: "{{ lookup('template', 'config.j2') }}"
```

#### Environment Variable Lookups

```yaml
- name: Get env var
  ansible.builtin.debug:
    msg: "{{ lookup('env', 'HOME') }}"
```

#### Password and Secret Lookups

```yaml
- name: Generate password
  ansible.builtin.set_fact:
    my_password: "{{ lookup('password', '/tmp/secret.txt length=20') }}"
```

#### URL and CSV Lookups

```yaml
- name: Fetch URL
  ansible.builtin.debug:
    msg: "{{ lookup('url', 'https://example.com/api/data') }}"

- name: CSV lookup
  ansible.builtin.debug:
    msg: "{{ lookup('csvfile', 'user lookupfile=users.csv col=0 delimiter=,') }}"
```

#### Using Lookups in Playbooks

- Lookups run on **control node**
- Use in `vars`, `set_fact`, module parameters
- Syntax: `{{ lookup('plugin_name', 'arg1', 'arg2') }}`

---

### Section 13.5 — Introduction to Custom Plugins

#### Overview of Plugin Types

| Type | Purpose |
|------|---------|
| **Filter** | Transform data: `{{ x \| my_filter }}` |
| **Lookup** | Retrieve data: `{{ lookup('my_lookup', 'arg') }}` |
| **Callback** | React to events (e.g., task start, end) |

#### When to Create Custom Plugins

- Reusable logic not in built-in filters
- Domain-specific transformations (e.g., network format)
- Integration with internal APIs

#### Basic Custom Filter Plugin Structure

```python
# filter_plugins/my_filters.py
def my_custom_filter(value, arg):
    """Transform value using arg."""
    return str(value) + str(arg)

class FilterModule:
    def filters(self):
        return {'my_custom_filter': my_custom_filter}
```

#### Installing and Using Custom Plugins

- Place in `filter_plugins/` next to playbook (or in role's `filter_plugins/`)
- Or set `filter_plugins` in `ansible.cfg`
- Use: `{{ var | my_custom_filter('suffix') }}`

---

## Part 2 — Hands-On: Jinja2 Templating (35 minutes)

### Step 1 — Create a Basic Template with Variable Substitution

**Purpose:** Generate a config file from a template.

**Detailed steps:**

1. Create `templates` directory in your project:

```bash
mkdir -p ~/ansible-filters-lab/templates
# Or on AAP server: sudo mkdir -p /var/lib/awx/projects/lab-playbooks/templates
```

2. Create `templates/app_config.j2`:

```jinja2
# Application configuration - generated by Ansible
# Host: {{ inventory_hostname }}
# Generated at: {{ ansible_date_time.iso8601 }}

[server]
hostname = {{ inventory_hostname }}
listen_port = {{ app_port | default(8080) }}
environment = {{ env | default('dev') }}
```

3. Create playbook `template_basic.yml`:

```yaml
---
- name: Deploy config from template
  hosts: all
  gather_facts: true
  vars:
    app_port: 8080
    env: lab
  tasks:
    - name: Deploy app config
      ansible.builtin.template:
        src: app_config.j2
        dest: /tmp/app_config.conf
        mode: '0644'
    - name: Display generated config
      ansible.builtin.slurp:
        src: /tmp/app_config.conf
      register: config_content
    - name: Show config
      ansible.builtin.debug:
        msg: "{{ config_content.content | b64decode }}"
```

4. Run the playbook (or create AAP job template and launch)

**Verify:** Config file on each host contains host-specific values.

---

### Step 2 — Use Conditionals and Loops in Templates

**Purpose:** Add logic to templates.

**Detailed steps:**

1. Create `templates/nginx_vhost.j2`:

```jinja2
# Nginx virtual host - {{ inventory_hostname }}
server {
    listen {{ http_port | default(80) }};
    server_name {{ inventory_hostname }};
    root {{ web_root | default('/var/www/html') }};

{% if enable_ssl | default(false) %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
{% endif %}

{% for path in extra_locations | default([]) %}
    location {{ path }} {
        proxy_pass http://localhost:8080;
    }
{% endfor %}
}
```

2. Create playbook with vars and run

**Verify:** Template renders with conditional SSL and looped locations.

---

## Part 3 — Hands-On: Common Filters (40 minutes)

### Step 3 — String and List Filters

**Purpose:** Apply string and list filters in playbooks.

**Detailed steps:**

1. Create playbook `filters_demo.yml`:

```yaml
---
- name: Filter demo
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    my_string: "  Hello World  "
    my_list: [3, 1, 2, 2, 4]
    my_dict: {a: 1, b: 2, c: 3}
  tasks:
    - name: String filters
      ansible.builtin.debug:
        msg: "upper={{ my_string | trim | upper }}, replace={{ my_string | replace(' ', '-') }}"
    - name: List filters
      ansible.builtin.debug:
        msg: "first={{ my_list | first }}, last={{ my_list | last }}, unique={{ my_list | unique | sort }}, join={{ my_list | join(',') }}"
    - name: Math filters
      ansible.builtin.debug:
        msg: "sum={{ my_list | sum }}, min={{ my_list | min }}, max={{ my_list | max }}"
```

2. Run the playbook

**Verify:** Output shows transformed values.

---

### Step 4 — Default and Dictionary Filters

**Purpose:** Use default values and dict transformations.

**Detailed steps:**

1. Add tasks to `filters_demo.yml`:

```yaml
    - name: Default filter
      ansible.builtin.debug:
        msg: "undefined={{ not_defined | default('fallback') }}, empty={{ '' | default('empty') }}"
    - name: Dict filters
      ansible.builtin.debug:
        msg: "dict2items={{ my_dict | dict2items | map(attribute='key') | list }}"
```

2. Run the playbook

**Verify:** Default and dict filters work as expected.

---

### Step 5 — Date Formatting

**Purpose:** Format dates with strftime.

**Detailed steps:**

1. Add task:

```yaml
    - name: Date filter
      ansible.builtin.debug:
        msg: "Date={{ ansible_date_time.epoch | int | strftime('%Y-%m-%d %H:%M') }}"
```

2. Run (use `hosts: all` and `gather_facts: true` to get `ansible_date_time`)

**Verify:** Formatted date appears in output.

---

## Part 4 — Hands-On: Advanced Filters (35 minutes)

### Step 6 — IP Address Filter (ansible.utils)

**Purpose:** Use ipaddr filter for network operations.

**Detailed steps:**

1. Ensure `ansible.utils` is installed: `ansible-galaxy collection install ansible.utils`
2. Create playbook `ip_filter_demo.yml`:

```yaml
---
- name: IP filter demo
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    ip_list: ["192.168.1.1", "10.0.0.1", "invalid", "172.16.0.1"]
  tasks:
    - name: Filter valid IPs
      ansible.builtin.debug:
        msg: "Valid IPs: {{ ip_list | ansible.utils.ipaddr }}"
    - name: Network from CIDR
      ansible.builtin.debug:
        msg: "Network: {{ '192.168.1.10/24' | ansible.utils.ipaddr('network') }}"
```

3. Run the playbook

**Verify:** Valid IPs filtered; network calculated.

---

### Step 7 — JSON and YAML Filters

**Purpose:** Convert and parse JSON/YAML.

**Detailed steps:**

1. Create playbook `json_yaml_demo.yml`:

```yaml
---
- name: JSON/YAML demo
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    my_dict: {name: "app", port: 8080}
    json_str: '{"key": "value"}'
  tasks:
    - name: To JSON
      ansible.builtin.debug:
        msg: "{{ my_dict | to_json }}"
    - name: From JSON
      ansible.builtin.debug:
        msg: "{{ json_str | from_json }}"
    - name: To YAML
      ansible.builtin.debug:
        msg: "{{ my_dict | to_yaml }}"
```

2. Run the playbook

**Verify:** JSON/YAML conversion works.

---

### Step 8 — Regex Filters

**Purpose:** Use regex_search and regex_replace.

**Detailed steps:**

1. Add tasks:

```yaml
    - name: Regex search
      ansible.builtin.debug:
        msg: "Digits: {{ 'host99prod' | regex_search('\\d+') }}"
    - name: Regex replace
      ansible.builtin.debug:
        msg: "Replaced: {{ 'foo123bar' | regex_replace('\\d+', '') }}"
```

2. Run the playbook

**Verify:** Regex filters produce expected output.

---

## Part 5 — Hands-On: Lookup Plugins (35 minutes)

### Step 9 — File Lookup

**Purpose:** Read file contents with lookup.

**Detailed steps:**

1. Create a test file: `echo "Hello from file" > /tmp/test_lookup.txt`
2. Create playbook `lookup_file.yml`:

```yaml
---
- name: File lookup
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Read file
      ansible.builtin.debug:
        msg: "{{ lookup('file', '/tmp/test_lookup.txt') }}"
```

3. Run (adjust path for your environment)

**Verify:** File content appears in output.

---

### Step 10 — Environment Variable Lookup

**Purpose:** Retrieve environment variables.

**Detailed steps:**

1. Add task:

```yaml
    - name: Env lookup
      ansible.builtin.debug:
        msg: "HOME={{ lookup('env', 'HOME') }}, USER={{ lookup('env', 'USER') }}"
```

2. Run the playbook

**Verify:** Environment variables are retrieved.

---

### Step 11 — Password Lookup

**Purpose:** Generate and store a password.

**Detailed steps:**

1. Create playbook `lookup_password.yml`:

```yaml
---
- name: Password lookup
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Generate password
      ansible.builtin.set_fact:
        gen_pass: "{{ lookup('password', '/tmp/ansible_generated_pass length=16') }}"
    - name: Show password (first run creates; subsequent runs reuse)
      ansible.builtin.debug:
        msg: "Generated: {{ gen_pass }}"
```

2. Run twice — first creates file; second reads existing

**Verify:** Password generated and persisted.

---

### Step 12 — Template Lookup

**Purpose:** Render template to variable (without copying to host).

**Detailed steps:**

1. Create `templates/snippet.j2`: `Host={{ inventory_hostname }}, Port={{ port }}`
2. Create playbook:

```yaml
---
- name: Template lookup
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    port: 8080
  tasks:
    - name: Render template to var
      ansible.builtin.set_fact:
        rendered: "{{ lookup('template', 'snippet.j2') }}"
    - name: Show
      ansible.builtin.debug:
        msg: "{{ rendered }}"
```

3. Run (ensure `templates/snippet.j2` is in playbook directory)

**Verify:** Template rendered to string.

---

## Part 6 — Hands-On: Custom Filter Plugin (30 minutes)

### Step 13 — Create a Simple Custom Filter

**Purpose:** Write and use a custom filter plugin.

**Detailed steps:**

1. Create `filter_plugins` directory next to your playbook:

```bash
mkdir -p ~/ansible-filters-lab/filter_plugins
# Or: /var/lib/awx/projects/lab-playbooks/filter_plugins
```

2. Create `filter_plugins/custom_filters.py`:

```python
# Custom filter plugins for Lab 14
def format_host_display(value, separator='-'):
    """Format hostname for display: replace dots with separator."""
    if not value:
        return ''
    return str(value).replace('.', separator)

def multiply_by(value, factor=1):
    """Multiply value by factor."""
    try:
        return int(value) * int(factor)
    except (ValueError, TypeError):
        return value

class FilterModule:
    def filters(self):
        return {
            'format_host_display': format_host_display,
            'multiply_by': multiply_by,
        }
```

3. Create playbook `custom_filter_demo.yml`:

```yaml
---
- name: Custom filter demo
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    hostname: "web1.lab.local"
    number: 5
  tasks:
    - name: Use format_host_display
      ansible.builtin.debug:
        msg: "Formatted: {{ hostname | format_host_display }} and {{ hostname | format_host_display('_') }}"
    - name: Use multiply_by
      ansible.builtin.debug:
        msg: "Result: {{ number | multiply_by(10) }}"
```

4. Run the playbook (from directory containing `filter_plugins/`)

**Verify:** Custom filters produce expected output.

---

### Step 14 — Use Custom Filter in Template

**Purpose:** Apply custom filter inside a Jinja2 template.

**Detailed steps:**

1. Create `templates/with_custom_filter.j2`:

```jinja2
# Config for {{ inventory_hostname | format_host_display }}
# Display name: {{ inventory_hostname | format_host_display('_') }}
```

2. Create playbook that uses this template with `ansible.builtin.template`
3. Run the playbook

**Verify:** Custom filter works inside template.

---

## Part 7 — Real-Time Use Case Discussion (20 minutes)

### Case Study: Network Operations — 500+ Router Configs from Single Data Source

**Background:**

A network operations team manages 500+ routers across multiple sites. Each router needs a unique configuration (hostname, IP, VLANs, ACLs) but follows the same structure. Previously, configs were maintained manually in spreadsheets and copied into devices — error-prone and slow.

**Challenges:**

| Challenge | Impact |
|-----------|--------|
| 500+ devices | Manual config impossible to scale |
| Consistency | Typos, missing sections, drift |
| Change management | Updates required days of work |
| Validation | No automated IP/network validation |

**Solution: Jinja2 Templates + Filters**

#### 1. Single Data Source (YAML/JSON)

```yaml
# group_vars/routers.yml
routers:
  - hostname: router-site01
    ip: 192.168.1.1
    vlan_10: 10.0.10.0/24
    vlan_20: 10.0.20.0/24
  - hostname: router-site02
    ip: 192.168.1.2
    ...
```

#### 2. Jinja2 Template for Router Config

```jinja2
! Router config - {{ item.hostname }}
hostname {{ item.hostname | upper }}
interface GigabitEthernet0/0
 ip address {{ item.ip }} {{ item.vlan_10 | ansible.utils.ipaddr('netmask') }}
!
ip access-list standard ALLOW-MGMT
 permit {{ mgmt_network | ansible.utils.ipaddr('network') }} {{ mgmt_network | ansible.utils.ipaddr('netmask') }}
!
```

#### 3. Filters Used

| Filter | Purpose |
|--------|---------|
| `upper` | Hostname in uppercase |
| `ansible.utils.ipaddr` | Validate IP; get network/netmask |
| `default` | Fallback for optional fields |
| `regex_replace` | Sanitize input |

#### 4. Lookups for External Data

- `lookup('file', 'acl_templates.txt')` — standard ACL blocks
- `lookup('csvfile', ...)` — site data from CMDB export
- `lookup('url', 'https://internal/api/sites')` — dynamic site list

**Results:**

| Metric | Before | After |
|--------|--------|------|
| Config generation time | 2 weeks (manual) | 15 minutes (automated) |
| Config errors | 5–10% | < 0.5% |
| Change deployment | Days | Minutes |
| Consistency | Low | High (template-driven) |

**Key takeaway:** Jinja2 templates plus filters and lookups turn a single data source into hundreds of consistent, validated configurations — essential for network automation at scale.

---

## Troubleshooting

### Template Not Found

| Cause | Fix |
|-------|-----|
| Wrong path | Use path relative to playbook or `templates/` |
| Missing directory | Create `templates/` in project |
| AAP project structure | Ensure templates are in synced project |

### Filter Not Found

| Cause | Fix |
|-------|-----|
| Wrong collection | Install collection (e.g., `ansible.utils` for ipaddr) |
| Typo | Use correct filter name; check docs |
| Custom filter | Ensure `filter_plugins/` in correct location |

### Lookup Fails

| Cause | Fix |
|-------|-----|
| File not found | Path is on control node; verify file exists |
| Permission denied | Check file permissions |
| URL unreachable | Ensure control node can reach URL |

### Custom Filter Not Loading

| Cause | Fix |
|-------|-----|
| Wrong location | `filter_plugins/` next to playbook or in role |
| Python syntax error | Validate plugin syntax |
| Wrong method name | `FilterModule.filters()` must return dict |

---

## Key Concepts Covered in This Lab

| Concept | What You Learned |
|---------|-----------------|
| Jinja2 | Templating for variable substitution, conditionals, loops |
| template module | Render .j2 file and deploy to host |
| String filters | upper, lower, replace, trim |
| List filters | first, last, unique, sort, join |
| Dict filters | dict2items, items2dict |
| default filter | Fallback for undefined/empty |
| Math filters | sum, min, max |
| ipaddr filter | ansible.utils; validate IP, get network |
| JSON/YAML | to_json, from_json, to_yaml, from_yaml |
| regex | regex_search, regex_replace |
| Lookups | file, env, password, template, url, csvfile |
| Custom filter | Python plugin in filter_plugins/ |

---

## Lab Completion Checklist

- [ ] Created Jinja2 template with variable substitution
- [ ] Used conditionals and loops in template
- [ ] Applied string filters (upper, lower, replace)
- [ ] Applied list filters (first, last, unique, sort, join)
- [ ] Applied dict filters (dict2items)
- [ ] Used default filter
- [ ] Used math filters (sum, min, max)
- [ ] Used date formatting (strftime)
- [ ] Used ipaddr filter (ansible.utils)
- [ ] Used to_json, from_json, to_yaml
- [ ] Used regex_search, regex_replace
- [ ] Used file lookup
- [ ] Used env lookup
- [ ] Used password lookup
- [ ] Used template lookup
- [ ] Created custom filter plugin
- [ ] Used custom filter in playbook and template
- [ ] Can explain network ops use case (500+ router configs)

---

## Quick Reference — Common Filters

| Filter | Example |
|--------|---------|
| default | `{{ x \| default('y') }}` |
| upper/lower | `{{ s \| upper }}` |
| join | `{{ list \| join(',') }}` |
| to_json | `{{ var \| to_json }}` |
| ipaddr | `{{ ip \| ansible.utils.ipaddr }}` |

---

## Quick Reference — Lookup Syntax

| Lookup | Example |
|--------|---------|
| file | `{{ lookup('file', '/path/file') }}` |
| env | `{{ lookup('env', 'VAR') }}` |
| password | `{{ lookup('password', '/path length=20') }}` |
| template | `{{ lookup('template', 'file.j2') }}` |

---

*End of Lab 14*
