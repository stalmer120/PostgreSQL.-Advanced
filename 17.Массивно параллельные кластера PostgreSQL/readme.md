# **Массивно параллельные кластера PostgreSQL**

- Развернем Yogabyte и 1 инстанс PostgreSQL
- Сравним их производительность



### **Yogabyte Deployment**
**Creating a network and subnet**
```
[root@test2 hw-11]# yc vpc network create --name "otus-net" --description "otus-net"
id: enphi3n2puv8mqnld9hq
folder_id: b1gidntdt0mi4p0gpkh5
created_at: "2025-01-06T15:18:35Z"
name: otus-net
description: otus-net
default_security_group_id: enpafatja1np0m9hq38u

[root@test2 hw-11]# yc vpc subnet create --name otus-subnet --range 10.95.111.0/24 --network-name otus-net --description "otus-subnet"
id: e9b34sqvgba21idi6at8
folder_id: b1gidntdt0mi4p0gpkh5
created_at: "2025-01-06T15:19:01Z"
name: otus-subnet
description: otus-subnet
network_id: enphi3n2puv8mqnld9hq
zone_id: ru-central1-a
v4_cidr_blocks:
  - 10.95.111.0/24
```

**Create VM**
```
[root@test2 hw-11]# for i in {1..6}; do yc compute instance create --name yuga$i --hostname yuga$i --cores 4 --memory 8 --core-fraction 20 --preemptible --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=almalinux-8 --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key /home/voronov/.ssh/id_rsa.pub --async ; done

[root@test2 hw-11]# yc compute instance list 
+----------------------+-------+---------------+---------+----------------+--------------+
|          ID          | NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+-------+---------------+---------+----------------+--------------+
| fhmavd55mp1efs10ta9i | yuga6 | ru-central1-a | RUNNING | 158.160.49.37  | 10.95.111.19 |
| fhmd277elvsasuh8d85n | yuga5 | ru-central1-a | RUNNING | 158.160.50.55  | 10.95.111.26 |
| fhmkpd700aaepeeseln7 | yuga4 | ru-central1-a | RUNNING | 51.250.13.59   | 10.95.111.20 |
| fhmpgjou9e1t3konmsmu | yuga3 | ru-central1-a | RUNNING | 89.169.150.171 | 10.95.111.31 |
| fhmqtipm7n05n1bg0sj2 | yuga1 | ru-central1-a | RUNNING | 51.250.0.85    | 10.95.111.37 |
| fhmt5dkqi1nd8979apjk | yuga2 | ru-central1-a | RUNNING | 89.169.142.204 | 10.95.111.28 |
+----------------------+-------+---------------+---------+----------------+--------------+
```

**Run commands on all nodes**
```
[yc-user@yuga1 ~]$ sudo dnf install -y python39 wget
[yc-user@yuga1 ~]$ sudo alternatives --set python /usr/bin/python3
[yc-user@yuga1 ~]$ wget https://downloads.yugabyte.com/releases/2024.2.0.0/yugabyte-2024.2.0.0-b145-linux-x86_64.tar.gz
[yc-user@yuga1 ~]$ sudo mkdir /opt/yugabyte
[yc-user@yuga1 ~]$ sudo mkdir -p /data/yugabyte
[yc-user@yuga1 ~]$ sudo adduser yugabyte
[yc-user@yuga1 ~]$ tar xzf yugabyte-2024.2.0.0-b145-linux-x86_64.tar.gz -C ./
[yc-user@yuga1 ~]$ sudo mv yugabyte-2024.2.0.0/* /opt/yugabyte/
[yc-user@yuga1 ~]$ sudo chown -R yugabyte:yugabyte /opt/yugabyte/
[yc-user@yuga1 ~]$ sudo chown yugabyte:yugabyte /data/yugabyte/
```

**Check post_install on all nodes**
```
[yc-user@yuga1 ~]$ sudo -u yugabyte /opt/yugabyte/bin/post_install.sh
OpenSSL binary: /opt/yugabyte/bin/../bin/../bin/openssl
FIPS module: /opt/yugabyte/bin/../bin/../lib/ossl-modules/fips.so
HMAC : (Module_Integrity) : Pass
SHA1 : (KAT_Digest) : Pass
SHA2 : (KAT_Digest) : Pass
SHA3 : (KAT_Digest) : Pass
TDES : (KAT_Cipher) : Pass
AES_GCM : (KAT_Cipher) : Pass
AES_ECB_Decrypt : (KAT_Cipher) : Pass
RSA : (KAT_Signature) : RNG : (Continuous_RNG_Test) : Pass
Pass
ECDSA : (PCT_Signature) : Pass
ECDSA : (PCT_Signature) : Pass
DSA : (PCT_Signature) : Pass
TLS13_KDF_EXTRACT : (KAT_KDF) : Pass
TLS13_KDF_EXPAND : (KAT_KDF) : Pass
TLS12_PRF : (KAT_KDF) : Pass
PBKDF2 : (KAT_KDF) : Pass
SSHKDF : (KAT_KDF) : Pass
KBKDF : (KAT_KDF) : Pass
HKDF : (KAT_KDF) : Pass
SSKDF : (KAT_KDF) : Pass
X963KDF : (KAT_KDF) : Pass
X942KDF : (KAT_KDF) : Pass
HASH : (DRBG) : Pass
CTR : (DRBG) : Pass
HMAC : (DRBG) : Pass
DH : (KAT_KA) : Pass
ECDH : (KAT_KA) : Pass
RSA_Encrypt : (KAT_AsymmetricCipher) : Pass
RSA_Decrypt : (KAT_AsymmetricCipher) : Pass
RSA_Decrypt : (KAT_AsymmetricCipher) : Pass
INSTALL PASSED
```

