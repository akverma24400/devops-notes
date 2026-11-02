# Ansible Lab: Control Node + 2 Managed Nodes with SSH Keys & `ansible.cfg`

Create 3 EC2 servers with the AWS CLI, install Ansible on one (the **control node**), and manage the other two (**node1**, **node2**) over SSH: first without `ansible.cfg`, then with it.

**OS:** Amazon Linux 2023 · **Region:** us-east-1 · **Size:** t3.micro · [Interview questions](interview-questions/README.md)

> All IDs and IPs below are **from my run**. When you do the lab, **replace them with yours**.

| Server | Instance ID | Security group | Private IP | Public IP |
|---|---|---|---|---|
| ansible-control | `i-0111ec6253b296e51` | `sg-0a2587a6c7da61e37` (ansible-control-sg) | `172.31.14.80` | `44.192.106.237` |
| node1 | `i-094cf172053ec2ed0` | `sg-08bc86c7cf682a5d7` (ansible-managed-sg) | `172.31.5.204` | `13.219.224.71` |
| node2 | `i-042255565324b6c10` | `sg-08bc86c7cf682a5d7` (ansible-managed-sg) | `172.31.10.232` | `44.214.4.44` |

```
laptop ──ssh 22──► ansible-control ──ssh 22 (private IPs)──► node1, node2
                   (Ansible installed)                      (no Ansible: agentless)
```

---

## Prerequisites (on your laptop)

| Need | Check command | Good result |
|---|---|---|
| AWS CLI v2 | `aws --version` | `aws-cli/2.x` |
| AWS login | `aws sts get-caller-identity` | shows your account ID |
| Region | `aws configure get region` | `us-east-1` |
| SSH client | `ssh -V` | `OpenSSH_...` |

If the AWS login fails, run `aws configure` and enter your access key, secret key and region.

---

# Part 1: Create the servers (on your laptop)

### Step 1: Get the latest Amazon Linux 2023 AMI ID

```bash
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text
```

**What it does:** AWS publishes the latest Amazon Linux 2023 image ID for every region. This command prints it, so you never use an old or wrong AMI ID.
**Expected:** `ami-0c230065b91270bc6`

### Step 2: Get your laptop's public IP

```bash
curl -s https://checkip.amazonaws.com
```

**What it does:** asks AWS "what is my IP?". We allow SSH to the control node **only from this IP**.
**Expected:** your IP, e.g. `2.82.252.24`

### Step 3: Create the SSH key pair

```bash
aws ec2 create-key-pair --key-name ansible-lab --query 'KeyMaterial' --output text > ansible-lab.pem
```

**What it does:** AWS creates a key pair called `ansible-lab`. AWS keeps the **public key** (and puts it on every server we launch with it). The **private key** is saved to `ansible-lab.pem` on your laptop; AWS shows it **only once**.

```bash
chmod 400 ansible-lab.pem
```

**What it does:** makes the key readable only by you, otherwise SSH refuses to use it.
**Expected:** `ls -l ansible-lab.pem` → `-r-------- ... ansible-lab.pem`

### Step 4: Create the control node security group + allow SSH from your IP

```bash
aws ec2 create-security-group --group-name ansible-control-sg --description "Ansible control node" && \
aws ec2 authorize-security-group-ingress --group-name ansible-control-sg --protocol tcp --port 22 --cidr 2.82.252.24/32
```

**What it does:**

| Command | Meaning |
|---|---|
| `create-security-group --group-name ansible-control-sg` | creates a firewall (security group) called `ansible-control-sg` for the control node |
| `authorize-security-group-ingress ... --port 22 --cidr 2.82.252.24/32` | **inbound rule**: SSH (port 22) allowed **only from your IP** (`/32` = exactly one IP, from Step 2) |

**Expected:** `"GroupId": "sg-0a2587a6c7da61e37"`, then `"Return": true`

