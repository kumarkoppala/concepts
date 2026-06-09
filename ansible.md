# Ansible

## Why Not Just Shell Scripts?

Shell scripts work, but they fall apart when you're managing many servers:

- You SSH into each server and run the script manually — doesn't scale
- Different servers might have different OS versions, users, paths — scripts break
- No easy way to run the same script on 50 servers at once
- Scripts are not idempotent by default — you have to write that logic yourself
- Hard to track what state each server is in

This is where **configuration management** tools like Ansible come in.

---

## What is Configuration Management?

When you spin up a fresh server, it's just a blank machine. You have to install packages, create users, update configs, start services — that's **configuring the server**.

**Configuration management** means doing all of this — create, read, update, delete — through code/scripts, not manually. So the server's state is always predictable and reproducible.

> In shell: everything is a command.  
> In Ansible: everything is a **module** (a pre-built, reusable unit of work).

---

## Push vs Pull Architecture

Most config management tools work in one of two ways:

**Pull** — every server checks in with a central server periodically asking "do I have new configs?" (like you going to the courier office every day to check if your package arrived)
- Wastes resources
- Unnecessary traffic even when nothing changed
- Examples: Chef, Puppet

**Push** — you sit at your machine and push config to all servers when you need to (like the courier delivering to your door when there's something to deliver)
- You control when changes happen
- No agent needed on target servers
- Ansible works this way — over plain SSH

---

## Inventory File

Ansible needs to know which servers to talk to. That list is called an **inventory**.

Simplest form — just an IP or hostname:

```
172.31.27.248
```

You can group servers:

```ini
[frontend]
172.31.10.5
172.31.10.6

[backend]
172.31.20.5

[db]
172.31.30.5
```

Then you can target a group by name in your commands or playbooks.

---

## Ad-hoc Commands

One-off commands without writing a playbook. Good for quick tasks or testing connectivity.

```bash
ansible all -i 172.31.27.248, -e ansible_user=ec2-user -e ansible_password=DevOps321 -m ping
```

| Part | What it means |
|------|---------------|
| `all` | target all hosts in inventory |
| `-i 172.31.27.248,` | inline inventory (comma at end is required for single IP) |
| `-e ansible_user=...` | extra variable — which user to SSH as |
| `-e ansible_password=...` | SSH password |
| `-m ping` | module to run (`ping` checks if Ansible can connect) |

The `ping` module doesn't ping like ICMP ping — it checks if Ansible can SSH in and Python is available on the target.

---

## Playbooks

When you have multiple tasks to run against a server, you write them in a file — that's a **playbook**.

```yaml
- name: ping the server
  hosts: frontend
  tasks:
    - name: ping the server
      ping:
```

- `hosts` — which group from your inventory to target
- `tasks` — list of modules to run in order
- Each task has a `name` (for readability) and a module (`ping`, `dnf`, `copy`, etc.)

---

## YAML Basics

Playbooks are written in YAML. It's just a structured way to write key-value data — like a form.

```yaml
name: sivakumar
branch: hyderabad
date: 01-jun-2026
```

**List** — use `-`:

```yaml
tasks:
  - name: install nginx
  - name: start nginx
```

**Map / nested** — indent under a key:

```yaml
hosts:
  frontend:
    ip: 172.31.10.5
```

Key rule: **indentation matters** — use spaces, never tabs.

---

## Variables

Variables let you avoid hardcoding values. You reference them with `{{ variable_name }}`.

### Play-level vars

Defined in the playbook itself, available to all tasks in that play.

```yaml
- name: demo
  hosts: localhost
  connection: local
  vars:
    COURSE: "Ansible"
    TRAINER: "sivakumar"
    DURATION: "150HRS"
  tasks:
  - debug:
      msg: "Learning {{ COURSE }} with {{ TRAINER }} for {{ DURATION }}"
```

### Task-level vars

Defined inside a specific task — overrides play-level vars for that task only.

```yaml
  tasks:
  - name: print info
    vars:
      COURSE: "DevSecOps with AIOps"  # overrides play-level COURSE
    debug:
      msg: "Learning {{ COURSE }}"
```

### vars_files

Keep variables in a separate YAML file and load it in the playbook. Cleaner for large configs.

```yaml
# course.yaml
COURSE: Ansible
TRAINER: sivakumar
DURATION: 10HRS
```

```yaml
- name: demo
  hosts: local
  connection: local
  vars_files:
  - course.yaml
  tasks:
  - debug:
      msg: "{{ COURSE }} by {{ TRAINER }}"
```

### vars_prompt

Ask the user for input at runtime (like `read` in shell).

```yaml
  vars_prompt:
  - name: username
    prompt: "Please enter username"
    private: false       # shows input as you type

  - name: password
    prompt: "Please enter password"
    private: true        # hides input
```

### group_vars

Variables scoped to an inventory group. Ansible auto-loads files from a `group_vars/` folder next to your playbook.

```
group_vars/
  all.yaml        # applies to every host
  frontend.yaml   # applies only to [frontend] group
```

```yaml
# group_vars/frontend.yaml
COURSE: "Ansible from group_vars"
TRAINER: "Sivakumar"
```

### Command-line vars (-e)

Pass variables at runtime — highest priority, overrides everything.

```bash
ansible-playbook 09-vars.yaml -e name=Sivakumar
```

```yaml
  tasks:
  - name: print name
    debug:
      msg: "Hello {{ name }}"
```

### Variable Preference Order (highest → lowest)

When the same variable is defined in multiple places, this is the winner:

```
1. Command-line args  (-e)
2. Task level         (vars: inside a task)
3. vars_files
4. vars_prompt
5. Play level         (vars: at play level)
6. Inventory / group_vars
```

---

## Data Types

```yaml
  vars:
    course: "DevSecOps with AIOps"   # string
    duration_in_hrs: 150             # number (int)
    is_live_sessions: true           # boolean
    topics:                          # list
    - linux
    - shell
    - ansible
    more_details:                    # map (key-value pairs)
      months: 5
      trainer: "Sivakumar Reddy"
```

Access map values: `{{ more_details.trainer }}`  
Access list values: `{{ topics[0] }}`

---

## Conditions

Use `when:` to run a task only if a condition is true. No brackets needed — it's a Jinja2 expression.

```yaml
  vars:
    live_sessions: 4
  tasks:
  - name: more than 5 sessions?
    debug:
      msg: "Yes, Ansible took more than 5 sessions"
    when: live_sessions > 5
```

Even / odd check:

```yaml
  vars:
    given_number: 7
  tasks:
  - name: is even
    debug:
      msg: "{{ given_number }} is even"
    when: given_number % 2 == 0

  - name: is odd
    debug:
      msg: "{{ given_number }} is odd"
    when: given_number % 2 != 0
```

---

## ansible_facts

Ansible automatically collects info about the target server before running tasks — OS, IP, architecture, memory, etc. This is stored in `ansible_facts`.

```yaml
  tasks:
  - name: print all facts
    debug:
      msg: "{{ ansible_facts }}"
```

Useful for writing playbooks that work on both RedHat and Debian servers:

```yaml
  tasks:
  - name: RedHat family
    debug:
      msg: "Package manager is dnf"
    when: ansible_facts.os_family == "RedHat"

  - name: Debian family
    debug:
      msg: "Package manager is apt-get"
    when: ansible_facts.os_family == "Debian"
```

`ansible_facts` is a **reserved special variable** — Ansible fills it automatically, you just use it.

---

## Loops

When you need to do the same thing for multiple items, use `loop:`. The current item is always available as `{{ item }}`.

### Simple list

```yaml
  tasks:
  - name: greet everyone
    debug:
      msg: "Hi {{ item }}"
    loop:
    - Ramesh
    - Raheem
    - John
```

### Install multiple packages

```yaml
- name: install packages
  hosts: frontend
  become: yes        # run as sudo
  tasks:
  - name: install packages
    package:
      name: "{{ item }}"
      state: present
    loop:
    - nginx
    - mysql
    - mongodb-mongosh
    - git
```

### Loop with maps — different state per item

When each item needs its own properties, use inline maps with `item.key`:

```yaml
  tasks:
  - name: manage packages
    package:
      name: "{{ item.name }}"
      state: "{{ item.state }}"
    loop:
    - { name: "nginx", state: "present" }   # install
    - { name: "mysql", state: "absent" }    # remove
    - { name: "zip",   state: "present" }   # install
```

---

## Filters

Ansible doesn't let you write custom functions, but it ships with a lot of built-in filters for manipulating data. Syntax: `{{ variable | filter }}`.

```yaml
  vars:
    topics: "Linux, Shell, Ansible, Terraform"
    full_name: "Sivakumar Reddy M"
    ip_address: "178.90.78.17"
    tagMap:
      name: Ansible
      project: roboshop
      environment: dev
    tagsList:
    - { key: "name",        value: "Ansible" }
    - { key: "project",     value: "roboshop" }
    - { key: "environment", value: "dev" }

  tasks:
  - name: string to list
    debug:
      msg: "{{ topics | split(',') }}"

  - name: default value if variable not defined
    debug:
      msg: "Hi {{ GREETING | default('Good Morning') }}"

  - name: map to list
    debug:
      msg: "{{ tagMap | dict2items }}"

  - name: list to map
    debug:
      msg: "{{ tagsList | items2dict }}"

  - name: uppercase
    debug:
      msg: "{{ full_name | upper }}"

  - name: lowercase
    debug:
      msg: "{{ full_name | lower }}"

  - name: validate ipv4
    debug:
      msg: "{{ ip_address | ansible.utils.ipv4 }}"
```

| Filter | What it does |
|--------|-------------|
| `split(',')` | Split string into list by delimiter |
| `default('val')` | Use fallback if variable is undefined |
| `dict2items` | Convert map → list of `{key, value}` pairs |
| `items2dict` | Convert list of `{key, value}` pairs → map |
| `upper` / `lower` | Change string case |
| `ansible.utils.ipv4` | Validate/filter IPv4 addresses |

---

## shell vs command

When there's no Ansible module for what you need, you can run raw Linux commands. Two modules for this:

| | `shell` | `command` |
|-|---------|-----------|
| Pipes (`\|`) | Yes | No |
| Redirections (`>`, `>>`) | Yes | No |
| Shell variables (`$VAR`) | Yes | No |
| Use when | You need pipes or redirections | Simple command, no shell features |

```yaml
  tasks:
  - name: needs pipe — use shell
    shell: cat /etc/passwd | grep root

  - name: redirect output to file
    shell: ls -ltr > /tmp/files.txt

  - name: simple command — no pipe needed
    command: id roboshop
```

**Rule of thumb:** prefer `command` when you don't need shell features — it's safer. Use `shell` only when you need pipes or redirections.

---

## register

Captures the output of a task into a variable, so you can use it in later tasks — same idea as `VAR=$(command)` in shell.

```yaml
  tasks:
  - name: run command and capture output
    shell: cat /etc/passwd | grep root
    register: users_output

  - name: use the captured output
    debug:
      msg: "{{ users_output }}"
```

`users_output` is a map with keys like `.stdout`, `.stderr`, `.rc` (return code), `.stdout_lines`.

---

## Error Handling

By default Ansible stops the whole play when any task fails. Use `ignore_errors: true` to continue even if a task fails — useful when failure is expected and you want to branch on it.

```yaml
  tasks:
  - name: check if roboshop user exists
    command: id roboshop
    register: user_output
    ignore_errors: true          # don't stop if user doesn't exist

  - name: create user only if missing
    command: useradd roboshop
    when: user_output.rc != 0   # rc=0 means success, non-zero means failed
```

Flow: run `id roboshop` → capture result → if it failed (rc != 0) → create the user.

---

## Idempotency

Ansible **modules** are idempotent by default — run the same playbook 10 times, the result is the same. The `user:` module checks if the user exists before creating:

```yaml
  tasks:
  - name: create expense user
    user:
      name: expense    # same as useradd expense, but idempotent
```

When you use `shell` or `command`, **you** are responsible for idempotency — Ansible can't know what your shell command does. The `register` + `when` + `ignore_errors` pattern above is the standard way to handle it manually.

---

## Real-world: AWS EC2 + Route53

Ansible has AWS modules (`amazon.aws.*`) so you can provision infrastructure the same way you configure servers. From `roboshop.yaml`:

```yaml
- name: roboshop instances management
  hosts: local
  connection: local
  vars:
    instances: ["mongodb", "redis", "mysql", "rabbitmq", "catalogue", "user", "cart", "shipping", "payment", "frontend"]
    action: create          # change to "delete" to tear down
  tasks:
  - name: create ec2 instances
    amazon.aws.ec2_instance:
      instance_type: t3.micro
      name: "roboshop-{{ item }}"
      image_id: "{{ image_id }}"
    loop: "{{ instances }}"
    register: ec2_output
    when: action == "create"

  - name: create r53 DNS records
    amazon.aws.route53:
      state: present
      zone: "{{ domain_name }}"
      record: "{{ item.item }}.{{ domain_name }}"   # mongodb.daws90s.shop
      type: A
      value: "{{ item.instances[0].private_ip_address }}"
    loop: "{{ ec2_output.results }}"
    when: action == "create"
```

Key ideas:
- `register: ec2_output` captures the created instance data
- `ec2_output.results` is a list — one entry per loop iteration
- `item.item` is the original loop value (`mongodb`, `redis`, etc.), `item.instances[0]` is the EC2 response
- Multiple `when` conditions — all must be true (AND logic):
  ```yaml
  when:
  - item.item == "frontend"
  - action == "create"
  ```

---

## ansible.cfg

Ansible looks for its config file in this order — first match wins:

```
1. ANSIBLE_CONFIG   (environment variable pointing to a file)
2. ./ansible.cfg    (current working directory)
3. ~/.ansible.cfg   (home directory)
4. /etc/ansible/ansible.cfg  (system default)
```

The most common setup is a project-level `ansible.cfg` in the same folder as your playbooks:

```ini
[defaults]
inventory = ./inventory.ini        # so you don't have to pass -i every time
log_path  = /var/log/roboshop/ansible.log
```

Without this, you'd have to type `ansible-playbook -i inventory.ini playbook.yaml` every run. With it, just `ansible-playbook playbook.yaml`.

---

## Templates

A template is a config file with **placeholders** instead of hardcoded values. Ansible fills in the actual values at deploy time using variables. Template files use the `.j2` extension (Jinja2).

**Why templates?** Config files often have values that change per environment (dev/staging/prod) or per service — hostnames, ports, URLs. Rather than maintaining separate files for each, you write one template and let variables do the work.

```
template file (.j2)  +  variables  →  final config file on the server
```

### Example — systemd service file

`catalogue.service.j2`:
```ini
[Unit]
Description = Catalogue Service

[Service]
User=roboshop
Environment=MONGO=true
Environment=MONGO_URL="mongodb://{{ MONGODB_HOST }}:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue

[Install]
WantedBy=multi-user.target
```

`cart.service.j2`:
```ini
[Service]
User=roboshop
Environment=REDIS_HOST={{ REDIS_HOST }}
Environment=CATALOGUE_HOST={{ CATALOGUE_HOST }}
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
```

`{{ MONGODB_HOST }}`, `{{ REDIS_HOST }}`, `{{ CATALOGUE_HOST }}` are placeholders — Ansible replaces them with real values from your variables.

### Using the template module

```yaml
  tasks:
  - name: deploy catalogue service file
    template:
      src: catalogue.service.j2    # template on your machine
      dest: /etc/systemd/system/catalogue.service   # destination on the server
```
---

## Tags

Tags let you run or skip specific tasks without changing the playbook. You add a `tags:` list to a task, then pass `--tags` or `--skip-tags` on the CLI.

```yaml
- name: check catalogue schema
  shell: mongosh --host "{{ MONGODB_HOST }}" --eval 'db.getMongo().getDBNames().indexOf("catalogue")'
  register: catalogue_output
  tags:
  - mongodb

- name: load products
  shell: mongosh --host "{{ MONGODB_HOST }}" </app/db/master-data.js
  when: catalogue_output.stdout | int < 0
  tags:
  - mongodb
```

```bash
ansible-playbook playbook.yaml --tags mongodb        # run only tasks tagged "mongodb"
ansible-playbook playbook.yaml --skip-tags mongodb   # run everything except "mongodb" tasks
ansible-playbook playbook.yaml --list-tags            # list all available tags
```

Real use case from `roboshop-ansible`: the MongoDB data-loading tasks are tagged `mongodb` so you can re-run only them after a schema change without re-running the entire playbook.

---

## Roles

A **role** is a standardised directory structure for organising a playbook into reusable pieces — tasks, variables, files, templates, handlers, and dependencies all live in well-known folders. Ansible auto-loads `main.yaml` from each folder.

```
roles/
  catalogue/
    tasks/        ← main.yaml (required — what the role does)
    vars/         ← main.yaml (role-specific variables)
    files/        ← static files referenced by copy module
    templates/    ← .j2 template files
    handlers/     ← main.yaml (triggered by notify)
    meta/         ← main.yaml (dependencies — other roles to run first)
    defaults/     ← main.yaml (lowest-priority default vars)
```

**Using a role in a play:**

```yaml
# roboshop.yaml
- name: configure "{{ component }}" server
  hosts: "{{ component }}"
  become: yes
  roles:
  - "{{ component }}"    # loads roles/mongodb/, roles/redis/, etc.
```

Run it:
```bash
ansible-playbook -e "component=mongodb" roboshop.yaml
ansible-playbook -e "component=redis"   roboshop.yaml
```

**Role dependencies (meta/main.yaml):**

A role can declare that it depends on another role. The dependency is always run first.

```yaml
# roles/catalogue/meta/main.yaml
dependencies:
  - role: common    # common role runs before catalogue role
```

**Calling a specific task file from another role:**

```yaml
# instead of running all of common's main.yaml, run just one file
- name: application setup
  import_role:
    name: common
    tasks_from: app-setup    # runs roles/common/tasks/app-setup.yaml
```

---

## Handlers

A **handler** is a task that only runs when it's explicitly notified by another task — and only once, at the end of the play, even if notified multiple times. This is perfect for service restarts: you don't want to restart nginx after every config file change if there are five changes; you want to restart it once at the end.

**Define a handler** in `roles/<role>/handlers/main.yaml`:

```yaml
# roles/frontend/handlers/main.yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
    enabled: yes
```

**Trigger it** with `notify:` in a task:

```yaml
# roles/frontend/tasks/main.yaml
- name: copy nginx conf
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx    # name must match handler name exactly
```

How it works:
- `notify` queues the handler name — it doesn't run immediately
- If the task didn't change anything (idempotent run), notify is not triggered
- All queued handlers run once at the end of the play
- If the same handler is notified 5 times, it still only runs once

---

## include_role vs import_role

Both load a role inside a task list, but they differ in **when the role is parsed**.

| | `import_role` | `include_role` |
|---|---|---|
| Parsed | Before execution (static) | At runtime when the task runs (dynamic) |
| Tags cascade to role tasks | Yes | No |
| `when:` applies to | All tasks inside the role | Only the include statement itself |
| Supports `loop:` | No | Yes |

```yaml
# static — parsed before run; tags and when: apply to all tasks inside common
- name: application setup
  import_role:
    name: common
    tasks_from: app-setup

# dynamic — parsed at runtime; useful with loops
- name: setup for multiple items
  include_role:
    name: common
  loop:
  - item1
  - item2
```

**When to use which:**

- Use `import_role` (default choice) — tags work correctly, `when:` guards the whole role, and errors are caught before execution starts.
- Use `include_role` when you need to loop over a role, or when the role name is determined dynamically at runtime.

```yaml
# from ansible-include-vs-import repo — demonstrates the difference
- name: call common role
  import_role:       # swap this for include_role to see behaviour differ
    name: common
  tags:
  - hi
  when: os == "RedHat"
```

With `import_role`: the `hi` tag is applied to every task inside `common`, and `when: os == "RedHat"` is checked for each task.  
With `include_role`: the `hi` tag is ignored inside `common`, and `when:` only guards whether the include runs — tasks inside run regardless of the condition.

---

## Ansible Vault

Ansible Vault encrypts sensitive files (passwords, API keys) so you can commit them to git safely. The encrypted file is still YAML but the content is AES-256 ciphertext.

**Create an encrypted file:**

```bash
ansible-vault create vault.yaml     # opens editor, saves encrypted
```

**Common vault commands:**

```bash
ansible-vault create vault.yaml     # create new encrypted file
ansible-vault edit vault.yaml       # edit (decrypts in memory, re-encrypts on save)
ansible-vault view vault.yaml       # view without editing
ansible-vault encrypt existing.yaml # encrypt a plain file in-place
ansible-vault decrypt vault.yaml    # decrypt to plaintext (careful — don't commit)
ansible-vault rekey vault.yaml      # change the vault password
```

An encrypted vault file looks like this (the content is ciphertext):

```
$ANSIBLE_VAULT;1.1;AES256
34646533346337613336666539636130376230303734316135373730366461646438366431303163
3231386464653064333037313039383566336665623165630a...
```

**Using vault in a playbook:**

Store secrets in `group_vars/all/vault.yaml` (encrypted), plain vars in `group_vars/all/all.yaml`:

```
group_vars/
  all/
    all.yaml      ← plain variables (committed openly)
    vault.yaml    ← encrypted secrets (committed, but unreadable without password)
```

**Running a playbook that uses vault:**

```bash
# prompt for password at runtime
ansible-playbook roboshop.yaml --ask-vault-pass

# read password from a file (for automation)
ansible-playbook roboshop.yaml --vault-password-file ~/.vault_pass
```

The password file should be in `.gitignore` — never commit it.

---

## AWS Secrets Manager Integration

Ansible Vault keeps secrets encrypted in your repo. For production, you often want secrets managed centrally in **AWS Secrets Manager** or **AWS SSM Parameter Store** instead — so secrets aren't in your repo at all and can be rotated without editing playbooks.

**Fetch a secret from AWS SSM Parameter Store using the `aws_ssm` lookup:**

```yaml
- name: get DB password from SSM
  debug:
    msg: "{{ lookup('amazon.aws.aws_ssm', '/roboshop/mysql/password', region='us-east-1') }}"
```

**Fetch a secret from AWS Secrets Manager:**

```yaml
- name: get secret from Secrets Manager
  debug:
    msg: "{{ lookup('amazon.aws.aws_secret', 'roboshop/mysql', region='us-east-1') }}"
```

**Typical pattern** — store the fetched value in a variable and use it across tasks:

```yaml
- name: configure payment server
  hosts: payment
  vars:
    DB_PASSWORD: "{{ lookup('amazon.aws.aws_ssm', '/roboshop/mysql/password', region='us-east-1') }}"
  tasks:
  - name: deploy payment config
    template:
      src: payment.conf.j2
      dest: /etc/payment/config
```

**How it compares:**

| | Ansible Vault | AWS Secrets Manager / SSM |
|---|---|---|
| Secrets location | Encrypted in your repo | AWS managed, outside repo |
| Secret rotation | Manual — edit vault, redeploy | AWS handles rotation |
| Access control | Anyone with vault password | IAM policies per secret |
| Best for | Small teams, simple setups | Production, compliance requirements |

The EC2 instance running Ansible needs an IAM role with `ssm:GetParameter` or `secretsmanager:GetSecretValue` permission — no keys needed.

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| Configuration management | Managing server state through code, not manually |
| Push architecture | You push changes to servers (Ansible's model) |
| Pull architecture | Servers pull changes from a central server (Chef/Puppet) |
| Inventory | File listing which servers Ansible should manage |
| Ad-hoc command | One-off Ansible task run from the terminal |
| Playbook | YAML file with a list of tasks to run on target servers |
| Module | Pre-built unit of work in Ansible (`ping`, `dnf`, `copy`, etc.) |
| `vars` | Define variables at play or task level |
| `vars_files` | Load variables from an external YAML file |
| `vars_prompt` | Prompt user for variable input at runtime |
| `group_vars/` | Auto-loaded variables scoped to an inventory group |
| `-e` | Command-line variable — highest priority |
| `when:` | Run a task conditionally |
| `ansible_facts` | Auto-collected system info (OS, IP, arch, etc.) |
| `loop:` | Repeat a task over a list; current item is `{{ item }}` |
| `become: yes` | Run task with sudo / privilege escalation |
| `ansible.cfg` | Config file — sets inventory, log path, defaults so you don't repeat CLI flags |
| `.j2` template | Config file with `{{ variable }}` placeholders filled at deploy time |
| `template` module | Renders a `.j2` file and copies it to the target server |
| Filters | Built-in data manipulation functions (`split`, `upper`, `default`, etc.) |
| `shell` module | Run Linux command with full shell (pipes, redirections) |
| `command` module | Run Linux command without shell features — safer |
| `register` | Capture task output into a variable for later use |
| `ignore_errors: true` | Continue play even if this task fails |
| `.rc` | Return code from a registered task — 0 = success, non-zero = fail |
| Idempotency | Modules handle it automatically; `shell`/`command` — you handle manually |
| `tags:` | Label tasks so you can run/skip them selectively with `--tags` / `--skip-tags` |
| `--tags` / `--skip-tags` | CLI flags to run only tagged tasks or exclude them |
| Role | Standardised directory structure (`tasks/`, `vars/`, `handlers/`, `meta/`, etc.) for reusable playbook components |
| `roles:` | Key in a play that loads one or more roles |
| `meta/main.yaml` | Declares role dependencies — they run before the role itself |
| Handler | Task that runs only when notified by another task, and only once per play |
| `notify:` | Queues a handler by name when the task actually changes something |
| `handlers/main.yaml` | Where handler definitions live inside a role |
| `import_role` | Statically loads a role — tags cascade, `when:` applies to all tasks, no loops |
| `include_role` | Dynamically loads a role at runtime — supports loops, tags don't cascade |
| `tasks_from:` | Load a specific task file from a role instead of `main.yaml` |
| Ansible Vault | AES-256 encryption for secrets files so they can be committed to git safely |
| `ansible-vault create` | Create a new encrypted vault file |
| `ansible-vault edit` | Edit an encrypted file (decrypts in memory, re-encrypts on save) |
| `--ask-vault-pass` | Prompt for vault password at playbook runtime |
| `--vault-password-file` | Read vault password from a file (for CI/automation) |
| `aws_ssm` lookup | Fetch secrets from AWS SSM Parameter Store at playbook runtime |
| `aws_secret` lookup | Fetch secrets from AWS Secrets Manager at playbook runtime |

---


