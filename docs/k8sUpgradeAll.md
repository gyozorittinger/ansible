# How Upgrade K8S Cluster with playbooks


## Variables

vars/k8s_upgrade_all_vars.yml

Variables | Description
--- | --- |
k8s_version | The Kubernetes version after upgrade
kubernetes_repo_version | The Kubernetes repo version after upgrade 
calico_current_version | The current Calico version 
calico_new_version | The Calico version after upgrade 
pod_cidr_addr | Pod CIDR address* 

*To get the Pod CIDR address use this command on master node:
```bash
calicoctl get ippool
```

Examlaple var file:
```bash
k8s_version: 1.31.10
kubernetes_repo_version: 1.31
calico_current_version: 3.29.3
calico_new_version: 3.30.1
pod_cidr_addr: 172.16.0.0/16
```

### Inventory file

```bash
[masters]
master1 ansible_host=<master1 FQDN or IP>
master2 ansible_host=<master2 FQDN or IP>
master3 ansible_host=<master3 FQDN or IP>

[workers]
worker1 ansible_host=<worker1 FQDN or IP>
worker2 ansible_host=<worker2 FQDN or IP>

[kube_cluster:children]
masters
workers

```

You need to use the FQDN address or IP address through which you can access the server.


### Playbook
k8s_upgrade_all.yml

This playbook Upgrades the K8S cluster

### Usage

```bash
ansible-playbook playbooks/k8s_upgrade_all.yml -i inventory/<inventory file> -v
```

### Tags

Skip master upgrade, upgarde only master2, master3 and teh workers
```bash
ansible-playbook playbooks/k8s_upgrade_all.yml -i inventory/<inventory file> -v --skip-tags master_upgrade
```

Skip Calico upgrade
```bash
ansible-playbook playbooks/k8s_upgrade_all.yml -i inventory/<inventory file> -v --skip-tags calico
```

### Manual steps

The playbook stops at the following phases:

- After the planning steps. Check if manual steps are required.

- When you need drain a node. You have to drain the current node with following command

```bash
kubectl drain <node name> --delete-emptydir-data --ignore-daemonsets
```

- Before Calico upgrade. You need check the POD CIDR address in /tmp/calico.yaml file on master1 node.

