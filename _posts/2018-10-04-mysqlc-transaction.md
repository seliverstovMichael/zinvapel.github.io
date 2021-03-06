---
layout: post
title: "[Конспект] Про блокировки и транзакции в MySQL"
date: "2018-10-04"
category:
  - prog
  - database
---
{% include image.html src="/assets/it/prog/stop.jpg" %}

В этом посте я хотел бы собрать воедино информацию о блокировках и транзакциях в MySQL.

<!--more-->

## Блокировки
### Типы блокировок
- **Разделяемые** (shared). Блокировки чтения, возможность читать записи есть у всех клиентов, а возможности записи нет. Каждый клиент может поставить такую блокировку одновременно.
- **Монопольные** (exclusive). Никто не сможет повесить другую блокировку на данные, а также выполнить изменения.

### Стратегии блокировок
- **Табличные блокировки**. Блокируется вся таблица, причём блокировка записи двигает блокировки чтения в очереди.
- **Блокировка строк**. Более накладной вариант блокировок, поддерживается в InnoDB и Falcon.

### Блокировки в InnoDB:
InnoDB использует блокировки на уровне строк. В зависимости от уровня изоляции транзакции могут блокироваться как строки, попавшие в результирующую выборку, так и все строки, что были просмотрены при поиске. Например, в REPEATABLE READ блокирующий запрос без использования индекса потребует перебора всей таблицы, а следовательно и блокировки всех записей.

Есть два базовых типа блокировок:
- **Shared lock** — совместная блокировка, позволяет другим транзакциям читать строку и ставить на нее такую же совместную блокировку, но не позволяет изменять строку или ставить исключительную блокировку.
- **Exclusive lock** — исключительная блокировка, запрещает другим транзакциям блокировать строку, а также может блокировать строку как на запись, так и на чтение в зависимости от текущего уровня изоляции (о коих ниже).

Если копнуть глубже, то выяснится, что есть еще 2 типа блокировок, назовем их блокировками «о намерениях». Нельзя просто так взять и заблокировать запись в InnoDB. Блокировки _intention shared_ и _intention exclusive_ являются блокировками на уровне таблицы и блокируют только создание других блокировок и операции на всей таблице типа LOCK TABLE. Наложение такой блокировки транзакцией лишь сообщает о намерении данной транзакции получить соответствующую совместную или исключительную блокировку строки.

InnoDB накладывает блокировки не на сами строки с данными, а на записи индексов. Та или иная блокировка может накладываться на:
- *Record lock* — блокировка записи индекса.
- *Gap lock* — блокировка промежутка между, до или после индексной записи.
- *Next-key lock* — блокировка записи индекса и промежутка перед ней.

Блокировка промежутков нужна для того, чтобы избежать появления фантомных записей, когда, например, между двумя одинаковыми чтениями диапазона соседняя транзакция успевает вставить запись в этот диапазон.

`SELECT… LOCK IN SHARE MODE` — блокирует считываемые строки на запись.

Другие сессии могут читать, но ждут окончания транзакции для изменения затронутых строк. Если же в момент такого SELECT'а строка уже изменена другой транзакцией, но еще не зафиксирована, то запрос ждет окончания транзакции и затем читает свежие данные. Данная конструкция нужна, как правило, для того чтобы получить свежайшие данные (независимо от времени жизни транзакции) и заодно убедиться в том, что их никто не изменит.

`SELECT… FOR UPDATE` — блокирует считываемые строки на чтение. Точно такую же блокировку ставит обычный UPDATE, когда считывает данные для обновления.

Взаимоблокировки (__deadlock__) возникают тогда, когда две и более транзакции взаимно удерживают и запрашивают блокировку одних и тех же ресурсов, создавая циклическую зависимость. InnoDB обрабатывает взаимоблокировки откатом той транзакции, которая захватила меньше всего монопольных блокировок строк (приблизительный показатель легкости отката). Нельзя справиться с взаимоблокировками без отката одной из транзакций, частичного либо полного.

