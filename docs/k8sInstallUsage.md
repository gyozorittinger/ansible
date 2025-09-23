#How to install K8S with playbooks

##Baseline

This playbook installs the required components like
- OS parameters
- Keepalived
- HaProxy
- Kubernetes repository
- Kurenetes binaries (kubeadm, kubectl, cri)
- Containerd

###Usage

```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v
```

###Tags

You can skip some steps with tags:

Don't install k9s client
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags k9s
```

Don't install Keepalived
```bash
ansible-playbook playbooks/k8s_ora9_baseline.yml -i inventory/k8s_init_aml11 -v --skip-tags keepalived
```

You can use multiple tags in a run.

##Install

This playbook install the K8S Cluster

###Usage

```bash
ansible-playbook playbooks/k8s_install.yml -i inventory/k8s_init_aml11
```
