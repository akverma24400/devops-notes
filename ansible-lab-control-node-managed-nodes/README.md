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
chmod 400 ansible-lab.pem
```

**What it does:** AWS creates a key pair called `ansible-lab`. AWS keeps the **public key** (and puts it on every server we launch with it). The **private key** is saved to `ansible-lab.pem` on your laptop; AWS shows it **only once**. `chmod 400` makes it readable only by you, otherwise SSH refuses to use it.
**Expected:** `ls -l ansible-lab.pem` → `-r-------- ... ansible-lab.pem`

### Step 4: Create the control node security group + allow SSH from your IP

```bash
aws ec2 create-security-group --group-name ansible-control-sg --description "Ansible control node"
aws ec2 authorize-security-group-ingress --group-name ansible-control-sg --protocol tcp --port 22 --cidr 2.82.252.24/32
```

**What it does:**

| Command | Meaning |
|---|---|
| `create-security-group --group-name ansible-control-sg` | creates a firewall (security group) called `ansible-control-sg` for the control node |
| `authorize-security-group-ingress ... --port 22 --cidr 2.82.252.24/32` | **inbound rule**: SSH (port 22) allowed **only from your IP** (`/32` = exactly one IP, from Step 2) |

**Expected:** `"GroupId": "sg-0a2587a6c7da61e37"`, then `"Return": true`

### Step 5: Create the managed nodes security group + allow SSH and HTTP

```bash
aws ec2 create-security-group --group-name ansible-managed-sg --description "Ansible managed nodes"
aws ec2 authorize-security-group-ingress --group-name ansible-managed-sg --protocol tcp --port 22 --source-group ansible-control-sg
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

### Step 12: Protect the key

```bash
chmod 400 ~/.ssh/ansible_lab
ls -lah ~/.ssh/
```

**What it does:** makes the key readable only by `ec2-user` (SSH refuses keys others can read), then lists the folder.
**Expected:** `ansible_lab` (`-r--------`, the private key) and `authorized_keys` (the public key AWS put here)

### Step 13: Install Ansible

```bash
sudo dnf install ansible-core -y
ansible --version
```

**What it does:** `dnf` installs Ansible (engine + built-in modules) as root. `ansible --version` confirms the install and shows which config file is used.
**Expected:** `ansible [core 2.x.x]` and `config file = None` (no `ansible.cfg` yet)

### Step 14: Check port 22 on node1 is reachable

```bash
sudo dnf install telnet -y
telnet 172.31.5.204 22
```

**What it does:** tries to open a TCP connection to port 22 on node1. This proves the **security group allows it**. It does **not** test the key. Exit with `Ctrl + ]`, then `quit`.
**Expected:** `Connected to 172.31.5.204.` and `SSH-2.0-OpenSSH...`

### Step 15: SSH to node1 and node2 with the key

```bash
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.5.204
exit
ssh -i ~/.ssh/ansible_lab ec2-user@172.31.10.232
exit
```

**What it does:** logs in to each node with the private key. This proves the **key works**. Typing `yes` the first time saves each node in `~/.ssh/known_hosts`, so Ansible won't stop to ask later.
**Expected:** the prompt changes to the node, then `exit` brings you back.

---

# Part 3: Ansible without `ansible.cfg`

### Step 16: Create the project folder

```bash
mkdir -p ~/ansible-raw-lab/inventory
cd ~/ansible-raw-lab
pwd
```

**What it does:** creates the project folder and an `inventory` folder inside it, then moves into the project.
**Expected:** `/home/ec2-user/ansible-raw-lab`

### Step 17: Create the inventory

```bash
cat > inventory/hosts <<'EOF'
[managed_nodes]
node1 ansible_host=172.31.5.204
node2 ansible_host=172.31.10.232
EOF
cat inventory/hosts
```

**What it does:** writes the **inventory**, the list of servers Ansible manages. `[managed_nodes]` is the group name; `node1` is a short name shown in the output; `ansible_host` is the real private IP Ansible connects to.
**Expected:** the 3 lines above.

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

### Step 20: Run two more commands (see the repetition)

```bash
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m command -a "uptime"
ansible all -i /home/ec2-user/ansible-raw-lab/inventory/hosts -u ec2-user --private-key /home/ec2-user/.ssh/ansible_lab -m shell -a "df -h"
```

**What it does:** runs `uptime` and `df -h` on both nodes (`-a` = the command to run). Notice `-i`, `-u` and `--private-key` are typed **again every time**. That's the problem `ansible.cfg` fixes.
**Expected:** uptime and disk usage for node1 and node2.

---

# Part 4: Ansible with `ansible.cfg`

### Step 21: Create `ansible.cfg`

```bash
cd ~/ansible-raw-lab
cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory/hosts
remote_user = ec2-user
private_key_file = /home/ec2-user/.ssh/ansible_lab
host_key_checking = False
EOF
cat ansible.cfg
```

**What it does:** writes Ansible's settings file in the project folder:

| Setting | Replaces |
|---|---|
| `inventory` | `-i` |
| `remote_user` | `-u` |
| `private_key_file` | `--private-key` |
| `host_key_checking = False` | typing `yes` for new servers (OK for labs, keep it on in production) |

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

**What it does:** lists the settings that differ from Ansible's defaults, which are the 4 from your `ansible.cfg`.

### Step 24: Ping again (the short way)

```bash
ansible all -m ping
ansible all -m command -a "uptime"
```

**What it does:** the same as Steps 19–20, but inventory, user and key now come **from `ansible.cfg`**.
**Expected:** `SUCCESS` / `pong` for node1 and node2.

### Step 25: Show that the command line overrides `ansible.cfg`

```bash
printf "[managed_nodes]\nnode1 ansible_host=172.31.5.204\n" > inventory/only-node1
ansible all -i inventory/only-node1 -m ping
```

