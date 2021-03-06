# Совместимость типов

## Введение

Совместимость типов в TypeScript основывается на структурной типизации.
Структурная типизация — это способ выявления отношений типов на основании исключительно состава их членов.
Этот подход отличается от номинативной типизации.
Посмотрим на следующий код:

```ts
interface Named {
  name: string
}

class Person {
  name: string
}

let p: Named
// Все подходит, поскольку используется структурная система типов
p = new Person()
```

В языках, подобных C# и Java, где используется номинативная система типов, аналогичный код привел бы к ошибке, поскольку класс `Person` не описывается явно, как реализующий интерфейс `Named`.

Структурная система типов TypeScript была спроектирована с учетом того, как обычно пишется код на JavaScript.
Поскольку в JavaScript широко используются анонимные объекты, такие как функциональные выражения и литералы объектов, гораздо более естественно будет описывать их отношения с помощью структурной системы, а не номинативной.

### Замечание относительно надежности

Система типов TypeScript допускает некоторые операции, относительно которых во время компиляции невозможно сказать, безопасны ли они. Когда система типов обладает подобным свойством, говорят, что она не является "надежной".
Места, где TypeScript допускает ненадежное поведение, были тщательно обдуманы, и в данной главе мы объясним, где это происходит, и по какой причине было позволено.

## Для начала

Основное правило системы типов TypeScript таково — `x` совместимо с `y`, если `y` имеет по крайней мере те же самые члены, что и `x`. К примеру:

```ts
interface Named {
  name: string
}

let x: Named
// выведенный для y тип — { name: string; location: string; }
let y = { name: 'Alice', location: 'Seattle' }
x = y
```

Чтобы понять, может ли `y` быть присвоена `x`, компилятор для каждого из свойств `x` ищет соответствующее совместимое свойство в `y`.
В данном случае переменная `y` должна иметь свойство под именем `name` строкового типа. Оно есть, и присваивание допускается.

То же самое правило используется в случае проверки аргументов при вызове функции:

```ts
function greet(n: Named) {
  alert('Привет, ' + n.name)
}
greet(y) // ОК
```

Обратите внимание, что `y` обладает дополнительным свойством `location`, но это не приводит к ошибке.
При проверке на совместимость учитываются только члены целевого типа (в данном случае это `Named`).

Процесс сравнения производится рекурсивно, затрагивая типы всех членов и подчленов.

## Сравнение двух функций

Сравнение типов двух примитивов или объектов происходит относительно просто, однако вопрос о том, какие функции должны считаться совместимыми, немного более сложен.
Начнем с простого примера с двумя функциями, отличающимися только списками параметров:

```ts
let x = (a: number) => 0
let y = (b: number, s: string) => 0

y = x // Все нормально
x = y // Ошибка
```

Чтобы проверить, допустимо ли присваивание `x` к `y`, сначала просматривается список параметров.
Для каждого параметра функции `x` у функции `y` должен быть соответствующий параметр совместимого типа.
Имена параметров не принимаются во внимание — важны лишь типы.
В данном случае для каждого параметра `x` есть соответствующий совместимый параметр в функции `y`, поэтому присваивание допускается.

Второе присваивание приводит к ошибке, поскольку `y` имеет обязательный второй параметр, которого нет у `x`, и операция не допускается.

Может показаться интересным, почему разрешается "терять" параметры функции, как это происходит при `y = x`.
Причина этому то, что игнорирование лишних параметров функции — довольно частая практика в JavaScript.
К примеру, `Array#forEach` передает функции обратного вызова три параметра: элемент массива, его индекс, и массив, в котором тот содержится.
Несмотря на это, очень удобно работать с функцией обратного вызова, которая использует лишь первый параметр:

```ts
let items = [1, 2, 3]

// Не заставлять использовать дополнительные параметры
items.forEach((item, index, array) => console.log(item))

// Все должно работать!
items.forEach((item) => console.log(item))
```

Теперь посмотрим, как обрабатываются типы возвращаемых значений. Для этого используем две функции, отличающиеся только типами возвращаемых значений:

```ts
let x = () => ({ name: 'Alice' })
let y = () => ({ name: 'Alice', location: 'Seattle' })

x = y // Работает
y = x // Ошибка, поскольку у x() нет свойства location
```

Необходимо, чтобы тип возвращаемого значения исходной функции был подтипом типа возвращаемого значения целевой функции.

### Бивариантность параметров функции

При сравнении типов параметров присваивание допускается, если параметр исходной функции может быть присвоен параметру целевой функции, или наоборот.
Это не является надежным, поскольку код может получить функцию, которая принимает более специализированный тип, и передать ей значение менее специализированного типа.
На практике такого рода ошибки редки, а допущение подобного позволяет использовать многие распространенные практики из JavaScript. Краткий пример:

```ts
enum EventType {
  Mouse,
  Keyboard,
}

interface Event {
  timestamp: number
}
interface MouseEvent extends Event {
  x: number
  y: number
}
interface KeyEvent extends Event {
  keyCode: number
}

function listenEvent(
  eventType: EventType,
  handler: (n: Event) => void
) {
  /* ... */
}

// Ненадежно, но полезно и часто используется
listenEvent(EventType.Mouse, (e: MouseEvent) =>
  console.log(e.x + ',' + e.y)
)

// Альтернативы, нежелательные из-за ненадежности
listenEvent(EventType.Mouse, (e: Event) =>
  console.log((<MouseEvent>e).x + ',' + (<MouseEvent>e).y)
)
listenEvent(EventType.Mouse, <(e: Event) => void>(
  ((e: MouseEvent) => console.log(e.x + ',' + e.y))
))

// Не допускается (явная ошибка). Требуется безопасность типов для полностью несовместимых типов
listenEvent(EventType.Mouse, (e: number) => console.log(e))
```

