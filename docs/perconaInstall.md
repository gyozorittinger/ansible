# How to install Percona with playbooks

## Prerequisit

- Install chrony for NTP

## Step 1: Percona Cluster install

### Playbook
percona_install.yml

This playbook installs the required components like
- Keepalived
- ETCD cluster
- HaProxy
- Percona repository
- Percona Cluster

### Vars

vars/percona_install_vars.yml

### Usage

```bash
ansible-playbook playbooks/percona_install.yml -i inventory/percona -v
```

### Tags

You can skip some steps with tags:

Don't install keepalived
```bash
ansible-playbook playbooks/percona_install.yml -i inventory/percona -v --skip-tags keepalived_script,keepalived
```

Skip reboot after update:
```bash
ansible-playbook playbooks/percona_install.yml -i inventory/percona -v --skip-tags reboot
```

## Step 2: Modify Percona Cluster config

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```
Sample config (Fine tune memeory parameters if needed):
```bash
loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    archive_command: pgbackrest --stanza=cluster_1 archive-push /database/pg_wal/%f
    archive_mode: true
    archive_timeout: 600s
    hot_standby: true
    logging_collector: 'on'
    maintenance_work_mem: 1024MB
    max_connections: 1000
    max_parallel_maintenance_workers: 8
    max_replication_slots: 10
    max_wal_senders: 5
    max_wal_size: 50000MB
    shared_buffers: 6144MB
    temp_buffers: 8MB
    wal_keep_segments: 10
    wal_level: replica
    wal_log_hints: true
    wal_segment_size: 64MB
    work_mem: 4MB
  recovery_conf:
    restore_command: pgbackrest --config=/etc/pgbackrest.conf --stanza=cluster_1 archive-get %f %p
  use_pg_rewind: true
  use_slots: true
retry_timeout: 10
ttl: 30
```


## Step 3: PGBackrest install

This playbook install the PGBackrest Component

### Playbook
pgbackrest_install.yml

### Usage

```bash

ansible-playbook playbooks/pgbackrest_install.yml -i inventory/percona -v
```

### Tags

Skip ssh key copy
```bash


ansible-playbook playbooks/pgbackrest_install.yml -i inventory/percona -v --skip-tags ssh-key-copy
```
## Step 4: Create Stanza and configure scheduled backup

On BGBackrest server:

```bash

sudo -iu postgres pgbackrest --stanza=cluster_1 stanza-create
 
sudo -iu postgres pgbackrest --stanza=cluster_1 --type=full backup
 
sudo -iu postgres pgbackrest --stanza=cluster_1 info
```

Edit cron from scheduled bachup:

/etc/cron.d/postrges_backup.sh

```bash

0 2 * * * postgres pgbackrest --stanza=cluster_1 --type=full backup --repo1-retention-full-type=count --repo1-retention-full=3

```
