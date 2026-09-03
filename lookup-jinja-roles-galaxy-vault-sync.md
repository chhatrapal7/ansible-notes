# Ansible Advanced Concepts

This document covers:

* LAMP / WAMP
* Lookup
* Jinja2 Templates
* Ansible Strategies
* PIP
* Ansible Roles
* Ansible Galaxy
* Nginx Role Example
* Ansible Vault
* Asynchronous & Polling

---

# 1. LAMP

## What is LAMP?

**LAMP** is a commonly used web application stack:

| Letter | Technology      |
| ------ | --------------- |
| L      | Linux           |
| A      | Apache          |
| M      | MySQL / MariaDB |
| P      | PHP             |

> Python can also be used in a Linux + Apache + MySQL environment, but the traditional LAMP acronym uses **PHP**.

### WAMP

**WAMP** stands for:

* **W** → Windows
* **A** → Apache
* **M** → MySQL
* **P** → PHP

---

## LAMP Installation using Ansible

Example for an Amazon Linux 2 managed node:

```yaml
---
- name: Install LAMP Components
  hosts: all
  become: yes

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Install MySQL client
      yum:
        name: mysql
        state: present

    - name: Install PHP
      yum:
        name: php
        state: present
```

### Important Note

Amazon Linux 2 already includes Python, which Ansible normally needs on the managed node.

If the requirement is specifically to install Python 3, use:

```yaml
- name: Install Python 3
  yum:
    name: python3
    state: present
```

However, installing Python is **not the "P" in LAMP**. Traditionally, `P` means PHP.

---

# 2. Ansible Lookup

## What is Lookup?

A **lookup** is used to retrieve data from external sources such as:

* Files
* Environment variables
* Password stores
* Configuration sources
* Other lookup plugins

One common example is reading data from a file.

---

## Example

Create a file on the **Ansible Control Node**:

```bash
vi /root/creds.txt
```

Content:

```text
username: reya
password: 123
```

Now create a playbook:

```yaml
---
- name: Lookup Example
  hosts: all

  vars:
    creds: "{{ lookup('file', '/root/creds.txt') }}"

  tasks:
    - name: Display credentials
      debug:
        msg: "My Credentials are {{ creds }}"
```

Run:

```bash
ansible-playbook lookup.yml
```

### How it works

```text
/root/creds.txt
       |
       v
lookup('file', ...)
       |
       v
creds variable
       |
       v
debug
```

### Important

The `file` lookup reads the file from the **control node**, not from the managed node.

> Do not use `debug` to print real passwords or secrets in production. For sensitive information, use **Ansible Vault** and consider `no_log: true`.

---

# 3. Jinja2 Templates in Ansible

## What is Jinja2?

**Jinja2** is a templating engine used by Ansible to dynamically generate:

* Configuration files
* HTML files
* Scripts
* Application configuration
* Server configuration

Variables are written using:

```jinja2
{{ variable_name }}
```

For example:

```jinja2
server_name {{ server_name }};
listen {{ nginx_port }};
```

During playbook execution, Ansible replaces the variables with their actual values.

---

# 4. Nginx Jinja2 Template Example

## Step 1: Create Template Directory

```bash
mkdir templates
```

Create:

```bash
vi templates/nginx.conf.j2
```

## `templates/nginx.conf.j2`

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    server {
        listen {{ nginx_port }};
        server_name {{ server_name }};

        location / {
            root {{ web_root }};
            index index.html;
        }
    }
}
```

### Variables used

```text
{{ nginx_port }}
{{ server_name }}
{{ web_root }}
```

These variables will be replaced by Ansible.

---

## Step 2: Create Playbook

Create:

```bash
vi nginx.yml
```

```yaml
---
- name: Deploy Nginx Config using Jinja2
  hosts: all
  become: yes

  vars:
    nginx_port: 80
    server_name: mywebsite.com
    web_root: /usr/share/nginx/html

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Deploy Nginx Configuration
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Run:

```bash
ansible-playbook nginx.yml
```

### What happens?

```text
nginx.conf.j2
      |
      | Variables replaced
      v
/etc/nginx/nginx.conf
      |
      | Configuration changed
      v
Notify Handler
      |
      v
Restart Nginx
```

---

# 5. Jinja2 HTTPD Example

Jinja2 can also be used to create Apache HTTPD configuration files.

## Step 1: Create Template

