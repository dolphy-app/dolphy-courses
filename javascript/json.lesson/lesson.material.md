# JSON и toJSON

Программам постоянно нужно передавать данные друг другу: сохранять в файл, отправлять по сети, класть в хранилище. Для этого нужен текстовый формат, который понимают все языки. Самый распространённый — JSON. В JavaScript его поддерживают два метода: `JSON.stringify` превращает значение в строку, а `JSON.parse` — обратно. После урока вы будете знать, какие значения JSON сохраняет, какие теряет и как настроить сериализацию.

## Что такое JSON

JSON (JavaScript Object Notation) — текстовый формат на основе синтаксиса литералов JavaScript, но с жёсткими правилами. Допустимые значения:

- объекты `{ "key": value }`;
- массивы `[ ... ]`;
- строки, числа, `true`, `false`, `null`.

Отличия от JS-литералов, о которые чаще всего спотыкаются:

- строки и имена свойств — **только в двойных кавычках**;
- нет комментариев, нет завершающих запятых;
- нет `undefined`, функций, символов, `NaN`/`Infinity`, дат — только перечисленные выше типы.

## JSON.stringify

`JSON.stringify(value)` возвращает строку. Результат — JSON-строка, пригодная для передачи или хранения (не «JSON-объект»).

```js
const user = { name: 'Иван', age: 30, tags: ['a', 'b'], active: true };
const json = JSON.stringify(user);
console.log(json); // {"name":"Иван","age":30,"tags":["a","b"],"active":true}
console.log(typeof json); // 'string'
```

Применять можно и к примитивам:

```js
console.log(JSON.stringify('hi')); // "hi"  (с кавычками)
console.log(JSON.stringify(5));    // 5
console.log(JSON.stringify(null)); // null
```

### Что пропускается

Часть значений метод молча теряет или заменяет:

```js
const data = {
  a: 1,
  b: undefined,        // пропускается
  c: () => 1,          // пропускается
  d: Symbol('s'),      // пропускается
  e: NaN,              // становится null
  f: Infinity,         // становится null
  g: new Date(0),      // превращается в строку ISO
  [Symbol('k')]: 1,    // символьные ключи пропускаются
};
console.log(JSON.stringify(data));
// {"a":1,"e":null,"f":null,"g":"1970-01-01T00:00:00.000Z"}
```

В **массивах** такие значения не пропускаются, чтобы не сдвинуть индексы, а заменяются на `null`:

```js
console.log(JSON.stringify([undefined, () => 1, Symbol('s')])); // [null,null,null]
```

Верхнеуровневое `undefined` или функция дают не строку, а `undefined`. Карты и множества превращаются в пустой объект: `JSON.stringify(new Map([[1, 2]]))` — `{}`. А `BigInt` вызывает `TypeError`.

### Циклические ссылки

Если объект ссылается на себя прямо или косвенно, `stringify` выбросит `TypeError`:

```js
const node = { n: 1 };
node.self = node;
// JSON.stringify(node); // TypeError: Converting circular structure to JSON
```

## Отступы и фильтрация

Полный вызов: `JSON.stringify(value, replacer, space)`.

Третий аргумент задаёт отступ для читаемого вывода — число пробелов или строку:

```js
console.log(JSON.stringify({ a: 1, b: [1, 2] }, null, 2));
// {
//   "a": 1,
//   "b": [
//     1,
//     2
//   ]
// }
```

Второй аргумент `replacer` — либо массив имён свойств, которые нужно оставить, либо функция `(key, value)`, результат которой попадает в вывод:

```js
const obj = { a: 1, b: 2, c: 3 };
console.log(JSON.stringify(obj, ['a', 'c'])); // {"a":1,"c":3}

const doubled = JSON.stringify(obj, (key, value) =>
  typeof value === 'number' ? value * 2 : value
);
console.log(doubled); // {"a":2,"b":4,"c":6}
```

