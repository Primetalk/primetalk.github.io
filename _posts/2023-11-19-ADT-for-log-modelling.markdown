---
layout: single
title:  "Применение алгебраических типов данных для моделирования ошибок и сообщений в журнале"
date:   2023-11-19 22:47:00 +0300
categories: scala
tags: scala algebraic-data-types
toc: false
toc_label: "Содержание"
toc_icon: "cog"
---

В функциональном программировании широко используются так называемые алгебраические типы данных. Такие данные формируются из более простых типов с использованием всего двух операций — "суммы" и "произведения". Использование таких математических операций оказывается очень удобным с точки зрения последующей обработки с помощью сопоставления с образцом ("паттерн-матчинг"/pattern matching). 

Помимо удобства при разработке, математичность получающихся структур данных позволяет делать более понятные, прозрачные и логичные конструкции, находящие применение в самых разных предметных областях.

В этой заметке посмотрим на примеры моделирования ошибок и сообщений логирования.

## Алгебраические типы данных (ADT и GADT)

Имея некоторый базовый набор типов (целые числа, строки, логические значения, ...) мы хотим иметь возможность строить более сложные структуры данных. Для этого уже имеющиеся типы можно объединять с использованием двух операций — произведения и суммы.

Произведением типов называется новый тип, который одновременно содержит значения всех типов элементов. 

Обыкновенный объект, запись, строчка в таблица, представляют собой типы произведения.

```sql
CREATE TABLE person (
  id   int,
  name text,
  age  int,
)
```

```scala
case class Person(id: Int, name: String, age: Int)
type Person2 = (Int, String, Int)
```

```haskell
data Person = Person Int String Int
type Person2 = (Int, String, Int)
```

```json
{
  "$id": "https://example.com/person.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Person",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer"
    },
    "name": {
      "type": "string"
    },
    "age": {
      "type": "integer"
    }
  }
}
```

В типах произведениях можно использовать метки для каждого входящего типа. С точки зрения математики, наличие меток не меняет свойств получаемого типа. Но для человека наличие меток важно для понимания программы.

Суммой типов называется новый тип, который может различимым образом принимать все возможные значения входящих в него типов элементов.

Примером суммы типов служит перечисление (`enum`).

```haskell
data Status = On | Off | Broken
```

```scala
enum Status:
  case On, Off, Broken
```
```json
{
  "$id": "https://example.com/enumerated-values.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Status",
  "type": "object",
  "properties": {
    "status": {
      "enum": ["on", "off", "broken"]
    }
  }
}
```

В перечислении в каждом случае (`case`'е) мы берём тип, содержащий ровно одно значение (такой тип можно назвать единицей — `Unit`'ом). Получающийся тип будет иметь уже `n` значений, по числу вариантов.

Посмотрим, как складываются более сложные типы. Допустим, что результатом функции может быть либо успех и какое-то значение, либо ошибка с пояснением.

```haskell
data Result a = Failure String | Success a
```

```scala
sealed trait Result[+A]
object Result:
  case class Failure(message: String) extends Result[Nothing]
  case class Success[+A](value: A) extends Result[A]
end Result
```

Каждый тип, входящий в сумму, снабжается меткой (`On`, `Off`, `Broken`, `Failure`, `Success`), позволяющей идентифицировать используемый вариант.

### Математические свойства*

Сумма `⊕` и произведение `⊗` обладают такими свойствами:

1. Имея значение любого из типов `A`, `B`, можно получить `i1`, `i2` значение суммы типов `A⊕B`. А имея значение суммы типов, можно получить значение только какого-то одного из суммируемых типов.
2. Имея по одному значению для каждого из входящих типов `A`, `B`, можно построить значение типа произведения `A⊗B`. А имея значение произведения типов, можно получить `π1`, `π2` сразу все значения всех входящих типов (и любой из них в отдельности).

Эти свойства обычно иллюстрируются такой диаграммой:

