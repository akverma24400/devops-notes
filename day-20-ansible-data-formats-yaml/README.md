# Day 20 - Ansible Setup: Create the Server, Data Formats (XML, JSON, YAML) & Variables

> **In short:** Running an app needs two things: the **code** and the **configuration** of the server. Ansible does the configuration. We create servers with the **AWS CLI**, tell Ansible which machines to target in the **inventory**, and in Ansible **everything is a module** (or a collection of modules). Playbooks are written in **YAML**, so we compared the same student details written in **XML**, **JSON** and **YAML**: `[]` is a **list**, `{}` is a **map/object**. Ansible also has programming basics: variables, data types, conditions, loops, functions and error handling, starting with **variables**.

**More in this folder:** [Interview questions](interview-questions/README.md)

## Table of Contents

1. [Code vs configuration](#code-vs-configuration)
2. [Create the server with AWS CLI](#create-the-server-with-aws-cli)
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
7. [Ansible as a programming language](#ansible-as-a-programming-language)
8. [Variables](#variables)
9. [What I learned](#what-i-learned)
10. [Interview Questions](#interview-questions)

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

## Create the server with AWS CLI

**Definition:** **`aws ec2 run-instances`** = AWS CLI command that **creates** (launches) an EC2 instance.

```bash
aws ec2 run-instances \
    --image-id ami-0055977a6e828a4f4 \
    --instance-type t3.micro \
    --security-groups "roboshop-common" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=roboshop-frontend}]' \
    --query 'Instances[0].InstanceId' \
    --output text
```

| Option | Means |
|---|---|
| `--image-id ami-...` | which **AMI** (OS image) to use |
| `--instance-type t3.micro` | size of the server (CPU/RAM) |
| `--security-groups "roboshop-common"` | security group (firewall) **by name** |
| `--tag-specifications ... Value=roboshop-frontend` | gives the server the **Name** tag `roboshop-frontend` |
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

**Definition:** **Variable** = a **name** that stores a **value**, so we write the value once and use it in many places.

In a playbook, variables go under `vars:` and are used with **`{{ name }}`**:

```yaml
- name: demo on variables
  hosts: local
  connection: local        # run on the Ansible server itself, no SSH
  vars:
    course: ansible
    trainer: Sivakumar Reddy M
    duration: 10 hrs
  tasks:
  - name: print the vars
    debug:
      msg: "Hi, I am learning {{ course }} from {{ trainer }}, duration is {{ duration }}"
```

| Part | Means |
|---|---|
| `vars:` | define variables (a **map**: name: value) |
| `{{ course }}` | use the variable's value |
| `debug` module | prints a message (like `echo`) |

| Shell | Ansible |
|---|---|
| `COURSE="ansible"` | `course: ansible` under `vars:` |
| `echo "$COURSE"` | `msg: "{{ course }}"` |

> Code for this course: [ansible-code](https://github.com/Abhignadumpala/ansible-code) (`05-variables.yaml` and onwards).

## What I learned

### Class summary

- An app needs **code** + **configuration**; Ansible does the configuration.
- Created the server with `aws ec2 run-instances`; `--query 'Instances[0].InstanceId' --output text` prints only the instance ID.
- In Ansible **everything is a module** or a collection of modules.
- **Inventory** = the machines Ansible targets, grouped (`[frontend]`, `[roboshop:children]`).
- **DTO** = data carried between systems; written in XML, JSON or YAML.
- **`[]` = list**, **`{}` = map/object**. In YAML: `-` = list item, `key: value` = map, indentation = nesting.
- Wrote the same student in XML (tags), JSON (brackets) and YAML (indentation): YAML is the shortest.
- Check YAML at **yamlchecker.com**.
- Ansible server: `sudo dnf install ansible -y` → `git clone` the code → `ansible-inventory -i inventory.ini --graph` shows `@all`, `@ungrouped`, `@roboshop`... as a tree.
- Ansible has variables, data types, conditions, loops, functions, error handling. Started with **variables**: `vars:` + `{{ name }}`.

```
AWS CLI ──► server created ──► inventory (which machines)
                                   │
                         playbook (YAML: lists -, maps key: value)
                                   │
                         modules + variables {{ }} ──► configured server
```

## Interview Questions

See [interview-questions/README.md](interview-questions/README.md).
