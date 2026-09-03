## ADHOC COMMANDS:
-----------------
these are simple Linux commands.
ADHOC commands  are great for tasks you repeat daily
These are used for temp works.

-a = argument

```bash
ansible all -a "yum update -y"
ansible all -a "yum install git -y"
ansible all -a "git --version"
ansible all -a "yum install maven -y"
ansible all -a "mvn --version"
ansible all -a "touch file1"
ansible all -a "touch reyaz.txt"
ansible all -a "ls"
ansible all -a "yum install httpd -y"
ansible all -a "systemctl status httpd"
ansible all -a "systemctl start httpd"
ansible all -a "user add reyaz"
ansible all -a "cat /etc/passwd"
ansible all -a "yum remove git* maven* httpd* -y"
```
