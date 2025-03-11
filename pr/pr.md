# **Построение геораспределенного кластера PostgreSQL на базе Patroni в 2 ЦОД**

## **Введение**
В современном мире, где доступность и отказоустойчивость информационных систем играют ключевую роль, использование геораспределенных кластеров становится все более востребованным решением. Одним из таких решений является Patroni — мощный инструмент для управления и автоматизации работы кластеров PostgreSQL, обеспечивающий высокую доступность и надежность баз данных даже в условиях распределенной инфраструктуры.  
## **Задача**
Построить отказоустойчивый/катострофоустойчивый кластер
## **Выбор архитектуры** 

1.  Выбираем 3-х узловой кластер на базе Patroni. В качестве распределенного хранилища конфигурации используем ETCD. В качестве единой точки входа используем vip-manager.    
2. Имеем 2 ЦОД.  
Необходимо рассмотреть схемы построения.  
2.1. В 1 ЦОД помещаем 2 ноды во второй ЦОД 1 ноду синхронную или асинхронную.  
Синхронная нода.  
![patroni1](https://github.com/stalmer120/PostgreSQL.-Advanced/blob/3f6cf1849ce77d64b362900e1432281194910b1b/pr/HA%20cluster%20patroni%202.png)  
Достоинсва:  
- Авто failover
- RPO = 0
Проблемы:  
- Деградация производительности из-за сетевых задержек.  
- Ложное переключение.  
- Нет высокой доступности в ЦОД 2.  
Асинхронная нода.  
![patroni3](https://github.com/stalmer120/PostgreSQL.-Advanced/blob/3f6cf1849ce77d64b362900e1432281194910b1b/pr/HA%20cluster%20patroni%201.png)  
Достоинсва:  
- Авто failover
- Нет деградации  
Проблемы:  
- Ложное переключение.  
- Нет высокой доступности в ЦОД 2.  
- RPO > 0  
2.2 В 1 ЦОД полноценная 3-х нодовая конфигурация, во втором ЦОД асинхроная реплика. 
![patroni2](https://github.com/stalmer120/PostgreSQL.-Advanced/blob/3704894ee94dd71f0ca1ceff1e9ce1e8c3e4207a/pr/HA%20cluster%20patroni%204.png)  
Достоинсва:  
- Высокая доступность в 1 ЦОД
- Нет ложных срабатываний
Проблемы:  
- RPO > 0 
- Нет высокой доступности в 2 ЦОД  
- Ручное переключение.  
2.3 В 1 ЦОД полноценная 3-х нодовая конфигурация, в 2 ЦОД полноценная 3-х нодовая конфигурация. 
![patroni3](https://github.com/stalmer120/PostgreSQL.-Advanced/blob/3f6cf1849ce77d64b362900e1432281194910b1b/pr/HA%20cluster%20patroni%203.png)  
Достоинсва:  
- Высокая доступность в 1 ЦОД и 2 ЦОД
- Нет деградации.
- Нет ложных срабатываний.  
Проблемы:  
- RPO > 0 
- Ручное переключение.  
### **Выбор: полноценная 3-х нодовая конфигурация в обоих ЦОД**  
Обоснование. Данная схема имеет наименьшие проблемы и будем учитывать, что падение любого из ЦОД это черезвучайная ситуация и ручное переключение здесь наименьшая проблема.
## **Утановка**  
Считаем, что инфраструктура подготовлена в ЦОД 1 и 2 сетевое взаимодествие настроено, ВМ созданы и сервер Ansible имеется.  
Выполним playbook для установки компонентов.  
Будет приведены только часть основных ролей.  
````yml
---
# vim: syntax=yaml tabstop=2 shiftwidth=2
- hosts: etcd,patroni,pgbackrest
  vars_files:
    - group_vars/{{ env }}.yml
    - group_vars/patroni.yml
  roles:
    - patroni-preinstall

- hosts: etcd
  vars_files:
    - group_vars/{{ env }}.yml
    - group_vars/patroni.yml
  roles:
    - { role: etcd, tags: etcd }
  post_tasks:
    - include_tasks: roles/etcd/tasks/etcd_enabled.yml
    - include_tasks: roles/etcd/tasks/reboot.yml
    - include_tasks: roles/etcd/tasks/change_initial_cluster_state.yml
    - include_tasks: roles/etcd/tasks/etcd_create_root.yml

- hosts: patroni
  vars_files:
    - group_vars/{{ env }}.yml
    - group_vars/patroni.yml
  roles:
    - { role: patroni, tags: patroni }
    - { role: sysctl, tags: [sysctl,preinstall] }
    - { role: vip-manager, tags: vip-manager }

- hosts: patroni,pgbackrest
  vars_files:
    - group_vars/{{ env }}.yml
    - group_vars/patroni.yml
  roles:
    - { role: pgbackrest, tags: pgbackrest }

- hosts: patroni,pgbackrest
  vars_files:
    - group_vars/{{ env }}.yml
    - group_vars/patroni.yml
  roles:
    - { role: pg_monitor, tags: monitoring }
...

````
patroni-preinstall
````yml
---
# vim: syntax=yaml tabstop=2 shiftwidth=2
- name: 'configure postgres repository'
  tags: preinstall
  block:
    - name: 'apt key add postgres repo'
      changed_when: false
      command: |
        apt-key adv --keyserver {{ keyserver }} --keyserver-options http-proxy={{ proxy }} --recv-keys {{ item }}
      loop: "{{ pg_repo_keys | list }}"

    - name: 'install postges repo'
      apt_repository:
        repo: '{{ pg_repo_url }}'
        state: present
        update_cache: true
  when: inventory_hostname in groups['patroni'] or inventory_hostname in groups['pgbackrest']

- name: install python packages
  tags: preinstall
  apt:
    name: '{{ item }}'
    state: present
    update_cache: true
    default_release: '{{ pg_default_release }}'
  loop:
    - python3-distutils
    - python3-etcd
    - python3-psutil
    - python3-psycopg2
    - python3-yaml

- name: install python packages
  tags: preinstall
  apt:
    name: '{{ item }}'
    state: present
    update_cache: true
    default_release: '{{ pg_default_release }}'
  loop:
    - python-psycopg2
  when: ansible_distribution | lower == 'astra linux'

- name: configure /etc/hosts
  template:
    src: patroni_hosts.j2
    dest: /etc/hosts
    owner: root
    group: root
    mode: 0644
    backup: true

````
role: patroni
````yml
---
- name: 'install postgres'
  apt:
    name: '{{ item }}'
    default_release: '{{ pg_default_release }}'
    update_cache: true
  loop:
    - postgresql-common
    - postgresql-client-{{ pg_major_version }}
    - postgresql-{{ pg_major_version }}
    - postgresql-{{ pg_major_version }}-repack
    - postgresql-{{ pg_major_version }}-repmgr
    - postgresql-contrib-{{ pg_major_version }}
  notify:
    - postgres disabled
  tags: patroni

- name: disable postgres
  systemd:
    name: postgresql
    state: stopped
    enabled: false
  tags: patroni

- name: 'create postgres directories'
  file:
    path: '{{ item }}'
    state: directory
    owner: '{{ pg_user }}'
    group: '{{ pg_group }}'
    mode: 0700
  loop:
    - '/pg_audit'
    - '/pg_data'
    - '/pg_data/data'
  tags: patroni

- name: 'install patroni from repo'
  apt:
    name: '{{ item }}'
    default_release: '{{ pg_default_release }}'
    state: present
  loop:
    - patroni
  tags: patroni

- name: 'create patrony config directory'
  file:
    path: '/etc/patroni'
    state: directory
    owner: root
    group: root
    mode: 0755
  tags: patroni

- name: 'generate patroni systemd unit'
  template:
    src: 'patroni.service.j2'
    dest: '/lib/systemd/system/patroni.service'
    owner: root
    group: root
    mode: 0644
    backup: true
  tags: patroni

- name: 'generate patroni dcs config'
  template:
    src: dcs.yml.j2
    dest: /etc/patroni/dcs.yml
    owner: root
    group: root
    mode: 0644
    backup: true
  tags: patroni

- name: 'generate patroni config'
  template:
    src: 'patroni.yml.j2'
    dest: '/etc/patroni/patroni.yml'
    owner: root
    group: root
    mode: 0644
    backup: true
  notify:
    - 'patroni enabled'
    - 'patroni restarted'
  tags: patroni

- name: 'copy sccript for postgresql log rotate'
  template: 
    src: 'rotate.sh.j2'
    dest: '/var/lib/postgresql/pglogrotate.sh'
    owner: '{{ pg_user }}'
    group: '{{ pg_group }}'
    mode: '0777'
  tags: patroni

- name: 'create crontask for logrotate'
  cron: 
    name: 'Logrotate postgresql'
    hour: '1'
    minute: '0'
    job: '/bin/bash /var/lib/postgresql/pglogrotate.sh > /var/log/postgresql/logrotate.log 2>&1'
  tags: patroni

...
````


## **Тестирование**  
### **Изначальная конфигурация** 
**1 ЦОД**  
*Диски*
````
sdb                               8:16   0   10G  0 disk
└─pg_data_ssd_vg-pg_data_ssd_lv 253:5    0   10G  0 lvm  /pg_data
sdc                               8:32   0    3G  0 disk
└─etcd_nvme_vg-etcd_nvme_lv     253:4    0    3G  0 lvm  /var/lib/etcd
sdd                               8:48   0    5G  0 disk
└─pg_audit_vg-pg_audit_lv       253:6    0    5G  0 lvm  /pg_audit
````
*ETCD*  
````
etcdctl cluster-health
member 1aa9a8cf4dff6406 is healthy: got healthy result from http://stsphr-pgc002lk:2379
member 349beba35f0bca89 is healthy: got healthy result from http://stsphr-pgc003lk:2379
member b3b40e88f9f7ac22 is healthy: got healthy result from http://stsphr-pgc001lk:2379
cluster is healthy
````
*Patroni*
````
+ Cluster: stsphr1 -----------------+--------------+-----------+----+-----------+
| Member          | Host            | Role         | State     | TL | Lag in MB |
+-----------------+-----------------+--------------+-----------+----+-----------+
| stsphr-pgc001lk | stsphr-pgc001lk | Leader       | running   |  2 |           |
| stsphr-pgc002lk | stsphr-pgc002lk | Sync Standby | streaming |  2 |         0 |
| stsphr-pgc003lk | stsphr-pgc003lk | Replica      | streaming |  2 |         0 |
+-----------------+-----------------+--------------+-----------+----+-----------+
````
*vip-manager*
````
● vip-manager.service - Manages Virtual IP for Patroni
   Loaded: loaded (/lib/systemd/system/vip-manager.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2025-03-11 10:59:39 MSK; 7h ago
 Main PID: 5988 (vip-manager)
    Tasks: 8 (limit: 4633)
   Memory: 7.2M
      CPU: 17.685s
   CGroup: /system.slice/vip-manager.service
           └─5988 /usr/bin/vip-manager --config=/etc/default/vip-manager.yml

Mar 11 18:40:19 stsphr-pgc002lk.inno.local vip-manager[5988]: 2025/03/11 18:40:19 IP address 10.10.255.88/24 state is true, desired true


````
**2 ЦОД**  
*Диски*
````
sdb                               8:16   0   10G  0 disk
└─pg_data_ssd_vg-pg_data_ssd_lv 253:5    0   10G  0 lvm  /pg_data
sdc                               8:32   0    3G  0 disk
└─etcd_nvme_vg-etcd_nvme_lv     253:4    0    3G  0 lvm  /var/lib/etcd
sdd                               8:48   0    5G  0 disk
└─pg_audit_vg-pg_audit_lv       253:6    0    5G  0 lvm  /pg_audit
````
*ETCD*  
````
 etcdctl cluster-health
member 2384c42a6bae4585 is healthy: got healthy result from http://stsphr-pgc005lk:2379
member 3f6df7bce7323733 is healthy: got healthy result from http://stsphr-pgc006lk:2379
member 79316af25795524e is healthy: got healthy result from http://stsphr-pgc004lk:2379
cluster is healthy
````
*Patroni*  
````

+ Cluster: stsphr2 -----------------+--------------+-----------+----+-----------+
| Member          | Host            | Role         | State     | TL | Lag in MB |
+-----------------+-----------------+--------------+-----------+----+-----------+
| stsphr-pgc004lk | stsphr-pgc004lk | Replica      | streaming |  2 |         0 |
| stsphr-pgc005lk | stsphr-pgc005lk | Sync Standby | streaming |  2 |         0 |
| stsphr-pgc006lk | stsphr-pgc006lk | Leader       | running   |  2 |           |
+-----------------+-----------------+--------------+-----------+----+-----------+
````
*vip-manager*  
````
● vip-manager.service - Manages Virtual IP for Patroni
   Loaded: loaded (/lib/systemd/system/vip-manager.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2025-03-11 11:02:51 MSK; 7h ago
 Main PID: 6085 (vip-manager)
    Tasks: 8 (limit: 4633)
   Memory: 7.2M
      CPU: 17.607s
   CGroup: /system.slice/vip-manager.service
           └─6085 /usr/bin/vip-manager --config=/etc/default/vip-manager.yml

Mar 11 18:51:11 stsphr-pgc004lk.inno.local vip-manager[6085]: 2025/03/11 18:51:11 IP address 10.10.255.117/24 state is true, desired true

````
Вносим изменения pg_hba.conf если необходимо, на ВМ в обоих ЦОД.  
Добавим в секцию bootstrap через patronictl edit-config следующие параметры.  
Согласно документации:
````yml
bootstrap:
    dcs:
        standby_cluster:
            host: 1.2.3.4
            port: 5432
            primary_slot_name: patroni
            create_replica_methods:
            - basebackup
````
Но достаточно host и port
````yml
bootstrap:
    dcs:
      standby_cluster:
      host: 10.10.255.88
      port: 5432
````
````
 patronictl -c /etc/patroni/patroni.yml edit-config
---
+++
@@ -29,3 +29,6 @@
 retry_timeout: 10
 synchronous_mode: true
 ttl: 30
+standby_cluster:
+  host: 10.10.255.88
+  port: 5432

Apply these changes? [y/N]: y
Configuration changed
````
### **Проверяем**  
**1ЦОД**  
````
  client_addr  |   state   | sync_priority | sync_state |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn
---------------+-----------+---------------+------------+------------+------------+------------+------------
 10.10.255.114 | streaming |             1 | sync       | 0/1D000000 | 0/1D000000 | 0/1D000000 | 0/1D000000
 10.10.255.110 | streaming |             0 | async      | 0/1D000000 | 0/1D000000 | 0/1D000000 | 0/1D000000
 10.10.255.115 | streaming |             0 | async      | 0/1D000000 | 0/1D000000 | 0/1D000000 | 0/1D000000
````
**2ЦОД** 
````
  client_addr  |   state   | sync_priority | sync_state |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn
---------------+-----------+---------------+------------+------------+------------+------------+------------
 10.10.255.4   | streaming |             0 | async      | 0/1D000000 | 0/1D000000 | 0/1D000000 | 0/1D000000
 10.10.255.116 | streaming |             0 | async      | 0/1D000000 | 0/1D000000 | 0/1D000000 | 0/1D000000

-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 16059
status                | streaming
receive_start_lsn     | 0/19000000
receive_start_tli     | 3
written_lsn           | 0/21000000
flushed_lsn           | 0/21000000
received_tli          | 3
last_msg_send_time    | 2025-03-11 19:40:36.092366+03
last_msg_receipt_time | 2025-03-11 19:40:36.092501+03
latest_end_lsn        | 0/21000000
latest_end_time       | 2025-03-11 19:35:05.380009+03
slot_name             |
sender_host           | 10.10.255.88
sender_port           | 5432
conninfo              | user=replicator passfile=/tmp/pgpass channel_binding=prefer dbname=replication host=10.10.255.88 port=5432 application_name=stsphr-pgc004lk fallback_application_name=stsphr2 sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable
````
*Создадим на мастере 1ЦОД БД otus и проверим на реплике 2ЦОД её наличие.*
````
postgres=# create database otus;
CREATE DATABASE
postgres=# \l+
                                                                    List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+-----------------------+---------+------------+--------------------------------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8537 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | unmodifiable empty database
           |          |          |             |             | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | default template for new databases
           |          |          |             |             | postgres=CTc/postgres |         |            |
(4 rows)
````
*2ЦОД*
````
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

````
### 1\. Switchover HA cluster 1ЦОД  
*Делаем переключение в 1ЦОД*
````
root@stsphr-pgc002lk:~# patronictl -c /etc/patroni/patroni.yml switchover --force
Current cluster topology
+ Cluster: stsphr1 -----------------+--------------+-----------+----+-----------+
| Member          | Host            | Role         | State     | TL | Lag in MB |
+-----------------+-----------------+--------------+-----------+----+-----------+
| stsphr-pgc001lk | stsphr-pgc001lk | Replica      | streaming |  3 |         0 |
| stsphr-pgc002lk | stsphr-pgc002lk | Leader       | running   |  3 |           |
| stsphr-pgc003lk | stsphr-pgc003lk | Sync Standby | streaming |  3 |         0 |
+-----------------+-----------------+--------------+-----------+----+-----------+

+ Cluster: stsphr1 -----------------+--------------+-----------+----+-----------+
| Member          | Host            | Role         | State     | TL | Lag in MB |
+-----------------+-----------------+--------------+-----------+----+-----------+
| stsphr-pgc001lk | stsphr-pgc001lk | Sync Standby | streaming |  4 |         0 |
| stsphr-pgc002lk | stsphr-pgc002lk | Replica      | streaming |  4 |         0 |
| stsphr-pgc003lk | stsphr-pgc003lk | Leader       | running   |  4 |           |
+-----------------+-----------------+--------------+-----------+----+-----------+
````
**Поверяем**  
*stsphr-pgc003lk*
````
  client_addr  |   state   | sync_priority | sync_state |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn
---------------+-----------+---------------+------------+------------+------------+------------+------------
 10.10.255.110 | streaming |             1 | sync       | 0/22000000 | 0/22000000 | 0/22000000 | 0/22000000
 10.10.255.109 | streaming |             0 | async      | 0/22000000 | 0/22000000 | 0/22000000 | 0/22000000
 10.10.255.115 | streaming |             0 | async      | 0/22000000 | 0/22000000 | 0/22000000 | 0/22000000
(3 rows)
````
*stsphr-pgc004lk*
````
-[ RECORD 1 ]---------+----------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 17048
status                | streaming
receive_start_lsn     | 0/21000000
receive_start_tli     | 4
written_lsn           | 0/22000000
flushed_lsn           | 0/22000000
received_tli          | 4
last_msg_send_time    | 2025-03-11 19:45:26.727109+03
last_msg_receipt_time | 2025-03-11 19:45:26.727426+03
latest_end_lsn        | 0/22000000
latest_end_time       | 2025-03-11 19:43:56.438955+03
slot_name             |
sender_host           | 10.10.255.88
sender_port           | 5432
conninfo              | user=replicator passfile=/tmp/pgpass host=10.10.255.88 port=5432 sslmode=prefer application_name=stsphr-pgc004lk gssencmode=prefer channel_binding=prefer

````
### 2\. Switchover HA cluster 2ЦОД 
````
 patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: stsphr2 -----------------+----------------+-----------+----+-----------+
| Member          | Host            | Role           | State     | TL | Lag in MB |
+-----------------+-----------------+----------------+-----------+----+-----------+
| stsphr-pgc004lk | stsphr-pgc004lk | Standby Leader | streaming |  4 |           |
| stsphr-pgc005lk | stsphr-pgc005lk | Sync Standby   | streaming |  4 |         0 |
| stsphr-pgc006lk | stsphr-pgc006lk | Replica        | streaming |  4 |         0 |
+-----------------+-----------------+----------------+-----------+----+-----------+

patronictl -c /etc/patroni/patroni.yml list
+ Cluster: stsphr2 -----------------+----------------+-----------+----+-----------+
| Member          | Host            | Role           | State     | TL | Lag in MB |
+-----------------+-----------------+----------------+-----------+----+-----------+
| stsphr-pgc004lk | stsphr-pgc004lk | Replica        | streaming |  4 |         0 |
| stsphr-pgc005lk | stsphr-pgc005lk | Standby Leader | streaming |  4 |           |
| stsphr-pgc006lk | stsphr-pgc006lk | Replica        | streaming |  4 |         0 |
+-----------------+-----------------+----------------+-----------+----+-----------+
````
**Поверяем**  
*stsphr-pgc005lk*
````
SELECT * FROM pg_stat_wal_receiver\gx
-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 20530
status                | streaming
receive_start_lsn     | 0/22000000
receive_start_tli     | 4
written_lsn           |
flushed_lsn           | 0/22000000
received_tli          | 4
last_msg_send_time    | 2025-03-11 19:50:18.004727+03
last_msg_receipt_time | 2025-03-11 19:50:18.004933+03
latest_end_lsn        | 0/22000000
latest_end_time       | 2025-03-11 19:49:17.879687+03
slot_name             |
sender_host           | 10.10.255.88
sender_port           | 5432
conninfo              | user=replicator passfile=/tmp/pgpass channel_binding=prefer dbname=replication host=10.10.255.88 port=5432 application_name=stsphr-pgc005lk fallback_application_name=stsphr2 sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

````
### 3\. shutdown HA cluster 1ЦОД
*Создадим еще 1 БД otus1 и переведем кластер в 2ЦОД в режим мастрера*
````
                                                                    List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+-----------------------+---------+------------+--------------------------------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 otus2     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8537 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | unmodifiable empty database
           |          |          |             |             | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | default template for new databases
           |          |          |             |             | postgres=CTc/postgres |         |            |
(5 rows)
````
Переключение Standby cluster в режим мастера сделаем через скрипт

````bash
#!/bin/bash
#
vip=10.169.11.19

lead=`patronictl -c /etc/patroni/patroni.yml list |grep Leader |awk '{print $2}'`
echo $lead

user=`cat /etc/patroni/patroni.yml | grep -C6 restapi |grep username |awk '{print $2}'`
echo $user

pass=`cat /etc/patroni/patroni.yml | grep -C6 restapi |grep password |awk '{print $2}'`
echo $pass

curl -u $user:$pass -s -XPATCH -d \
        '{"standby_cluster":null}' \
        http://$vip:8008/config |jq .

````
````
+ Cluster: stsphr2 -----------------+--------------+-----------+----+-----------+
| Member          | Host            | Role         | State     | TL | Lag in MB |
+-----------------+-----------------+--------------+-----------+----+-----------+
| stsphr-pgc004lk | stsphr-pgc004lk | Replica      | streaming |  5 |         0 |
| stsphr-pgc005lk | stsphr-pgc005lk | Leader       | running   |  5 |           |
| stsphr-pgc006lk | stsphr-pgc006lk | Sync Standby | streaming |  5 |         0 |
+-----------------+-----------------+--------------+-----------+----+-----------+
````
stsphr-pgc005lk проверим создание БД
````
postgres=# create database otus3;
CREATE DATABASE
postgres=# \l+
                                                                    List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+-----------------------+---------+------------+--------------------------------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 otus2     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 otus3     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8385 kB | pg_default |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8537 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | unmodifiable empty database
           |          |          |             |             | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8385 kB | pg_default | default template for new databases
           |          |          |             |             | postgres=CTc/postgres |         |            |
(6 rows)

````
### 3\. Вернем кластер 2ЦОД в режим stanbay  
````yml
root@stsphr-pgc005lk:~# patronictl -c /etc/patroni/patroni.yml edit-config
---
+++
@@ -29,3 +29,6 @@
 retry_timeout: 10
 synchronous_mode: true
 ttl: 30
+standby_cluster:
+  host: 10.10.255.88
+  port: 5432

Apply these changes? [y/N]: y
Configuration changed
````
*stsphr-pgc005lk*
````
 SELECT * FROM pg_stat_wal_receiver\gx
-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 21001
status                | streaming
receive_start_lsn     | 0/23000000
receive_start_tli     | 4
written_lsn           | 0/24000000
flushed_lsn           | 0/24000000
received_tli          | 4
last_msg_send_time    | 2025-03-11 20:05:57.386572+03
last_msg_receipt_time | 2025-03-11 20:05:57.386819+03
latest_end_lsn        | 0/24000000
latest_end_time       | 2025-03-11 20:04:57.026184+03
slot_name             |
sender_host           | 10.10.255.88
sender_port           | 5432
conninfo              | user=replicator passfile=/tmp/pgpass channel_binding=prefer dbname=replication host=10.10.255.88 port=5432 application_name=stsphr-pgc005lk fallback_application_name=stsphr2 sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable
````
````
+ Cluster: stsphr2 -----------------+----------------+-----------+----+-----------+
| Member          | Host            | Role           | State     | TL | Lag in MB |
+-----------------+-----------------+----------------+-----------+----+-----------+
| stsphr-pgc004lk | stsphr-pgc004lk | Replica        | streaming |  4 |         0 |
| stsphr-pgc005lk | stsphr-pgc005lk | Standby Leader | streaming |  4 |           |
| stsphr-pgc006lk | stsphr-pgc006lk | Sync Standby   | streaming |  4 |         0 |
+-----------------+-----------------+----------------+-----------+----+-----------+
````
*stsphr-pgc006lk*
````
root@stsphr-pgc006lk:~# sudo -u postgres psql -l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 otus2     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)
````
## **Заключение**
Standby cluster от patroni довольно прост в реализации и обслуживании, но при этом надежен и обеспечивает высокую доступность. При этом также легко перевести кластер 1ЦОД в режим sandby cluster.