**install clockbound on all nodes**
```
[yc-user@yuga1 ~]$ sudo bash /opt/yugabyte/bin/configure_clockbound.sh

[yc-user@yuga1 ~]$ systemctl status clockbound
● clockbound.service - ClockBound
   Loaded: loaded (/usr/lib/systemd/system/clockbound.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2025-01-06 15:56:03 UTC; 4s ago
 Main PID: 8183 (clockbound)
    Tasks: 3 (limit: 48754)
   Memory: 444.0K
   CGroup: /system.slice/clockbound.service
           └─8183 /usr/local/bin/clockbound --max-drift-rate 50
```

**Tuned all nodes**
```
[yc-user@yuga1 ~]$ sudo bash -c 'sysctl vm.swappiness=0 >> /etc/sysctl.conf'
[yc-user@yuga1 ~]$ sudo sysctl kernel.core_pattern=/home/yugabyte/cores/core_%p_%t_%E
kernel.core_pattern = /home/yugabyte/cores/core_%p_%t_%E
[yc-user@yuga1 ~]$ sudo sysctl -w vm.max_map_count=262144
vm.max_map_count = 262144
[yc-user@yuga1 ~]$ sudo bash -c 'sysctl vm.max_map_count=262144 >> /etc/sysctl.conf'
[yc-user@yuga1 ~]$ sysctl vm.max_map_count
vm.max_map_count = 262144
```

**create yugabyte-master.service on all nodes**
```
# /etc/systemd/system/yugabyte-master.service
[Unit]
Wants=network-online.target
After=network-online.target
Description=yugabyte-master

[Service]
RestartForceExitStatus=SIGPIPE
EnvironmentFile=/etc/sysconfig/yugabyte_env
StartLimitInterval=0
ExecStart=/bin/bash -c '/opt/yugabyte/bin/yb-master \
--fs_data_dirs=/data/yugabyte \
--master_addresses 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 \
--rpc_bind_addresses=10.95.111.37:7100 \
--webserver_interface=10.95.111.37 \
--webserver_port=7000 \
--time_source=clockbound \
--leader_failure_max_missed_heartbeat_periods 10 \
--placement_cloud=yandex \
--placement_region=region-a \
--placement_zone=zone-a \
--callhome_collection_level=low \
--logtostderr '

LimitCORE=infinity
TimeoutStartSec=30
WorkingDirectory=/data/yugabyte
LimitNOFILE=1048576
LimitNPROC=12000
RestartSec=5
PermissionsStartOnly=True
User=yugabyte
TimeoutStopSec=300
Restart=always

[Install]
WantedBy=multi-user.target
```

**start yugabyte-master.service on all nodes**
```
[yc-user@yuga1 ~]$ sudo touch /etc/sysconfig/yugabyte_env
[yc-user@yuga1 ~]$ sudo systemctl daemon-reload
[yc-user@yuga1 ~]$ sudo systemctl enable yugabyte-master.service
[yc-user@yuga1 ~]$ sudo systemctl start yugabyte-master.service

[yc-user@yuga1 ~]$ systemctl status yugabyte-master.service
● yugabyte-master.service - yugabyte-master
   Loaded: loaded (/etc/systemd/system/yugabyte-master.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2025-01-06 16:14:38 UTC; 1min 12s ago
 Main PID: 8367 (yb-master)
    Tasks: 31 (limit: 48754)
   Memory: 55.0M
   CGroup: /system.slice/yugabyte-master.service
           └─8367 /opt/yugabyte/bin/yb-master --fs_data_dirs=/data/yugabyte --master_addresses 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 --rpc_bind_addresses=10.95.111.37:7100 --webserver_interface=10.95.111.>
```

**Check status of masters**
```
[yc-user@yuga1 ~]$ /opt/yugabyte/bin/yb-admin --master_addresses 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 list_all_masters
Master UUID                      	RPC Host/Port        	State    	Role 	Broadcast Host/Port 
572cf8c3d1d8470196dd6653acf48545 	10.95.111.37:7100    	ALIVE    	FOLLOWER 	N/A                 
ac025688ee564a4787982ac5d0060acd 	10.95.111.28:7100    	ALIVE    	LEADER 	N/A                 
4409f72677f5477e84e78cf635d72478 	10.95.111.31:7100    	ALIVE    	FOLLOWER 	N/A  
```

