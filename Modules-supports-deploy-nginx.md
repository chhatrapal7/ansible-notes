# Ansible Modules & Deploy index page in Nginx & Playbooks Notes

## 1. ADHOC Commands

**Adhoc commands** are useful for tasks that need to be performed repeatedly, without creating a playbook.

### Examples

```bash
ansible all -a "yum install git -y"
ansible all -a "yum remove git* -y"
```

---

# 2. Ansible Modules

Ansible **modules** are units of code that can control system resources or execute system commands.

Common modules:

* `yum`
* `service`
* `user`
* `copy`
* `debug`
* `setup`
* `ping`

---

# 3. Module States

## Yum Module — Package Management

| State     | Meaning                              |
| --------- | ------------------------------------ |
| `present` | Install package                      |
| `absent`  | Uninstall package                    |
| `latest`  | Install or upgrade to latest version |

### Example

```bash
ansible all -m yum -a "name=git state=present"
```

Here:

* `-m` = module
* `yum` = module name
* `name=git` = package name
* `state=present` = install package

---

## Service Module — Service Management

| State          | Meaning                             |
| -------------- | ----------------------------------- |
| `started`      | Start service                       |
| `stopped`      | Stop service                        |
| `restarted`    | Restart service                     |
| `enabled=true` | Start service automatically on boot |

---

# 4. Installing Packages Using Modules

### Install Git

```bash
ansible all -m yum -a "name=git state=present"
```

Check Git:

```bash
ansible all -a "git --version"
```

### Install Maven

```bash
ansible all -m yum -a "name=maven state=present"
```

---

# 5. Install and Manage Apache

## Install Apache

```bash
ansible all -m yum -a "name=httpd state=present"
```

## Check Apache Version

```bash
ansible all -a "httpd -v"
```

## Start Apache

```bash
ansible all -m service -a "name=httpd state=started"
```

> `yum` is used to install packages, while `service` is used to manage services.

## Check Apache Status

```bash
ansible all -a "systemctl status httpd"
```

## Stop Apache

```bash
ansible all -m service -a "name=httpd state=stopped"
```

## Update Apache to Latest Version

```bash
ansible all -m yum -a "name=httpd state=latest"
```

## Enable Apache on Boot

```bash
ansible all -m service -a "name=httpd enabled=true"
```

## Uninstall Apache

```bash
ansible all -m yum -a "name=httpd state=absent"
```

---

# 6. Creating Users

Create a user on all managed nodes:

```bash
ansible all -m user -a "name=reyaz state=present"
```

Check users:

```bash
ansible all -a "cat /etc/passwd"
```

Create another user:

```bash
ansible all -m user -a "name=ramesh state=present"
```

---

# 7. Copying Files

Create a file locally:

```bash
touch file1
```

Copy the file to managed nodes:

```bash
ansible all -m copy -a "src=file1 dest=/home/ec2-user/"
```

Check the file:

```bash
ansible all -a "ls /home/ec2-user"
```

---

# 8. Remove Packages

Remove Maven, Git and Apache:

```bash
ansible all -a "yum remove maven* git* httpd* -y"
```

---

# 9. Playbooks

A **playbook** is a collection of tasks/modules.

Playbooks allow us to execute multiple modules in an organized way.

### Important Points

* Playbooks are written in **YAML**.
* File extension can be `.yml` or `.yaml`.
* YAML is human-readable and serializable.
* YAML uses **key-value pairs**.
* Indentation/spaces are important in YAML.
* `#` is used for comments.
* YAML commonly starts with `---`.
* `...` can be used to mark the end of a YAML document.

---

# 10. Basic Playbook Structure

```yaml
---
- name: First Playbook
  hosts: all

  tasks:
    - name: Task Name
      module_name:
```

---

# 11. Playbook 1 — Test Host Connectivity

The `ping` module is used to test connectivity between the Ansible controller and managed nodes.

Create:

```bash
vi pb1.yml
```

### `pb1.yml`

```yaml
---
- name: First Playbook
  hosts: all

  tasks:
    - name: Test connectivity
      ping:
```

### Syntax Check

```bash
ansible-playbook --syntax-check pb1.yml
```

### Execute Playbook

```bash
ansible-playbook pb1.yml
```

---

# 12. Playbook 2 — Install Apache and Print Message

Create:

