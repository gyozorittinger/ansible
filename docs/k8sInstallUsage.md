# How to install K8S with playbooks

## Baseline

This playbook installs the required components like
- OS parameters
- Keepalived
- HaProxy
- Kubernetes repository
- Kurenetes binaries (kubeadm, kubectl, cri)
- Containerd

### Usage

```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v
```

### Tags

You can skip some steps with tags:

Don't install k9s client
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags k9s
```

Don't install Keepalived
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags keepalived
```

Don't reboot node after update
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags reboot
```

Don't configure masters (Adding new worker nodes)
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags masters
```

You can use multiple tags in a run.

## Install

### IMPORTANT

The masters have to master1, master2, master3 names in the inventory file!

This playbook install the K8S Cluster

### Calico prompt

At calico install step the playbook stops. You should check the POD_CIDR in /tmp/calico.yaml on the master1 node before continue the install. 

### Usage

```bash
ansible-playbook playbooks/k8s_install.yml -i inventory/k8s_init_aml11 -v
```
