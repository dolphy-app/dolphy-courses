# Типы данных

Значение в JavaScript всегда относится к какому-то типу. Тип определяет, что с ним можно делать: числа складывают, строки склеивают, объекты хранят свойства. Переменная типа не имеет — тип есть у значения, поэтому одна переменная может поочерёдно хранить значения разных типов.

В языке восемь типов: семь **примитивных** (`number`, `bigint`, `string`, `boolean`, `null`, `undefined`, `symbol`) и один — `object`.

## Number

Тип `number` хранит и целые, и дробные числа. Поддерживаются записи вроде `1e3`, `0xff`, `0b101` и разделители разрядов `1_000_000`.

```js
console.log(5 / 2);      // 2.5
console.log(1e3);        // 1000
console.log(0xff);       // 255
console.log(1_000_000);  // 1000000
```

Кроме обычных чисел есть три особых значения:

```js
console.log(1 / 0);      // Infinity
console.log(-1 / 0);     // -Infinity
console.log('abc' / 2);  // NaN
```

`NaN` (Not a Number) означает ошибку вычисления, и любая операция с ним тоже даёт `NaN`. Главная странность: `NaN` не равен даже сам себе, поэтому проверять его нужно функцией `Number.isNaN`.

```js
console.log(NaN === NaN);       // false
console.log(Number.isNaN(NaN)); // true
console.log(NaN + 1);           // NaN
```

Числа хранятся в двоичном формате с плавающей точкой (64 бита), поэтому не все десятичные дроби представимы точно, а целые безопасны только до `Number.MAX_SAFE_INTEGER`:

```js
console.log(0.1 + 0.2);            // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);    // false
console.log(Number.MAX_SAFE_INTEGER); // 9007199254740991
console.log(2 ** 53 + 1);          // 9007199254740992
```

Это не ошибка JavaScript — так работают числа с плавающей точкой в любом языке. Для денег хранят целые копейки или используют специальные библиотеки.

## BigInt

Для целых любого размера есть `bigint`. Литерал — число с `n` на конце. Смешивать `bigint` и `number` в арифметике нельзя.

```js
console.log(9007199254740991n * 2n); // 18014398509481982n
console.log(7n / 2n);                // 3n
console.log(typeof 10n);             // bigint
console.log(1n + 1);                 // TypeError: Cannot mix BigInt and other types, use explicit conversions
```

## String

Строка — текст в кавычках. Есть три вида кавычек: одинарные и двойные равноправны, а обратные (`` ` ``) позволяют вставлять выражения через `${...}` и писать текст на нескольких строках.

```js
const name = 'Аня';
console.log('Привет, ' + name);        // Привет, Аня
console.log(`Привет, ${name}! ${1 + 2}`); // Привет, Аня! 3
console.log('Привет, ${name}');        // Привет, ${name}
console.log('abc'.length);             // 3
```

Длина строки — свойство `length`. У JavaScript нет отдельного типа «символ»: символ — это строка из одного знака.

## Boolean, null, undefined

`boolean` принимает значения `true` и `false` и получается, как правило, в результате сравнений.

`null` — «пустое значение», которое программист присваивает намеренно: «здесь ничего нет». `undefined` — «значение не присвоено»: так выглядит переменная, объявленная без значения.

```js
let notAssigned;
const nothing = null;
console.log(notAssigned); // undefined
console.log(nothing);     // null
console.log(2 > 1);       // true
```

## Symbol и object

`symbol` создаёт уникальный идентификатор: два символа с одинаковым описанием не равны. Он нужен для скрытых ключей свойств; подробнее — позже.

```js
console.log(Symbol('id') === Symbol('id')); // false
```

Все остальные значения — объекты: словари, массивы, функции, даты. Примитивы хранят одно значение и неизменяемы, а объекты — составные структуры. Их мы разберём в отдельных уроках.

## Оператор typeof

`typeof` возвращает строку с названием типа:

```js
console.log(typeof 1);          // number
console.log(typeof 1n);         // bigint
console.log(typeof 'a');        // string
console.log(typeof true);       // boolean
console.log(typeof undefined);  // undefined
console.log(typeof Symbol('x'));// symbol
console.log(typeof {});         // object
```

Три случая, о которых нужно знать:

```js
console.log(typeof null);        // object (историческая ошибка языка)
console.log(typeof [1, 2]);      // object (массив — тоже объект)
console.log(typeof function () {}); // function (отдельного типа нет, но typeof различает)
```

Чтобы отличить массив, используют `Array.isArray`, а `null` проверяют сравнением `value === null`.

## Типичные ошибки

> [!WARNING]
>
> - Проверять число на `NaN` через `===`.
> - Ждать, что `0.1 + 0.2` равно `0.3`.
> - Определять `null` через `typeof`: он вернёт `'object'`.
> - Смешивать `bigint` и `number` в выражении.
> - Путать `null` и `undefined`: первое присваивают намеренно, второе означает «не присвоено».

## Коротко

> [!TIP]
>
> - Восемь типов: семь примитивов и `object`.
> - `number` включает `Infinity`, `-Infinity` и `NaN`; дроби неточны.
> - `bigint` — для больших целых, с `number` не смешивается.
> - Строки: три вида кавычек, обратные позволяют `${...}`.
> - `typeof null` — `'object'`, `typeof` функции — `'function'`.

---

*Тема по мотивам статьи [«Типы данных»](https://learn.javascript.ru/types) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