```bash
mkdir -p templates
vi templates/httpd.conf.j2
```

Content:

```apache
Listen {{ http_port }}

<VirtualHost *:{{ http_port }}>
    ServerName {{ server_name }}
    DocumentRoot {{ document_root }}

    <Directory "{{ document_root }}">
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/{{ server_name }}_error.log
    CustomLog /var/log/httpd/{{ server_name }}_access.log combined
</VirtualHost>
```

### Variables

| Variable        | Purpose               |
| --------------- | --------------------- |
| `http_port`     | Apache listening port |
| `server_name`   | Domain/server name    |
| `document_root` | Website document root |

---

## Step 2: Create Playbook

Create:

```bash
vi apache_setup.yml
```

```yaml
---
- name: Configure Apache HTTPD with Jinja2
  hosts: all
  become: yes

  vars:
    http_port: 80
    server_name: example.com
    document_root: /var/www/html

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Deploy Apache Configuration
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/example.conf
      notify: Restart Apache

  handlers:
    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

Run:

```bash
ansible-playbook apache_setup.yml
```

### What this playbook does

1. Installs Apache.
2. Deploys the Jinja2 configuration.
3. If the configuration changes, the handler restarts Apache.

> Using a file under `/etc/httpd/conf.d/` is generally safer than replacing the entire `/etc/httpd/conf/httpd.conf` with a small custom configuration.

---

# 6. Ansible Strategies

## What is a Strategy?

Ansible **strategy** controls how tasks are executed across multiple managed hosts.

Common strategies include:

* `linear`
* `free`
* `host_pinned`

---

# 7. Linear Strategy

`linear` is the **default strategy**.

Ansible waits for all hosts to complete the current task before moving to the next task.

Example:

```text
             Task 1
        ┌──────┼──────┐
        ↓      ↓      ↓
      Node1  Node2  Node3
        │      │      │
        └──────┼──────┘
               ↓
             Task 2
        ┌──────┼──────┐
        ↓      ↓      ↓
      Node1  Node2  Node3
```

For example:

```yaml
---
- name: Linear Strategy Example
  hosts: all
  strategy: linear

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start Apache
      service:
        name: httpd
        state: started
```

### Execution

```text
Task 1
  ↓
All hosts complete Task 1
  ↓
Task 2
  ↓
All hosts complete Task 2
```

---

# 8. Free Strategy

`free` allows each host to progress through the play independently.

A fast host does not have to wait for a slower host to finish the same task.

Example:

```yaml
---
- name: Free Strategy Example
  hosts: all
  strategy: free

  tasks:
    - name: Task 1
      command: sleep 10

    - name: Task 2
      command: echo "Task 2 completed"

    - name: Task 3
      command: echo "Task 3 completed"
```

Conceptually:

```text
Node1                 Node2                 Node3

Task 1                Task 1                Task 1
  ↓                     ↓                     ↓
Task 2                Task 2                Task 2
  ↓                     ↓                     ↓
Task 3                Task 3                Task 3
```

The important difference is that a host can continue without waiting for other hosts.

### Use Case

Useful when:

* Hosts are independent.
* You want faster execution.
* One slow host should not hold up other hosts.

---

# 9. Host Pinned Strategy

`host_pinned` is related to the `free` strategy.

It allows hosts to run independently while maintaining a worker assignment to a host.

It is **not simply "one host at a time."**

For most beginner use cases, remember:

```text
linear      → hosts stay synchronized per task
free        → hosts can progress independently
host_pinned → free-like execution with host-to-worker pinning
```

---

# 10. Debug Strategy

Ansible also provides a `debug` strategy for debugging playbook execution interactively.

Example:

```yaml
---
- name: Debug Strategy Example
  hosts: all
  strategy: debug

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present
```

When a task fails, the debug strategy can help troubleshoot the failure interactively.

---

# 11. PIP

## What is PIP?

**pip** is Python's package manager.

It is used to install Python packages/libraries such as:

* NumPy
* Pandas
* Requests
* Flask
* Django

For example:

```bash
pip install numpy
pip install pandas
```

---

# 12. Installing Python Packages using Ansible

The Ansible `pip` module can install Python packages.

Example:

```yaml
---
- name: Install Python Packages
  hosts: all
  become: yes

  tasks:
    - name: Install pip
      yum:
        name: python3-pip
        state: present

    - name: Install NumPy
      pip:
        name: numpy
        state: present

    - name: Install Pandas
      pip:
        name: pandas
        state: present