**Create systemd unit file to start the yugabyte-tserver service**
```
# /etc/systemd/system/yugabyte-tserver.service
[Unit]
Wants=network-online.target
After=network-online.target
Description=yugabyte-master

[Service]
RestartForceExitStatus=SIGPIPE
EnvironmentFile=/etc/sysconfig/yugabyte_env
StartLimitInterval=0
ExecStart=/bin/bash -c '/opt/yugabyte/bin/yb-tserver \
--fs_data_dirs=/data/yugabyte \
--tserver_master_addrs 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 \
--rpc_bind_addresses=0.0.0.0 \
--enable_ysql \
--pgsql_proxy_bind_address 0.0.0.0:5433 \
--cql_proxy_bind_address 0.0.0.0:9042 \
--time_source=clockbound \
--leader_failure_max_missed_heartbeat_periods 10 \
--placement_cloud=yandex \
--placement_region=region-a \
--placement_zone=zone-a \
--logtostderr '

LimitCORE=infinity
TimeoutStartSec=30
WorkingDirectory=/data/yugabyte
LimitNOFILE=1048576
LimitNPROC=12000
RestartSec=5
PermissionsStartOnly=True
User=yugabyte
TimeoutStopSec=300
Restart=always

[Install]
WantedBy=multi-user.target
```

**Start yugabyte-tserver service on all nodes**
```
[yc-user@yuga1 ~]$ sudo systemctl daemon-reload
[yc-user@yuga1 ~]$ sudo systemctl enable yugabyte-tserver.service
[yc-user@yuga1 ~]$ sudo systemctl start yugabyte-tserver.service

[yc-user@yuga1 ~]$ sudo systemctl status yugabyte-tserver.service
● yugabyte-tserver.service - yugabyte-master
   Loaded: loaded (/etc/systemd/system/yugabyte-tserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2025-01-06 17:47:44 UTC; 2min 3s ago
 Main PID: 20773 (yb-tserver)
    Tasks: 62 (limit: 48754)
   Memory: 114.0M
   CGroup: /system.slice/yugabyte-tserver.service
           ├─20773 /opt/yugabyte/bin/yb-tserver --fs_data_dirs=/data/yugabyte --tserver_master_addrs 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 --rpc_bind_addresses=0.0.0.0 --enable_ysql --pgsql_proxy_bind_address 0.0.0.0:5433 --cql_>
           ├─20828 /opt/yugabyte/postgres/bin/postgres -D /data/yugabyte/pg_data -p 5433 -h 0.0.0.0 -k /tmp/.yb.0.0.0.0:5433 -c unix_socket_permissions=0700 -c yb_pg_metrics.node_name=yuga1.ru-central1.internal:9000 -c yb_pg_metrics.port=13000 >
           ├─20852 postgres: YSQL webserver   
           ├─20854 postgres: checkpointer   
           └─20855 postgres: stats collector  
```

**Check status yugabyte-tserver**
```
[yc-user@yuga1 ~]$ /opt/yugabyte/bin/yb-admin --master_addresses 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 list_all_tablet_servers
Tablet Server UUID               RPC Host/Port Heartbeat delay Status   Reads/s  Writes/s Uptime   SST total size  SST uncomp size SST #files      Memory   Broadcast Host/Port 
9990e7f99c7a4d99b9d18376f349e370 yuga5.ru-central1.internal:9100 0.04s           ALIVE    0.00     0.00     201      0 B             0 B             0               57.43 MB N/A
4b9bd70ece114bb8b6b8676367ee733c yuga4.ru-central1.internal:9100 0.03s           ALIVE    0.00     0.00     307      0 B             0 B             0               63.10 MB N/A
9846f830a9f644249f6df337be0482eb yuga6.ru-central1.internal:9100 0.00s           ALIVE    0.00     0.00     71       0 B             0 B             0               55.49 MB N/A
c5c49c56daa9463da057374c27910616 yuga2.ru-central1.internal:9100 0.05s           ALIVE    0.00     0.00     1182     0 B             0 B             0               58.69 MB N/A
1f5e35bc21634153ac9fd2fca70cbe2b yuga3.ru-central1.internal:9100 0.05s           ALIVE    0.00     0.00     1116     0 B             0 B             0               61.61 MB N/A
86bad98858bc45b0a298777e06d3b597 yuga1.ru-central1.internal:9100 0.08s           ALIVE    0.00     0.00     1417     0 B             0 B             0               56.34 MB N/A
```

**Connect to the YugabyteDB service and check the version**
```
[yc-user@yuga1 ~]$ /opt/yugabyte/bin/ysqlsh -t -h 127.0.0.1 -p 5433 -U yugabyte -c 'SELECT version()'
 PostgreSQL 11.2-YB-2024.2.0.0-b0 on x86_64-pc-linux-gnu, compiled by clang version 17.0.6 (https://github.com/yugabyte/llvm-project.git 9b881774e40024e901fc6f3d313607b071c08631), 64-bit
```



