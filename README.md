# Домашнее задание к занятию «Репликация и масштабирование. Часть 2»

## ` Дмитрий Климов `

## Задание 1

Опишите основные преимущества использования масштабирования методами:

активный master-сервер и пассивный репликационный slave-сервер;
master-сервер и несколько slave-серверов;
Дайте ответ в свободной форме.

## Ответ:

В архитектуре “master-slave” главный сервер (master) обрабатывает все операции записи, а подчиненные серверы (slaves) реплицируют данные с master-сервера и обслуживают операции чтения. Рассмотрим преимущества разных вариантов этой архитектуры:

### 1. Активный master-сервер и пассивный репликационный slave-сервер:

Преимущества:
Отказоустойчивость (Failover): Основное преимущество - повышенная отказоустойчивость. Если master-сервер выходит из строя, slave-сервер можно повысить до master-сервера (хотя этот процесс может потребовать ручного вмешательства или автоматизации с помощью инструментов типа Sentinel в Redis). Это позволяет минимизировать время простоя.
Возможность резервного копирования без влияния на основной сервер: Slave-сервер можно использовать для создания резервных копий базы данных, не нагружая master-сервер во время этой ресурсоемкой операции.
Снижение нагрузки на master (в некоторых случаях): Хотя slave не принимает записи, он может принимать часть запросов на чтение. Это может быть полезно, если master перегружен запросами на чтение (но обычно в этом случае используют несколько slaves).

Недостатки:
Сложность управления failover: Переключение на slave требует времени и может быть сопряжено с потерей данных, если репликация не была полностью завершена на момент отказа master. Автоматизация этого процесса сложна и требует дополнительной инфраструктуры.
Использование slave только для резервирования: Slave простаивает большую часть времени, если не используется для чтения или резервного копирования, что может рассматриваться как неэффективное использование ресурсов.
Не масштабирует операции записи: Все записи все равно должны проходить через master-сервер, поэтому этот вариант не решает проблемы масштабирования записи.
Возможная задержка репликации (replication lag): Данные на slave-сервере могут быть не всегда актуальными, что может привести к проблемам консистентности данных при чтении с slave.

### 2. Master-сервер и несколько slave-серверов:

Преимущества:
Масштабирование чтения (Read Scaling): Основное преимущество - возможность масштабирования операций чтения. Запросы на чтение можно распределить между несколькими slave-серверами, что существенно снижает нагрузку на master-сервер и увеличивает пропускную способность всей системы.
Улучшенная отказоустойчивость: Несколько slave-серверов обеспечивают более надежную отказоустойчивость. Если один slave-сервер выходит из строя, другие продолжают обслуживать запросы на чтение.
Размещение slave-серверов в разных географических регионах: Slave-серверы могут быть расположены в разных географических регионах для улучшения производительности и снижения задержек для пользователей, находящихся в этих регионах. (Географическое масштабирование)
Возможность выделения ресурсов для разных типов запросов: Можно направить определенные типы запросов на чтение на определенные slave-серверы, например, сложные аналитические запросы на отдельный slave, чтобы не влиять на производительность других запросов.
Резервное копирование без простоя (Non-blocking backup): Можно делать резервные копии с любого из slave-серверов, не прерывая обслуживание клиентских запросов на чтение.

Недостатки:
Сложность настройки и управления: Несколько slave-серверов требуют более сложной настройки, мониторинга и управления.
Увеличенная стоимость: Поддержание нескольких серверов увеличивает затраты на оборудование, электроэнергию и администрирование.
Проблемы консистентности данных: Как и в случае с одним slave, существует риск расхождения данных между master и slaves, что может привести к проблемам консистентности. Необходимы стратегии для обработки возможной несогласованности данных.
Не масштабирует операции записи: Все записи все равно должны проходить через master-сервер, что может стать узким местом.
Replication Lag (задержка репликации): Задержка в репликации данных с master на slaves остаётся проблемой.

