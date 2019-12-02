## Как запустить и использовать

#### Поднять сервис можно командой:

```
docker-compose up otusdb_sergei_baranov
```

#### Остановить:

```
docker-compose stop otusdb_sergei_baranov
```

#### При проблемах вида
"Problem while dropping database.
Can't remove database directory ... Please remove it manually."
и т.п.:

- Открываем терминал в контейнере:

```
docker-compose exec otusdb_sergei_baranov /bin/sh
```

- и в терминале в контейнере:

```
cd /var/lib/mysql
rm -R some_business
```

#### Для подключения к БД используйте команду:

```
docker-compose exec otusdb_sergei_baranov mysql -u root -p12345
```

#### Для использования в клиентских приложениях можно использовать команду:

```
mysql -u root -p12345 --host=127.0.0.1 --port=3309 --protocol=tcp
```

## Sysbench

- Сначала выполняю

```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=mysql --threads=4 --mysql-host=127.0.0.1 --mysql-port=3309 --mysql-user=root --mysql-password=12345 --tables=10 --table-size=50000 --rate=20 --db-ps-mode='disable' prepare
```

- Затем меняю значения переменных в консольном клиенте mysql, и сравниваю результаты от

```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=mysql --threads=4 --mysql-host=127.0.0.1 --mysql-port=3309 --mysql-user=root --mysql-password=12345 --tables=10 --table-size=50000 --rate=20 --db-ps-mode='disable' run
```

- После всех тестов

```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=mysql --threads=4 --mysql-host=127.0.0.1 --mysql-port=3309 --mysql-user=root --mysql-password=12345 --tables=10 --table-size=50000 --rate=20 --db-ps-mode='disable' cleanup
```

Вообще всегда разные от запуска к запуску. Думаю, потому что мало данных и потому что ssd.
NB: прогнать тесты пожёстче, на заполненных данных и запустив докер на hdd


#### innodb_buffer_pool_size;

- SET GLOBAL innodb_buffer_pool_size = 134217728*1;

    transactions:                        192    (18.69 per sec.)
    queries:                             3840   (373.78 per sec.)


- SET GLOBAL innodb_buffer_pool_size = 134217728*64;

    transactions:                        200    (19.45 per sec.)
    queries:                             4000   (389.05 per sec.)

- SET GLOBAL innodb_buffer_pool_size = 134217728*512;

    transactions:                        181    (17.34 per sec.)
    queries:                             3620   (346.88 per sec.)
 
 #### innodb_flush_log_at_trx_commit
 
 - SET GLOBAL innodb_flush_log_at_trx_commit = 1;
 
     transactions:                        193    (19.03 per sec.)
     queries:                             3860   (380.57 per sec.)
 
 - SET GLOBAL innodb_flush_log_at_trx_commit = 0;
 
     transactions:                        208    (20.22 per sec.)
     queries:                             4160   (404.34 per sec.)

 
 - SET GLOBAL innodb_flush_log_at_trx_commit = 2;

    transactions:                        188    (18.49 per sec.)
    queries:                             3760   (369.81 per sec.)

#### sync_binlog

 - SET GLOBAL sync_binlog = 1;
 
     transactions:                        205    (20.46 per sec.)
     queries:                             4100   (409.20 per sec.)

 - SET GLOBAL sync_binlog = 0;
 
     transactions:                        201    (19.56 per sec.)
     queries:                             4020   (391.13 per sec.)

#### innodb_flush_method

- fsync

все тесты выше были при fsync

- O_DIRECT

рестартую с кастомным конфигом с опциями innodb_flush_method=O_DIRECT sync_binlog=0 innodb_flush_log_at_trx_commit=0

    transactions:                        197    (19.51 per sec.)
    queries:                             3940   (390.16 per sec.)
    
#### всё вместе
innodb_buffer_pool_size=2M
innodb_flush_log_at_trx_commit=0
sync_binlog = 0
innodb_flush_method=O_DIRECT
skip_name_resolve=ON
innodb_adaptive_hash_index_parts=24

    transactions:                        227    (21.69 per sec.)
    queries:                             4540   (433.82 per sec.)

