# DevOps Sandbox Automation on MacOS with Vagrant, Docker, Kubernetes and Ansible

> This repository houses Ansible playbooks for adding and removing a Kubernetes node to a local virtual cluster running on MacOS.  This README file walks a user through the steps of setting up the Kubernetes cluster and running the playbooks.  There is also a deatiled explaination of the contents of the playbooks.

## Overview
The purpose of this repository is to maintain the automation of adding and removing a Kubernetes node to an existing Kubernetes cluster.  This facilitates the ability to simply scale up and down your virtual hardware that supports a Kubernetes test environment.  Instructions are provided in this README file to manually setup your environment at which point you'll be able to manage it using the included Ansible Playbooks.

### Kubernetes Manual Install with Vagrant
Using Vagrant, we’ll setup two CentOS7 virtual machines, centos7-01 and centos7-02.  Centos7-01 will act as the master in our local Kubernetes (K8) virtual cluster.  All subsequent VMs will be provisioned to run as K8 minions (nodes).  Setup for these two VMs will be performed manually.  Later, we’ll install Ansible and provision additional K8 nodes in an automated manner.

#### Walkthrough
First let’s install dependencies we’ll need to run the cluster on MacOS.

###### Install Brew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

###### Install Vagrant and Dependencies
```
brew cask install virtualbox
brew cask install vagrant
brew cask install vagrant-manager
brew cask install wget
```

###### Configure the CentOS VMs
```
cd ~/Documents
mkdir local && cd local
mkdir vagrant && cd vagrant
mkdir centos7-01 && cd centos7-01
vagrant init centos/7
```

Edit the Vagrantfile to allow insecure mode for downloading the box (cert issue at the host) and for the creation of a private IP address for the VM.  The Vagrantfile should look like this:
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_download_insecure = true
  config.vm.network "private_network", type: "dhcp"
end
```

###### Create the VM by running Vagrant's 'up' command
```
vagrant up
```

Repeat these configuration steps to create centos7-02.  It’s likely you want to refer to these instructions to create another minion VM in order to test Kubernetes replication and pod distribution across multiple minions, but for now, one minion will suffice.

###### SSH into your VMs and edit the host files so they can communicate.
```
cd ~/Documents/local/vagrant/centos7-01
vagrant ssh
sudo su
ip address show
```

Copy the IP address on the eth1.  Do this for both VMs.  Edit the three /etc/hosts files, the 2 VMs and the host machine so that you can reference these machines by name.  The files should add these two lines:
```
<IP of centos7-01>    centos-master
<IP of centos7-02>   centos-minion01
```

###### Install Kubernetes and Docker on the VMs.
On centos7-01 and centos7-02, run the following commands
```
sudo su
yum install -y ntp
systemctl enable ntpd && systemctl start ntpd
yum install -y kubernetes docker etcd
```

###### Configure the Kubernetes Master (Kubernetes config)
```
cd /etcd/kubernetes/
vi config
```
Alter the following configuration file lines
```
KUBE_MASTER="--master=http://centos-master:8080"
KUBE_ETCD_SERVERS=”--etcd-servers=http://centos-master:2379”
```

###### Configure the Kubernetes Master (Etcd config)
```
cd /etc/etcd
vi etcd.conf
```
Alter the following configuation file lines
```
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
```

###### Configure the Kubernetes Master (API server config)
```
cd /etc/kubernetes
vi apiserver
```
Alter the following configuation file lines
```
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
```
```
KUBE_API_ADDRESS="--address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBELET_PORT="--kubelet-port=10250"
#KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
```

###### Start the Kubernetes deamons
```
systemctl enable etcd kube-apiserver kube-controller-manager kube-scheduler
systemctl start etcd kube-apiserver kube-controller-manager kube-scheduler
```
You can check to see if everything is running OK by issuing the following command
```
systemctl status etcd kube-apiserver kube-controller-manager kube-scheduler | grep "(running)" | wc -l
```

###### Configure the Kubernetes Minion (Kubernetes config)
```
vi /etc/kubernetes/config
```
Alter the following configuation file lines
```
KUBE_MASTER="--master=http://centos-master:8080"
KUBE_ETCD_SERVERS="--etcd-servers=http://centos-master:2379"
```

###### Configure the Kubernetes Minion (Kubernetes kubelet)
```
vi /etc/kubernetes/kubelet
```
Alter the following configuation file lines
```
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname-override=centos-minion1"
KUBELET_API_SERVER="--api-servers=http://centos-master:8080"
#KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
```

###### Start the Kubernetes and Docker deamons
```
systemctl enable kube-proxy kubelet docker
systemctl start kube-proxy kubelet docker
```
You can check to see if everything is running OK by issuing the following commands
```
systemctl status kube-proxy kubelet docker  | grep "(running)" | wc -l
docker images
docker --version
docker pull hello-world
docker run hello-world
```

## Automating Kubernetes Node Creation with Ansible

### Overview
We can automate the manual steps of provisioning and deploying the Vagrant VM we created in the previous steps using Ansible.  The following instructions will walk you thought the creation of an Ansible project and two Ansible playbooks, one to create a Vagrant VM to be used as a Kubernetes node and another to remove that node from the cluster.

###### Project Layout

``` sh
.
└── vagrant_kubernetes_node
    ├── roles (not used currently)
    ├── vagrant_kubernetes_node_vars
        ├── vars.yml
        ├── vault.yml
    ├── create_vagrant_kubernetes_node_template.txt
    ├── create_vagrant_kubernetes_node.yml
    ├── inventory
    ├── remove_vagrant_kubernetes_node.yml
    ├── requirements.yml
