# Ansible Configuration Management (`ansible.cfg`)
## From an Empty AWS Account to Ansible Managing Two Servers

**Technology:** Ansible (`ansible-core`)
**Environment:** AWS EC2 – Amazon Linux 2023
**Servers:** 1 Control Node (`ansible-control`) + 2 Managed Nodes (`node1`, `node2`)
**Authentication:** SSH key-based (no passwords)
**Tools used:** AWS CLI v2, SSH, Ansible
**Time:** ~60–90 minutes · **Cost:** 3 × `t3.micro` (free-tier eligible in most regions). Delete everything at the end ([Section 14](#14-clean-up-delete-everything)).

**More in this folder:** [Interview questions](interview-questions/README.md) · Theory: [Day 19](../day-19-shell-disadvantages-ansible-intro/README.md), [Day 20](../day-20-ansible-data-formats-yaml/README.md)

---

## Table of Contents

1. [Introduction: what we are building and why](#1-introduction-what-we-are-building-and-why)
2. [Understand the Ansible architecture](#2-understand-the-ansible-architecture)
3. [Prerequisites: what you need before starting](#3-prerequisites-what-you-need-before-starting)
4. [Create the EC2 servers with the AWS CLI](#4-create-the-ec2-servers-with-the-aws-cli)
5. [Prepare the control node](#5-prepare-the-control-node)
6. [Verify connectivity: TCP and SSH](#6-verify-connectivity-tcp-and-ssh)
7. [Approach 1: Ansible without `ansible.cfg`](#7-approach-1-ansible-without-ansiblecfg)
8. [Approach 2: Ansible with `ansible.cfg`](#8-approach-2-ansible-with-ansiblecfg)
9. [How Ansible finds `ansible.cfg` (config discovery)](#9-how-ansible-finds-ansiblecfg-config-discovery)
10. [Environment variables: temporary and permanent `ANSIBLE_CONFIG`](#10-environment-variables-temporary-and-permanent-ansible_config)
11. [Use it: install nginx on both nodes with a playbook](#11-use-it-install-nginx-on-both-nodes-with-a-playbook)
12. [Troubleshooting scenarios](#12-troubleshooting-scenarios)
13. [Configuration precedence vs discovery](#13-configuration-precedence-vs-discovery)
14. [Clean up: delete everything](#14-clean-up-delete-everything)
15. [Comparison tables](#15-comparison-tables)
16. [Final summary: the whole lab in order](#16-final-summary-the-whole-lab-in-order)

---

# 1. Introduction: what we are building and why

**Definition:** **Ansible** = an **agentless configuration management tool**. From one server (the control node) it connects over **SSH** to many servers (managed nodes) and configures them: install packages, start services, copy files.

Before Ansible can talk to any server, it needs a few details every time:

- **Where is the inventory?** (the list of servers)
- **Which SSH user** should it log in as?
- **Which SSH private key** should it use?
- **Should it check the server's host key?**

There are two ways to give Ansible these details:

1. **Without `ansible.cfg`**: type them as command-line flags **on every command**.
2. **With `ansible.cfg`**: write them **once** in a config file; Ansible finds it and reuses the values.

**Why does this matter?** Imagine running 50 Ansible commands a day. Typing the inventory path, username and key path 50 times is slow and one typo breaks the run. `ansible.cfg` gives **one central place** for these settings, shorter commands, and the same settings for everyone on the team.

**In this lab we will:**

1. Create 3 EC2 servers from scratch with the AWS CLI (key pair, security groups, instances).
2. Install Ansible on the control node and give it the SSH key.
3. Run Ansible the **long way** (no config file), then the **short way** (with `ansible.cfg`).
4. Learn how Ansible **finds** its config file and how `ANSIBLE_CONFIG` changes that.
5. Install nginx on both servers with a playbook.
6. Fix common errors, then delete everything.

# 2. Understand the Ansible architecture

**Definition:**
- **Control node** = the server where **Ansible is installed**. Every Ansible command runs from here.
- **Managed node** = a server that Ansible **connects to and configures**. It only needs **SSH** and **Python**; Ansible is **not** installed on it. That's what **agentless** means.

## 2.1 Architecture diagram

```
                       Your laptop (AWS CLI + SSH)
                                  │
                                  │ SSH port 22 (allowed only from MY IP)
                                  ▼
      ┌──────────────────── AWS default VPC ──────────────────────┐
      │                                                           │
      │   CONTROL NODE: ansible-control   (SG: ansible-control-sg)│
      │     • ansible-core installed                              │
      │     • ~/.ssh/ansible_lab   (SSH private key)              │
      │     • inventory/hosts                                     │
      │     • ansible.cfg                                         │
      │     • $ ansible all -m ping                               │
      │                     │                                     │
      │                     │ SSH port 22 (private IPs)           │
      │          ┌──────────┴──────────┐                          │
      │          ▼                     ▼                          │
      │   MANAGED NODE 1         MANAGED NODE 2                   │
      │   node1 172.31.16.19     node2 172.31.20.45               │
      │   (SG: ansible-managed-sg)                                │
      │   public key in ~/.ssh/authorized_keys, python3           │
      └───────────────────────────────────────────────────────────┘
```

## 2.2 Components and their purposes

| Component | Purpose |
|---|---|
| Control node | Runs Ansible commands and playbooks |
| Managed nodes | Receive and run the tasks Ansible sends |
| Inventory | File listing the managed nodes |
| SSH private key | Kept on the control node; proves who we are |
| SSH public key | Sits on each managed node in `~/.ssh/authorized_keys`; lets the matching private key in |
| `ansible.cfg` | Stores default Ansible settings |
| Module | One unit of work (`ping`, `command`, `package`, `service`) |

# 3. Prerequisites: what you need before starting

Check every item **on your laptop** before Section 4.

| # | You need | Check with | Good result |
|---|---|---|---|
| 1 | An AWS account (free tier is fine) | log in to the AWS console | you see the dashboard |
| 2 | **AWS CLI v2** installed | `aws --version` | `aws-cli/2.x.x ...` |
| 3 | AWS CLI **configured** with an IAM user that can manage EC2 | `aws sts get-caller-identity` | JSON with your `Account` and `Arn` |
| 4 | A **default region** | `aws configure get region` | e.g. `us-east-1` |
| 5 | A **default VPC** in that region | `aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text` | `vpc-0abc...` |
| 6 | An **SSH client** (`ssh`, `scp`) | `ssh -V` | `OpenSSH_...` (Windows: use **Git Bash**) |
| 7 | Basic Linux: `cd`, `vi`, `cat`, `chmod` | | see [Day 3](../day-03-linux-basics/README.md), [Day 4](../day-04-vim-users-permissions/README.md) |

**What if a check fails?**
- `aws: command not found` → install AWS CLI v2 from the AWS docs.
- `Unable to locate credentials` → run `aws configure` and enter your **Access Key ID**, **Secret Access Key** and **default region**.
- No default VPC → create one: `aws ec2 create-default-vpc`.

# 4. Create the EC2 servers with the AWS CLI

> **Important:** run **all of Section 4 in one terminal on your laptop**. Each step saves values (AMI ID, security group IDs, instance IDs) into **shell variables** like `$AMI_ID`. Later commands reuse them. If you close the terminal, see [Section 4.7](#47-new-terminal-load-all-ids-again).

## 4.0 Read this first: how `$` and variables work

**Definition:** **Shell variable** = a **box with a name** that stores a value in the current terminal.

| You want to | Write | Example |
|---|---|---|
| **put** a value in the box | `NAME=value` (no `$`, no spaces around `=`) | `CONTROL_SG=sg-0a2587a6c7da61e37` |
| **put a command's output** in the box | `NAME=$(command)` | `AMI_ID=$(aws ssm get-parameter ...)` |
| **use** what's in the box | `$NAME` | `--security-group-ids $CONTROL_SG` |
| **check** what's in the box | `echo $NAME` | `echo $CONTROL_SG` → `sg-0a2587a6c7da61e37` |

**The golden rule:** a `$` is followed by a **variable name**, never by a real ID.

| ✅ Correct | ✅ Also correct | ❌ Wrong |
|---|---|---|
| `--image-id $AMI_ID` | `--image-id ami-0c230065b91270bc6` | `--image-id $ami-0c230065b91270bc6` |
| `--security-group-ids $CONTROL_SG` | `--security-group-ids sg-0a2587a6c7da61e37` | `--security-group-ids $sg-0a2587a6c7da61e37` |

**Why is the wrong one wrong?** Bash reads `$ami` as a variable called `ami`. That variable doesn't exist, so it's **empty**, and AWS receives only `-0c230065b91270bc6`. These are real errors from my run:

| Error | Cause | Fix |
|---|---|---|
| `InvalidAMIID.Malformed: Invalid id: "-0c230065b91270bc6" (expecting "ami-...")` | wrote `$ami-0c23...` | `$AMI_ID` or `ami-0c23...` (no `$`) |
| `Value () for parameter groupId is invalid. The value cannot be empty` | wrote `$sg-0a25...`, or `$CONTROL_SG` was empty (new terminal) | `echo $CONTROL_SG` first; if empty, fill it again ([4.7](#47-new-terminal-load-all-ids-again)) |
| `echo $CONTROL_ID` prints nothing | variable was never filled in **this** terminal | fill it again ([4.7](#47-new-terminal-load-all-ids-again)) |

> **Habit:** before any command that uses `$SOMETHING`, run `echo $SOMETHING`. Empty line = stop and fill it first.

## 4.1 Step 1: Find the AMI and your public IP

**Definition:** **AMI (Amazon Machine Image)** = the OS image a server boots from. Each region has **different AMI IDs** for the same OS.

```bash
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)

echo "AMI: $AMI_ID   My IP: $MY_IP"
```

**What do these commands do?**

- `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/...`: AWS publishes the **latest Amazon Linux 2023 AMI ID** for every region as a public parameter. Reading it means we never hard-code an AMI ID that could be old or wrong for our region.
- `--query 'Parameter.Value' --output text`: from the JSON answer, keep only the AMI ID as plain text.
- `curl -s https://checkip.amazonaws.com`: asks AWS "what is my public IP?". We need it so that **only our laptop** may SSH into the control node.

Expected (your values will differ):

```
AMI: ami-0c230065b91270bc6   My IP: 2.82.252.24
```

If either one is empty, stop. Check `aws sts get-caller-identity` (AWS login) and your internet connection.

## 4.2 Step 2: Create an SSH key pair

**Definition:** **Key pair** = a **private key** (secret file, stays with you) + a **public key** (AWS places it on the server). The server lets you in **only if your private key matches its public key**. No password is sent.

```bash
aws ec2 create-key-pair \
  --key-name ansible-lab \
  --query 'KeyMaterial' --output text > ansible-lab.pem

chmod 400 ansible-lab.pem
ls -l ansible-lab.pem
```

**What does this do?**

- `create-key-pair --key-name ansible-lab`: AWS creates a key pair named `ansible-lab`. AWS keeps the **public** key; the **private** key is returned **only once**.
- `--query 'KeyMaterial' --output text > ansible-lab.pem`: saves that private key into the file `ansible-lab.pem` on your laptop. **If you lose it, you can't download it again**; you'd have to create a new key pair.
- `chmod 400 ansible-lab.pem`: makes the file **read-only for you only**. SSH refuses to use a private key that others can read (`UNPROTECTED PRIVATE KEY FILE`).

Expected:

```
-r-------- 1 you you 1679 Nov  2 10:00 ansible-lab.pem
```

> In this lab the **same key** is used for laptop → control node and control node → managed nodes, to keep it simple. The real-project way is shown in [Section 5.1](#51-step-1-copy-the-private-key-to-the-control-node).

## 4.3 Step 3: Create two security groups

**Definition:** **Security group (SG)** = a virtual firewall attached to an EC2 server. **Inbound rules** decide **who** may connect **in** and on **which port**. Everything not allowed is blocked.

We create **two** SGs so SSH is as locked down as possible:

| Security group | Inbound rule | Why |
|---|---|---|
| `ansible-control-sg` | SSH **22** from **my IP only** | only my laptop can log in to the control node |
| `ansible-managed-sg` | SSH **22** from **`ansible-control-sg`** | only the control node can SSH to the managed nodes |
| `ansible-managed-sg` | HTTP **80** from anywhere | so we can open nginx in a browser in Section 11 |

**3a. Control node security group:**

```bash
CONTROL_SG=$(aws ec2 create-security-group \
  --group-name ansible-control-sg \
  --description "Ansible control node" \
  --query 'GroupId' --output text)

echo "Control SG: $CONTROL_SG"

aws ec2 authorize-security-group-ingress \
  --group-id $CONTROL_SG --protocol tcp --port 22 --cidr ${MY_IP}/32
```

Expected: `Control SG: sg-...`, then a JSON block with `"Return": true`, `"FromPort": 22` and `"CidrIpv4": "<your-ip>/32"`.

**What does this do?**

- `create-security-group`: creates an empty SG in the default VPC and returns its ID (`sg-...`), saved in `$CONTROL_SG`.
- `authorize-security-group-ingress`: **adds an inbound rule**.
- `--protocol tcp --port 22`: allow SSH.
- `--cidr ${MY_IP}/32`: only from **one** IP address (`/32` = exactly one address), your laptop.

**3b. Managed nodes security group:**

```bash
MANAGED_SG=$(aws ec2 create-security-group \
  --group-name ansible-managed-sg \
  --description "Ansible managed nodes" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $MANAGED_SG --protocol tcp --port 22 --source-group $CONTROL_SG

aws ec2 authorize-security-group-ingress \
  --group-id $MANAGED_SG --protocol tcp --port 80 --cidr 0.0.0.0/0

echo "Control SG: $CONTROL_SG   Managed SG: $MANAGED_SG"
```

Expected: two JSON blocks with `"Return": true` (one for port 22 with `"ReferencedGroupInfo"` = the control SG, one for port 80), then:

```
Control SG: sg-0a2587a6c7da61e37   Managed SG: sg-08bc86c7cf682a5d7
```

> If you run `create-security-group` a second time you get `InvalidGroup.Duplicate`: the group already exists. Don't create it again; load its ID with [Section 4.7](#47-new-terminal-load-all-ids-again).

**What does this do?**

- `--source-group $CONTROL_SG`: instead of an IP, the source is **another security group**. Any server that has `ansible-control-sg` attached may connect on port 22. Even if the control node's IP changes, the rule still works.
- `--port 80 --cidr 0.0.0.0/0`: HTTP open to everyone, only for the nginx test page.

```
laptop (MY_IP) ──22──► [ansible-control-sg] control ──22──► [ansible-managed-sg] node1, node2
internet       ──80────────────────────────────────────► [ansible-managed-sg] nginx
internet       ──22──► ✗ blocked (only MY_IP can SSH, and only to the control node)
```

> **Outbound rules:** leave the default "allow all outbound". The control node must reach the internet on port **443** to download Ansible with `dnf`. If you limit outbound to only port 22, the install fails.

## 4.4 Step 4: Launch the control node

**Definition:** **`aws ec2 run-instances`** = the AWS CLI command that **creates and starts** EC2 servers.

```bash
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids $CONTROL_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-control}]' \
  --query 'Instances[0].InstanceId' --output text
```

**Same command with my real values** (no `$`, the IDs are typed directly):

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-0a2587a6c7da61e37 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-control}]' \
  --query 'Instances[0].InstanceId' --output text
```

Expected: one instance ID on the screen:

```
i-0111ec6253b296e51
```

> **Run it only once.** Every run creates **another** server. If you're not sure whether it worked, check first: `aws ec2 describe-instances --filters "Name=tag:Name,Values=ansible-control" "Name=instance-state-name,Values=pending,running" --query 'Reservations[].Instances[].InstanceId' --output text`

### Understand every option

| Option | Meaning |
|---|---|
| `--image-id $AMI_ID` | boot Amazon Linux 2023 (the AMI from Step 1) |
| `--instance-type t3.micro` | size: 2 vCPU, 1 GB RAM, free-tier eligible |
| `--key-name ansible-lab` | AWS puts the **public key** of `ansible-lab` into `/home/ec2-user/.ssh/authorized_keys` |
| `--security-group-ids $CONTROL_SG` | attach the control node firewall |
| `--tag-specifications ... Value=ansible-control` | the **Name** shown in the EC2 console |
| `--query 'Instances[0].InstanceId'` | from the big JSON output, keep only the instance ID (`Instances` is a list, `[0]` = first item) |
| `--output text` | print it as plain text |

We save the ID into a box in [Step 4.6](#46-step-6-save-the-ids-and-get-the-ip-addresses), once all 3 servers exist.

## 4.5 Step 5: Launch the two managed nodes

Same command; only the **security group** (`$MANAGED_SG`) and the **Name** tag change.

```bash
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids $MANAGED_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node1}]' \
  --query 'Instances[0].InstanceId' --output text

aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids $MANAGED_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node2}]' \
  --query 'Instances[0].InstanceId' --output text
```

**Same commands with my real values:**

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-08bc86c7cf682a5d7 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node1}]' \
  --query 'Instances[0].InstanceId' --output text
```

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-08bc86c7cf682a5d7 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node2}]' \
  --query 'Instances[0].InstanceId' --output text
```

Expected: two more IDs, one per line:

```
i-094cf172053ec2ed0
i-042255565324b6c10
```

> ⚠️ node1 and node2 use **`$MANAGED_SG`**, not `$CONTROL_SG`. A wrong SG here means the control node can't SSH to them later.

**Notice what we did NOT do:** we did not install anything on node1 or node2. Amazon Linux 2023 already has **SSH** and **python3**, which is all Ansible needs on a managed node.

| | Control node | node1 / node2 |
|---|---|---|
| Security group | `ansible-control-sg` | `ansible-managed-sg` |
| Ansible installed? | ✅ yes (Section 5) | ❌ no (agentless) |
| Needs | Ansible, private key, inventory, `ansible.cfg` | SSH + python3 (already there) |

> **Prefer the AWS console?** EC2 → **Launch instance** → Name `ansible-control` → AMI **Amazon Linux 2023** → type `t3.micro` → key pair `ansible-lab` → **Select existing security group** `ansible-control-sg` → Launch. Repeat for `node1` and `node2` with `ansible-managed-sg`. Same result.

## 4.6 Step 6: Save the IDs and get the IP addresses

**Definition:**
- **Public IP** = reachable from the internet. Used for: laptop → control node, browser → nginx.
- **Private IP** = reachable only inside the VPC. Used for: control node → managed nodes (faster, free, not exposed).

**1. Wait until all 3 servers are running:**

```bash
aws ec2 wait instance-running \
  --filters "Name=tag:Name,Values=ansible-control,node1,node2" "Name=instance-state-name,Values=pending,running" \
  && echo "All 3 running"
```

**What does `aws ec2 wait instance-running` do?** It **pauses** until the servers reach the `running` state, then prints `All 3 running`. Without it, the next commands might ask for IPs before the servers have them.

**2. Show all 3 servers with their IPs:**

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ansible-control,node1,node2" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value, InstanceId, PrivateIpAddress, PublicIpAddress]' \
  --output table
```

**What does this do?** `describe-instances` returns everything about your servers. `--filters` keeps only our 3 running servers (by **Name** tag). `--query` picks **Name, instance ID, private IP, public IP**, and `--output table` prints them neatly.

Expected:

```
-----------------------------------------------------------------------------
|                             DescribeInstances                             |
+-----------------+-----------------------+---------------+-----------------+
|  ansible-control|  i-0111ec6253b296e51  |  172.31.30.10 |  54.210.11.22   |
|  node1          |  i-0bbb...            |  172.31.16.19 |  3.88.44.55     |
|  node2          |  i-0ccc...            |  172.31.20.45 |  18.207.66.77   |
+-----------------+-----------------------+---------------+-----------------+
```

> Each name must appear **once**. If a name appears twice, you created it twice: terminate the extra one (`aws ec2 terminate-instances --instance-ids <extra-id>`).

**3. Save the IDs and the control node's public IP into boxes** (Ansible needs them in Section 5; cleanup needs them in Section 14). Run [Section 4.7](#47-new-terminal-load-all-ids-again) now. It loads everything by name, with no copy-paste.

**4. Write your values down.** The rest of this lab uses the example IPs `172.31.16.19` (node1) and `172.31.20.45` (node2). **Replace them with yours.**

| Server | Instance ID | Private IP | Public IP |
|---|---|---|---|
| ansible-control | | | |
| node1 | | | |
| node2 | | | |

## 4.7 New terminal? Load all IDs again

Variables live only in the terminal where you set them. In a **new terminal** (or after a mistake), this block **looks everything up in AWS by name** and fills all the boxes. It's safe to run any time; it doesn't create anything.

```bash
AMI_ID=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query 'Parameter.Value' --output text)
MY_IP=$(curl -s https://checkip.amazonaws.com)

CONTROL_SG=$(aws ec2 describe-security-groups --group-names ansible-control-sg --query 'SecurityGroups[0].GroupId' --output text)
MANAGED_SG=$(aws ec2 describe-security-groups --group-names ansible-managed-sg --query 'SecurityGroups[0].GroupId' --output text)

FILTER="Name=instance-state-name,Values=pending,running,stopped"
CONTROL_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=ansible-control" "$FILTER" --query 'Reservations[0].Instances[0].InstanceId' --output text)
NODE1_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=node1" "$FILTER" --query 'Reservations[0].Instances[0].InstanceId' --output text)
NODE2_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=node2" "$FILTER" --query 'Reservations[0].Instances[0].InstanceId' --output text)
CONTROL_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $CONTROL_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "AMI_ID=$AMI_ID  MY_IP=$MY_IP"
echo "CONTROL_SG=$CONTROL_SG  MANAGED_SG=$MANAGED_SG"
echo "CONTROL_ID=$CONTROL_ID  NODE1_ID=$NODE1_ID  NODE2_ID=$NODE2_ID"
echo "CONTROL_PUBLIC_IP=$CONTROL_PUBLIC_IP"
```

| You see | Means |
|---|---|
| a real value (`sg-...`, `i-...`, an IP) | ✅ box is filled |
| `None` | that thing doesn't exist yet (e.g. you haven't launched `node1`) → create it first |
| `InvalidGroup.NotFound` | the security group isn't created yet → do Step 3 |

**Or fill a box by hand**, with the real value and no `$` on the left:

```bash
CONTROL_ID=i-0111ec6253b296e51
echo $CONTROL_ID
```

### My lab values (from my run, us-east-1)

These are the real IDs from my lab, ready to paste. **Your IDs will be different**: use the block above to load yours.

```bash
AMI_ID=ami-0c230065b91270bc6          # Amazon Linux 2023
CONTROL_SG=sg-0a2587a6c7da61e37       # ansible-control-sg
MANAGED_SG=sg-08bc86c7cf682a5d7       # ansible-managed-sg
CONTROL_ID=i-0111ec6253b296e51        # ansible-control
NODE1_ID=i-094cf172053ec2ed0          # node1
NODE2_ID=i-042255565324b6c10          # node2
MY_IP=$(curl -s https://checkip.amazonaws.com)

echo "AMI=$AMI_ID  CONTROL_SG=$CONTROL_SG  MANAGED_SG=$MANAGED_SG  CONTROL_ID=$CONTROL_ID  NODE1_ID=$NODE1_ID  NODE2_ID=$NODE2_ID  MY_IP=$MY_IP"
```

| Name | Type | ID |
|---|---|---|
| Amazon Linux 2023 | AMI | `ami-0c230065b91270bc6` |
| `ansible-lab` | key pair | `ansible-lab.pem` on my laptop |
| `ansible-control-sg` | security group | `sg-0a2587a6c7da61e37` |
| `ansible-managed-sg` | security group | `sg-08bc86c7cf682a5d7` |
| `ansible-control` | EC2 instance | `i-0111ec6253b296e51` |
| `node1` | EC2 instance | `i-094cf172053ec2ed0` |
| `node2` | EC2 instance | `i-042255565324b6c10` |

# 5. Prepare the control node

## 5.1 Step 1: Copy the private key to the control node

**Why?** The control node will SSH into node1 and node2. To do that, it needs the **private key** that matches the public key AWS put on them. Right now that key is only on your laptop.

```bash
# on your LAPTOP
scp -i ansible-lab.pem ansible-lab.pem ec2-user@$CONTROL_PUBLIC_IP:/home/ec2-user/.ssh/ansible_lab
```

**What does this do?** `scp` = **secure copy** over SSH. `-i ansible-lab.pem` = log in with this key; then copy the file `ansible-lab.pem` to the control node as `/home/ec2-user/.ssh/ansible_lab`. The first time, it asks `Are you sure you want to continue connecting (yes/no)?` → type **`yes`**.

Now log in to the control node:

```bash
ssh -i ansible-lab.pem ec2-user@$CONTROL_PUBLIC_IP
```

**Everything from here runs on the control node** (the prompt shows `[ec2-user@ip-172-31-30-10 ~]$`).

```bash
chmod 400 ~/.ssh/ansible_lab
ls -lah ~/.ssh/
```

Expected:

```
-r--------. 1 ec2-user ec2-user 1.7K  ansible_lab
-rw-------. 1 ec2-user ec2-user  391  authorized_keys
```

Here:

- `ansible_lab` = **private key**. The control node uses it to log in to node1 and node2.
- `authorized_keys` = **public keys** allowed to log in **to this** server (AWS put the `ansible-lab` public key here).
- `known_hosts` (appears after the first SSH) = servers we have already trusted.

The private key must **match** the public key on the managed nodes. Here it does, because all 3 servers were launched with `--key-name ansible-lab`.

> 💡 **Real-project way:** don't copy your personal key around. On the control node, create a dedicated Ansible key with `ssh-keygen -t ed25519 -f ~/.ssh/ansible_lab -N ""`, then add `~/.ssh/ansible_lab.pub` to `~/.ssh/authorized_keys` on every managed node (for example with EC2 user-data, or `ssh-copy-id`). Then `ls ~/.ssh` shows both `ansible_lab` (private) and `ansible_lab.pub` (public).

## 5.2 Step 2: Install Ansible

**Definition:** **`ansible-core`** = the Ansible engine + the built-in modules (`ansible.builtin`: `ping`, `command`, `shell`, `package`, `service`, `copy`, `debug`, ...).

```bash
sudo dnf install ansible-core -y
ansible --version
```

**What does this do?** `dnf` is the package manager of Amazon Linux 2023. `sudo` runs it as root (installing software needs root). `-y` answers "yes" automatically. `ansible --version` proves it installed and shows which config file is in use.

Expected (your version may be newer):

```
ansible [core 2.15.3]
  config file = None
  configured module search path = [...]
  python version = 3.9.x
```

**What does `config file = None` mean?** Ansible looked for an `ansible.cfg` and found **none**. That's expected; we haven't created one yet. We'll watch this line change in Section 8.

# 6. Verify connectivity: TCP and SSH

Before running Ansible, check the network in **two separate steps**. This tells you exactly **which layer** is broken if something fails.

| Check | Answers | Tool |
|---|---|---|
| **TCP** | Is port 22 **reachable**? (security group, routing) | `telnet` |
| **SSH login** | Does the **key** work? (authentication) | `ssh -i` |

## 6.1 TCP check with telnet

```bash
sudo dnf install telnet -y
telnet 172.31.16.19 22
```

**What does this do?** `telnet <ip> <port>` just tries to **open a TCP connection** to that port. If the SG allows it and something is listening, it connects.

Expected:

```
Trying 172.31.16.19...
Connected to 172.31.16.19.
Escape character is '^]'.
SSH-2.0-OpenSSH_8.7
```

`SSH-2.0-OpenSSH...` is the SSH server saying hello. Exit with **`Ctrl + ]`**, then type **`quit`**.

> ⚠️ `Connected` proves **only TCP connectivity**: the port is open. It does **not** prove the key works. If it hangs on `Trying...` and times out, the **security group** is blocking (see [Section 12](#12-troubleshooting-scenarios)).

## 6.2 SSH login check (both nodes)

```bash
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.16.19
hostname
exit

ssh -i ~/.ssh/ansible_lab ec2-user@172.31.20.45
hostname
exit
```

**What does this do?** Logs in to each managed node with the private key. The first time per node, SSH asks `Are you sure you want to continue connecting (yes/no)?` → type **`yes`**. That saves the node's **host key** in `~/.ssh/known_hosts`, so Ansible won't get stuck on this question later.

✅ **Checkpoint:** if you can log in to both nodes by hand, Ansible will be able to as well. Ansible uses the same SSH underneath.

# 7. Approach 1: Ansible without `ansible.cfg`

## 7.1 Introduction

In this approach we **do not create** an `ansible.cfg` file. Every Ansible command must carry all the settings itself, as command-line flags. We do it this way first so we feel **why** `ansible.cfg` exists.

## 7.2 Step 1: Create a project directory

```bash
mkdir ~/ansible-raw-lab
cd ~/ansible-raw-lab
pwd
```

**What does this do?** Creates a folder for our Ansible project and moves into it. `pwd` (print working directory) confirms where we are.

Expected:

```
/home/ec2-user/ansible-raw-lab
```

## 7.3 Step 2: Create the inventory

**Definition:** **Inventory** = a file that tells Ansible **which machines** it should manage, optionally grouped.

```bash
mkdir inventory
vi inventory/hosts
```

Press `i` to type, add the following, then `Esc` and `:wq` to save:

```ini
[managed_nodes]
node1 ansible_host=172.31.16.19
node2 ansible_host=172.31.20.45
```

### What does each part mean?

| Part | Meaning |
|---|---|
| `[managed_nodes]` | **group name**; we can target the whole group at once |
| `node1` | the **inventory hostname**: a short, friendly name shown in Ansible's output |
| `ansible_host=172.31.16.19` | the **real address** Ansible connects to (node1's **private** IP) |
| `node2 ansible_host=172.31.20.45` | the second managed node |

> Why `node1 ansible_host=<ip>` instead of just the IP? Output says `node1 | SUCCESS` instead of a hard-to-read IP, and if the IP changes you edit one place only.

## 7.4 Step 3: Verify the inventory

```bash
cat inventory/hosts
ansible-inventory -i inventory/hosts --graph
```

**What does this do?** `cat` shows the file. `ansible-inventory --graph` shows how **Ansible understood** it, as a tree. It only **reads the file**; it does not connect to the servers.

Expected:

```
@all:
  |--@ungrouped:
  |--@managed_nodes:
  |  |--node1
  |  |--node2
```

`@` marks a **group**. `@all` always contains every host.

## 7.5 Step 4: Run the Ansible ping module (the long way)

```bash
ansible all \
  -i /home/ec2-user/ansible-raw-lab/inventory/hosts \
  -u ec2-user \
  --private-key /home/ec2-user/.ssh/ansible_lab \
  -m ping
```

### Understand every argument

| Argument | Meaning |
|---|---|
| `ansible` | run an **ad-hoc command** (one quick task, no playbook) |
| `all` | target **all hosts** in the inventory |
| `-i <path>` | **which inventory file** to use |
| `-u ec2-user` | **SSH username** on the managed nodes (Amazon Linux default user) |
| `--private-key <path>` | **which private key** to log in with |
| `-m ping` | run the **ping module** |

### Expected output

```
node1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### What does this output mean?

- `SUCCESS`: Ansible connected to the node and ran the module.
- `"changed": false`: nothing on the server was modified (ping only checks).
- `"ping": "pong"`: the module ran correctly and replied.

**Important:** Ansible's `ping` module is **not** the Linux `ping` (ICMP). It logs in over **SSH**, copies a small **Python** program to the node, runs it and gets `pong` back. So `pong` proves **network + SSH login + Python** all work.

> You may also see `"discovered_interpreter_python": "/usr/bin/python3..."` or a warning about the Python interpreter. That's harmless; Ansible is just telling you which Python it found.

## 7.6 Why is this approach repetitive?

Let's run two more modules:

```bash
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m command -a "uptime"
```

```bash
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m shell -a "df -h"
```

**What do these do?** `-m command -a "uptime"` runs `uptime` on both nodes (how long they've been running). `-m shell -a "df -h"` runs `df -h` (disk usage). `-a` = the **arguments** for the module.

**Notice what repeats every single time:**

1. the inventory path (`-i ...`)
2. the SSH username (`-u ec2-user`)
3. the private key path (`--private-key ...`)

**Only the module (`-m`) and its arguments (`-a`) change.** With many servers and many commands, this is slow, hard to read, and easy to get wrong. That's the problem `ansible.cfg` solves.

# 8. Approach 2: Ansible with `ansible.cfg`

## 8.1 Introduction

**Definition:** **`ansible.cfg`** = Ansible's **configuration file**. It stores **default values** (inventory, user, key, ...) that Ansible discovers automatically and applies to every command.

## 8.2 Step 1: Understand the project structure

Now:

```
ansible-raw-lab/
└── inventory/
    └── hosts
```

After this section:

```
ansible-raw-lab/
├── ansible.cfg        ← new
└── inventory/
    └── hosts
```

## 8.3 Step 2: Create `ansible.cfg`

```bash
cd /home/ec2-user/ansible-raw-lab
vi ansible.cfg
```

Add:

```ini
[defaults]
inventory = ./inventory/hosts
remote_user = ec2-user
private_key_file = /home/ec2-user/.ssh/ansible_lab
host_key_checking = False
```

### Understand every setting

| Setting | Replaces the flag | Meaning |
|---|---|---|
| `[defaults]` | | the main section for general Ansible settings |
| `inventory = ./inventory/hosts` | `-i` | default inventory file |
| `remote_user = ec2-user` | `-u` | default SSH username |
| `private_key_file = ...` | `--private-key` | default SSH private key |
| `host_key_checking = False` | typing `yes` | don't stop and ask "Are you sure you want to continue connecting?" for new servers |

**Security note:** turning off host key checking is fine for **throw-away labs** where servers are created and deleted often. In production, keep it **on** and manage trusted host keys (`known_hosts`). The check protects you from connecting to a fake server pretending to be yours.

## 8.4 Step 3: Verify the configuration file

```bash
cat ansible.cfg
```

Expected: the same 5 lines you typed.

## 8.5 Step 4: Verify Ansible detects it

```bash
ansible --version
```

Expected:

```
ansible [core 2.15.3]
  config file = /home/ec2-user/ansible-raw-lab/ansible.cfg
```

**What changed?** In Section 5.2 this line said `config file = None`. Now Ansible found our file **automatically**, because it's in the **current folder**. ([Section 9](#9-how-ansible-finds-ansiblecfg-config-discovery) explains the search order.)

To see **exactly which settings** your file changed from Ansible's defaults:

```bash
ansible-config dump --only-changed
```

Expected (similar):

```
CONFIG_FILE() = /home/ec2-user/ansible-raw-lab/ansible.cfg
DEFAULT_HOST_LIST(...) = ['/home/ec2-user/ansible-raw-lab/inventory/hosts']
DEFAULT_PRIVATE_KEY_FILE(...) = /home/ec2-user/.ssh/ansible_lab
DEFAULT_REMOTE_USER(...) = ec2-user
HOST_KEY_CHECKING(...) = False
```

## 8.6 Step 5: Run ping again, the short way

Before (Section 7.5):

```bash
ansible all \
  -i /home/ec2-user/ansible-raw-lab/inventory/hosts \
  -u ec2-user \
  --private-key /home/ec2-user/.ssh/ansible_lab \
  -m ping
```

Now:

```bash
ansible all -m ping
ansible all -m command -a "uptime"
ansible all -m shell -a "df -h"
```

Expected: the same `SUCCESS` / `pong` for node1 and node2.

**What happened?** Ansible automatically took:

- the **inventory path** from `ansible.cfg`,
- the **SSH username** from `ansible.cfg`,
- the **private key** from `ansible.cfg`.

We no longer type these on every command.

## 8.7 Step 6: The command line still wins

Settings in `ansible.cfg` are **defaults**. A flag on the command line **overrides** them. Let's prove it with a second inventory that has only node1:

```bash
printf "[managed_nodes]\nnode1 ansible_host=172.31.16.19\n" > inventory/only-node1
ansible all -i inventory/only-node1 -m ping
```

**What does this do?** `printf` writes a 2-line inventory file. Running with `-i inventory/only-node1` makes Ansible ignore the `inventory` in `ansible.cfg`, so **only node1** answers. The same is true for `-u` and `--private-key`.

# 9. How Ansible finds `ansible.cfg` (config discovery)

**Definition:** **Config discovery** = the order in which Ansible **searches** for its configuration file. It uses the **first file it finds** and **ignores the rest**. Files are **never merged**.

| # | Ansible checks | Example |
|---|---|---|
| 1 | the **`ANSIBLE_CONFIG`** environment variable | `/home/ec2-user/ansible-raw-lab/ansible.cfg` |
| 2 | `ansible.cfg` in the **current working directory** | `./ansible.cfg` |
| 3 | `.ansible.cfg` in your **home directory** (note the dot) | `~/.ansible.cfg` |
| 4 | the **system-wide** file | `/etc/ansible/ansible.cfg` |

```
             ansible all -m ping
                      │
                      ▼
   ANSIBLE_CONFIG set AND that file exists? ── yes ──► use it
                      │ no
                      ▼
   ./ansible.cfg exists?                    ── yes ──► use it
                      │ no
                      ▼
   ~/.ansible.cfg exists?                   ── yes ──► use it
                      │ no
                      ▼
   /etc/ansible/ansible.cfg exists?         ── yes ──► use it
                      │ no
                      ▼
   config file = None  (Ansible's built-in defaults)
                      │
                      ▼
   read inventory ─► SSH login ─► run module ─► SUCCESS / FAILURE
```

**Try it yourself:**

```bash
cd ~
ansible --version      # config file = None  (no ansible.cfg in /home/ec2-user)
cd ~/ansible-raw-lab
ansible --version      # config file = /home/ec2-user/ansible-raw-lab/ansible.cfg
```

**Lesson:** the folder you run Ansible **from** decides which `ansible.cfg` is used. Always `cd` into the project folder first, or use `ANSIBLE_CONFIG` (next section).

# 10. Environment variables: temporary and permanent `ANSIBLE_CONFIG`

## 10.1 What is an environment variable?

**Definition:** **Environment variable** = a named value stored in the shell's environment. Every program started from that shell can read it and change its behaviour.

Ansible reads the environment variable **`ANSIBLE_CONFIG`**: it tells Ansible **exactly which config file to use**, no matter which folder you are in. It's #1 in the search order.

## 10.2 Temporary: `export` (this terminal only)

Suppose we are in the home folder, but the config file is in the project folder:

```bash
cd ~
export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg
echo $ANSIBLE_CONFIG
```

**What does `export` do?** It sets the variable in the **current shell** and passes it to every program started from it, including `ansible`.

Expected:

```
/home/ec2-user/ansible-raw-lab/ansible.cfg
```

Now check Ansible, still from `~`:

```bash
ansible --version       # config file = /home/ec2-user/ansible-raw-lab/ansible.cfg
ansible all -m ping     # works from the home folder
```

> **Why does `inventory = ./inventory/hosts` still work from `~`?** Relative paths in `ansible.cfg` are resolved **relative to the folder of the `ansible.cfg` file**, not to your current folder.

**What makes it temporary?** The variable lives only in **this** shell. Close the terminal (or log out of the EC2 server) and it's gone.

## 10.3 Remove it: `unset`

```bash
unset ANSIBLE_CONFIG
echo $ANSIBLE_CONFIG          # empty line
ansible --version             # config file = None   (we are still in ~)

cd /home/ec2-user/ansible-raw-lab
ansible --version             # config file = /home/ec2-user/ansible-raw-lab/ansible.cfg
```

**Why does the last one still find the file?** No environment variable is needed when you are **inside** the project folder: rule #2 (current directory) finds `./ansible.cfg`.

## 10.4 Permanent: add it to `~/.bashrc`

**Definition:** **`~/.bashrc`** = a script that bash runs **automatically every time a new interactive terminal starts**. Put `export` lines in it to set variables for every new session.

**Step 1: add the line** (at the bottom of the file):

```bash
echo 'export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg' >> ~/.bashrc
tail -1 ~/.bashrc
```

`>>` **appends** to the end of the file. ⚠️ A single `>` would **overwrite** the whole `.bashrc`! (You can also open it with `vi ~/.bashrc` and add the line by hand.)

**Step 2: reload it in the current shell:**

```bash
source ~/.bashrc
```

**What does `source` do?** It runs the contents of `.bashrc` **inside the current shell**, so the new variable is active right now. **No EC2 reboot and no re-login needed.**

**Step 3: verify:**

```bash
echo $ANSIBLE_CONFIG
printenv ANSIBLE_CONFIG       # another way to print it
ansible --version             # config file = /home/ec2-user/ansible-raw-lab/ansible.cfg
```

From now on, every **new** terminal on this server sets `ANSIBLE_CONFIG` automatically.

## 10.5 Gotcha: deleting the line does NOT remove the variable

```bash
echo 'export DEMO=HELLO' >> ~/.bashrc
source ~/.bashrc
echo $DEMO                     # HELLO

sed -i '/export DEMO=HELLO/d' ~/.bashrc   # remove the line from .bashrc
source ~/.bashrc
echo $DEMO                     # still HELLO!
```

**Why is it still there?** `source` only **runs** what's in the file. Removing a line doesn't **un-set** a variable that already exists in the running shell. You must remove it yourself:

```bash
unset DEMO
echo $DEMO                     # empty
```

The same applies to `ANSIBLE_CONFIG`: to fully undo it, **delete the line from `.bashrc` AND run `unset ANSIBLE_CONFIG`** (or open a new terminal).

## 10.6 Temporary vs permanent

| | Temporary | Permanent |
|---|---|---|
| How | `export VAR=value` | add the `export` line to `~/.bashrc` |
| Current terminal | ✅ | ✅ after `source ~/.bashrc` |
| New terminal | ❌ | ✅ |
| Reboot needed | no | no |
| Remove | `unset VAR` | delete the line **and** `unset VAR` |
| Use for | quick tests | daily setup |

**Important:** `.bashrc` is read only by **interactive bash** terminals. System services, cron jobs and CI/CD pipelines **don't** read it; set the variable in those places separately.

**Before moving on**, remove it so the next sections use the normal folder search:

```bash
sed -i '/ANSIBLE_CONFIG/d' ~/.bashrc && unset ANSIBLE_CONFIG
```

# 11. Use it: install nginx on both nodes with a playbook

**Definition:** **Playbook** = a **YAML** file with one or more **plays**. Each play says **which hosts** and **which tasks** (modules) to run on them. (YAML basics: [Day 20](../day-20-ansible-data-formats-yaml/README.md#student-details-in-yaml).)

Now that `ansible.cfg` is set up, let's do real configuration work.

```bash
cd ~/ansible-raw-lab
mkdir playbooks
vi playbooks/nginx.yaml
```

```yaml
- name: install and start nginx
  hosts: managed_nodes
  become: yes                  # run tasks as root (sudo)
  tasks:
  - name: install nginx
    package:
      name: nginx
      state: present

  - name: start and enable nginx
    service:
      name: nginx
      state: started
      enabled: yes
```

### What does each part do?

| Line | Meaning |
|---|---|
| `hosts: managed_nodes` | run on the group from `inventory/hosts` (node1 + node2) |
| `become: yes` | use **sudo**; installing packages and starting services needs root |
| `package: name=nginx state=present` | make sure nginx is **installed** (uses `dnf` on Amazon Linux) |
| `service: state=started` | make sure nginx is **running** (`systemctl start nginx`) |
| `service: enabled=yes` | make sure nginx **starts on boot** (`systemctl enable nginx`) |

Run it:

```bash
ansible-playbook playbooks/nginx.yaml
```

**Notice:** no `-i`, no `-u`, no `--private-key`. All of it comes from `ansible.cfg`.

Expected (short):

```
PLAY [install and start nginx] *************************
TASK [Gathering Facts]       ok: [node1]  ok: [node2]
TASK [install nginx]         changed: [node1]  changed: [node2]
TASK [start and enable nginx] changed: [node1]  changed: [node2]

PLAY RECAP *********************************************
node1 : ok=3  changed=2  unreachable=0  failed=0
node2 : ok=3  changed=2  unreachable=0  failed=0
```

**Run it a second time** → `changed=0`. nginx is already installed and running, so Ansible does nothing. That's **idempotency**: running the same playbook again is safe.

**Check nginx is working:**

| Where | Command | Expected |
|---|---|---|
| Control node | `curl -I http://172.31.16.19` | `HTTP/1.1 200 OK`, `Server: nginx` |
| Control node | `ansible all -m command -a "systemctl is-active nginx"` | `active` for node1 and node2 |
| Your browser | `http://<node1-public-ip>` | nginx welcome page |

```
ansible.cfg + inventory ──► ansible-playbook nginx.yaml ──SSH 22──► node1, node2: nginx running
browser ───────────────────── HTTP 80 (ansible-managed-sg) ───────► node1 / node2
```

# 12. Troubleshooting scenarios

## Scenario 1: Config file not detected

```
config file = None
```

**Possible reasons:** no `ansible.cfg` in the searched places; you're in the **wrong folder**; `ANSIBLE_CONFIG` points to a file that doesn't exist.

**Check:**

```bash
pwd
ls -l ansible.cfg
echo $ANSIBLE_CONFIG
ansible --version
```

**Fix:** `cd ~/ansible-raw-lab`, or correct `ANSIBLE_CONFIG`.

## Scenario 2: Wrong `ANSIBLE_CONFIG` path

`.bashrc` has a typo:

```bash
export ANSIBLE_CONFIG=/home/ec2-user/ansible-lab/ansible.cfg        # wrong folder
```

but the real file is `/home/ec2-user/ansible-raw-lab/ansible.cfg`.

**What happens?** Ansible checks `ANSIBLE_CONFIG` first, sees the file **doesn't exist**, and **silently moves on** to the next place in the search order (current folder, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`). There's **no error**, so you may unknowingly run with a **different** config file, or with none.

**Fix:**

```bash
echo $ANSIBLE_CONFIG                 # spot the wrong path
unset ANSIBLE_CONFIG                 # remove it now
vi ~/.bashrc                         # fix or delete the line
cd /home/ec2-user/ansible-raw-lab
ansible --version                    # confirm the right file
```

> **Habit:** whenever something looks odd, run `ansible --version` and read the `config file =` line first.

## Scenario 3: `ansible.cfg` ignored, "world writable directory"

```
[WARNING]: Ansible is being run in a world writable directory (/home/ec2-user/ansible-raw-lab), ignoring it as an ansible.cfg source.
```

**Reason:** the folder has `777` permissions, so anyone could plant a malicious `ansible.cfg`. Ansible refuses to trust it.

**Fix:** `chmod 755 ~/ansible-raw-lab`

## Scenario 4: Private key not found

```
no such identity: /home/ec2-user/.ssh/ansible_lab: No such file or directory
```

**Reason:** `private_key_file` in `ansible.cfg` points to a path where no key exists.

**Fix:** list the real key name and put that exact path in `ansible.cfg`:

```bash
ls -lah ~/.ssh/
vi ansible.cfg        # private_key_file = /home/ec2-user/.ssh/<real-key-name>
```

## Scenario 5: Permission denied (publickey)

```
Permission denied (publickey,gssapi-keyex,gssapi-with-mic)
```

**Possible causes:** wrong private key; wrong username (Amazon Linux = `ec2-user`, Ubuntu = `ubuntu`); the public key is not on the managed node; key permissions too open.

**Check:**

```bash
chmod 400 ~/.ssh/ansible_lab
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.16.19     # test without Ansible
```

If plain `ssh` fails, the problem is SSH, not Ansible. Fix SSH first.

## Scenario 6: Inventory not detected

```
[WARNING]: provided hosts list is empty, only localhost is available
[WARNING]: Could not match supplied host pattern, ignoring: managed_nodes
```

**Check:**

```bash
ansible-inventory --graph
ansible-config dump --only-changed       # is DEFAULT_HOST_LIST set?
```

Expected:

```
@all:
  |--@ungrouped:
  |--@managed_nodes:
  |  |--node1
  |  |--node2
```

**Fix:** correct the `inventory =` path in `ansible.cfg`, or the group name in your command/playbook.

## Scenario 7: TCP connectivity failure (timeout)

```
node1 | UNREACHABLE! => ... "ssh: connect to host 172.31.16.19 port 22: Connection timed out"
```

**Check, in this order:**

1. `ansible-managed-sg` inbound allows **22** from **`ansible-control-sg`**.
2. The control node actually has `ansible-control-sg` attached.
3. You used the **private** IP in the inventory.
4. Network ACLs / route tables (default VPC: normally fine).
5. `telnet 172.31.16.19 22` → `Connected`?

| Error | Means |
|---|---|
| `Connection timed out` | packets are **blocked** (security group / network) |
| `Connection refused` | the server is reachable but **nothing listens** on 22 (sshd down, or still booting: wait a minute) |

Remember: `telnet` `Connected` = **TCP OK**, not **login OK**.

## Scenario 8: nginx page doesn't open in the browser

| Symptom | Cause | Fix |
|---|---|---|
| browser keeps loading, then times out | port 80 not allowed | `ansible-managed-sg` inbound **80** |
| connection refused | nginx not running | re-run the playbook |
| wrong page | wrong IP | use the node's **public** IP |

# 13. Configuration precedence vs discovery

These are **two different ideas** that are easy to mix up:

| | Config **discovery** | Config **precedence** |
|---|---|---|
| Question it answers | **Which file** is loaded? | **Which value wins** for one setting? |
| Order | `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg` | **command line** → **environment variable** → **`ansible.cfg`** → **built-in default** |

**Precedence example:**

`ansible.cfg` says:

```ini
[defaults]
remote_user = ec2-user
```

and you run:

```bash
ansible all -m ping -u root
```

`-u root` on the command line **wins** over `remote_user = ec2-user`. (On Amazon Linux, AWS blocks direct `root` login, so this ping fails. That failure is the proof that `root` was used.)

**Real-world use of precedence:** one `ansible.cfg` for the project, a different inventory per environment chosen with `-i`:

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── dev/hosts
│   ├── staging/hosts
│   └── production/hosts
├── playbooks/
│   ├── install_nginx.yml
│   └── deploy_application.yml
└── roles/
    ├── nginx/
    └── monitoring/
```

```bash
ansible all -i inventory/dev/hosts -m ping
ansible all -i inventory/staging/hosts -m ping
ansible all -i inventory/production/hosts -m ping
```

`-i` overrides the default inventory in `ansible.cfg`; everything else (user, key) still comes from the file.

# 14. Clean up: delete everything

**Why?** EC2 servers cost money while they run, even when you're not using them. Delete everything the lab created.

Run from your **laptop**. First check the boxes are filled: `echo $CONTROL_ID $NODE1_ID $NODE2_ID $CONTROL_SG $MANAGED_SG`. If any are empty, run [Section 4.7](#47-new-terminal-load-all-ids-again) first.

```bash
# 1. delete the 3 servers and wait until they are fully gone
aws ec2 terminate-instances --instance-ids $CONTROL_ID $NODE1_ID $NODE2_ID
aws ec2 wait instance-terminated --instance-ids $CONTROL_ID $NODE1_ID $NODE2_ID

# 2. delete the security groups (managed first)
aws ec2 delete-security-group --group-id $MANAGED_SG
aws ec2 delete-security-group --group-id $CONTROL_SG

# 3. delete the key pair (AWS side + local file)
aws ec2 delete-key-pair --key-name ansible-lab
rm -f ansible-lab.pem
```

**Why this order?**

| Order | Reason |
|---|---|
| terminate, then **wait** | a security group can't be deleted while a server still uses it |
| managed SG **before** control SG | the managed SG's rule points to the control SG; AWS won't delete a group that another rule references |
| key pair last | not needed once servers are gone |

## 14.1 Lost your variables?

If you closed the terminal, `$CONTROL_ID`, `$MANAGED_SG` etc. are empty. Run the block in [Section 4.7](#47-new-terminal-load-all-ids-again) to load them all again by name, check that the `echo` lines show real values, then run the cleanup above.

# 15. Comparison tables

## 15.1 Without vs with `ansible.cfg`

| | Without `ansible.cfg` | With `ansible.cfg` |
|---|---|---|
| Inventory | `-i` every time | configured once |
| SSH username | `-u` every time | configured once |
| Private key | `--private-key` every time | configured once |
| Host key prompt | answer `yes` per new server | `host_key_checking` (lab only) |
| Command length | long | `ansible all -m ping` |
| Repetition / typo risk | high | low |
| Team use | everyone must know all flags | settings live **in the repo**, shared by all |

## 15.2 This lab vs the Day 20 class setup

| | [Day 20 class](../day-20-ansible-data-formats-yaml/README.md) | This lab |
|---|---|---|
| Servers | `ansible-server` + `roboshop-frontend` | `ansible-control` + `node1` + `node2` |
| Security group | one shared `roboshop-common` | two SGs: SSH only **laptop → control → nodes** |
| Login to nodes | **password**: `-e ansible_user=ec2-user -e ansible_password=...` (needs `sshpass` on the control node) | **SSH key**: `private_key_file` in `ansible.cfg` |
| Inventory | `inventory.ini`: IPs, groups, `[group:vars]` | `inventory/hosts`: aliases `node1 ansible_host=<ip>` |
| Settings | flags on every command | `ansible.cfg` → short commands |
| Focus | YAML, playbooks, variables | config file, discovery, env vars, troubleshooting |

Both work. **Key-based login + `ansible.cfg`** is how real teams run Ansible; Day 20 covers what goes **inside** the playbooks.

# 16. Final summary: the whole lab in order

### Phase 1: AWS setup (laptop)
1. Check prerequisites: AWS CLI v2, credentials, region, default VPC, SSH client.
2. Get the latest Amazon Linux 2023 AMI and your public IP.
3. Create the `ansible-lab` key pair; `chmod 400` the `.pem`.
4. Create `ansible-control-sg` (22 from my IP) and `ansible-managed-sg` (22 from control SG, 80 from anywhere).
5. Launch `ansible-control`, `node1`, `node2`; wait until running.
6. Note the private and public IPs.

### Phase 2: Control node setup
7. Copy the private key to the control node as `~/.ssh/ansible_lab`; `chmod 400`.
8. Install `ansible-core`; see `config file = None`.
9. `telnet <ip> 22` → TCP OK. `ssh -i` → login OK, host keys saved.

### Phase 3: Without `ansible.cfg`
10. Create `ansible-raw-lab/inventory/hosts` with `node1`/`node2` aliases.
11. `ansible-inventory --graph` to verify.
12. Run ping with `-i`, `-u`, `--private-key` → `SUCCESS`, `pong`.
13. Feel the repetition with `command` and `shell` modules.

### Phase 4: With `ansible.cfg`
14. Create `ansible.cfg`: `inventory`, `remote_user`, `private_key_file`, `host_key_checking`.
15. `ansible --version` → `config file = .../ansible.cfg`; `ansible-config dump --only-changed`.
16. `ansible all -m ping` → same result, short command.
17. Prove the command line overrides the file (`-i only-node1`).

### Phase 5: Discovery and environment variables
18. Discovery order: `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`; first found wins, no merging.
19. `export` / `echo` / `printenv` / `unset` `ANSIBLE_CONFIG` (temporary).
20. Add it to `~/.bashrc` + `source` (permanent, no reboot).
21. Deleting the `.bashrc` line does not unset the variable; `unset` it too.

### Phase 6: Use, troubleshoot, clean up
22. Install nginx on both nodes with one playbook; second run `changed=0` (idempotent).
23. Troubleshoot: config not found, wrong `ANSIBLE_CONFIG`, world-writable folder, key not found, `Permission denied`, inventory empty, timeout vs refused.
24. Discovery (**which file**) ≠ precedence (**which value**: CLI > env > cfg > default).
25. Clean up: terminate → wait → delete SGs (managed first) → delete key pair.

## Final takeaway

`ansible.cfg` is **not mandatory**: Ansible works without it. But once you go from 2 servers to dozens or hundreds, writing settings **once** in a file that lives with your code becomes essential. Together with **key-based SSH** and **least-privilege security groups**, this is the foundation for running Ansible in real cloud and DevOps environments.

## Interview Questions

See [interview-questions/README.md](interview-questions/README.md).