### **Uploading a dataset to Yogabyte**
**Download dataset opensky**
```
[yc-user@yuga1 ~]$ wget -O- https://zenodo.org/record/5092942 | grep -oP 'https://zenodo.org/records/5092942/files/flightlist_\d+_\d+\.csv\.gz' | xargs wget

[yc-user@yuga1 ~]$ ls -al flightlist*
-rw-rw-r--. 1 yc-user yc-user 149656072 Jan  6 18:56 flightlist_20190101_20190131.csv.gz
-rw-rw-r--. 1 yc-user yc-user 139872800 Jan  6 18:52 flightlist_20190201_20190228.csv.gz
-rw-rw-r--. 1 yc-user yc-user 159072441 Jan  6 18:55 flightlist_20190301_20190331.csv.gz
-rw-rw-r--. 1 yc-user yc-user 166006708 Jan  6 18:55 flightlist_20190401_20190430.csv.gz
-rw-rw-r--. 1 yc-user yc-user 177692774 Jan  6 18:52 flightlist_20190501_20190531.csv.gz
-rw-rw-r--. 1 yc-user yc-user 186373210 Jan  6 18:54 flightlist_20190601_20190630.csv.gz
-rw-rw-r--. 1 yc-user yc-user 203480200 Jan  6 18:54 flightlist_20190701_20190731.csv.gz
-rw-rw-r--. 1 yc-user yc-user 210148935 Jan  6 18:53 flightlist_20190801_20190831.csv.gz
-rw-rw-r--. 1 yc-user yc-user 191374713 Jan  6 18:53 flightlist_20190901_20190930.csv.gz
-rw-rw-r--. 1 yc-user yc-user 206917730 Jan  6 18:53 flightlist_20191001_20191031.csv.gz
-rw-rw-r--. 1 yc-user yc-user 190775945 Jan  6 18:53 flightlist_20191101_20191130.csv.gz
-rw-rw-r--. 1 yc-user yc-user 189553155 Jan  6 18:53 flightlist_20191201_20191231.csv.gz
-rw-rw-r--. 1 yc-user yc-user 193891069 Jan  6 18:54 flightlist_20200101_20200131.csv.gz
-rw-rw-r--. 1 yc-user yc-user 186334754 Jan  6 18:55 flightlist_20200201_20200229.csv.gz
-rw-rw-r--. 1 yc-user yc-user 151571888 Jan  6 18:55 flightlist_20200301_20200331.csv.gz
-rw-rw-r--. 1 yc-user yc-user  58544368 Jan  6 18:55 flightlist_20200401_20200430.csv.gz
-rw-rw-r--. 1 yc-user yc-user  75376842 Jan  6 18:55 flightlist_20200501_20200531.csv.gz
-rw-rw-r--. 1 yc-user yc-user 100336756 Jan  6 18:54 flightlist_20200601_20200630.csv.gz
-rw-rw-r--. 1 yc-user yc-user 134445252 Jan  6 18:54 flightlist_20200701_20200731.csv.gz
-rw-rw-r--. 1 yc-user yc-user 144364225 Jan  6 18:53 flightlist_20200801_20200831.csv.gz
-rw-rw-r--. 1 yc-user yc-user 136524682 Jan  6 18:53 flightlist_20200901_20200930.csv.gz
-rw-rw-r--. 1 yc-user yc-user 138560754 Jan  6 18:53 flightlist_20201001_20201031.csv.gz
-rw-rw-r--. 1 yc-user yc-user 126932585 Jan  6 18:52 flightlist_20201101_20201130.csv.gz
-rw-rw-r--. 1 yc-user yc-user 132372973 Jan  6 18:52 flightlist_20201201_20201231.csv.gz
-rw-rw-r--. 1 yc-user yc-user 123902516 Jan  6 18:52 flightlist_20210101_20210131.csv.gz
-rw-rw-r--. 1 yc-user yc-user 112332587 Jan  6 18:53 flightlist_20210201_20210228.csv.gz
-rw-rw-r--. 1 yc-user yc-user 144126125 Jan  6 18:53 flightlist_20210301_20210331.csv.gz
-rw-rw-r--. 1 yc-user yc-user 154290585 Jan  6 18:53 flightlist_20210401_20210430.csv.gz
-rw-rw-r--. 1 yc-user yc-user 158083429 Jan  6 18:53 flightlist_20210501_20210530.csv.gz
-rw-rw-r--. 1 yc-user yc-user 174242634 Jan  6 18:53 flightlist_20210601_20210630.csv.gz
```

**Create DB and table for dataset**
```
yugabyte=# CREATE DATABASE opensky;
CREATE DATABASE
yugabyte=# \dx
 pg_stat_statements | 1.6-yb-1.0 | pg_catalog | track execution statistics of all SQL statements executed
 plpgsql            | 1.0        | pg_catalog | PL/pgSQL procedural language

yugabyte=# \c opensky 

opensky=# CREATE TABLE opensky
(
    callsign TEXT,
    number TEXT,
    icao24 TEXT,
    registration TEXT,
    typecode TEXT,
    origin TEXT,
    destination TEXT NULL,
    firstseen TIMESTAMP WITH TIME ZONE NOT NULL,
    lastseen TIMESTAMP WITH TIME ZONE NOT NULL,
    day TIMESTAMP WITH TIME ZONE NOT NULL,
    latitude_1 NUMERIC,
    longitude_1 NUMERIC,
    altitude_1 NUMERIC,
    latitude_2 NUMERIC,
    longitude_2 NUMERIC,
    altitude_2 NUMERIC
);
CREATE TABLE
```

