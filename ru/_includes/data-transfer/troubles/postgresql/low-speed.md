### Низкая скорость трансфера  {#low-speed}

​Может возникать у трансферов типа _{{ dt-type-copy }}_ или _{{ dt-type-copy-repl }}_ из {{ PG }} в {{ PG }}.

Возможные причины:

* Протокол записи.

    В нормальном режиме трансфер работает через быстрый протокол `copy`, но при конфликтах записи батча переходит на медленную построчную запись. Чем больше конфликтов записи, тем ниже скорость трансфера.

    **Решение:** установите в эндпоинте-приемнике тип политики очистки `Drop` и исключите другие пишущие процессы.

* Параллельность чтения таблиц.

    Параллельность доступна только для таблиц, которые содержат первичный ключ в [режиме serial](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-SERIAL).

    **Решение:** укажите количество инстансов и процессов в [параметре трансфера](../../../../data-transfer/operations/transfer.md#update) **Среда выполнения** → **{{ yandex-cloud }}** → **Параметры шардированного копирования** и [активируйте](../../../../data-transfer/operations/transfer.md#activate) его повторно.
