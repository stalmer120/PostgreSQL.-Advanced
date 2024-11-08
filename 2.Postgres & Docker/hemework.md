# Postgres & Docker

**Установлен**  
Docker Desktop для windows
>Docker version 27.2.0, build 3ab4256  
>Docker version 27.2.0, build 3ab4256

**создан docker-compose.yml**
```yml
version: "2.2"
networks:
  db:
services:
  postgres:
    image: postgres:14
    ports: 
      - 6432:5432
    environment:
      - POSTGRES_PASSWORD=P@$$w0rd
    volumes:
      - ~/docpos/postgres:/var/lib/postgresql/data
    networks:
      - db
  grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    networks:
      - db
  prometheus:
    image: bitnami/prometheus
    networks:
      - db
    volumes:
      - ~/docpos/prometheus/prometheus/prometheus.yml:/opt/bitnami/prometheus/conf/prometheus.yml
    ports:
      - 9090:9090
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:v0.10.1
    environment:
      DATA_SOURCE_URL: "postgres:5432/postgres"
      DATA_SOURCE_USER: "postgres"
      DATA_SOURCE_PASS: "P@$$w0rd"
    networks:
      - db
```
Для postgres 14 прописан порт 6432 для внешнего подключения через DBeaver, для prometheus порт 9090. Для volumes созданы папки postgres и prometheus  (prometheus yml сразу помещен в папку)

**prometheus.yml**
```yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090", "postgres_exporter:9187"]
```

> docker-compose up -d

```
CONTAINER ID   IMAGE                                           COMMAND                  CREATED             STATUS             PORTS                    NAMES
7020a1990ab0   postgres:14                                     "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:6432->5432/tcp   stalmer-postgres-1
04c156db78b3   bitnami/prometheus                              "/opt/bitnami/promet…"   2 hours ago         Up 2 hours         0.0.0.0:9090->9090/tcp   stalmer-prometheus-1
906da3d4b05f   grafana/grafana                                 "/run.sh"                2 hours ago         Up 2 hours         0.0.0.0:3000->3000/tcp   stalmer-grafana-1
d7d16f417fbf   prometheuscommunity/postgres-exporter:v0.10.1   "/bin/postgres_expor…"   2 hours ago         Up 2 hours         9187/tcp                 stalmer-postgres_exporter-1
```
Подключаемся
>docker exec -it stalmer-postgres-1 sh

создаем:  
* роль otus
* БД otus
* Схему otus
* таблицу superheroes и наполняем её данными

```
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 otus      |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
```
  List of schemas
  Name  |  Owner
--------+----------
 otus   | otus
 public | postgres
(2 rows)
```
```
  List of schemas
  Name  |  Owner
--------+----------
 otus   | otus
 public | postgres
(2 rows)
```
>select * from superheroes;
```
 id |                name                 |       align        |    eye     |    hair    |      gender       | appearances | year | universe 
