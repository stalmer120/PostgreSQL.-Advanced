# pgbackrest + postgres (wal archive + full + diff backup)
Имеем
| Назначение | Имя |
| ---- | ---- |
| postgresql | srv-postgres |
| pgbackrest | srv-pgbackup |
## Устанавливаем pgbackrest на кластер и на ВМ для backup. Версии дожны совпадать до минорной
```
apt update
apt-cache madison pgbackrest
apt install pgbackrest
pgbackrest version
```
# Проверяем настройки, вносим изменения  
```
vi /etc/postgresql/14/main/postgresql.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=otus archive-push %p'
restore_command = 'pgbackrest --stanza=otus archive-get %f \"%p\"'
```
# Настраиваем SSH между srv-postgres и srv-pgbackup
обмениваемся открытыми ключами в authorized_keys и проверяем доступность.
# Вносим изменения в cat /etc/pgbackrest.conf на srv-postgres
```
echo "
[otus]
pg1-path=/pg_data/data
pg1-socket-path=/var/run/postgresql

[global]
repo1-host=srv-pgbackup
">/etc/pgbackrest.conf
```
# Вносим глобальные изменения в vi /etc/pgbackrest.conf на srv-pgbackup
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
* process-max=2: Максимальное количество процессов, которые могут быть запущены одновременно для выполнения резервного копирования или восстановления.  
* repo1-path=/pgbackrest: Путь к первому репозиторию, где будут храниться резервные копии.  
* repo1-retention-full-type=time: Тип хранения полных резервных копий. В данном случае используется временной тип.  
* repo1-retention-full=15: Хранить полные резервные копии в течение 15 дней.  
* repo1-retention-diff=7: Хранить дифференциальные резервные копии в течение 7 дней.  
* start-fast=y: Указывает, что резервное копирование должно начинаться быстро, без ожидания завершения предыдущих операций.  
* log-level-file=detail: Уровень детализации логирования для файлового журнала. "detail" означает, что будет записываться подробная информация о процессе.  
* config-include-path=/etc/pgbackrest/conf.d: Путь к дополнительным конфигурационным файлам, которые могут быть включены в основную конфигурацию.  
* compress-level=3: Уровень сжатия для архивируемых данных. Значение 3 указывает на средний уровень сжатия, что может помочь уменьшить объем хранимых данных, сохраняя при этом разумное время обработки.
# создаем файл с настройками для конкретного postgres на srv-pgbackup
```
touch /etc/pgbackrest/conf.d/otus.conf

echo "
[otus]
pg1-host=srv-postgres
pg1-path=/pg_data/data
pg1-socket-path=/var/run/postgresql
">/etc/pgbackrest/conf.d/otus.conf
```
# Создаем расписание в crontab.
```
#[otus] stanza
0 23 * * 0 sudo -u pgbackrest pgbackrest --stanza=otus --type=full backup
0 23 * * 1-6 sudo -u pgbackrest pgbackrest --stanza=otus --type=diff backup
```
# создание stanza, проверка, первый полный бэкап.
```
sudo -u pgbackrest pgbackrest --stanza=otus stanza-create
sudo -u pgbackrest pgbackrest --stanza=otus --log-level-console=info check
sudo -u pgbackrest pgbackrest --stanza=otus --type=full backup
```
## Примеры
sudo -u pgbackrest pgbackrest --stanza=otus --log-level-console=info check
```
2024-12-22 19:39:19.317 P00   INFO: check command begin 2.49:
 --exec-id=178376-68b9503f
 --log-level-console=info
 --log-level-file=detail
 --pg1-host=otus --pg1-path=/pg_data/data 
 --pg1-socket-path=/var/run/postgresql 
 --repo1-path=/pgbackrest 
 --stanza=otus
2024-12-22 19:39:22.722 P00   INFO: check repo1 (standby)
2024-12-22 19:39:22.723 P00   INFO: switch wal not performed because this is a standby
2024-12-22 19:39:22.724 P00   INFO: check repo1 configuration (primary)
2024-12-22 19:39:22.925 P00   INFO: check repo1 archive for WAL (primary)
2024-12-22 19:39:24.028 P00   INFO: WAL segment 000000010000000000000009 successfully archived to '/pgbackrest/archive/otus/14-1/0000000100000000/000000010000000000000009-2a1d0fb9db071732c1d60adef024907742adf7db.gz' on repo1
2024-12-22 19:39:24.329 P00   INFO: check command end: completed successfully (5013ms)
```
sudo -u pgbackrest pgbackrest --stanza=otus info
```
stanza: otus
    status: ok
    cipher: none    db (current)
        wal archive min/max (14): 000000010000000000000001/00000001000000000000000D
        full backup: 20241221-230001F
            timestamp start/stop: 2024-12-21 23:00:01+03 / 2024-12-21 23:00:07+03
            wal start/stop: 000000010000000000000006 / 000000010000000000000007
            database size: 26.2MB, database backup size: 26.2MB
            repo1: backup set size: 3.3MB, backup size: 3.3MB
```
