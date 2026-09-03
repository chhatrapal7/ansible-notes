Modules supports
-----------------

 
ADHOC commands  are great for tasks you repeat daily

ADHOC     : ansible all -a "yum install git -y"
          ansible all -a "yum remove git* -y"
-----

Ansible Modules are units of code that can control system resources or execute system commands


States:
-------
yum modules (States for Package Management)
-----------
Present = install
absent = uninstall
latest = install or upgrades to latest version

Services module (States for Service Management)
---------------
started = start
stopped = stop
restarted = restart the service
enabled = Ensures the service starts on boot.

MODULE    : ansible all -m yum -a "name=git state=present"  --> -m is module , name = packagename, , state = present(install)
------

install packages with modules
--------------------------------
ansible all -m yum -a "name=git state=present"

ansible all -a "git -v"

ansible all -m yum -a "name=maven state=present"

To install Apache service
------------------------

ansible all -m yum -a "name=httpd state=present"

TO see the Version of Apache
----------------------------

ansible all -a "httpd -v"

TO start Apache
----------------

ansible all -m service -a "name=httpd state=started"  

---> this is use to start the service {we changed module yum to service, because yum to install, service to start}

TO see the status of Apache
------------------------

ansible all -a "systemctl status httpd" --> to see the status of httpd in all servers

To Stop Apache service
---------------------

ansible all -m service -a "name=httpd state=stopped" --> to stop the service

Update Apache to latest version
------------------------------

ansible all -m yum -a "name=httpd state=latest"   --> to update to latest version


AutoEnable Apache
------------------
ansible all -m service -a "name=httpd enabled=true"

uninstall apache:
---------------

ansible all -m yum -a "name=httpd state=absent"  ---> to uninstall httpd, absent = uninstall



==================================

TO create users in servers
--------------------

ansible all -m user -a "name=reyaz state=present"  --> present = to create

ansible all -a "cat /etc/passwd"

create another user

==============================

To create a file
--------------

touch file1

ansible all -m copy -a "src=file1 dest=/home/ec2-user/"

ansible all -a "ls /home/ec2-user"

====================

Remove all packages
----------------

ansible all -a "yum remove  maven* git* httpd* -y"

==============================================

yum , service, user etc all these are modules


PLAYBOOK:
its a collection of modules.
we can execute multiple modules inside playbooks.
playbook is written on YAML langauge.
YAML=YET ANOTHER MARKUP LANGUAGE
Extension: .yml or .yaml
its a human redable and seraliazable language.
it consits key-value pairs (dictionary).
space blw key and value must.
playbook start with --- and end with ... (opt)


PLAYBOOK-1:

# for comments, no option for multiline comment, only single line with #

- meaning description


===========================================
Test all Hosts Connectivity: ping module
===========================================


vi pb1.yml

---
- name: First Playbook
  hosts: all
 
  tasks:
    - name: Test connectivity
      ping:

ansible-playbook --syntax-check pb1.yml

ansible-playbook pb1.yml



===========================================
Installing APACHE and print Message
===========================================

vi pb2.yml

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
        msg: Apache Installed


ansible-playbook pb2.yml

===========================================
Installing and start Apache, yum and service module
 ===========================================


vi pb2.yml

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
      debug: msg="Apache Installed"

    - name: Start Apache
      service:
        name: httpd
        state: started


ansible-playbook pb2.yml


===========================================
Installing GIT and Docker and starting docker
===========================================

vi pb3.yml

---
- name: Install GIT and DOCKER Playbook
  hosts: all
  tasks:
    - name: Installing GIT
      yum: name=git state=present

    - name: Installing Docker
      yum: name=docker state=present

    - name: Starting Docker Service
      service: name=docker state=started

ansible-playbook pb3.yml

==================================
Creating User and Copy files playbook
====================================

vi pb4.yml
---
- name: Create users and copy files
  hosts: all
  tasks:
    - name: Creating User
      user: name=ramesh state=present

    - name: Copy files
      copy: src=file.txt dest=/home/ec2-user/


ansible-playbook --syntax-check pb4.yml  --> this will show you syntax errors

you can correct yourself syntax or use online tool called yamllint.com to see where are the errors

ansible-playbook pb3.yml

COLOR CODES
-----------
Yellow = successfully executed
Red = Failed
Green = Already executed
Blue = Skipped


Gather facts : First it will get all information about all worker nodes , default task performed by ansible

OK = total number of tasks
changed = no of tasks successfully executed


--> again run the same playbook --> now ok =6 , changed =0

---> now all are installed on workernodes , now try to uninstall , to uninstall , state=absent

--> use sed command to replace present to absent

=============================================
Uninstall docker and GIT playbook
=============================================

sed -i 's/present/absent/g' pb3.yml

vi pb3.yml

---
- name: Uninstall GIT and DOCKER Playbook
  hosts: all
  tasks:
    - name: Uninstalling GIT
      yum: name=git state=absent

    - name: Uninstalling Docker
      yum: name=docker state=absent

    - name: Starting Docker Service
      service: name=docker state=started


ansible-playbook pb3.yml

---> now you see some error for docker , because dokcer package is removed do that it is not starting

--> we can ignore these error by placing ignore_errors: true in playbook

============================
Ignore ERROR messages
=============================

---
- name: Uninstall GIT and DOCKER Playbook
  hosts: all
  ignore_errors: true
  tasks:
    - name: Uninstalling GIT
      yum: name=git state=present

    - name: Uninstalling Docker
      yum: name=docker state=present

    - name: Starting Docker Service
      service: name=docker state=started



================================
Install NGINX using Playbook
=================================

vi nginx.yml

---
- name: Install Nginx and Start it
  hosts: all
  become: yes # Run as root
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


ansible-playbook nginx.yml


================================
Deploy application on NGINX using Playbook
=================================

First uninstall httpd otherwise nginx will not start because remote machines already port 80 allocated for httpd

ansible all -a "yum remove httpd* -y"


vi nginx.yml

---
- name: Install Nginx and Start it
  hosts: all
  become: yes # Run as root
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



==============================
Setup module
=============================
 
ansible all -m setup

this command will give you full information about all workernodes, the above command shows a lot of info, but if you want to get only particular data grep

ansible all -m setup | grep -i cpu
ansible all -m setup | grep -i mem

cpu         : ansible_processor_vcpus
mem        : ansible_memtotal_mb
host        : ansible_nodename
flavour        : ansible_os_family
pkg manager    : ansible_pkg_mgr
blk        : block_available
kernel        : BOOT_IMAGE

DEBUG Module - to print values
---------  

debug module: it will print messages

---
- name: Printing messages
  hosts: all
  tasks:
    - name: printing a msg
      debug:
        msg: hai all my name is Reyaz



---
- name: Printing messages
  hosts: all
  tasks:
    - name: printging a msg
      debug:
        msg: "server name is: {{ansible_nodename}}, number of cpus: {{ansible_processor_vcpus}}, total memsize: {{ansible_memtotal_mb}}, the flavour is: {{ansible_os_family}}, package manager is: {{ansible_pkg_mgr}}"
