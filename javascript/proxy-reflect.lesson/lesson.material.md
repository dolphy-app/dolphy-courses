# Proxy и Reflect

`Proxy` — это объект-посредник. Он оборачивает другой объект (цель, *target*) и перехватывает операции над ним: чтение и запись свойств, проверку `in`, удаление, вызов функции, `new` и другие. Для каждой операции вы можете решить, что делать: переслать её цели, изменить, запретить. Вместе с `Reflect` — набором функций, повторяющих «стандартное» поведение этих операций, — прокси позволяет делать валидацию, логирование, значения по умолчанию, скрытые свойства и реактивность.

## Создание прокси

```js
const proxy = new Proxy(target, handler);
```

- `target` — оборачиваемый объект (функции и массивы тоже подходят);
- `handler` — объект с методами-перехватчиками (*traps*): `get`, `set`, `has`, `deleteProperty`, `ownKeys`, `apply`, `construct` и т. д.

Если для операции ловушки нет, она прозрачно пересылается цели:

```js
const target = { a: 1 };
const p = new Proxy(target, {});
p.c = 3;
console.log(target.c); // 3
console.log(p === target); // false
```

Прокси — отдельный объект: он не равен цели, а значит, `Set`, `Map` и сравнения через `===` различают их. Для кода снаружи прокси выглядит как цель: `Array.isArray(new Proxy([], {}))` даёт `true`, а `typeof` прокси функции — `"function"`.

## Ловушка get: чтение

Сигнатура: `get(target, property, receiver)`. Например, словарь, который вместо `undefined` возвращает подсказку:

```js
const dict = { hello: 'Привет' };
const d = new Proxy(dict, {
  get(t, key, receiver) {
    return key in t ? Reflect.get(t, key, receiver) : `<${String(key)}>`;
  },
});
console.log(d.hello, d.bye); // Привет <bye>
```

Обратите внимание на `String(key)`: ключом может быть символ, а шаблонная строка со `Symbol` бросит `TypeError`. Движок сам читает символьные свойства (`Symbol.iterator`, `Symbol.toPrimitive`), поэтому ловушка должна уметь с ними обращаться.

Пример посложнее — отрицательные индексы массива:

```js
const arr = new Proxy([1, 2, 3], {
  get(t, key, receiver) {
    const i = typeof key === 'string' ? Number(key) : NaN;
    if (Number.isInteger(i) && i < 0) key = String(t.length + i);
    return Reflect.get(t, key, receiver);
  },
});
console.log(arr[-1], arr[0], arr.length); // 3 1 3
```

## Ловушка set: запись и валидация

`set(target, property, value, receiver)` обязана **вернуть `true`**, если запись удалась. Если вернуть `false` (или ничего), то в строгом режиме выбросится `TypeError`, а в нестрогом запись молча игнорируется.

```js
const nums = new Proxy([], {
  set(t, key, value, receiver) {
    if (typeof value !== 'number') return false;
    return Reflect.set(t, key, value, receiver);
  },
});
nums.push(1);
try {
  nums.push('x');
} catch (e) {
  console.log(e.constructor.name); // TypeError
}
console.log(nums.length); // 1
```

`push` — метод в строгом режиме, поэтому отказ ловушки превратился в исключение. Типичная ошибка новичка — забыть `return true` в конце: код «работает» в нестрогом режиме и ломается в модулях и классах.

## has, deleteProperty, ownKeys

- `has(target, key)` перехватывает оператор `in`. Это позволяет, например, сделать «виртуальный диапазон»:

```js
const range = new Proxy({}, {
  has(t, key) {
    return Number(key) >= 1 && Number(key) <= 5;
  },
});
console.log(3 in range, 9 in range); // true false
```

- `deleteProperty(target, key)` — оператор `delete`;
- `ownKeys(target)` — `Object.keys`, `Object.getOwnPropertyNames`, `for...in`, `Reflect.ownKeys`. Чтобы спрятать «приватные» поля с префиксом `_`, перехватывают `ownKeys` и `get`:

```js
const user = { name: 'Аня', _pw: 'secret', age: 30 };
const hidden = new Proxy(user, {
  ownKeys(t) {
    return Reflect.ownKeys(t).filter((k) => !String(k).startsWith('_'));
  },
  get(t, key, receiver) {
    if (typeof key === 'string' && key.startsWith('_')) return undefined;
    return Reflect.get(t, key, receiver);
  },
});
console.log(Object.keys(hidden)); // [ 'name', 'age' ]
console.log(JSON.stringify(hidden)); // {"name":"Аня","age":30}
```

Тонкость: `Object.keys` ещё спрашивает у объекта дескриптор каждого ключа (`getOwnPropertyDescriptor`), чтобы проверить `enumerable`. Если `ownKeys` вернул ключ, которого у цели нет, то `Object.keys` его отфильтрует, а вот `Reflect.ownKeys` — покажет.

## apply и construct: функции

Для прокси над функцией работают `apply(target, thisArg, args)` (вызов) и `construct(target, args, newTarget)` (вызов через `new`):

```js
function sum(a, b) { return a + b; }
const calls = [];
const spy = new Proxy(sum, {
  apply(t, thisArg, args) {
    calls.push(args);
    return Reflect.apply(t, thisArg, args);
  },
});
console.log(spy(1, 2), calls); // 3 [ [ 1, 2 ] ]
```