```

#### Walkthrough
###### Install Ansible
```
sudo easy_install pip
sudo pip install ansible
```

###### Clone this repository
```
brew install git
git config --global user.name "Your Name"
git config --global user.email "youremail@yourprovider"
cd ~/Documents
mkdir local && cd local
git init
git clone https://<this repo>
cd sandbox-ansible
```

##### Documentation and How-To

###### requirements.yml
Use Ansible Roles from the Ansible Galaxy repository to take care of some of the dependencies we need to install for VM creation.  Those roles need to be installed as a prefatory matter to running the Ansible Playbook that will create our virtual K8 node.
File contents
```
# from galaxy
- src: geerlingguy.homebrew
```
- ![#f03c15](https://placehold.it/15/f03c15/000000?text=+) Install the requirements by running the following command
```
ansible-galaxy install -r requirements.yml
```

###### inventory
The inventory file identifies the physical or virtual machines that Ansible can manage.  The Ansibile playbooks in this project will manage this file, adding and removing VMs as needed.
File contents
```
localhost ansible_connection=local

[vagrant_k8_cluster]
centos-master   ansible_connection=ssh  ansible_user=root
centos-minion1  ansible_connection=ssh  ansible_user=root
```

###### vagrant_kubernetes_node_vars/vars.yml
The vars.yml file declares some variables the Playbook needs and assigns them some default values.  These values can be overidden on the command line when the playbook is run.  Specifically, the machine_number is overridden in the example below.
File contents
```
---
machine_number: 00
machine_path: "~/Documents/local/"
ansible_inventory_file: "~/Documents/local/sandbox-ansible/vagrant_kubernetes_node/inventory"
```

###### create_vagrant_kubernetes_node_vagrantfile_template.txt
The Playbook uses the contents of this file to populate the Vagrantfile which will create the VM.  File commands in the template indicates the VM should run Centos 7, sets up networking and copies the local ssh key to the VMs so that Ansible can access it.
File contents
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_download_insecure = true
  config.vm.network "private_network", type: "dhcp"
  config.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/me.pub"
  config.vm.provision 'shell', inline: "sudo cat /home/vagrant/.ssh/me.pub >> /home/vagrant/.ssh/authorized_keys", privileged: false
  config.vm.provision 'shell', inline: "sudo mkdir /root/.ssh"
  config.vm.provision 'shell', inline: "sudo cp /home/vagrant/.ssh/authorized_keys /root/.ssh/"
end
```

