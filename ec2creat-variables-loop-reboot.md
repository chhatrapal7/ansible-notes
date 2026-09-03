# Ansible: EC2 Instances, Variables, Loops & Reboot

> **Notes by Chhatrapal**
>
> GitHub-ready learning notes with corrected YAML syntax and safer examples.

---

# 1. Launch an EC2 Instance

Ansible can manage AWS EC2 instances through the `amazon.aws` collection.

For AWS resources, we can execute the module on the Ansible control node.

## Basic EC2 Playbook

```yaml
---
- name: Launch EC2 instance by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    region: ap-south-1
    ami: ami-0bc7aabcf58d1e02a
    instance_type: t3.micro
    key_name: MyKey
    security_group: sg-deb102b3
    subnet_id: subnet-a6c089ce

  tasks:
    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: Chhatrapal-Server
        region: "{{ region }}"
        image_id: "{{ ami }}"
        instance_type: "{{ instance_type }}"
        key_name: "{{ key_name }}"
        security_group: "{{ security_group }}"
        vpc_subnet_id: "{{ subnet_id }}"
        network:
          assign_public_ip: true
        wait: true
        state: running
```

Run:

```bash
ansible-playbook ec2.yml
```

> **Important:** The AMI ID, subnet ID, security group ID, and key-pair name are AWS-account/region specific. Replace the example values with values from your own AWS environment.

---

# 2. Using `register` with EC2

We can capture the result of the EC2 task using `register`.

```yaml
---
- name: Launch EC2 instance by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    region: ap-south-1
    ami: ami-0bc7aabcf58d1e02a
    instance_type: t3.micro
    key_name: MyKey
    security_group: sg-deb102b3
    subnet_id: subnet-a6c089ce

  tasks:
    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: Chhatrapal-Server
        region: "{{ region }}"
        image_id: "{{ ami }}"
        instance_type: "{{ instance_type }}"
        key_name: "{{ key_name }}"
        security_group: "{{ security_group }}"
        vpc_subnet_id: "{{ subnet_id }}"
        network:
          assign_public_ip: true
        wait: true
        state: running
      register: ec2

    - name: Display instance details
      ansible.builtin.debug:
        var: ec2.instances
```

The registered result can contain information such as:

- Instance ID
- Public IP
- Private IP
- Instance state
- Availability Zone
- Tags
- AMI information

Run:

```bash
ansible-playbook ec2.yml
```

---

# 3. EC2 Instance States

The `state` parameter controls the desired state.

| State | Meaning |
|---|---|
| `running` | Ensure the instance is running |
| `stopped` | Ensure the instance is stopped |
| `started` | Start an existing stopped instance |
| `restarted` | Restart an existing instance |
| `terminated` | Terminate matching instances |
| `absent` | Ensure matching instances do not exist |

### Terminate

For a lab environment, if you want to terminate the instance, change the desired state appropriately and verify the module's matching behavior before running destructive operations.

---

# 4. Launch Multiple EC2 Instances

One way to launch multiple instances with different names is to use a list and a loop.

```yaml
---
- name: Launch multiple EC2 instances by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    region: ap-south-1
    ami: ami-0bc7aabcf58d1e02a
    instance_type: t3.micro
    key_name: MyKey
    security_group: sg-deb102b3
    subnet_id: subnet-a6c089ce

    instance_names:
      - Web-Server
      - App-Server
      - DB-Server

  tasks:
    - name: Launch EC2 instances
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

    - name: Display instance details
      ansible.builtin.debug:
        var: ec2_output.results
```

Run:

```bash
ansible-playbook diffnames.yml
```

---

# 5. Different Names and Different Instance Types

A list of dictionaries can store different values for each instance.

