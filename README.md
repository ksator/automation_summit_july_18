# About this project

Lab guide for Juniper automation summit  
July 2018 session - day 3 - Hands on Labs around Event Driven automation. 

# About the lab

## Building blocks 
- Junos devices
- Ubuntu VMs
- SaltStack
- Docker
- Gitlab
- RT (Request Tracker)

## Diagram

## Management IP addresses
| Name | Operating system | Management IP address  | Username | Password|
| ------------- | ------------- | ------------- |------------- | ------------- |
| minion1    | Ubuntu | 100.123.35.2    | jcluser | Juniper!1 |
| master1    | Ubuntu | 100.123.35.0    | jcluser | Juniper!1 |
| vMX-1    | Junos | 100.123.1.1    | jcluser | Juniper!1 |
| vMX-2    | Junos | 100.123.1.2    | jcluser | Juniper!1 |

# Use cases

##  Junos configuration automatic backup on Git

At each junos commit, SaltStack automatically collects the new junos configuration file and archives it to a git server: 
- When a Junos commit is completed, the Junos device send a syslog message ```UI_COMMIT_COMPLETED```.  
- The junos devices are configured to send this syslog message to SaltStack.  
- Each time SaltStack receives this syslog message, SaltStack automatically collects the new junos configuration file from the
JUNOS device that send this commit syslog message, and SaltStack automatically archives the new junos configuration file to a git server  

![continous_backup.png](resources/continous_backup.png)  

## Automated Junos show commands collection
Junos automation demo using SaltStack and Gitlab:
- Junos devices send syslog messages to SaltStack.
- Based on syslog messages received, SaltStack automatically collects junos "show commands" output from the JUNOS device that sent a syslog message, and SaltStack automatically archives the collected data on a Git server.

![automated_junos_show_commands_collection_with_syslog_saltstack.png](resources/automated_junos_show_commands_collection_with_syslog_saltstack.png)

## Automated tickets management
Junos automation demo using SaltStack and a ticketing system (Request Tracker):  
- Junos devices send syslog messages to SaltStack.  
- Based on syslog messages received from junos devices: 
  - SaltStack automatically creates a new RT (Request Tracker) ticket to track this issue. If there is already an existing ticket to track this issue, SaltStack updates the existing ticket instead of creating a new one. The syslog messages are added to the appropriate tickets.  
  - SaltStack automatically collects "show commands" output from junos devices and attach the devices output to the appropriate tickets. 

![RT.png](resources/RT.png)  

# Lab instructions

## Overview 
- Docker will be installed on the ubuntu host ```minion1```.
- Gitlab and Request Tracker will run on the ubuntu host ```minion1``` (containers).
- SaltStack master will be installed on the ubuntu host ```master1```.
- SaltStack minion will be installed on the ubuntu host ```minion1```.
- SaltStack Junos proxy will be installed on the  ubuntu host ```master1```.

## Ubuntu

Here's some system information from the Ubuntu hosts we will use: 
```
$ uname -a
Linux ubuntu 4.4.0-87-generic #110-Ubuntu SMP Tue Jul 18 12:55:35 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

## Docker

### Install Docker on the ubuntu host ```minion1```

Check if Docker is already installed on the ubuntu host ```minion1```
```
$ docker --version
```

If it was not already installed, install it:
```
$ sudo apt-get update
```
```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```
$ sudo apt-get update
```
```
$ sudo apt-get install docker-ce
```
```
$ sudo docker run hello-world
```
```
$ sudo groupadd docker
```
```
$ sudo usermod -aG docker $USER
```

Exit the ssh session to ```minion1``` and open an new ssh session to ```minion1``` and run these commands to verify you installed Docker properly:  
```
$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
```
$ docker --version
Docker version 18.03.1-ce, build 9ee9f40
```

## Request Tracker

There is a Request Tracker Docker image available https://hub.docker.com/r/netsandbox/request-tracker/  

### Pull a Request Tracker Docker image on the ubuntu host ```minion1```  

Check if you already have it locally:   
```
$ docker images
```

if not, pull the image:
```
$ docker pull netsandbox/request-tracker
```
Verify: 
```
$ docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
netsandbox/request-tracker   latest              b3843a7d4744        4 months ago        423MB
```

### Instanciate a Request Tracker container on the ubuntu host ```minion1``` 
```
$ docker run -d --rm --name rt -p 9081:80 netsandbox/request-tracker
```
Verify: 
```
$ docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                  PORTS                                                 NAMES
0945209bfe14        netsandbox/request-tracker   "/usr/sbin/apache2 -…"   26 hours ago        Up 26 hours             0.0.0.0:9081->80/tcp                                  rt
```

### Verify you can access to RT GUI

Access RT GUI with ```http://100.123.35.2:9081``` in a browser.  
The default ```root``` user password is ```password```

