# Ansible: Dictionaries, Handlers, Blocks & Shell

---

## Table of Contents

1. [Dictionaries in Ansible](#1-dictionaries-in-ansible)
2. [List of Dictionaries](#2-list-of-dictionaries)
3. [Nested Dictionaries](#3-nested-dictionaries)
4. [Notify and Handlers](#4-notify-and-handlers)
5. [Handlers with Apache](#5-handlers-with-apache)
6. [Blocks in Ansible](#6-blocks-in-ansible)
7. [Block + Rescue + Always + Handlers](#7-block--rescue--always--handlers)
8. [Shell in Ansible Playbook](#8-shell-in-ansible-playbook)
9. [Shell vs Command vs Raw](#9-shell-vs-command-vs-raw)
10. [Quick Revision](#10-quick-revision)

---

# 1. Dictionaries in Ansible

A **dictionary** is a collection of **key-value pairs**.

It is similar to:

* Python dictionary
* JSON object

### Simple Variable

Instead of storing only one value:

```yaml
package: httpd
```

we can store multiple related values together:

```yaml
server:
  name: Web-Server
  package: httpd
  port: 80
```

Here:

```text
server
 ├── name    → Web-Server
 ├── package → httpd
 └── port    → 80
```

So `server` is the main key, and it contains multiple key-value pairs.

---

## Accessing Dictionary Values

We can access dictionary values using **dot notation**:

```text
server.name
server.package
server.port
```

In Ansible/Jinja:

```yaml
"{{ server.name }}"
"{{ server.package }}"
"{{ server.port }}"
```

### Example

Create `pb.yml`:

```yaml
---
- name: Dictionary Playbook
  hosts: all
  gather_facts: false

  vars:
    server:
      name: Web-Server
      package: httpd
      port: 80

  tasks:

    - name: Display server name
      debug:
        msg: "{{ server.name }}"

    - name: Display package
      debug:
        msg: "{{ server.package }}"

    - name: Display port
      debug:
        msg: "{{ server.port }}"
```

Run:

```bash
ansible-playbook pb.yml
```

### Expected Values

```text
server.name    → Web-Server
server.package → httpd
server.port    → 80
```

---

# 2. List of Dictionaries

A **list of dictionaries** means:

> Multiple dictionary objects are stored inside a list.

Example:

```yaml
users:
  - name: reyaz
    uid: 1001

  - name: john
    uid: 1002

  - name: alice
    uid: 1003
```

Here:

```text
users
 │
 ├── Dictionary 1
 │    ├── name → reyaz
 │    └── uid  → 1001
 │
 ├── Dictionary 2
 │    ├── name → john
 │    └── uid  → 1002
 │
 └── Dictionary 3
      ├── name → alice
      └── uid  → 1003
```

The `-` indicates a new item in the list.

---

## Create Multiple Users Using List of Dictionaries

Example:

```yaml
---
- name: List of Dictionaries
  hosts: all
  become: yes

  vars:
    users:
      - name: reyaz
        uid: 1001

      - name: john
        uid: 1002

      - name: alice
        uid: 1003

  tasks:

    - name: Create Users
      user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        state: present
      loop: "{{ users }}"
```

### How the Loop Works

First iteration:

```text
item.name → reyaz
item.uid  → 1001
```

Second iteration:

```text
item.name → john
item.uid  → 1002
```

Third iteration:

```text
item.name → alice
item.uid  → 1003
```

So Ansible effectively performs:

```text
Create reyaz  → UID 1001
Create john   → UID 1002
Create alice  → UID 1003
```

### Important

`users` is a list, and each item in the list is a dictionary.

```yaml
users:
  - name: reyaz
    uid: 1001
```

Therefore, we use:

```yaml
item.name
item.uid
```

inside the loop.

---

# 3. Nested Dictionaries

A dictionary can contain another dictionary.

This is called a **nested dictionary**.

Example:

```yaml
vars:

  employee:
    name: Reyaz

    address:
      city: Hyderabad
      state: Telangana
      country: India
```

Structure:

```text
employee
 │
 ├── name → Reyaz
 │
 └── address
      ├── city    → Hyderabad
      ├── state   → Telangana
      └── country → India
```

---

## Accessing Nested Dictionary Values

### Employee Name

```yaml
"{{ employee.name }}"
```

Output:

```text
Reyaz
```

### Employee City

```yaml
"{{ employee.address.city }}"
```

Output:

```text
Hyderabad
```

### Employee Country

```yaml
"{{ employee.address.country }}"
```

Output:

```text
India
```

### General Pattern

```text
parent.child
```

For nested data:

```text
parent.child.grandchild
```

Example:

```text
employee.address.city
```

---

# 4. Notify and Handlers

## What is a Handler?

A **handler** is a special type of Ansible task that runs **only when it is notified**.

Handlers are commonly used when a service needs to be:

* Started
* Restarted
* Reloaded

For example:

```text
Configuration changed
        ↓
     notify
        ↓
     handler
        ↓
Restart Apache
```

---

## Why Do We Need Handlers?

Suppose we modify an Apache configuration file.

We don't want to restart Apache every time the playbook runs.

We want:

```text
If configuration changed
        ↓
Restart Apache
```

If nothing changed:

```text
No change
   ↓
No restart
```

This prevents unnecessary service restarts.

---

# 5. Handlers with Apache

Create:

```bash
vi handlers.yml
```

Example:

```yaml
---
- name: Handlers
  hosts: all
  become: yes

  tasks:

    - name: Installing Apache
      yum:
        name: httpd
        state: present
      notify: Starting Apache

  handlers:

    - name: Starting Apache
      service:
        name: httpd
        state: started
```

Run:

```bash
ansible-playbook handlers.yml
```

---

## How `notify` Works

This task:

```yaml
- name: Installing Apache
  yum:
    name: httpd
    state: present
  notify: Starting Apache
```

means:

> If this task makes a change, notify the handler named `Starting Apache`.

Handler:

```yaml
handlers:

  - name: Starting Apache
    service:
      name: httpd
      state: started
```

Flow:

```text
Install Apache
      ↓
Did task change anything?
      ↓
   YES
      ↓
   notify
      ↓
Starting Apache handler
```

If Apache is already installed and the task makes no change:

```text
Install Apache
      ↓
No change
      ↓
Handler not triggered
```

---

## Important Handler Point

Handlers normally execute **after the regular tasks in the play have completed**.

So `notify` does not mean:

> "Run the handler immediately."

It means:

> "Schedule this handler to run when Ansible reaches the handler execution point."

---

# Uninstall Example

Suppose we change:

```yaml
state: present
```

to:

```yaml
state: absent
```

Then Apache will be removed.

However, if you notify:

```yaml
notify: Starting Apache
```

the handler will still be triggered when the uninstall task reports a change.

But Apache no longer exists, so starting the service can fail.

Example:

```yaml
---
- name: Handlers Uninstall
  hosts: all
  become: yes

  tasks:

    - name: Uninstalling Apache
      yum:
        name: httpd
        state: absent
      notify: Starting Apache

  handlers:

    - name: Starting Apache
      service:
        name: httpd
        state: started
      ignore_errors: yes
```

Run:

```bash
ansible-playbook handlersuninstall.yml
```

### Better Understanding

The issue here is not that `notify` is wrong.

The logic itself is unusual:

```text
Remove Apache
     ↓
notify "Start Apache"
     ↓
Apache doesn't exist
     ↓
Start service fails
```

So in a real project, you would normally notify a handler that makes sense for the operation being performed.

---

# Another Handler Example

This example shows two different handlers:

* Start Apache
* Restart Apache

```yaml
---
- name: Simple Web Server with Handlers
  hosts: all
  become: yes

  tasks:

    - name: Install Apache web server
      yum:
        name: httpd
        state: present
      notify: Start Apache

    - name: Copy simple HTML page
      copy:
        content: "<h1>Hello from Ansible Class with YUM!</h1>"
        dest: /var/www/html/index.html
      notify: Restart Apache

  handlers:

    - name: Start Apache
      service:
        name: httpd
        state: started

    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

### Flow

```text
Install Apache
      ↓
   changed?
      ↓
     YES
      ↓
notify → Start Apache


Copy index.html
      ↓
   changed?
      ↓
     YES
      ↓
notify → Restart Apache
```

---

# 6. Blocks in Ansible

A **block** groups multiple related tasks together.

It is useful when you want to:

* Group tasks
* Apply common settings
* Handle failures
* Use `rescue`
* Use `always`

Basic structure:

```yaml
- block:

    - task 1
    - task 2
    - task 3

  rescue:

    - recovery task

  always:

    - cleanup/status task
```

---

## Block vs If/Else

`block/rescue/always` is **not exactly if/else**.

A better programming analogy is:

| Programming | Ansible  |
| ----------- | -------- |
| `try`       | `block`  |
| `catch`     | `rescue` |
| `finally`   | `always` |
| `if`        | `when`   |

### Remember

```text
when   → condition
block  → group tasks
rescue → handle failure
always → always execute
```

---

# 7. Block + Rescue + Always + Handlers

This is a useful real-world deployment example.

It demonstrates:

* `block`
* `rescue`
* `always`
* `notify`
* `handlers`
* `register`
* Rollback
* Service management

```yaml
---
- name: Deploy Nginx Website
  hosts: all
  become: yes

  vars:
    webpage: |
      <html>
      <head><title>DNA by Reyaz</title></head>
      <body>
        <h1>Welcome to DNA by Reyaz</h1>
        <h2>Nginx deployed successfully using Ansible!</h2>
      </body>
      </html>

  tasks:

    - block:

        - name: Install Nginx
          dnf:
            name: nginx
            state: present

        - name: Deploy index.html
          copy:
            content: "{{ webpage }}"
            dest: /usr/share/nginx/html/index.html
            owner: root
            group: root
            mode: '0644'
          notify: Restart Nginx

        - name: Start and Enable Nginx
          service:
            name: nginx
            state: started
            enabled: yes

      rescue:

        - name: Display Failure Message
          debug:
            msg: "Deployment failed. Starting rollback..."

        - name: Remove Nginx Package
          dnf:
            name: nginx
            state: absent
          ignore_errors: yes

        - name: Remove index.html
          file:
            path: /usr/share/nginx/html/index.html
            state: absent
          ignore_errors: yes

      always:

        - name: Deployment Status
          debug:
            msg: "Deployment process completed."

        - name: Collect Nginx Status
          shell: systemctl status nginx --no-pager
          register: nginx_status
          ignore_errors: yes

        - name: Show Service Status
          debug:
            var: nginx_status.stdout_lines

  handlers:

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

---

## Block

The `block` contains the main deployment steps:

```text
1. Install Nginx
2. Deploy index.html
3. Start and enable Nginx
```

---

## `notify`

```yaml
notify: Restart Nginx
```

This does **not** restart Nginx immediately.

It tells Ansible:

> If this task changes something, trigger the `Restart Nginx` handler.

For example:

```text
index.html changed
       ↓
notify
       ↓
Restart Nginx handler
```

If `index.html` is already the same:

```text
No change
   ↓
Handler not triggered
```

---

## Handler

```yaml
handlers:

  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```

Handlers are useful because they prevent unnecessary service restarts.

---

# `rescue`

`rescue` runs when a task inside the block fails.

Example:

```text
Install Nginx       ✅
       ↓
Copy index.html     ❌
       ↓
Start Nginx         ⏭️
       ↓
rescue              ✅
```

Typical rescue actions:

* Rollback changes
* Remove partially installed software
* Restore backups
* Cleanup files
* Send alerts

---

# `always`

`always` runs after the block/rescue processing regardless of whether the block succeeded or failed.

Typical uses:

* Collect logs
* Print deployment status
* Cleanup temporary files
* Check service status
* Send notifications

Flow:

```text
              BLOCK
                │
       ┌────────┴────────┐
       │                 │
    SUCCESS           FAILURE
       │                 │
       │              RESCUE
       │                 │
       └────────┬────────┘
                ↓
             ALWAYS
```

---

# 8. Shell in Ansible Playbook

Ansible can execute Linux commands through different modules.

Common options are:

* `command`
* `shell`
* `raw`

However, they are **not all the same**.

---

## `command`

Use `command` when you simply need to execute a command and don't need shell features.

Example:

```yaml
- name: Check Git version
  command: git --version
```

`command` does not process shell features such as:

```text
|
>
<
&&
||
*
$
```

---

## `shell`

Use `shell` when you specifically need shell functionality.

For example:

```yaml
- name: Show running processes
  shell: ps aux | grep nginx
```

Here `|` is a shell pipe, so `shell` is appropriate.

---

## `raw`

`raw` executes the command directly on the remote machine without requiring the normal Ansible module execution environment.

It is particularly useful for **bootstrapping** systems where Python may not yet be installed.

Example:

```yaml
- name: Install Python
  raw: yum install python3 -y
```

Once the required environment is available, normal Ansible modules should generally be preferred.

---

# 9. Shell vs Command vs Raw

| Module    | Use Case                                                |
| --------- | ------------------------------------------------------- |
| `command` | Normal command execution                                |
| `shell`   | Shell features like pipes, redirection, `&&`, variables |
| `raw`     | Bootstrapping / systems where Python is unavailable     |

### Easy Rule

```text
Normal command?
      ↓
   command

Need shell features?
      ↓
    shell

Remote system doesn't have Python / bootstrap?
      ↓
     raw
```

### Best Practice

> **Use `command` whenever possible.**

> **Use `shell` only when shell features are actually required.**

> **Use `raw` mainly for bootstrapping systems where normal Ansible modules cannot yet run.**

---

# 10. Shell Playbook Example

Create:

```bash
vi shell.yml
```

Example:

```yaml
---
- name: Shell in Playbook
  hosts: all
  become: yes

  tasks:

    - name: Installing Apache
      shell: yum install httpd -y

    - name: Installing Git
      command: yum install git -y

    - name: Installing Maven
      raw: yum install maven -y
```

Run:

```bash
ansible-playbook shell.yml
```

---

## Important Note About This Example

For package installation, using `shell`, `command`, or `raw` is generally **not the preferred Ansible approach**.

Instead of:

```yaml
shell: yum install httpd -y
```

prefer:

```yaml
yum:
  name: httpd
  state: present
```

Why?

Because Ansible modules provide **idempotency**.

For example:

```yaml
yum:
  name: httpd
  state: present
```

means:

> Make sure Apache is installed.

If it is already installed, Ansible doesn't need to reinstall it.

---

# Uninstall Example

If you want to remove the packages:

```yaml
---
- name: Remove Packages
  hosts: all
  become: yes

  tasks:

    - name: Remove Apache
      yum:
        name: httpd
        state: absent

    - name: Remove Git
      yum:
        name: git
        state: absent

    - name: Remove Maven
      yum:
        name: maven
        state: absent
```

Run:

```bash
ansible-playbook shellremove.yml
```

### Preferred Approach

For package management:

```text
yum/dnf module
     ↓
preferred
```

instead of:

```text
shell → yum install
command → yum install
```

---

# Quick Revision

## Dictionaries

```yaml
server:
  name: Web-Server
  package: httpd
  port: 80
```

Access:

```yaml
{{ server.name }}
{{ server.package }}
{{ server.port }}
```

---

## List of Dictionaries

```yaml
users:
  - name: reyaz
    uid: 1001
  - name: john
    uid: 1002
```

Loop:

```yaml
loop: "{{ users }}"
```

Access:

```yaml
{{ item.name }}
{{ item.uid }}
```

---

## Nested Dictionary

```yaml
employee:
  name: Reyaz
  address:
    city: Hyderabad
    state: Telangana
```

Access:

```yaml
{{ employee.name }}
{{ employee.address.city }}
```

---

## Notify + Handler

```text
Task changes something
        ↓
     notify
        ↓
    handler
        ↓
restart/reload service
```

---

## Block

```yaml
block:
  - task

rescue:
  - recovery

always:
  - cleanup
```

Programming analogy:

```text
try     → block
catch   → rescue
finally → always
```

---

## Command vs Shell vs Raw

```text
command → normal command
shell   → shell features
raw     → bootstrap systems without Python
```

---

# Interview Points

### Q1. What is a dictionary in Ansible?

A dictionary is a collection of key-value pairs used to group related data.

### Q2. How do you access dictionary values?

Using dot notation:

```yaml
{{ server.package }}
```

### Q3. What is a list of dictionaries?

A list containing multiple dictionary objects.

Example:

```yaml
users:
  - name: reyaz
    uid: 1001
  - name: john
    uid: 1002
```

### Q4. What is a handler?

A handler is a special task that runs when notified by another task.

### Q5. When does a handler normally run?

Handlers normally run after regular tasks have completed, when they have been notified.

### Q6. What is a block?

A block groups multiple related tasks and can be combined with `rescue` and `always`.

### Q7. What is the difference between `rescue` and `always`?

```text
rescue → runs when a block task fails
always → runs after the block/rescue processing regardless of success/failure
```

### Q8. Is `block/rescue` the same as if/else?

No.

```text
when          → if/condition
block/rescue  → try/catch style error handling
always        → finally
```

### Q9. When should you use `shell`?

Use `shell` when you need shell-specific features such as pipes, redirection, or shell operators.

### Q10. When should you use `raw`?

Primarily for bootstrapping remote systems where the normal Ansible module environment, especially Python, is not yet available.

---

# Final Cheat Sheet

```text
Dictionary
    ↓
key-value pairs

List of Dictionaries
    ↓
multiple dictionaries in a list
    ↓
loop + item

Nested Dictionary
    ↓
dictionary inside dictionary
    ↓
employee.address.city

notify
    ↓
tells Ansible to trigger a handler

handler
    ↓
runs when notified

block
    ↓
groups related tasks

rescue
    ↓
handles block failure

always
    ↓
runs regardless of block success/failure

when
    ↓
condition / if

command
    ↓
normal command

shell
    ↓
shell-specific features

raw
    ↓
bootstrap / no Python environment
```

## Most Important Concept

```text
when       = IF
block      = TRY
rescue     = CATCH
always     = FINALLY
notify     = TRIGGER
handler    = ACTION
loop       = REPEAT
dictionary = GROUP RELATED DATA
```