```yaml
---
- name: Launch EC2 instances by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    region: ap-south-1
    ami: ami-0bc7aabcf58d1e02a
    key_name: MyKey
    security_group: sg-deb102b3
    subnet_id: subnet-a6c089ce

    instances:
      - name: Web-Server
        type: t3.micro

      - name: App-Server
        type: t3.small

      - name: DB-Server
        type: t3.medium

  tasks:
    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "{{ item.name }}"
        region: "{{ region }}"
        image_id: "{{ ami }}"
        instance_type: "{{ item.type }}"
        key_name: "{{ key_name }}"
        security_group: "{{ security_group }}"
        vpc_subnet_id: "{{ subnet_id }}"
        network:
          assign_public_ip: true
        state: running
        wait: true
      loop: "{{ instances }}"
      register: ec2_output

    - name: Display instance details
      ansible.builtin.debug:
        var: ec2_output.results
```

Here:

```text
item.name
```

gives the instance name.

And:

```text
item.type
```

gives the instance type.

---

# 6. Different Names, Types and Tags

We can add an environment value to each dictionary and use it as an EC2 tag.

```yaml
---
- name: Launch EC2 instances with tags by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    region: ap-south-1
    ami: ami-0bc7aabcf58d1e02a
    key_name: MyKey
    security_group: sg-deb102b3
    subnet_id: subnet-a6c089ce

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

  tasks:
    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "{{ item.name }}"
        region: "{{ region }}"
        image_id: "{{ ami }}"
        instance_type: "{{ item.type }}"
        key_name: "{{ key_name }}"
        security_group: "{{ security_group }}"
        vpc_subnet_id: "{{ subnet_id }}"
        network:
          assign_public_ip: true
        tags:
          Environment: "{{ item.env }}"
          Owner: Chhatrapal
        state: running
        wait: true
      loop: "{{ instances }}"
      register: ec2_output

    - name: Display instance details
      ansible.builtin.debug:
        var: ec2_output.results
```

---

# 7. Dynamic Values Using `vars.yml`

Instead of keeping all variables inside the main playbook, we can store them in a separate file.

## `vars.yml`

```yaml
region: ap-south-1
ami: ami-0bc7aabcf58d1e02a
instance_type: t3.micro
key_name: MyKey
security_group: sg-deb102b3
subnet_id: subnet-a6c089ce
```

## `multipleec2.yml`

```yaml
---
- name: Launch EC2 instance by Chhatrapal
  hosts: localhost
  connection: local
  gather_facts: false

  vars_files:
    - vars.yml

  tasks:
    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: Chhatrapal-Server
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
```

Run:

```bash
ansible-playbook multipleec2.yml
```

---

# 8. Reboot Module

The `reboot` module is used to reboot a managed machine and wait for it to become available again before continuing.

Example:

```yaml
---
- name: Reboot server by Chhatrapal
  hosts: prod
  become: true

  tasks:
    - name: Reboot server
      ansible.builtin.reboot:
        msg: "Server is rebooting. Please wait..."
```

The `reboot` module is useful when a configuration change requires a system restart.

---

# 9. Practice Tasks

Try these exercises yourself.

### Practice 1: Different Names, Types and Tags

Create three EC2 instances:

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
loop: "{{ instances }}"
```

and:

```yaml
name: "{{ item.name }}"
instance_type: "{{ item.type }}"
```

Create the EC2 tag:

```yaml
tags:
  Environment: "{{ item.env }}"
  Owner: Chhatrapal
```

---

# 10. Important GitHub Safety Notes

Do **not** commit real AWS credentials to GitHub.

Avoid putting these directly into your repository:

```text
AWS Access Key
AWS Secret Access Key
Private SSH Key
Passwords
API Tokens
Vault Password
```

Also remember that these values are environment-specific:

```text
AMI ID
Security Group ID
Subnet ID
Key Pair Name
```

Use your own AWS values when running the examples.

---

## Quick Revision

```text
amazon.aws collection
        |
        +-- S3
        |
        +-- EC2
        |
        +-- Other AWS resources

register
        |
        +-- stdout
        +-- stderr
        +-- rc
        +-- changed
        +-- failed

loop
        |
        +-- list
        +-- list of dictionaries

vars_files
        |
        +-- external variable file

reboot
        |
        +-- reboot managed server
        +-- wait for server to return
```
