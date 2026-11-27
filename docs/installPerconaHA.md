# Percona PostgreSQL HA — Installation Guide

This guide covers installation, reconfiguration and removal. For everyday operations, see [Administration Guide](adminPerconaHA.md).

## 1. Prerequisites

The tested platform is Oracle Linux 10, Percona PostgreSQL 16 and Ansible Core 2.19. Prepare three dedicated database nodes and one dedicated backup server. The Ansible controller may be an existing Linux administration server.

The installer deploys PostgreSQL and contrib, Patroni, etcd, HAProxy, Keepalived and pgBackRest with TLS. It is intended for new installations, not migrations or major version upgrades.

### Storage

Create persistent filesystems and configure `/etc/fstab` before installation. Storage provisioning is manual.

On each database node:

```bash
findmnt /database
df -h /database
```

On the backup server:

```bash
findmnt /backup
df -h /backup
```

Default data directory: `/database/pg16data`. Default backup repository: `/backup/pgbackrest`. Mount protection is enabled by default.

### Network requirements

| Protocol / port | Source | Destination | Purpose |
|---|---|---|---|
| TCP 22 | Controller | All servers | SSH |
| TCP 2379, 2380 | Database nodes | Database nodes | etcd |
| TCP 5432 | Database nodes | Database nodes | PostgreSQL |
| TCP 8008 | Database nodes | Database nodes | Patroni REST API |
| TCP 5000, 5001 | Applications | Cluster VIP | Read-write / read-only |
| TCP 8432 | Database and backup nodes | Database and backup nodes | pgBackRest TLS |
| IP protocol 112 | Database nodes | Database nodes | VRRP |

HAProxy statistics listen locally on TCP 7000. The network must support the floating VIP. Package repositories must be accessible during installation.

The default installation disables SELinux and does not configure firewall rules. etcd and Patroni REST API use unauthenticated HTTP; restrict them to trusted networks. PostgreSQL application TLS is not configured by this installer.

### SSH access

On the controller, configure root SSH key authentication. Replace the example hostnames:

```bash
ssh-copy-id root@example-pgsql-node-01.example.com
ssh-copy-id root@example-pgsql-node-02.example.com
ssh-copy-id root@example-pgsql-node-03.example.com
ssh-copy-id root@example-pgsql-backup-01.example.com
```

## 2. Prepare the controller

Follow [Ansible installation guide](installAnsible.md). The commands below assume the Ansible 2.19 aliases described there.

```bash
ansible219 --version
git clone https://github.com/gyozorittinger/ansible.git
cd ansible
ansible-galaxy219 collection install -r collections/requirements.yml
```

Run all Ansible commands from the repository root.

## 3. Configure the installation

Edit these three files:

```bash
vi inventory/percona-ha.yaml
vi vars/percona_ha_vars.yml
vi vars/percona_ha_pgbackrest_vars.yml
```

### Inventory

Replace the example hostnames, IP addresses, cluster name, VIP, interface and subnet prefix. Keep exactly three database nodes and one backup server. Do not rename the inventory groups.

```yaml
example-pgsql-node-01:
  ansible_host: example-pgsql-node-01.example.com
  percona_node_ip: "192.0.2.11"
```

The inventory hostname identifies the member; `ansible_host` is used for SSH; `percona_node_ip` is used for cluster communication. The example addresses are reserved for documentation and must be replaced.

```yaml
percona_keepalived_interface: ens192
percona_keepalived_vip: "192.0.2.10"
percona_keepalived_vip_prefix: 24
percona_keepalived_vrid: 51
```

On each database node, identify the interface with `ip -br address`. Use a free VIP and a VRRP ID unique on the relevant network. The stanza name and etcd token default to the cluster name.

### Cluster settings

| Setting | Default |
|---|---|
| PostgreSQL version | 16 |
| Internal PostgreSQL port | 5432 |
| Database mount | /database |
| Database directory | /database/pg16data |
| SELinux | disabled |
| Automatic reboot permission | false |
| HAProxy read-write port | 5000 |
| HAProxy read-only port | 5001 |

### Backup settings

| Setting | Default |
|---|---|
| Backup mount | /backup |
| Repository | /backup/pgbackrest |
| TLS port | 8432 |
| Full backup retention | 2 |
| Differential backup retention | 7 |
| Parallel processes | 4 |
| Compression | zst, level 3 |
| Backup from standby | Enabled |
| WAL archiving | Enabled |
| Archive timeout | 60s |

Retention values are backup counts, not days. Keep PostgreSQL version consistent across all three files and backup storage settings consistent across both vars files. See the comments in each file for details.

### Validate the inventory and connectivity

```bash
ansible-inventory219 -i inventory/percona-ha.yaml --graph
ansible219 -i inventory/percona-ha.yaml 'percona_cluster:percona_backup' -m ping
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_install.yml --syntax-check
```

All four remote servers must return SUCCESS for the ping test. Syntax checking does not validate runtime behavior.

## 4. Install a new cluster

On the controller:

```bash
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_install.yml
```

The installer configures all HA and backup components, generates credentials and certificates, initializes the stanza and configures WAL archiving. It performs controlled rolling restarts and switchover when required. Do not run initdb manually.

