# Подключаем диск
lsblk  
pvdisplay Инициализация  
pvcreate /dev/sdb  
vgdisplay Создание групп томов  
vgcreate pg_data_vg /dev/sdb  
lvdisplay Создание логических томов  
lvcreate -l 100%FREE -n pg_data_lv pg_data_vg  
mkfs.xfs /dev/mapper/pg_data_vg-pg_data_lv  

Запись в fstab  
echo "/dev/mapper/pg_data_vg-pg_data_lv /pg_data xfs defaults 0 0" >> /etc/fstab  
Монтируем раздел
mount /dev/mapper/pg_data_vg-pg_data_lv /pg_data  
mkdir /pg_data/data  
chown -R postgres:postgres /pg_data/  
chmod -R 700 /pg_data/  

# rsync
systemctl stop postgresql.service && systemctl status postgresql.service  

	#14  
		rsync -arv /var/lib/postgresql/14/main/ /pg_data/data/  
	#15  
		rsync -arv /var/lib/postgresql/15/main/ /pg_data/data/  
		
# edit postgresql.conf
	#14	 
		vi /etc/postgresql/14/main/postgresql.conf  
	#15	 
		vi /etc/postgresql/15/main/postgresql.conf  

Изменяем/добавляем в конец конфига  
```
data_directory = '/pg_data/data'    # use data in another directory
```
# запускаем проверяем
```
systemctl start postgresql.service && systemctl status postgresql.service
```

```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
14  main    5432 online postgres /pg_data/data  /var/log/postgresql/postgresql-14-main.log

```
