---
layout: single
title:  "Квитанции как способ отражения сделанной работы на уровне типов"
date:   2023-11-04 22:47:00 +0300
categories: scala
tags: scala type-level-programming receipt-effect
toc: false
toc_label: "Содержание"
toc_icon: "cog"
---

Функциональное программирование одной из целей ставит отражение логики программы в типах входных/выходных значений функций. Типы аргументов и результатов накладывают существенные ограничения на то, как может быть реализована функция. Тем самым, позволяют делать разумные выводы о работе функции, ориентируясь только на её сигнатуру. Такое явление называется "параметричность". Замечательным примером параметричности служит такая сигнатура:
```scala
val f: [A] => A => A
```

Эту сигнатуру можно прочитать так: для любого типа, получив значение этого типа, вернуть какое-то значение того же типа. Исходя из того, что тип может быть любым, и никаких операций над этим типом мы не определили, единственной продуктивно завершающейся реализацией является `identity`. Здесь и далее мы исключаем непродуктивные решения вида `f(a) = f(a)` (зависание/отсутствие завершения) или `f(a) = throw Exception()` (исключение).

Для представления эффектов часто используется конструкция `IO[A]`. Значение из этого объекта можно получить, только выполнив код, содержащийся внутри. Довольно часто можно столкнуться с ситуацией, когда само значение нам не настолько интересно, как факт выполнения определённой операции. Обычно используется тип возвращаемого значения `IO[Unit]`. В этой заметке предлагается воспользоваться параметричностью, чтобы получить определённые гарантии.

<cut/>

## Гарантия логирования

Допустим, нам требуется сохранять какую-то информацию в журнал. Бизнес-логика работы, очевидно, никак не связана с процессом логирования. 

Разделим приложение на два модуля — библиотека логирования и прикладное приложение. В библиотеке будем возвращать экземпляр типа, который может быть создан только в самой этой библиотеке:

```scala
sealed trait LogReceipt

def log(message: String): IO[LogReceipt] = 
  IO{/* собственно логирование */}
    .as(new LogReceipt{})
```

На уровне приложения, при вызове библиотеки мы получим экземпляр `LogReceipt`. Наличие этого экземпляра гарантирует, что мы обратились к библиотеке и эффект был выполнен. Других способов получения этого экземпляра не существует по построению.

Приложение само является сложным и многослойным. Если по бизнес-требованиям необходимо, чтобы реализация выполнила логирование, то нам достаточно на уровне типов потребовать, чтобы реализация среди прочего вернула экземпляр `LogReceipt`. Этим мы автоматически получим доказательство того, что бизнес-требование выполнено.

## Гарантия сохранения в базу

Если мы задумаемся о том, чтобы аналогичным образом потребовать сохранения в базу, то возникнет некоторое затруднение. Непонятно, что именно будет сохранено. Например, на уровне бизнес-требований необходимо, чтобы была сохранена сущность, а в реализации будет сохраняться что-нибудь малозначительное. Нам необходимо отразить в типе, что именно мы сохраняем.

В библиотеке сделаем тип с дженерик-параметром:
```scala
sealed trait SavedToDBReceipt[A]

def saveToDB[A](a: A): IO[SavedToDBReceipt[A]] =
  IO{/*эффект — сохранение в БД*/}
    .as(new SavedToDBReceipt[A]{})
```

Здесь мы получаем свидетельство того, что в базу сохранена именно та сущность, которая и должна была быть сохранена.

## Альтернативная реализация

Вместо объекта, создаваемого внутри библиотеки, можно использовать идентификатор, сгенерированный базой. Для этого достаточно воспользоваться `opaque` типами:

```scala
opaque type SavedToDBReceipt[A] = Long

def saveToDB[A](a: A): IO[SavedToDBReceipt[A]] =
  IO{/*эффект — сохранение в БД, возвращает идентификатор из БД*/}
    .map(id => id: SavedToDBReceipt[A])

```