> **`&& \`** joins the commands into **one**: `&&` = run the next part **only if the previous one succeeded** (if the SG can't be created, no rule is added); `\` = the command continues on the next line. AWS has no single call that creates an SG **with** rules, so we chain them.

### Step 5: Create the managed nodes security group + allow SSH and HTTP

```bash
aws ec2 create-security-group --group-name ansible-managed-sg --description "Ansible managed nodes" && \
aws ec2 authorize-security-group-ingress --group-name ansible-managed-sg --protocol tcp --port 22 --source-group ansible-control-sg && \
aws ec2 authorize-security-group-ingress --group-name ansible-managed-sg --protocol tcp --port 80 --cidr 0.0.0.0/0
```

**What it does:** creates **one** security group that we attach to **both** node1 and node2:

| Command | Meaning |
|---|---|
| `create-security-group --group-name ansible-managed-sg` | creates the firewall for the managed nodes |
| `--port 22 --source-group ansible-control-sg` | SSH allowed **only from servers that have `ansible-control-sg`**, so only the control node can SSH to the nodes |
| `--port 80 --cidr 0.0.0.0/0` | HTTP from anywhere, only so we can **open the nginx page in a browser** at the end (installing nginx doesn't need it) |

**Expected:** `"GroupId": "sg-08bc86c7cf682a5d7"`, then `"Return": true` twice

> `--group-name` works in the **default VPC**, so we use names instead of copying `sg-...` IDs. Write down both `GroupId`s; Steps 6–8 use them.

> **Outbound rules:** nothing to add. Every new security group already allows **all outbound** traffic, and security groups are **stateful** (replies are allowed back automatically).

### Step 6: Create the control node

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-0a2587a6c7da61e37 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-control}]' \
  --query 'Instances[0].InstanceId' --output text
```

**What it does:** launches one EC2 server:

| Option | Meaning |
|---|---|
| `--image-id` | OS image: Amazon Linux 2023 |
| `--instance-type t3.micro` | small, free-tier eligible size |
| `--key-name ansible-lab` | puts our public key on the server, so our `.pem` can log in |
| `--security-group-ids` | attaches the **control** security group |
| `--tag-specifications ... Value=ansible-control` | the **Name** shown in the console |
| `--query 'Instances[0].InstanceId' --output text` | prints only the instance ID |

**Expected:** `i-0111ec6253b296e51`

> Run it **once**. Every run creates another server.

### Step 7: Create node1

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-08bc86c7cf682a5d7 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node1}]' \
  --query 'Instances[0].InstanceId' --output text
```

**What it does:** same as Step 6, but with the **managed** security group and the name `node1`.
**Expected:** `i-094cf172053ec2ed0`

### Step 8: Create node2

```bash
aws ec2 run-instances \
  --image-id ami-0c230065b91270bc6 \
  --instance-type t3.micro \
  --key-name ansible-lab \
  --security-group-ids sg-08bc86c7cf682a5d7 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=node2}]' \
  --query 'Instances[0].InstanceId' --output text
```

**What it does:** same as Step 7, name `node2`.
**Expected:** `i-042255565324b6c10`

### Step 9: Get the IP addresses

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ansible-control,node1,node2" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value, InstanceId, PrivateIpAddress, PublicIpAddress]' \
  --output table
```

**What it does:** lists our 3 running servers with their **private IP** (used inside AWS: control node → nodes) and **public IP** (used from the internet: laptop → control node, browser → nginx).
**Expected:**

```
+-----------------+-----------------------+----------------+------------------+
|  node2          |  i-042255565324b6c10  |  172.31.10.232 |  44.214.4.44     |
|  node1          |  i-094cf172053ec2ed0  |  172.31.5.204  |  13.219.224.71   |
|  ansible-control|  i-0111ec6253b296e51  |  172.31.14.80  |  44.192.106.237  |
+-----------------+-----------------------+----------------+------------------+
```

