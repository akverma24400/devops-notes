# Day 20 - Interview Questions

[← Back to Day 20 notes](../README.md)

## Other Questions

**1. What two things are needed to run an application on a server?**

The **code** (the application) and the **configuration** of the server (packages, users, folders, service files). Ansible handles the configuration.

**2. What does `--query 'Instances[0].InstanceId' --output text` do in `aws ec2 run-instances`?**

It picks only the instance ID of the first created instance from the JSON output and prints it as plain text, so a script can use it directly.

**3. What is an inventory in Ansible?**

The list of machines Ansible targets, usually in `inventory.ini`, grouped like `[frontend]`, `[backend]`. A group of groups uses `[name:children]` (e.g. `[roboshop:children]` = frontend + backend + database). IPs at the top without a group are **orphans** (Ansible's `ungrouped` group).

**4. What is a DTO?**

Data Transfer Object: an object used only to carry data between systems, e.g. login details sent from an app to a server through an API, usually as JSON.

**5. What do `[]` and `{}` mean?**

`[]` = list (items in order). `{}` = map/object (key: value pairs). In YAML, a list item starts with `-` and a map is `key: value` lines.

**6. XML vs JSON vs YAML?**

XML uses open/close tags and is the longest. JSON uses `{}`, `[]`, quotes and commas, and has no comments. YAML uses `key: value` and indentation, supports `#` comments, and is the shortest; Ansible playbooks are YAML.

**7. What are common YAML mistakes?**

Tabs instead of spaces, wrong indentation, no space after `:`, and commas at the end of lines (they become part of the value). Check with a YAML validator such as yamlchecker.com.

**8. How do you define and use a variable in a playbook?**

Define under `vars:` (e.g. `course: ansible`) and use with `{{ course }}`, e.g. in a `debug` module's `msg`.
