# Домашнее задание к занятию "Резервное копирование баз данных" - Морозов Александр

---

### Задание 1. Резервное копирование

### Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования. 

Необходимо описать, какие варианты резервного копирования подходят в случаях: 

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.

1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.

1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.

*Приведите ответ в свободной форме.*

Ответ:
1.1 Необходимо выполнять полное резервное копирование раз в неделю. А раз в день дифференциальный бэкап. Для восстановления данных в полном объеме за заданный день нужно будет восстановить полный бэкап на начало недели и на него накатить тот дифференциальный бэкап за тот день, за который нужно восстановить данные.

1.2 В этом случае как и в первом основой является полный бэкап но уже раз в день (в нерабочее время, когда операции не происходят или нагрузка снижена). А с необходимой частотой (в нашем случае раз в час) осуществляется инкрементный бэкап. Для восстановления данных необходимо восстановить нанные на начало дня полным бэкапом, а далее поочередно накатывать инкрементные бэкапы до того искомого часа.

1.3* Данный кейс возможен в рамках репликации а не резервного копирования. При обрушении мастера слейв, хранящий последние реплицированные данные, переводится в режим мастера руками (медленно) или автоматически (скриптом и быстро).

---

### Задание 2. PostgreSQL

2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

2.2.* Возможно ли автоматизировать этот процесс? Если да, то как?

*Приведите ответ в свободной форме.*

Ответ:

2.1 С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

Следующая команда делает дамп базы данных, используя специальный формат дампа:

```
pg_dump -Fc bd_name > dump.sql
```

Специальный формат дампа не является скриптом для psql и должен восстанавливаться с помощью команды 
pg_restore, например:

```
pg_restore –d bd_name dump.sql
```

2.2* Возможно ли автоматизировать этот процесс? Если да, то как?**

Скрипт для автоматического резервного копирования
postgresql_dump.sh

``` 
#!/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
PGPASSWORD=password

export PGPASSWORD

pathB=/backup

dbUser=dbuser
database=db

find $pathB \( -name "*-1[^5].*" -o -name "*-[023]?.*" \) -ctime +61 -delete

pg_dump -U $dbUser $database | gzip >  $pathB/pgsql_$(date "+%Y-%m-%d").sql.gz

unset PGPASSWORD
```

или для запуска резервного копирования по расписанию, сохраняем скрипт в файл, например, /scripts/postgresql_dump.sh и создаем задание в планировщике:

```
crontab -e

6 0 * * * /scripts/postgresql_dump.sh
```

Скрипт будет запускаться каждый день в 06:00.

---

### Задание 3. MySQL

3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL. 

3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?

*Приведите ответ в свободной форме.*

ОТВЕТ:
3.1. Создание инкрементных резервных копий с помощью двоичного журнала

MySQL поддерживает инкрементные резервные копии: следует запустить сервер с опцией --log-bin , чтобы включить двоичное журналирование, см. раздел 6.4.4. Двоичные файлы журнала предоставляют Вам информацию, Вы должны тиражировать изменения в базу данных, что делается после точки, в которой Вы выполняли резервное копирование. Когда Вы хотите сделать инкрементное резервное копирование (содержащее все изменения, которые произошли, начиная с последнего полного или инкрементного резервного копирования), следует ротировать двоичный журнал, используя FLUSH LOGS. Вы должны скопировать в резервную копию все двоичные журналы, которые находятся в диапазоне от момента последнего полного или инкрементного резервного копирования до предпоследнего. Эти двоичные журналы и есть инкрементное резервное копирование, во время восстановления Вы применяете их как объяснено в разделе 8.5. В следующий раз, когда Вы делаете полное резервное копирование, следует также ротировать журнал с помощью FLUSH LOGS или mysqldump --flush-logs.

Инкрементальная резервная копия содержит только ту информацию, которая изменилась после создания предыдущей резервной копии. Это существенно уменьшает размер резервных копий и позволяет вам делать такие резервные копии очень часто.

В MySQL вы можете реализовать создание инкрементальных резервных копий с помощью резервного копирования двоичных файлов журнала. Все транзакции, применяемые к серверу MySQL, последовательно записываются в двоичные файлы журнала. Следовательно, вы всегда можете восстановить исходную базу данных из этих файлов.

```
mysqldump --flush-logs --delete-master-logs --single-transaction --all-databases | gzip > /var/backups/mysql/$(date +%d-%m-%Y_%H-%M-%S)-inc.gz
```
---