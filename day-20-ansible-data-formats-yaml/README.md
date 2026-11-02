# Day 20 - Ansible Setup: Create the Server, Data Formats (XML, JSON, YAML) & Variables

> **In short:** Running an app needs two things: the **code** and the **configuration** of the server. Ansible does the configuration. We create servers with the **AWS CLI**, tell Ansible which machines to target in the **inventory**, and in Ansible **everything is a module** (or a collection of modules). Playbooks are written in **YAML**, so we compared the same student details written in **XML**, **JSON** and **YAML**: `[]` is a **list**, `{}` is a **map/object**. Ansible also has programming basics: variables, data types, conditions, loops, functions and error handling, starting with **variables**.

**More in this folder:** [Interview questions](interview-questions/README.md) · **Hands-on lab:** [Control node + managed nodes with SSH keys & ansible.cfg](../ansible-lab-control-node-managed-nodes/README.md)

## Table of Contents

1. [Code vs configuration](#code-vs-configuration)
2. [Create the Ansible server with AWS CLI](#create-the-ansible-server-with-aws-cli)
3. [Ansible: everything is a module](#ansible-everything-is-a-module)
4. [Inventory: the machines Ansible targets](#inventory-the-machines-ansible-targets)
5. [DTO and data formats](#dto-and-data-formats)
   - [List `[]` and map `{}`](#list--and-map-)
   - [Student details in XML](#student-details-in-xml)
   - [Student details in JSON](#student-details-in-json)
   - [Student details in YAML](#student-details-in-yaml)
   - [XML vs JSON vs YAML](#xml-vs-json-vs-yaml)
   - [Check your YAML: yamlchecker.com](#check-your-yaml-yamlcheckercom)
6. [Ansible installation and checking the inventory](#ansible-installation-and-checking-the-inventory)
7. [First playbooks](#first-playbooks)
   - [01-playbook.yaml: ping the Ansible server itself](#01-playbookyaml-ping-the-ansible-server-itself)
   - [02-playbook.yaml: print a message (debug)](#02-playbookyaml-print-a-message-debug)
   - [03-multi-play.yaml: many plays in one playbook](#03-multi-playyaml-many-plays-in-one-playbook)
   - [Run a playbook](#run-a-playbook)
8. [Create the frontend server (managed node)](#create-the-frontend-server-managed-node)
9. [04-nginx.yaml: install and start nginx on the frontend](#04-nginxyaml-install-and-start-nginx-on-the-frontend)
   - [How the `package` module decides: dnf or apt?](#how-the-package-module-decides-dnf-or-apt)
10. [Ansible as a programming language](#ansible-as-a-programming-language)
11. [Variables](#variables)
   - [1. Play-level variables](#1-play-level-variables-05-variablesyaml)
   - [2. Task-level variables](#2-task-level-variables-06-variablesyaml)
   - [3. Variables from a file](#3-variables-from-a-file-07-variablesyaml)
   - [4. Variables from a prompt](#4-variables-from-a-prompt-08-variablesyaml)
   - [5. Variables from the inventory](#5-variables-from-the-inventory-09-variablesyaml)
   - [6. Variables from the command line](#6-variables-from-the-command-line-10-variablesyaml)
   - [Variable preference](#variable-preference-11-vars-preferenceyaml)
12. [What I learned](#what-i-learned)
13. [Interview Questions](#interview-questions)

---

## Code vs configuration

**Definition:** To run an application we need **2 things**:

| # | Part | What it is | Who handles it |
|---|---|---|---|
| 1 | **Code** | the application itself (Node.js, Java, Python files) | developers, kept in Git |
| 2 | **Configuration** | everything on the server so the code can run: packages, users, folders, service files | **Ansible** (configuration management) |

```
plain server ──(configuration: Ansible)──► ready server ──(+ code)──► app running
```

## Create the Ansible server with AWS CLI

**Definition:** **`aws ec2 run-instances`** = AWS CLI command that **creates** (launches) an EC2 instance.

First we create the **Ansible server** (control node), where Ansible will be installed. We **attach the `roboshop-common` security group** to it (`--security-groups "roboshop-common"`).

```bash
aws ec2 run-instances \
    --image-id ami-0055977a6e828a4f4 \
    --instance-type t3.micro \
    --security-groups "roboshop-common" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-server}]' \
    --query 'Instances[0].InstanceId' \
    --output text
```

| Option | Means |
|---|---|
| `--image-id ami-...` | which **AMI** (OS image) to use |
| `--instance-type t3.micro` | size of the server (CPU/RAM) |
| `--security-groups "roboshop-common"` | attach the **`roboshop-common`** security group (firewall) **by name** |
| `--tag-specifications ... Value=ansible-server` | gives the server the **Name** tag `ansible-server` |
| `--query 'Instances[0].InstanceId'` | from the big JSON output, pick only the **instance ID** of the first instance |
| `--output text` | print it as plain text (no JSON) |

Output: just the ID, e.g. `i-0abc123...`. Easy to use in scripts.

> `Instances[0]` → `Instances` is a **list** `[]`, `[0]` = the **first** item. Same idea as lists in JSON/YAML below.

## Ansible: everything is a module

**Definition:** In Ansible, **everything is a module or a collection**. A **module** = one unit of work (like a Linux command). A **collection** = a package of modules.

| Linux | Ansible |
|---|---|
| `dnf install nginx` (a **command**) | `dnf` **module**: `name: nginx`, `state: installed` |
| `ping` | `ping` **module** |
| many commands | a **collection** of modules (e.g. `ansible.builtin`) |

## Inventory: the machines Ansible targets

**Definition:** **Inventory** = the list of **machines Ansible is targeting** (IPs or hostnames), usually in a file like `inventory.ini`.

```ini
; orphans: IPs at the top, not in any group
172.12.123.13
172.12.123.15
172.12.123.16

[database]
172.12.123.45
172.12.123.46
172.12.123.47

[backend]
172.12.123.50
172.12.123.51
172.12.123.56

[frontend]
172.31.20.30

; group of groups
[roboshop:children]
frontend
backend
database

[local]
localhost

[frontend:vars]
course=ansible
trainer=Sivakumar Reddy M
duration=10 hrs
```

| Part | Means |
|---|---|
| IPs at the top (no `[group]` above them) | **orphans**: we don't know which group they belong to, so they are written at the **top** (Ansible calls them `ungrouped`) |
| `[database]` | **group**: all DB server IPs go here |
| `[backend]` | **group**: all backend server IPs |
| `[frontend]` | **group**: frontend server IP |
| `[roboshop:children]` | **group of groups**: all RoboShop servers = `frontend` + `backend` + `database` |
| `[local]` | group with `localhost` (the Ansible server itself) |
| `[frontend:vars]` | **variables** for every server in the `frontend` group |
| `;` | comment in an `.ini` file |

```
                 roboshop  (group of groups)
        ┌───────────┼────────────┐
    frontend     backend      database
   172.31.20.30  .50 .51 .56  .45 .46 .47

  orphans (ungrouped): .13 .15 .16
```

**Target what you need:**

| Command | Runs on |
|---|---|
| `ansible database -i inventory.ini -m ping` | only the 3 DB servers |
| `ansible backend -i inventory.ini -m ping` | only the 3 backend servers |
| `ansible roboshop -i inventory.ini -m ping` | **all RoboShop servers** (frontend + backend + database) |
| `ansible ungrouped -i inventory.ini -m ping` | only the orphans |
| `ansible all -i inventory.ini -m ping` | **every** server in the file |

## DTO and data formats

**Definition:** **DTO (Data Transfer Object)** = an object used only to **carry data** between systems (e.g. app → API → server). XML, JSON and YAML are formats to write that data.

Same student details, written 3 ways.

### List `[]` and map `{}`

**Definition:**
- **List** = many **items in order**. JSON: `[ ]`. YAML: each item starts with `-`.
- **Map / object** = **key: value** pairs (named fields). JSON: `{ }`. YAML: `key: value` lines.

| | JSON | YAML |
|---|---|---|
| **List** | `["Linux", "Ansible"]` | `- Linux`<br>`- Ansible` |
| **Map / object** | `{"OS": "Linux", "CM": "ansible"}` | `OS: Linux`<br>`CM: ansible` |

A student is a **map** (name, dob, ...). A student has 2 addresses → `addresses` is a **list** of maps. `skills` is a **map**.

### Student details in XML

**Definition:** **XML** = data written with **open and close tags** `<tag>value</tag>`.

```xml
<Student>
    <firstName>Sivakumar</firstName>
    <lastName>M</lastName>
    <dob>01-01-01</dob>
    <addresses>
        <address type="temporary">
            <doorNo>89-899</doorNo>
            <streetName>Gandi Nagar</streetName>
            <city>Hyderabad</city>
        </address>
        <address type="permanent">
            <doorNo>89-899</doorNo>
            <streetName>Gandi Nagar</streetName>
            <city>Hyderabad</city>
        </address>
    </addresses>
</Student>
```

- Every value needs an **opening** and **closing** tag → long.
- `type="temporary"` is an **attribute** (extra info inside the tag).

### Student details in JSON

**Definition:** **JSON** = data written with **`{}` for maps**, **`[]` for lists**, keys in **double quotes**, items separated by **commas**.

```json
{
    "firstName": "Sivakumar Reddy",
    "lastName": "M",
    "dob": "01-01-01",
    "addresses": [
        {
            "type": "permanent",
            "doorNo": "1-566",
            "streetName": "Gandi Nagar",
            "city": "Hyderabad"
        },
        {
            "type": "temporary",
            "doorNo": "1-566",
            "streetName": "Gandi Nagar",
            "city": "Hyderabad"
        }
    ],
    "skills": {
        "OS": "Linux",
        "IDE": "VsCode",
        "CM": "ansible"
    }
}
```

| Part | Type |
|---|---|
| whole student `{ ... }` | map / object |
| `"addresses": [ {...}, {...} ]` | **list** of 2 maps |
| `"skills": { ... }` | map |

### Student details in YAML

**Definition:** **YAML** = data written as **`key: value`**, **indentation (spaces)** shows what is inside what, and **`-`** starts a list item. No brackets, no quotes needed.

```yaml
firstName: Sivakumar Reddy
lastName: M
dob: 01-01-01
addresses:            # - is for list
  - type: permanent
    doorNo: 1-566
    streetName: Gandi Nagar
    city: Hyderabad
  - type: temporary
    doorNo: 1-566
    streetName: Gandi Nagar
    city: Hyderabad
skills:               # map (no -)
  OS: Linux
  IDE: VsCode
  CM: ansible
```

**Many students = a list of students:** put `-` in front of each student:

```yaml
- firstName: Sivakumar Reddy      # student 1
  lastName: M
  skills:
    CM: ansible

- firstName: John                 # student 2
  lastName: Doe
  skills:
    CM: ansible
```

**YAML rules:**

| Rule | Example |
|---|---|
| `key: value` (space after `:`) | `city: Hyderabad` |
| indentation = **spaces only**, no tabs | 2 spaces inside `addresses:` |
| `-` = list item | `- type: permanent` |
| `#` = comment | `# - is for list` |
| same level = same indentation | `firstName`, `lastName`, `dob` all start at column 1 |

> ⚠️ **Watch out:**
> - No comma at the end of a line: `doorNo: 1-566,` keeps the comma **inside the value** (`"1-566,"`). Commas are for JSON, not YAML.
> - Use the **same key names** everywhere (`firstName` everywhere, not `first_name` in one place and `firstName` in another).

### XML vs JSON vs YAML

| | XML | JSON | YAML |
|---|---|---|---|
| Map / object | `<tag>...</tag>` | `{ }` | `key: value` + indentation |
| List | repeated tags | `[ ]` | `-` |
| Quotes | not needed | needed for keys and text | usually not needed |
| Comments | `<!-- -->` | ❌ none | `#` |
| Length | longest | medium | shortest |
| Used in | Maven `pom.xml`, old APIs | APIs, AWS CLI output | **Ansible**, Kubernetes, GitHub Actions |

### Check your YAML: yamlchecker.com

**Definition:** **[yamlchecker.com](https://yamlchecker.com)** = a website to **check the format / syntax** of YAML code. Paste the YAML → it shows if it's valid, or the line with the error (wrong indentation, tabs, missing space after `:`).

Check the YAML before running the playbook, so a small spacing mistake doesn't fail the run.

## Ansible installation and checking the inventory

**Definition:** **`ansible-inventory --graph`** = shows all machines in the inventory as a **tree of groups**, so we can check the inventory is correct before running anything.

**Steps:**

| # | Step | Command |
|---|---|---|
| 1 | Connect to the EC2 server (Ansible server) | `ssh ec2-user@<ansible-server-ip>` |
| 2 | Install Ansible | `sudo dnf install ansible -y` |
| 3 | Clone the code repo | `git clone https://github.com/Abhignadumpala/ansible-code.git` |
| 4 | Go into the repo | `cd ansible-code` |
| 5 | Check the inventory machines | `ansible-inventory -i inventory.ini --graph` |

```bash
sudo dnf install ansible -y
git clone https://github.com/Abhignadumpala/ansible-code.git
cd ansible-code
ansible-inventory -i inventory.ini --graph
```

**Expected output** (for the inventory above):

```
@all:
  |--@ungrouped:
  |  |--172.12.123.13
  |  |--172.12.123.15
  |  |--172.12.123.16
  |--@roboshop:
  |  |--@frontend:
  |  |  |--172.31.20.30
  |  |--@backend:
  |  |  |--172.12.123.50
  |  |  |--172.12.123.51
  |  |  |--172.12.123.56
  |  |--@database:
  |  |  |--172.12.123.45
  |  |  |--172.12.123.46
  |  |  |--172.12.123.47
  |--@local:
  |  |--localhost
```

| In the output | Means |
|---|---|
| `@` | a **group** (no `@` = a machine) |
| `@all` | every machine in the inventory |
| `@ungrouped` | the **orphans** (IPs at the top) |
| `@roboshop` → `@frontend`, `@backend`, `@database` | the **group of groups** and its child groups |
| `@local` → `localhost` | the Ansible server itself |

> `--graph` only **reads the inventory file**; it does not connect to the servers. To check the connection, use `-m ping`.

**Show only one group:** add the group name at the end:

| Command | Shows |
|---|---|
| `ansible-inventory -i inventory.ini --graph database` | `@database` and its 3 IPs |
| `ansible-inventory -i inventory.ini --graph backend` | `@backend` and its 3 IPs |
| `ansible-inventory -i inventory.ini --graph ungrouped` | the orphans |

## First playbooks

**Definition:** **Playbook** = a **YAML file** with a list of **plays**. Each play says **which machines** (`hosts`) and **what to do** (`tasks`). Each task runs **one module**.

```
playbook.yaml
 └── play        (name, hosts, connection)
      └── tasks
           ├── task 1  → module
           └── task 2  → module
```

### 01-playbook.yaml: ping the Ansible server itself

```yaml
- name: ping demo
  hosts: local # target ansible server itself
  connection: local # It will not expect credentials
  tasks: # tasks == modules == collections
  - name: ping localhost
    ping:
```

| Line | Means |
|---|---|
| `- name: ping demo` | start of a **play** (`-` = list item, a playbook is a list of plays); `name` = a label shown in the output |
| `hosts: local` | **which machines**: the `[local]` group from `inventory.ini` (= `localhost`, the Ansible server itself) |
| `connection: local` | run **directly** on this machine, **no SSH** → no username/password needed |
| `tasks:` | the **list of tasks** to run (each task = one module) |
| `- name: ping localhost` | first task, with a label |
| `ping:` | the **module**: checks Ansible works on the machine, answers `pong` |

### 02-playbook.yaml: print a message (debug)

```yaml
- name: demo on debug
  hosts: local
  connection: local
  tasks:
  - name: print hello world
    ansible.builtin.debug:
      msg: "Hello Ansible..!"
```

| Line | Means |
|---|---|
| `hosts: local` + `connection: local` | same as before: run on the Ansible server itself |
| `ansible.builtin.debug:` | the **`debug` module** = prints a message (like `echo` in shell) |
| `ansible.builtin.` | the **collection** the module comes from (`ansible.builtin` = built-in modules). `debug:` alone also works |
| `msg: "Hello Ansible..!"` | the message to print |

### 03-multi-play.yaml: many plays in one playbook

**Definition:** **Multi-play playbook** = one playbook file with **more than one play**. Each play has its own `name`, `hosts` and `tasks`, and the plays run **one after another, top to bottom**.

```yaml
- name: PLAY-1
  hosts: local
  connection: local
  tasks:
  - name: PLAY-1 and TASK-1
    debug:
      msg: "Hello from PLAY-1 TASK-1"

  - name: PLAY-1 and TASK-2
    debug:
      msg: "Hello from PLAY-1 TASK-2"

- name: PLAY-2
  hosts: local
  connection: local
  tasks:
  - name: PLAY-2 and TASK-1
    debug:
      msg: "Hello from PLAY-2 TASK-1"

  - name: PLAY-2 and TASK-2
    debug:
      msg: "Hello from PLAY-2 TASK-2"
```

**Structure:**

```
03-multi-play.yaml   (a list of plays)
 ├── - PLAY-1   (hosts: local)
 │     ├── - TASK-1  → debug
 │     └── - TASK-2  → debug
 └── - PLAY-2   (hosts: local)
       ├── - TASK-1  → debug
       └── - TASK-2  → debug
```

| Part | Means |
|---|---|
| `- name: PLAY-1` (no indentation) | **play 1** starts; the `-` at the left edge = a new item in the list of plays |
| `- name: PLAY-2` (no indentation) | **play 2** starts |
| `- name: PLAY-1 and TASK-1` (under `tasks:`) | a **task** inside the play (`-` = item in the list of tasks) |
| `debug:` + `msg:` | each task prints its message |

> The **indentation** tells play from task: `-` at the left edge = **play**, `-` under `tasks:` = **task**.

**Run order:** PLAY-1 TASK-1 → PLAY-1 TASK-2 → PLAY-2 TASK-1 → PLAY-2 TASK-2.

**Why use more than one play?** Each play can target **different machines**. Example for RoboShop:

```yaml
- name: set up database
  hosts: database          # play 1 → DB servers
  ...
- name: set up backend
  hosts: backend           # play 2 → backend servers
  ...
- name: set up frontend
  hosts: frontend          # play 3 → frontend server
  ...
```

One file, one command → DB first, then backend, then frontend, in the **right order**.

**Expected output** of `ansible-playbook -i inventory.ini 03-multi-play.yaml` (short):

```
PLAY [PLAY-1] ******************************
TASK [Gathering Facts]        ok: [localhost]
TASK [PLAY-1 and TASK-1]      ok: [localhost]  msg: Hello from PLAY-1 TASK-1
TASK [PLAY-1 and TASK-2]      ok: [localhost]  msg: Hello from PLAY-1 TASK-2

PLAY [PLAY-2] ******************************
TASK [Gathering Facts]        ok: [localhost]
TASK [PLAY-2 and TASK-1]      ok: [localhost]  msg: Hello from PLAY-2 TASK-1
TASK [PLAY-2 and TASK-2]      ok: [localhost]  msg: Hello from PLAY-2 TASK-2

PLAY RECAP *********************************
localhost : ok=6  changed=0  unreachable=0  failed=0
```

`ok=6` = 2 plays × (1 Gathering Facts + 2 tasks). **Gathering Facts runs once per play.**

### Run a playbook

```bash
ansible-playbook -i inventory.ini 01-playbook.yaml
ansible-playbook -i inventory.ini 02-playbook.yaml
```

| Part | Means |
|---|---|
| `ansible-playbook` | run a **playbook** (`ansible` = ad-hoc command) |
| `-i inventory.ini` | inventory file, so `hosts: local` can be found |
| `02-playbook.yaml` | the playbook file |

**Expected output** of `02-playbook.yaml`:

```
PLAY [demo on debug] ***********************************************

TASK [Gathering Facts] *********************************************
ok: [localhost]

TASK [print hello world] *******************************************
ok: [localhost] =>
  msg: Hello Ansible..!

PLAY RECAP *********************************************************
localhost : ok=2  changed=0  unreachable=0  failed=0  skipped=0
```

| Output part | Means |
|---|---|
| `PLAY [demo on debug]` | the play's `name` |
| `TASK [Gathering Facts]` | Ansible first collects info about the machine (runs automatically) |
| `TASK [print hello world]` | our task's `name` |
| `ok` | task ran, nothing changed |
| `PLAY RECAP` | summary: how many tasks were `ok`, `changed`, `failed` per machine |

> Always give every play and task a clear `name`: it's what you read in the output.

**Examples:**

`01-playbook.yaml`: `ping localhost` → `ok`, recap `ok=2 changed=0 failed=0`

`git pull` to get the new `02-playbook.yaml`, then run it: the `debug` module prints `msg`

## Create the frontend server (managed node)

**Definition:** **Managed node** = a server that Ansible **connects to and configures**. Here: the RoboShop **frontend** server.

Now that Ansible works on the Ansible server, we create a **second** server, `roboshop-frontend`, for Ansible to configure. Same command, only the **Name** tag changes:

```bash
aws ec2 run-instances \
    --image-id ami-0055977a6e828a4f4 \
    --instance-type t3.micro \
    --security-groups "roboshop-common" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=roboshop-frontend}]' \
    --query 'Instances[0].InstanceId' \
    --output text
```

| | Ansible server | Frontend server |
|---|---|---|
| Name tag | `ansible-server` | `roboshop-frontend` |
| Role | **control node**: Ansible installed, runs playbooks | **managed node**: Ansible connects to it over SSH |
| Ansible installed? | ✅ yes | ❌ no (agentless) |

**Steps:**

| # | Step | Command / where |
|---|---|---|
| 1 | Create the frontend server | the `aws ec2 run-instances` command above |
| 2 | Get its **private IP** | AWS console, or `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text` |
| 3 | Put the IP under `[frontend]` in `inventory.ini` | `[frontend]`<br>`<frontend-private-ip>` |
| 4 | Check the connection from the Ansible server | `ansible frontend -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> -m ping` → `pong` |

```
ansible-server ── ssh (port 22, private IP) ──► roboshop-frontend
 (control node)                                   (managed node)
```

> **Attach `roboshop-common` SG to both servers** (Ansible server and frontend). Port **22** must be allowed in it, so the Ansible server can SSH to the frontend.

## 04-nginx.yaml: install and start nginx on the frontend

**Definition:** **`04-nginx.yaml`** = a playbook that **installs** nginx and **starts** it on every server in the `[frontend]` group.

**Step 1: add the frontend IP to the inventory.** After the frontend server is created, put its **private IP** under `[frontend]` in `inventory.ini`:

```ini
[frontend]
<frontend-private-ip>      ; e.g. 172.31.20.30
```

**Step 2: write the playbook.**

```yaml
- name: install and start nginx
  hosts: frontend
  become: yes # providing sudo access
  tasks:
  - name: install nginx
    package:
      name: nginx
      state: present

  - name: start nginx
    service:
      name: nginx
      state: started
      enabled: yes
```

| Line | Means |
|---|---|
| `- name: install and start nginx` | the play's name |
| `hosts: frontend` | run on the `[frontend]` group from `inventory.ini` (the frontend server's IP) |
| *(no `connection: local`)* | so Ansible connects over **SSH** to the frontend server |
| `become: yes` | run tasks as **root** (sudo); installing packages needs root. Same as `-b` in ad-hoc |
| `package:` | module that installs a package with the server's package manager (`dnf` on RHEL, `apt` on Ubuntu), so it works on **any distro** |
| `name: nginx` + `state: present` | nginx should be **installed** (`present` = `installed`) |
| `service:` | module that manages services (like `systemctl`) |
| `state: started` | nginx should be **running** (`systemctl start nginx`) |
| `enabled: yes` | nginx should **start on boot** (`systemctl enable nginx`) |

### How the `package` module decides: dnf or apt?

**Definition:** **`package` module** = a **generic** install module. It **checks the target machine's OS** and then uses the **right package manager** for that OS.

We only write `package: name=nginx`. Ansible does the rest **in the background**:

```
package: name=nginx state=present
        │
        ▼
Ansible checks the target machine's OS   (Gathering Facts)
        │
        ├── RHEL / Amazon Linux / CentOS ──► dnf (or yum) install nginx
        ├── Ubuntu / Debian              ──► apt-get install nginx
        └── other OS                     ──► its own package manager
```

| Target OS | What Ansible uses in the background |
|---|---|
| RHEL 9, Amazon Linux, CentOS | `dnf` (older: `yum`) |
| Ubuntu, Debian | `apt` / `apt-get` |
| SUSE | `zypper` |

**All the checks happen in the background:**
- which OS is it? → pick `dnf` / `apt`
- is nginx **already installed**? → yes: do nothing (`ok`); no: install it (`changed`)

**That is Ansible's simplicity:** we only say **what** we want (`nginx` should be present). **How** to do it on each OS is Ansible's job.

| Shell script | Ansible `package` |
|---|---|
| we write `if` checks for the OS: `dnf` on RHEL, `apt` on Ubuntu | one line, works on any OS |
| we write the "already installed?" check | Ansible checks it |
| works only on **one** OS family (homogeneous, Day 19) | works on **any** OS (heterogeneous) |

**Same as Linux commands:**

| Task | Linux command | Ansible |
|---|---|---|
| install nginx | `sudo dnf install nginx -y` | `package: name=nginx state=present` |
| start nginx | `sudo systemctl start nginx` | `service: state=started` |
| start on boot | `sudo systemctl enable nginx` | `service: enabled=yes` |

**Step 3: run it from the Ansible server.** The playbook has no username/password, so we pass them with `-e`:

```bash
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> 04-nginx.yaml
```

`git pull` gets the new `04-nginx.yaml`, then the run starts: `Gathering Facts` → `ok: [172.31.20.30]` (the frontend IP from the inventory), then `TASK [install nginx]`.

**Expected output:**

| Run | Result | Why |
|---|---|---|
| 1st run | `changed` for both tasks | nginx installed and started |
| 2nd run | `ok`, `changed=0` | already installed and running → **idempotent**, nothing to do |

**Step 4: start nginx with the service module (ad-hoc)**

**Definition:** **`service` module** = starts, stops, restarts or enables a service on the target machine (like `systemctl`).

The playbook already starts nginx. To start it as a **one-time ad-hoc command** (same thing, from the Ansible server):

```bash
ansible frontend -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> -b -m service -a "name=nginx state=started enabled=yes"
```

| Part | Means |
|---|---|
| `frontend` | target the `[frontend]` group |
| `-b` | become root (starting services needs root) |
| `-m service` | use the **service** module |
| `name=nginx` | which service |
| `state=started` | start it now (`systemctl start nginx`) |
| `enabled=yes` | start on boot (`systemctl enable nginx`) |

Output: `CHANGED` if nginx was stopped, `SUCCESS` (`changed: false`) if it was already running.

**Step 5: allow port 80 in the `roboshop-common` security group**

**Definition:** **Inbound rule** = a security group rule that **allows traffic into** the server on a port. nginx listens on **port 80 (HTTP)**, so the browser needs port 80 open.

AWS console: **EC2 → Security Groups → `roboshop-common` → Inbound rules → Edit inbound rules → Add rule**

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | **80** | `0.0.0.0/0` (anywhere) |

Or with AWS CLI:

```bash
aws ec2 authorize-security-group-ingress \
    --group-name roboshop-common \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

| Option | Means |
|---|---|
| `authorize-security-group-ingress` | **add an inbound** rule |
| `--group-name roboshop-common` | to this security group |
| `--protocol tcp --port 80` | allow TCP port 80 (HTTP) |
| `--cidr 0.0.0.0/0` | from **anywhere** on the internet |

**Step 6: check nginx from the web**

| Where | Command | Expected |
|---|---|---|
| Browser | `http://<frontend-public-ip>` | **nginx welcome page** |
| Terminal (your laptop) | `curl -I http://<frontend-public-ip>` | `HTTP/1.1 200 OK`, `Server: nginx` |
| Ansible server (private IP) | `curl -I http://<frontend-private-ip>` | `HTTP/1.1 200 OK` |

**If the page doesn't open:**

| Symptom | Cause | Fix |
|---|---|---|
| browser keeps loading → **timed out** | port 80 **not allowed** in the SG | add the port 80 inbound rule (Step 5) |
| **connection refused** | nginx **not running** | run Step 4 (`state=started`) |
| page opens but not nginx | wrong IP | use the **frontend's public IP**, not the Ansible server's |

```
inventory.ini [frontend] ──► ansible-playbook 04-nginx.yaml ──ssh (22)──► frontend: nginx installed + started
                                                                              ▲
browser ─────────────── http (port 80, allowed in roboshop-common) ──────────┘
```

## Ansible as a programming language

**Definition:** Ansible playbooks also have the **basic building blocks of a programming language**, like shell scripting did:

| # | Topic | Meaning |
|---|---|---|
| 1 | **Variables** | store a value with a name, use it many times |
| 2 | **Data types** | kind of value: string, number, boolean, list, map |
| 3 | **Conditions** | run a task only **when** something is true |
| 4 | **Loops** | run one task for **many items** |
| 5 | **Functions** | reusable logic (in Ansible: filters/modules) |
| 6 | **Error handling** | what to do when a task fails |

## Variables

**Definition:** **Variable** = a **name** that stores a **value**, so we write the value once and use it in many places. In Ansible we use it with **`{{ name }}`**.

**6 places to define variables in Ansible:**

| # | Where | Keyword | File |
|---|---|---|---|
| 1 | in the **play** | `vars:` | `05-variables.yaml` |
| 2 | in the **task** | `vars:` under a task | `06-variables.yaml` |
| 3 | in a **separate file** | `vars_files:` | `07-variables.yaml` + `course.yaml` |
| 4 | **prompt** (ask the user at run time) | `vars_prompt:` | `08-variables.yaml` |
| 5 | in the **inventory** | `[group:vars]` | `09-variables.yaml` + `inventory.ini` |
| 6 | from the **command line** | `-e "name=value"` | `10-variables.yaml` |

### 1. Play-level variables (05-variables.yaml)

**Definition:** **Play-level variables** = variables under `vars:` in the **play**; every task in that play can use them.

```yaml
- name: demo on variables
  hosts: local
  connection: local
  vars:
    course: "ansible"
    trainer: Sivakumar Reddy M
    duration: 10 hrs
  tasks:
  - name: print the vars
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

| Part | Means |
|---|---|
| `vars:` | define variables (a **map**: `name: value`) |
| `{{ course }}` | use the variable's value |
| `debug` + `msg` | print the message |

Output: `Hi, I am learning ansible from Sivakumar Reddy M, duration is 10 hrs`

| Shell | Ansible |
|---|---|
| `COURSE="ansible"` | `course: ansible` under `vars:` |
| `echo "$COURSE"` | `msg: "{{ course }}"` |

### 2. Task-level variables (06-variables.yaml)

**Definition:** **Task-level variables** = variables under `vars:` **inside one task**; they work only for that task and **win over** play-level variables with the same name.

```yaml
- name: demo on task level variables
  hosts: local
  connection: local
  vars:
    property: "100acres"          # play level
  tasks:
  - name: print the vars
    debug:
      msg: "Hi, I got {{ property }} from my father"

  - name: print the vars again
    vars:
      property: "10acres"         # task level
    debug:
      msg: "Hi, I got {{ property }} from my father"
```

| Task | `property` used | Why |
|---|---|---|
| print the vars | `100acres` | no task variable → uses the **play** variable |
| print the vars again | `10acres` | task has its own `property` → **task level has priority** |

### 3. Variables from a file (07-variables.yaml)

**Definition:** **`vars_files`** = load variables from a **separate YAML file**. Ansible reads the file, loads the data, and the playbook can use it.

`course.yaml` (only variables, `key: value`):

```yaml
course: ansible from file
trainer: Sivakumar Reddy M
duration: 10 hrs
```

`07-variables.yaml`:

```yaml
- name: variables from files
  hosts: local
  connection: local
  vars_files:
  - course.yaml                   # list of files to load
  tasks:
  - name: print course information
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

**Why a separate file?**
- keep the variables **away from the code**: change values without touching the playbook
- many playbooks can use the **same** variables file

### 4. Variables from a prompt (08-variables.yaml)

**Definition:** **`vars_prompt`** = Ansible **asks the user** to type the value when the playbook runs. Used for **dynamic** or **confidential** values (username, password) that we don't want to keep in files.

```yaml
- name: variables from prompt
  hosts: local
  connection: local
  vars_prompt:
  - name: username
    prompt: Please enter your username
    private: false                # shows what you type

  - name: password
    prompt: Please enter your password
    private: true                 # hides what you type
  tasks:
  - name: print the confidential information
    debug:
      msg: "username is: {{ username }}, password is {{ password }}"
```

| Key | Means |
|---|---|
| `name: username` | the variable name |
| `prompt:` | the question shown on screen |
| `private: false` | you **see** what you type (fine for a username) |
| `private: true` | what you type is **hidden** on screen (for a password) |

> ⚠️ Here we **print** the password only **for testing**, to see the variable works. In real projects **never print confidential information**.

### 5. Variables from the inventory (09-variables.yaml)

**Definition:** **Inventory variables** = variables written in `inventory.ini` under **`[group:vars]`**; every server in that group gets them. Ansible loads them automatically.

`inventory.ini`:

```ini
[frontend]
172.31.20.30

[frontend:vars]
course=ansible
trainer=Sivakumar Reddy M
duration=10 hrs
```

`09-variables.yaml` (no `vars:` in the playbook):

```yaml
- name: variables from inventory
  hosts: frontend
  tasks:
  - name: print the variables
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

```bash
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> 09-variables.yaml
```

- `hosts: frontend` → runs on the frontend server (over SSH, so user/password are needed)
- the values come from `[frontend:vars]`

### 6. Variables from the command line (10-variables.yaml)

**Definition:** **Extra vars (`-e`)** = pass a variable **when running** the playbook: `-e "name=value"`.

```yaml
- name: print greeting
  hosts: local
  connection: local
  tasks:
  - name: print greeting
    debug:
      msg: "Hello {{ person_name }}"
```

`person_name` is **not defined anywhere**:

| Run | Result |
|---|---|
| `ansible-playbook -i inventory.ini 10-variables.yaml` | ❌ `FAILED! ... 'person_name' is undefined` |
| `ansible-playbook -i inventory.ini 10-variables.yaml -e "person_name=Sivakumar"` | ✅ `Hello Sivakumar` |

> **Undefined variable = error.** Every `{{ variable }}` must be defined somewhere (play, task, file, prompt, inventory or `-e`).

### Variable preference (11-vars-preference.yaml)

**Definition:** **Variable preference (precedence)** = when the **same variable name** is defined in many places, Ansible uses a fixed **order** to decide which value wins.

`11-vars-preference.yaml`: `course` is defined in **every** place:

```yaml
- name: variables from files
  hosts: local
  connection: local
  vars:
    course: "ansible from PLAY"
  vars_prompt:
  - name: course
    prompt: Please enter your course
    private: false
  vars_files:
  - course.yaml                   # course: ansible from file
  tasks:
  - name: print course information
    vars:
      course: "ansible from task"
    debug:
      msg: "Hi, I am learning {{ course }}"
```

+ `inventory.ini`: `[local:vars]` → `course=ansible from inventory`

**Preference order (1 = highest, wins):**

| # | Where | Example value |
|---|---|---|
| 1 | **Command line** `-e` | `ansible from cmd` |
| 2 | **Task** `vars:` | `ansible from task` |
| 3 | **File** `vars_files:` | `ansible from file` |
| 4 | **Prompt** `vars_prompt:` | `ansible from prompt` |
| 5 | **Play** `vars:` | `ansible from PLAY` |
| 6 | **Inventory** `[group:vars]` | `ansible from inventory` |

```
CMD  >  task  >  file  >  prompt  >  PLAY  >  inventory
(wins)                                       (lowest)
```

**How to test it (remove the winner each time, the next one wins).** Steps 1–3 were shown in class; 4–6 continue the same way:

| Step | What we did | Output |
|---|---|---|
| 1 | run with `-e "course='ansible from cmd'"` | `ansible from cmd` → **CMD wins** (1st), prompt is skipped |
| 2 | run without `-e`, type `ansible from prompt` | `ansible from task` → **task wins** (2nd) over prompt |
| 3 | comment out the task `vars:` | `ansible from file` → **file** (3rd) |
| 4 | comment out `course` in `course.yaml` | `ansible from prompt` → **prompt** (4th) |
| 5 | remove the prompt | `ansible from PLAY` → **play** (5th) |
| 6 | remove play `vars:` | `ansible from inventory` → **inventory** (6th) |

> **Command line is always 1st preference**, in almost any programming language or tool: what you pass at run time overrides everything.

#### ⚠️ Spaces in `-e` values need inner quotes

| Command | Output | Why |
|---|---|---|
| `-e "course=ansible from cmd"` | `Hi, I am learning ansible` ❌ | `-e "key=value"` splits on **spaces** → `course=ansible` only |
| `-e "course='ansible from cmd'"` | `Hi, I am learning ansible from cmd` ✅ | inner `'...'` keeps the spaces in one value |

## What I learned

### Class summary

- An app needs **code** + **configuration**; Ansible does the configuration.
- Created the **Ansible server** with `aws ec2 run-instances`, then (after the first playbooks) the **frontend server** with the same command and Name tag `roboshop-frontend`; `--query 'Instances[0].InstanceId' --output text` prints only the instance ID.
- In Ansible **everything is a module** or a collection of modules.
- **Inventory** = the machines Ansible targets, grouped (`[frontend]`, `[roboshop:children]`).
- **DTO** = data carried between systems; written in XML, JSON or YAML.
- **`[]` = list**, **`{}` = map/object**. In YAML: `-` = list item, `key: value` = map, indentation = nesting.
- Wrote the same student in XML (tags), JSON (brackets) and YAML (indentation): YAML is the shortest.
- Check YAML at **yamlchecker.com**.
- Ansible server: `sudo dnf install ansible -y` → `git clone` the code → `ansible-inventory -i inventory.ini --graph` shows `@all`, `@ungrouped`, `@roboshop`... as a tree.
- First playbooks: a play = `name` + `hosts` + `tasks`; `hosts: local` + `connection: local` = run on the Ansible server itself, no SSH. `ping` module → pong, `debug` module → prints `msg`. Run with `ansible-playbook -i inventory.ini <file>.yaml`.
- Added the frontend IP under `[frontend]` and ran `04-nginx.yaml`: `become: yes` (root), `package` (install, any distro), `service` (`started` + `enabled`).
- Ansible has variables, data types, conditions, loops, functions, error handling. **Variables** can be defined in 6 places: play `vars`, task `vars`, `vars_files`, `vars_prompt` (`private: true` hides input), inventory `[group:vars]`, and `-e` on the command line. Undefined variable = error.
- **Preference** when the same name is everywhere: **CMD > task > file > prompt > PLAY > inventory**. Use inner quotes for spaces: `-e "course='ansible from cmd'"`.

```
AWS CLI ──► server created ──► inventory (which machines)
                                   │
                         playbook (YAML: lists -, maps key: value)
                                   │
                         modules + variables {{ }} ──► configured server
```

## Interview Questions

See [interview-questions/README.md](interview-questions/README.md).