```

### Important

On different Linux distributions, the package name for pip can be different.

For Python 3, a common package name is:

```text
python3-pip
```

---

# 13. Ansible Roles

## What is an Ansible Role?

Ansible Roles provide a standardized way to organize large playbooks into reusable components.

A role can contain:

* Tasks
* Handlers
* Variables
* Default variables
* Templates
* Files
* Metadata

### Simple Definition

```text
Playbook = What should happen

Role = Reusable package containing everything
       required to perform a particular job
```

Roles make automation:

* Modular
* Reusable
* Scalable
* Organized

---

# 14. Role Structure

A typical role looks like:

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── files/
    ├── handlers/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── templates/
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    ├── vars/
    │   └── main.yml
    └── README.md
```

---

# 15. Why Use Roles?

### Modular and Reusable

Write once and use the role multiple times.

### Scalable

Easy to manage configurations across many servers.

### Organized

Instead of keeping everything inside one huge playbook, related files are organized inside a role.

---

# 16. Manual Role Creation

Create a project:

```bash
mkdir playbooks
cd playbooks
```

Create role directories:

```bash
mkdir -p roles/pkgs/tasks
mkdir -p roles/users/tasks
mkdir -p roles/webserver/tasks
```

Create the task files:

```bash
touch roles/pkgs/tasks/main.yml
touch roles/users/tasks/main.yml
touch roles/webserver/tasks/main.yml
```

Check structure:

```bash
tree
```

Output:

```text
.
└── roles
    ├── pkgs
    │   └── tasks
    │       └── main.yml
    ├── users
    │   └── tasks
    │       └── main.yml
    └── webserver
        └── tasks
            └── main.yml
```

---

# 17. Package Role

Create:

```bash
vi roles/pkgs/tasks/main.yml
```

```yaml
---
- name: Install packages
  ansible.builtin.yum:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - java-1.8.0-openjdk
    - tree
    - docker
    - maven
```

> Package names can vary by Linux distribution and repository availability.

---

# 18. User Role

Create:

```bash
vi roles/users/tasks/main.yml
```

```yaml
---
- name: Create users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - luckyy
    - imthiaz
    - siva
    - rajesh
    - pavan
```

You can also use:

```yaml
with_items:
  - luckyy
  - imthiaz
  - siva
  - rajesh
  - pavan
```

However, `loop` is the modern and preferred syntax for most new playbooks.

---

# 19. Webserver Role

Create:

```bash
vi roles/webserver/tasks/main.yml
```

```yaml
---
- name: Install Apache
  ansible.builtin.yum:
    name: httpd
    state: present

- name: Start Apache
  ansible.builtin.service:
    name: httpd
    state: started
```

---

# 20. Master Playbook

Create:

```bash
vi master.yml
```

```yaml
---
- name: Execute Ansible Roles
  hosts: all
  become: yes

  roles:
    - pkgs
    - users
    - webserver
```

Run:

```bash
ansible-playbook master.yml
```

### Role execution flow

```text
master.yml
    |
    +---- pkgs
    |       |
    |       +-- Install packages
    |
    +---- users
    |       |
    |       +-- Create users
    |
    +---- webserver
            |
            +-- Install Apache
            +-- Start Apache
```

> Make sure the role name in `master.yml` exactly matches the directory name. For example, if the role directory is `users`, use `users`, not `user`.

---

# 21. Changing `present` to `absent`

If you want to uninstall packages/users managed by your example, you can change:

```yaml
state: present
```

to:

```yaml
state: absent
```

For one file:

```bash
sed -i 's/present/absent/g' roles/pkgs/tasks/main.yml
```

To replace it in all files under the current directory:

```bash
find . -type f -exec sed -i 's/present/absent/g' {} \;
```

### Explanation

```text
find .
```

Searches the current directory and its subdirectories.

```text
-type f
```

Finds files only.

```text
-exec ... \;
```

Executes a command on each file found.

```text
sed -i 's/present/absent/g'
```

Replaces every occurrence of `present` with `absent`.

```text
{}
```

Represents the current file found by `find`.

### Warning

Do not blindly run this command in a directory containing unrelated YAML/configuration files.

A safer approach is to modify the specific role files intentionally.

---

# 22. Ansible Galaxy