Помимо отсутствия лишних аллокаций, наличие идентифкатора, сгенерированного базой данных, служит неоспоримым доказательством того, что операция действительно выполнена.

## Свидетельство нескольких эффектов

В программе может быть необходимость выполнения нескольких эффектов. Например, логирования, сохранения в базу самой сущности и события создания этой сущности.
Причём нам может быть не важно, в каком порядке происходили эффекты.

Накапливать свидетельства можно в обыкновенном tuple. В Scala 3 появился удобный оператор `*:`, наподобие `HList`'а. 

```scala
def create[A](a: A): F[(LogReceipt, SavedToDBReceipt[A], SavedToDBReceipt[Event[A]])] = 
  for
    lr <- log("create")
    sa <- saveToDB(a)
    sea <-saveToDB(event(a, Created))
  yield
    (lr, sa, sea)
```

`Tuple` помимо собственно значений и их типов также сохраняет порядок. Если для уровня бизнес-требований порядок неважен, то можно воспользоваться структурой в пространстве типов — множеством. Один из вариантов такой структуры реализован в библиотеке [type-sets](https://github.com/Primetalk/type-sets):


```scala
def create[A](a: A): F[Set3[LogReceipt, SavedToDBReceipt[A], SavedToDBReceipt[Event[A]]]] = 
  for
    lr <- log("create")
    sa <- saveToDB(a)
    sea <-saveToDB(event(a, Created))
  yield
    Set(lr, sa, sea)
```

В этом случае можно проверить, что в результате получены свидетельства необходимых типов с помощью функции `setEquals`:

```scala
def handle(request): IO[Unit] = 
  for
    a        <- request.as[A]
    receipts <- create(a)
    _        = setEquals[Set3[
      SavedToDBReceipt[A], 
      SavedToDBReceipt[Event[A]], 
      LogReceipt
    ]](receipts)
    _        = setIsASuperset[Set1[LogReceipt]](receipts)
  yeild
    Http.SuccessOk
```

Эта функция проверяет, что в типе предъявленного значения присутствуют все интересующие нас типы. А также, что нет лишних типов. Если допустимо выполнение дополнительных операций, которые нам не важны, мы можем использовать функцию `setIsASuperset`.

(Внимание! упомянутые функции реализованы в пространстве типов, работают на этапе компиляции, имеют сложность `O(n^2)`.)

## Проблема `F[Unit]`

В свете вышеизложенного можно понять, что привычный способ представления эффектов в виде `F[Unit]` обладает существенным недостатком — на уровне типов не отражается существенное явление с точки зрения бизнес-логики. Следовательно, компилятор не защищает нас от ошибок пропуска важного действия. Функция `f(): F[Unit]`, внутри которой имеется логирование, ничем снаружи не отличается от функции `g(): F[Unit]`, в которой такого логирования нет.

Если же мы будем требовать сигнатуру вида `def serve(f: () => F[LogReceipt])`, то такому требованию может удовлетворить только функция, на самом деле выполняющая требуемый эффект.

## Взаимодействие с "полномочиями" (capabilities)

Мартин Одерски ввёл в оборот идею контекстных функций. Такие функции принимают на вход несколько контекстных параметров, которыми в дальнейшем могут пользоваться для каких-то целей. Мы можем совместить квитанции с полномочиями и получить гарантию, что код воспользуется предоставленными полномочиями хотя бы раз.

```scala
val f: [A] => ((a: A) => F[SavedToDB[A]]) ?=> F[SavedToDB[A]]
```

(Естественно, результат может содержать что-то ещё, помимо квитанции.)

# Заключение

В настоящей заметке кратко изложена идея использования квитанций для подтверждения выполненной работы. Обычное представление "результата" эффекта в виде `Unit`'а, не позволяет на верхнем уровне приложения убедиться, что все необходимые эффекты были на самом деле выполнены.

При наличии экземпляра квитанции, мы можем быть уверены, что в ходе работы программы все требуемые эффекты были выполнены.