If a server is missing, wait 30 seconds and run it again.

---

# Part 2: Prepare the control node

### Step 10: Copy the private key to the control node (on your laptop)

```bash
scp -i ansible-lab.pem ansible-lab.pem ec2-user@44.192.106.237:/home/ec2-user/.ssh/ansible_lab
```

**What it does:** `scp` copies a file over SSH. It logs in to the control node with `ansible-lab.pem` and copies that same key there as `~/.ssh/ansible_lab`. The control node needs it to SSH into node1 and node2. Type `yes` the first time.
**Expected:** `ansible-lab.pem   100%  1679 ...`

### Step 11: Log in to the control node

```bash
ssh -i ansible-lab.pem ec2-user@44.192.106.237
```

**What it does:** opens a terminal on the control node. **All the next steps run here.**
**Expected:** prompt `[ec2-user@ip-172-31-14-80 ~]$`

> Stay as `ec2-user`. Don't switch to root with `sudo su -`: as root, `~` means `/root` and the key in `/home/ec2-user/.ssh/` isn't found. Commands that need admin rights already use `sudo`.

### Step 12: Protect the key

```bash
chmod 400 ~/.ssh/ansible_lab
```

**What it does:** makes the key readable only by `ec2-user` (SSH refuses keys others can read).

```bash
ls -lah ~/.ssh/
```

**What it does:** lists the folder to check.
**Expected:** `ansible_lab` (`-r--------`, the private key) and `authorized_keys` (the public key AWS put here)

### Step 13: Install Ansible

```bash
sudo dnf install ansible-core -y
```

**What it does:** `dnf` installs Ansible (engine + built-in modules) as root.

```bash
ansible --version
```

**What it does:** confirms the install and shows which config file is used.
**Expected:** `ansible [core 2.x.x]` and `config file = None` (no `ansible.cfg` yet)

### Step 14: Check port 22 on node1 is reachable

```bash
sudo dnf install telnet -y
```

**What it does:** installs the `telnet` tool.

```bash
telnet 172.31.5.204 22
```

