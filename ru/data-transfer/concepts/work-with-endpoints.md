---
title: "Работа {{ data-transfer-full-name }} с источниками и приемниками"
description: "{{ data-transfer-full-name }} учитывает особенности источников и приемников при подготовке транфсера и во время передачи данных"
---

# Особенности работы с эндпоинтами

Сервис {{ data-transfer-full-name }} имеет ряд ограничений и особенностей функционирования в зависимости от типов эндпоинтов.

## {{ CH }} {#clickhouse}

Трансферы типов {{ dt-type-copy }} и {{ dt-type-copy-repl }} (на стадии копирования) _из {{ CH }} в {{ CH }}_ не поддерживают работу с `VIEW`. В эндпоинте-источнике типа {{ CH }} `VIEW` должны быть включены в "Список исключённых таблиц", если "Список включённых таблиц" пуст или не указан. Если "Список включённых таблиц" непустой, в нём не должно присутствовать объектов `VIEW`.

Источник поддерживает `MATERIALIZED VIEW`, но работает с ними как с обыкновенными таблицами. Таким образом, в трансферах _из {{ CH }} в {{ CH }}_ `MATERIALIZED VIEW` переносятся как таблицы, а не как объекты `MATERIALIZED VIEW`.

Если на приемнике {{ CH }} включена репликация, то движки для воссоздания таблиц будут выбраны в зависимости от типа источника:

* При переносе данных из строковых СУБД будут использоваться движки [ReplicatedReplacingMergeTree]({{ ch.docs }}/engines/table-engines/mergetree-family/replication/) и [ReplacingMergeTree]({{ ch.docs }}/engines/table-engines/mergetree-family/replacingmergetree/).
* При переносе данных из {{ CH }} будут использоваться движки семейства [ReplicatedMergeTree]({{ ch.docs }}/engines/table-engines/mergetree-family/replication/).

## {{ GP }} {#greenplum}

В трансферах _из {{ GP }} в {{ GP }}_ и _из {{ GP }} в {{ PG }}_ схема не переносится в текущей версии {{data-transfer-full-name}}. При наличии пользовательских типов данных в таблицах в таких трансферах, такие пользовательские типы необходимо создать в целевой базе данных вручную до запуска трансфера. Для переноса схемы вручную можно использовать утилиту [`pg_dump`](https://gpdb.docs.pivotal.io/6-19/utility_guide/ref/pg_dump.html).

Источник считает `FOREIGN TABLE` и `EXTERNAL TABLE` обыкновенными представлениями и работает с ними в соответствии с общим алгоритмом для `VIEW`.

Источник никогда не переносит данные из `MATERIALIZED VIEW`, даже в трансферах из {{ GP }} в другую базу данных.

## {{ MG }} {#mongodb}

По умолчанию сервис не шардирует коллекции, переносимые в шардированный кластер. Подробнее см. в разделе [Подготовка к трансферу](../operations/prepare.md#target-mg).

Трансфер в {{ MG }} не переносит индексы. Когда трансфер перейдет в статус {{ dt-status-repl }}, создайте индекс для каждой шардируемой коллекции вручную:

```javascript
db.<имя коллекции>.createIndex(<свойства индекса>)
```

Описание функции `createIndex()` см. в [документации {{ MG }}](https://www.mongodb.com/docs/manual/reference/method/db.collection.createIndex/#mongodb-method-db.collection.createIndex).

## {{ PG }} {#postgresql}

Источник никогда не переносит данные из `MATERIALIZED VIEW`, даже в трансферах из {{ PG }} в другую базу данных. В трансферах _из {{ PG }} в {{ PG }}_ `MATERIALIZED VIEW` считаются обыкновенными представлениями и обрабатываются в соответствии с общим алгоритмом для `VIEW`.

Источник считает `FOREIGN TABLE` обыкновенными представлениями и работает с ними в соответствии с общим алгоритмом для представлений.

Если в источнике трансфера _из {{ PG }} в {{ PG }}_ задан непустой "Список включённых таблиц", пользовательские типы данных, присутствующие в этих таблицах, не переносятся. В этом случае перенесите пользовательские типы данных вручную.

При переносе [партиционированных таблиц](https://www.postgresql.org/docs/current/ddl-partitioning.html) необходимо учитывать следующее:

* **Для таблиц, партиционированных декларативным методом:**

    * Пользователю необходим доступ к основной таблице и всем ее партициям на источнике.
    * Трансфер выполняется по принципу <q>как есть</q>: на приемнике будут созданы все партиции и основная таблица.
    * На этапе копирования партиции переносятся на приемник независимо друг от друга. Благодаря этому пользователь может ускорить трансфер, включив шардирование в [настройках трансфера](../operations/transfer#create).
    * На этапе репликации данные будут автоматически попадать в нужные партиции.
    * Если после того, как трансфер перешел на этап репликации, на источнике будут созданы новые партиции, необходимо будет перенести их на приемник вручную.
    * Пользователь может перенести на приемник только часть партиций. Для этого он должен добавить эти партиции в [Список включенных таблиц](../operations/endpoint/source/postgresql#additional-settings) или закрыть доступ к ненужным партициям на источнике.

* **Для таблиц, партиционированных методом наследования:**

    * Пользователю необходим доступ к родительской таблице и всем таблицам-наследникам.
    * На этапе копирования данные из таблиц-наследников не дублируются в родительскую таблицу. Чтобы перенести данные из таблиц-наследников, их необходимо явно указать в списке переносимых таблиц.
    * На этапе копирования таблицы-наследники переносятся на приемник независимо друг от друга. Благодаря этому пользователь может ускорить трансфер, включив шардирование в [настройках трансфера](../operations/transfer#create).
    * На этапе репликации данные будут автоматически попадать в нужные таблицы-наследники или в родительскую таблицу, если наследование используется не для партиционирования.
    * Если после того, как трансфер перешел на этап репликации, на источнике будут созданы новые таблицы-наследники, необходимо будет перенести их на приемник вручную.

    При переносе базы данных из {{ PG }} в другую СУБД пользователь может включить в эндпоинте-источнике настройку [Объединять наследуемые таблицы](../operations/endpoint/source/postgresql#additional-settings). В этом случае:

    * На приемник будет перенесена только родительская таблица, которая будет содержать данные тех таблиц-наследников, которые были явно указаны в списке переносимых таблиц.
    * Пользователь по-прежнему может ускорить трансфер, включив шардирование в [настройках трансфера](../operations/transfer#create), поскольку копирование таблиц-наследников с источника в общую таблицу на приемнике происходит параллельно.



## {{ yds-full-name }} {#yds}

По умолчанию при переносе данных из {{ yds-name }} в {{ CH }} для каждой партиции создается отдельная таблица. Чтобы все данные попадали в одну таблицу, укажите правила конвертации в [дополнительных настройках эндпоинта для источника](../operations/endpoint/source/data-streams.md#additional-settings).


## Oracle {#oracle}

Источник игнорирует `VIEW` и `MATERIALIZED VIEW` в трансферах любого типа.
