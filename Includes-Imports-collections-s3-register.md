# Ansible: Includes, Imports, Collections, S3 & Register

> **Notes by Chhatrapal**
>
> GitHub-ready learning notes with corrected YAML syntax and terminology.

---

## 1. Includes & Imports in Ansible

As playbooks grow larger, putting everything into one YAML file becomes difficult to manage.

Instead of writing 500–1000 lines in one file, Ansible lets us split automation into multiple files.

There are three important ways:

| Keyword | Purpose | Type |
|---|---|---|
| `import_tasks` | Import another task file | Static |
| `include_tasks` | Include another task file | Dynamic |
| `import_playbook` | Import another complete playbook | Static |

---

## 2. `import_tasks`

`import_tasks` statically imports a task file when the playbook is parsed.

### Directory structure

```text
ansible/
├── site.yml
├── install.yml
├── configure.yml
├── deploy.yml
└── inventory
```

### `install.yml`

```yaml
---
- name: Install Nginx
  ansible.builtin.dnf:
    name: nginx
    state: present
```

### `configure.yml`

```yaml
---
- name: Copy index.html
  ansible.builtin.copy:
    src: index.html
    dest: /usr/share/nginx/html/index.html
```

### `site.yml`

```yaml
---
- name: Configure web server
  hosts: all
  become: true

  tasks:
    - name: Import installation tasks
      ansible.builtin.import_tasks: install.yml

    - name: Import configuration tasks
      ansible.builtin.import_tasks: configure.yml
```

Run:

```bash
ansible-playbook site.yml
```

### When to use `import_tasks`

Use it when:

- Tasks are static.
- The workflow is fixed.
- The task file is always required.
- You want a predictable production workflow.

---

## 3. `include_tasks`

`include_tasks` loads a task file dynamically at runtime.

This is useful when conditions, variables, or loops decide which tasks should run.

Think of it as:

> "During execution, decide whether I need to load and run these tasks."

### `install.yml`

```yaml
---
- name: Install Apache
  ansible.builtin.dnf:
    name: httpd
    state: present
```

### `site.yml`

```yaml
---
- name: Install web server
  hosts: prod
  become: true

  vars:
    install_webserver: true

  tasks:
    - name: Include installation tasks
      ansible.builtin.include_tasks: install.yml
      when: install_webserver
```

If:

```yaml
install_webserver: true
```

the task file is included.

If:

```yaml
install_webserver: false
```

the task file is skipped.

### Variable-driven example

Suppose we support two web servers:

```yaml
webserver: nginx
```

Then:

```yaml
- name: Include Nginx tasks
  ansible.builtin.include_tasks: nginx.yml
  when: webserver == "nginx"

- name: Include Apache tasks
  ansible.builtin.include_tasks: apache.yml
  when: webserver == "apache"
```

Only the matching task file is included.

---

## 4. `import_playbook`

`import_playbook` imports an entire playbook, not just a list of tasks.

### `install.yml`

```yaml
---
- name: Install Nginx
  hosts: all
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present
```

### `deploy.yml`

```yaml
---
- name: Deploy website
  hosts: all
  become: true

  tasks:
    - name: Copy index.html
      ansible.builtin.copy:
        src: index.html
        dest: /usr/share/nginx/html/index.html
```

### `backup.yml`

```yaml
---
- name: Backup website
  hosts: all
  become: true

  tasks:
    - name: Create a backup
      ansible.builtin.shell: cp index.html index-copy.html
```

### `master.yml`

```yaml
---
- name: Run installation playbook
  ansible.builtin.import_playbook: install.yml

- name: Run deployment playbook
  ansible.builtin.import_playbook: deploy.yml

- name: Run backup playbook
  ansible.builtin.import_playbook: backup.yml
```

Run:

```bash
ansible-playbook master.yml
```

---

## 5. Quick Difference

```text
Main Playbook
      |
      +-- import_tasks
      |      |
      |      +-- Static
      |      +-- Task file
      |
      +-- include_tasks
      |      |
      |      +-- Dynamic
      |      +-- Task file
      |
      +-- import_playbook
             |
             +-- Entire playbook
```

### Summary

- `import_tasks` → static task-file import.
- `include_tasks` → dynamic task-file inclusion.
- `import_playbook` → imports another complete playbook.

---

# 6. Ansible Collections

An **Ansible Collection** is a standardized bundle of Ansible content for a particular technology, vendor, platform, or use case.