Функция вызывается для каждой пары, включая самую первую с пустым ключом `''` и самим значением. Вернуть `undefined` — значит исключить свойство.

## Свой формат через toJSON

Если у объекта есть метод `toJSON`, `stringify` вызовет его и сериализует то, что он вернул. Именно так работают даты (`Date.prototype.toJSON` возвращает ISO-строку). Метод нужен, когда у объекта внутреннее представление неудобно для передачи.

```js
const room = {
  number: 23,
  toJSON() {
    return this.number;
  },
};
console.log(JSON.stringify({ room })); // {"room":23}

const wrapper = { toJSON() { return { k: 1 }; } };
console.log(JSON.stringify([wrapper])); // [{"k":1}]
```

## JSON.parse

`JSON.parse(str)` разбирает строку и возвращает значение. Строка обязана быть строго корректной, иначе выбрасывается `SyntaxError`:

```js
console.log(JSON.parse('[1, 2]'));            // [1, 2]
console.log(JSON.parse('{"a":{"b":1}}'));     // { a: { b: 1 } }
console.log(JSON.parse('"s"'));               // 's'
console.log(JSON.parse('null'));              // null
// JSON.parse("{a:1}")    -> SyntaxError (ключ без кавычек)
// JSON.parse("{'a':1}")  -> SyntaxError (одинарные кавычки)
// JSON.parse('{"a":1,}') -> SyntaxError (лишняя запятая)
```

Пользовательские данные разбирайте в `try...catch`, иначе одна кривая строка остановит программу.

Если в объекте повторяется ключ, побеждает последний: `JSON.parse('{"a":1,"a":2}')` даст `{ a: 2 }`.

### Оживление: reviver

Второй аргумент `reviver(key, value)` вызывается для каждого значения (снизу вверх) и позволяет преобразовать его. Классический пример — восстановить даты, которые в JSON хранятся строками:

```js
const text = '{"when":"2024-01-02T00:00:00.000Z","n":5}';
const parsed = JSON.parse(text, (key, value) =>
  key === 'when' ? new Date(value) : value
);
console.log(parsed.when.getTime()); // 1704153600000
```

## Копирование через JSON

Конструкция `JSON.parse(JSON.stringify(obj))` создаёт глубокую копию простых данных, но теряет всё, что JSON не умеет:

```js
const copy = JSON.parse(JSON.stringify({ a: [1, { b: 2 }], d: new Date(0), u: undefined }));
console.log(copy); // { a: [ 1, { b: 2 } ], d: '1970-01-01T00:00:00.000Z' }
```

Даты стали строками, `undefined` исчез. Для обычных конфигураций — подходит, для «живых» объектов — нет.

## Типичные ошибки

> [!WARNING]
>
> - Путать JSON-строку и объект: у строки нет свойств, пока её не разобрали `JSON.parse`.
> - Писать JSON руками с одинарными кавычками или комментариями.
> - Ждать, что `stringify` сохранит `undefined`, функции, `Map`, `Set` или `Date` как исходные типы.
> - Забывать про циклические ссылки.
> - Не оборачивать `JSON.parse` в `try...catch` для данных извне.

## Коротко

> [!TIP]
>
> - `JSON.stringify` превращает значение в строку, `JSON.parse` — обратно; формат строгий (двойные кавычки, без комментариев).
> - Пропускаются `undefined`, функции, символы (в массивах они дают `null`); `NaN` и `Infinity` — `null`.
> - Третий аргумент `stringify` — отступ, второй — массив ключей или функция-фильтр.
> - `toJSON` определяет, как объект превращается в JSON; на нём работают даты.
> - `reviver` в `parse` восстанавливает типы, например `Date`.
> - Циклические ссылки приводят к `TypeError`, некорректный JSON — к `SyntaxError`.

---

*Тема по мотивам статьи [«JSON и toJSON»](https://learn.javascript.ru/json) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
