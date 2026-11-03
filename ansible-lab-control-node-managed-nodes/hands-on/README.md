# Hands-on Screenshots: Ansible Control Node + 2 Managed Nodes

Screenshots from my run of the [lab](../README.md), in the same order as the lab steps.

## Steps 6–8: Create the 3 servers

**Creating the control node, node1 and node2 (each prints its instance ID)**

![Creating the control node, node1 and node2 (each prints its instance ID)](images/01-create-3-servers.png)

## Step 9: Get the IP addresses

**All 3 running, with private and public IPs**

![All 3 running, with private and public IPs](images/02-wait-and-get-ips.png)

**The same 3 servers in the EC2 console**

![The same 3 servers in the EC2 console](images/03-console-3-running.png)

## Step 10: Copy the private key to the control node (on your laptop)

**Key copied to the control node with scp**

![Key copied to the control node with scp](images/04-scp-key.png)

## Steps 11–12: Log in and protect the key

**Logged in to the control node, key protected with chmod 400**

![Logged in to the control node, key protected with chmod 400](images/05-login-chmod-key.png)

## Step 13: Install Ansible

**ansible-core 2.15.3 installed, config file = None**

![ansible-core 2.15.3 installed, config file = None](images/06-install-ansible.png)

## Steps 18–19: Check the inventory and ping (the long way)

**Inventory graph + ping the long way (note the yellow Python warnings)**

![Inventory graph + ping the long way (note the yellow Python warnings)](images/07-inventory-graph-and-ping-long.png)

## Step 20: Run two more commands (see the repetition)

**uptime and df -h, typing -i, -u and --private-key every time**

![uptime and df -h, typing -i, -u and --private-key every time](images/08-uptime-df-long-way.png)

## Step 21: Create `ansible.cfg`

**ansible.cfg in vim**

![ansible.cfg in vim](images/09-ansible-cfg.png)

## Step 22: Check Ansible found it

**Key path found with realpath, and Ansible now uses ansible-raw-lab/ansible.cfg**

![Key path found with realpath, and Ansible now uses ansible-raw-lab/ansible.cfg](images/10-key-path-and-config-found.png)

## Step 24: Ping again (the short way)

**Short commands, no warnings**

![Short commands, no warnings](images/11-ping-short-way.png)

## Step 26: Show that the folder decides which `ansible.cfg` is used

**In ~ → config file = None. In ~/ansible-raw-lab → ansible.cfg found**

![In ~ → config file = None. In ~/ansible-raw-lab → ansible.cfg found](images/12-folder-decides-config.png)

## Step 27: Point Ansible to the config from any folder (temporary)

**export ANSIBLE_CONFIG works from ~**

![export ANSIBLE_CONFIG works from ~](images/13-ansible-config-export.png)

## Step 28: Remove it

**After unset → config file = None again**

![After unset → config file = None again](images/14-ansible-config-unset.png)

## Steps 29–30: Make it permanent with `.bashrc`, then undo it

**Permanent with .bashrc, then undone**

![Permanent with .bashrc, then undone](images/15-bashrc-permanent-and-undo.png)

## Step 31: Write the playbook

**The nginx playbook in vim**

![The nginx playbook in vim](images/16-nginx-playbook.png)

## Step 32: Run the playbook

**Playbook run: changed=2, failed=0 on both nodes**

![Playbook run: changed=2, failed=0 on both nodes](images/17-run-playbook.png)

## Step 33: Check nginx

**curl from the control node: 200 OK, Server: nginx**

![curl from the control node: 200 OK, Server: nginx](images/18-curl-nginx.png)

**node1 in the browser**

![node1 in the browser](images/19-browser-node1.png)

**node2 in the browser**

![node2 in the browser](images/20-browser-node2.png)

## Step 34: Delete the servers

**All 3 servers terminated**

![All 3 servers terminated](images/21-terminated.png)