### Install the ```rt``` python library on the ubuntu host ```master1```

There are python libraries that provide an easy programming interface for dealing with RT:  
- [rtapi](https://github.com/Rickerd0613/rtapi) 
- [python-rtkit](https://github.com/z4r/python-rtkit)
- [rt](https://github.com/CZ-NIC/python-rt) 

Install the ```rt``` library on the ubuntu host ```master1```

```
$ sudo -s
```
```
# apt-get install python-pip
```
```
# pip install requests nose six rt
```
Verify
```
# pip list
```

### Verify you can use ```rt``` Python library on the ubuntu host ```master1```
Python interactive session on the ubuntu host ```master1```: 
```
# python
Python 2.7.12 (default, Dec  4 2017, 14:50:18)
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import rt
>>> tracker = rt.Rt('http://100.123.35.2:9081/REST/1.0/', 'root', 'password')
>>> tracker.url
'http://100.123.35.2:9081/REST/1.0/'
>>> tracker.login()
True
>>> tracker.search(Queue='General', Status='new')
[]
>>> tracker.create_ticket(Queue='General', Subject='abc', Text='bla bla bla')
1
>>> tracker.edit_ticket(1, Priority=3)
True
>>> tracker.reply(1, text='notes you want to add to the ticket 1')
True
>>> tracker.search(Queue='General')
[{u'Status': u'open', u'Priority': u'3', u'Resolved': u'Not set', u'TimeLeft': u'0', u'Creator': u'root', u'Started': u'Wed Jul 11 09:30:57 2018', u'Starts': u'Not set', u'Created': u'Wed Jul 11 09:30:10 2018', u'Due': u'Not set', u'LastUpdated': u'Wed Jul 11 09:30:57 2018', u'FinalPriority': u'0', u'Queue': u'General', 'Requestors': [u''], u'Owner': u'Nobody', u'Told': u'Not set', u'TimeEstimated': u'0', u'InitialPriority': u'0', u'id': u'ticket/1', u'TimeWorked': u'0', u'Subject': u'abc'}]
>>> for item in  tracker.search(Queue='General'):
...    print item['id']
...
ticket/1
>>> tracker.logout()
True
>>> exit()
```

## Gitlab

There is a Gitlab docker image available https://hub.docker.com/r/gitlab/gitlab-ce/  

### Pull a Gitlab Docker image on the ubuntu host ```minion1```

Check if you already have it locally: 
```
$ docker images
```

if not, pull the image:
```
$ docker pull gitlab/gitlab-ce
```
Verify: 
```
$ docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-ce             latest              504ada597edc        6 days ago          1.46GB
```

### Instanciate a Gitlab container on the ubuntu host ```minion1```

```
$ docker run -d --name gitlab -p 3022:22 -p 9080:80 gitlab/gitlab-ce
```
Verify: 
```
$ docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                  PORTS                                                 NAMES
eca5b63dcf99        gitlab/gitlab-ce             "/assets/wrapper"        26 hours ago        Up 26 hours (healthy)   443/tcp, 0.0.0.0:3022->22/tcp, 0.0.0.0:9080->80/tcp   gitlab
```

Wait for Gitlab container status to be ```healthy```.  
It takes about 5 mns.  
```
$ watch -n 10 'docker ps'
```

### Verify you can access to Gitlab GUI

Access Gitlab GUI with ```http://100.123.35.2:9080``` in a browser. 
Gitlab user is ```root```    
Create a password ```password```  
Sign in with ```root``` and ```password```  

### Create a group 
Name it ```automation_demo``` (Public)

### Create new projects
Create these new projects in the group ```automation_demo``` 
  - ```variables``` (Public, add Readme)
  - ```files_server``` (Public, add Readme)
  - ```configuration_backup``` (Public, add Readme)
  - ```show_commands_collected``` (Public, add Readme)

### Add your public key to Gitlab

Generate ssh keys on ubuntu host ```master1```
```
$ sudo -s
```
```
# ssh-keygen -t rsa -C "your.email@example.com" -b 4096
```
```
# ls /root/.ssh/
id_rsa  id_rsa.pub  known_hosts
```
Add the public key to Gitlab.  
On ubuntu host ```master1```, copy the public key:
```
# more /root/.ssh/id_rsa.pub
```
Access Gitlab GUI with ```http://100.123.35.2:9080``` in a browser, and add the public key to ```User Settings``` > ```SSH Keys```

### Update your ssh configuration on ubuntu host ```master1```
on ubuntu host ```master1```
```
$ sudo -s
```
```
# ls /root/.ssh/
config       id_rsa       id_rsa.pub   known_hosts
```
```
# more /root/.ssh/config
Host 100.123.35.2
Port 3022
Host *
Port 22
```

### Configure your Git client
on ubuntu host ```master1```
```
$ sudo -s
# git config --global user.email "you@example.com"
# git config --global user.name "Your Name"
```

### Verify you can use Git and Gitlab
on ubuntu host ```master1```
```
$ sudo -s
# git clone git@:100.123.35.2:automation_demo/variables.git
# git clone git@:100.123.35.2:automation_demo/files_server.git
# git clone git@:100.123.35.2:automation_demo/configuration_backup.git
# git clone git@:100.123.35.2:automation_demo/show_commands_collected.git
# ls
# cd variables
# git remote -v
# git branch 
# ls
# vi README.md
# git status
# git diff README.md
# git add README.md
# git status
# git commit -m 'first commit'
# git log --oneline
# git log
# git push origin master
```

## SaltStack 

Let's install: 
- SaltStack master on ubuntu host ```master1```.
- SaltStack minion on ubuntu host ```minion1```.
- SaltStack Junos proxy on ubuntu host ```master1```.

### Install SaltStack master on the ubuntu host ```master1``` 
Check if SaltStack master is already installed on the ubuntu host ```master1``` 
```
$ sudo -s
```
```
# salt --version
```
```
# salt-master --version
```
if SaltStack master was not already installed on the ubuntu host ```master1```, then install it: 
```
$ sudo -s
```
```
# wget -O - https://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2/SALTSTACK-GPG-KEY.pub | sudo apt-key add -
```
Add ```deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main``` in file ```/etc/apt/sources.list.d/saltstack.list```
```
# more /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main
```
```
# sudo apt-get update
```
```
# sudo apt-get install salt-master
```
Verify you installed properly SaltStack master on the ubuntu host ```master1```
```
# salt --version
salt 2018.3.2 (Oxygen)
```
```
# salt-master --version
salt-master 2018.3.2 (Oxygen)
```
### Configure SaltStack master

on the ubuntu host ```master1```, copy this [SaltStack master configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/master) in the file ```/etc/salt/master```

### start the salt-master

To see the Salt processes: 
```
# ps -ef | grep salt
```
To check the status: 
```
# systemctl status salt-master.service
```
You can use the ```service salt-master``` command with these options: 
```
# service salt-master
force-reload  restart       start         status        stop
```
To start it manually with a debug log level: 
```
# salt-master -l debug
```
if you prefer to run the salt-master as a daemon:
```
# salt-master -d
```

### SaltStack master log

```
# more /var/log/salt/master 
```
```
# tail -f /var/log/salt/master
```

### install SaltStack minion on ubuntu host ```minion1``` 
Check if SaltStack minion is already installed on the ubuntu host ```minion1```  
```
# salt-minion --version
```
```
# salt --version
```

if SaltStack minion was not already installed on the ubuntu host ```minion1```, then install it: 
```
$ sudo -s
```
```
# wget -O - https://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2/SALTSTACK-GPG-KEY.pub | sudo apt-key add -
```
Add ```deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main``` in file ```/etc/apt/sources.list.d/saltstack.list```
```
# more /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main
```
```
# sudo apt-get update
```
```
$ sudo apt-get install salt-minion
```
And verify if salt-minion was installed properly installation 
```
# salt-minion --version
salt-minion 2018.3.2 (Oxygen)
```
```
# salt --version
```

### configure SaltStack minion 

On the minion, copy this [minion configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/minion) in the file ```/etc/salt/minion```

### start the Salt-minion
On the minion:

To see the Salt processes: 
```
# ps -ef | grep salt
```
To check the status: 
```
# systemctl status salt-minion.service
```
You can use the ```service salt-minion``` command with these options: 
```
# service salt-minion
force-reload  restart       start         status        stop
```
To start it manually with a debug log level:
```
# salt-minion -l debug
```
if you prefer to run the salt-minion as a daemon:
```
# salt-minion -d
```
### Verify the keys on the master 
Once the minion is started,  list all public keys on the master. 
```
# salt-key -L
Accepted Keys:
minion1
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```

### verify master <-> minion communication 
on the master 
```
# salt minion1 test.ping
```
```
# salt "minion1" cmd.run "pwd"
```

### Install requirements for SaltStack Junos proxy

Run these commands on the host that will run a Junos proxy daemon.  
Let's do it on the master.  

```
# apt install python-pip
```
```
sudo python -m easy_install --upgrade pyOpenSSL
```
```
pip install junos-eznc jxmlease jsnapy
```

### junos-eznc test

Verify you can use junos-eznc on the master
```
# python
Python 2.7.12 (default, Dec  4 2017, 14:50:18)
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from jnpr.junos import Device
>>> dev=Device(host='100.123.1.1', user="jcluser", password="Juniper!1")
>>> dev.open()
Device(100.123.1.1)
>>> dev.facts['version']
'17.4R1-S2.2'
>>> dev.close()
>>> exit()
```
### Pillars configuration

Pillars are variables (for templates, sls files ...).    
They are defined in sls files, with a yaml data structure.  
There is a ```top``` file. ```top.sls``` file map minions to sls (pillars) files.  
Refer to the [master configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/master) to know the location for pillars.  
Copy [these files](https://github.com/ksator/automation_summit_july_18/tree/master/pillars) at the root of the repository ```variables``` (organization ```automation_demo```, Gitlab server ```100.123.35.2```)  

### Pillars configuration verification
```
$ sudo -s
```
```
# salt-run pillar.show_pillar
```
```
# salt-run pillar.show_pillar vMX-1
```




**root@ubuntu:~# more /etc/hosts
127.0.0.1       localhost
127.0.1.1       ubuntu
127.0.0.1       salt
**

### Start SaltStack proxies 

You need one salt proxy process per device.
to start the proxy for the device ```vMX-1``` with a debug log level, use this command:
```
# sudo salt-proxy -l debug --proxyid=vMX-1
```
if you prefer to run it as a daemon, use this command:
```
# sudo salt-proxy -d --proxyid=vMX-1
```
Start a proxy for the device vMX-2 as well
```
# sudo salt-proxy -d --proxyid=vMX-2
```
To see the SaltStack processes, run this command: 
```
# ps -ef | grep salt
```
### SaltStack keys 

By default, you need to accept the minions/proxies public public keys on the master.  

To list all public keys:
```
# salt-key -L
```
To accept a specified public key:
```
# salt-key -a vMX-1 -y
```
Or, to accept all pending keys:
```
# salt-key -A -y
```
We changed the [master configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/master) to auto accept the keys.  
So the keys are automatically accepted: 
```
# salt-key -L
Accepted Keys:
minion1
vMX-1
vMX-2
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```

### master <-> proxies communication verification

On the master: 

```
# salt 'vMX-1' test.ping
```
```
# salt 'vMX-2' test.ping
```

### Get the pillars for a minion/proxy

```
# salt 'vMX-1' pillar.ls
```
```
# salt 'vMX-1' pillar.items
```

### Junos execution modules documentation 

```
# salt 'vMX-1' junos -d
```
```
# salt 'vMX-1' junos.cli -d
```
### Junos execution modules examples 
```
# salt 'vMX-1' junos.cli "show version"
```
```
# salt 'vMX1' junos.rpc get-software-information
```

### grains

Grains are information collected from minions/proxies.  

Available grains can be listed by using the 'grains.ls' module:  
```
# salt 'vMX-1' grains.ls
```
Return all grains:
```
# salt 'vMX-1' grains.items
```
Return one or more grains:
```
# salt 'vMX-1' grains.item os_family zmqversion
```
### Junos facts and SaltStack grains

Displays the facts gathered during the connection:
```
# salt 'vMX-1' junos.facts
```
Junos facts are stored in proxy grains
```
# salt 'vMX-1' grains.item junos_facts
```

### flexible targeting system

```
# salt 'vMX-1' junos.cli "show version"
```
```
salt 'vMX*' junos.cli "show version"
```
```
salt -L 'vMX-1,vMX-2' junos.cli "show version"
```
```
salt -G 'junos_facts:model:vMX' junos.cli "show version"
```

### various output formats
```
salt 'vMX-1' junos.rpc get-software-information --output=yaml
```
```
salt 'vMX-1' junos.cli "show version" --output=json
```


### Install the junos syslog engine dependencies

```
# pip install pyparsing, twisted
```

### SaltStack files server

Salt runs a file server to deliver files to minions and proxies.  
The [master configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/master) indicates the location for the file servers.  
We are using an external files servers (repository ```files_server``` in the organization ```automation_demo``` of the Gitlab server ```100.123.35.2```).  

### Junos configuration templates 

Copy these [Junos templates](https://github.com/ksator/automation_summit_july_18/tree/master/junos) at the root of the repository ```files_server``` (organization ```automation_demo``` of the Gitlab server ```100.123.35.2```).  


### SaltStack state files

Copy these [states files](https://github.com/ksator/automation_summit_july_18/tree/master/states) at the root of the repository ```files_server``` (organization ```automation_demo``` of the Gitlab srever ```100.123.35.2```).  

### Junos state file demo

To execute [the syslog.sls state file](https://github.com/ksator/automation_summit_july_18/blob/master/states/syslog.sls), run this command on the master: 
```
# salt vMX-1 state.apply collect_show_commands_example
```

Run this command on the master to know the name of the host that runs the Junos proxy daemon for the device ```vMX1```:
```
# salt vMX-1 grains.item nodename
```

On that host, run these commands:
```
# ls /tmp/
```
```
# more /tmp/show_chassis_hardware.txt
```
```
# more /tmp/show_version.txt
```

### Configure syslog on Junos devices 

The state file [syslog.sls](https://github.com/ksator/automation_summit_july_18/blob/master/states/syslog.sls) uses the junos template [syslog.conf](https://github.com/ksator/automation_summit_july_18/blob/master/junos/syslog.conf).  

You already copied both the state file [syslog.sls](https://github.com/ksator/automation_summit_july_18/blob/master/states/syslog.sls) and the junos template [syslog.conf](https://github.com/ksator/automation_summit_july_18/blob/master/junos/syslog.conf) at the root of the SaltStack file servers (repository ```files_server``` in the organization ```automation_demo``` of the Gitlab srever ```100.123.35.2```).  

To execute the state file [syslog.sls](https://github.com/ksator/automation_summit_july_18/blob/master/states/syslog.sls), run this command on the master:  
```
# salt ''vMX-1' pillar.item syslog_host
```
```
# salt 'vMX*' state.apply syslog
```
```
# salt vMX-1 junos.cli "show system commit"
```
```
# salt vMX1 junos.cli "show configuration | compare rollback 1"
```
```
# salt vMX1 junos.cli "show configuration system syslog host 100.123.35.0"
```

### Runners

The runner directory is indicated in the [master configuration file](https://github.com/ksator/automation_summit_july_18/blob/master/master)  

On the master, add the file [request_tracker.py](https://github.com/ksator/automation_summit_july_18/blob/master/runners/request_tracker.py) to the directory ```/srv/runners/```

### Configure SaltStack for automated tickets management

```
# more /etc/salt/master.d/reactor.conf
reactor:
   - 'jnpr/syslog/*/SNMP_TRAP_LINK_*':
       - /srv/reactor/show_commands_output_collection_and_attachment_to_RT.sls
```
```
# more /srv/reactor/show_commands_output_collection_and_attachment_to_RT.sls
{% if data['data'] is defined %}
{% set d = data['data'] %}
{% else %}
{% set d = data %}
{% endif %}
{% set interface = d['message'].split(' ')[-1] %}
{% set interface = interface.split('.')[0] %}

create_a_new_ticket_or_update_the_existing_one:
  runner.request_tracker_saltstack_runner.create_ticket:
    - args:
        subject: "device {{ d['hostname'] }} had its interface {{ interface }} status that changed"
        text: " {{ d['message'] }}"



show_commands_output_collection:
  local.state.apply:
    - tgt: "{{ d['hostname'] }}"
    - arg:
      - collect_data_locally


attach_files_to_a_ticket:
  runner.request_tracker_saltstack_runner.attach_files_to_ticket:
    - args:
        subject: "device {{ d['hostname'] }} had its interface {{ interface }} status that changed"
        device_directory: "{{ d['hostname'] }}"
    - require:
        - show_commands_output_collection
        - create_a_new_ticket_or_update_the_existing_one

```
```
# more /srv/salt/collect_data_locally.sls
{% set device_directory = grains['id'] %}

make sure the device directory is presents:
  file.directory:
    - name: /tmp/{{ device_directory }}

{% for item in pillar['data_collection'] %}

{{ item.command }}:
  junos.cli:
    - name: {{ item.command }}
    - dest: /tmp/{{ device_directory }}/{{ item.command }}.txt
    - format: text

{% endfor %}
```
```
# salt vMX-1 state.apply collect_data_locally
```
```
# ls /tmp/vMX-1/
show chassis hardware.txt  show interfaces.txt  show version.txt
```
```
# more /tmp/vMX-1/show\ chassis\ hardware.txt

Hardware inventory:
Item             Version  Part number  Serial number     Description
Chassis                                VM5AE25B176A      VMX
Midplane
Routing Engine 0                                         RE-VMX
CB 0                                                     VMX SCB
FPC 0                                                    Virtual FPC
  CPU            Rev. 1.0 RIOT-LITE    BUILTIN
  MIC 0                                                  Virtual
    PIC 0                 BUILTIN      BUILTIN           Virtual

```
```
# salt-run request_tracker_saltstack_runner.create_ticket subject='test subject' text='test text'
```
```
# salt-run request_tracker_saltstack_runner.change_ticket_status_to_resolved ticket_id=2
```

tcpdump -i eth0 port 516 -vv
salt-run state.event pretty=True


service salt-master force-reload


root@ubuntu:~# nano /srv/runners/request_tracker_saltstack_runner.py



service salt-master restart

# ps -ef | grep salt


```
mkdir /srv/reactor/
```

nano /srv/reactor/show_commands_output_collection_and_attachment_to_RT.sls


```
salt-run reactor.list
```

```
salt-proxy -d --proxyid=vMX-2
```
```
# more /srv/pillar/vMX-2-details.sls
proxy:
      proxytype: junos
      host: 100.123.1.2
      username: jcluser
      port: 830
      passwd: Juniper!1

```
```
root@ubuntu:~# salt -N vmxlab test.ping
vMX-2:
    True
vMX-1:
    True
root@ubuntu:~#
```

```
# more /etc/salt/master
runner_dirs:
  - /srv/runners
engines:
  - junos_syslog:
      port: 516
  - webhook:
      port: 5001
pillar_roots:
 base:
  - /srv/pillar
ext_pillar:
  - git:
    - master git@100.123.35.0:summit/network_parameters.git
fileserver_backend:
  - git
  - roots
gitfs_remotes:
  - ssh://git@100.123.35.0/summit/network_model.git
file_roots:
  base:
    - /srv/salt
auto_accept: True
nodegroups:
 vmxlab: 'L@vMX-1,vMX-2'
```
```
salt vMX-1 state.apply collect_data_and_archive_to_git
```
```
 more /srv/reactor/automate_show_commands.sls
{% if data['data'] is defined %}
{% set d = data['data'] %}
{% else %}
{% set d = data %}
{% endif %}

automate_show_commands:
  local.state.apply:
    - tgt: "{{ d['hostname'] }}"
    - arg:
       - collect_data_and_archive_to_git

root@ubuntu:~#
```
```
root@ubuntu:~# service salt-master restart
root@ubuntu:~# salt-run reactor.list
```


Once it is started you can list all public keys
```
# salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```