В заключение:
Выбор между этими архитектурами зависит от конкретных требований приложения. Если основная задача - обеспечить отказоустойчивость, то достаточно одного пассивного slave-сервера. Если же необходимо масштабировать операции чтения и обеспечить высокую доступность, то лучше использовать несколько slave-серверов. Для масштабирования операций записи, необходимо рассмотреть другие архитектуры, такие как sharding (шардирование) или использование баз данных, поддерживающих multi-master репликацию. Также, важно помнить о проблеме консистентности данных и разработать стратегию для ее решения, особенно при использовании нескольких slave-серверов.


## Задание 2

Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц:

пользователи,
книги,
магазины (столбцы произвольно).
Опишите принципы построения системы и их разграничение или разбивку между базами данных.

Пришлите блоксхему, где и что будет располагаться. Опишите, в каких режимах будут работать сервера.

## Ответ:


Данный план описывает стратегию горизонтального и вертикального шаринга базы данных, состоящей из трех таблиц: пользователи, книги и магазины.

Цель: Достичь масштабируемости, повышения производительности и отказоустойчивости базы данных путем распределения данных по нескольким серверам.

Принципы:

Минимизация изменений в приложении: Использовать решения, требующие минимальных изменений в коде приложения.
Консистентность данных: Обеспечить согласованность данных между различными шардами.
Прозрачность для пользователей: Минимизировать влияние шаринга на пользовательский опыт.
Стратегия шаринга:

В данном случае целесообразно применить комбинацию горизонтального (шардинга) и вертикального (разделение таблиц) шаринга.

1. Вертикальный шаринг (Разделение таблиц):

Идея: Разделить таблицы по функциональным областям на разные базы данных.
Реализация:
DB_USERS: Содержит таблицу пользователи.
DB_BOOKS: Содержит таблицу книги.
DB_STORES: Содержит таблицу магазины.
Обоснование: Разделение по функционалу позволяет оптимизировать базы данных под специфические нагрузки. Например, база данных DB_USERS может быть оптимизирована для чтения и записи данных пользователей, а база данных DB_BOOKS – для поиска и фильтрации книг.

2. Горизонтальный шаринг (Шардинг):

Идея: Разделить данные каждой таблицы (из DB_USERS, DB_BOOKS, DB_STORES) на несколько шардов (горизонтальных разделов), расположенных на разных серверах.
Реализация:
Shard Key (Ключ шардирования): Необходимо выбрать ключ, на основе которого будут распределяться данные по шардам. Выбор ключа зависит от паттернов доступа к данным и должен обеспечивать равномерное распределение.
пользователи: Можно использовать user_id (если он числовой и последовательный) или, например, hash(user_id) % N (где N – количество шардов).
книги: Можно использовать book_id или publisher_id (если необходимо оптимизировать запросы по издателям).
магазины: Можно использовать store_id или region_id (если необходимо оптимизировать запросы по регионам).
Количество шардов: Зависит от объема данных и требуемой производительности. Важно предусмотреть возможность добавления новых шардов в будущем.
Shard Mapping (Сопоставление шардов): Необходимо определить, какой шард содержит данные для определенного ключа. Это можно реализовать с помощью:
Consistent Hashing: Обеспечивает более плавное добавление и удаление шардов.
Lookup Table: Содержит явное сопоставление ключей и шардов.
Режимы работы серверов:

DB_USERS_Shard_1, DB_USERS_Shard_2, …: Содержат шарды таблицы пользователи. Могут работать в режиме active-active (чтение и запись на все шарды) с репликацией между шадами или в режиме active-passive (один шард для записи, остальные – для чтения). Второй вариант требует более сложной логики переключения между шардами в случае отказа основного.
DB_BOOKS_Shard_1, DB_BOOKS_Shard_2, …: Содержат шарды таблицы книги. Аналогично DB_USERS, могут работать в режиме active-active или active-passive.
DB_STORES_Shard_1, DB_STORES_Shard_2, …: Содержат шарды таблицы магазины. Аналогично DB_USERS и DB_BOOKS, могут работать в режиме active-active или active-passive.
Блок-схема:

