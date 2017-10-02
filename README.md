# DevOps Sandbox Automation on MacOS with Vagrant, Docker, Kubernetes and Ansible

> An Ansible playbook for adding and removing a Kubernetes node to a local virtual cluster

## Overview
The purpose of this repository is to maintain the automation of adding and removing a Kubernetes node to an existing Kubernetes cluster.  This facilitates the ability to simply scale up and down your virtual hardware that supports a Kubernetes test environment.  Instructions are provided in this README file to manually setup your environment at which point you'll be able to manage it using the included Ansible Playbooks.

``` sh
.
└── vagrant_kubernetes_node
    ├── roles (not used currently)
    ├── vagrant_kubernetes_node_vars
```

## Walkthrough

### Kubernetes Manual Install with Vagrant
Using Vagrant, we’ll setup two CentOS7 virtual machines, centos7-01 and centos7-02.  Centos7-01 will act as the master in our local Kubernetes (K8) virtual cluster.  All subsequent VMs will be provisioned to run as K8 minions (nodes).  Setup for these two VMs will be performed manually.  Later, we’ll install Ansible and provision additional K8 nodes in an automated manner.

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
        vars.yml
        vault.yml
    create_vagrant_kubernetes_node_template.txt
    create_vagrant_kubernetes_node.yml
    inventory
    remove_vagrant_kubernetes_node.yml
    requirements.yml
```

##### Walkthrough
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
git clone 
```

Use Ansible Roles from the Ansible Galaxy repository to take care of some of the dependencies we need to install for VM creation.  Those roles need to be installed as a prefatory matter to running the Ansible Playbook that will create our virtual K8 node.

###### requirements.yml
```
vi requirements.yml
```
Populate the requirements.yml with the following 

