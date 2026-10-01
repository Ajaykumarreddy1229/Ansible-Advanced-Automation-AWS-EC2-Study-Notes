# Ansible-Advanced-Automation-AWS-EC2-Study-Notes

## 📚 Topics Covered

In this learning session, I practiced the following Ansible concepts:

* Registering task output
* `stdout`, `stderr`, `rc`, `changed`, and `failed`
* Launching EC2 instances using Ansible
* Managing multiple EC2 instances
* Using variables with Ansible
* Loops for multiple resources
* Different instance names and instance types
* EC2 state management
* Reboot module
* File management
* `copy` module
* `fetch` module
* `synchronize` module
* `archive` module
* Website deployment using Nginx
* Handlers
* `delegate_to`
* `run_once`
* Ansible Navigator
* Execution Environments
* Fully Qualified Collection Names (FQCN)
* Idempotency
* Ansible plugins
* AWS dynamic inventory

---

# 1. Register in Ansible

The `register` keyword captures the output of a task and stores it in a variable.

This allows the output to be used later in the playbook.

### Example

```yaml
- name: Check Hostname
  command: hostname
  register: result

- debug:
    msg: "Hostname : {{ result.stdout }}"

- debug:
    msg: "Return Code : {{ result.rc }}"

- debug:
    msg: "Changed : {{ result.changed }}"

- debug:
    msg: "Failed : {{ result.failed }}"
```

### Important Registered Variables

| Variable  | Purpose                             |
| --------- | ----------------------------------- |
| `stdout`  | Standard output                     |
| `stderr`  | Error output                        |
| `rc`      | Return code                         |
| `changed` | Whether the task changed the system |
| `failed`  | Whether the task failed             |

Example:

```text
result.stdout
result.stderr
result.rc
result.changed
result.failed
```

A return code of `0` indicates successful command execution in the examples practiced.

---

# 2. Register – Failure Example

```yaml
- name: Invalid Command
  command: xyz123
  register: result
  ignore_errors: yes

- debug:
    var: result.stderr

- debug:
    var: result.rc

- debug:
    var: result.failed
```

Example result:

```text
stderr: xyz123: command not found
rc: 127
failed: true
```

### Why use register?

`register` is useful for:

* Debugging
* Reporting
* Conditional execution
* Capturing command output
* Working with AWS resource information

---

# 3. Ansible + AWS EC2

Ansible can be used to automate AWS infrastructure.

For AWS EC2 automation, the notes use:

```bash
sudo dnf install python3-pip -y
pip3 install boto3 botocore
```

An IAM Role can be attached to the Ansible control node so that Ansible can interact with AWS.

Inventory example:

```ini
[prod]
172.31.20.91
172.31.26.7

[local]
localhost ansible_connection=local
```

---

# 4. Launch EC2 Instance Using Ansible

The `amazon.aws.ec2_instance` module can be used to launch EC2 instances.

Example:

```yaml
- name: Launch EC2 Instance
  hosts: local
  gather_facts: false

  collections:
    - amazon.aws

  vars:
    region: ap-south-1
    ami: ami-xxxxxxxx
    instance_type: t3.micro
    key_name: MyKey
    security_group: sg-xxxxxxxx
    subnet_id: subnet-xxxxxxxx

  tasks:

    - name: Launch EC2 Instance
      amazon.aws.ec2_instance:
        name: DNA-Server
        region: "{{ region }}"
        image_id: "{{ ami }}"
        instance_type: "{{ instance_type }}"
        key_name: "{{ key_name }}"
        security_group: "{{ security_group }}"
        vpc_subnet_id: "{{ subnet_id }}"

        network:
          assign_public_ip: true

        tags:
          Owner: Demo
          Environment: Demo

        wait: true
        state: running

      register: ec2

    - name: Display Instance Details
      debug:
        var: ec2.instances
```

### Important

Do not commit real:

* AWS credentials
* Private keys
* Secrets
* Sensitive infrastructure information

to GitHub.

---

# 5. EC2 Instance States

| State        | Purpose                                |
| ------------ | -------------------------------------- |
| `running`    | Launch or start instance               |
| `stopped`    | Stop instance                          |
| `started`    | Start an existing stopped instance     |
| `restarted`  | Reboot instance                        |
| `terminated` | Permanently delete instance            |
| `absent`     | Ensure matching instances do not exist |