```
+---------------------+    +---------------------+    +---------------------+
|  Client Application |    |  Client Application |    |  Client Application |
+---------------------+    +---------------------+    +---------------------+
          |                      |                      |
          v                      v                      v
+-------------------------------------------------------+
|                   Database Router                     |  (Proxy / Middleware)
+-------------------------------------------------------+
          |                      |                      |
          v                      v                      v
+---------------------+ +---------------------+ +---------------------+
|     DB_USERS        | |     DB_BOOKS        | |    DB_STORES        |
+---------------------+ +---------------------+ +---------------------+
          |                      |                      |
+---------+---------+  +---------+---------+  +---------+---------+
| Shard 1 | Shard 2 |  | Shard 1 | Shard 2 |  | Shard 1 | Shard 2 |  ...
+---------+---------+  +---------+---------+  +---------+---------+
   |           |          |             |         |          |
   v           v          v             v         v          v
+---------+---------+  +---------+---------+  +---------+---------+
| DB_USER | DB_USER |  | DB_BOOK | DB_BOOK |  | DB_STORE| DB_STORE|  ...
| SERVER 1| SERVER 2|  | SERVER 1| SERVER 2|  | SERVER 1| SERVER 2|
+---------+---------+  +---------+---------+  +---------+---------+

```
Описание блоков:

Client Application: Приложение, которое взаимодействует с базой данных.
Database Router (Proxy / Middleware): Ключевой компонент, который отвечает за:
Определение, в какую базу данных (DB_USERS, DB_BOOKS, DB_STORES) необходимо направить запрос.
Определение, на какой шард необходимо направить запрос, на основе ключа шардирования.
Агрегацию результатов, если запрос требует доступа к нескольким шардам.
Кэширование.
DB_USERS, DB_BOOKS, DB_STORES: Логические базы данных, каждая из которых содержит соответствующую таблицу.
Shard 1, Shard 2, …: Логические шарды данных, представляющие горизонтальные разделы каждой таблицы.
DB_USER SERVER 1, DB_USER SERVER 2, … DB_BOOK SERVER 1, DB_BOOK SERVER 2, … DB_STORE SERVER 1, DB_STORE SERVER 2…: Физические серверы баз данных, на которых расположены шарды.

Технологии:
Database Router:
ProxySQL: Высокопроизводительный SQL proxy, поддерживающий шардинг и балансировку нагрузки.
HAProxy: Балансировщик нагрузки, который можно использовать для распределения трафика между шардами.
Custom Middleware: Реализация логики шардинга в коде приложения или в отдельном middleware.

База данных:
MySQL: Широко используемая реляционная база данных, поддерживающая шардинг и репликацию.
PostgreSQL: Мощная реляционная база данных с хорошей поддержкой шардинга.
CockroachDB: Распределенная SQL база данных, разработанная для обеспечения масштабируемости и отказоустойчивости.

Преимущества:
Масштабируемость: Возможность добавления новых шардов для увеличения объема хранимых данных и повышения производительности.
Производительность: Параллельная обработка запросов на разных шардах.
Отказоустойчивость: В случае отказа одного шарда, остальные продолжают работать.
Гибкость: Возможность оптимизации каждой базы данных и каждого шарда под специфические нагрузки.

Недостатки:
Сложность реализации: Требует разработки и поддержки Database Router и shard mapping.
Транзакционная целостность: Поддержка транзакций, охватывающих несколько шардов, может быть сложной и требовать дополнительных усилий.
Cross-shard запросы: Запросы, требующие доступа к данным, расположенным на нескольких шардах, могут быть медленными.

Заключение:
Данный план предлагает комплексную стратегию шаринга, сочетающую вертикальное и горизонтальное разделение данных. Выбор конкретных технологий и конфигураций зависит от требований к производительности, масштабируемости и отказоустойчивости системы, а также от доступных ресурсов и экспертизы. Важно тщательно продумать ключ шардирования и shard mapping для обеспечения оптимальной производительности и равномерного распределения данных. Также необходимо учитывать сложности, связанные с транзакциями, охватывающими несколько шардов, и разработать стратегию обработки cross-shard запросов.


## Задание 3*

Выполните настройку выбранных методов шардинга из задания 2.

Пришлите конфиг Docker и SQL скрипт с командами для базы данных.

## Ответ:

### 1. Конфигурация Docker Compose (docker-compose.yml):

