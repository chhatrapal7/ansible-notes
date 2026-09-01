# Ansible Jinja2 Template – Nginx Practical

## 1. Create Template Directory

On the Ansible Control Node:

```bash
mkdir -p /root/templates
cd /root
```

Check:

```bash
ls
```

You should see:

```text
templates
```

---

## 2. Create Jinja2 Template

```bash
vi /root/templates/nginx.conf.j2
```

Paste:

```nginx
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

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

Save:

```text
ESC
:wq
ENTER
```

### Jinja2 Variables

The following are dynamic variables:

{{ nginx_port }}

{{ server_name }} 

{{ web_root }}

These values will come from the Ansible playbook.

For example:

```yaml
nginx_port: 80
server_name: mywebsite.com
web_root: /usr/share/nginx/html
```

After processing, the Managed Node will have:

```nginx
listen 80;
server_name mywebsite.com;
root /usr/share/nginx/html;
```

---

# 3. Create Ansible Playbook

Go to `/root`:

```bash
cd /root
vi nginx.yml
```

Paste:

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

    - name: Copy Nginx Config with Template
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

Save:

```text
ESC
:wq
ENTER
```

---

# 4. Check Directory Structure

Run:

```bash
cd /root
find . -maxdepth 2 -type f
```

Expected:

```text
./nginx.yml
./templates/nginx.conf.j2
```

Structure:

```text
/root
│
├── nginx.yml
│
└── templates
    └── nginx.conf.j2
```

---

# 5. Check Ansible Connectivity

Before running the playbook:

```bash
ansible all -m ping
```

Expected:

```text
SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you get `pong`, the Control Node can connect to the Managed Node.

---

# 6. Check Playbook Syntax

Run:

```bash
ansible-playbook nginx.yml --syntax-check
```

Expected:

```text
playbook: nginx.yml
```

---

# 7. Run the Playbook

```bash
ansible-playbook nginx.yml
```

Expected output will be similar to:

```text
PLAY [Deploy Nginx Config using Jinja2]

TASK [Gathering Facts]
ok

TASK [Install Nginx]
changed

TASK [Copy Nginx Config with Template]
changed

RUNNING HANDLER [Restart Nginx]
changed

PLAY RECAP
```

The important part is:

```yaml
template:
  src: templates/nginx.conf.j2
  dest: /etc/nginx/nginx.conf
```

This takes the Jinja2 template from the Control Node and generates the final Nginx configuration on the Managed Node.

---

# 8. Verify Nginx Configuration

SSH into the Managed Node:

```bash
ssh ec2-user@PRIVATE_IP
```

Check the generated configuration:

```bash
sudo cat /etc/nginx/nginx.conf
```

You should NOT see:

```text
{{ nginx_port }}
{{ server_name }}
{{ web_root }}
```

Instead, you should see:

```nginx
server {
    listen 80;
    server_name mywebsite.com;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

This proves that Jinja2 variables were replaced with their actual values.

---

# 9. Check Nginx Service

Run:

```bash
sudo systemctl status nginx
```

Expected:

```text
Active: active (running)
```

Or:

```bash
sudo systemctl is-active nginx
```

Expected:

```text
active
```

---

# 10. Test Nginx Configuration

Run:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

---

# 11. Create a Test Web Page

On the Managed Node:

```bash
echo "Hello from Ansible Jinja2 Nginx" | sudo tee /usr/share/nginx/html/index.html
```

Check:

```bash
cat /usr/share/nginx/html/index.html
```

Expected:

```text
Hello from Ansible Jinja2 Nginx
```

---

# 12. Test Using curl

Run:

```bash
curl http://localhost
```

Expected:

```text
Hello from Ansible Jinja2 Nginx
```

---

# 13. Test from Browser

Open the Managed EC2 instance's Public IPv4 address:

```text
http://PUBLIC-IP
```

You should see:

```text
Hello from Ansible Jinja2 Nginx
```

Make sure the EC2 Security Group allows:

```text
HTTP
Port: 80
Source: 0.0.0.0/0
```

---

# 14. Understand Jinja2 Dynamically

Suppose your playbook has:

```yaml
vars:
  nginx_port: 80
  server_name: mywebsite.com
  web_root: /usr/share/nginx/html
```

The template contains:

```nginx
listen {{ nginx_port }};
server_name {{ server_name }};
root {{ web_root }};
```

After running:

```bash
ansible-playbook nginx.yml
```

Ansible generates:

```nginx
listen 80;
server_name mywebsite.com;
root /usr/share/nginx/html;
```

If you change the variables:

```yaml
vars:
  nginx_port: 8080
  server_name: test.com
  web_root: /var/www/html
```

and run:

```bash
ansible-playbook nginx.yml
```

the configuration becomes:

```nginx
listen 8080;
server_name test.com;
root /var/www/html;
```

This is the main purpose of **Jinja2 templating**: dynamically generating configuration files from variables.

---

# 15. Understand `notify` and `handler`

In the playbook:

```yaml
notify: Restart Nginx
```

means:

> If the Nginx configuration changes, trigger the `Restart Nginx` handler.

Handler:

```yaml
handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```

### First Run

If the configuration changes:

```text
template → changed
handler  → restart Nginx
```

### Second Run

If there is no change:

```text
template → ok
handler  → not executed
```

This is related to Ansible's **idempotency** concept.

---

# Complete Flow

```text
                 CONTROL NODE
                      |
                      |
                  nginx.yml
                      +
             nginx.conf.j2
                      |
                      | Ansible
                      ↓
                MANAGED NODE
                      |
                      ↓
           /etc/nginx/nginx.conf
                      |
                      ↓
                 nginx -t
                      |
                      ↓
               Restart Nginx
                      |
                      ↓
                   Port 80
                      |
                      ↓
                 Web Browser
                      |
                      ↓
       Hello from Ansible Jinja2 Nginx
```

## Main Concepts Learned

1. **Jinja2 Template** – Used to create dynamic configuration files.
2. **`{{ variable }}`** – Used to insert Ansible variable values.
3. **`template`**** module** – Deploys the Jinja2 template to the Managed Node.
4. **`notify`** – Triggers a handler when a task changes something.
5. **`handler`** – Performs actions such as restarting Nginx.
6. **Idempotency** – Ansible avoids unnecessary changes and restarts when the configuration is already correct.
