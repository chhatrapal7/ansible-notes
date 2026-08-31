# Ansible Setup
Simply It is a IT automation tool
its a Configuration Management Tool.

### Launch Amazon Linux 2023 , t3.micro

```bash
sudo dnf install ansible -y

sudo dnf install python3 python3-pip -y

ansible --version
```

CREATE 4 SERVERS - Amazon Linux 2023 [1=ANSIBLE Master (Lounched), 2=PROD, 2=DEV] --> Master Alredy Lounch
run this command in all Server

```bash
sudo -i 
```
Now Root user from ansible server needs to login to all servers using root username and password
Do below commands in all servers using multi-exec (MobaXterm)

## first set the password for root
```bash
passwd root  
set new password:******
```
## enable all server to login as root

```bash
vi /etc/ssh/sshd_config (40 & 65 uncomment both lines)  [65: passwordauthentication yes  40: PermitR ootLogin yes]
systemctl restart sshd
systemctl status sshd
hostname -i  -- to see the private ip
```

## Inventory file

```bash
vi /etc/ansible/hosts
[prod]
172.31.20.40  --> pvt ips of prod1
172.31.21.25  --> pvt ips of prod2
[dev]
172.31.31.77  --> pvt ips of dev1
172.31.22.114 --> pvt ips of dev2
```

## Go to Ansible Master
--------------------
Lets Generate SSH Keys, using this KEY Ansible server will communicate with worker nodes

```bash
ssh-keygen

ssh-copy-id root@(private ip of prod-1)
ssh-copy-id root@(private ip of prod-2)
ssh-copy-id root@(private ip of dev-1)
ssh-copy-id root@(private ip of dev-2)

```
Now You Can Login Master --> Node
To check worker node connection with ansible server.
```bash
ansible -m ping all
```
