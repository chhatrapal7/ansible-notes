# Ansible Tags, Variables & Loops

## Table of Contents

1. [Ansible Tags](#1-ansible-tags)
2. [Single Tag](#2-single-tag)
3. [Multiple Tags](#3-multiple-tags)
4. [Skip Tags](#4-skip-tags)
5. [Always Tag](#5-always-tag)
6. [Never Tag](#6-never-tag)
7. [Always + Never Example](#7-always--never-example)
8. [Ansible Variables](#8-ansible-variables)
9. [Static Variables](#9-static-variables)
10. [Dynamic Variables](#10-dynamic-variables)
11. [Loops](#11-loops)
12. [Loop for Installing Packages](#12-loop-for-installing-packages)
13. [Loop for Uninstalling Packages](#13-loop-for-uninstalling-packages)
14. [Loop for Creating Users](#14-loop-for-creating-users)
15. [Loop for Removing Users](#15-loop-for-removing-users)
16. [Nested Loops](#16-nested-loops)
17. [Nested Loop with `product()`](#17-nested-loop-with-product)
18. [Quick Revision](#18-quick-revision)

---

# 1. Ansible Tags

**Tags** help us execute only selected tasks from an Ansible playbook.

Tags are used to:

* Execute specific tasks
* Skip specific tasks
* Organize large playbooks
* Run only the required part of a playbook

### Basic Syntax

```yaml
tasks:
  - name: Installing Git
    yum:
      name: git
      state: present
    tags:
      - git
```

---

# 2. Single Tag

Example playbook:

```yaml
---
- name: TAGS Playbook
  hosts: all
  become: yes

  tasks:
    - name: Installing Git
      yum:
        name: git
        state: present
      tags:
        - git

    - name: Installing Docker
      yum:
        name: docker
        state: present
      tags:
        - dockerinstall

    - name: Starting Docker
      service:
        name: docker
        state: started
      tags:
        - dockerstart

    - name: Create User
      user:
        name: suresh
        state: present
      tags:
        - user
```

To execute only the task with the `user` tag:

```bash
ansible-playbook reyaz.yml --tags user
```

Only the task tagged `user` will be executed.

---

# 3. Multiple Tags

We can execute multiple selected tags together.

```bash
ansible-playbook reyaz.yml --tags dockerinstall,dockerstart
```

This executes:

* `dockerinstall`
* `dockerstart`

---

# 4. Skip Tags

Tags can also be skipped during playbook execution.

## Skip Single Tag

```bash
ansible-playbook reyaz.yml --skip-tags "git"
```

The task with the `git` tag will be skipped.

## Skip Multiple Tags

```bash
ansible-playbook reyaz.yml --skip-tags "dockerinstall,dockerstart"
```

Both Docker installation and Docker start tasks will be skipped.

---

# 5. Always Tag

The `always` tag means that a task runs every time, even when specific tags are requested.

### Example

```yaml
---
- hosts: all
  gather_facts: false

  tasks:

    - name: Display Welcome Message
      debug:
        msg: "Welcome to DNA by Reyaz"
      tags:
        - always

    - name: Install Apache
      debug:
        msg: "Installing Apache"
      tags:
        - install

    - name: Install Nginx
      debug:
        msg: "Installing Nginx"
      tags:
        - nginx
```

Run:

```bash
ansible-playbook play.yml --tags install
```

### Result

The `install` task runs.

The `always` task also runs because it has the `always` tag.

The `nginx` task does not run because its tag was not requested.

### Remember

```text
always = Run this task regardless of selected tags
```

---

# 6. Never Tag

The `never` tag means that a task **does not run by default**.

It only runs when:

* The `never` tag is explicitly requested, or
* Another tag assigned to the same task is explicitly requested.

### Example

```yaml
---
- hosts: all
  gather_facts: false

  tasks:

    - name: Install Apache
      debug:
        msg: "Installing Apache"

    - name: Delete Everything
      debug:
        msg: "Deleting all files..."
      tags:
        - never
```

Run the playbook normally:

```bash
ansible-playbook play.yml
```

Output:

```text
Installing Apache
```

The `Delete Everything` task is skipped because it has the `never` tag.

### Explicitly Run `never`

```bash
ansible-playbook play.yml --tags never
```

Now the `Delete Everything` task runs.

### Real-Time Use

The `never` tag can be useful for potentially destructive or cleanup operations that should only run when explicitly requested.

For example:

```text
Cleanup
Delete files
Remove packages
Destroy infrastructure
```

---

# 7. Always + Never Example

We can use both `always` and `never` together with normal tags.

```yaml
---
- hosts: all
  gather_facts: false

  tasks:

    - name: Welcome
      debug:
        msg: "Welcome"
      tags:
        - always

    - name: Install Apache
      debug:
        msg: "Installing Apache"
      tags:
        - install

    - name: Delete Logs
      debug:
        msg: "Deleting Logs"
      tags:
        - never
        - cleanup
```

## Run Normally

```bash
ansible-playbook play.yml
```

The `Welcome` task runs.

The `Install Apache` task runs.

The `Delete Logs` task is skipped because it has `never`.

---

## Run Install Tag

```bash
ansible-playbook play.yml --tags install
```

The following tasks run:

```text
Welcome
Install Apache
```

`Welcome` runs because of `always`.

`Install Apache` runs because of `install`.

`Delete Logs` is skipped because of `never`.

---

## Run Cleanup Tag

```bash
ansible-playbook play.yml --tags cleanup
```

The following tasks run:

```text
Welcome
Delete Logs
```

The cleanup task runs because `cleanup` was explicitly requested.

---

# 8. Ansible Variables

Variables are used to store values that can be reused inside playbooks.

There are many types of variables in Ansible.

In this section:

1. Static Variables
2. Dynamic Variables

---

# 9. Static Variables

**Static variables** are declared inside the playbook.

They generally remain unchanged until we modify the playbook.

### Example

Create:

```bash
vi staticvar.yml
```

### `staticvar.yml`

```yaml
---
- name: STATIC Variable
  hosts: all
  become: yes

  vars:
    a: git
    b: maven

  tasks:

    - name: Installing Git
      yum:
        name: "{{ a }}"
        state: present

    - name: Installing Maven
      yum:
        name: "{{ b }}"
        state: present
```

Run:

```bash
ansible-playbook staticvar.yml
```

### How It Works

```yaml
vars:
  a: git
  b: maven
```

The variables are:

```text
a = git
b = maven
```

They are used inside tasks:

```yaml
name: "{{ a }}"
```

and:

```yaml
name: "{{ b }}"
```

---

## Uninstall Using Static Variables

Change:

```yaml
state: present
```

to:

```yaml
state: absent
```

For example:

```yaml
---
- name: STATIC Variable
  hosts: all
  become: yes

  vars:
    a: git
    b: maven

  tasks:

    - name: Uninstalling Git
      yum:
        name: "{{ a }}"
        state: absent

    - name: Uninstalling Maven
      yum:
        name: "{{ b }}"
        state: absent
```

Run:

```bash
ansible-playbook staticvar.yml
```

---

# 10. Dynamic Variables

**Dynamic variables** are provided from outside the playbook.

This is useful when the values need to change frequently without modifying the playbook itself.

### Example

Create:

```bash
vi dynamicvar.yml
```

### `dynamicvar.yml`

```yaml
---
- name: Dynamic Vars
  hosts: all
  become: yes

  tasks:

    - name: Installing Package A
      yum:
        name: "{{ a }}"
        state: present

    - name: Installing Package B
      yum:
        name: "{{ b }}"
        state: present
```

Run with external variables:

```bash
ansible-playbook dynamicvar.yml --extra-vars "a=git b=maven"
```

Here:

```text
a = git
b = maven
```

The playbook installs:

```text
git
maven
```

---

## Dynamic Variables — Another Example

The same playbook can be reused with different packages:

```bash
ansible-playbook dynamicvar.yml --extra-vars "a=tree b=httpd"
```

Now:

```text
a = tree
b = httpd
```

The playbook installs:

```text
tree
httpd
```

### Main Advantage

We don't need to edit the playbook every time.

```text
Playbook remains the same
        +
Different extra-vars
        ↓
Different result
```

---

# 11. Loops

Loops help reduce the number of lines of code.

Instead of writing separate tasks for every package/user, we can use one task and provide multiple items.

### Without Loop

```yaml
tasks:
  - name: Install Git
    yum:
      name: git
      state: present

  - name: Install Tree
    yum:
      name: tree
      state: present

  - name: Install Docker
    yum:
      name: docker
      state: present
```

This requires multiple tasks.

### With Loop

```yaml
tasks:
  - name: Install Packages
    yum:
      name: "{{ item }}"
      state: present
    loop:
      - git
      - tree
      - docker
```

Much less code.

---

# 12. Loop for Installing Packages

Create:

```bash
vi loop.yml
```

### `loop.yml`

```yaml
---
- name: Loops Playbook
  hosts: all
  become: yes
  gather_facts: false

  tasks:

    - name: Installing Packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - tree
        - docker
        - httpd
        - java-1.8.0-openjdk
        - maven
```

Run:

```bash
ansible-playbook loop.yml
```

The same task runs for every item.

### `item`

`item` represents the current value in the loop.

For example:

```text
item = git
item = tree
item = docker
item = httpd
item = java-1.8.0-openjdk
item = maven
```

---

## Verify Installed Packages

Check Maven:

```bash
ansible all -a "mvn -v"
```

Check Docker:

```bash
ansible all -a "docker --version"
```

---

# 13. Loop for Uninstalling Packages

Create:

```bash
vi loopuninstall.yml
```

### `loopuninstall.yml`

```yaml
---
- name: Uninstalling Packages
  hosts: all
  become: yes

  tasks:

    - name: Uninstalling Packages
      yum:
        name: "{{ item }}"
        state: absent
      loop:
        - git
        - tree
        - docker
        - httpd
        - java-1.8.0-openjdk
        - maven
```

Run:

```bash
ansible-playbook loopuninstall.yml
```

Verify:

```bash
ansible all -a "mvn -v"
ansible all -a "docker --version"
```

---

# 14. Loop for Creating Users

Loops are also useful for creating multiple users.

Create:

```bash
vi loopusers.yml
```

### `loopusers.yml`

```yaml
---
- name: Loops Playbook
  hosts: all
  become: yes

  tasks:

    - name: Creating Users
      user:
        name: "{{ item }}"
        state: present
      loop:
        - lucky
        - imthiaz
        - siva
        - rajesh
        - pavan
```

Run:

```bash
ansible-playbook loopusers.yml
```

Check users:

```bash
ansible all -a "cat /etc/passwd"
```

---

# 15. Loop for Removing Users

Create:

```bash
vi removeloopusers.yml
```

### `removeloopusers.yml`

```yaml
---
- name: Remove Users
  hosts: all
  become: yes

  tasks:

    - name: Removing Users
      user:
        name: "{{ item }}"
        state: absent
      loop:
        - lucky
        - imthiaz
        - siva
        - rajesh
        - pavan
```

Run:

```bash
ansible-playbook removeloopusers.yml
```

---

# 16. Nested Loops

A **nested loop** means working with combinations from two or more lists.

For example, suppose we have:

### Students

```yaml
students:
  - Reyaz
  - John
```

### Subjects

```yaml
subjects:
  - Linux
  - AWS
```

We want every student to be paired with every subject.

Expected combinations:

```text
Reyaz + Linux
Reyaz + AWS
John + Linux
John + AWS
```

---

# 17. Nested Loop with `product()`

Ansible/Jinja provides the `product()` filter to create all possible combinations between lists.

### Example

```yaml
---
- name: Nested Loops
  hosts: all
  gather_facts: false

  vars:

    students:
      - Reyaz
      - John

    subjects:
      - Linux
      - AWS

  tasks:

    - name: Student Subject Combination
      debug:
        msg: "{{ item.0 }} learns {{ item.1 }}"
      loop: "{{ students | product(subjects) | list }}"
```

---

## How `product()` Works

Given:

```yaml
students:
  - Reyaz
  - John

subjects:
  - Linux
  - AWS
```

The expression:

```yaml
students | product(subjects)
```

creates every possible combination.

Result:

```text
(Reyaz, Linux)
(Reyaz, AWS)
(John, Linux)
(John, AWS)
```

---

## Understanding `item.0` and `item.1`

Each combination contains two values.

```text
item.0 → Student
item.1 → Subject
```

Therefore:

```yaml
msg: "{{ item.0 }} learns {{ item.1 }}"
```

produces:

```text
Reyaz learns Linux
Reyaz learns AWS
John learns Linux
John learns AWS
```

---

# Why `| list` Is Used

The `product()` filter returns an iterator.

We use:

```yaml
| list
```

to convert the result into a list that Ansible can iterate over.

Therefore:

```yaml
students | product(subjects) | list
```

means:

```text
students
   ↓
product(subjects)
   ↓
all possible combinations
   ↓
convert to list
   ↓
loop through the list
```

---

# Simple Way to Remember `product()`

Think of:

```text
students × subjects
```

For example:

```text
2 students × 2 subjects
= 4 combinations
```

### Students

```text
Reyaz
John
```

### Subjects

```text
Linux
AWS
```

### Combinations

```text
Reyaz → Linux
Reyaz → AWS
John  → Linux
John  → AWS
```

---

# Nested Loop Output

```text
Reyaz learns Linux
Reyaz learns AWS
John learns Linux
John learns AWS
```

---

# 18. Quick Revision

## Tags

```text
--tags
```

Used to execute selected tasks.

```bash
ansible-playbook play.yml --tags user
```

---

## Multiple Tags

```bash
ansible-playbook play.yml --tags dockerinstall,dockerstart
```

---

## Skip Tags

```bash
ansible-playbook play.yml --skip-tags git
```

Multiple:

```bash
ansible-playbook play.yml --skip-tags dockerinstall,dockerstart
```

---

## Always

```text
always = Task runs even when specific tags are selected
```

---

## Never

```text
never = Task does not run by default
```

Run explicitly:

```bash
ansible-playbook play.yml --tags never
```

---

## Variables

### Static

Declared inside the playbook:

```yaml
vars:
  a: git
  b: maven
```

### Dynamic

Passed from the command line:

```bash
ansible-playbook playbook.yml --extra-vars "a=git b=maven"
```

---

## Loops

Loops reduce repetitive code.

```yaml
loop:
  - git
  - tree
  - docker
```

Use:

```yaml
name: "{{ item }}"
```

`item` represents the current loop value.

---

## Nested Loops

Use `product()` for combinations:

```yaml
loop: "{{ students | product(subjects) | list }}"
```

Access values using:

```text
item.0 → First list
item.1 → Second list
```

---

# Final Cheat Sheet

| Feature           | Purpose                   | Example                         |
| ----------------- | ------------------------- | ------------------------------- |
| `tags`            | Select tasks              | `--tags user`                   |
| `--skip-tags`     | Skip tasks                | `--skip-tags git`               |
| `always`          | Always run task           | `tags: [always]`                |
| `never`           | Don't run by default      | `tags: [never]`                 |
| Static variables  | Variables inside playbook | `vars:`                         |
| Dynamic variables | Variables from outside    | `--extra-vars`                  |
| `loop`            | Repeat task for items     | `loop: [...]`                   |
| `item`            | Current loop value        | `{{ item }}`                    |
| `product()`       | Create combinations       | `students \| product(subjects)` |
| `item.0`          | First value               | Student                         |
| `item.1`          | Second value              | Subject                         |

---

# Practice Commands

```bash
# Run specific tag
ansible-playbook play.yml --tags user

# Run multiple tags
ansible-playbook play.yml --tags dockerinstall,dockerstart

# Skip a tag
ansible-playbook play.yml --skip-tags git

# Run never-tagged task
ansible-playbook play.yml --tags never

# Static variables
ansible-playbook staticvar.yml

# Dynamic variables
ansible-playbook dynamicvar.yml --extra-vars "a=git b=maven"

# Install packages using loop
ansible-playbook loop.yml

# Uninstall packages using loop
ansible-playbook loopuninstall.yml

# Create users using loop
ansible-playbook loopusers.yml

# Remove users using loop
ansible-playbook removeloopusers.yml
```

---

# Topics Covered

```text
Ansible Tags
     ↓
Single Tag
     ↓
Multiple Tags
     ↓
Skip Tags
     ↓
Always Tag
     ↓
Never Tag
     ↓
Variables
     ↓
Static Variables
     ↓
Dynamic Variables
     ↓
Loops
     ↓
Package Loops
     ↓
User Loops
     ↓
Nested Loops
     ↓
product() Filter
```

## End of Notes