В отличие от обёртки-функции, прокси сохраняет все свойства оригинала (`name`, `length`, статические поля), потому что неперехваченные операции уходят цели.

## Reflect: «стандартное поведение» в виде функций

`Reflect` — встроенный объект, у которого есть метод на каждую ловушку: `Reflect.get`, `Reflect.set`, `Reflect.has`, `Reflect.deleteProperty`, `Reflect.ownKeys`, `Reflect.apply`, `Reflect.construct`, `Reflect.defineProperty`, `Reflect.getPrototypeOf` и так далее. Сигнатуры совпадают с сигнатурами ловушек, поэтому внутри перехватчика достаточно вызвать `Reflect.<имя>(...arguments)`, чтобы выполнить стандартное действие.

Чем `Reflect` лучше «ручного» `t[key] = value`:

- возвращает булево значение успеха (`Reflect.set(Object.freeze({}), 'a', 1)` даёт `false`), что идеально подходит для результата ловушки `set`;
- не бросает исключение там, где `Object.defineProperty` бросил бы (`Reflect.defineProperty` возвращает `false`);
- принимает `receiver` и корректно работает с геттерами и наследованием;
- превращает операторы (`in`, `delete`, `new`) в функции.

Но `Reflect` принимает только объекты: `Reflect.get(1, 'a')` — `TypeError`.

## receiver: зачем третий аргумент

Если у цели есть геттер, обращающийся к `this`, то при `t[key]` внутри ловушки `this` окажется целью, а не прокси или наследником. Сравните:

```js
const base = { get who() { return this.name; } };

const bad = new Proxy(base, { get(t, key) { return t[key]; } });
const child1 = Object.create(bad);
child1.name = 'child1';
console.log(child1.who); // undefined

const good = new Proxy(base, { get(t, key, r) { return Reflect.get(t, key, r); } });
const child2 = Object.create(good);
child2.name = 'child2';
console.log(child2.who); // child2
```

В первом случае геттер вызван с `this === base`, у которого нет `name`. Во втором `receiver` — это `child2`, и `Reflect.get` передаёт его как `this`. Правило: ловушки `get` и `set` по умолчанию пересылайте через `Reflect` вместе с `receiver`.

## Ограничение: внутренние слоты

У встроенных `Map`, `Set`, `Date`, а также у объектов с приватными полями `#x` данные хранятся во «внутренних слотах», которые у прокси отсутствуют. Метод, вызванный с `this === proxy`, не находит слот и падает:

```js
const map = new Map();
const pm = new Proxy(map, {});
try {
  pm.set(1, 2);
} catch (e) {
  console.log(e.constructor.name); // TypeError
}
```

Решение — привязывать методы к цели:

```js
const pm2 = new Proxy(map, {
  get(t, key) {
    const v = Reflect.get(t, key, t);
    return typeof v === 'function' ? v.bind(t) : v;
  },
});
pm2.set(1, 2);
console.log(pm2.get(1), pm2.size); // 2 1
```

С массивами проблем нет: у них нет внутренних слотов с данными, и `Array.isArray` распознаёт прокси.

## Инварианты

Прокси не вправе нарушать неизменяемые свойства цели. Если свойство заморожено, `get` обязан вернуть то же значение:

```js
const frozen = Object.freeze({ x: 1 });
const liar = new Proxy(frozen, { get() { return 2; } });
try {
  liar.x;
} catch (e) {
  console.log(e.constructor.name); // TypeError
}
```

Аналогичные правила есть у `set`, `has`, `deleteProperty`, `ownKeys`: нельзя «потерять» несконфигурируемое свойство или заявить новое у нерасширяемого объекта.

## Отключаемый прокси

`Proxy.revocable(target, handler)` возвращает `{ proxy, revoke }`. После `revoke()` любая операция над прокси бросает `TypeError`:

```js
const { proxy, revoke } = Proxy.revocable({ a: 1 }, {});
console.log(proxy.a); // 1
revoke();
try {
  proxy.a;
} catch (e) {
  console.log(e.message); // Cannot perform 'get' on a proxy that has been revoked
}
```

Удобно выдавать доступ к объекту на ограниченное время. Текст сообщения зависит от движка, опираться на него в коде не стоит.

## Типичные ошибки

> [!WARNING]
>
> - Нет `return true` в `set`: в строгом режиме — `TypeError`.
> - Чтение `target[key]` вместо `Reflect.get(target, key, receiver)`: ломаются геттеры и наследование.
> - Строка из `key` без проверки на символ.
> - Ожидание, что `proxy === target`, или что прокси «подменит» исходный объект там, где на него уже есть прямые ссылки: прокси перехватывает только обращения *через себя*.
> - Прокси над `Map`/`Set`/классом с `#приватными` полями без `bind`.

## Коротко

> [!TIP]
>
> - `new Proxy(target, handler)` перехватывает операции, ненайденные ловушки пересылаются цели.
> - `set` должен вернуть `true`; `get` и `set` пересылайте через `Reflect` с `receiver`.
> - `has`, `deleteProperty`, `ownKeys`, `apply`, `construct` покрывают остальные операции.
> - Для встроенных коллекций и приватных полей методы нужно привязывать к цели.
> - Инварианты не позволяют прокси лгать про неизменяемые свойства цели.
> - `Proxy.revocable` даёт отзываемый доступ.

---

*Тема по мотивам статьи [«Proxy и Reflect»](https://learn.javascript.ru/proxy) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
