# Cluster patroni + ETCD + VIP-manager + pgbackrest
## VM раззвернуты в собстенной среде, через Aansible

| Name | CPU | RAM | OS | patroni | etcd | postgresql | pgbackrest | vip-manager |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| otus-pg-0 | 8 | 64 | Ubuntu 20.04.5 LTS | 4.0.4 | 3.4.21 | 14.15 | 2.49-1.pgdg100+1 | 1.0.2 |
| otus-pg-1 | 8 | 64 | Ubuntu 20.04.5 LTS | 4.0.4 | 3.4.21 | 14.15 | 2.49-1.pgdg100+1 | 1.0.2 |
| otus-pg-2 | 8 | 64 | Ubuntu 20.04.5 LTS | 4.0.4 | 3.4.21 | 14.15 | 2.49-1.pgdg100+1 | 1.0.2 |
| otus-pgbkp | 8 | 16 | Ubuntu 20.04.5 LTS | - | - | - | 2.49-1.pgdg100+1 | - |

Для нод postgresql смонтированы разделы для:  
PGDATA: /pg_data  
ETCD:   /var/lib/etcd  
audit:  /pg_audit  

Patroni с синхронной и асинхронной нодой
```
+ Cluster: otus-pg (7450929508804994575) -----------+----+-----------+
| Member     | Host      | Role         | State     | TL | Lag in MB |
+------------+-----------+--------------+-----------+----+-----------+
| otus-pg-0  | otus-pg-0 | Leader       | running   |  1 |           |
| otus-pg-1  | otus-pg-1 | Sync Standby | streaming |  1 |         0 |
| otus-pg-2  | otus-pg-2 | Replica      | streaming |  1 |         0 |
+------------+-----------+--------------+-----------+----+-----------+
```
### patronictl show-config
```
loop_wait: 10
master_start_timeout: 300
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    autovacuum_analyze_scale_factor: 0.01
    autovacuum_vacuum_scale_factor: 0.05
    checkpoint_timeout: 30min
    client_connection_check_interval: 1000
    hot_standby: 'on'
    idle_in_transaction_session_timeout: 3min
    log_directory: /pg_audit/log
    log_filename: postgresql-%Y-%m-%d-%H%M.log
    log_line_prefix: '%m [%p] %q%h %a %u@%d'
    log_rotation_age: 1d
    log_rotation_size: 0
    log_truncate_on_rotation: true
    logging_collector: true
    maintenance_work_mem: 2GB
    max_connections: 300
    max_replication_slots: 10
    max_wal_senders: 10
    max_worker_processes: 8
    wal_keep_segments: 1024
    wal_level: replica
  use_pg_rewind: true
  use_slots: true
retry_timeout: 10
synchronous_mode: true
ttl: 30
```
### postgresql.base.conf
```
# Add settings for extensions here

shared_buffers = '21 GB'
work_mem = '32 MB'
effective_cache_size = '45 GB'
random_page_cost = 1.25
checkpoint_timeout = '30 min'
max_wal_size = '10240 MB'
min_wal_size = '5120 MB'
wal_compression = on
wal_buffers = -1
wal_keep_size = '16 GB'
bgwriter_flush_after = 0
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
max_parallel_workers = 8
track_activity_query_size = '4kB'
restore_command = 'pgbackrest --stanza=otus-pg archive-get %f \"%p\"'
huge_pages = ON
```
### ETCD
#### etcdctl member list
```
3703be9b213df23d: name=otus-pg-0 peerURLs=http://otus-pg-0:2380 clientURLs=http://otus-pg-0:2379 isLeader=false
89819775bccc8fc8: name=otus-pg-2 peerURLs=http://otus-pg-2:2380 clientURLs=http://otus-pg-2:2379 isLeader=true
dbd730a21fb6b1f3: name=otus-pg-1 peerURLs=http://otus-pg-1:2380 clientURLs=http://otus-pg-1:2379 isLeader=false
```
#### etcdctl cluster-health
``` 
member 3703be9b213df23d is healthy: got healthy result from http://otus-pg-0:2379
member 89819775bccc8fc8 is healthy: got healthy result from http://otus-pg-2:2379
member dbd730a21fb6b1f3 is healthy: got healthy result from http://otus-pg-1:2379
cluster is healthy
```
### vip-manager
#### cat /etc/default/vip-manager.yml
```
interval: 1000
trigger-key: "/otus-pg/otus-pg/leader"
trigger-value: "otus-pg-0"
ip: 192.168.50.254 # the virtual ip address to manage
netmask: 24 # netmask for the virtual ip
interface: eth0 #interface to which the virtual ip will be added
hosting-type: basic # possible values: basic, or hetzner.
dcs-type: etcd # etcd or consul
dcs-endpoints:
  - http://192.168.50.14:2379
  - http://192.168.50.15:2379
  - http://192.168.50.16:2379
etcd-user: "root"
etcd-password: "********"
retry-num: 2
retry-after: 250 #in milliseconds
verbose: false
```
### Backup
#### pgbackrest.conf for cluster node
```
[otus-pg]
pg1-path=/pg_data/data
pg1-socket-path=/var/run/postgresql

[global]
repo1-host=otus-pgbkp
```
#### global pgbackrest.conf for otus-pgbkp
```
[global]
process-max=2
repo1-path=/pgbackrest
repo1-retention-full-type=time
repo1-retention-full=15
repo1-retention-diff=7
start-fast=y
log-level-file=detail
config-include-path=/etc/pgbackrest/conf.d

[global:archive-push]
compress-level=3
```
#### stanza (otus-pg.conf) pgbackrest.conf for otus-pgbkp
```
[otus-pg]
pg1-host=otus-pg-0
pg1-path=/pg_data/data
pg1-socket-path=/var/run/postgresql
pg2-host=otus-pg-1
pg2-path=/pg_data/data
pg2-socket-path=/var/run/postgresql
pg3-host=otus-pg-2
pg3-path=/pg_data/data
pg3-socket-path=/var/run/postgresql
```
#### crontab
```
#Ansible: full backup otus-pg
0 23 * * 0 sudo -u pgbackrest pgbackrest --stanza=otus-pg --type=full backup
#Ansible: diff backup otus-pg
0 23 * * 1-6 sudo -u pgbackrest pgbackrest --stanza=otus-pg --type=diff backup
```
#### sudo -u pgbackrest pgbackrest --stanza=otus-pg info
```
WARN: configuration file contains command-line only option 'config-include-path'
stanza: otus-pg
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/00000001000000000000001E

        full backup: 20241221-230001F
            timestamp start/stop: 2024-12-21 23:00:01+03 / 2024-12-21 23:00:07+03
            wal start/stop: 000000010000000000000006 / 000000010000000000000007
            database size: 26.2MB, database backup size: 26.2MB
            repo1: backup set size: 3.3MB, backup size: 3.3MB

        full backup: 20241222-225955F
            timestamp start/stop: 2024-12-22 22:59:55+03 / 2024-12-22 23:00:00+03
            wal start/stop: 00000001000000000000000F / 00000001000000000000000F
            database size: 38.3MB, database backup size: 38.3MB
            repo1: backup set size: 4MB, backup size: 4MB

        diff backup: 20241222-225955F_20241223-225950D
            timestamp start/stop: 2024-12-23 22:59:50+03 / 2024-12-23 22:59:53+03
            wal start/stop: 000000010000000000000015 / 000000010000000000000015
            database size: 38.3MB, database backup size: 90.9KB
            repo1: backup set size: 4MB, backup size: 21.7KB
            backup reference list: 20241222-225955F

        diff backup: 20241222-225955F_20241224-225944D
            timestamp start/stop: 2024-12-24 22:59:44+03 / 2024-12-24 22:59:48+03
            wal start/stop: 00000001000000000000001B / 00000001000000000000001D
            database size: 38.3MB, database backup size: 970.9KB
            repo1: backup set size: 4MB, backup size: 122.4KB
            backup reference list: 20241222-225955F
```
# switcover
```
+ Cluster: otus-pg (7450929508804994575) ----------+----+-----------+
| Member    | Host      | Role         | State     | TL | Lag in MB |
+-----------+-----------+--------------+-----------+----+-----------+
| otus-pg-0 | otus-pg-0 | Leader       | running   |  1 |           |
| otus-pg-1 | otus-pg-1 | Sync Standby | streaming |  1 |         0 |
| otus-pg-2 | otus-pg-2 | Replica      | streaming |  1 |         0 |
+-----------+-----------+--------------+-----------+----+-----------+
```
#### pgbackrest
```
stanza: otus-pg
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/00000001000000000000001E
```
## switchover
```
+ Cluster: otus-pg (7450929508804994575) -----+----+-----------+
| Member    | Host      | Role    | State     | TL | Lag in MB |
+-----------+-----------+---------+-----------+----+-----------+
| otus-pg-0 | otus-pg-0 | Replica | streaming |  2 |         0 |
| otus-pg-1 | otus-pg-1 | Leader  | running   |  2 |           |
| otus-pg-2 | otus-pg-2 | Replica | streaming |  2 |         0 |
+-----------+-----------+---------+-----------+----+-----------+
```
#### pgbackrest
```
stanza: otus-pg
    status: ok
    cipher: none

    db (current)
        wal archive min/max (14): 000000010000000000000001/000000020000000000000021
```

