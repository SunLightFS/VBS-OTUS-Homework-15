# ДЗ: Multi master

## Вариант №1: Развернуть CockroachDB в GKE или GCE

Для выполнения данного варианта развернул в GKE три инстанса:
- instance-1 (10.128.0.6)
- instance-2 (10.128.0.7)
- instance-3 (10.128.0.8)

На каждом из инстансов развернул cockroach:
```
wget -qO- https://binaries.cockroachdb.com/cockroach-v21.1.6.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v21.1.6.linux-amd64/cockroach /usr/local/bin/ && sudo mkdir -p /opt/cockroach
mkdir -p /usr/local/lib/cockroach
cp -i cockroach-v21.1.6.linux-amd64/lib/libgeos.so /usr/local/lib/cockroach/
cp -i cockroach-v21.1.6.linux-amd64/lib/libgeos_c.so /usr/local/lib/cockroach/
cp -i cockroach-v21.1.6.linux-amd64/cockroach /usr/local/sbin/

cockroach start --insecure --advertise-addr=10.128.0.6 --join=10.128.0.6,10.128.0.7,10.128.0.8 --cache=.25 --max-sql-memory=.25 --background
```
Далее только на instance-1:
```
cockroach init --insecure --host=10.128.0.6
```

Далее пришлось помучиться с загрузкой датасета (очень долго разбирался, как в Cockroach нормально загрузить данные), но по итогу пришел к следующим действиям:
1) Создаем БД и таблицу, в которую будем заливать данные. Тут пришлось немного схитрить и поменять типы у части полей таблицы на varchar, т.к. cockroach ругался, если я пытался загрузить пустое значение в поле типа bigint (почти уверен, что это можно как-то адекватно обойти)
```
create database otus;
\c otus
create table taxi_trips (
    unique_key text, 
    taxi_id text, 
    trip_start_timestamp TIMESTAMP, 
    trip_end_timestamp varchar, 
    trip_seconds varchar, 
    trip_miles varchar, 
    pickup_census_tract varchar, 
    dropoff_census_tract varchar, 
    pickup_community_area varchar, 
    dropoff_community_area varchar, 
    fare varchar, 
    tips varchar, 
    tolls varchar, 
    extras varchar, 
    trip_total varchar, 
    payment_type text, 
    company text, 
    pickup_latitude varchar, 
    pickup_longitude varchar, 
    pickup_location text, 
    dropoff_latitude varchar, 
    dropoff_longitude varchar, 
    dropoff_location text
);
```
2) Разворачиваем отдельную ВМ с PostgreSQL 13 (все равно понадобится для выполнения этого варианта)  
Тут развернул instance-4 с ip 10.128.0.9. Дальнейшие шаги выполняются на нем
3) Скачиваем данные из Cloud Storage (только первые 40 файлов датасета, т.к. их вес в сумме примерно 10 Гб) python-скриптом:
```
# coding=utf-8

import requests
import os

i = 0
j = 0
res = []
for i in range(0,4):
    for j in range(0,10):
        num = str(i) + str(j)
        res.append(num)

print(res)

for el in res:
    URL = 'gs://hw-taxi/taxi0000000000' + el + '.csv'
    os.system('gsutil -m cp %s /tmp/taxi' % URL)
```
4) В отдельном окне включаем базовый питоновский http-сервер, чтобы расшарить скачаные csv:
```
cd /tmp/taxi
python3 -m http.server
```
5) Для импорта csv-файлов использовал следующий python-скрипт:
```
import os


i = 0
j = 0
res = []
for i in range(0,4):
    for j in range(0,10):
        num = str(i) + str(j)
        query = "import into taxi_trips(unique_key, \
            taxi_id, \
            trip_start_timestamp, \
            trip_end_timestamp, \
            trip_seconds, \
            trip_miles, \
            pickup_census_tract, \
            dropoff_census_tract, \
            pickup_community_area, \
            dropoff_community_area, \
            fare, \
            tips, \
            tolls, \
            extras, \
            trip_total, \
            payment_type, \
            company, \
            pickup_latitude, \
            pickup_longitude, \
            pickup_location, \
            dropoff_latitude, \
            dropoff_longitude, \
            dropoff_location) \
            CSV DATA ('http://10.128.0.9:8000/taxi0000000000%s.csv') WITH DELIMITER = ',', SKIP = '1';" % num

        os.system('sudo -u postgres psql postgres://root@10.128.0.6:26257/otus -c "' + query + '"')
```

Далее я загрузил тот же набор данных в БД на instance-4:
```
su - postgres

for f in /tmp/taxi/taxi*;
do
    echo -e "Processing $f file...";
    psql "host=localhost port=5432 dbname=otus user=postgres" -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER";
done
```

Сравнение длительности выполнения запросов показало, что cockroach выполняет запросы примерно в 2 раза быстрее (хотя я не пробовал оптимизировать настройки PostgreSQL):
1) На сервере с Postgres
```
otus=# explain analyze select count(*) from taxi_trips group by taxi_id;
                                                                          QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------
--------------------------------
 Finalize GroupAggregate  (cost=1585280.78..1586788.46 rows=5951 width=139) (actual time=53240.395..53265.245 rows=8199 loops=
1)
   Group Key: taxi_id
   ->  Gather Merge  (cost=1585280.78..1586669.44 rows=11902 width=139) (actual time=53240.380..53258.440 rows=24144 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=1584280.76..1584295.63 rows=5951 width=139) (actual time=53217.579..53219.448 rows=8048 loops=3)
               Sort Key: taxi_id
               Sort Method: quicksort  Memory: 2335kB
               Worker 0:  Sort Method: quicksort  Memory: 2324kB
               Worker 1:  Sort Method: quicksort  Memory: 2331kB
               ->  Partial HashAggregate  (cost=1583848.15..1583907.66 rows=5951 width=139) (actual time=53179.650..53186.674
rows=8048 loops=3)
                     Group Key: taxi_id
                     Batches: 1  Memory Usage: 2961kB
                     Worker 0:  Batches: 1  Memory Usage: 2961kB
                     Worker 1:  Batches: 1  Memory Usage: 2961kB
                     ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1529553.77 rows=10858877 width=131) (actual time=0.683..
46951.114 rows=8681641 loops=3)
 Planning Time: 11.077 ms
 Execution Time: 53267.975 ms
(18 rows)
```
2) На сервере с Cockroach:
```
root@:26257/otus> explain analyze select count(*) from taxi_trips group by taxi_id;
                                            info
---------------------------------------------------------------------------------------------
  planning time: 529µs
  execution time: 28.3s
  distribution: full
  vectorized: true
  rows read from KV: 26,044,922 (11 GiB)
  cumulative time spent in KV: 47s
  maximum memory usage: 43 MiB
  network usage: 2.2 MiB (54 messages)

  • group
  │ nodes: n1, n2, n3
  │ actual row count: 8,199
  │ estimated row count: 8,198
  │ group by: taxi_id
  │
  └── • scan
        nodes: n1, n2, n3
        actual row count: 26,044,922
        KV rows read: 26,044,922
        KV bytes read: 11 GiB
        estimated row count: 26,044,922 (100% of the table; stats collected 31 minutes ago)
        table: taxi_trips@primary
        spans: FULL SCAN
(23 rows)

Time: 28.353s total (execution 28.352s / network 0.001s)
```