## What is Ansible Galaxy?

**Ansible Galaxy** is a platform/ecosystem for finding and sharing Ansible content, including roles and collections.

Official website:

https://galaxy.ansible.com/

You can search for existing roles and install them instead of creating everything from scratch.

Example:

```bash
ansible-galaxy search tomcat
```

Install a role:

```bash
ansible-galaxy role install criecm.tomcat
```

The exact role name should always be checked on Ansible Galaxy before installation.

---

# 23. Role Installation Path

Depending on configuration and environment, roles can be installed into Ansible's configured role search paths.

You can check your configuration with:

```bash
ansible-config dump | grep DEFAULT_ROLES_PATH
```

You can configure a custom role path in `ansible.cfg`.

Example:

```ini
[defaults]
roles_path = /etc/ansible/roles
```

> Configuration option names are case-insensitive in `ansible.cfg`; `roles_path` is the commonly documented form.

---

# 24. Creating a Role using `ansible-galaxy init`

Instead of manually creating all directories, use:

```bash
ansible-galaxy init nginx
```

Or:

```bash
ansible-galaxy init roles/nginx
```

This automatically creates the standard role structure.

Example:

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── files/
    ├── handlers/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── README.md
    ├── tasks/
    │   └── main.yml
    ├── templates/
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    └── vars/
        └── main.yml
```

---

# 25. Role Directory Purpose

| Directory    | Purpose                          |
| ------------ | -------------------------------- |
| `tasks/`     | Main tasks performed by the role |
| `handlers/`  | Handlers triggered by `notify`   |
| `templates/` | Jinja2 templates                 |
| `files/`     | Static files                     |
| `vars/`      | Role variables                   |
| `defaults/`  | Default variables                |
| `meta/`      | Role metadata and dependencies   |
| `tests/`     | Role testing files               |
| `README.md`  | Role documentation               |

---

# 26. Nginx Role Example

Create a project:

```bash
mkdir nginx-role-project
cd nginx-role-project
```

Create the role:

```bash
ansible-galaxy init roles/nginx
```

Check:

```bash
tree
```

---

## Step 1: Create Default Variables

Edit:

```bash
vi roles/nginx/defaults/main.yml
```

```yaml
---
nginx_package: nginx
nginx_service: nginx
nginx_port: 80
```

---

# 27. Nginx Role Tasks

Edit:

```bash
vi roles/nginx/tasks/main.yml
```

```yaml
---
- name: Install Nginx
  ansible.builtin.package:
    name: "{{ nginx_package }}"
    state: present

- name: Start and enable Nginx
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: started
    enabled: true

- name: Deploy custom index page
  ansible.builtin.template:
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html
  notify: Restart Nginx
```

---

# 28. Nginx Handler

Edit:

```bash
vi roles/nginx/handlers/main.yml
```

```yaml
---
- name: Restart Nginx
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: restarted
```

The handler runs when a task sends:

```yaml
notify: Restart Nginx
```

The handler normally runs at the end of the play if it has been notified.

---

# 29. Nginx Jinja2 Template

Edit:

```bash
vi roles/nginx/templates/index.html.j2
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>DNA by Reyaz</title>
</head>
<body>
    <h1>Welcome to NGINX Server</h1>
    <p>Deployed using Ansible Role</p>
    <p>Running on port {{ nginx_port }}</p>
</body>
</html>
```

The variable:

```jinja2
{{ nginx_port }}
```

will be replaced with:

```text
80
```

---

# 30. Site Playbook

Create:

```bash
vi site.yml
```

```yaml
---
- name: Install Nginx using Role
  hosts: prod
  become: true

  roles:
    - nginx
```

Run:

```bash
ansible-playbook site.yml
```

### Execution flow

```text
site.yml
   |
   v
nginx role
   |
   +-- Install Nginx
   |
   +-- Start and enable Nginx
   |
   +-- Deploy index.html
   |
   +-- Notify Restart Nginx
              |
              v
        Restart Nginx