###### Communicating with the VMs via SSH
- ![#f03c15](https://placehold.it/15/f03c15/000000?text=+) If you haven't created ssh keys on your MacOS, do so by running the following commands
```
ssh-keygen -t rsa
```
In order to facilitate secure communication between Ansible and the VMs, the Playbook in this project will add the Vagrant configuration settings required to place this ssh key in the /.ssh/authorized_keys directory in the VMs being created.

###### create_vagrant_kubernetes_node.yml
What follows is a detailed explaination of the Ansible playbook that creates a Vagrant VM, configures it as a Kubernetes node and adds it to the existing Kubernetes cluster.  Ansible files are either in the yaml or json format.  Yaml is preferred, so that is what this project uses.

- ![#f03c15](https://placehold.it/15/f03c15/000000?text=+) Run the playbook by issuing the following command
```
ansible-playbook create_vagrant_kubernetes_node.yml --ask-sudo-pass --extra-vars "machine_number=04" -i ./inventory
```

Explaination of the Playbook file
```
---
- hosts: localhost
  connection: local
```
The three dashes declares the file as being in the Anisble format.
The ``hosts`` line defines the machine the Playbook command are targeting.  In this case, we're targeting our local machine

```
  pre_tasks:
    # Include variables from project directory
    - name: Include vars of create_vagrant_kubernetes_node_vars/vars.yml into the 'varcont' variable
      include_vars:
        file: vagrant_kubernetes_node_vars/vars.yml
    # Check to see if the machine directory exists...
    - name: Check if machine number is taken
      stat:
        path: "{{ machine_path }}centos7-{{ machine_number }}"
      register: machinecheck

    # Exit if the machine already exists
    - name: Check to see if VM already exists
      fail:
        msg: "The machine centos7-{{ machine_number }} exists."
      when: machinecheck.stat.exists
```
The pre_tasks section groups the setup commands.  The first command (denoted by the ``- name`` line) sets some local variables that we need to use in the Playbook.  Those variables are located in the /create_vagrant_kubernetes_node_vars/vars.yml file.
Next, the playbook checks the directory structure of your project to see if the machine has already been created.  The result of this check is stored in the ``machinecheck`` variable.
The final step in the pre_tasks group will exit the playbook if the machine exists.  If the machine does not exist, the playbook continues.

```
  vars:
    # the geerlingguy.homebrew role defaults homebrew_cask_apps to cask firefox, we don't want that so set that array var to empty []
    homebrew_cask_apps: []
  roles:
    -  geerlingguy.homebrew
```
This command runs the Ansible Galaxy Role we installed with the requirements.yml file.  The Role installs Homebrew if it is not installed.

```
    # Install Virtualbox
    - name: Check if Virtualbox is already installed
      stat:
        path: /usr/local/bin/Virtualbox
      register: virtualboxcheck
    - name: Install Virtualbox
      shell: brew cask install virtualbox
      when: not virtualboxcheck.stat.exists

    # Install Vagrant
    - name: Check if Vagrant is already installed
      stat:
        path: /usr/local/bin/vagrant
      register: vagrantcheck
    - name: Install Vagrant
      shell: brew cask install vagrant
      when: not vagrantcheck.stat.exists

    # Install Vagrant-Manager
    - name: Check if Vagrant-Manager is already installed
      stat:
        path: /usr/local/Caskroom/vagrant-manager
      register: vagrantmanagercheck
    - name: Install Vagrant
      shell: brew cask install vagrant-manager
      when: not vagrantmanagercheck.stat.exists

    # Install wget
    - name: Check if wget is already installed
      stat:
        path: /usr/local/bin/wget
      register: wgetcheck
    - name: Install wget
      shell: brew install wget
      when: not wgetcheck.stat.exists
```
These tasks install additional dependencies required to create the VMs.

```
    # Create directory for the machine's Vagrantfile
    - name: Create directory for new VM Vagrantfile
      file:
        path: "{{ machine_path }}centos7-{{ machine_number }}"
        state: directory
        mode: 0755
```
A directory is created to hold the Vagrantfile needed to create the VM.  This directory is also used as a check to see if the machine has already been provisioned.

```
    # Initialize the direcotry for Vagrant utilizations
    - name: Initialize the direcotry for Vagrant utilization
      shell: cd {{ machine_path }}centos7-{{ machine_number }} && vagrant init centos/7
```
Enter the directory just created and initialize it for vagrant.  The ``vagrant init centos/7`` command instructs vagrant that a Centos 7 VM will be created.

```
    # Update the contents of the Vagrantfile from the template
    - name: Update the contents of the Vagrantfile from the template
      copy: src="./create_vagrant_kubernetes_node_vagrantfile_template.txt" dest="{{ machine_path }}centos7-{{ machine_number }}/Vagrantfile"
```
This step creates the Vagrantfile based on the template in the project.

```
    # Create the VM
    - name: Create the Vagrant VM
      shell: cd {{ machine_path }}centos7-{{ machine_number }} && vagrant up
```
Create the VM by running the ``vagrant up`` command.

```
    - name: Update inventory file for ansible
      lineinfile:
        path: '{{ ansible_inventory_file }}'
        line: 'centos-minion{{machine_number}}  ansible_connection=ssh  ansible_user=root'
    # Refresh inventory to ensure new instaces exist in inventory
    - name: Refresh inventory to ensure new instaces exist in inventory
      meta: refresh_inventory
```
Add the newly created VM to the Ansible inventory file and refresh it so that the Playbooks steps to configure the VM that follow will succeed.

```
    # Shell script to get the IP of the new VM
    - name: Shell script to get the IP of the new VM
      shell: cd {{ machine_path }}centos7-{{ machine_number }} && echo $(vagrant ssh -c "ip address show eth1 | grep 'inet ' | sed -e 's/^.*inet //' -e 's/\/.*$//'" -- -q)
      register: new_ip_from_shell
    - set_fact:
        new_ip_from_shell: "{{ new_ip_from_shell.stdout }}"
    - debug: var=new_ip_from_shell
    - debug: msg="{{ new_ip_from_shell }}"
    # Update host /etc/host file
    - name: Update host files in the cluster so the machines can see each other
      lineinfile:
        path: '/etc/hosts'
        line: '{{ new_ip_from_shell }} centos-minion{{machine_number}}'
      become: true
      become_user: root
```
Find the IP address of the VM and place that information in a variable ``new_ip_from_shell`` using the ``set_fact`` command.  The debug command exists to check to assure the output is valid, which is an optional step.  The step the follows uses the ``new_ip_from_shell`` value to create a line in the /etc/hosts file so that the VM is visible to the local machine.

```
# Update each /etc/hosts file in the cluster
- hosts: vagrant_k8_cluster
  tasks:
    - name: Update host files in the cluster so the machines can see each other
      lineinfile:
        dest: /etc/hosts
        regexp: '.*{{ item }}$'
        line: '{{ hostvars[item].ansible_eth1.ipv4.address }} {{item}}'
        state: present
      with_items: '{{ hostvars.localhost.groups["vagrant_k8_cluster"] }}'
```
At this point, the playbook stops issuing commands to the local machine and begins issuing commands to all the virtual machines in the Kubernetes cluster.
The ``- hosts: vagrant_k8_cluster`` line instructs the playbook to apply the commands that follow to all the machines in our inventory file that are part of the vagrant_k8_cluster group.
The command that follows adds the new VM's IP and alias to the /etc/hosts file on each machine so they can communicate with each other.
The ``hostvars`` variable is automatically populated by Ansible when the ``hosts`` command is issued.  Ansible collects information about the machine or machines specified by the hosts command and stores that information in a system variable called ``hostvars``.  We can use that information to get the IP addresses of each VM.
The ``with_items`` command instructs Ansible to apply this command to each item in the vagrant_k8_cluster group.  This results in each /etc/hosts file containing the IP and aliases for each machine in the cluster.

```
# Configure the new VM as a Kubernetes Node
- hosts: centos-minion{{ machine_number }}
  tasks:
```
At this point, the playbook stops issuing commands to the entire cluster and begins issuing commands to the virtual machine that was just created.

```
    # Install ntpd
    - name: Install ntpd
      yum:
        name: ntp
        state: latest

    # Enable and start ntpd
    - name: Enable and start ntpd
      shell: systemctl enable ntpd && systemctl start ntpd

    # Install kubernetes
    - name: Install kubernetes
      yum:
        name: kubernetes
        state: latest

    # Install docker
    - name: Install docker
      yum:
        name: docker
        state: latest

    # Install etcd
    - name: Install etcd
      yum:
        name: etcd
        state: latest
```
Install dependencies requried to configure the VM as a Kubernetes node

```
    # Update the /etc/kubernetes/config file
    - name: Update the KUBE_MASTER configuration
      lineinfile:
        dest: /etc/kubernetes/config
        regexp: 'KUBE_MASTER'
        line: 'KUBE_MASTER="--master=http://centos-master:8080"'
        backrefs: yes

    - name: Add the KUBE_ETCD_SERVERS configuration
      lineinfile:
        dest: /etc/kubernetes/config
        regexp: 'KUBE_ETCD_SERVERS'
        line: 'KUBE_ETCD_SERVERS="--etcd-servers=http://centos-master:2379"'
        state: present
```
Use the ``lineinfile`` command to update the /etc/kubernetes/config file with the information it requires to communicate with the Kubernetes master.  
The command uses the ``regexp`` command to find the line it needs to replace.

```
    # Update the /etc/kubernetes/kubelet file
    - name: Update the KUBELET_ADDRESS configuration
      lineinfile:
        dest: /etc/kubernetes/kubelet
        regexp: 'KUBELET_ADDRESS'
        line: 'KUBELET_ADDRESS="--address=0.0.0.0"'
        backrefs: yes

    - name: Update the KUBELET_PORT configuration
      lineinfile:
        dest: /etc/kubernetes/kubelet
        regexp: 'KUBELET_PORT'
        line: 'KUBELET_PORT="--port=10250"'
        backrefs: yes

    - name: Update the KUBELET_HOSTNAME configuration
      lineinfile:
        dest: /etc/kubernetes/kubelet
        regexp: 'KUBELET_HOSTNAME'
        line: 'KUBELET_HOSTNAME="--hostname-override=centos-minion{{ machine_number }}"'
        backrefs: yes

    - name: Update the KUBELET_API_SERVER configuration
      lineinfile:
        dest: /etc/kubernetes/kubelet
        regexp: 'KUBELET_API_SERVER'
        line: 'KUBELET_API_SERVER="--api-servers=http://centos-master:8080"'
        backrefs: yes

    - name: Update the KUBELET_POD_INFRA_CONTAINER configuration
      lineinfile:
        dest: /etc/kubernetes/kubelet
        regexp: 'KUBELET_POD_INFRA_CONTAINER'
        line: '#KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"'
        backrefs: yes
```
Use the ``lineinfile`` command to update the /etc/kubernetes/kubelet file with the information it requires perform the duties of a Kubernetes node.  
The command uses the ``regexp`` command to find the line it needs to replace.

```
    # Enable kube-proxy kubelet and docker
    - name: Enable kube-proxy kubelet and docker
      shell: systemctl enable kube-proxy kubelet docker

    # Start kube-proxy kubelet and docker
    - name: Start kube-proxy kubelet and docker
      shell: systemctl start kube-proxy kubelet docker
```
Enable and start the kubernetes and docker daemons.  After these commands are issued, the VM will be an available member of the Kubernetes cluster.

### Automating the Removal of a Kubernetes node with Ansible
Just as we automated the creation of a Vagrant VM to be added to a Kubernetes cluster, we can automate the removal of the node and the destruction of the VM as well.  The Ansible playbook "remove_vagrant_kubernetes_node.yml" in this project handles this task.

###### remove_vagrant_kubernetes_node.yml
What follows is a detailed explaination of the Ansible playbook that removes Kubernetes node from the cluster and destroys the virtual machine.  The remove_vagrant_kubernetes_node playbook will remove the machine from the cluster, clean up the ~/.ssh/known_hosts and /etc/hosts files on the Ansible runner, delete the /etc/hosts entries on the cluster machines and finally destroy the VM.

- ![#f03c15](https://placehold.it/15/f03c15/000000?text=+) Run the playbook by issuing the following command
```
ansible-playbook remove_vagrant_kubernetes_node.yml --ask-sudo-pass --extra-vars "machine_number=04"
```

Explaination of the Playbook file
```
---
- hosts: localhost
  connection: local
```
The three dashes declares the file as being in the Anisble format.
The ``hosts`` line defines the machine the Playbook command are targeting.  In this case, we're targeting our local machine.

```
  pre_tasks:
    # Include variables from project directory
    - name: Include vars of create_vagrant_kubernetes_node_vars/vars.yml
      include_vars:
        file: vagrant_kubernetes_node_vars/vars.yml
    # Check to see if the machine directory exists...
    - name: Check if machine directory exists
      stat:
        path: "{{ machine_path }}centos7-{{ machine_number }}"
      register: machinecheck

    # Exit if the machine doesn't exist
    - name: Check to see if VM already exists
      fail:
        msg: "The machine centos7-{{ machine_number }} does not seem to exist"
      when: not machinecheck.stat.exists
```
The pre_tasks for the removal playbook is nearly identical to the pre_tasks for the creation playbook.  The only difference is that the last step will exit the playbook if the machine *doesn't* exist.

```
    # Destroy VM
    - name: Destroy VM
      shell: cd {{ machine_path }}centos7-{{ machine_number }} && vagrant destroy -f default

    # Remove directory structure for the VM
    - name: Remove directory structure for the VM
      shell: rm -fr {{ machine_path }}centos7-{{ machine_number }}
```
These steps run shell commands on the localhost to destroy the vitural machine and remove the directory we created to host the Vagrantfile to create the machine.

```
    # Remove line from inventory file
    - name: Remove inventory file for ansible
      lineinfile:
        path: '{{ ansible_inventory_file }}'
        state: absent
        line: 'centos-minion{{machine_number}}  ansible_connection=ssh  ansible_user=root'
```
Because the machine no longer exists, we can remove it from our Ansible inventory file.  The ``lineinfile`` command has a ``state`` value of ``absent`` which instructs the command to remove the ``line`` from the file if it exists.

```
# Remove line from /etc/hosts file
    - name: Remove line from /etc/hosts file
      lineinfile:
        path: '/etc/hosts'
        state: absent
        regexp: 'centos-minion{{machine_number}}'
      become: true
      become_user: root
```
Removes the destroyed virtual machine from the localhost's /etc/hosts file.  This time, lineinfile uses a regular espression ``regexp`` command to find the machine we created in the /etc/hosts file.  If found,  it's removed.  The ``become`` parameter instructs the command to take on a new identity, the ``become_user`` parameter defines that identity as root.  This playbook is run with the ``--ask-sudo-pass`` command line parameter so that the user enters their root password allowing this command to run successfully.

```
    # Remove the entry for the VM from ~/.ssh/known_hosts file
    - name: Remove line from known_hosts file
      lineinfile:
        path: '~/.ssh/known_hosts'
        state: absent
        regexp: 'centos-minion{{machine_number}}'
```
The destroyed VM's information is removed from the ~/.ssh/known_hosts file as well.

```
# Update the /etc/hosts files in the cluster to remove the VM
- hosts: vagrant_k8_cluster
  tasks:
    - name: Detete the record for the VM from the host files in the cluster
      lineinfile:
        path: '/etc/hosts'
        state: absent
        regexp: 'centos-minion{{machine_number}}'
```
At this point, the playbook stops issuing commands to the local machine and begins issuing commands to all the virtual machines in the Kubernetes cluster.
The ``- hosts: vagrant_k8_cluster`` line instructs the playbook to apply the commands that follow to all the machines in our inventory file that are part of the vagrant_k8_cluster group.
The command that follows removes the new VM's IP and alias from the /etc/hosts file on each machine in the cluster so that they no longer attempt to communicate with the removed node.
After this command is issued, the VM will be destroyed and completely removed from Kubernetes cluster.
