# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- ELK Configuration
  - Beats in Use
  - Machines Being Monitored
- How to Use the Ansible Build
- Access Policies

### Description of the Topology
This repository includes code defining the infrastructure below. 

![alt text](Diagrams/ELK-Project.png "ELK-Project Diagram")

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that work to process incoming traffic will be shared by both vulnerable web servers. Access controls will ensure that only authorized users — namely, ourselves — will be able to connect in the first place.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; ip packets; etc.

The configuration details of each machine may be found below.

| Name     |   Function  | IP Address | Operating System |
|----------|-------------|------------|------------------|
| Jump Box | Gateway     | 10.1.0.4   | Linux            |
| DVWA 1   | Web Server  | 10.1.0.5   | Linux            |
| DVWA 2   | Web Server  | 10.1.0.6   | Linux            |
| DVWA 3   | Web Server  | 10.1.0.7   | Linux            |
| ELK      | Monitoring  | 10.2.0.4   | Linux            |

In addition to the above, Azure has provisioned a **load balancer** in front of all webservers. The load balancer's targets are organized into the following availability zones:
- **Availability Zone 1**: DVWA 1 
- **Availability Zone 2**: DVWA 2
- **Availability Zone 2**: DVWA 3

## ELK Server Configuration
The ELK VM exposes an Elastic Stack instance. **Docker** is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task. This playbook is duplicated below.

To use this playbook, one must log into the Jump Box docker container by doing the following:

1. Start up container: `sudo docker start <"container-name">`
2. Log into it: `sudo docker attach <"container-name">`
 - You should see prompt change to something like: `root@54c1d8713328:~#`
3. From this prompt you can run: `ansible-playbook install_elk.yml elk`. This runs the `install_elk.yml` playbook on the `elk` host.

 - Hosts are added to /etc/ansible/hosts configuration file as follows:
   
```
[webservers]
10.1.0.5 ansible_python_interpreter=/usr/bin/python3
10.1.0.6 ansible_python_interpreter=/usr/bin/python3
10.1.0.7 ansible_python_interpreter=/usr/bin/python3
[elk]
10.2.0.4 ansible_python_interpreter=/usr/bin/python3
```

### Access Policies
The machines on the internal network are _not_ exposed to the public Internet. 

The **Jump Box** machine can accept connections from the Internet via IP address `20.70.15.57`

Machines _within_ the network can only be accessed by **each other**. The DVWA 1,  DVWA 2, and DVWA 3 VMs send log traffic to the ELK server.

A summary of the access policies in place can be found in the table below.

| Name     | Publicly Accessible | Allowed IP Addresses |
|----------|---------------------|----------------------|
| Jump Box | Yes                 | 64.72.118.76         |
| DVWA 1   | No                  | 10.1.0.0-254         |
| DVWA 2   | No                  | 10.1.0.0-254         |
| DVWA 3   | No                  | 10.1.0.0-254         |
| ELK      | No                  | 10.2.0.0-254         |

### Elk Configuration

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because...

- A new instance can be redeployed quickly if server were to fail. Also Ansible goes through a checklist to insure everything was provisioned correctly with no       errors. 

The playbook implements the following tasks:
- Install Docker
- Download python3-pip module and Docker python module
- Increase memory size so ELK will function properly
- Download and launch Docker ELK container
- Enable service docker on boot

The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.



  ![Alt text](Images\ELK-docker-ps.png "ELK docker ps command")



The playbook is duplicated below.

```yaml
---
# install_elk.yml
- name: Configure Elk VM with Docker
  hosts: elk
  remote_user:
  become: true
  tasks:
    # Use apt module
    - name: Install docker.io
      apt:
        update_cache: yes
        name: docker.io
        state: present

      # Use apt module
    - name: Install pip3
      apt:
        force_apt_get: yes
        name: python3-pip
        state: present

      # Use pip module
    - name: Install Docker python module
      pip:
        name: docker
        state: present
        
      # Use sysctl module
    - name: Use more memory
      sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: yes

      # Use docker_container module
    - name: download and launch a docker elk container
      docker_container:
        name: elk
        image: sebp/elk:761
        state: started
        restart_policy: always
        published_ports:
          - 5601:5601
          - 9200:9200
          - 5044:5044
      
      # Use systemd module    
    - name: Enable service docker on boot
      systemd:
        name: docker
        enabled: yes
```

### Target Machines & Beats
This ELK server is configured to monitor the DVWA 1, 2, and 3 VMs, at `10.1.0.5`, `10.1.0.6`, and `10.1.0.7` respectively.

We have installed the following Beats on these machines:
- Filebeat
- Metricbeat
- Packetbeat

These Beats allow us to collect the following information from each machine:
- **Filebeat**: Filebeat detects changes to the filesystem. Specifically, we use it to collect Apache logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU usage. We use it to detect SSH login attempts, failed `sudo` escalations, and CPU/RAM statistics.
- **Packetbeat**: Packetbeat collects packets that pass through the NIC, similar to Wireshark. We use it to generate a trace of all activity that takes place on the network, in case later forensic analysis should be warranted.

The playbook below installs Metricbeat on the target hosts. The playbook for installing Filebeat is not included, but looks essentially identical — simply replace `metricbeat` with `filebeat`, and it will work as expected.

```yaml
---
- name: Install metric beat
  hosts: webservers
  become: true
  tasks:
    # Use command module
  - name: Download metricbeat
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb

    # Use command module
  - name: install metricbeat
    command: dpkg -i metricbeat-7.4.0-amd64.deb

    # Use copy module
  - name: drop in metricbeat config
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml

    # Use command module
  - name: enable and configure docker module for metric beat
    command: metricbeat modules enable docker

    # Use command module
  - name: setup metric beat
    command: metricbeat setup

    # Use command module
  - name: start metric beat
    command: service metricbeat start
```

### Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. We use the **jump box** for this purpose.

To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
# Clone Repository + IaC Files
$ git clone https://github.com/johnny-atx/ELK-Project.git
# Move Playbooks and hosts file Into `/etc/ansible`
$ cp ELK-Project/Ansible/Playbooks/* .
$ cp ELK-Project/Ansible/files/* ./files
```

This copies the playbook files to the correct place.

Next, you must create a `hosts` file to specify which VMs to run each playbook on. Run the commands below:

```bash
$ cd /etc/ansible
$ cat > hosts <<EOF
[webservers]
10.1.0.5
10.1.0.6
10.1.0.7

[elk]
10.2.0.4
EOF
```

After this, the commands below run the playbook:

 ```bash
 $ cd /etc/ansible
 $ ansible-playbook install_elk.yml elk
 $ ansible-playbook filebeat-playbook.yml webservers
 $ ansible-playbook metricbeat-playbook.yml webservers
 ```

To verify success, wait five minutes to give ELK time to start up. 

Then, run: `curl http://10.2.0.4:5601`. This is the address of Kibana. If the installation succeeded, this command should print HTML to the console.


---

© 2020 Trilogy Education Services, a 2U, Inc. brand. All Rights Reserved.  