**What it does:** tries to open a TCP connection to port 22 on node1. This proves the **security group allows it**. It does **not** test the key. Exit with `Ctrl + ]`, then `quit`. (Non-US keyboard? Press `Ctrl + C` or `Enter`: the SSH server doesn't understand it and closes the connection: `Connection closed by foreign host.`)
**Expected:** `Connected to 172.31.5.204.` and `SSH-2.0-OpenSSH...`

### Step 15: SSH to node1 and node2 with the key

```bash
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.5.204
```

**What it does:** logs in to node1 with the private key. This proves the **key works**. Type `yes` the first time: it saves node1 in `~/.ssh/known_hosts`, so Ansible won't stop to ask later.
**Expected:** prompt changes to `[ec2-user@ip-172-31-5-204 ~]$`

```bash
exit
```

**What it does:** logs out of node1 and brings you back to the control node.

```bash
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.10.232
```

**What it does:** the same for node2.
**Expected:** prompt `[ec2-user@ip-172-31-10-232 ~]$`

```bash
exit
```

---

# Part 3: Ansible without `ansible.cfg`

> Run everything from here on the **control node**, as `ec2-user` (not root). One command per box, run them **one by one**.

### Step 16: Create the project folder

```bash
mkdir ansible-raw-lab
```

**What it does:** creates the project folder in your home (`/home/ec2-user`).

```bash
cd ansible-raw-lab
```

**What it does:** moves into the project folder.

```bash
mkdir inventory
```

**What it does:** creates a folder called `inventory` inside the project.

```bash
pwd
```

**Expected:** `/home/ec2-user/ansible-raw-lab`

### Step 17: Create the inventory file

```bash
cd inventory
```

**What it does:** moves into the `inventory` folder.

```bash
vim hosts
```

**What it does:** opens a new file called `hosts`. Press `i` (insert mode) and type:

```ini
[managed_nodes]
node1 ansible_host=172.31.5.204
node2 ansible_host=172.31.10.232
```

Save and quit: press `Esc`, type `:wq`, press `Enter`.

| Line | Meaning |
|---|---|
| `[managed_nodes]` | group name |
| `node1` | short name shown in Ansible output |
| `ansible_host=172.31.5.204` | the real private IP Ansible connects to |

```bash
cat hosts
```

**What it does:** prints the file to check it.
**Expected:** the 3 lines above.

```bash
cd ..
```

**What it does:** goes back up to `~/ansible-raw-lab`. All next commands run from here.

Folder now looks like this:

```
ansible-raw-lab/
└── inventory/      ← folder
    └── hosts       ← file
```

### Step 18: Check the inventory

```bash
ansible-inventory -i inventory/hosts --graph
```

**What it does:** shows how Ansible understood the inventory, as a tree. It only reads the file; it doesn't connect.
**Expected:**

```
@all:
  |--@ungrouped:
  |--@managed_nodes:
  |  |--node1
  |  |--node2
```

### Step 19: Ping both nodes (the long way)

```bash
ansible all \
  -i /home/ec2-user/ansible-raw-lab/inventory/hosts \
  -u ec2-user \
  --private-key /home/ec2-user/.ssh/ansible_lab \
  -m ping
```

(This is **one** command; the `\` at the end of a line means "continues on the next line".)

**What it does:**

| Part | Meaning |
|---|---|
| `ansible all` | run one task on all hosts in the inventory |
| `-i` | which inventory file |
| `-u ec2-user` | SSH username on the nodes |
| `--private-key` | which private key to log in with |
| `-m ping` | run the `ping` module: logs in over SSH, runs a small Python check, answers `pong` (not ICMP ping) |

**Expected:**

```
node1 | SUCCESS => { "changed": false, "ping": "pong" }
node2 | SUCCESS => { "changed": false, "ping": "pong" }
```

You may also see a yellow `[WARNING]: ... using the discovered Python interpreter at /usr/bin/python3.9`. It's harmless; Step 21 silences it.

### Step 20: Run two more commands (see the repetition)

```bash
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m command -a "uptime"
```

**What it does:** runs `uptime` on both nodes (`-m command` = run a simple command, `-a` = the command).

```bash
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m shell -a "df -h"
```

**What it does:** runs `df -h` (disk usage) on both nodes. `-m shell` also allows pipes `|`, `command` does not.

**Notice:** `-i`, `-u` and `--private-key` are typed **again every time**. That's the problem `ansible.cfg` fixes.

---

# Part 4: Ansible with `ansible.cfg`

### Step 21: Create `ansible.cfg`

```bash
cd ~/ansible-raw-lab
```

**What it does:** makes sure you're in the project folder (the config file must be created here).

```bash
vim ansible.cfg
```

**What it does:** opens a new file called `ansible.cfg`. Press `i` and type:

```ini
[defaults]
inventory = ./inventory/hosts
remote_user = ec2-user
private_key_file = /home/ec2-user/.ssh/ansible_lab
host_key_checking = False
interpreter_python = auto_silent
```

Save and quit: `Esc` → `:wq` → `Enter`.

| Setting | Replaces |
|---|---|
| `inventory` | `-i` |
| `remote_user` | `-u` |
| `private_key_file` | `--private-key` |
| `host_key_checking = False` | typing `yes` for new servers (OK for labs, keep it on in production) |
| `interpreter_python = auto_silent` | still finds Python automatically, but hides the yellow warning |

```bash
cat ansible.cfg
```

**What it does:** prints the file to check it.
**Expected:** the 6 lines above.

Folder now looks like this:

```
ansible-raw-lab/
├── ansible.cfg
└── inventory/
    └── hosts
```

### Step 22: Check Ansible found it

```bash
ansible --version
```

**What it does:** shows which config file Ansible is using.
**Expected:** `config file = /home/ec2-user/ansible-raw-lab/ansible.cfg` (was `None` in Step 13)

### Step 23: See only the settings you changed

```bash
ansible-config dump --only-changed
```

**What it does:** lists the settings that differ from Ansible's defaults, which are the ones from your `ansible.cfg`.

### Step 24: Ping again (the short way)

```bash
ansible all -m ping
```

**What it does:** the same as Step 19, but inventory, user and key now come **from `ansible.cfg`**.
**Expected:** `SUCCESS` / `pong` for node1 and node2, no warning.

```bash
ansible all -m command -a "uptime"
```

**What it does:** the same as Step 20, much shorter.

### Step 25: Show that the command line overrides `ansible.cfg`

```bash
vim inventory/only-node1
```

**What it does:** creates a second inventory with only node1. Press `i` and type:

```ini
[managed_nodes]
node1 ansible_host=172.31.5.204
```

Save and quit: `Esc` → `:wq` → `Enter`.

```bash
ansible all -i inventory/only-node1 -m ping
```

**What it does:** passes the new inventory with `-i`. The flag **wins** over `inventory =` in `ansible.cfg`.
**Expected:** only `node1 | SUCCESS`

### Step 26: Show that the folder decides which `ansible.cfg` is used

```bash
cd ~
```

```bash
ansible --version
```

**Expected:** `config file = None` (no `ansible.cfg` in your home folder)

```bash
cd ~/ansible-raw-lab
```

```bash
ansible --version
```

**Expected:** `config file = /home/ec2-user/ansible-raw-lab/ansible.cfg`

**Why:** Ansible looks for `ansible.cfg` in this order and uses the **first** it finds: `ANSIBLE_CONFIG` variable → `./ansible.cfg` (current folder) → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`.

---

# Part 5: `ANSIBLE_CONFIG` (environment variable)

### Step 27: Point Ansible to the config from any folder (temporary)

```bash
cd ~
```

```bash
export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg
```

**What it does:** sets an environment variable in **this terminal only**. Ansible checks `ANSIBLE_CONFIG` first.

```bash
ansible --version
```

**Expected:** `config file = /home/ec2-user/ansible-raw-lab/ansible.cfg`, even though you're in `~`.

```bash
ansible all -m ping
```

**Expected:** `pong` from both nodes.

### Step 28: Remove it

```bash
unset ANSIBLE_CONFIG
```

**What it does:** deletes the variable from the current terminal.

```bash
ansible --version
```

**Expected:** `config file = None` (we're still in `~`)

### Step 29: Make it permanent with `.bashrc`

```bash
echo 'export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg' >> ~/.bashrc
```

**What it does:** `>>` adds the line to the **end** of `~/.bashrc` (a single `>` would overwrite the file!). Bash runs `.bashrc` every time a new terminal opens.

```bash
source ~/.bashrc
```

**What it does:** runs `.bashrc` **now**, so no re-login is needed.

```bash
printenv ANSIBLE_CONFIG
```

**Expected:** `/home/ec2-user/ansible-raw-lab/ansible.cfg`

### Step 30: Undo it

```bash
sed -i '/ANSIBLE_CONFIG/d' ~/.bashrc
```

**What it does:** deletes the line from `.bashrc`.

```bash
unset ANSIBLE_CONFIG
```

**What it does:** deleting the line **doesn't remove** a variable that's already set, so we also `unset` it.

```bash
printenv ANSIBLE_CONFIG
```

**Expected:** empty output.

---

# Part 6: Install nginx with a playbook

### Step 31: Write the playbook

```bash
cd ~/ansible-raw-lab
```

```bash
mkdir playbooks
```

**What it does:** creates a folder for playbooks.

```bash
vim playbooks/nginx.yaml
```

**What it does:** opens a new playbook file. Press `i` and type (spaces matter in YAML, no tabs):

```yaml
- name: install and start nginx
  hosts: managed_nodes
  become: yes
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

Save and quit: `Esc` → `:wq` → `Enter`.

**What it means:** runs on the `managed_nodes` group as root (`become: yes`). The `package` module installs nginx; the `service` module starts it now (`started`) and on every boot (`enabled`).

> Tip: if vim auto-indents pasted text into a staircase, type `:set paste` before pressing `i`.

Folder now looks like this:

```
ansible-raw-lab/
├── ansible.cfg
├── inventory/
│   ├── hosts
│   └── only-node1
└── playbooks/
    └── nginx.yaml
```

### Step 32: Run the playbook

```bash
ansible-playbook playbooks/nginx.yaml
```

**What it does:** runs the playbook on node1 and node2. No `-i`, `-u` or `--private-key` needed, because `ansible.cfg` provides them.
**Expected:** `changed=2` for each node. Run it again → `changed=0` (already done: **idempotent**).

### Step 33: Check nginx

```bash
curl -I http://172.31.5.204
```

```bash
curl -I http://172.31.10.232
```

**What it does:** requests the nginx page from each node (from the control node, using private IPs).
**Expected:** `HTTP/1.1 200 OK` and `Server: nginx`. In a browser, `http://13.219.224.71` and `http://44.214.4.44` show the nginx welcome page.

---

# Part 7: Clean up (on your laptop)

### Step 34: Delete the servers

```bash
aws ec2 terminate-instances --instance-ids i-0111ec6253b296e51 i-094cf172053ec2ed0 i-042255565324b6c10
```

**What it does:** deletes the 3 servers.

```bash
aws ec2 wait instance-terminated --instance-ids i-0111ec6253b296e51 i-094cf172053ec2ed0 i-042255565324b6c10
```

**What it does:** **waits** until they're fully gone (security groups can't be deleted while a server still uses them). Returns with no output when done.

### Step 35: Delete the security groups

```bash
aws ec2 delete-security-group --group-id sg-08bc86c7cf682a5d7
```

**What it does:** deletes the managed SG **first** (its rule points to the control SG).

```bash
aws ec2 delete-security-group --group-id sg-0a2587a6c7da61e37
```

**What it does:** deletes the control SG.

### Step 36: Delete the key pair

```bash
aws ec2 delete-key-pair --key-name ansible-lab
```

**What it does:** deletes the key pair in AWS.

```bash
rm -f ansible-lab.pem
```

**What it does:** deletes the `.pem` file on your laptop.

---

## If something fails

| Error | Cause | Fix |
|---|---|---|
| `InvalidAMIID.Malformed: Invalid id: "-0c23..."` | typed `$ami-...` (a `$` before a real ID) | remove the `$` |
| `parameter groupId is invalid. The value cannot be empty` | typed `$sg-...` | remove the `$` |
| `WARNING: connection is not using a post-quantum key exchange algorithm` | laptop has a newer OpenSSH than Amazon Linux 2023; they fall back to standard encryption | **harmless warning**, ignore it |
| `ssh`/`scp` to the control node: `Connection timed out` | your laptop's IP changed | redo Step 2, then add the new IP to the control SG (Step 4, second command) |
| `telnet`/`ansible`: `Connection timed out` to a node | wrong SG or wrong IP | node needs `ansible-managed-sg`; use the **private** IP |
| `Permission denied (publickey)` | wrong key or user | test `ssh -i ~/.ssh/ansible_lab ec2-user@<ip>`; `chmod 400` the key |
| `config file = None` | not in the project folder | `cd ~/ansible-raw-lab` |
| `no such identity: ... ansible_lab` | wrong `private_key_file` path | `ls ~/.ssh/` and fix `ansible.cfg` |
| browser can't open nginx | port 80 not open / wrong IP | Step 5; use the node's **public** IP |