# Восстановление из бэкапа pgbackrest на другой кластер
Предварительно сохранить pg_hba.conf, postgresql.base.conf, pgbackrest.conf. Отредактировать pgbackrest.conf настроить на нужный сервер бэкапов.
1. Перевести кластер в режим обслуживания
2. Остановить PostgreSQL на всех членах кластера
3. Удалить содержимое каталога с данными БД лидера кластера
4. На лидере запустить восстановление из последнего бэкапа  кластера  
    ```
    pgbackrest --stanza=otus-pg --db-path=/pg_data/data --archive-mode=off --log-level-console=info  --log-level-file=info restore
    ```
5. Необходимо дождаться наката всех WAL-логов восстановления. Окончание восстановления отобразится в логе PostgreSQL следующими строками:
    ```
    LOG:  archive recovery complete
    LOG:  database system is ready to accept connections
    ```
6. стартуем postgresql
7. Удалить содержимое каталога с данными БД на репликах кластера  
    ```
    rm -rf /pg_data/data/
    ```
8. Выполнить переинициализацию кластера  
    ```
    patronictl -c /etc/patroni/patroni.yml remove otus-pg
    ```
9. Теперь необходимо инициализировать каждую реплику  
    ```
    patronictl -c /etc/patroni/patroni.yml reinit
    ```
10. После инициализации реплик, проверьте состояние кластера