**Install "yb-voyager" **
```
[yc-user@yuga1 ~]$ sudo yum install https://s3.us-west-2.amazonaws.com/downloads.yugabyte.com/repos/reporpms/yb-yum-repo-1.1-0.noarch.rpm
[yc-user@yuga1 ~]$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
[yc-user@yuga1 ~]$ sudo yum install oracle-instant-clients-repo
[yc-user@yuga1 ~]$ sudo yum --disablerepo=* -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[yc-user@yuga1 ~]$ sudo dnf -qy module disable postgresql
[yc-user@yuga1 ~]$ sudo yum install perl-open.noarch
[yc-user@yuga1 ~]$ sudo yum update
[yc-user@yuga1 ~]$ sudo yum install yb-voyager
```
**Version yb-voyager**
```
[yc-user@yuga1 ~]$ yb-voyager version
GIT_COMMIT_HASH=90233738b503df7497865ba37a52d0bb522780ce
VERSION=1.8.8
```
**Download dataset opensky**
```
[yc-user@yuga1 ~]$ yb-voyager import data file --export-dir ./tmp/ --target-db-host 10.95.111.37 --target-db-user yugabyte --target-db-password password --target-db-name opensky --data-dir ./csv/  --file-table-map 'flightlist_*.csv:opensky' --format csv --has-header true
GIT_COMMIT_HASH=90233738b503df7497865ba37a52d0bb522780ce
migrationID: 676a68d6-6c1c-414c-b788-7f4625cd27b5
Using 1-12 parallel jobs (adaptive)
yugabytedb version: 11.2-YB-2024.2.0.0-b0

import of data in "opensky" database started
Already imported tables: []
Tables to import: [public."opensky"]
snapshot data import complete

import report

SCHEMA	TABLE  	IMPORTED ROW COUNT
public	opensky	66010794          


Import data complete.

2025-01-07 13:16:37.783377 INFO common.go:80 Time taken: 1h16m54.995359417s (4615.00 seconds)
```

**verification of uploading**
```
[yc-user@yuga1 ~]$ yb-voyager import data status --export-dir ./tmp/
Import Data Status for TargetDB

TABLE           	FILE                            	STATUS	TOTAL SIZE	IMPORTED SIZE	PERCENTAGE
public."opensky"	flightlist_20190101_20190131.csv	DONE  	406.98 MiB	406.98 MiB   	100.00    
public."opensky"	flightlist_20190201_20190228.csv	DONE  	380.65 MiB	380.65 MiB   	100.00    
public."opensky"	flightlist_20190301_20190331.csv	DONE  	433.30 MiB	433.30 MiB   	100.00    
public."opensky"	flightlist_20190401_20190430.csv	DONE  	451.23 MiB	451.23 MiB   	100.00    
public."opensky"	flightlist_20190501_20190531.csv	DONE  	482.62 MiB	482.62 MiB   	100.00    
public."opensky"	flightlist_20190601_20190630.csv	DONE  	505.61 MiB	505.61 MiB   	100.00    
public."opensky"	flightlist_20190701_20190731.csv	DONE  	551.30 MiB	551.30 MiB   	100.00    
public."opensky"	flightlist_20190801_20190831.csv	DONE  	569.18 MiB	569.18 MiB   	100.00    
public."opensky"	flightlist_20190901_20190930.csv	DONE  	518.25 MiB	518.25 MiB   	100.00    
public."opensky"	flightlist_20191001_20191031.csv	DONE  	561.07 MiB	561.07 MiB   	100.00    
public."opensky"	flightlist_20191101_20191130.csv	DONE  	518.16 MiB	518.16 MiB   	100.00    
public."opensky"	flightlist_20191201_20191231.csv	DONE  	514.75 MiB	514.75 MiB   	100.00    
public."opensky"	flightlist_20200101_20200131.csv	DONE  	524.80 MiB	524.80 MiB   	100.00    
public."opensky"	flightlist_20200201_20200229.csv	DONE  	505.43 MiB	505.43 MiB   	100.00    
public."opensky"	flightlist_20200301_20200331.csv	DONE  	411.55 MiB	411.55 MiB   	100.00    
public."opensky"	flightlist_20200401_20200430.csv	DONE  	160.65 MiB	160.65 MiB   	100.00    
public."opensky"	flightlist_20200501_20200531.csv	DONE  	206.77 MiB	206.77 MiB   	100.00    
public."opensky"	flightlist_20200601_20200630.csv	DONE  	274.34 MiB	274.34 MiB   	100.00    
public."opensky"	flightlist_20200701_20200731.csv	DONE  	364.64 MiB	364.64 MiB   	100.00    
public."opensky"	flightlist_20200801_20200831.csv	DONE  	391.11 MiB	391.11 MiB   	100.00    
public."opensky"	flightlist_20200901_20200930.csv	DONE  	369.84 MiB	369.84 MiB   	100.00    
public."opensky"	flightlist_20201001_20201031.csv	DONE  	377.55 MiB	377.55 MiB   	100.00    
public."opensky"	flightlist_20201101_20201130.csv	DONE  	346.79 MiB	346.79 MiB   	100.00    
public."opensky"	flightlist_20201201_20201231.csv	DONE  	360.72 MiB	360.72 MiB   	100.00    
public."opensky"	flightlist_20210101_20210131.csv	DONE  	338.42 MiB	338.42 MiB   	100.00    
public."opensky"	flightlist_20210201_20210228.csv	DONE  	306.84 MiB	306.84 MiB   	100.00    
public."opensky"	flightlist_20210301_20210331.csv	DONE  	394.18 MiB	394.18 MiB   	100.00    
public."opensky"	flightlist_20210401_20210430.csv	DONE  	421.99 MiB	421.99 MiB   	100.00    
public."opensky"	flightlist_20210501_20210530.csv	DONE  	431.75 MiB	431.75 MiB   	100.00    
public."opensky"	flightlist_20210601_20210630.csv	DONE  	478.05 MiB	478.05 MiB   	100.00  
```