```

---

# 31. Ansible Vault

## What is Ansible Vault?

**Ansible Vault** is used to encrypt sensitive data.

Examples:

* Passwords
* API keys
* Database credentials
* Private keys
* Sensitive variables

Vault helps prevent sensitive information from being stored as plain text.

---

# 32. Creating a Vault File

Create:

```bash
ansible-vault create secret.yml
```

Ansible will ask for a Vault password.

Example content:

```yaml
db_password: mysecurepassword
api_key: "12345-abcde-67890"
```

The file will be stored in encrypted form.

Check:

```bash
cat secret.yml
```

You will see encrypted content rather than the original YAML.

---

# 33. Edit Vault File

To edit the encrypted file:

```bash
ansible-vault edit secret.yml
```

Enter the Vault password.

Make your changes and save the file.

The file remains encrypted on disk.

---

# 34. Rekey Vault

To change the Vault password:

```bash
ansible-vault rekey secret.yml
```

Ansible will ask for:

1. Old password
2. New password

---

# 35. Decrypt Vault

To permanently decrypt the file:

```bash
ansible-vault decrypt secret.yml
```

After this, the file becomes normal plaintext.

Check:

```bash
cat secret.yml
```

---

# 36. Encrypt an Existing File

If you have an existing plaintext file:

```bash
ansible-vault encrypt secret.yml
```

The file will become encrypted.

---

# 37. View Vault Content

To view the decrypted content without permanently decrypting the file:

```bash
ansible-vault view secret.yml
```

This is different from:

```bash
ansible-vault decrypt secret.yml
```

### Difference

| Command         | Result                                              |
| --------------- | --------------------------------------------------- |
| `vault view`    | Displays decrypted content but keeps file encrypted |
| `vault decrypt` | Permanently decrypts the file                       |

---

# 38. Using Vault Variables in a Playbook

Example:

```yaml
---
- name: Use Vault Variables
  hosts: all

  vars_files:
    - secret.yml

  tasks:
    - name: Show database password
      debug:
        msg: "{{ db_password }}"
```

> Do not print real secrets using `debug` in production.

A safer approach for sensitive tasks is:

```yaml
- name: Use sensitive credential
  some_module:
    password: "{{ db_password }}"
  no_log: true
```

---

# 39. Run Playbook with Vault

If Ansible needs the Vault password interactively:

```bash
ansible-playbook site.yml --ask-vault-pass
```

You can also use a Vault password file:

```bash
ansible-playbook site.yml --vault-password-file .vault_pass
```

### Important

Never commit the Vault password file to GitHub.

Add it to `.gitignore`:

```gitignore
.vault_pass
```

---

# 40. Asynchronous and Polling Actions

## Synchronous Execution

By default, Ansible executes tasks synchronously.

This means:

```text
Task 1
  ↓
Wait until Task 1 finishes
  ↓
Task 2
  ↓
Wait until Task 2 finishes
  ↓
Task 3
```

For long-running tasks, Ansible provides **asynchronous execution**.

---

# 41. `async` and `poll`

Two important parameters are:

```yaml
async:
poll:
```

### `async`

Defines the maximum amount of time, in seconds, that an asynchronous job is allowed to run.

Example:

```yaml
async: 300
```

Means:

```text
Maximum runtime = 300 seconds
```

### `poll`

Defines how often Ansible checks the status of the asynchronous job.

Example:

```yaml
poll: 10
```

Means Ansible checks approximately every 10 seconds.

> `poll` is **not** the timeout.

---

# 42. Async Example

Original example:

```yaml
---
- name: Async and Poll Playbook
  hosts: all

  tasks:
    - name: Sleep for 30 seconds
      command: sleep 30
      async: 20
      poll: 10

    - name: Install Git
      yum:
        name: git
        state: present
```

### Problem with this example

The command needs:

```text
30 seconds
```

But:

```yaml
async: 20
```

allows only:

```text
20 seconds
```

So the async job can time out before `sleep 30` finishes.

A better example is:

```yaml
---
- name: Async and Poll Playbook
  hosts: all
  become: yes

  tasks:
    - name: Sleep for 30 seconds
      ansible.builtin.command: sleep 30
      async: 60
      poll: 10

    - name: Install Git
      ansible.builtin.yum:
        name: git
        state: present
```

Here:

```text
sleep = 30 seconds
async = 60 seconds
poll  = 10 seconds
```

So Ansible allows the task up to 60 seconds and checks its status periodically.

---

# 43. `poll: 0`

When:

```yaml
poll: 0
```

Ansible starts the asynchronous task and does not wait for it to finish before continuing.

Example:

```yaml
---
- name: Fire and Forget Example
  hosts: all

  tasks:
    - name: Run long task
      ansible.builtin.command: /opt/backup.sh
      async: 600
      poll: 0

    - name: Continue immediately
      ansible.builtin.debug:
        msg: "Ansible continues without waiting for the backup."
