# Day 21 - Ansible Facts, Config Order, Data Types, Conditions, Loops, Filters & Error Handling

## About this class

In this class we learn how to make playbooks **smart**: read information about the server (**facts**), decide whether a task should run (**conditions**), repeat a task for many items (**loops**), change values (**filters**), and deal with commands that fail (**error handling**).

**What we cover:**

| Part | Topic |
|---|---|
| **Part 1: Concepts** | **facts** and gather facts · which **ansible.cfg** is used · **data types** · **conditions** (`when`) · **loops** (`loop`) · **filters** (Ansible's functions) · **error handling** (`register`, `rc`, `ignore_errors`) · **command vs shell** · **idempotency** |
| **Part 2: Lab** | better **security group** rules · run one playbook for each concept on the Ansible server |

**Before this:** [Day 20 - Ansible Basics](../day-20-ansible-data-formats-yaml/README.md) (inventory, playbooks, variables) · **Hands-on lab:** [Control node + managed nodes with ansible.cfg](../ansible-lab-control-node-managed-nodes/README.md)

## Table of Contents

**Part 1: Concepts**
1. [Facts](#1-facts)
2. [Which ansible.cfg is used?](#2-which-ansiblecfg-is-used)
3. [Data types](#3-data-types)
4. [Conditions (`when`)](#4-conditions-when)
5. [Loops (`loop`)](#5-loops-loop)
6. [Filters (Ansible's functions)](#6-filters-ansibles-functions)
7. [Error handling](#7-error-handling)
8. [command vs shell](#8-command-vs-shell)
9. [Idempotency](#9-idempotency)

**Part 2: Lab**
- [Step 1: Security group rules](#step-1-security-group-rules)
- [Step 2: Get the latest class code](#step-2-get-the-latest-class-code)
- [Step 3: Check which ansible.cfg is used](#step-3-check-which-ansiblecfg-is-used)
- [Step 4: Print facts](#step-4-print-facts)
- [Step 5: Data types](#step-5-data-types)
- [Step 6: Condition: even or odd](#step-6-condition-even-or-odd)
- [Step 7: Condition: install by OS](#step-7-condition-install-by-os)
- [Step 8: Loop over names](#step-8-loop-over-names)
- [Step 9: Loop to install packages](#step-9-loop-to-install-packages)
- [Step 10: Loop over a list of dictionaries](#step-10-loop-over-a-list-of-dictionaries)
- [Step 11: Filters](#step-11-filters)
- [Step 12: Error handling: create a user only if missing](#step-12-error-handling-create-a-user-only-if-missing)
- [Step 13: Idempotency: the same with the user module](#step-13-idempotency-the-same-with-the-user-module)
- [Step 14: command vs shell](#step-14-command-vs-shell)

[What I learned](#what-i-learned) · [Interview one-liners](#interview-one-liners)

---

# Part 1: Concepts

## 1. Facts

**Definition:** **Facts** = information Ansible **collects automatically** about a managed server (OS, IP, CPU, memory, disks…) and stores as **variables**.

**Gathering facts** = the step that collects them. You see it as the **first task of every play**: `TASK [Gathering Facts]`, even though you never wrote it.

### What does gathering facts do?

![How gathering facts works](images/04-how-gather-facts-works.svg)

| # | What happens |
|---|---|
| 1 | The play starts. Before your tasks, Ansible connects to each server (SSH, or `connection: local` for the Ansible server itself) |
| 2 | It runs the **`setup`** module on that server |
| 3 | `setup` reads the system: OS name and version, hostname, IP addresses, CPU, RAM, disks, Python version… |
| 4 | It sends everything back to the Ansible server as **JSON** |
| 5 | Ansible saves it in a variable called **`ansible_facts`**, one per server |
| 6 | Your tasks can now use it: `{{ ansible_facts['os_family'] }}` |

**Facts are variables** (`variables == facts`). You use them exactly like your own variables, but **you don't define them**: Ansible fills them in from the real server.

### Who collects the facts?

The Ansible server **asks**, but each managed node **collects facts about itself** (only that server knows its own IP, RAM and disks).

| Who | Does |
|---|---|
| **Ansible server** | connects and sends the `setup` module to the node |
| **Managed node** | runs `setup`, reads its own system, sends the result back as **JSON** |
| **Ansible server** | receives the JSON and keeps it as `ansible_facts` |

### Where are facts stored?

**By default: in the Ansible server's RAM (temporary memory), only while the playbook runs.** They are **not saved to any file**.

```
Playbook starts
   ↓
Gathering Facts → node sends JSON → Ansible server keeps it in RAM
   ↓
Your tasks use {{ ansible_facts... }}   ← read from RAM
   ↓
Playbook ends → RAM cleared → facts gone
   ↓
Next run → gathers them again (always fresh)
```

**Like a shell variable:** it exists while the script runs and is gone when the script ends.

You can see this default in the class `ansible.cfg`: `;fact_caching=memory` (memory = temporary).

**Saving facts to disk (fact caching):** with hundreds of servers, gathering facts on every run is slow. You can save them as files and reuse them:

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400
```

| Setting | Means |
|---|---|
| `gathering = smart` | gather facts only if they are **not** already in the cache |
| `fact_caching = jsonfile` | save facts as JSON files instead of only RAM |
| `fact_caching_connection` | the folder on the Ansible server (one file per server) |
| `fact_caching_timeout = 86400` | facts are valid for 1 day (in seconds), then gathered again |

| | Default (`memory`) | With `jsonfile` |
|---|---|---|
| Where | RAM of the Ansible server | JSON files on the Ansible server's disk |
| How long | only during that playbook run | until the timeout (e.g. 1 day) |
| After the run | gone | still there, reused by the next run |

### Why do we use facts?

A playbook usually runs on **many different servers**. Facts let the playbook **look at each server and decide**, instead of you hard-coding values.

| Need | Without facts | With facts |
|---|---|---|
| Install a package on Amazon Linux **and** Ubuntu | write 2 playbooks | one playbook: `when: ansible_facts.os_family == "RedHat"` → `dnf`, `"Debian"` → `apt` |
| Put the server's private IP in a config file | look up and type each IP by hand | `{{ ansible_facts['default_ipv4']['address'] }}` |
| Skip small servers | check each server yourself | `when: ansible_facts.memtotal_mb > 1000` |
| Print a report of all servers (OS, IP, RAM) | log in to each one | one `debug` task |

**In one line:** facts tell the playbook **"who is this server?"**, so the same playbook works correctly on every server.

### Useful facts

| Fact | Example value | Use it for |
|---|---|---|
| `ansible_facts['os_family']` | `RedHat`, `Debian` | pick `dnf` or `apt` |
| `ansible_facts['distribution']` | `Amazon`, `Ubuntu` | OS name |
| `ansible_facts['distribution_version']` | `2023` | OS version |
| `ansible_facts['default_ipv4']['address']` | `172.31.20.30` | private IP of the server |
| `ansible_facts['hostname']` | `ip-172-31-20-30` | server name |
| `ansible_facts['memtotal_mb']` | `916` | RAM in MB |
| `ansible_facts['processor_vcpus']` | `2` | number of CPUs |

**Two ways to write the same fact:**

| Style | Example |
|---|---|
| bracket style | `ansible_facts['default_ipv4']['address']` |
| dot style | `ansible_facts.default_ipv4.address` |

`default_ipv4` is a **dictionary** inside `ansible_facts`, and `address` is a key inside it. That's why there are two levels.

> Use `ansible_facts['os_family']`, not the old `ansible_os_family` form. Newer Ansible versions warn that the old form is deprecated.

### Turning facts off

Gathering facts takes a few seconds **per server**. If a play doesn't use any fact, turn it off:

```yaml
- name: no facts needed
  hosts: local
  gather_facts: false
```

| `gather_facts` | Result |
|---|---|
| `true` (default) | `TASK [Gathering Facts]` runs, `ansible_facts` is filled |
| `false` | faster, but any task that uses `ansible_facts` **fails** (the value is undefined) |

**See facts without a playbook** (ad-hoc): `ansible local -c local -m setup`. Add `-a "filter=ansible_os_family"` to see just one.

## 2. Which ansible.cfg is used?

**Definition:** **`ansible.cfg`** = Ansible's **settings file** (where the inventory is, which user, which key…).

Ansible checks **4 places in order** and uses the **first** file it finds:

![ansible.cfg search order](images/01-ansible-cfg-search-order.svg)

| Order | Location | When to use |
|---|---|---|
| 1 | `ANSIBLE_CONFIG` environment variable | point to a config file from any folder |
| 2 | `./ansible.cfg` (current folder) | **the normal way**: one config per project, kept in Git |
| 3 | `~/.ansible.cfg` (home folder) | one config for your user |
| 4 | `/etc/ansible/ansible.cfg` | default for the whole server |

**Check which file is used:** `ansible --version` → the line `config file = ...`

**Our class `ansible.cfg`:**

```ini
[defaults]
inventory=inventory.ini
stdout_callback=yaml
```

| Setting | Means |
|---|---|
| `inventory=inventory.ini` | no need to type `-i inventory.ini` every time |
| `stdout_callback=yaml` | print task results as **YAML** (easier to read than JSON) |

> **If you get** `Could not load 'yaml' callback plugin`: newer Ansible versions removed that plugin. Use this line instead, which does the same:
> `callback_result_format=yaml`

More hands-on practice (`-i` overriding the config, `ANSIBLE_CONFIG` temporary vs permanent): [hands-on lab, Parts 4 and 5](../ansible-lab-control-node-managed-nodes/README.md).

## 3. Data types

**Definition:** **Data type** = the **kind of value** a variable holds (text, number, true/false, a list, a dictionary).

| Type | In Ansible (`type_debug`) | Example in YAML |
|---|---|---|
| String (text) | `str` | `course: "AIOps with DevSecOps on AWS"` |
| Number (whole) | `int` | `duration_in_hrs: 150` |
| Number (decimal) | `float` | `exact_duration_in_months: 4.5` |
| Boolean (true/false) | `bool` | `is_live_sessions: true` |
| List (many values) | `list` | `topics:` then `- Linux`, `- Shell` on new lines |
| Dictionary / map (key: value pairs) | `dict` | `more_details:` then `trainer: sivakumar` |

**List vs dictionary:**

```yaml
topics:            # list: items with "-"
- Linux
- Shell

more_details:      # dictionary: key: value
  trainer: sivakumar
  months: 4
```

**Find the type of any variable:** `{{ course | type_debug }}` → `str`

> **Quote numbers when you want text:** `duration: "4"` is a **string**, `duration: 4` is an **int**. The `int` filter converts text to a number (see [Filters](#6-filters-ansibles-functions)).

## 4. Conditions (`when`)

**Definition:** **Condition** = a check that decides **whether a task should run or not**. The keyword is **`when`**.

```yaml
- name: is even
  debug:
    msg: "{{ given_number }} is even"
  when: given_number % 2 == 0
```

- Condition is **true** → task runs (`ok` / `changed`)
- Condition is **false** → task is **skipped** (`skipping`, counted in `skipped=` in the recap)

| Operator | Means | Example |
|---|---|---|
| `==` | equal | `when: ansible_facts.os_family == "RedHat"` |
| `!=` | not equal | `when: user_output.rc != 0` |
| `>` `<` `>=` `<=` | compare numbers | `when: ansible_facts.memtotal_mb > 1000` |
| `%` | remainder (for even/odd) | `when: given_number % 2 == 0` |
| `and` / `or` / `not` | combine | `when: x > 1 and y < 5` |
| `in` | value is in a list | `when: "'nginx' in packages"` |
| `is defined` | variable exists | `when: person_name is defined` |

> **No `{{ }}` inside `when`.** `when` is already an expression. Write `when: given_number % 2 == 0`, not `when: "{{ given_number }} % 2 == 0"`.

**Facts + conditions = one playbook for any OS:**

![Facts and conditions](images/02-facts-and-conditions.svg)

## 5. Loops (`loop`)

**Definition:** **Loop** = run **the same task many times**, once for each item in a list. The keyword is **`loop`**.

**`item`** = a **reserved word** that holds the **current** value in each round.

```yaml
- name: print names
  debug:
    msg: "Hi {{ item }}"
  loop:
  - Ramesh
  - Suresh
  - Raheem
```

Output: `Hi Ramesh`, `Hi Suresh`, `Hi Raheem` (one task, 3 rounds).

**3 ways to give the list:**

| Way | Example | `item` is |
|---|---|---|
| Write the list under `loop:` | `loop:` then `- John`, `- Ram` | a string |
| Use a list variable | `loop: "{{ packages }}"` | a string |
| List of dictionaries | `- { name: 'nginx', state: 'absent' }` | a dictionary → use `item.name`, `item.state` |

> **Without a loop** you would copy the same task 5 times for 5 packages. With a loop it's one task.

## 6. Filters (Ansible's functions)

**Definition:** **Filter** = a built-in **function** that **changes a value**. You write it after the pipe symbol **`|`**: `{{ value | filter }}`.

Ansible has no way to write your own functions in YAML. You use the **filters** it gives you. (Custom filters are possible, but they are written in **Python**.)

| Filter | Does | Example | Result |
|---|---|---|---|
| `default('x')` | use `x` if the variable is not defined | `{{ person_name \| default('Ramesh') }}` | `Ramesh` |
| `split(',')` | string → list | `{{ "a, b" \| split(',') }}` | `['a', ' b']` |
| `map('trim') \| list` | remove spaces from each item | `{{ "a, b" \| split(',') \| map('trim') \| list }}` | `['a', 'b']` |
| `join(' ')` | list → string | `{{ names \| join(' ') }}` | `Sivakumar Reddy Mettukuru` |
| `lower` / `upper` | change case | `{{ "ANSIBLE" \| lower }}` | `ansible` |
| `int` | string → number | `{{ "4" \| int }}` | `4` |
| `length` | count items | `{{ topics \| length }}` | `4` |
| `type_debug` | show the data type | `{{ 4.5 \| type_debug }}` | `float` |

**Filters can be chained:** `{{ topics | split(',') | map('trim') | list }}` runs left to right.

## 7. Error handling

**Definition:** **Error handling** = deciding **what Ansible should do when a task fails**, instead of stopping the whole playbook.

**Default behaviour:** a task fails → Ansible **stops** on that server (the next tasks don't run there).

**The rule:**

| Error | What to do | Example |
|---|---|---|
| **Known** error (you expect it) | **handle it** | `id roboshop` fails when the user doesn't exist yet. That's normal, so keep going. |
| **Unknown** error (you didn't expect it) | **don't handle it, let it fail** | disk full, package not found, typo. You want to **see** these, not hide them. |

**Exit code (`rc`):** every Linux command returns a number when it finishes.

| `rc` | Means |
|---|---|
| `0` | **success** |
| not `0` (1, 2, 127…) | **failure** |

**The 3 keywords:**

| Keyword | Does |
|---|---|
| `register: my_var` | saves the task's result (`rc`, `stdout`, `stderr`…) in a variable |
| `ignore_errors: true` | if this task fails, **don't stop**, continue with the next task |
| `when: my_var.rc != 0` | use the saved result to decide whether the next task runs |

**When there's no module:** if Ansible has no module for what you need, use a **Linux command** (`command` / `shell` module). `id roboshop` is one example: it checks whether a user exists.

![Error handling with register and rc](images/03-error-handling-register-rc.svg)

More: `failed_when`, `block`/`rescue`/`always`, `retries` → [ansible-notes: Error handling](https://github.com/Abhignadumpala/ansible-notes/tree/main/06-error-handling)

## 8. command vs shell

**Definition:**
- **`command`** module = runs **one simple command**, **without** a shell.
- **`shell`** module = runs the command **through a shell** (`/bin/sh`), so shell features work.

| | `command` | `shell` |
|---|---|---|
| Pipes `\|` | ❌ | ✅ `cat /etc/passwd \| grep ec2-user` |
| Redirect `>` `>>` | ❌ | ✅ `ls -la > /tmp/files.txt` |
| Variables like `$HOME`, `&&`, `*` | ❌ | ✅ |
| Safer | ✅ (no shell tricks) | less safe |
| Use when | a plain command is enough | you need a pipe, redirect or `&&` |

With `command`, `|` is **not** a pipe. It is passed to `cat` as a file name, so it fails: `cat: '|': No such file or directory`.

> Both are **not idempotent**: they run (and report `changed`) **every time**. Use a real module when one exists.

## 9. Idempotency

**Definition:** **Idempotency** = running the same playbook **many times gives the same result**. If the server is already in the right state, Ansible **changes nothing**.

| Run | Module (`user: name=roboshop`) | Plain command (`useradd roboshop`) |
|---|---|---|
| 1st | creates the user → `changed` | creates the user |
| 2nd | user exists → `ok` (does nothing) | **fails**: `user 'roboshop' already exists` |

**Two ways to get the same safe result:**

| Way | How | Class file |
|---|---|---|
| Hard way | `command: id roboshop` + `register` + `ignore_errors` + `when: rc != 0` | `20-error-handling.yaml` |
| **Easy way** | `user:` module, which checks by itself | `21-idempotency.yaml` |

**Rule:** use a **module** if one exists (it's idempotent). Use `command`/`shell` **only** when there is no module, and then **you** must add the check (`register` + `when`).

---

# Part 2: Lab

```
Step 1  security group rules
Step 2–3  class code + which ansible.cfg
Steps 4–14  one playbook per concept, run on the Ansible server itself (hosts: local, connection: local)
```

All playbooks in Steps 4–14 run on the **Ansible server itself** (`hosts: local` + `connection: local`), so no other server is needed.

### Step 1: Security group rules

**Definition:** **Security group** = AWS firewall for a server. **Inbound rule** = what traffic is allowed **in**.

In Day 20 port 22 and port 80 were open to the whole internet on `roboshop-common`. Now we make it safer and **split** the rules into 2 groups:

| Security group | Port | From | Why | Attach to |
|---|---|---|---|---|
| `roboshop-common` | 22 (SSH) | **my IP only** | only I can log in | **all** servers |
| `roboshop-frontend` | 80 (HTTP) | `0.0.0.0/0` (anyone) | users open the website | **frontend** only |

The frontend server gets **both** groups: `roboshop-common` + `roboshop-frontend`.

**Commands (on your laptop):**

```bash
curl -s https://checkip.amazonaws.com
```

**What it does:** prints your laptop's public IP. Use it as `<your-ip>` below.

```bash
aws ec2 authorize-security-group-ingress --group-name roboshop-common --protocol tcp --port 22 --cidr <your-ip>/32
```

**What it does:** allows SSH (port 22) only from your IP (`/32` = exactly one IP).

```bash
aws ec2 revoke-security-group-ingress --group-name roboshop-common --protocol tcp --port 22 --cidr 0.0.0.0/0
```

**What it does:** removes the old "SSH from anywhere" rule from Day 20.

```bash
aws ec2 create-security-group --group-name roboshop-frontend --description "HTTP for frontend"
```

**What it does:** creates the new security group for the frontend.

```bash
aws ec2 authorize-security-group-ingress --group-name roboshop-frontend --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**What it does:** allows HTTP (port 80) from anywhere, only on servers that have this group.

**Expected:** `"Return": true` for each rule. `InvalidPermission.Duplicate` = the rule already exists (fine). `InvalidPermission.NotFound` on revoke = it was never there (fine).

> **Ansible server → managed nodes over SSH:** "port 22 from my IP" blocks the Ansible server too (it has a different IP). When Ansible needs to SSH to other servers, also allow port 22 **from the security group itself**:
> `aws ec2 authorize-security-group-ingress --group-name roboshop-common --protocol tcp --port 22 --source-group roboshop-common`
> Today's playbooks only run on the Ansible server itself, so this isn't needed yet.

### Step 2: Get the latest class code

Log in to the Ansible server, then:

```bash
cd ansible-code
```

```bash
git pull
```

**What it does:** downloads the new playbooks (`12-gather-facts.yaml` … `22-shell-commad.yaml`).

### Step 3: Check which ansible.cfg is used

```bash
ansible --version
```

**Expected:** `config file = /home/ec2-user/ansible-code/ansible.cfg` (the file in the current folder, order #2).

```bash
cd ~
```

```bash
ansible --version
```

**Expected:** `config file = /etc/ansible/ansible.cfg` if that file exists, otherwise `None`.

```bash
cd ~/ansible-code
```

**What it does:** go back. **Always run the playbooks from this folder**, so `inventory=inventory.ini` is found.

### Step 4: Print facts

**Why print them?** When the play starts, Ansible **gathers the facts and stores them** in `ansible_facts` (in memory, until the playbook ends). But we **can't see them on screen**: `TASK [Gathering Facts]` only shows `ok: [localhost]`.

**If you want to see what it gathered, print it with `debug`.** That's what this playbook does.

```
Gathering Facts → stored in ansible_facts (not shown on screen)
      ↓
debug task      → prints them so you can see them
      ↓
playbook ends   → facts gone from memory
```

> In real playbooks we **don't print** facts, we just **use** them (e.g. `when: ansible_facts['os_family'] == "RedHat"`). Printing is only to **learn the fact names** and **check their values** (debugging).

`12-gather-facts.yaml`:

```yaml
- name: print facts
  hosts: local
  connection: local
  tasks:
  - name: print the facts
    debug:
      msg: "{{ ansible_facts }}"

  - name: print some info
    debug:
      msg: |
        OS Family: {{ ansible_facts['os_family'] }}
        private_ip: {{ ansible_facts.default_ipv4.address }}
        private_ip: {{ ansible_facts['default_ipv4']['address'] }}
```

| Line | Means |
|---|---|
| `msg: "{{ ansible_facts }}"` | prints **all** facts (very long) |
| `msg: \|` | multi-line message (`\|` keeps the new lines) |
| two `private_ip` lines | the same fact in **bracket** and **dot** style |

```bash
ansible-playbook 12-gather-facts.yaml
```

**Expected:** a long list of facts, then `OS Family: RedHat` and the server's private IP **twice** (both styles give the same value).

**See only one fact (ad-hoc):**

```bash
ansible local -c local -m setup -a "filter=ansible_os_family"
```

**What it does:** runs the `setup` module (the one behind Gathering Facts) and shows only `os_family`. `-c local` = connection local.
**Expected:** `ansible_os_family: RedHat`

### Step 5: Data types

`13-data-types.yaml`:

```yaml
- name: course information
  hosts: local
  connection: local
  vars:
    course: "AIOps with DevSecOps on AWS" # string
    duration_in_hrs: 150 # number
    is_live_sessions: true # boolean
    topics: # list
    - Linux
    - Shell
    - Ansible
    - Terraform
    more_details: # dictionary/map
      trainer: sivakumar
      months: 4
    exact_duration_in_months: 4.5
  tasks:
  - name: print info and type
    debug:
      msg: |
        name: "{{ course }} : {{ course | type_debug }} "
        duration: {{ duration_in_hrs }} : {{ duration_in_hrs | type_debug }}
        is live: {{ is_live_sessions }} : {{ is_live_sessions | type_debug }}
        topics are: {{ topics }} : {{ topics | type_debug }}
        more details: {{ more_details }} : {{ more_details | type_debug }}
        duration: {{ exact_duration_in_months }} : {{ exact_duration_in_months | type_debug }}
```

```bash
ansible-playbook 13-data-types.yaml
```

**Expected:**

```
name: "AIOps with DevSecOps on AWS : str "
duration: 150 : int
is live: True : bool
topics are: ['Linux', 'Shell', 'Ansible', 'Terraform'] : list
more details: {'trainer': 'sivakumar', 'months': 4} : dict
duration: 4.5 : float
```

### Step 6: Condition: even or odd

`14-conditions.yaml`:

```yaml
- name: even or odd
  hosts: local
  connection: local
  vars:
    given_number: 10
  tasks:
  - name: is even
    debug:
      msg: "Given  number: {{ given_number }} is even"
    when: given_number % 2 == 0 # expression

  - name: is odd
    debug:
      msg: "Given  number: {{ given_number }} is odd"
    when: given_number % 2 != 0 # expression
```

```bash
ansible-playbook 14-conditions.yaml
```

**Expected:** `is even` → `Given  number: 10 is even`. `is odd` → `skipping: [localhost]`. Recap: `skipped=1`.

Try an odd number (`-e` overrides the play variable):

```bash
ansible-playbook 14-conditions.yaml -e '{"given_number": 7}'
```

**Expected:** now `is even` is skipped and `is odd` prints `7 is odd`.

> **Why not `-e given_number=7`?** `key=value` on the command line is always a **string** (`"7"`), and `%` on a string fails: `not all arguments converted during string formatting`. The JSON form `'{"given_number": 7}'` keeps it a **number**. Another fix is the `int` filter in the condition: `when: given_number | int % 2 == 0`.

### Step 7: Condition: install by OS

`15-conditions.yaml`:

```yaml
- name: print OS is RedHat or not
  hosts: local
  become: yes
  connection: local
  tasks:
  - name: Install nginx in RedHat
    dnf:
      state: present
      name: nginx
    when: ansible_facts.os_family == "RedHat"

  - name: Install nginx in debian
    apt:
      state: present
      name: nginx
    when: ansible_facts.os_family == "Debian"
```

| Line | Means |
|---|---|
| `become: yes` | run as root (`sudo`), needed to install packages |
| `dnf:` / `apt:` | package modules for RedHat family / Debian family |
| `when: ansible_facts.os_family == ...` | uses a **fact** to pick the right task |

```bash
ansible-playbook 15-conditions.yaml
```

**Expected (Amazon Linux = RedHat family):** `Install nginx in RedHat` → `changed` (or `ok` if nginx is already installed). `Install nginx in debian` → `skipping`.

### Step 8: Loop over names

`16-loops.yaml`:

```yaml
- name: demo on loops
  hosts: local
  connection: local
  tasks:
  - name: print names
    debug:
      msg: "Hi {{ item }}" # item is reserved keyword
    loop:
    - John
    - Ram
    - Raheem
```

```bash
ansible-playbook 16-loops.yaml
```

**Expected:** one task, three results:

```
ok: [localhost] => (item=John) =>   msg: Hi John
ok: [localhost] => (item=Ram) =>    msg: Hi Ram
ok: [localhost] => (item=Raheem) => msg: Hi Raheem
```

### Step 9: Loop to install packages

`17-loops.yaml`:

```yaml
- name: install packages
  hosts: local
  connection: local
  become: yes
  vars:
    packages:
    - nginx
    - mysql
    - git
    - net-tools
    - htop
  tasks:
  - name: install packages
    package:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
```

| Line | Means |
|---|---|
| `packages:` | a **list variable** |
| `package:` | generic module: uses `dnf` or `apt` automatically |
| `loop: "{{ packages }}"` | loop over the variable (needs quotes + `{{ }}`) |

```bash
ansible-playbook 17-loops.yaml
```

**Expected:** one line per package, `changed` (or `ok` if already installed).

> If a package doesn't exist in your OS's repositories, that item fails with a "no package found" message. Remove it from the list and run again.

### Step 10: Loop over a list of dictionaries

`18-loops.yaml`:

```yaml
- name: install packages
  hosts: local
  connection: local
  become: yes
  vars:
    packages:
    - { name: 'nginx', state: 'absent' }
    - { name: 'mysql', state: 'present' }
    - { name: 'git', state: 'present' }
    - { name: 'net-tools', state: 'present' }
    - { name: 'htop', state: 'absent' }
  tasks:
  - name: install packages
    package:
      name: "{{ item.name }}"
      state: "{{ item.state }}"
    loop: "{{ packages }}"
```

| Line | Means |
|---|---|
| `{ name: 'nginx', state: 'absent' }` | each item is a **dictionary** with 2 keys |
| `item.name` / `item.state` | read one key from the current item |
| `state: absent` | **remove** the package · `present` = install it |

```bash
ansible-playbook 18-loops.yaml
```

**Expected:** nginx and htop **removed** (`changed`), the others `ok` (installed in Step 9). One task both installs and removes.

### Step 11: Filters

`19-filter.yaml`:

```yaml
- name: demo on filters
  hosts: local
  connection: local
  vars:
    topics: Linux, Shell, Ansible, Terraform, Docker
    names:
    - Sivakumar
    - Reddy
    - Mettukuru
    topic: ANSIBLE
    duration: "4"
  tasks:
  - name: print the name
    debug:
      msg: "Hi, {{ person_name | default('Ramesh') }}"

  - name: convert string to list
    debug:
      msg: "Topics are: {{ topics | split(',') | map('trim') | list }}"

  - name: convert list to string
    debug:
      msg: "Full name: {{ names | join(' ') }}"

  - name: convert to lower case
    debug:
      msg: "Topic: {{ topic | lower }}"

  - name: convert to string to int
    debug:
      msg: "Duration: {{ duration | int }}"
```

```bash
ansible-playbook 19-filter.yaml
```

**Expected:**

| Task | Output |
|---|---|
| print the name | `Hi, Ramesh` (`person_name` is not defined, so `default` is used) |
| convert string to list | `Topics are: ['Linux', 'Shell', 'Ansible', 'Terraform', 'Docker']` |
| convert list to string | `Full name: Sivakumar Reddy Mettukuru` |
| convert to lower case | `Topic: ansible` |
| convert to string to int | `Duration: 4` |

> **Why `map('trim') | list`?** `split(',')` keeps the space after each comma: `' Shell'`. `map('trim')` removes it from **every** item. (Writing just `| trim` after the list does **not** remove them: the spaces are still there.)

Give a name to see `default` not used:

```bash
ansible-playbook 19-filter.yaml -e person_name=Suresh
```

**Expected:** `Hi, Suresh`

### Step 12: Error handling: create a user only if missing

`20-error-handling.yaml`:

```yaml
- name: create user
  hosts: local
  connection: local
  become: yes
  tasks:
  - name: check roboshop user exist or not
    command: id roboshop # Linux command, when there is no ansible module exist
    register: user_output
    ignore_errors: true

  - name: print the output
    debug:
      msg: "{{ user_output }}"

  - name: create user
    command: useradd roboshop
    when: user_output.rc != 0
```

| Line | Means |
|---|---|
| `command: id roboshop` | Linux check: does the user exist? |
| `register: user_output` | save the result (`rc`, `stdout`, `stderr`) |
| `ignore_errors: true` | user missing = **known error**, don't stop |
| `when: user_output.rc != 0` | create the user **only if** the check failed |

**1st run:**

```bash
ansible-playbook 20-error-handling.yaml
```

**Expected:**
- check task: red `FAILED`, `rc: 1`, `stderr: 'id: ''roboshop'': no such user'`, then `...ignoring`
- `create user` → `changed`
- recap: `failed=0` `ignored=1`

**2nd run (same command again):**

```bash
ansible-playbook 20-error-handling.yaml
```

**Expected:** check task `rc: 0` (user exists) → `create user` is **skipping**. No "already exists" error.

### Step 13: Idempotency: the same with the user module

`21-idempotency.yaml`:

```yaml
- name: create user
  connection: local
  hosts: local
  become: yes
  tasks:
  - name: create user
    user:
      name: roboshop # useradd roboshop
```

```bash
ansible-playbook 21-idempotency.yaml
```

**Expected:** `ok` (the user already exists from Step 12, so nothing changes). On a fresh server: `changed` the first time, `ok` after that.

**Same result as Step 12, with 1 task instead of 3.** The `user` module checks by itself.

### Step 14: command vs shell

`22-shell-commad.yaml`:

```yaml
- name: shell vs command
  hosts: local
  connection: local
  tasks:
  - name: test the command module
    #command: cat /etc/passwd | grep ec2-user
    shell: cat /etc/passwd | grep ec2-user
    register: ec2_user_output

  - name: print the output
    debug:
      msg: "{{ ec2_user_output }}"

  - name: get the files
    #command: ls -la > /tmp/files.txt
    shell: ls -la > /tmp/files.txt
    register: files_output

  - name: print files info
    debug:
      msg: "{{ files_output }}"
```

```bash
ansible-playbook 22-shell-commad.yaml
```

**Expected:** `stdout` shows the `ec2-user` line from `/etc/passwd`, and `/tmp/files.txt` is created.

```bash
cat /tmp/files.txt
```

**What it does:** shows the file the playbook created with `>`.

**Try it with `command`:** in the file, put `#` in front of the two `shell:` lines and remove `#` from the two `command:` lines, then run again.
**Expected:** `cat: '|': No such file or directory` → `command` doesn't understand `|` or `>`.

### Clean up (optional)

```bash
sudo userdel -r roboshop
```

**What it does:** deletes the test user (`-r` also deletes its home folder).

---

## What I learned

**Part 1: Concepts**

| Topic | One line |
|---|---|
| Facts | each server sends its own info (OS, IP, RAM) as JSON. It's kept in the Ansible server's RAM only during the run |
| ansible.cfg order | `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`, first one wins |
| Data types | `str`, `int`, `float`, `bool`, `list`, `dict`. Check with `type_debug` |
| Conditions | `when:` runs the task only if true, otherwise `skipping` |
| Loops | `loop:` runs one task per item, current value in `item` |
| Filters | Ansible's functions, written after `\|`: `default`, `split`, `join`, `lower`, `int` |
| Error handling | handle **known** errors (`register` + `ignore_errors` + `when rc != 0`), let **unknown** ones fail |
| command vs shell | `shell` supports `\|` `>` `&&`. `command` doesn't |
| Idempotency | modules check first and change only what's needed. Commands don't |

**Part 2: Lab**
- Port 22 only from my IP (`roboshop-common`), port 80 only on the frontend (`roboshop-frontend`).
- Ran one playbook per concept on the Ansible server itself.

```
Gathering Facts ──► when: (condition) ──► loop: (repeat) ──► register + rc (check result) ──► module = idempotent
```

## Interview one-liners

| Question | Answer |
|---|---|
| What are facts? | System info (OS, IP, CPU, memory) Ansible gathers from each host with the `setup` module before running tasks. |
| Where are facts stored? | In the control node's memory, only for that run. Saved to disk only with fact caching (`fact_caching = jsonfile` or `redis`). |
| How do you disable facts? | `gather_facts: false` in the play. Faster, but `ansible_facts` can't be used. |
| ansible.cfg precedence? | `ANSIBLE_CONFIG` env var → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. First one found wins. |
| How do you write a condition? | With `when:`, no `{{ }}`, e.g. `when: ansible_facts.os_family == "RedHat"`. |
| How do you loop? | `loop:` with a list. The current value is `item`, e.g. `item.name` for a list of dicts. |
| What is a filter? | A function applied with `\|`, e.g. `{{ name \| default('x') }}`, `{{ list \| join(',') }}`. |
| What does `register` do? | Saves a task's result (`rc`, `stdout`, `stderr`, `changed`) in a variable for later tasks. |
| What is `rc`? | The command's exit code: `0` = success, anything else = failure. |
| `ignore_errors` vs handling? | `ignore_errors` only for **expected** failures. Unknown errors should fail so you notice them. |
| command vs shell? | `command` has no shell (no pipes/redirects, safer). `shell` runs in `/bin/sh`, so `\|`, `>`, `&&` work. |
| What is idempotency? | Running the same playbook again changes nothing if the server is already correct. Modules do this, `command`/`shell` don't. |