----+-------------------------------------+--------------------+------------+------------+-------------------+-------------+------+----------
  1 | Spider-Man (Peter Parker)           | Good Characters    | Hazel Eyes | Brown Hair | Male Characters   |        4043 | 1962 | marvel
  2 | Captain America (Steven Rogers)     | Good Characters    | Blue Eyes  | White Hair | Male Characters   |        3360 | 1941 | marvel
  3 | Wolverine (James \"Logan\" Howlett) | Neutral Characters | Blue Eyes  | Black Hair | Male Characters   |        3061 | 1974 | marvel
  4 | Iron Man (Anthony \"Tony\" Stark)   | Good Characters    | Blue Eyes  | Black Hair | Male Characters   |        2961 | 1963 | marvel
  5 | Thor (Thor Odinson)                 | Good Characters    | Blue Eyes  | Blond Hair | Male Characters   |        2258 | 1950 | marvel
  6 | Benjamin Grimm (Earth-616)          | Good Characters    | Blue Eyes  | No Hair    | Male Characters   |        2255 | 1961 | marvel
  7 | Reed Richards (Earth-616)           | Good Characters    | Brown Eyes | Brown Hair | Male Characters   |        2072 | 1961 | marvel
  8 | Hulk (Robert Bruce Banner)          | Good Characters    | Brown Eyes | Brown Hair | Male Characters   |        2017 | 1962 | marvel
  9 | Scott Summers (Earth-616)           | Neutral Characters | Brown Eyes | Brown Hair | Male Characters   |        1955 | 1963 | marvel
 10 | Jonathan Storm (Earth-616)          | Good Characters    | Blue Eyes  | Blond Hair | Male Characters   |        1934 | 1961 | marvel
 11 | Henry McCoy (Earth-616)             | Good Characters    | Blue Eyes  | Blue Hair  | Male Characters   |        1825 | 1963 | marvel
 12 | Susan Storm (Earth-616)             | Good Characters    | Blue Eyes  | Blond Hair | Female Characters |        1713 | 1961 | marvel
 13 | Ororo Munroe (Earth-616)            | Good Characters    | Blue Eyes  | White Hair | Female Characters |        1512 | 1975 | marvel
 14 | Clinton Barton (Earth-616)          | Good Characters    | Blue Eyes  | Blond Hair | Male Characters   |        1394 | 1964 | marvel
 15 | Matthew Murdock (Earth-616)         | Good Characters    | Blue Eyes  | Red Hair   | Male Characters   |        1338 | 1964 | marvel
 16 | Stephen Strange (Earth-616)         | Good Characters    | Grey Eyes  | Black Hair | Male Characters   |        1307 | 1963 | marvel
 17 | Mary Jane Watson (Earth-616)        | Good Characters    | Green Eyes | Red Hair   | Female Characters |        1304 | 1965 | marvel
 18 | John Jonah Jameson (Earth-616)      | Neutral Characters | Blue Eyes  | Black Hair | Male Characters   |        1266 | 1963 | marvel
(18 rows)
```
**Подключимся через DBeaver на порт 6432 с ролью otus**

![DBeaver](https://github.com/stalmer120/PostgreSQL.-Advanced/blob/main/2.Postgres%20%26%20Docker/image.png)

соединение установлено.

**удалить контейнер с сервером**

>docker rm -f 7020a1990ab0  
>docker rm -f 04c156db78b3  
>docker rm -f 906da3d4b05f  
>docker rm -f d7d16f417fbf  

**создать его заново**
> docker-compose up -d
```
CONTAINER ID   IMAGE                                           COMMAND                  CREATED          STATUS          PORTS                    NAMES
04e4f9cf3b0a   postgres:14                                     "docker-entrypoint.s…"   28 seconds ago   Up 27 seconds   0.0.0.0:6432->5432/tcp   stalmer-postgres-1
637780aa5fe1   grafana/grafana                                 "/run.sh"                28 seconds ago   Up 27 seconds   0.0.0.0:3000->3000/tcp   stalmer-grafana-1
6f64e99762e2   bitnami/prometheus                              "/opt/bitnami/promet…"   28 seconds ago   Up 27 seconds   0.0.0.0:9090->9090/tcp   stalmer-prometheus-1
5ec2af960d85   prometheuscommunity/postgres-exporter:v0.10.1   "/bin/postgres_expor…"   28 seconds ago   Up 27 seconds   9187/tcp                 stalmer-postgres_exporter-1
```
**подключится снова из контейнера с клиентом к контейнеру с сервером**
>docker exec -it stalmer-postgres-1 sh

заходим под ролью otus и попадаем в БД otus
> select * from superheroes limit 5;
```
 id |                name                 |       align        |    eye     |    hair    |     gender      | appearances | year | universe 
----+-------------------------------------+--------------------+------------+------------+-----------------+-------------+------+----------
  1 | Spider-Man (Peter Parker)           | Good Characters    | Hazel Eyes | Brown Hair | Male Characters |        4043 | 1962 | marvel
  2 | Captain America (Steven Rogers)     | Good Characters    | Blue Eyes  | White Hair | Male Characters |        3360 | 1941 | marvel
  3 | Wolverine (James \"Logan\" Howlett) | Neutral Characters | Blue Eyes  | Black Hair | Male Characters |        3061 | 1974 | marvel
  4 | Iron Man (Anthony \"Tony\" Stark)   | Good Characters    | Blue Eyes  | Black Hair | Male Characters |        2961 | 1963 | marvel
  5 | Thor (Thor Odinson)                 | Good Characters    | Blue Eyes  | Blond Hair | Male Characters |        2258 | 1950 | marvel
(5 rows)
```
Все на месте.