```bash
vi pb2.yml
```

### `pb2.yml`

```yaml
---
- name: Installing Apache
  hosts: all
  become: yes

  tasks:
    - name: Installing Apache
      yum:
        name: httpd
        state: present

    - name: Print Message
      debug:
        msg: "Apache Installed"
```

Run:

```bash
ansible-playbook pb2.yml
```

---

# 13. Install and Start Apache

```yaml
---
- name: Installing Apache
  hosts: all
  become: yes

  tasks:
    - name: Installing Apache
      yum:
        name: httpd
        state: present

    - name: Print Message
      debug:
        msg: "Apache Installed"

    - name: Start Apache
      service:
        name: httpd
        state: started
```

Run:

```bash
ansible-playbook pb2.yml
```

---

# 14. Playbook 3 — Install Git and Docker

Create:

```bash
vi pb3.yml
```

### `pb3.yml`

```yaml
---
- name: Install GIT and DOCKER Playbook
  hosts: all
  become: yes

  tasks:
    - name: Installing GIT
      yum:
        name: git
        state: present

    - name: Installing Docker
      yum:
        name: docker
        state: present

    - name: Starting Docker Service
      service:
        name: docker
        state: started
```

Run:

```bash
ansible-playbook pb3.yml
```

---

# 15. Creating User and Copying Files

Create:

```bash
vi pb4.yml
```

### `pb4.yml`

```yaml
---
- name: Create users and copy files
  hosts: all
  become: yes

  tasks:
    - name: Creating User
      user:
        name: ramesh
        state: present

    - name: Copy files
      copy:
        src: file.txt
        dest: /home/ec2-user/
```

### Syntax Check

```bash
ansible-playbook --syntax-check pb4.yml
```

### Execute

```bash
ansible-playbook pb4.yml
```

For YAML syntax checking, you can also use tools such as **yamllint**.

---

# 16. Ansible Output / Status

Common Ansible output indicators:

| Status        | Meaning                           |
| ------------- | --------------------------------- |
| `ok`          | Task completed without changes    |
| `changed`     | Task made a change                |
| `failed`      | Task failed                       |
| `skipped`     | Task was skipped                  |
| `unreachable` | Managed node could not be reached |

### Example

When a playbook is run for the first time:

```text
ok=1
changed=3
```

If you run the same playbook again and the system is already in the desired state, you may see more `ok` and fewer/no `changed` tasks.

This is an important feature of Ansible called **idempotency**.

---

# 17. Gather Facts

By default, Ansible gathers information about managed nodes before executing tasks.

This information is called **facts**.

The `setup` module is used to gather facts.

```bash
ansible all -m setup
```

This command returns a large amount of information.

You can search for specific information:

```bash
ansible all -m setup | grep -i cpu
ansible all -m setup | grep -i mem
```

### Common Ansible Facts

| Information     | Ansible Fact              |
| --------------- | ------------------------- |
| CPU count       | `ansible_processor_vcpus` |
| Total memory    | `ansible_memtotal_mb`     |
| Host name       | `ansible_nodename`        |
| OS family       | `ansible_os_family`       |
| Package manager | `ansible_pkg_mgr`         |
| Kernel          | `ansible_kernel`          |

---

# 18. Debug Module

The `debug` module is used to print messages and variable values.

## Simple Message

```yaml
---
- name: Printing messages
  hosts: all

  tasks:
    - name: Printing a message
      debug:
        msg: "Hi all, my name is Reyaz"
```

---

# 19. Debug Module with Ansible Facts

We can use Jinja2 expressions `{{ }}` to access variables.

```yaml
---
- name: Printing messages
  hosts: all

  tasks:
    - name: Printing server information
      debug:
        msg: "Server name is: {{ ansible_nodename }}, number of CPUs: {{ ansible_processor_vcpus }}, total memory: {{ ansible_memtotal_mb }} MB, OS family: {{ ansible_os_family }}, package manager: {{ ansible_pkg_mgr }}"
```

---

# 20. Uninstall Git and Docker

We can change:

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
- name: Uninstall GIT and DOCKER Playbook
  hosts: all
  become: yes

  tasks:
    - name: Uninstalling GIT
      yum:
        name: git
        state: absent

    - name: Uninstalling Docker
      yum:
        name: docker
        state: absent

    - name: Stop Docker Service
      service:
        name: docker
        state: stopped