If SELinux configuration requires a reboot, and this is a fresh installation where reboots are acceptable:

```bash
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_install.yml -e percona_baseline_allow_reboot=true
```

Do not use this option as a general production maintenance procedure.

All hosts must report `failed=0` and `unreachable=0`. Correct any installation error before continuing. Do not use partial host selections or `--limit` for cluster bootstrap.

### Initial verification

On any database node:

```bash
patronictl -c /etc/patroni/patroni.yml list
etcdctl endpoint health --cluster
```

Expect one running leader, two streaming replicas and three healthy etcd endpoints.

On the backup server (replace the stanza name):

```bash
sudo -u postgres pgbackrest --stanza=example-pgsql check
sudo -u postgres pgbackrest --stanza=example-pgsql --type=full backup
```

For application access, further verification and backup scheduling, follow the [Administration Guide](adminPerconaHA.md).

## 5. Install or reconfigure pgBackRest separately

Use this procedure for a compatible cluster deployed with this installer, not an arbitrary existing PostgreSQL cluster.

Review the inventory and both vars files, then run on the controller:

```bash
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_pgbackrest_install.yml
```

This configures packages, TLS, backup server, clients, stanza and WAL settings. Enabling WAL archiving may require restarts and switchover. Use a maintenance window on an active cluster.

## 6. Reconfigure an installed cluster

The installer is primarily a deployment tool, not a continuous configuration management system. Do not blindly rerun it against a manually customized production cluster: it may overwrite local configuration files.

For PostgreSQL parameter changes, use Patroni as described in the [Administration Guide](adminPerconaHA.md). Bootstrap settings in Ansible do not automatically update an existing cluster's dynamic configuration.

For backup infrastructure changes, review the current configuration and use the separate pgBackRest playbook only when appropriate. Preserve the original credentials and TLS CA material.

## 7. Remove a cluster

**WARNING: Destructive operation. Local database data will be permanently removed.**

Verify the inventory, stop application access, create and verify backups if data must be retained, and confirm the servers are dedicated to this cluster.

On the controller:

```bash
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_uninstall.yml -e percona_confirm_uninstall=true
```

The playbook removes cluster packages, services, configuration and local data from the three database nodes. It does not remove filesystems, the backup server, the backup repository or controller credentials.

### Prepare the backup server for a fresh cluster

A newly initialized PostgreSQL cluster has a different system identifier. Do not reuse the previous stanza repository for an unrelated new cluster.

On the backup server, stop scheduled backup jobs first. Preserve the old configuration, certificates and credentials securely. Verify the backup mount before moving the repository:

```bash
mountpoint -q /backup && systemctl stop pgbackrest && \
  mv /backup/pgbackrest "/backup/pgbackrest-old-$(date +%Y%m%d-%H%M%S)"
```

Confirm the move succeeded before proceeding. If a complete cleanup of the dedicated backup server is required:

```bash
dnf remove -y percona-pgbackrest
rm -rf /etc/pgbackrest /etc/pgbackrest.conf /var/lib/pgbackrest /var/log/pgbackrest
```

Do not delete the preserved repository. Return to section 4 to install a new cluster.

## 8. Post-installation cleanup

The installed services do not depend on Ansible at runtime. The cloned repository may be deleted after successful verification, but securely preserve:

- The actual inventory and both vars files.
- The complete `/root/.percona-credentials/<cluster-name>/` directory on the controller, including passwords and TLS CA private key.
- The installation log and Git commit ID (`git rev-parse HEAD`).
- Records of subsequent manual configuration changes.

Never commit credentials or customer-specific configuration to a public repository.

## 9. Tested functionality and limitations

Oracle Linux 10 / PostgreSQL 16: package installation, etcd, Patroni, replication, HAProxy routing, VIP failover, controlled switchover, failover after controlled service stop, credentials, TLS, stanza, WAL archiving, rolling restart, repository check, full backup, uninstall and reinstallation were tested.

A clean installation exposed a propagation issue which was corrected; the resumed installation succeeded. A further uninterrupted clean installation was not performed. Integrated idempotency was verified with zero changed and failed tasks on all four servers before the final restore_command enhancement.

The automated restore_command enhancement has not been execution-tested. Full restore, PITR, abrupt server failure and disaster recovery remain unvalidated. Other OS and PostgreSQL versions are not validated. Backup scheduling, certificate renewal, storage provisioning and firewall configuration are manual. Migration and major upgrades are out of scope.

## 10. Quick command reference

On the controller:

```bash
# Install
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_install.yml

# Configure backup infrastructure
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_pgbackrest_install.yml

# Remove cluster — DESTRUCTIVE
ansible-playbook219 -i inventory/percona-ha.yaml playbooks/percona_ha_uninstall.yml -e percona_confirm_uninstall=true
```

## References

- [Percona PostgreSQL 16](https://docs.percona.com/postgresql/16/)
- [Percona HA with Patroni](https://docs.percona.com/postgresql/16/solutions/ha-patroni.html)
- [Ansible documentation](https://docs.ansible.com/)
- [Administration Guide](adminPerconaHA.md)
