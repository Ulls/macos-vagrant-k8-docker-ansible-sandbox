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
