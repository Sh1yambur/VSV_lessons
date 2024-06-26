![](../../commonmedia/header.png)

***

   

Триггеры
========

Сегодня познакомимся со специфическим видом процедур в SQL - триггерами.

**Триггер** \- это объект БД, который запускает триггерную функцию в качестве реакции на какой-либо запрос.

**Триггерная функция** (**триггерная процедура**\*) - SQL-функция. Мы познакомились с ними в [прошлой статье](/PLpgSQL-Funkcii-i-procedury-09-30). Но есть небольшой нюанс, о нем - в следующем разделе.

> \* Независимо от названия, де-факто это функция. По крайней мере, в PostgreSQL.

По сути, речь идет о функции, которая выполнится до, после или (при работе с view, в рамках статьи не затронем) вместо _INSERT_, _UPDATE_, _DELETE_ или _TRUNCATE_ запросов определенной таблицы.

Прежде чем перейдем к разбору синтаксиса, подчеркну два аспекта:

1.  Нюансы работы с триггерами могут ощутимо различаться в разных СУБД. Как синтаксически, так и в плане ограничений к их применению;
2.  Триггеры нужно использовать осторожно. Ваша основная зона ответственности как бэкенд-разработчика - именно уровень приложения. Не стоит пихать в триггеры бизнес-логику или какие-то неочевидные действия (или хотя бы документируйте их везде, где это возможно). Использование триггеров может быть более производительным, нежели выполнение тех же операций через Java-код. Но это размазывает ответственность между приложением и БД, затрудняя понимание и поддерживаемость продукта. В идеале, триггеры, если они есть, должны отвечать за вещи, не имеющие отношения к домену вашего приложения. Сбор статистики, наполнение view, логирование и т.д. И даже такие действия лучше документировать, чтобы не превращать приложение в коробку с непонятным черным колдунством.

  

### Синтаксис

Любой триггер можно классифицировать на основании трех аспектов.

По оператору, к которому он привязан:

*   _INSERT_;
*   _UPDATE_;
*   _DELETE_;
*   _TRUNCATE_.

По моменту срабатывания:

*   **_BEFORE_**;
*   **_INSTEAD OF_**. Как говорил выше, в рамках статьи этот вариант не рассматриваем в силу узкой специфики;
*   **_AFTER_**.

По способу применения:

*   К оператору (**_FOR EACH STATEMENT_**). Иными словами, один раз на запрос, независимо от того, сколько строк этот запрос добавил/изменил/удалил. От нуля (т.е. сработает даже если де-факто запрос ничего не сделал) до всего содержимого;
*   К строке (**_FOR EACH ROW_**). Т.е. вызов произойдет на каждую добавленную/измененную/удаленную строку.