```
version: "3.8"

services:
  # ProxySQL
  proxysql:
    image: proxysql/proxysql:2.5
    ports:
      - "6032:6032"  # Admin interface
      - "3306:3306"  # Application port
    volumes:
      - ./proxysql/proxysql.cnf:/etc/proxysql.cnf
    depends_on:
      - mysql_users_shard1
      - mysql_users_shard2
      - mysql_books_shard1
      - mysql_books_shard2
      - mysql_stores_shard1
      - mysql_stores_shard2
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword

  # MySQL - users shard 1
  mysql_users_shard1:
    image: mysql:8.0
    ports:
      - "3307:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_users_shard1
    volumes:
      - ./mysql/users_shard1/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MySQL - users shard 2
  mysql_users_shard2:
    image: mysql:8.0
    ports:
      - "3308:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_users_shard2
    volumes:
      - ./mysql/users_shard2/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MySQL - books shard 1
  mysql_books_shard1:
    image: mysql:8.0
    ports:
      - "3309:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_books_shard1
    volumes:
      - ./mysql/books_shard1/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MySQL - books shard 2
  mysql_books_shard2:
    image: mysql:8.0
    ports:
      - "3310:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_books_shard2
    volumes:
      - ./mysql/books_shard2/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MySQL - stores shard 1
  mysql_stores_shard1:
    image: mysql:8.0
    ports:
      - "3311:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_stores_shard1
    volumes:
      - ./mysql/stores_shard1/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MySQL - stores shard 2
  mysql_stores_shard2:
    image: mysql:8.0
    ports:
      - "3312:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: db_stores_shard2
    volumes:
      - ./mysql/stores_shard2/init.sql:/docker-entrypoint-initdb.d/init.sql
```

Структура каталогов:

Составим структуру этого каталога:

```
.
├── docker-compose.yml
├── mysql
│   ├── books_shard1
│   │   └── init.sql
│   ├── books_shard2
│   │   └── init.sql
│   ├── stores_shard1
│   │   └── init.sql
│   ├── stores_shard2
│   │   └── init.sql
│   ├── users_shard1
│   │   └── init.sql
│   └── users_shard2
│       └── init.sql
└── proxysql
    └── proxysql.cnf
```

### 2. Скрипты инициализации MySQL (файлы init.sql):
(Пример для mysql/users_shard1/init.sql)

```
-- Create the database if it doesn't exist
CREATE DATABASE IF NOT EXISTS db_users_shard1;
USE db_users_shard1;

-- Create the users table
CREATE TABLE IF NOT EXISTS users (
    user_id INT PRIMARY KEY,
    username VARCHAR(255),
    email VARCHAR(255)
);

-- Insert some sample data (user_id % 2 == 0 for shard1)
INSERT INTO users (user_id, username, email) VALUES
(2, 'User2', 'user2@example.com'),
(4, 'User4', 'user4@example.com'),
(6, 'User6', 'user6@example.com');

```

(Пример для mysql/users_shard2/init.sql)

```
-- Create the database if it doesn't exist
CREATE DATABASE IF NOT EXISTS db_users_shard2;
USE db_users_shard2;

-- Create the users table
CREATE TABLE IF NOT EXISTS users (
    user_id INT PRIMARY KEY,
    username VARCHAR(255),
    email VARCHAR(255)
);

-- Insert some sample data (user_id % 2 != 0 for shard2)
INSERT INTO users (user_id, username, email) VALUES
(1, 'User1', 'user1@example.com'),
(3, 'User3', 'user3@example.com'),
(5, 'User5', 'user5@example.com');

```

### 3. Конфигурация ProxySQL (proxysql/proxysql.cnf):

