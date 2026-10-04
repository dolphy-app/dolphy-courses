# Флаги и дескрипторы свойств

Свойство объекта — не просто пара «ключ — значение». У каждого свойства есть три скрытых настройки (флаги), которые определяют, можно ли его менять, перебирать и удалять. Обычно они включены, и мы их не замечаем. В этом уроке — как их читать, менять и как с их помощью защищать объекты.

## Три флага

- `writable` — значение можно изменять; если `false`, свойство только для чтения.
- `enumerable` — свойство попадает в перебор (`for...in`, `Object.keys`, `JSON.stringify`, spread).
- `configurable` — свойство можно удалить, а также менять его флаги и тип.

У свойств, созданных обычным присваиванием или литералом, все три равны `true`.

## Чтение: getOwnPropertyDescriptor

Вместе со значением флаги образуют **дескриптор** свойства:

```js
const user = { name: 'Ира' };
console.log(Object.getOwnPropertyDescriptor(user, 'name'));
// { value: 'Ира', writable: true, enumerable: true, configurable: true }
```

Для всех собственных свойств сразу есть `Object.getOwnPropertyDescriptors(obj)`.

## Запись: defineProperty

`Object.defineProperty(obj, prop, descriptor)` создаёт свойство или изменяет существующее. Важная деталь: для **нового** свойства не указанные флаги считаются `false`:

```js
const item = {};
Object.defineProperty(item, 'id', { value: 7 });
console.log(Object.getOwnPropertyDescriptor(item, 'id'));
// { value: 7, writable: false, enumerable: false, configurable: false }
```

Для **существующего** свойства меняются только указанные поля, остальные сохраняются.

## writable: false

Присвоение в обычном режиме молча игнорируется, в строгом (`'use strict'`, модули, классы) — бросает `TypeError`:

```js
const point = {};
Object.defineProperty(point, 'x', { value: 1, enumerable: true });
point.x = 99;
console.log(point.x); // 1

(function () {
  'use strict';
  try {
    point.x = 99;
  } catch (err) {
    console.log(err.name); // TypeError
  }
})();
```

Молчаливое игнорирование — частый источник путаницы: код не падает, но и не работает.

## enumerable: false

Скрытые от перебора свойства есть у встроенных объектов: например, `toString` у `Object.prototype`, поэтому он не виден в `for...in`. Своё скрытое свойство делают так:

```js
const account = { owner: 'Олег' };
Object.defineProperty(account, 'secret', { value: 'abc', enumerable: false });
console.log(Object.keys(account)); // [ 'owner' ]
console.log(JSON.stringify(account)); // {"owner":"Олег"}
console.log(account.secret); // abc
```

Прочитать такое свойство можно по имени, а вот перебор и `JSON.stringify` его пропускают. Список вообще всех собственных строковых ключей вернёт `Object.getOwnPropertyNames`.

## configurable: false

Такое свойство нельзя удалить, нельзя изменить тип (данные ↔ аксессор), нельзя включить обратно `enumerable` или `configurable`. Единственное исключение: `writable` можно перевести из `true` в `false` (и значение изменить, если `writable` ещё `true`). Это односторонняя дверь:

```js
const cfg = { limit: 10 };
Object.defineProperty(cfg, 'limit', { configurable: false });

delete cfg.limit;
console.log(cfg.limit); // 10

try {
  Object.defineProperty(cfg, 'limit', { enumerable: false });
} catch (err) {
  console.log(err.name); // TypeError
}

Object.defineProperty(cfg, 'limit', { writable: false }); // допустимо
```

Пример из стандартной библиотеки: `Math.PI` — `writable: false, configurable: false`.

## defineProperties и клонирование с флагами

Несколько свойств можно задать за раз через `Object.defineProperties(obj, descriptors)`. А `Object.getOwnPropertyDescriptors` вместе с ним даёт «глубокую» копию, сохраняющую флаги и аксессоры (то, что теряет `{ ...obj }` и `Object.assign`):

```js
const src = {};
Object.defineProperty(src, 'hidden', { value: 1, enumerable: false });
const spread = { ...src };
const full = Object.defineProperties({}, Object.getOwnPropertyDescriptors(src));
console.log(Object.getOwnPropertyNames(spread)); // []
console.log(Object.getOwnPropertyNames(full)); // [ 'hidden' ]
```

## Запечатывание объекта целиком

Флаги управляют одним свойством. Для целого объекта есть методы:

| Метод | Добавлять свойства | Удалять | Менять значения |
|---|---|---|---|
| `Object.preventExtensions(obj)` | нет | да | да |
| `Object.seal(obj)` | нет | нет | да |
| `Object.freeze(obj)` | нет | нет | нет |

Проверяют состояние `Object.isExtensible`, `Object.isSealed`, `Object.isFrozen`. Заморозка **поверхностная**: вложенные объекты остаются изменяемыми.

```js
const frozen = Object.freeze({ a: 1, nested: { b: 2 } });
frozen.a = 100;
frozen.nested.b = 200;
console.log(frozen.a, frozen.nested.b); // 1 200
console.log(Object.isFrozen(frozen)); // true
```

## Типичные ошибки

- Думать, что в `defineProperty` пропущенные флаги остаются `true`: у нового свойства они `false`.
- Не заметить, что запись в `writable: false` в обычном режиме молча не срабатывает.
- Рассчитывать, что `Object.freeze` защищает вложенные объекты.
- Пытаться «вернуть» `configurable: true`: это невозможно.

## Коротко

- Флаги свойства: `writable`, `enumerable`, `configurable`; по умолчанию для обычных свойств все `true`, для `defineProperty` — `false`.
- `getOwnPropertyDescriptor(s)` читает, `defineProperty/-ies` пишет.
- Нарушение `writable: false` в строгом режиме — `TypeError`, в обычном — тишина.
- `configurable: false` необратим (кроме перевода `writable` в `false`).
- `preventExtensions`, `seal`, `freeze` ограничивают объект целиком; заморозка поверхностная.

---

*Тема по мотивам статьи [«Флаги и дескрипторы свойств»](https://learn.javascript.ru/property-descriptors) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