**Checking the size of the data directory**
```
[root@test2 hw-11]# for ii in 158.160.49.37 158.160.50.55 51.250.13.59 89.169.150.171 130.193.51.102 89.169.142.204 ; do ssh yc-user@${ii} sudo du -sh /data ; done 
Warning: Permanently added '158.160.49.37' (ED25519) to the list of known hosts.
6.0G	/data
Warning: Permanently added '158.160.50.55' (ED25519) to the list of known hosts.
6.0G	/data
Warning: Permanently added '51.250.13.59' (ED25519) to the list of known hosts.
6.0G	/data
Warning: Permanently added '89.169.150.171' (ED25519) to the list of known hosts.
6.3G	/data
Warning: Permanently added '130.193.51.102' (ED25519) to the list of known hosts.
6.1G	/data
Warning: Permanently added '89.169.142.204' (ED25519) to the list of known hosts.
6.0G	/data
```
**list all tablet_servers**
```
[yc-user@yuga1 ~]$ /opt/yugabyte/bin/yb-admin --master_addresses 10.95.111.37:7100,10.95.111.28:7100,10.95.111.31:7100 list_all_tablet_servers
Tablet Server UUID               RPC Host/Port Heartbeat delay Status   Reads/s  Writes/s Uptime   SST total size  SST uncomp size SST #files      Memory   Broadcast Host/Port 
1f5e35bc21634153ac9fd2fca70cbe2b yuga3.ru-central1.internal:9100 0.08s           ALIVE    0.00     0.00     5796     5.45 GB         7.95 GB         15              2.40 GB  N/A
9990e7f99c7a4d99b9d18376f349e370 yuga5.ru-central1.internal:9100 0.87s           ALIVE    0.00     0.00     5802     5.44 GB         7.93 GB         16              2.15 GB  N/A
9846f830a9f644249f6df337be0482eb yuga6.ru-central1.internal:9100 0.69s           ALIVE    0.00     0.00     5796     5.41 GB         7.93 GB         17              1.99 GB  N/A
c5c49c56daa9463da057374c27910616 yuga2.ru-central1.internal:9100 0.53s           ALIVE    0.00     0.00     5793     5.41 GB         7.93 GB         16              2.14 GB  N/A
4b9bd70ece114bb8b6b8676367ee733c yuga4.ru-central1.internal:9100 0.29s           ALIVE    0.00     0.00     5797     5.44 GB         7.95 GB         16              1.77 GB  N/A
86bad98858bc45b0a298777e06d3b597 yuga1.ru-central1.internal:9100 0.08s           ALIVE    0.00     0.00     5910     5.45 GB         7.95 GB         12              2.13 GB  N/A
```



###**Running analytical queries in YugabyteDB**
- **Total number of flights**
```
opensky=# select count(*) from opensky;
 66010794

Time: 16467.666 ms (00:16.468)
```
- **Number of flights "callsign IN ('UUEE', 'UUDD', 'UUWW')"**
```
opensky=# SELECT COUNT(*) FROM opensky WHERE callsign IN ('UUEE', 'UUDD', 'UUWW');
    14

Time: 77963.198 ms (01:17.963)
```
**Top 10 airports with max number of flights**
```
opensky=# SELECT origin, COUNT(*) AS c FROM opensky WHERE origin != '' GROUP BY origin ORDER BY c DESC limit 10;
 KORD   | 745007
 KDFW   | 696702
 KATL   | 667286
 KDEN   | 582708
 KLAX   | 581952
 KLAS   | 447789
 KPHX   | 428558
 KSEA   | 412592
 KCLT   | 404612
 VIDP   | 363073

Time: 45007.615 ms (00:45.008)
```

###**Testing YugabyteDB performance using pgbench**
**Create a test DB pgbench**
```
[root@test2 hw-11]# pgbench -h 127.0.0.1 -p 5433 -U yugabyte  -s 100 -i pgbench
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
WARNING:  storage parameter fillfactor is unsupported, ignoring
WARNING:  storage parameter fillfactor is unsupported, ignoring
WARNING:  storage parameter fillfactor is unsupported, ignoring
generating data (client-side)...
WARNING:  Batched COPY is not supported in transaction blocks. Defaulting to using one transaction for the entire copy.
HINT:  Either run this COPY outside of a transaction block or set rows_per_transaction option to `0` to disable batching and remove this warning.
10000000 of 10000000 tuples (100%) done (elapsed 290.49 s, remaining 0.00 s)
vacuuming...
NOTICE:  VACUUM is a no-op statement since YugabyteDB performs garbage collection of dead tuples automatically
NOTICE:  VACUUM is a no-op statement since YugabyteDB performs garbage collection of dead tuples automatically
NOTICE:  VACUUM is a no-op statement since YugabyteDB performs garbage collection of dead tuples automatically
NOTICE:  VACUUM is a no-op statement since YugabyteDB performs garbage collection of dead tuples automatically
creating primary keys...
NOTICE:  table rewrite may lead to inconsistencies
DETAIL:  Concurrent DMLs may not be reflected in the new table.
HINT:  See https://github.com/yugabyte/yugabyte-db/issues/19860. Set 'ysql_suppress_unsafe_alter_notice' yb-tserver gflag to true to suppress this notice.
NOTICE:  table rewrite may lead to inconsistencies
DETAIL:  Concurrent DMLs may not be reflected in the new table.
HINT:  See https://github.com/yugabyte/yugabyte-db/issues/19860. Set 'ysql_suppress_unsafe_alter_notice' yb-tserver gflag to true to suppress this notice.
NOTICE:  table rewrite may lead to inconsistencies
DETAIL:  Concurrent DMLs may not be reflected in the new table.
HINT:  See https://github.com/yugabyte/yugabyte-db/issues/19860. Set 'ysql_suppress_unsafe_alter_notice' yb-tserver gflag to true to suppress this notice.
done in 1023.79 s (drop tables 0.08 s, create tables 2.63 s, client-side generate 357.95 s, vacuum 22.42 s, primary keys 640.71 s).
```
**pgbench select-only**
```
[root@test2 hw-11]# pgbench -h 127.0.0.1 -p 5433 -U yugabyte -c 50 -j 4 -T 120 -r --select-only --no-vacuum pgbench
pgbench (14.3, server 11.2-YB-2024.2.0.0-b0)
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 50
number of threads: 4
duration: 120 s
number of transactions actually processed: 144220
latency average = 40.917 ms
initial connection time = 2009.740 ms
tps = 1221.977158 (without initial connection time)
statement latencies in milliseconds:
         0.003  \set aid random(1, 100000 * :scale)
        41.047  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```