Run:

```bash
ansible-playbook ec2.yml
```

To terminate an instance, change:

```yaml
state: running
```

to:

```yaml
state: terminated
```

---

# 6. Launch Multiple EC2 Instances

Multiple instances can be created using variables and loops.

Example:

```yaml
instance_names:
  - Web-Server
  - App-Server
  - DB-Server
```

Then:

```yaml
- name: Launch EC2 Instances
  amazon.aws.ec2_instance:
    name: "{{ item }}"
    region: "{{ region }}"
    image_id: "{{ ami }}"
    instance_type: "{{ instance_type }}"
    key_name: "{{ key_name }}"
    security_group: "{{ security_group }}"
    vpc_subnet_id: "{{ subnet_id }}"

    network:
      assign_public_ip: true

    state: running
    wait: true

  loop: "{{ instance_names }}"
  register: ec2_output
```

Display results:

```yaml
- debug:
    var: ec2_output.results
```

---

# 7. Different Names and Different Instance Types

Ansible variables can contain multiple dictionaries.

```yaml
instance_names:
  - name: Web-Server
    type: t3.micro

  - name: App-Server
    type: m7i-flex.large

  - name: DB-Server
    type: t3.small
```

Use them with:

```yaml
- name: Launch EC2 Instances
  amazon.aws.ec2_instance:
    name: "{{ item.name }}"
    instance_type: "{{ item.type }}"
    region: "{{ region }}"
    image_id: "{{ ami }}"
    key_name: "{{ key_name }}"
    security_group: "{{ security_group }}"
    vpc_subnet_id: "{{ subnet_id }}"

    network:
      assign_public_ip: true

    state: running
    wait: true

  loop: "{{ instance_names }}"
  register: ec2_output
```

---

# 8. EC2 Tags Using Variables

Example:

```yaml
instances:
  - name: Web
    type: t3.micro
    env: Dev

  - name: App
    type: t3.small
    env: Test

  - name: DB
    type: t3.medium
    env: Prod
```

Use:

```yaml
tags:
  Environment: "{{ item.env }}"
  Owner: Demo
```

This allows infrastructure to be created with different names, instance types, and environments.

---

# 9. Reboot Module

The `reboot` module restarts a remote machine and waits for it to become available again.

```yaml
- hosts: prod
  become: yes

  tasks:

    - reboot:
        msg: "Server is rebooting. Please wait..."
```

---

# 10. File Management in Ansible

Ansible provides modules for managing files between the control node and managed nodes.

Important modules practiced:

```text
copy
fetch
synchronize
archive
unarchive
```

---

# 11. Copy Module

The `copy` module copies files from the Ansible Control Node to a Managed Node.

Example:

```yaml
- name: Copy Website
  copy:
    src: index.html
    dest: /usr/share/nginx/html/index.html
    owner: root
    group: root
    mode: '0644'
```

You can also create content directly:

```yaml
- copy:
    content: "Welcome to Ansible"
    dest: /tmp/message.txt
```

---

# 12. Fetch Module

`fetch` performs the opposite direction of `copy`.

It copies a file from the Managed Node to the Control Node.

```yaml
- name: Fetch nginx.conf
  fetch:
    src: /etc/nginx/nginx.conf
    dest: backups/
```

### Easy way to remember

```text
copy
Control Node ───────────► Managed Node

fetch
Managed Node ───────────► Control Node
```

---

# 13. Synchronize Module

The `synchronize` module uses `rsync` to efficiently synchronize directories.

Install rsync:

```yaml
- name: Install rsync
  dnf:
    name: rsync
    state: present
```

Synchronize website:

```yaml
- name: Synchronize Website
  synchronize:
    src: website/
    dest: /usr/share/nginx/html/
```

---

# 14. Archive Module

The `archive` module can be used to create backups.

Example:

```yaml
- name: Create Backup
  archive:
    path: /usr/share/nginx/html
    dest: /tmp/website_backup.tar.gz
```

This is useful before deploying a new website version.

---

# 15. Website Deployment Using Ansible

