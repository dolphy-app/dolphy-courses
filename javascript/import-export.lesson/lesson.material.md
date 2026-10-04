# Экспорт и импорт

Модуль полезен, пока он что-то отдаёт и что-то берёт. Эти две операции задаются директивами `export` и `import`. Синтаксис у них гибкий: есть именованные и «по умолчанию» экспорты, переименование, импорт всего модуля разом и реэкспорт. Ниже — все варианты и подводные камни.

## Именованный экспорт

`export` можно поставить перед объявлением:

```js
// shapes.mjs
export const PI = 3;
export let area = 0;
export function square(side) { return side * side; }
export class Box {}
```

Либо отдельным списком в любом месте файла, обычно в конце:

```js
const PI = 3;
function square(side) { return side * side; }
export { PI, square };
```

Точка с запятой после `export function`/`export class` не нужна — это объявления, а не выражения.

## Именованный импорт

```js
import { PI, square } from './shapes.mjs';
console.log(square(4) * PI); // 48
```

Имена в фигурных скобках должны **совпадать** с экспортированными. Нет такого имени — `SyntaxError` ещё до запуска кода. Если нужно много имён, удобнее импортировать всё сразу в объект-пространство имён:

```js
import * as shapes from './shapes.mjs';
console.log(shapes.square(3)); // 9
```

Пространство имён — особый объект: прототип `null`, ключи отсортированы по алфавиту, свойства недоступны для присваивания, значения «живые».

```js
import * as ns from './counter.mjs'; // экспортирует PI, count, inc, default
console.log(Object.keys(ns)); // [ 'PI', 'count', 'default', 'inc' ]
console.log(Object.getPrototypeOf(ns)); // null
console.log(Object.prototype.toString.call(ns)); // [object Module]
```

Современные сборщики умеют убирать из итогового пакета неиспользуемые именованные экспорты (tree shaking) — это одна из причин не импортировать «всё подряд», когда нужна пара функций.

## Переименование: `as`

И при экспорте, и при импорте имя можно поменять:

```js
// lib.mjs
function internalName() { return 1; }
export { internalName as publicName };

// main.mjs
import { publicName as short } from './lib.mjs';
console.log(short()); // 1
```

Приём помогает разрешать конфликты имён: два модуля экспортируют `format` — один импортируйте как `formatDate`, другой как `formatMoney`.

## Экспорт по умолчанию

Модуль, который описывает одну сущность (класс, функцию), удобно оформить с `export default`. У модуля может быть **только один** default.

```js
// user.mjs
export default class User {}
export const role = 'guest';
```

При импорте default берётся **без** фигурных скобок и под любым именем:

```js
import User from './user.mjs';
import Person, { role } from './user.mjs'; // default + именованный
console.log(typeof User, role); // function guest
```

`export default` допускает безымянные функции и классы (`export default function () {}`), а также выражения: `export default 42`. Но default на самом деле — обычный экспорт с именем `default`, и обратиться к нему можно явно:

```js
import { default as U, role } from './user.mjs';
import * as u from './user.mjs';
console.log(typeof U, typeof new u.default()); // function object
```

Какой стиль выбрать? Именованные экспорты фиксируют имя: опечатка сразу даёт ошибку, редактор подсказывает, поиск по проекту находит. Default позволяет каждому называть как хочет, из-за чего одна сущность получает разные имена в разных файлах. Многие командные гайдлайны требуют именованные экспорты, а default оставляют для «одна сущность на файл».

## Живые привязки и запрет присваивания

Импорт — это ссылка на переменную экспортёра, а не копия. Если модуль меняет `export let`, импортёры видят новое значение:

```js
// counter.mjs
export let count = 0;
export function inc() { count++; }

// main.mjs
import { count, inc } from './counter.mjs';
console.log(count); // 0
inc();
console.log(count); // 1
```

Присвоить в импортированное имя нельзя:

```js
try {
  count = 5;
} catch (e) {
  console.log(e.constructor.name, e.message); // TypeError Assignment to constant variable.
}
```

Единственный, кто может менять значение, — модуль-владелец. Хотите менять извне — экспортируйте функцию-сеттер (как `inc` выше) или объект, свойства которого можно менять.

## Реэкспорт

Модуль-«витрина» собирает экспорты из других файлов, не импортируя их для себя:

```js
// index.mjs
export * from './user.mjs';                 // все именованные
export { default as Person } from './user.mjs'; // default — явно, под именем
export { role as defaultRole } from './user.mjs';
```

Важный нюанс: `export * from` **не** реэкспортирует default. Поэтому

```js
import d, { role } from './index.mjs';
// SyntaxError: The requested module './index.mjs' does not provide an export named 'default'
```

Нужно писать `export { default } from './user.mjs'` или `export { default as Person } from ...`. А если два `export *` приносят одно и то же имя из разных модулей, оно считается неоднозначным и не экспортируется (импорт такого имени — `SyntaxError`).

## Типичные ошибки

- Писать `import { User } from` для default-экспорта — такого именованного экспорта нет.
- Ждать, что `export *` вынесет default.
- Присваивать в импортированное имя; для изменения нужна функция в модуле-владельце.
- Забыть расширение в Node: в ES-модулях Node нужен полный путь `./file.mjs` (в отличие от CommonJS и сборщиков).
- Ставить `export`/`import` внутри блока или функции: допустим только верхний уровень.

## Коротко

- `export` бывает именованным (любое число) и `default` (не более одного).
- Именованный импорт — в фигурных скобках с точными именами; default — без скобок, имя любое.
- `as` переименовывает при экспорте и импорте, `* as ns` собирает пространство имён.
- Импортированные имена — живые привязки только для чтения.
- `export * from` пересылает все именованные экспорты, кроме default.

---

*Тема по мотивам статьи [«Экспорт и импорт»](https://learn.javascript.ru/import-export) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