### **Install single Postgres instance**

**install postgresql16-server**
```
[root@yuga1 yc-user]# dnf update -y
[root@yuga1 yc-user]# dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@yuga1 yc-user]# dnf -qy module disable postgresql
[root@yuga1 yc-user]# dnf install postgresql16-server postgresql16-contrib
```

**initdb and start service**
```
[root@yuga1 yc-user]# /usr/pgsql-16/bin/postgresql-16-setup initdb
[root@yuga1 yc-user]# systemctl enable postgresql-16
[root@yuga1 yc-user]# systemctl start postgresql-16
[root@yuga1 yc-user]# systemctl status postgresql-16
● postgresql-16.service - PostgreSQL 16 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-16.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2025-01-07 19:17:38 UTC; 5s ago
     Docs: https://www.postgresql.org/docs/16/static/
  Process: 10985 ExecStartPre=/usr/pgsql-16/bin/postgresql-16-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 10991 (postgres)
    Tasks: 7 (limit: 48754)
   Memory: 17.7M
   CGroup: /system.slice/postgresql-16.service
           ├─10991 /usr/pgsql-16/bin/postgres -D /var/lib/pgsql/16/data/
           ├─10992 postgres: logger 
           ├─10993 postgres: checkpointer 
           ├─10994 postgres: background writer 
           ├─10996 postgres: walwriter 
           ├─10997 postgres: autovacuum launcher 
           └─10998 postgres: logical replication launcher 
```

**Tuned**
```

# Memory Configuration
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 41MB
maintenance_work_mem = 410MB

# Checkpoint Related Configuration
min_wal_size = 2GB
max_wal_size = 3GB
checkpoint_completion_target = 0.9
wal_buffers = -1

# Network Related Configuration
listen_addresses = '*'
max_connections = 100

# Storage Configuration
random_page_cost = 1.1
effective_io_concurrency = 200

# Worker Processes Configuration
max_worker_processes = 8
max_parallel_workers_per_gather = 2
max_parallel_workers = 2

```
### **Upload dataset opensky**
**Create DB, Table :**
```

postgres=# CREATE DATABASE opensky;
CREATE DATABASE
postgres=# \c opensky 
You are now connected to database "opensky" as user "postgres".
opensky=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description          
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)

opensky=# CREATE TABLE opensky
(
    callsign TEXT,
    number TEXT,
    icao24 TEXT,
    registration TEXT,
    typecode TEXT,
    origin TEXT,
    destination TEXT NULL,
    firstseen TIMESTAMP WITH TIME ZONE NOT NULL,
    lastseen TIMESTAMP WITH TIME ZONE NOT NULL,
    day TIMESTAMP WITH TIME ZONE NOT NULL,
    latitude_1 NUMERIC,
    longitude_1 NUMERIC,
    altitude_1 NUMERIC,
    latitude_2 NUMERIC,
    longitude_2 NUMERIC,
    altitude_2 NUMERIC
);
```