```

> If the Docker package/service does not exist, the service task can fail. It is better to manage the package and service state consistently.

---

# 21. Ignore Errors

Ansible can continue execution even if a task fails by using:

```yaml
ignore_errors: true
```

Example:

```yaml
---
- name: Example Playbook
  hosts: all
  become: yes

  tasks:
    - name: Install Git
      yum:
        name: git
        state: present

    - name: Start Docker
      service:
        name: docker
        state: started
      ignore_errors: true
```

> Use `ignore_errors` carefully. It can hide real problems.

---

# 22. Install NGINX Using Playbook

Create:

```bash
vi nginx.yml
```

### `nginx.yml`

```yaml
---
- name: Install Nginx and Start it
  hosts: all
  become: yes

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

Run:

```bash
ansible-playbook nginx.yml
```

---

# 23. Deploy Web Application on NGINX

If Apache (`httpd`) is already using port 80, NGINX may not be able to bind to the same port.

Remove Apache if it is not required:

```bash
ansible all -a "yum remove httpd* -y"
```

Create an `index.html` file on the Ansible controller.

Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My NGINX Application</title>
</head>
<body>
    <h1>Hello from Ansible and NGINX!</h1>
</body>
</html>
```

### `nginx.yml`

```yaml
---
- name: Install Nginx and Deploy Web Page
  hosts: all
  become: yes

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy a web page
      copy:
        src: index.html
        dest: /usr/share/nginx/html/index.html

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Run:

```bash
ansible-playbook nginx.yml
```

---

# 24. Important Ansible Modules

| Module    | Purpose                        |
| --------- | ------------------------------ |
| `ping`    | Test connectivity              |
| `yum`     | Install/remove/update packages |
| `service` | Start/stop/restart services    |
| `user`    | Create/manage users            |
| `copy`    | Copy files                     |
| `debug`   | Print messages/variables       |
| `setup`   | Gather system facts            |

---

# 25. Important Ansible Commands

### Check Connectivity

```bash
ansible all -m ping
```

### Run Adhoc Command

```bash
ansible all -a "uptime"
```

### Check Syntax

```bash
ansible-playbook --syntax-check playbook.yml
```

### Run Playbook

```bash
ansible-playbook playbook.yml
```

### Gather Facts

```bash
ansible all -m setup
```

### Install Package

```bash
ansible all -m yum -a "name=git state=present"
```

### Remove Package

```bash
ansible all -m yum -a "name=git state=absent"
```

### Start Service

```bash
ansible all -m service -a "name=httpd state=started"
```

### Stop Service

```bash
ansible all -m service -a "name=httpd state=stopped"
```

---

# 26. Quick Revision

### Adhoc Command

```bash
ansible all -a "command"
```

### Module Syntax

```bash
ansible all -m MODULE -a "ARGUMENTS"
```

### Playbook

```bash
ansible-playbook playbook.yml
```

### Package States

```text
present  = Install
absent   = Uninstall
latest   = Latest version
```

### Service States

```text
started   = Start
stopped   = Stop
restarted = Restart
enabled   = Start automatically on boot
```

### Key Modules

```text
yum      → Package management
service  → Service management
user     → User management
copy     → Copy files
debug    → Print information
setup    → Gather facts
ping     → Connectivity testing
```

---

# 27. Ansible Flow

```text
Ansible Controller
        |
        | SSH
        v
+-------------------+
| Managed Node 1    |
| Managed Node 2    |
| Managed Node 3    |
+-------------------+

        |
        v

Adhoc Commands / Playbooks
        |
        v
     Modules
        |
        v
Desired State
```

---

# 28. Recommended Practice Order

Learn and practice Ansible in this order:

1. Ansible inventory
2. SSH connectivity
3. Adhoc commands
4. `ping` module
5. `yum` module
6. `service` module
7. `user` module
8. `copy` module
9. `debug` module
10. `setup` module
11. YAML syntax
12. Playbooks
13. Variables
14. Facts
15. Conditionals
16. Loops
17. Handlers
18. Templates
19. Roles
20. Ansible Vault

---

## End of Notes

**Topics covered:**
Adhoc Commands → Modules → States → Packages → Services → Users → Copy → Playbooks → YAML → Debug → Facts → NGINX Deployment → Error Handling