## Транзакции в MySQL
MySQL предоставляет пользователям две транзакционные подсистемы хранения данных: InnoDB и NDB Cluster. MySQL позволяет устанавливать уровень изолированности с помощью команды `SЕТ TRANSACТION ISOLAТION LEVEL`, которая начинает действовать со следующей транзакции. Можете настроить уровень изолированности для всего сервера в конфигурационном файле или только для своей сессии.

По умолчанию MySQL работает в режиме `АUТОСОММIT`. Это означает, что, пока вы не начали транзакцию явно, каждый запрос автоматически выполняется в отдельной транзакции. Некоторые команды, будучи запущенными во время начатой транзакции, заставляют MySQL подтвердить транзакцию до их выполнения. Обычно это команды языка определения данных (Data Definition Language, DDL), которые вносят изменения в структуру таблиц, например ALТЕR TABLE, но LOCK TABLES и другие директивы также обладают этим свойством.

Если вы используете транзакционные и нетранзакционные таблицы (например, таблицы InnoDB и MyISAM) в одной транзакции, то все будет работать хорошо, пока не произойдет что-то неожиданное, откатить данные из нетранзакционных таблиц невозможно.

### ACID:
- _A_ - Атомарность

Транзакция должна функционировать как единая неделимая единица работы таким образом, чтобы вся транзакция была либо выполнена, либо отменена.

- _C_ - Консистентность

База данных должна всегда переходить из одного непротиворечивого состояния в последующее.

- _I_ - Изолированность

Результаты транзакции обычно невидимы другим транзакциям, пока она не закончена.

- _D_ - Долговечность

Будучи зафиксированы, внесенные в ходе транзакции изменения становятся постоянными. Это означает, что изменения должны быть записаны так, чтобы данные не могли быть потеряны в случае сбоя системы.

### Уровни изоляции транзакций
Проблемы параллельного доступа с использованием транзакций:

| Проблема | Описание |
| - | - |
| Lost update - потерянное обновление | При одновременном изменении одного блока данных разными транзакциями одно из изменений теряется. |
| Dirty read - грязное чтение | Чтение данных, добавленных или изменённых транзакцией, которая впоследствии не подтвердится. |
| Non-repeatable read - неповторяемое чтение | При повторном чтении в рамках одной транзакции ранее прочитанные данные оказываются изменёнными. |
| Phantom read - фантомное чтение | При повторном чтении в рамках одной транзакции одна и та же выборка дает разные множества строк. |

### Уровни изоляции:

| Название | Описание | Реализация |
| - | - |
| READ UNCOMMITED | Гарантирует только отсутствие потерянных обновлений. Если несколько параллельных транзакций пытаются изменять одну и ту же строку таблицы, то в окончательном варианте строка будет иметь значение, определенное всем набором успешно выполненных транзакций. Защита от Lost Update. | В рамках транзакции Т1 накладывается разделяемая блокировка на изменяемые данные, все остальные транзакции, желающие изменить эти данные, ждут завершения Т1. |
| READ COMMITED | Обеспечивается защита от грязного чтения, тем не менее, в процессе работы одной транзакции другая может быть успешно завершена и сделанные ею изменения зафиксированы. В итоге первая транзакция будет работать с другим набором данных. | В рамках транзакции Т1 создается снимок изменяемых строк, с которым будут работать все остальные клиенты до завершения Т1. Если будут несколько изменений одних и тех же строк, то зафиксированы будут только изменения Т1. |
| REPEATABLE READ | Читающая транзакция «не видит» изменения данных, которые были ею ранее прочитаны. При этом никакая другая транзакция не может изменять данные, читаемые текущей транзакцией, пока та не окончена. | В рамках транзакции Т1 накладывается монопольная блокировка на считываемые данные, все остальные транзакции, желающие изменить эти данные, ждут завершения Т1. |
| SERIALIZABLE | Транзакции полностью изолируются друг от друга, каждая выполняется так, как будто параллельных транзакций не существует. | Параллельным транзакциям данные блокируются даже для чтения. |

