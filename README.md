# Ansible Infrastructure Automation

A collection of Ansible playbooks for deploying, configuring and maintaining Linux-based infrastructure.

## Available solutions

| Solution | Description |
|---|---|
| Oracle Linux | Operating system installation preparation, configuration and updates |
| Kubernetes | Cluster installation, configuration and upgrades |
| Percona PostgreSQL HA | Automated deployment of a highly available PostgreSQL cluster with integrated backup infrastructure |

## Getting started

Before running the playbooks, prepare the Ansible controller and install the required collections.

- [Ansible Installation Guide](docs/installAnsible.md)

## Kubernetes

Playbooks for installing and maintaining Kubernetes clusters on Oracle Linux.

- [Kubernetes Installation Guide](docs/k8sInstallUsage.md)
- [Kubernetes Upgrade Guide](docs/k8sUpgradeAll.md)

## Percona PostgreSQL High Availability

Automated deployment of a three-node Percona PostgreSQL 16 cluster with a dedicated backup server.

### Components

- Percona PostgreSQL 16 and contrib extensions
- Patroni
- etcd
- HAProxy
- Keepalived
- pgBackRest with TLS communication
- Automated credential and certificate generation
- Continuous WAL archiving

### Documentation

- [Installation, Reconfiguration and Removal Guide](docs/installPerconaHA.md)
- [Administration, Backup and Troubleshooting Guide](docs/adminPerconaHA.md)
- [Example Inventory](inventory/percona-ha.yaml)
- [Cluster Configuration](vars/percona_ha_vars.yml)
- [Backup Configuration](vars/percona_ha_pgbackrest_vars.yml)

### Supported platform

The new PostgreSQL HA installer has been developed and tested using Oracle Linux 10, Percona PostgreSQL 16 and Ansible Core 2.19.

### Legacy installation

The previous Percona installation playbooks remain available for reference.

- [Legacy Percona Installation Guide](docs/perconaInstall.md)

## Repository structure

| Directory | Purpose |
|---|---|
| `playbooks/` | Ansible playbooks |
| `roles/` | Reusable Ansible roles |
| `inventory/` | Example inventory files |
| `vars/` | Playbook configuration variables |
| `collections/` | Required Ansible collections |
| `docs/` | Installation and administration guides |

## Important notes

- Review and customize the inventory and configuration variables before running any playbook.
- Example inventories must not contain production credentials or confidential customer information.
- Some playbooks perform destructive operations. Always verify the target inventory before execution.
- Preserve generated credentials and certificates securely.
- Consult the solution-specific documentation for tested platforms, prerequisites and known limitations.