## Опциональные и остаточные параметры

При проверке функций на совместимость опциональные и обязательные параметры взаимозаменяемы.
Лишние опциональные параметры исходного типа не приводят к ошибке, так же как и опциональные параметры целевого типа, для которых нет соответствующих параметров.

Когда у функции есть остаточный параметр, он расценивается так, словно представляет собой бесконечное число опциональных параметров.

С точки зрения системы типов это не является надежным, но с точки зрения выполняющегося кода сами опциональные параметры, как правило, не являются чем-то четко определенным, поскольку для большинства функций они эквивалентны передаче `undefined`.

В пример полезности этого приведем распространенный прием — функцию, которая принимает функцию обратного вызова и вызывает ее с некоторым предсказуемым (для разработчика), но неизвестным (для системы типов) числом аргументов:

```ts
function invokeLater(
  args: any[],
  callback: (...args: any[]) => void
) {
  /* ... Вызвать функцию в аргументами `args` ... */
}

// Ненадежно — invokeLater может получить любое число аргументов
invokeLater([1, 2], (x, y) => console.log(x + ', ' + y))

// Сбивает с толку (x и y на самом деле необходимы), и незаметно
invokeLater([1, 2], (x?, y?) => console.log(x + ', ' + y))
```

### Функции с перегрузками

Когда у функции есть перегрузки, для каждой из перегрузок исходного типа у целевого типа должна найтись совместимая сигнатура.
Это гарантирует, что целевая функция может быть вызвана в каждой из тех ситуаций, в которых может быть вызвана исходная функция.

## Перечисления

Перечисления совместимы с числами, а числа совместимы с перечислениями. Значения из различных перечислений считаются несовместимыми друг с другом. К примеру,

```ts
enum Status {
  Ready,
  Waiting,
}
enum Color {
  Red,
  Blue,
  Green,
}

let status = Status.Ready
status = Color.Green // ошибка
```

## Классы

Классы работают подобно типам объектных литералов и интерфейсам, но с одним исключением: у них есть тип статической части и тип экземпляра.
При сравнении двух объектов, которые имеют классовый тип, сравниваются только члены экземпляра.
Статические члены и конструкторы не влияют на совместимость.

```ts
class Animal {
  feet: number
  constructor(name: string, numFeet: number) {}
}

class Size {
  feet: number
  constructor(numFeet: number) {}
}

let a: Animal
let s: Size

a = s //OK
s = a //OK
```

### Приватные и защищенные члены классов

Приватные и защищенные члены классов влияют на их совместимость.
Когда экземпляр класса проходит проверку на совместимость, то, если у него есть приватный член, у целевого типа тоже должен быть приватный член, объявленный в том же классе.
Это относится и к экземплярам с защищенными членами.
Такая особенность позволяет классам быть совместимыми при присваивании базовому классу, но не классу из другой иерархии наследования, даже если бы он имел такую же форму.

## Обобщения

Поскольку в TypeScript используется структурная система типов, типовые параметры влияют на получаемый тип только в том случае, если используются как часть типа члена. К примеру,

```ts
interface Empty<T> {}
let x: Empty<number>
let y: Empty<string>

x = y // все в порядке, у подходит по структуре к x
```

В вышеприведенном примере `x` и `y` совместимы, поскольку типовый параметр не используется так, чтобы между ними возникла разница.
Изменим этот пример, добавив член к `Empty<T>`:

```ts
interface NotEmpty<T> {
  data: T
}
let x: NotEmpty<number>
let y: NotEmpty<string>

x = y // ошибка, x и y несовместимы
```

В этом случае обобщенный тип, для которого указаны типовые аргументы, ведет себя так же, как и обычный, необобщенный тип.

В случае обобщенных типов, для которых не указаны типовые аргументы, совместимость проверяется, словно в качестве пропущенных типовых аргументов был указан `any`.
Затем полученные типы проверяются на совместимость, так же, как и обычные.

К примеру,

```ts
let identity = function<T>(x: T): T {
    // ...
}

let reverse = function&lt;U&gt;(y: U): U {
    // ...
}

identity = reverse;  // Все хорошо, так как (x: any)=>any совпадает с (y:any)=>any
```

## Продвинутые темы

### Подтипы и присваивание

Все это время мы использовали термин "совместимость", хотя он и не определен в спецификации языка.
В TypeScript существует два вида совместимости: совместимость подтипов и совместимость при присваивании.
Различие между ними лишь в том, что совместимость при присваивании расширяет совместимость подтипов правилами, позволяющими присваивать что-либо к `any` и `any` к чему-либо, а также перечисления и соответствующие числовые значения друг другу.

В различных местах компилятор использует тот или иной механизм проверки совместимости в зависимости от ситуации.
Из практических соображений совместимость типов диктуется совместимостью при присваивании, в том числе в конструкциях `implements` и `extends`.
За более подробной информацией обращайтесь к [спецификации TypeScript](https://github.com/Microsoft/TypeScript/blob/master/doc/spec.md).

## Ссылки

- [Совместимость типов](http://typescript-lang.ru/docs/Type%20Compatibility.html)