тут же второй раз

    transactions:                        200    (19.26 per sec.)
    queries:                             4000   (385.22 per sec.)

раз на раз не приходится

Всё то же при innodb_buffer_pool_size=200M и innodb_buffer_pool_size=2G

### увеличиваю тогда нагрузку теста

```
sysbench /usr/share/sysbench/oltp_read_write.lua
--db-driver=mysql --threads=8 --mysql-host=127.0.0.1 --mysql-port=3309 --mysql-user=root --mysql-password=12345
--tables=40 --table-size=500000 --rate=20 --db-ps-mode='disable' prepare
```

#### закомментировал все кастомные опции

\#innodb_buffer_pool_size=2G
innodb_buffer_pool_size=200M
\#innodb_flush_log_at_trx_commit=0
\#sync_binlog = 0
\#innodb_flush_method=O_DIRECT
\#skip_name_resolve=ON
\#innodb_adaptive_hash_index_parts=24

    transactions:                        218    (20.68 per sec.)
    queries:                             4360   (413.67 per sec.)

#### innodb_buffer_pool_size 2G

innodb_buffer_pool_size=2G
\#innodb_flush_log_at_trx_commit=0
\#sync_binlog = 0
\#innodb_flush_method=O_DIRECT
\#skip_name_resolve=ON
\#innodb_adaptive_hash_index_parts=24

    transactions:                        197    (18.71 per sec.)
    queries:                             3940   (374.27 per sec.)

#### innodb_flush_log_at_trx_commit=0

innodb_buffer_pool_size=2G
innodb_flush_log_at_trx_commit=0
\#sync_binlog = 0
\#innodb_flush_method=O_DIRECT
\#skip_name_resolve=ON
\#innodb_adaptive_hash_index_parts=24

    transactions:                        213    (20.53 per sec.)
    queries:                             4260   (410.70 per sec.)


#### sync_binlog=0

innodb_buffer_pool_size=2G
innodb_flush_log_at_trx_commit=0
sync_binlog=0
\#innodb_flush_method=O_DIRECT
\#skip_name_resolve=ON
\#innodb_adaptive_hash_index_parts=24

    transactions:                        210    (20.31 per sec.)
    queries:                             4200   (406.12 per sec.)

#### all

innodb_buffer_pool_size=2G
innodb_flush_log_at_trx_commit=0
sync_binlog=0
innodb_flush_method=O_DIRECT
skip_name_resolve=ON
sbtest.sbtest12=24

    transactions:                        186    (18.19 per sec.)
    queries:                             3720   (363.88 per sec.)

    transactions:                        203    (19.75 per sec.)
    queries:                             4060   (395.01 per sec.)

    transactions:                        190    (18.62 per sec.)
    queries:                             3800   (372.38 per sec.)

НИЧЕГО ОСОБО НЕ МЕНЯЕТСЯ

### примонтировал HDD как /media/feynman/d2t

```
sysbench /usr/share/sysbench/oltp_read_write.lua
--db-driver=mysql --threads=8 --mysql-host=127.0.0.1 --mysql-port=3309 --mysql-user=root --mysql-password=12345
--tables=10 --table-size=250000 --rate=20 --db-ps-mode='disable' prepare
```

#### innodb_buffer_pool_size=200M

    transactions: 212 (19.98 per sec.)
    queries: 4240 (399.66 per sec.)

#### innodb_buffer_pool_size=2G

    transactions:                        252    (24.49 per sec.)
    queries:                             5040   (489.76 per sec.)

#### sync_binlog=0

default-authentication-plugin=mysql_native_password
innodb_buffer_pool_size=2G
sync_binlog=0

    transactions:                        215    (20.69 per sec.)
    queries:                             4300   (413.84 per sec.)
    
#### all

default-authentication-plugin=mysql_native_password
innodb_buffer_pool_size=2G
innodb_flush_log_at_trx_commit=0
sync_binlog=0
innodb_flush_method=O_DIRECT
innodb_adaptive_hash_index_parts=24

    transactions:                        200    (19.00 per sec.)
    queries:                             4000   (379.92 per sec.)
    
ОПЯТЬ ВСЁ ТАК ЖЕ

