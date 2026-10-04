# Методы прототипов

Свойство `__proto__` — устаревший способ работать с прототипом. В современном JavaScript для этого есть полноценные функции: `Object.create`, `Object.getPrototypeOf`, `Object.setPrototypeOf`, а также `Object.hasOwn` для проверки собственных свойств. В уроке разберём их, научимся копировать объекты вместе с прототипом и создавать «чистые» словари без прототипа.

## Основные функции

| Функция | Что делает |
|---|---|
| `Object.create(proto, descriptors?)` | создаёт пустой объект с `[[Prototype]] = proto` (объект или `null`) |
| `Object.getPrototypeOf(obj)` | возвращает прототип объекта |
| `Object.setPrototypeOf(obj, proto)` | заменяет прототип существующего объекта |
| `Object.hasOwn(obj, key)` | `true`, если свойство `key` собственное |

```js
const animal = {
  eats: true,
  describe() {
    return this.eats ? 'ест' : 'не ест';
  },
};

const rabbit = Object.create(animal, {
  jumps: { value: true, enumerable: true },
});

console.log(rabbit.describe()); // ест
console.log(rabbit.jumps); // true
console.log(Object.getPrototypeOf(rabbit) === animal); // true
console.log(Object.keys(rabbit)); // [ 'jumps' ]
```

Второй аргумент `Object.create` — дескрипторы свойств, как у `Object.defineProperties`. Важно: если не указать `enumerable: true`, свойство окажется неперечислимым.

## Клонирование вместе с прототипом

Часто `Object.assign({}, obj)` или спред `{ ...obj }` достаточно, но они копируют только собственные перечислимые свойства, и результат получает обычный `Object.prototype`. Чтобы сделать клон с тем же прототипом, воспользуйтесь `Object.create` и `Object.getOwnPropertyDescriptors`: так копируются и неперечислимые свойства, и геттеры/сеттеры.

```js
const proto = { hello: () => 'hi' };
const source = Object.create(proto);
source.a = 1;
Object.defineProperty(source, 'hidden', { value: 2, enumerable: false });

const shallow = { ...source };
const full = Object.create(
  Object.getPrototypeOf(source),
  Object.getOwnPropertyDescriptors(source),
);

console.log(Object.getPrototypeOf(shallow) === proto); // false
console.log(shallow.hidden); // undefined
console.log(Object.getPrototypeOf(full) === proto); // true
console.log(full.hidden); // 2
```

Клон остаётся поверхностным: вложенные объекты общие.

## Объект как словарь: проблема __proto__

Объекты часто используют как словари «ключ → значение». Тут и подстерегает неприятность: обычный объект наследует `Object.prototype`, поэтому `'toString' in dict` истинно даже в пустом словаре, а присваивание ключа `__proto__` меняет прототип, а не добавляет ключ.

```js
const dict = {};

console.log('toString' in dict); // true
console.log(Object.hasOwn(dict, 'toString')); // false

dict['__proto__'] = { injected: 1 };
console.log(Object.keys(dict)); // []
console.log(dict.injected); // 1
```

В последних двух строках ключ не появился, зато «появилось» чужое свойство `injected`. Для данных от пользователя это источник ошибок (и уязвимостей — загрязнения прототипа).

### Объект без прототипа

Решение — словарь на `Object.create(null)`. У него нет прототипа, поэтому нет ни `toString`, ни особого `__proto__`: это обычный ключ.

```js
const safe = Object.create(null);
safe['__proto__'] = 'просто строка';
safe.toString = 1;

console.log(Object.keys(safe)); // [ '__proto__', 'toString' ]
console.log('hasOwnProperty' in safe); // false
console.log(Object.getPrototypeOf(safe)); // null
```

Обратная сторона: у такого объекта нет и `hasOwnProperty`, `toString` и прочих методов, так что `safe.hasOwnProperty('x')` упадёт с `TypeError`. Используйте функции: `Object.hasOwn(safe, 'x')`, `Object.keys(safe)`.

Если нужны произвольные ключи, часто лучше другой инструмент — `Map`. Но для JSON-подобных данных и для небольших внутренних словарей `Object.create(null)` вполне подходит.

## Что делает __proto__

`__proto__` — геттер/сеттер, лежащий в `Object.prototype`. Он работает во всех современных средах (в браузерах — по стандарту, в Node.js — тоже), но:

- не работает на объектах без прототипа: геттер лежит в `Object.prototype`, а такие объекты его не наследуют;
- в литерале `{ __proto__: obj }` — особая запись, задающая прототип; вычисляемый ключ `['__proto__']` особым не считается;
- новые программы должны использовать функции из таблицы выше.

```js
const base = { a: 1 };
const withLiteral = { __proto__: base };
const withComputed = { ['__proto__']: base };

console.log(Object.getPrototypeOf(withLiteral) === base); // true
console.log(Object.getPrototypeOf(withComputed) === base); // false
console.log(Object.hasOwn(withComputed, '__proto__')); // true
```

## Типичные ошибки

> [!WARNING]
>
> - Менять прототип объекта через `Object.setPrototypeOf` в горячем коде: движки оптимизируют объекты по форме, и смена прототипа тормозит доступ к свойствам.
> - Вызывать `obj.hasOwnProperty(key)` у объектов из внешних источников: метод может быть затёрт или отсутствовать. Надёжнее `Object.hasOwn(obj, key)`.
> - Использовать `{}` как словарь с произвольными ключами.
> - Копировать объект спредом и ожидать сохранения прототипа и геттеров.

## Коротко

> [!TIP]
>
> - `Object.create(proto)` создаёт объект с нужным прототипом; `getPrototypeOf` / `setPrototypeOf` читают и меняют его.
> - `Object.hasOwn(obj, key)` проверяет только собственные свойства и безопасен для объектов без прототипа.
> - Полный клон: `Object.create(getPrototypeOf(o), getOwnPropertyDescriptors(o))`.
> - Спред и `Object.assign` теряют прототип, неперечислимые свойства и превращают геттеры в значения.
> - `Object.create(null)` — «чистый» словарь: ни унаследованных ключей, ни особого `__proto__`.

---

*Тема по мотивам статьи [«Методы прототипов, объекты без свойства __proto__»](https://learn.javascript.ru/prototype-methods) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
