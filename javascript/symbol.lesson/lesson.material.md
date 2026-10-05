# Тип Symbol

Ключом свойства объекта может быть строка или символ. Символ — примитивный тип, значение которого всегда уникально. Он нужен для «скрытых» свойств, которые не конфликтуют с чужими, и для системных настроек поведения объектов. После урока вы сможете создавать символы, использовать их как ключи и пользоваться глобальным реестром.

## Создание и уникальность

Символ создаёт вызов `Symbol()`, необязательный аргумент — описание, нужное для отладки:

```js
const id = Symbol("id");
console.log(typeof id);          // symbol
console.log(id.description);     // id
console.log(Symbol("id") === Symbol("id")); // false
```

Каждый вызов выдаёт новое значение: одинаковые описания не делают символы равными. `new Symbol()` писать нельзя — это `TypeError`, ведь символ примитив, а не объект.

## Символы не превращаются в строки сами

Строковое преобразование для символов не выполняется неявно — это защита от случайного смешения с обычными ключами:

```js
const s = Symbol("tag");
try {
  console.log("метка: " + s);
} catch (e) {
  console.log(e.name); // TypeError
}
console.log(String(s));      // Symbol(tag)
console.log(s.toString());   // Symbol(tag)
console.log(s.description);  // tag
```

Явное `String(s)` разрешено, как и `s.toString()`. Шаблонная строка `${s}` ведёт себя так же, как `+`, и тоже бросает `TypeError`.

## Символы как ключи

Используйте квадратные скобки: точка подразумевает строковое имя.

```js
const id = Symbol("id");
const user = { name: "Ана", [id]: 42 };
console.log(user[id]);  // 42
console.log(user.id);   // undefined
user[id] = 43;
console.log(user[id]);  // 43
```

Если описание совпадёт с обычным ключом `"id"`, конфликта не будет: это два разных свойства. Поэтому символы удобны, когда вы дописываете метаданные в чужой объект — случайно затереть чужое свойство вы не сможете, а чужой код не знает вашего символа.

## Символы скрыты от обычного перебора

Свойства-символы пропускаются `for..in`, `Object.keys`, `Object.values`, `Object.entries` и `JSON.stringify`:

```js
const id = Symbol("id");
const user = { name: "Ана", [id]: 42 };
console.log(Object.keys(user));       // [ 'name' ]
console.log(JSON.stringify(user));    // {"name":"Ана"}
for (const key in user) console.log(key); // name
```

Но «скрытость» условная. Специальные методы видят символы:

```js
console.log(Object.getOwnPropertySymbols(user)); // [ Symbol(id) ]
console.log(Reflect.ownKeys(user));              // [ 'name', Symbol(id) ]
```

Поэтому это защита от случайных коллизий, а не настоящая приватность. Кстати, `Object.assign` и spread копируют символьные свойства — при клонировании теряться они не должны:

```js
const id = Symbol("id");
const copy = Object.assign({}, { [id]: 1 });
console.log(copy[id]); // 1
```

## Глобальный реестр

Иногда нужно, чтобы разные части программы получили один и тот же символ по имени. Для этого есть реестр: `Symbol.for(key)` возвращает символ с ключом `key`, создавая его при первом обращении.

```js
const a = Symbol.for("app.id");
const b = Symbol.for("app.id");
console.log(a === b);                    // true
console.log(Symbol("app.id") === a);     // false
```

Обратная операция: `Symbol.keyFor(sym)` возвращает ключ глобального символа. Для обычного символа результат `undefined` — у него просто нет ключа в реестре.

```js
console.log(Symbol.keyFor(Symbol.for("app.id"))); // app.id
console.log(Symbol.keyFor(Symbol("local")));      // undefined
```

Для любого символа доступно `description` — оно просто берёт текст описания.

## Системные символы

Язык использует «известные» символы, чтобы объект мог настроить встроенное поведение: `Symbol.iterator` (перебор через `for..of`), `Symbol.toPrimitive` (преобразование в примитив), `Symbol.hasInstance` и другие. Они доступны как свойства `Symbol`:

```js
const range = {
  from: 1,
  to: 3,
  *[Symbol.iterator]() {
    for (let i = this.from; i <= this.to; i++) yield i;
  }
};
console.log([...range]); // [ 1, 2, 3 ]
```

Подробнее с ними знакомятся в своих темах; в этом уроке достаточно понимать, что это обычные символы-ключи со встроенным смыслом.

## Типичные ошибки

> [!WARNING]
>
> - Ждать, что `Symbol("x") === Symbol("x")`. Для равенства нужен `Symbol.for`.
> - Обращаться к символьному свойству через точку: `obj.id` вместо `obj[id]`.
> - Конкатенировать символ со строкой. Используйте `String(sym)` или `sym.description`.
> - Считать символьные свойства приватными: `Reflect.ownKeys` их покажет.
> - Забыть, что `JSON.stringify` символьные ключи не сериализует, — при передаче данных они потеряются.

## Коротко

> [!TIP]
>
> - Символ — примитив с гарантированно уникальным значением; создаётся `Symbol(description)`.
> - Ключ-символ задаётся в квадратных скобках, не попадает в `for..in`, `Object.keys`, `JSON.stringify`.
> - Увидеть такие ключи можно через `Object.getOwnPropertySymbols` и `Reflect.ownKeys`.
> - `Symbol.for(key)` читает/создаёт символ в глобальном реестре, `Symbol.keyFor` возвращает ключ.
> - Системные символы (`Symbol.iterator`, `Symbol.toPrimitive`) настраивают поведение объектов.

---

*Тема по мотивам статьи [«Тип данных Symbol»](https://learn.javascript.ru/symbol) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