В виде компактной таблицы эту информацию с указанными ограничениями можно найти в документации: [ссылка](https://postgrespro.ru/docs/postgresql/9.6/sql-createtrigger#:~:text=%D0%B8%20%D1%81%D1%82%D0%BE%D1%80%D0%BE%D0%BD%D0%BD%D0%B8%D1%85%20%D1%82%D0%B0%D0%B1%D0%BB%D0%B8%D1%86%3A-,%D0%9A%D0%BE%D0%B3%D0%B4%D0%B0,-%D0%A1%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B5).

Синтаксис работы с триггерами на базовом уровне крайне прост. Но сначала подготовим почву для примеров.

Классические примеры для работы триггеров - таблица, в которой будут логироваться действия с другой таблицей.

Логировать будем _INSERT_, _UPDATE_ и _DELETE_ у простенькой таблицы _t1_:

```java
create table t1 (
  id bigint primary key,
  col bigint
);
```

Создадим таблицу-журнал:

```java
create table t1_log (
  t1_id            bigint        not null,
  action_type      varchar(10)   not null,
  col_old_value    bigint,
  col_new_value    bigint,
  happened         timestamp     not null default now()
);
```

Также мы можем написать ряд функций - по одной на каждый из операторов. Но это не слишком удобно, поэтому лучше познакомимся с одной интересной SQL-конструкцией.

Напишем следующую функцию:

```java
create or replace function log_t1() returns trigger
language plpgsql
as $$
  declare
    id bigint = (
      select case 
        when TG_OP = 'DELETE' then old.id
        else new.id 
      end
     );
  begin
    insert into 
      t1_log(t1_id, action_type, col_old_value, col_new_value)
      values (id, TG_OP, old.col, new.col);

    return new;
  end
$$;
```

Разберем моменты, которые, скорее всего, будут непонятны.

В первую очередь, нюанс, о котором я говорил в начале статьи. Индикатор триггерной функции - тип возвращаемого значения указан как **_trigger_**. Кроме того, у такой функции не должно быть задано параметров.

Далее вы можете заметить интересную конструкцию **_CASE_** в _DELCARE_\-блоке. По сути, это аналог _if-else_ из мира SQL. Именно SQL, а не PL/SQL - последний, как раз, использует более привычные if’ы, хоть их синтаксис и отличен от знакомого нам из Java. Подробнее о case можно почитать здесь: [_CASE_ в SQL](https://postgrespro.ru/docs/postgresql/9.6/functions-conditional#functions-case). Суть использования этой конструкции здесь станет понятна в следующем абзаце.

Следующим непонятным моментов будет, вероятно **_TG\_OP_**, **_NEW_** и **_OLD_**, которые содержат какие-то значения, но откуда он взялся - решительно непонятно. Это переменные триггеров, которые заполняются автоматически (за редким исключением) и могут быть использованы внутри триггерной функции. Подробнее можно почитать здесь: [переменные триггеров](https://postgrespro.ru/docs/postgresql/11/plpgsql-trigger#PLPGSQL-DML-TRIGGER).

Как вы можете увидеть в документации, для DELETE-запроса _NEW_ будет _NULL_, что и стало причиной использования конструкции _CASE_ в нашей функции.

В целом, рекомендую прочитать весь пункт 43.10.1 по ссылке выше. Обратить внимание стоит как минимум на две вещи (не считая описания упомянутых выше переменных):

1.  Переменная **_TG\_ARGV\[\]_** - она позволяет передавать в триггерную функцию собственные параметры, тем самым обходя ограничение на отсутствие параметров при определении функции.
2.  Описание работы с возвращаемым значением из триггерной функции. Абзац с _INSTEAD OF_ для нас не критичен, а вот оставшиеся два - вполне. Весьма важный пункт, который там хорошо описан, поэтому я не вижу смысла дублировать эту информацию в рамках статьи.

Итак, мы создали лаконичную триггерную функцию, которая может для каждой изменяемой строки таблицы записать простой лог.

Осталось привязать эту функцию триггером к таблице.

Создадим простой триггер, который будет отрабатывать на операцию _INSERT_ для каждой строки:

```java
create or replace trigger log_t1_insert_trigger
after insert on t1
for each row
execute procedure log_t1();
```

Как можете видеть, триггер тоже поддерживает _CREATE OR REPLACE_ наравне с обычным _CREATE_.

После названия триггера указано, когда он сработает - _AFTER_ операции _INSERT_ для таблицы _t1_. Будет отрабатывать _FOR EACH ROW_ и заключается в вызове функции (английский термин для триггерной функции - trigger procedure) _log\_t1()_.

Теперь осталось добавить такие же триггеры на _UPDATE_ и _DELETE_. Но мы сделаем немного проще:

*   Удалим добавленный триггер (обратите внимание, что он жестко привязан к конкретной таблице):

```java
drop trigger log_t1_insert_trigger on t1;
```

*   Добавим общий триггер на все интересующие нас операции:

```java
create or replace trigger log_t1_trigger
after update or insert or delete on t1
for each row
execute procedure log_t1();
```

Теперь можем протестировать вставку, изменение и удаление строк в таблице _t1_ и посмотреть результат в _t1\_log_:

```java
insert into t1 values (1, 1), (2, 2), (3, 3);
delete from t1 where id = 1;
update t1 set col = 10 where id != 1;
```

Из основного синтаксиса проигнорированным остался только _ALTER_. Для триггеров (на нашем уровне изучения) доступно только переименование, с ним все просто:

```java
alter trigger log_t1_trigger on t1 rename to log_t1_tg;
```

Кроме того, подчеркну ряд моментов, которые не были освещены выше.

В рамках таблицы на одну и ту же операцию может быть повешено более одного триггера. Особенно это чувствительно для _BEFORE_\-триггеров, где порядок вызова может иметь значение.

Собственно, порядок вызова зависит от СУБД. Стандарт SQL регламентирует вызовы триггеров в порядке их добавления, но PostgreSQL отходит от стандарта и ориентируется на имя триггера - они выполняются в алфавитном порядке (точнее, в порядке естественной сортировки строк по возрастанию).

Также при создании триггеры можно привязать не просто к конкретной операции, а сузить их использование до следующих ситуаций:

*   Выполнения логического условия с помощью оператора _WHEN_. В рамках него доступны переменные триггера (включая _OLD_ и _NEW_), что позволяет вызывать триггеры только в заданных ситуациях, а не на каждый вызов оператора. Актуально, в основном, для _FOR EACH ROW_ триггеров. Примеры: [ссылка](https://postgrespro.ru/docs/postgresql/9.6/sql-createtrigger#:~:text=EXECUTE%20PROCEDURE%20check_account_update()%3B-,%D0%92%20%D1%8D%D1%82%D0%BE%D0%BC%20%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80%D0%B5,-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D0%B1%D1%83%D0%B4%D0%B5%D1%82%20%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D1%82%D1%8C%D1%81%D1%8F);
*   Для _UPDATE_\-триггеров доступно предложение _OF_, которое позволяет указать столбцы, которые должны быть изменены, чтобы триггер сработал. Пример: [ссылка](https://postgrespro.ru/docs/postgresql/9.6/sql-createtrigger#:~:text=EXECUTE%20PROCEDURE%20check_account_update()%3B-,%D0%A2%D0%BE%20%D0%B6%D0%B5%20%D1%81%D0%B0%D0%BC%D0%BE%D0%B5,-%2C%20%D0%BD%D0%BE%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D1%82%D1%80%D0%B8%D0%B3%D0%B3%D0%B5%D1%80%D0%B0).

  

С теорией на сегодня все!

![](../../commonmedia/footer.png)

Переходим к практике:

### Задача 1

Реализуйте логирование действий для всех таблиц в тестовой БД в виде _FOR EACH STATEMENT_ триггеров, которые будут писать в таблицу логов примененную операцию, таблицу, к которой была применена операция и дату срабатывания триггера.

  

### Задача 2

Реализуйте триггер, который будет переводить имя и фамилию в верхний регистр при добавлении или изменении (имени или фамилии) пассажира.

  

Если что-то непонятно или не получается – welcome в комменты к посту или в лс:)

Канал: [https://t.me/ViamSupervadetVadens](https://t.me/ViamSupervadetVadens)

Мой тг: [https://t.me/ironicMotherfucker](https://t.me/ironicMotherfucker)

_Дорогу осилит идущий!_