Конкретная реализация каждого уровня изоляции определяется подсистемой хранения данных MySQL.

### Уровни изоляции InnoDB

*REPEATABLE READ*
- Согласованное чтение (SELECT) ничего не блокирует, читает строки из снимка, который создается при первом чтении в транзакции. Одинаковые запросы всегда вернут одинаковый результат.
- Для блокирующего чтения (SELECT… FOR UPDATE/LOCK IN SHARE MODE), UPDATE и DELETE блокировка будет зависит от типа условия. Если условие уникально (WHERE id=42), то блокируется только найденная индексная запись (record lock). Если условие с диапазоном (WHERE id > 42), то блокируются весь диапазон (gap lock или next-key lock).


*READ COMMITED*
- Согласованное чтение ничего не блокирует, но каждый раз происходит из свежего снимка.
- Блокирующее чтение (SELECT… FOR UPDATE/LOCK IN SHARE MODE), UPDATE и DELETE блокирует только искомые индексные записи (record lock). Таким образом возможна вставка параллельным потоком записей в промежутки между индексами. Промежутки блокируются (gap lock) только при проверках внешних ключей и дублирующихся ключей. Также блокировки просканированных строк (record lock), не удовлетворяющих WHERE, снимаются сразу же после обработки WHERE.


*READ UNCOMMITED*
- Все запросы SELECT читают в неблокирующей манере. Изменения незавершенной транзакции могут быть прочитаны в других транзакциях, а изменения эти могут быть еще и впоследствии откачены. Это так называемое «грязное чтение» (несогласованное).
В остальном все так же, как и при READ COMMITED.


*SERIALIZABLE*
- Аналогичен REPEATABLE READ, за исключением одного момента. Если autocommit выключен (а при явном старте транзакции он выключен), то все простые запросы SELECT неявно превращаются в SELECT… LOCK IN SHARE MODE, если включен — каждый SELECT идет в отдельной транзакции. Используется, как правило, для того чтобы превратить все запросы чтения в SELECT… LOCK IN SHARE MODE, если этого нельзя сделать в коде приложения.

### Журнал транзакций
Ведение журнала помогает сделать транзакции более эффективными. Вместо обнов­ления таблиц на диске после каждого изменения подсистема хранения данных может изменить находящуюся в памяти копию данных. Это происходит очень быстро. Затем подсистема хранения запишет сведения об изменениях в журнал транзакции, который хранится на диске и поэтому долговечен. Это тоже доволь­но быстрая операция, поскольку добавление событий в журнал сводится к опе­рации последовательного ввода/вывода в пределах ограниченной области диска вместо случайного ввода/вывода в разных местах. Позже процесс обновит табли­цу на диске.

## MVCC - multiversion concurrency control
MVCC сохраняет мгновенный снимок состояния данных в определенный момент времени.

InnoDB реализует MVCC, сохраняя вместе с каждой строкой два скрытых значения, которые показывают, когда строка была создана и когда истек срок ее хранения (или она была удалена). Вместо хранения фактического момента времени, когда произошли данные события, строка хранит номер версии системы для этого момента.

Применение для Repeatable Read:
- SELECT. InnoDB должна проверить каждую строку на соответствие двум критериям.
     - Найти версию строки, по крайней мере такой же старой, как версия транзакции (то есть ее номер версии должен быть меньше номера версии транзакции или равен ему). Это гарантирует, что либо строка существовала до начала транзакции, либо транзакция создала или изменила эту строку.
     - Версия удаления строки должна быть не определена, или ее значение должно быть больше, чем версия транзакции. Это гарантирует, что строка не была удалена до начала транзакции.
- INSERT. InnoDB записывает текущий номер версии системы вместе с новой строкой.
- DELEТE. InnoDB записывает текущий номер версии системы как ID удаления строки.
- UPDATE. InnoDB записывает новую копию строки, используя номер версии системы в качестве версии новой строки. Она также записывает номер версии системы как версию удаления старой строки.