![product & coproduct из статьи John D. Cook, PhD [Commutative diagrams in LaTeX](https://www.johndcook.com/blog/2014/04/14/commutative-diagrams-in-latex/)](../assets/images/prod_coprod.png)

На этом минутка математики закончилась...

### Примеры

Посмотрим, как используются типы суммы и произведения в примерах из области программирования.

#### Список

Классический односвязный список может быть описан следующим образом:

```haskell
data List a = Nil | Cons a (List a)
```

```scala
sealed trait List[+A]
case object Nil extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
```

Здесь тип списка представляет собой сумму двух вариантов — единичного типа, помеченного меткой `Nil`, и типа произведения, помеченного меткой `Cons`. Тип произведения в свою очередь содержит пару типов: тип `A` и тип `List[A]`.

#### Бинарное дерево

Один из вариантов бинарного дерева можно представить так:

```haskell
data Stree a = Tip | Node (Stree a) a (Stree a)
```

```scala
sealed trait Stree[+A]
case object Tip extends Stree[Nothing]
case class Node[+A](left: Stree[A], value: A, right: Stree[B]) extends Stree[A]
```

Здесь также тип представляет сумму двух типов — единичного и типа произведения. Тип произведения уже основан на трёх типах.

### Реляционная алгебра и алгебраические типы данных *

Заслуженно популярным способом хранения данных является какая-либо реляционная база данных, например, Postgres. Реляционные алгебры замечательно поддерживают типы-произведения. Обычная таблица БД представляет собой как раз такой тип-произведение. Join'ы (декартово произведение, cross product) — также являются определённой разновидностью типов-произведений.

По-видимому, проблемой является отсутствие полноценной поддержки типов сумм в классической реляционной алгебре. Впрочем, некоторые частные случаи типов-сумм поддерживаются — `enum`, наследование таблиц.

В статье  [Matt Parsons: Sum Types In SQL](https://www.parsonsmatt.org/2019/03/19/sum_types_in_sql.html) рассматриваются варианты представления типов-сумм в SQL. Например, можно для каждого варианта завести свой набор nullable-колонок.

#### Пример представления типа-суммы

Рассмотрим пример. Пусть в базе данных хранятся сведения о гражданах и каждый гражданин может использовать один из нескольких разрешённых документов для идентификации — паспорт, водительское удостоверение, ... С использованием ADT мы можем записать такой тип следующим образом:

```haskell
data PersonIdentificationDocument = DrivingLicense String | Passport String Date
```

```scala
sealed trait PersonIdentificationDocument 
object PersonIdentificationDocument:
  case class DrivingLicense(dlNumber: String) extends PersonIdentificationDocument
  case class Passport(passportNumber: String, issueDate: Date) extends PersonIdentificationDocument
end PersonIdentificationDocument
```
(Для простоты ограничим количество полей и будем считать, что номера документов имеют разные шаблоны.)

В базе данных такой тип может быть представлен, например, так:

```sql
CREATE TYPE PersonIdentificationDocumentKind AS ENUM (DrivingLicense, Passport)

CREATE TABLE PersonIdentificationDocument (
  kind PersonIdentificationDocumentKind,
  dlNumber text NULL,
  passportNumber text NULL,
  issueDate date NULL
)
ALTER TABLE PersonIdentificationDocument ADD CHECK (
  (CASE WHEN kind = DrivingLicense THEN (dlNumber IS NOT NULL)::integer END)
+ 
  (CASE WHEN kind = Passport THEN (passportNumber IS NOT NULL AND issueDate IS NOT NULL)::integer END)
  = 1
)
```
Уже на этом простом примере видны некоторые проблемы с представлением типов-сумм в реляционных таблицах:
- необходимость хранения разреженных записей, содержащих только несколько `NOT NULL` значений;
- сложные ограничения;
- отсуствие `XOR`, что вынуждает использовать трюки разной степени нетривиальности;
- наглядность представления далека от идеала.

## Модель ошибок

Попробуем применить две операции для моделирования ошибок, возникающих в типичной информационной системе, обрабатывающей пользовательские данные (`CRUD`, клиент-сервер, трёхзвенная система).

По-видимому, на верхнем уровне мы вводим тип ошибки `Error` как тип-сумму нескольких вариантов ошибок.

### Классификация ошибок HTTP

Для HTTP уже проведена достаточно подробная классификация ошибок, выражаемых кодами возврата. Клиентские ошибки в диапазоне 400-499, серверные — 500-599. Например, 403 Forbidden, 404 Not found, 500 Internal Server Error, 503 Service Unavailable.

Эту классификацию вполне можно использовать и в прикладных программах:
```scala
sealed trait Error
sealed trait RequestError extends Error
object RequestError:
  case class BadRequest() extends RequestError
  case class Unauthorized() extends RequestError
  case class Forbidden() extends RequestError
  case class NotFound() extends RequestError
end RequestError
sealed trait ServerError extends Error
object ServerError:
  case class InternalServerError(logMessageId: LogReceipt) extends ServerError
  case class ServiceUnavailable() extends ServerError
  case class OtherServiceUnavailable(serviceId: ServiceId) extends ServerError
end ServerError
```

### Ошибки валидации входных данных

При получении данных мы проверяем, соответствуют ли эти данные определённым условиям. Мы можем выразить ошибки валидации, например, таким образом:

```scala
case class ConstraintFailure(name: String)
case class FieldValidationFailure(fieldName: String, constraints: Seq[ConstraintFailure])
case class EntityValidationFailure(failures: Seq[FieldValidationFailure])
```
Желательно, чтобы обработка данных была построена таким образом, чтобы полученные после валидации данные гарантированно обрабатывались. Если всё же данные на каком-то этапе не удалось обработать, то это уже будет баг.

### Ошибки взаимодействия с другими данными

Даже если пользовательские данные сами по себе корректные, может оказаться, что на каком-то этапе обработки обнаружится противоречие с другими данными.
```scala
sealed trait DataConflict
case class MissingRelatedEntity() extends DataConflict
case class AlreadyExists() extends DataConflict
```

### Баги

В современном нам программировании принято считать, что программ без багов не существует. При этом, с одной стороны, довольно проблематично для самой программы обнаружить баг, а, с другой стороны, программа пока не может самостоятельно ничего с ним сделать.

Из предшествующего опыта оказывается, что несмотря на наличие багов, программа всё равно может быть полезной. Например, если баг проявляется не в каждом запросе пользователя, то программа сможет часть запросов успешно обработать. Или если баг заключается в утечке памяти, перезапуск программы (даже автоматический), позволит программе в течение некоторого времени успешно обрабатывать запросы пользователей.

Для багов, по-видимому, разумно полагаться на технологию исключений и их обработки, а также на библиотеки работы с эффектами. При этом конкретный запрос, конечно, не будет обработан, но остальным запросам, возможно, повезёт больше.


## Модель сообщений для журнала

Современные приложения являются распределёнными и параллельными, обрабатывают множество запросов разных клиентов одновременно на разных стадиях на нескольких узлах/процессах. Старинные текстовые журналы, в которых линейным образом записываются сообщения одного процесса, постепенно отходят в прошлое. Современные системы подразумевают, что сообщения от любой части распределённой системы (микросервиса)собираются в централизованную систему сбора и анализа логов зачастую в режиме реального времени. Сообщения снабжаются корреляционными метками:
- момент времени (`timestamp`, `moment`, `instant`);
- идентификатор обслуживаемого запроса (`traceId`);
- идентификатор пользователя, прошедшего аутентификацию (`userId`);
- идентификатор компонента (`componentId`);
- идентификатор процесса/потока (`processId`, `threadId`);
- версия системы (`versionId`, `commitId`, `sha`);
- уровень детализации (`logLevel`);
- ...

Такие метки позволяют относительно легко найти все сообщения, относящиеся к интересующему нас событию.

Если посмотреть на какой-нибудь журнал приложения, то можно заметить, что после отделения всей формализованной информации в свойства сообщения, от текстовой строки лога остаётся относительно небольшая константа. Такую константу, по-видимому, вполне можно представить с помощью `enum`'а.

По-видимому, вполне можно окончательно отказаться от текстовой формы журнала и перейти к использованию структурированных сообщений, включающих все необходимые поля.

На верхнем уровне мы должны предоставлять механизм отправки наших сообщений в централизованную систему. Пусть такой механизм принимает сообщения типа `LowLevelMessage`. Например, это может быть банальный `json`. Этот механизм мы должны изолировать от всего остального приложения и предоставить возможность логирования только структурированных сообщений.

Обычно исходный запрос, полученный от пользователя, обогащается контекстной информацией, получаемой на последующих этапах обработки. Предположим, что в заголовках запроса непосредственно имеется `traceId`. В таком случае сообщение будем иметь вид:
```scala
case class TraceIdLogMessage(traceId: TraceId, ???)
```

И мы должны уметь конвертировать это сообщение в `LowLevelMessage`:
```scala
given Conversion[TraceIdLogMessage, LowLevelMessage]
```

Иногда к нам могут поступать дефектные сообщения, не содержащие `traceId`. Дальнейшая работа приложения в таком случае не имеет смысла, т.к. не будут удовлетворены требования к выполнению. Однако, возможно, мы захотим сохранить в журнале информацию о таких дефектных запросах для проведения расследования.

```scala
sealed trait RequestLogMessage
object RequestLogMessage:
  case class InvalidRequest(request: Request) extends RequestLogMessage
  case class TraceIdLogMessage(traceId: TraceId, ???) extends RequestLogMessage
end RequestLogMessage
```
(где-то на этом уровне можно передать `timestamp`, либо неявно "подшить" при передаче в нижележащую систему, если это не критично.)
Посмотрим, что может скрываться за `???`. Пусть, например, следующий этап обработки запроса — аутентификация и авторизация. Здесь возможны такие исходы:
- пользователь успешно аутентифицирован;
- пользователь не аутентифицирован.

```scala
sealed trait AuthenticatedLogMessage
object AuthenticatedLogMessage:
  case class InvalidUser(...) extends AuthenticatedLogMessage
  case class ValidUserLogMessage(userId: UserId, ???) extends AuthenticatedLogMessage
end AuthenticatedLogMessage
```

Далее мы можем проверить авторизацию пользователя:
```scala
sealed trait AuthorizedLogMessage
object AuthorizedLogMessage:
  case class Deny(...) extends AuthorizedLogMessage
  case class Allow(???) extends AuthorizedLogMessage
end AuthorizedLogMessage
```
Таких уровней может быть несколько. По мере продвижения в обработке запроса, все наши сообщения получают контекст, известный к моменту формирования сообщения.

Ну и на каком-то семантическом уровне обработки у нас появится `DomainAction`:
```scala
sealed trait DomainActionLogMessage
object DomainActionLogMessage:
  case class Action1(...) extends DomainActionLogMessage
  case class Action2(...) extends DomainActionLogMessage
  case object Action3 extends DomainActionLogMessage
  
  ...
end DomainActionLogMessage
```

Такое структурированное логирование обеспечивает полный контекст и исчерпывающий перечень сообщений, генерируемых системой. 

### Отладочные и ad-hoc сообщения

Имея такую структуру, нет возможности записывать произвольные сообщения. Если всё же возникла такая необходимость, то можно добавить к соответствующему `sealed trait`'у специальный случай:
```scala
  @deprecated
  case class Debug(msg: String) extends ...LogMessage
```
(и пометить `deprecated`, чтобы не забыть удалить такие посторонние сообщения.)

### Обобщённые сообщения *

Глядя на структуру сообщений, невольно закрадывается мысль, что встречаются похожие конструкции и можно было бы использовать обобщённое программирование. Например, таким образом:

```scala
case class WithTraceId[A](traceId: TraceId, m: A)
case class WithUserId[A](userId: UserId, m: A)

type TraceIdLogMessage = WithTraceId[WithUserId[DomainActionLogMessage]]
```
Для представления вариантов — `Either[Invalid, Valid]`.

### Маскирование секретов*

Программы, обрабатывающие секретные данные (например, PHI — personal health information, PII — personal identifiable information, и т.п.), также нуждаются в логировании, как и другие программы. При этом попадание секретной информации в логи является утечкой и этого стараются избежать. Типичным способом является апостериорное обнаружение секретов в строковых записях журналов с помощью регулярных выражений.

Если мы пользуемся структурированными записями, то никто нам не мешает особым образом сериализовать поля, содержащие секреты.

Например, так:
```scala
opaque type Secret = String
object Secret:
  given Codec[Secret] = Codec[String].map(_ => "<***>")
end Secret
```
Такое секретное поле при сериализации автоматически превратится в константную строку.

### Интерфейс библиотеки логирования

Библиотека логирования может иметь примерно такой интерфейс:

```scala
trait CanLog[F[_]:Functor, -A]:
  sealed trait Logged[A]
  def log(msg: A): F[Logged[A]] = logU(msg).as[new Logged[A]{}] // надо переделать сигнатуру для контравариантности по A
  def logU(msg: A): F[Unit]
  def comap[-B](f: B => A): CanLog[F, B]
```
(Здесь мы также используем идею квитанций, [описанную ранее](https://habr.com/ru/articles/771946/).)

У нас должна быть какая-то базовая библиотека логирования, позволяющая отправлять `LowLevelMessage` в централизованную систему сбора логов. Т.е., мы исходим из предположения, что в нашем распоряжении имеется экземпляр `CanLog[LowLevelMessage]`:
```scala
given CanLog[LowLevelMessage]
```

# Заключение

В настоящей заметке изложены несколько примеров использования алгебраических типов данных для моделирования различных предметных областей. При использовании ADT для формирования журнала, мы можем получить типобезопасный журнал, содержащий готовые для анализа записи.
