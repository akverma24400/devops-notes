# Ansible Lab (`ansible.cfg`) - Interview Questions

[← Back to the lab](../README.md)

## Other Questions

**1. What is the difference between a control node and a managed node?**

The control node has Ansible installed and runs all commands and playbooks. A managed node is a server Ansible connects to over SSH and configures; it only needs SSH and Python, no Ansible (agentless).

**2. What is `ansible.cfg` and why use it?**

Ansible's configuration file. It stores defaults like the inventory path, SSH user and private key, so commands shrink from `ansible all -i ... -u ... --private-key ... -m ping` to `ansible all -m ping`. It also keeps settings in the repo, shared by the whole team.

**3. In what order does Ansible look for its config file?**

1. `ANSIBLE_CONFIG` environment variable, 2. `ansible.cfg` in the current folder, 3. `~/.ansible.cfg`, 4. `/etc/ansible/ansible.cfg`. It uses the first one it finds and does not merge them.

**4. How do you check which config file Ansible is using?**

`ansible --version` → the `config file =` line. `ansible-config dump --only-changed` shows which settings differ from the defaults.

**5. What happens if `ANSIBLE_CONFIG` points to a file that doesn't exist?**

Ansible skips it silently and uses the next file in the search order (or none). No error is shown, so always confirm with `ansible --version`.

**6. What is the difference between config discovery and config precedence?**

Discovery decides **which file** is loaded. Precedence decides **which value wins** for one setting: command line > environment variable > `ansible.cfg` > built-in default. Example: `-u root` overrides `remote_user = ec2-user`.

**7. Does the Ansible `ping` module send ICMP packets?**

No. It logs in over SSH, runs a small Python module on the node and returns `pong`. Success proves network, SSH login and Python all work.

**8. What does `host_key_checking = False` do, and is it safe?**

It stops Ansible from asking "Are you sure you want to continue connecting?" for unknown servers. Fine for throw-away labs; in production keep it on and manage `known_hosts` to avoid connecting to a fake server.

**9. What does `node1 ansible_host=172.31.16.19` mean in an inventory?**

`node1` is the friendly inventory name shown in output; `ansible_host` is the real IP Ansible connects to.

**10. `telnet <ip> 22` says `Connected` but Ansible fails with `Permission denied (publickey)`. Why?**

Telnet only proves TCP port 22 is reachable. Authentication is a separate step: wrong key, wrong user, public key missing on the node, or key permissions too open (`chmod 400`).

**11. `Connection timed out` vs `Connection refused`?**

Timed out = packets are blocked (security group / network). Refused = the server is reachable but nothing is listening on that port (service down or still booting).

**12. How do you make an environment variable permanent, and how do you remove it?**

Add `export VAR=value` to `~/.bashrc` and run `source ~/.bashrc` (no reboot). To remove: delete the line **and** run `unset VAR`, because re-sourcing doesn't remove a variable that's already set.

**13. Why does Ansible ignore `ansible.cfg` in a world-writable directory?**

Security: anyone could drop a malicious `ansible.cfg` there. Fix with `chmod 755` on the folder.

**14. How would you secure SSH between the control node and managed nodes on AWS?**

Key-based auth (no passwords), private key `chmod 400`, SSH to the control node only from your IP, and SSH to managed nodes only from the control node's security group (`--source-group`), never `0.0.0.0/0`.

**15. How do you use one `ansible.cfg` for dev, staging and production?**

Keep common defaults in `ansible.cfg` and keep one inventory per environment (`inventory/dev/hosts`, ...). Choose with `-i`, which overrides the default inventory.