**Load the data into opensky table**
```
[postgres@yuga1 csv]$ for ii in flightlist_2019* flightlist_2020* flightlist_2021* ; do echo ${ii} ; cat ${ii} | /usr/pgsql-16/bin/psql -d opensky -U postgres -c "COPY opensky from stdin with delimiter ',' CSV HEADER" ; done
flightlist_20190101_20190131.csv
COPY 2145469
flightlist_20190201_20190228.csv
COPY 2005958
flightlist_20190301_20190331.csv
COPY 2283154
flightlist_20190401_20190430.csv
COPY 2375102
flightlist_20190501_20190531.csv
COPY 2539167
flightlist_20190601_20190630.csv
COPY 2660901
flightlist_20190701_20190731.csv
COPY 2898415
flightlist_20190801_20190831.csv
COPY 2990061
flightlist_20190901_20190930.csv
COPY 2721743
flightlist_20191001_20191031.csv
COPY 2946779
flightlist_20191101_20191130.csv
COPY 2721437
flightlist_20191201_20191231.csv
COPY 2701295
flightlist_20200101_20200131.csv
COPY 2734791
flightlist_20200201_20200229.csv
COPY 2648835
flightlist_20200301_20200331.csv
COPY 2152157
flightlist_20200401_20200430.csv
COPY 842905
flightlist_20200501_20200531.csv
COPY 1088267
flightlist_20200601_20200630.csv
COPY 1444224
flightlist_20200701_20200731.csv
COPY 1905528
flightlist_20200801_20200831.csv
COPY 2042040
flightlist_20200901_20200930.csv
COPY 1930868
flightlist_20201001_20201031.csv
COPY 1985145
flightlist_20201101_20201130.csv
COPY 1825015
flightlist_20201201_20201231.csv
COPY 1894751
flightlist_20210101_20210131.csv
COPY 1783384
flightlist_20210201_20210228.csv
COPY 1617845
flightlist_20210301_20210331.csv
COPY 2079436
flightlist_20210401_20210430.csv
COPY 2227362
flightlist_20210501_20210530.csv
COPY 2278298
flightlist_20210601_20210630.csv
COPY 2540487
```
**pg_size_pretty**
```
opensky=# select pg_size_pretty(pg_database_size('opensky'));
 pg_size_pretty 
----------------
 10 GB
(1 row)
```

### **Running analytical queries in Postgres**

- **Total number of flights**
```
opensky=# \timing on
Timing is on.
opensky=# SELECT COUNT(*) FROM opensky;
  count   
----------
 66010819
(1 row)

Time: 289492.005 ms (04:49.492)
```
- **Number of flights "callsign IN ('UUEE', 'UUDD', 'UUWW')"**
```
opensky=# SELECT COUNT(*) FROM opensky WHERE callsign IN ('UUEE', 'UUDD', 'UUWW');
 count 
-------
    14
(1 row)

Time: 286825.770 ms (04:46.826)
```
- **Тop 10 airports with max number of flights**
```
opensky=# SELECT origin, COUNT(*) AS c FROM opensky WHERE origin != '' GROUP BY origin ORDER BY c DESC limit 10;
 origin |    c     
--------+----------
 KORD   |   745007
 KDFW   |   696702
 KATL   |   667286
 KDEN   |   582709
 KLAX   |   581952
 KLAS   |   447789
 KPHX   |   428558
 KSEA   |   412592
 KCLT   |   404612
 VIDP   |   363073
(10 rows)

Time: 288593.289 ms (04:48.593)
```

### **Testing Postgres performance using pgbench**
**create DB pgbench**
```
opensky=# create database pgbench;
CREATE DATABASE
Time: 79.227 ms
opensky=# \q

[postgres@yuga1 ~]$ /usr/pgsql-16/bin/pgbench -s 100 -i pgbench
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 50.55 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 91.19 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 59.80 s, vacuum 2.43 s, primary keys 28.94 s).
```
**Start pgbench**
```
[postgres@yuga1 ~]$ /usr/pgsql-16/bin/pgbench -c 50 -j 4 -r -T 120 pgbench
pgbench (16.6)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 50
number of threads: 4
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 305742
number of failed transactions: 0 (0.000%)
latency average = 19.657 ms
initial connection time = 62.886 ms
tps = 2543.571797 (without initial connection time)
statement latencies in milliseconds and failures:
         0.001           0  \set aid random(1, 100000 * :scale)
         0.000           0  \set bid random(1, 1 * :scale)
         0.000           0  \set tid random(1, 10 * :scale)
         0.000           0  \set delta random(-5000, 5000)
         0.744           0  BEGIN;
         1.037           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.690           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         1.030           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         3.779           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.585           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
        11.716           0  END;
```
**Start pgbench select-only**
```
[postgres@yuga1 ~]$ /usr/pgsql-16/bin/pgbench -c 50 -j 4 -r -T 120 --select-only pgbench
pgbench (16.6)
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 50
number of threads: 4
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 3921733
number of failed transactions: 0 (0.000%)
latency average = 1.531 ms
initial connection time = 77.431 ms
tps = 32665.409610 (without initial connection time)
statement latencies in milliseconds and failures:
         0.000           0  \set aid random(1, 100000 * :scale)
         1.472           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

## **Postgres VS YugabyteDB**

|                                                          |**Postgres**|**YugabyteDB**|
|----------------------------------------------------------|------------|--------------|
|**BENCH_RW** number of transactions actually processed    | 305742     |  -           |
|**BENCH_RW** latency average                              | 19.657 ms  |  -           |
|**BENCH_RW** initial connection time                      | 62.886 ms  |  -           |
|**BENCH_RW** tps (without initial connection time)        | 2543       |  -           |
|                                                          |            |              |
|**BENCH_RO** number of transactions actually processed    | 3921733    | 144220       |
|**BENCH_RO** latency average                              | 1.531 ms   | 40.917 ms    |
|**BENCH_RO** initial connection time                      | 77.431 ms  | 2009.740 ms  |
|**BENCH_RO** tps (without initial connection time)        | 32665      | 1222         |
|                                                          |            |              |
| Total number of flights                                  | 289492 ms  | 16467 ms     |
| number of flights "callsign IN ('UUEE', 'UUDD', 'UUWW')" | 286825 ms  | 77963 ms     |
| Тop 10 airports with max number of flights               | 288593 ms  | 45007 ms     |


**Result:**
- OLTP Postgres win  
- OLAP YugabyteDB win
