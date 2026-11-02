# Day 20 - Interview Questions

[← Back to Day 20 notes](../README.md)

## ⭐ Very Important (asked in class)

**1. If the same variable name is defined in many places, which value does Ansible use?**

Ansible follows a preference (precedence) order. From highest to lowest: **command line `-e`** > **task `vars`** > **`vars_files`** > **`vars_prompt`** > **play `vars`** > **inventory `[group:vars]`**. The command line always wins, so a value passed at run time overrides everything.

## Other Questions

**1. What two things are needed to run an application on a server?**

The **code** (the application) and the **configuration** of the server (packages, users, folders, service files). Ansible handles the configuration.

**2. What does `--query 'Instances[0].InstanceId' --output text` do in `aws ec2 run-instances`?**

It picks only the instance ID of the first created instance from the JSON output and prints it as plain text, so a script can use it directly.

**3. What is an inventory in Ansible?**

The list of machines Ansible targets, usually in `inventory.ini`, grouped like `[frontend]`, `[backend]`. A group of groups uses `[name:children]` (e.g. `[roboshop:children]` = frontend + backend + database). IPs at the top without a group are **orphans** (Ansible's `ungrouped` group).

**4. What do `[]` and `{}` mean?**

`[]` = list (items in order). `{}` = map/object (key: value pairs). In YAML, a list item starts with `-` and a map is `key: value` lines.

**5. XML vs JSON vs YAML?**

XML uses open/close tags and is the longest. JSON uses `{}`, `[]`, quotes and commas, and has no comments. YAML uses `key: value` and indentation, supports `#` comments, and is the shortest; Ansible playbooks are YAML.

**6. What are common YAML mistakes?**

Tabs instead of spaces, wrong indentation, no space after `:`, and commas at the end of lines (they become part of the value). Check with a YAML validator such as yamlchecker.com.

**7. How do you define and use a variable in a playbook?**

Define under `vars:` (e.g. `course: ansible`) and use with `{{ course }}`, e.g. in a `debug` module's `msg`.

**8. How do you check which machines are in your inventory?**

`ansible-inventory -i inventory.ini --graph`. It prints a tree of groups (`@all`, `@ungrouped`, `@roboshop`, ...) and their machines. It only reads the file; use `-m ping` to test the connection.

**9. What is the structure of a playbook?**

A YAML list of plays. Each play has a `name`, `hosts` (which machines from the inventory) and `tasks` (a list where each task runs one module). Run it with `ansible-playbook -i inventory.ini file.yaml`.

**10. What does `connection: local` do?**

Runs the tasks directly on the Ansible server itself instead of over SSH, so no username/password is needed.

**11. What is the `debug` module used for?**

To print a message or a variable's value during a playbook run, like `echo` in shell: `debug: msg: "Hello"`.

**12. Can one playbook have more than one play? Why?**

Yes, a multi-play playbook: a list of plays, each with its own `hosts` and `tasks`, run top to bottom. It lets one file configure different groups in order, e.g. database, then backend, then frontend.

**13. `package` module vs `dnf` module?**

`dnf` works only on RHEL-family servers. `package` uses whatever package manager the server has (`dnf`, `yum`, `apt`), so the same playbook works on any distro.

**14. In the `service` module, what is the difference between `state: started` and `enabled: yes`?**

`state: started` starts the service now (`systemctl start`). `enabled: yes` makes it start on boot (`systemctl enable`).

**15. What does `become: yes` do?**

Runs the tasks as root (sudo). Needed for installing packages and managing services. It is the playbook version of `-b` in ad-hoc commands.

**16. nginx is running but the browser times out. What do you check?**

The security group: port 80 must be allowed inbound (e.g. `0.0.0.0/0` for public HTTP). A timeout usually means the firewall/SG drops the traffic; "connection refused" means the port is reachable but nothing is listening (nginx not running).

**17. In how many ways can you define variables in Ansible?**

Play `vars:`, task `vars:`, `vars_files:` (separate file), `vars_prompt:` (ask at run time), inventory `[group:vars]`, and `-e` on the command line.

**18. How do you take a password as input without showing it on screen?**

`vars_prompt` with `private: true`. Never print it with `debug` in real projects.

**19. What happens if a playbook uses a variable that is not defined?**

The task fails: `'person_name' is undefined`. Define it somewhere or pass it with `-e "person_name=value"`.

**20. Why does `-e "course=ansible from cmd"` give only `ansible`?**

`-e` with `key=value` splits on spaces. Use inner quotes: `-e "course='ansible from cmd'"`.