A complete deployment flow practiced:

```text
Control Node
     │
     ▼
Install Nginx
     │
     ▼
Start Nginx
     │
     ▼
Backup Existing Website
     │
     ▼
Copy Website Package
     │
     ▼
Extract Website
     │
     ▼
Restart Nginx
     │
     ▼
Verify Website
```

Example:

```yaml
- name: Website Deployment
  hosts: prod
  become: yes

  handlers:

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted

  tasks:

    - name: Install Nginx
      dnf:
        name: nginx
        state: present

    - name: Enable Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Backup Existing Website
      archive:
        path: /usr/share/nginx/html
        dest: /tmp/site_backup.tar.gz

    - name: Copy Website Package
      copy:
        src: website.tar.gz
        dest: /tmp/

    - name: Deploy Website
      unarchive:
        src: /tmp/website.tar.gz
        dest: /usr/share/nginx/html/
        remote_src: yes
      notify: Restart Nginx

    - name: Verify Website
      uri:
        url: http://localhost
        return_content: yes
      register: website

    - name: Show Website Content
      debug:
        var: website.content
```

---

# 16. Handlers

Handlers are tasks that execute when notified by another task.

Example:

```yaml
- name: Deploy Website
  unarchive:
    src: /tmp/website.tar.gz
    dest: /usr/share/nginx/html/
    remote_src: yes
  notify: Restart Nginx
```

Handler:

```yaml
handlers:

  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```

This helps restart a service only when a relevant change occurs.

---

# 17. Delegation in Ansible

Normally, Ansible executes tasks on the target hosts.

Sometimes a task needs to execute:

* On the control node
* On another server
* Only once

This is where delegation is useful.

---

# 18. delegate_to

`delegate_to` tells Ansible to execute a task on another machine instead of the current target host.

Example:

```yaml
- name: Generate Deployment Report
  copy:
    content: "Deployment completed successfully."
    dest: "/tmp/deployment-report.txt"
  delegate_to: localhost
  run_once: true
```

The deployment may happen on the managed server while the report is generated on the control node.

---

# 19. run_once

`run_once` makes a task execute only once even when multiple hosts are targeted.

Example:

```yaml
- name: Generate Deployment Report
  copy:
    content: "Deployment completed successfully."
    dest: "/tmp/deployment-report.txt"
  delegate_to: localhost
  run_once: true
```

Useful for:

* Generating reports
* Sending notifications
* Database initialization
* Updating a CMDB

---

# 20. delegate_to vs local_action vs run_once

| Feature                 | delegate_to | local_action | run_once |
| ----------------------- | ----------- | ------------ | -------- |
| Execute on another host | ✅           | ✅            | ❌        |
| Execute on control node | ✅           | ✅            | ❌        |
| Run only once           | ❌           | ❌            | ✅        |
| Can be combined         | ✅           | ✅            | ✅        |

---

# 21. Ansible Navigator

Ansible Navigator is a CLI tool from Red Hat for running, debugging, and exploring Ansible content, especially with Execution Environments.

### Simple comparison

```text
ansible-playbook
      │
      └── Runs playbooks

Ansible Navigator
      │
      ├── Runs
      ├── Visualizes
      ├── Debugs
      └── Explores
```

---

# 22. Execution Environment

Ansible Navigator can run Ansible content inside containerized Execution Environments.

Conceptually:

```text
Playbook
   │
   ▼
Ansible Navigator
   │
   ▼
Execution Environment
   │
   ▼
Target Hosts
```

This helps provide a more consistent runtime environment and reduces dependency conflicts.

---

# 23. Install Ansible Navigator

```bash
sudo dnf install python3-pip -y

pip3 install ansible-navigator

ansible-navigator --version
```

Docker can be installed for container-based execution:

```bash
sudo dnf install docker -y

sudo systemctl start docker
```

Run a playbook:

```bash
ansible-navigator run nginx.yml \
-i /etc/ansible/hosts \
--container-engine docker
```

---

# 24. Fully Qualified Collection Name (FQCN)

Modern Ansible playbooks can use fully qualified module names.

Example:

```yaml
ansible.builtin.package:
```

instead of:

```yaml
package:
```

Example:

```yaml
ansible.builtin.service:
```

