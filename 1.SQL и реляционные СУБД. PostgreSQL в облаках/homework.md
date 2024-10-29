*Проверим версию.*
> psql --version

```
psql (PostgreSQL) 15.8 (Debian 15.8-1.pgdg100+1)
```
*При необходимости вносим измененния настройки  в pg_hba.conf перечитаем конфиг*  

*Создаем роль с возможностью входа (-l)  и паролем (-P).*  
> createuser -l -P otus

```
                  List of roles
 Role name | Attributes | Member of | Description
-----------+------------+-----------+-------------
 otus      |            | {}        |
```
*Создадим для упрощения одноименную БД с владельцем (-O) otus для простоты подключения.*  
> createdb -O otus otus
```
                                                              List of databases
 Name | Owner | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider | Access privileges |  Size   | Tablespace | Description
------+-------+----------+-------------+-------------+------------+-----------------+-------------------+---------+------------+-------------
 otus | otus  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                   | 7713 kB | pg_default |
```
*Подключаемся*

> psql -U otus

*Проверим правильность подключения.*

> \conninfo

```
You are connected to database "otus" as user "otus" via socket in "/var/run/postgresql" at port "5432".
```
*Создадим одноименную схему.* 
> CREATE SCHEMA IF NOT EXISTS otus AUTHORIZATION otus;

*Проверим создание схемы*

> \dn+
```
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description
--------+----------+----------------------+------------------------
 otus   | otus     |                      |
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         |
(2 rows)
```
*Проверим текущий путь поиска схем.*
> otus=> show search_path ;
```
   search_path
-----------------
 "$user", public
(1 row)
```
*Создадим таблицы.*  
*  **выключить auto commit**  
> \set AUTOCOMMIT OFF

```
otus=> create table persons(id serial, first_name text, second_name text);
CREATE TABLE
otus=*> insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
otus=*> insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
otus=*> commit;
COMMIT
```
*Проверим уровень изоляции.*
> otus=> SHOW transaction_isolation;
```
 transaction_isolation
-----------------------
 read committed
```
*Проверим*
> otus=> \dt+
```
                                  List of relations
 Schema |  Name   | Type  | Owner | Persistence | Access method | Size  | Description
--------+---------+-------+-------+-------------+---------------+-------+-------------
 otus   | persons | table | otus  | permanent   | heap          | 16 kB |
```
> otus=> SELECT * FROM persons;
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
```



*1 Сеанс*  
`insert into persons(first_name, second_name) values('sergey', 'sergeev');`  
*2 Сеанс*  
`select * from persons`  
Новой строки не видно так как транзакция незафиксированна.  
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
*1 Сеанс*  
> commit;  

*2 Сеанс*  
  
> select * from persons;  

Строка видна транзакция зафиксирована.  
В Read Committed снимки данных строятся в начале выполнения каждого оператора SQL. Таким образом, оператор работает с неизменной и согласованной картиной данных, но два одинаковых запроса, следующих один за другим, могут показать разные данные.
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```  

> commit;

*1 Сеанс / 2 Сеанс*

 > set transaction isolation level repeatable read;  

*1 Сеанс*  
>insert into persons(first_name, second_name) values('sveta', 'svetova');  

*2 Сеанс*  
> select * from persons;  
```
otus=*> select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```
Новой строки не видно repeatable read также не допускает фантомное чтение.  
*1 Сеанс*  
> commit;  

*2 Сеанс*  
> select * from persons;  
```
otus=*> select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```
Новой строки не видно в repeatable read снимок строится в начале транзакции(при выполнении первого оператора) — поэтому все запросы в одной транзакции видят одни и те же данные.  
*2 Сеанс*  
> commit;

> select * from persons
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
Строка видна транзакция зафиксирована, на момент начала транзакции в сеансе 2 строка id=4 существует.
