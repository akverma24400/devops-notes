# Day 20 - Ansible Basics: How It Works, Inventory, Modules, YAML, Playbooks & Variables

> **In short:** Ansible configures servers for you. You write **what** you want in a **playbook** (a YAML file), list your servers in an **inventory**, and Ansible logs in to each server over **SSH** and does the work with **modules**. Nothing needs to be installed on the servers it manages.

**More in this folder:** [Interview questions](interview-questions/README.md) · **Hands-on lab:** [Control node + managed nodes with SSH keys & ansible.cfg](../ansible-lab-control-node-managed-nodes/README.md)

## Table of Contents

1. [What is Ansible?](#1-what-is-ansible)
2. [How Ansible works](#2-how-ansible-works)
3. [Step 1: Create the Ansible server](#3-step-1-create-the-ansible-server)
4. [Step 2: Install Ansible](#4-step-2-install-ansible)
5. [Inventory: which servers](#5-inventory-which-servers)
6. [Modules: what to do](#6-modules-what-to-do)
7. [YAML: the language of playbooks](#7-yaml-the-language-of-playbooks)
8. [Playbooks](#8-playbooks)
9. [Step 3: Create the frontend server](#9-step-3-create-the-frontend-server)
10. [Step 4: Install nginx with a playbook](#10-step-4-install-nginx-with-a-playbook)
11. [Variables](#11-variables)
12. [What I learned](#12-what-i-learned)
13. [Interview Questions](#interview-questions)

---

## 1. What is Ansible?

**Definition:** **Ansible** = a **configuration management** tool. It sets up servers automatically: installs packages, creates users and folders, starts services.

To run any application you need **2 things**:

| # | Part | What it is | Who handles it |
|---|---|---|---|
| 1 | **Code** | the application (Java, Node.js, Python files) | developers, kept in Git |
| 2 | **Configuration** | everything on the server so the code can run: packages, users, folders, services | **Ansible** |

```
plain server ──(Ansible configures it)──► ready server ──(+ code)──► app running
```

**Why not just a shell script?** A script works on one server and one OS, and breaks if you run it twice. Ansible works on **many servers at once**, on **any Linux OS**, and is **safe to run again and again**.

## 2. How Ansible works

**Definition:**
- **Control node** = the server where Ansible is **installed**. You run all commands from here.
- **Managed node** = a server Ansible **configures**. It only needs **SSH + Python**, no Ansible. That's why Ansible is called **agentless**.

![How Ansible works: control node connects over SSH to managed nodes](images/01-how-ansible-works.svg)

| | Control node | Managed node |
|---|---|---|
| Ansible installed? | ✅ yes | ❌ no |
| What it has | inventory, playbooks, SSH key/password | SSH server + Python |
| Role | sends the work | receives and runs the work |
| Connection | starts SSH (port 22) | accepts SSH (port 22) |

> In one line: **Ansible = SSH + Python + a list of tasks.**

## 3. Step 1: Create the Ansible server

**Definition:** **`aws ec2 run-instances`** = the AWS CLI command that **creates** an EC2 server.

We create the control node and attach the **`roboshop-common`** security group, which allows port 22 (SSH).

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
| `--image-id` | which OS image (AMI) to use |
| `--instance-type t3.micro` | size of the server (CPU/RAM) |
| `--security-groups "roboshop-common"` | the firewall to attach, by name |
| `--tag-specifications ... Value=ansible-server` | gives the server the name `ansible-server` |
| `--query 'Instances[0].InstanceId'` | from the big output, print only the **instance ID** |
| `--output text` | print it as plain text |

**Expected:** only the ID, e.g. `i-0abc123...`

## 4. Step 2: Install Ansible

Log in to the Ansible server:

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Install Ansible:

```bash
sudo dnf install ansible -y
```

Check it:

```bash
ansible --version
```

**Expected:** `ansible [core 2.x.x]`

Get the class code (inventory + playbooks):

```bash
git clone https://github.com/Abhignadumpala/ansible-code.git
```

```bash
cd ansible-code
```

## 5. Inventory: which servers

**Definition:** **Inventory** = a file that lists the **servers Ansible manages**, put into **groups**. Usually called `inventory.ini`.

![Inventory: servers in groups, and a group of groups](images/02-inventory.svg)

```ini
[frontend]
172.31.20.30

[backend]
172.31.20.50
172.31.20.51

[database]
172.31.20.45

[roboshop:children]
frontend
backend
database

[local]
localhost
```

| Part | Means |
|---|---|
| `[frontend]` | a **group** called frontend, with its server IPs below |
| `[roboshop:children]` | a **group of groups**: roboshop = frontend + backend + database |
| `[local]` + `localhost` | the Ansible server itself |
| `;` | a comment |

**Check the inventory:**

```bash
ansible-inventory -i inventory.ini --graph
```

**What it does:** shows all groups and servers as a tree. It only **reads the file**; it doesn't connect to anything.

**Expected:**

```
@all:
  |--@ungrouped:
  |--@roboshop:
  |  |--@frontend:
  |  |  |--172.31.20.30
  |  |--@backend:
  |  |  |--172.31.20.50
  |  |  |--172.31.20.51
  |  |--@database:
  |  |  |--172.31.20.45
  |--@local:
  |  |--localhost
```

`@` = a group. `@all` = every server. `@ungrouped` = servers written at the top of the file without a group (none here).

**Target one group:** put the group name in the command.

| Command | Runs on |
|---|---|
| `ansible frontend -i inventory.ini -m ping` | only the frontend server |
| `ansible roboshop -i inventory.ini -m ping` | all RoboShop servers |
| `ansible all -i inventory.ini -m ping` | every server in the file |

## 6. Modules: what to do

**Definition:** **Module** = a small ready-made program that does **one job** on a server: install a package, start a service, copy a file. In Ansible, **everything is done by a module**. A **collection** is a package of modules (for example `ansible.builtin`).

![Module: you write what you want, Ansible decides how, and reports changed or ok](images/03-module.svg)

| Linux command | Ansible module |
|---|---|
| `sudo dnf install nginx -y` | `package: name=nginx state=present` |
| `sudo systemctl start nginx` | `service: name=nginx state=started` |
| `sudo systemctl enable nginx` | `service: name=nginx enabled=yes` |
| `echo "hello"` | `debug: msg="hello"` |

**You say WHAT, Ansible works out HOW.** The `package` module checks the OS first, then uses `dnf` on Amazon Linux or `apt` on Ubuntu. It also checks whether nginx is already installed:

| Result | Meaning |
|---|---|
| `changed` (yellow) | Ansible had to do something (nginx was installed now) |
| `ok` (green) | already correct, nothing to do |
| `failed` (red) | something went wrong |

**Idempotent** = running the same thing many times gives the same result. Run it twice and the second run just says `ok`.

**Run one module from the command line (ad-hoc):**

```bash
ansible local -i inventory.ini -m ping
```

**What it does:** runs the `ping` module on the `local` group. `-m` = which module. `ping` doesn't send a network ping; it checks that Ansible can log in and run Python.

**Expected:** `localhost | SUCCESS => { "ping": "pong" }`

## 7. YAML: the language of playbooks

**Definition:** **YAML** = a simple text format for data. It uses **`key: value`** pairs, **indentation (spaces)** to show what's inside what, and **`-`** to start a list item. Playbooks are written in YAML.

**Only 2 shapes to know:**

| Shape | Meaning | YAML | JSON (same data) |
|---|---|---|---|
| **Map** | named fields | `OS: Linux`<br>`CM: ansible` | `{"OS": "Linux", "CM": "ansible"}` |
| **List** | items in order | `- Linux`<br>`- Ansible` | `["Linux", "Ansible"]` |

**Example: a student in YAML**

```yaml
firstName: Sivakumar          # map: key: value
lastName: M
addresses:                    # list of 2 addresses (each starts with -)
  - type: permanent
    city: Hyderabad
  - type: temporary
    city: Hyderabad
skills:                       # map inside a map (no -)
  OS: Linux
  CM: ansible
```

**The same student in JSON**, for comparison:

```json
{
  "firstName": "Sivakumar",
  "lastName": "M",
  "addresses": [
    { "type": "permanent", "city": "Hyderabad" },
    { "type": "temporary", "city": "Hyderabad" }
  ],
  "skills": { "OS": "Linux", "CM": "ansible" }
}
```

| | XML | JSON | YAML |
|---|---|---|---|
| Looks like | `<city>Hyderabad</city>` | `{"city": "Hyderabad"}` | `city: Hyderabad` |
| Comments | `<!-- -->` | ❌ none | `#` |
| Length | longest | medium | **shortest** |
| Used in | Maven `pom.xml` | APIs, AWS CLI output | **Ansible**, Kubernetes, GitHub Actions |

**YAML rules:**

| Rule | Example |
|---|---|
| space after `:` | `city: Hyderabad` ✅, `city:Hyderabad` ❌ |
| indent with **spaces only**, never tabs | 2 spaces |
| same level = same indentation | `firstName` and `lastName` start in the same column |
| `-` = list item | `- type: permanent` |
| `#` = comment | `# list of addresses` |
| no commas at the end of lines | `city: Hyderabad,` makes the comma part of the value |

> **Tip:** paste your YAML into **[yamlchecker.com](https://yamlchecker.com)** before running it. It shows the line with a wrong space or tab.

## 8. Playbooks

**Definition:** **Playbook** = a YAML file that tells Ansible **which servers** to work on and **what tasks** to run on them, in order.

![Playbook structure: a playbook has plays, a play has tasks, each task uses one module](images/04-playbook-structure.svg)

| Word | Meaning |
|---|---|
| **Playbook** | the `.yaml` file |
| **Play** | one block in it: `name` + `hosts` (which servers) + `tasks` |
| **Task** | one step, with a `name` |
| **Module** | what the task actually does (`package`, `service`, `debug`, ...) |

### 8.1 First playbook: ping the Ansible server itself

`01-playbook.yaml`:

```yaml
- name: ping demo
  hosts: local
  connection: local
  tasks:
  - name: ping localhost
    ping:
```

| Line | Means |
|---|---|
| `- name: ping demo` | start of a **play**, with a label shown in the output |
| `hosts: local` | run on the `[local]` group (the Ansible server itself) |
| `connection: local` | run directly here, **no SSH**, so no username/password needed |
| `tasks:` | the list of tasks |
| `ping:` | the module |

Run it:

```bash
ansible-playbook -i inventory.ini 01-playbook.yaml
```

**What it does:** `ansible-playbook` runs a playbook file. `-i` tells it where the inventory is.

### 8.2 Print a message: the `debug` module

`02-playbook.yaml`:

```yaml
- name: demo on debug
  hosts: local
  connection: local
  tasks:
  - name: print hello world
    ansible.builtin.debug:
      msg: "Hello Ansible..!"
```

`debug` prints a message, like `echo`. `ansible.builtin.` is the collection it comes from; writing just `debug:` also works.

```bash
ansible-playbook -i inventory.ini 02-playbook.yaml
```

**Expected:**

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

| Output | Means |
|---|---|
| `PLAY [...]` | the play's `name` |
| `TASK [Gathering Facts]` | Ansible first collects info about the server (OS, IP...). Runs automatically. |
| `TASK [print hello world]` | your task's `name` |
| `PLAY RECAP` | the summary per server: how many `ok`, `changed`, `failed` |

> Always give every play and task a clear `name`. It's what you read in the output.

### 8.3 Many plays in one playbook

**Definition:** a playbook can have **more than one play**. They run **top to bottom**, and each play can target **different servers**.

`03-multi-play.yaml`:

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
```

**Run order:** PLAY-1 TASK-1 → PLAY-1 TASK-2 → PLAY-2 TASK-1.

> `-` at the **left edge** = a new **play**. `-` under `tasks:` = a new **task**.

**Real use:** set up the servers in the right order with one command:

```yaml
- name: set up database
  hosts: database
  ...
- name: set up backend
  hosts: backend
  ...
- name: set up frontend
  hosts: frontend
  ...
```

## 9. Step 3: Create the frontend server

**Definition:** the **frontend** server is our first **managed node**: a server Ansible connects to over SSH and configures.

Same command as Step 1, only the name changes:

```bash
aws ec2 run-instances \
    --image-id ami-0055977a6e828a4f4 \
    --instance-type t3.micro \
    --security-groups "roboshop-common" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=roboshop-frontend}]' \
    --query 'Instances[0].InstanceId' \
    --output text
```

Get its **private IP** (the Ansible server talks to it inside AWS):

```bash
aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text
```

Put the IP under `[frontend]` in `inventory.ini`:

```ini
[frontend]
172.31.20.30
```

Check that Ansible can log in to it:

```bash
ansible frontend -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> -m ping
```

**What it does:** Ansible connects to the frontend over **SSH (port 22)** using the username and password passed with `-e`.

**Expected:** `172.31.20.30 | SUCCESS => { "ping": "pong" }`

```
ansible-server ── SSH (port 22, private IP) ──► roboshop-frontend
 (control node)                                  (managed node)
```

## 10. Step 4: Install nginx with a playbook

**Definition:** **`04-nginx.yaml`** = a playbook that **installs** nginx and **starts** it on every server in the `[frontend]` group.

```yaml
- name: install and start nginx
  hosts: frontend
  become: yes
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
| `hosts: frontend` | run on the `[frontend]` group, over **SSH** (no `connection: local` this time) |
| `become: yes` | run as **root** (sudo). Installing software needs root. |
| `package:` `name: nginx` `state: present` | nginx must be **installed** (uses `dnf` or `apt` automatically) |
| `service:` `state: started` | nginx must be **running** now |
| `enabled: yes` | nginx must **start on every boot** |

**Run it:**

```bash
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=<password> 04-nginx.yaml
```

**Expected:**

| Run | Result | Why |
|---|---|---|
| 1st run | `changed` for both tasks | nginx was installed and started |
| 2nd run | `ok`, `changed=0` | already done, nothing to change (**idempotent**) |

**Open port 80** so a browser can reach nginx. Add an inbound rule to `roboshop-common`:

```bash
aws ec2 authorize-security-group-ingress --group-name roboshop-common --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**What it does:** allows HTTP (port 80) into the servers from anywhere. Port 80 is only needed to **view** the page; Ansible itself only uses port 22.

**Check nginx:**

| Where | Command | Expected |
|---|---|---|
| Browser | `http://<frontend-public-ip>` | nginx welcome page |
| Terminal | `curl -I http://<frontend-public-ip>` | `HTTP/1.1 200 OK` |

**If the page doesn't open:**

| Symptom | Cause | Fix |
|---|---|---|
| browser keeps loading, then **timed out** | port 80 not allowed in the security group | add the port 80 rule above |
| **connection refused** | nginx not running | run the playbook again |
| wrong page | wrong IP | use the **frontend's** public IP, not the Ansible server's |

## 11. Variables

**Definition:** **Variable** = a **name** that stores a **value**. Write the value once, use it in many places with **`{{ name }}`**.

Ansible can define variables in **6 places**. We try each one with a small playbook.

| # | Where | Keyword | File |
|---|---|---|---|
| 1 | in the **play** | `vars:` | `05-variables.yaml` |
| 2 | in a **task** | `vars:` under a task | `06-variables.yaml` |
| 3 | in a **separate file** | `vars_files:` | `07-variables.yaml` + `course.yaml` |
| 4 | **asked at run time** | `vars_prompt:` | `08-variables.yaml` |
| 5 | in the **inventory** | `[group:vars]` | `09-variables.yaml` |
| 6 | on the **command line** | `-e "name=value"` | `10-variables.yaml` |

### 11.1 Play variables

**Definition:** variables under `vars:` in the **play**. Every task in that play can use them.

```yaml
- name: demo on variables
  hosts: local
  connection: local
  vars:
    course: ansible
    trainer: Sivakumar Reddy M
    duration: 10 hrs
  tasks:
  - name: print the vars
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

**Expected:** `Hi, I am learning ansible from Sivakumar Reddy M, duration is 10 hrs`

| Shell | Ansible |
|---|---|
| `COURSE="ansible"` | `course: ansible` under `vars:` |
| `echo "$COURSE"` | `msg: "{{ course }}"` |

### 11.2 Task variables

**Definition:** variables under `vars:` **inside one task**. They work only in that task and **win over** play variables with the same name.

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

| Task | Prints | Why |
|---|---|---|
| print the vars | `100acres` | no task variable, so it uses the play variable |
| print the vars again | `10acres` | the task's own variable wins |

### 11.3 Variables from a file

**Definition:** **`vars_files`** = load variables from a separate YAML file, so you can change values without touching the playbook.

`course.yaml`:

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
  - course.yaml
  tasks:
  - name: print course information
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

### 11.4 Variables from a prompt

**Definition:** **`vars_prompt`** = Ansible **asks you** to type the value when the playbook runs. Good for passwords you don't want to save in files.

```yaml
- name: variables from prompt
  hosts: local
  connection: local
  vars_prompt:
  - name: username
    prompt: Please enter your username
    private: false                # you see what you type
  - name: password
    prompt: Please enter your password
    private: true                 # typing is hidden
  tasks:
  - name: print the information
    debug:
      msg: "username is: {{ username }}"
```

> ⚠️ Never print passwords in real projects.

### 11.5 Variables from the inventory

**Definition:** variables under **`[group:vars]`** in `inventory.ini`. Every server in that group gets them automatically.

```ini
[frontend]
172.31.20.30

[frontend:vars]
course=ansible
trainer=Sivakumar Reddy M
duration=10 hrs
```

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

### 11.6 Variables from the command line

**Definition:** **`-e`** (extra vars) = pass a value **when you run** the playbook.

```yaml
- name: print greeting
  hosts: local
  connection: local
  tasks:
  - name: print greeting
    debug:
      msg: "Hello {{ person_name }}"
```

| Run | Result |
|---|---|
| `ansible-playbook -i inventory.ini 10-variables.yaml` | ❌ `'person_name' is undefined` |
| `ansible-playbook -i inventory.ini 10-variables.yaml -e "person_name=Sivakumar"` | ✅ `Hello Sivakumar` |

> Every `{{ variable }}` must be defined somewhere, otherwise the playbook **fails**.

**Values with spaces need inner quotes:**

| Command | Prints |
|---|---|
| `-e "course=ansible from cmd"` | `ansible` ❌ (stops at the first space) |
| `-e "course='ansible from cmd'"` | `ansible from cmd` ✅ |

### 11.7 Which value wins?

**Definition:** **Variable precedence** = when the **same variable name** is defined in many places, Ansible picks the one with the **highest priority**.

![Variable priority: command line wins, then task, file, prompt, play, inventory](images/05-variable-priority.svg)

```
command line  >  task  >  file  >  prompt  >  play  >  inventory
   (wins)                                              (lowest)
```

**How we tested it** (`11-vars-preference.yaml` defines `course` in every place): run it, then remove the winner and run again. The next one in the list wins each time.

| Run | Prints | Winner |
|---|---|---|
| with `-e "course='ansible from cmd'"` | `ansible from cmd` | command line |
| without `-e` | `ansible from task` | task |
| task `vars:` commented out | `ansible from file` | file |

> **The command line always wins.** What you pass at run time overrides everything in files.

## 12. What I learned

- **Ansible** configures servers: packages, users, services. The app **code** comes separately.
- **Control node** has Ansible; **managed nodes** only need **SSH + Python** (agentless).
- **Inventory** = which servers, in groups. Check it with `ansible-inventory -i inventory.ini --graph`.
- **Module** = one job (`ping`, `debug`, `package`, `service`). You say **what**; Ansible works out **how** (`dnf` or `apt`).
- Results: `ok` = already correct, `changed` = Ansible fixed it, `failed` = error. Running twice is safe (**idempotent**).
- **YAML**: `key: value` = map, `-` = list item, spaces (never tabs) show nesting.
- **Playbook** → plays (`hosts`) → tasks (`name`) → modules. Run with `ansible-playbook -i inventory.ini <file>.yaml`.
- `become: yes` = run as root. `connection: local` = run on the Ansible server itself, no SSH.
- **Variables** `{{ name }}` come from 6 places. Priority: **command line > task > file > prompt > play > inventory**.

```
inventory (WHICH servers) ──┐
                            ├──► ansible-playbook ──SSH──► servers configured
playbook (WHAT to do) ──────┘
```

## Interview Questions

See [interview-questions/README.md](interview-questions/README.md).