A collection can contain modules, plugins, roles, and documentation.

Think of a collection as a **toolbox** for a particular technology.

### Example: AWS

For AWS automation, the `amazon.aws` collection provides AWS-related Ansible content.

Check installed collections:

```bash
ansible-galaxy collection list
```

Install the AWS collection:

```bash
ansible-galaxy collection install amazon.aws
```

---

# 7. S3 Bucket with Ansible

To manage AWS resources using the `amazon.aws` collection, Python libraries such as `boto3` and `botocore` are required.

Install them on the machine where the AWS module will execute:

```bash
sudo dnf install python3-pip -y
pip3 install boto3 botocore
```

For this example, the S3 task runs on the Ansible control node.

### S3 Playbook

```yaml
---
- name: Create S3 bucket by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Create S3 bucket
      amazon.aws.s3_bucket:
        name: unique-demo-ansible-bukt-845
        state: present
        region: ap-south-1
```

Run:

```bash
ansible-playbook s3.yml
```

### Why `localhost`?

S3 is an AWS cloud resource. We do not need to run the bucket-creation task separately on every managed server.

Using:

```yaml
hosts: localhost
connection: local
```

makes the AWS API call from the Ansible control node.

### AWS credentials

The machine executing the S3 module must have access to valid AWS credentials.

You can verify AWS authentication with:

```bash
aws sts get-caller-identity
```

If credentials are configured through the AWS CLI:

```bash
aws configure
```

> Never hard-code AWS access keys or secret keys directly in a GitHub repository.

### Delete the bucket

To ensure the bucket is absent:

```yaml
---
- name: Delete S3 bucket by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Delete S3 bucket
      amazon.aws.s3_bucket:
        name: unique-demo-ansible-bukt-845
        state: absent
        region: ap-south-1
```

---

# 8. `register` in Ansible

The `register` keyword stores the result of a task in a variable.

That variable can be used later for:

- Debugging
- Reporting
- Conditional execution
- Using command output in later tasks

Common result fields include:

| Field | Purpose |
|---|---|
| `stdout` | Standard output |
| `stderr` | Standard error |
| `rc` | Return code |
| `changed` | Whether the task changed something |
| `failed` | Whether the task failed |

---

## 9. Register Example

```yaml
---
- name: Register example by Chhatrapal
  hosts: all
  gather_facts: false

  tasks:
    - name: Check hostname
      ansible.builtin.command: hostname
      register: result

    - name: Display hostname
      ansible.builtin.debug:
        msg: "Hostname: {{ result.stdout }}"

    - name: Display return code
      ansible.builtin.debug:
        msg: "Return Code: {{ result.rc }}"

    - name: Display changed status
      ansible.builtin.debug:
        msg: "Changed: {{ result.changed }}"

    - name: Display failed status
      ansible.builtin.debug:
        msg: "Failed: {{ result.failed }}"
```

Example result:

```text
Hostname: ip-172-31-20-91
Return Code: 0
Changed: true
Failed: false
```

> Note: `command: hostname` normally reports `changed: true` because the command module executes the command. `changed` does not mean that the hostname itself was modified.

---

## 10. Register with a Failed Command

```yaml
---
- name: Failure example by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Run invalid command
      ansible.builtin.command: xyz123
      register: result
      ignore_errors: true

    - name: Display error
      ansible.builtin.debug:
        var: result.stderr

    - name: Display return code
      ansible.builtin.debug:
        var: result.rc

    - name: Display failed status
      ansible.builtin.debug:
        var: result.failed
```

Typical result:

```text
rc: 127
failed: true
```

The exact `stderr` text can differ depending on the operating system.

---

## 11. What does `register` do?

**Answer:**

`register` captures the result of a task and stores it in a variable. The registered variable can contain useful information such as `stdout`, `stderr`, `rc`, `changed`, and `failed`.

Example:

```yaml
register: result
```

Later:

```yaml
{{ result.stdout }}
{{ result.rc }}
{{ result.changed }}
{{ result.failed }}
```

---

## 12. Important GitHub Practice

Do not commit sensitive information such as:

- AWS access keys
- AWS secret keys
- Passwords
- Private SSH keys
- Vault passwords
- API tokens

Use:

- Ansible Vault
- Environment variables
- IAM roles
- GitHub Actions secrets
- AWS credential/configuration mechanisms

instead of hard-coding credentials in YAML files.