FQCN improves clarity and avoids ambiguity in collection-based environments.

---

# 25. Ansible Navigator + Nginx Example

```yaml
- name: Install Nginx using Navigator
  hosts: prod
  become: true
  gather_facts: true

  tasks:

    - name: Install nginx package
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Start and enable nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Verify nginx service
      ansible.builtin.command: systemctl is-active nginx
      register: nginx_status

    - name: Show nginx status
      ansible.builtin.debug:
        var: nginx_status.stdout
```

Flow:

```text
Connect
   ↓
Install Nginx
   ↓
Start Service
   ↓
Check Status
   ↓
Print Result
```

---

# 26. ansible-playbook vs Ansible Navigator

| Feature               | ansible-playbook | ansible-navigator |
| --------------------- | ---------------- | ----------------- |
| Run Playbooks         | ✅                | ✅                 |
| Container Support     | ❌                | ✅                 |
| Interactive UI        | ❌                | ✅                 |
| Execution Environment | Limited          | ✅                 |
| Explore/Debug         | Basic            | ✅                 |

---

# 27. Idempotency

Idempotency means that running an Ansible playbook multiple times should not repeatedly change a system after the desired state has already been reached.

Example:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

First execution:

```text
changed=1
```

Later execution:

```text
ok=1
```

because Nginx is already in the desired state.

### Easy Definition

> Idempotent = Safe to run multiple times.

---

# 28. Why Idempotency Matters

Without idempotency:

```text
❌ Services may restart unnecessarily
❌ Files may be duplicated
❌ Configurations may break
❌ Automation becomes unpredictable
```

With idempotency:

```text
Desired State
     │
     ▼
Check Current State
     │
     ├── Already correct → No change
     │
     └── Different → Apply change
```

---

# 29. Ansible Plugins

Ansible plugins are components that extend or modify how Ansible works.

Examples practiced:

```text
Connection plugins
Lookup plugins
Strategy plugins
Inventory plugins
Filter plugins
```

Example:

```yaml
ansible_connection=local
```

The `local` connection uses an Ansible connection plugin.

Example filter:

```jinja2
{{ "hello ansible" | upper }}
```

The `upper` filter converts the text to uppercase.

---

# 30. Dynamic Inventory

Instead of maintaining a static inventory manually, Ansible can use dynamic inventory.

For AWS:

```yaml
plugin: amazon.aws.aws_ec2
```

This allows Ansible to discover AWS EC2 instances dynamically.

Concept:

```text
AWS
 │
 ├── EC2 Instance 1
 ├── EC2 Instance 2
 └── EC2 Instance 3
        │
        ▼
AWS Dynamic Inventory
        │
        ▼
Ansible
```

---

# 🎯 Key Takeaways

After this practice, I learned how to:

✅ Capture task output using `register`

✅ Read `stdout`, `stderr`, `rc`, `changed`, and `failed`

✅ Provision EC2 instances using Ansible

✅ Launch multiple EC2 instances using loops

✅ Use variables for dynamic infrastructure

✅ Assign different instance types and tags

✅ Manage EC2 instance states

✅ Reboot remote servers

✅ Copy and fetch files

✅ Synchronize directories using rsync

✅ Create website backups

✅ Deploy websites using Nginx

✅ Use handlers

✅ Delegate tasks using `delegate_to`

✅ Execute tasks once using `run_once`

✅ Use Ansible Navigator

✅ Understand Execution Environments

✅ Use FQCN

✅ Understand idempotency

✅ Understand Ansible plugins

✅ Understand AWS dynamic inventory

---

# 🚀 Learning Flow

```text
Ansible Basics
      ↓
Task Output & register
      ↓
AWS EC2 Automation
      ↓
Variables & Loops
      ↓
File Management
      ↓
Nginx Website Deployment
      ↓
Handlers
      ↓
Delegation
      ↓
Ansible Navigator
      ↓
Execution Environments
      ↓
Idempotency
      ↓
Plugins
      ↓
Dynamic Inventory
```

## 📌 Conclusion

This hands-on practice helped me understand how Ansible can be used not only for server configuration but also for AWS infrastructure provisioning, application/website deployment, file management, server operations, and scalable automation.