**What it does:** creates an inventory with only node1 and passes it with `-i`. The flag **wins** over `inventory =` in `ansible.cfg`.
**Expected:** only `node1 | SUCCESS`

### Step 26: Show that the folder decides which `ansible.cfg` is used

```bash
cd ~
ansible --version
cd ~/ansible-raw-lab
ansible --version
```

**What it does:** Ansible looks for `ansible.cfg` in this order and uses the **first** it finds: `ANSIBLE_CONFIG` variable → `./ansible.cfg` (current folder) → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`.
**Expected:** in `~` → `config file = None`; in the project → `config file = /home/ec2-user/ansible-raw-lab/ansible.cfg`

---

# Part 5: `ANSIBLE_CONFIG` (environment variable)

### Step 27: Point Ansible to the config from any folder (temporary)

```bash
cd ~
export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg
echo $ANSIBLE_CONFIG
ansible --version
ansible all -m ping
```

**What it does:** `export` sets an environment variable in **this terminal only**. Ansible checks `ANSIBLE_CONFIG` first, so it uses our file even from `~`.
**Expected:** `config file = /home/ec2-user/ansible-raw-lab/ansible.cfg` and `pong` from both nodes.

### Step 28: Remove it

```bash
unset ANSIBLE_CONFIG
echo $ANSIBLE_CONFIG
ansible --version
```

**What it does:** `unset` deletes the variable from the current terminal.
**Expected:** empty line, then `config file = None` (we're still in `~`)

### Step 29: Make it permanent with `.bashrc`

```bash
echo 'export ANSIBLE_CONFIG=/home/ec2-user/ansible-raw-lab/ansible.cfg' >> ~/.bashrc
source ~/.bashrc
printenv ANSIBLE_CONFIG
```

**What it does:** `>>` adds the line to the **end** of `~/.bashrc` (a single `>` would overwrite the file!). Bash runs `.bashrc` every time a new terminal opens. `source` runs it **now**, so no reboot or re-login is needed.
**Expected:** `/home/ec2-user/ansible-raw-lab/ansible.cfg`

### Step 30: Undo it

```bash
sed -i '/ANSIBLE_CONFIG/d' ~/.bashrc
unset ANSIBLE_CONFIG
printenv ANSIBLE_CONFIG
```

**What it does:** `sed -i '/.../d'` deletes the line from `.bashrc`. But deleting the line **doesn't remove** a variable that's already set, so we also `unset` it.
**Expected:** empty output.

---

# Part 6: Install nginx with a playbook

### Step 31: Write the playbook

```bash
cd ~/ansible-raw-lab
mkdir -p playbooks
cat > playbooks/nginx.yaml <<'EOF'
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
EOF
```

**What it does:** creates a playbook that runs on the `managed_nodes` group as root (`become: yes`). The `package` module installs nginx; the `service` module starts it now (`started`) and on every boot (`enabled`).

### Step 32: Run the playbook

```bash
ansible-playbook playbooks/nginx.yaml
```

**What it does:** runs the playbook on node1 and node2. No `-i`, `-u` or `--private-key` needed, because `ansible.cfg` provides them.
**Expected:** `changed=2` for each node. Run it again → `changed=0` (already done: **idempotent**).

### Step 33: Check nginx

```bash
curl -I http://172.31.5.204
curl -I http://172.31.10.232
```

**What it does:** requests the nginx page from each node (from the control node, using private IPs).
**Expected:** `HTTP/1.1 200 OK` and `Server: nginx`. In a browser, `http://13.219.224.71` and `http://44.214.4.44` show the nginx welcome page.

---

# Part 7: Clean up (on your laptop)

### Step 34: Delete the servers

```bash
aws ec2 terminate-instances --instance-ids i-0111ec6253b296e51 i-094cf172053ec2ed0 i-042255565324b6c10
aws ec2 wait instance-terminated --instance-ids i-0111ec6253b296e51 i-094cf172053ec2ed0 i-042255565324b6c10
```

**What it does:** deletes the 3 servers, then **waits** until they're fully gone (security groups can't be deleted while a server still uses them).

### Step 35: Delete the security groups

```bash
aws ec2 delete-security-group --group-id sg-08bc86c7cf682a5d7
aws ec2 delete-security-group --group-id sg-0a2587a6c7da61e37
```

**What it does:** deletes the managed SG **first** (its rule points to the control SG), then the control SG.

### Step 36: Delete the key pair

```bash
aws ec2 delete-key-pair --key-name ansible-lab
rm -f ansible-lab.pem
```

**What it does:** deletes the key pair in AWS and the `.pem` file on your laptop.

---

## If something fails

| Error | Cause | Fix |
|---|---|---|
| `InvalidAMIID.Malformed: Invalid id: "-0c23..."` | typed `$ami-...` (a `$` before a real ID) | remove the `$` |
| `parameter groupId is invalid. The value cannot be empty` | typed `$sg-...` | remove the `$` |
| `ssh`/`scp` to the control node: `Connection timed out` | your laptop's IP changed | redo Step 2, then add the new IP to the control SG (Step 4, second command) |
| `telnet`/`ansible`: `Connection timed out` to a node | wrong SG or wrong IP | node needs `ansible-managed-sg`; use the **private** IP |
| `Permission denied (publickey)` | wrong key or user | test `ssh -i ~/.ssh/ansible_lab ec2-user@<ip>`; `chmod 400` the key |
| `config file = None` | not in the project folder | `cd ~/ansible-raw-lab` |
| `no such identity: ... ansible_lab` | wrong `private_key_file` path | `ls ~/.ssh/` and fix `ansible.cfg` |
| browser can't open nginx | port 80 not open / wrong IP | Step 5; use the node's **public** IP |