```

Conceptually:

```text
Start Task 1
     |
     | poll: 0
     ↓
Do NOT wait
     |
     ↓
Task 2 starts
```

---

# 44. Checking a Background Job with `async_status`

When using:

```yaml
poll: 0
```

you can register the job information and later check its status.

Example:

```yaml
---
- name: Async Status Example
  hosts: all

  tasks:
    - name: Start backup
      ansible.builtin.command: /opt/backup.sh
      async: 1800
      poll: 0
      register: backup_job

    - name: Check backup status
      ansible.builtin.async_status:
        jid: "{{ backup_job.ansible_job_id }}"
      register: backup_result
      until: backup_result.finished
      retries: 60
      delay: 30
```

### Explanation

```text
Start backup
     |
     | poll: 0
     ↓
Continue playbook
     |
     ↓
async_status
     |
     ↓
Check job
     |
     ├── Not finished → wait 30 sec → check again
     |
     └── Finished → continue
```

---

# 45. Async Quick Reference

| Setting        | Meaning                                |
| -------------- | -------------------------------------- |
| `async: 300`   | Job can run for up to 300 seconds      |
| `poll: 10`     | Check job status every ~10 seconds     |
| `poll: 0`      | Start job and do not wait              |
| `async_status` | Check status of a background async job |

### Easy way to remember

```text
async = Kitna maximum time task ko diya?

poll = Kitne interval mein status check karna hai?
```

---

# 46. Overall Ansible Concepts

The concepts covered in this document fit together like this:

```text
                    ANSIBLE
                       |
       +---------------+----------------+
       |               |                |
    Playbooks        Roles          Templates
       |               |                |
       |               |             Jinja2
       |               |
       |          Reusable Structure
       |
   Strategies
       |
   +---+---+
   |       |
linear    free

       +-----------------------------+
       |                             |
     Vault                         Async
       |                             |
   Secrets                     Long-running jobs
```

---

# 47. Important Commands

## Playbook

```bash
ansible-playbook site.yml
```

## Syntax Check

```bash
ansible-playbook site.yml --syntax-check
```

## Vault

```bash
ansible-vault create secret.yml
ansible-vault edit secret.yml
ansible-vault view secret.yml
ansible-vault rekey secret.yml
ansible-vault encrypt secret.yml
ansible-vault decrypt secret.yml
```

## Galaxy

```bash
ansible-galaxy search tomcat
ansible-galaxy role install criecm.tomcat
ansible-galaxy init roles/nginx
```

## Directory Tree

```bash
tree
```

---

# 48. Quick Revision

### LAMP

```text
Linux + Apache + MySQL + PHP
```

### Lookup

```text
Used to retrieve data from external sources.
```

### Jinja2

```text
Used for dynamic templates.

{{ variable }}
```

### Strategy

```text
linear → synchronized task execution
free   → hosts progress independently
```

### PIP

```text
Python package manager
```

### Role

```text
Reusable and organized Ansible automation
```

### Galaxy

```text
Repository/ecosystem for Ansible content
```

### Vault

```text
Encrypt sensitive data
```

### Async

```text
Run long-running jobs asynchronously
```

### Poll

```text
Check the status of an async job periodically
```

---

# 49. One-Line Interview Definitions

**LAMP:**
LAMP is a web application stack consisting traditionally of Linux, Apache, MySQL/MariaDB, and PHP.

**Lookup:**
Ansible lookup plugins retrieve data from external sources such as files and environment variables.

**Jinja2:**
Jinja2 is a templating engine used by Ansible to generate dynamic configuration and other files.

**Strategy:**
Ansible strategy controls how tasks are scheduled and executed across managed hosts.

**PIP:**
pip is the package manager used to install Python packages.

**Role:**
An Ansible role is a reusable, standardized directory structure containing tasks, handlers, variables, templates, files, and metadata.

**Ansible Galaxy:**
Ansible Galaxy is a platform/ecosystem for finding and sharing Ansible content.

**Ansible Vault:**
Ansible Vault encrypts sensitive data such as passwords and API keys.

**Async:**
`async` allows a task to run asynchronously for a specified maximum duration.

**Poll:**
`poll` specifies how frequently Ansible checks the status of an asynchronous job.