```
[mysql_variables]
mysql-default_charset=utf8mb4
mysql-default_collation=utf8mb4_general_ci
mysql-interfaces = 0.0.0.0:3306
mysql-monitor_username = monitor
mysql-monitor_password = monitorpassword
mysql-monitor_connect_timeout = 1000
mysql-monitor_read_timeout = 2000
mysql-monitor_write_timeout = 2000

[mysql_users]
admin = adminpassword, all
monitor = monitorpassword, monitor

[mysql_servers]
users_shard1 = mysql_users_shard1:3306:10
users_shard2 = mysql_users_shard2:3306:10
books_shard1 = mysql_books_shard1:3306:10
books_shard2 = mysql_books_shard2:3306:10
stores_shard1 = mysql_stores_shard1:3306:10
stores_shard2 = mysql_stores_shard2:3306:10

[mysql_query_rules]
# Users rules
users_shard1_rule = SELECT db_users_shard1 WHERE (dest_host LIKE 'users_shard1') AND (MOD(CAST(substr(SQL, instr(SQL,'user_id')+8, 5) AS UNSIGNED),2) = 0)
users_shard2_rule = SELECT db_users_shard2 WHERE (dest_host LIKE 'users_shard2') AND (MOD(CAST(substr(SQL, instr(SQL,'user_id')+8, 5) AS UNSIGNED),2) = 1)

# Books rules
books_shard1_rule = SELECT db_books_shard1 WHERE (dest_host LIKE 'books_shard1') AND (MOD(CAST(substr(SQL, instr(SQL,'book_id')+8, 5) AS UNSIGNED),2) = 0)
books_shard2_rule = SELECT db_books_shard2 WHERE (dest_host LIKE 'books_shard2') AND (MOD(CAST(substr(SQL, instr(SQL,'book_id')+8, 5) AS UNSIGNED),2) = 1)

# Stores rules
stores_shard1_rule = SELECT db_stores_shard1 WHERE (dest_host LIKE 'stores_shard1') AND (MOD(CAST(substr(SQL, instr(SQL,'store_id')+9, 5) AS UNSIGNED),2) = 0)
stores_shard2_rule = SELECT db_stores_shard2 WHERE (dest_host LIKE 'stores_shard2') AND (MOD(CAST(substr(SQL, instr(SQL,'store_id')+9, 5) AS UNSIGNED),2) = 1)

[scheduler]
# This is a basic example and you will have to write something similar for a production environment
scheduler_default_timeout = 60000

```

### 4. Инициализация ProxySQL с помощью SQL:

После запуска контейнеров подключимся к интерфейсу администратора ProxySQL (порт 6032):

```
docker exec -it proxysql mysql -u admin -padminpassword -h 127.0.0.1 -P 6032

```
Затем выполним следующие команды SQL для настройки ProxySQL:

```
-- Add MySQL servers to ProxySQL
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections) VALUES
(1, 'mysql_users_shard1', 3306, 100, 1000),
(2, 'mysql_users_shard2', 3306, 100, 1000),
(3, 'mysql_books_shard1', 3306, 100, 1000),
(4, 'mysql_books_shard2', 3306, 100, 1000),
(5, 'mysql_stores_shard1', 3306, 100, 1000),
(6, 'mysql_stores_shard2', 3306, 100, 1000);

-- Add hostgroups for routing.  Each "group" will only have one actual server due to the shard logic
-- Hostgroup 1: users_shard1
-- Hostgroup 2: users_shard2
-- Hostgroup 3: books_shard1
-- Hostgroup 4: books_shard2
-- Hostgroup 5: stores_shard1
-- Hostgroup 6: stores_shard2

-- Add rules to route queries to the correct hostgroup
-- These use a very simple shard key based on user_id % 2.
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, dest_hostgroup, apply) VALUES
(1, 1, 'SELECT .* FROM users WHERE user_id=([0-9]+)', 1, 1),
(2, 1, 'SELECT .* FROM books WHERE book_id=([0-9]+)', 3, 1),
(3, 1, 'SELECT .* FROM stores WHERE store_id=([0-9]+)', 5, 1);

INSERT INTO mysql_query_rules (rule_id, active, match_pattern, dest_hostgroup, apply) VALUES
(4, 1, 'SELECT .* FROM users WHERE user_id=([0-9]+)', 2, 1),
(5, 1, 'SELECT .* FROM books WHERE book_id=([0-9]+)', 4, 1),
(6, 1, 'SELECT .* FROM stores WHERE store_id=([0-9]+)', 6, 1);


-- Load configuration and apply it
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;

```

### Пример запроса:

```
SELECT * FROM users WHERE user_id=1;  -- Will be routed to users_shard2
SELECT * FROM books WHERE book_id=2;  -- Will be routed to books_shard1

```













