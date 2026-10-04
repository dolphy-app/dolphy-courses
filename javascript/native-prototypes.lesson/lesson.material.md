# Встроенные прототипы

Когда вы пишете `[1, 2, 3].map(...)` или `'abc'.toUpperCase()`, откуда берутся эти методы? Они лежат в прототипах встроенных конструкторов: `Array.prototype`, `String.prototype` и так далее. Зная, как устроена эта иерархия, вы поймёте, почему даже у примитивов есть методы, и когда (редко!) стоит трогать встроенные прототипы.

## Object.prototype: вершина иерархии

Литерал `{}` — то же самое, что `new Object()`. Поэтому прототип любого обычного объекта — `Object.prototype`, а у него прототипа уже нет.

```js
const obj = {};

console.log(Object.getPrototypeOf(obj) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype)); // null
console.log(Object.prototype.constructor === Object); // true
console.log(obj.toString === Object.prototype.toString); // true
```

Именно там живут `toString`, `hasOwnProperty`, `valueOf`, `isPrototypeOf` и другие методы, доступные «всем объектам».

## Цепочки встроенных типов

Остальные встроенные типы строят свои цепочки поверх `Object.prototype`. У массива — `Array.prototype`, у функции — `Function.prototype`, у даты — `Date.prototype`. Все они в конце приходят к `Object.prototype`.

```js
const arr = [1, 2];
const fn = function () {};

console.log(Object.getPrototypeOf(arr) === Array.prototype); // true
console.log(Object.getPrototypeOf(Array.prototype) === Object.prototype); // true
console.log(Object.getPrototypeOf(fn) === Function.prototype); // true
console.log(Object.getPrototypeOf(Function.prototype) === Object.prototype); // true
```

Благодаря этому методы можно переопределять на нужном уровне: у `Array.prototype` свой `toString`, который склеивает элементы через запятую, а у `Object.prototype` — другой.

```js
console.log([1, 2, 3].toString()); // 1,2,3
console.log({}.toString()); // [object Object]
console.log(Object.prototype.toString.call([1, 2, 3])); // [object Array]
console.log(Object.prototype.toString.call(null)); // [object Null]
```

Последние две строки — полезный приём: `Object.prototype.toString.call(value)` определяет «внутренний тип» значения даже для `null`, где обычные способы не работают.

## Примитивы и обёртки

У числа, строки и булева значения нет собственных свойств, но методы вызвать можно. Когда вы пишете `'abc'.toUpperCase()`, движок временно создаёт объект-обёртку (`String`, `Number`, `Boolean`), берёт метод у `String.prototype` и затем выбрасывает обёртку. У `null` и `undefined` обёрток нет, поэтому обращение к их свойствам даёт `TypeError`.

```js
const s = 'abc';

console.log(s.toUpperCase()); // ABC
console.log(Object.getPrototypeOf(Object(s)) === String.prototype); // true
console.log(typeof s); // string
console.log(typeof new String('abc')); // object

try {
  null.length;
} catch (error) {
  console.log(error.name); // TypeError
}
```

Не используйте `new String`, `new Number` и `new Boolean` вручную: результат — объект, а не примитив, и сравнения ведут себя неожиданно.

```js
console.log(new Boolean(false) ? 'истина' : 'ложь'); // истина
console.log(new Number(5) === 5); // false
console.log(Number('5') === 5); // true
```

Вызов без `new` (`Number('5')`) выполняет преобразование и возвращает примитив.

## Изменение встроенных прототипов

Прототипы можно дополнять, и новые методы увидят все значения этого типа. Так в старые движки добавляли «полифилы» — реализации новых функций языка. Идея в том, чтобы добавить метод только если его нет.

```js
if (!String.prototype.shout) {
  String.prototype.shout = function () {
    return this.toUpperCase() + '!';
  };
}

console.log('привет'.shout()); // ПРИВЕТ!
delete String.prototype.shout;
```

Однако в повседневном коде так делать не стоит:

- имя может конфликтовать с будущим методом стандарта или с библиотекой, написавшей такое же;
- метод в `Object.prototype` окажется виден у всех объектов, а если он перечислимый — попадёт в `for...in`;
- чужой код неожиданно изменит поведение.

Допустимо лишь одно оправдание — полифил по спецификации, когда вы воспроизводите точное поведение стандартного метода.

## Заимствование методов

Метод встроенного прототипа можно «позаимствовать» для похожего объекта. Классический пример — псевдомассив (индексы и `length`, но не настоящий массив). Через `call` подставляем его в качестве `this`.

```js
const arrayLike = { 0: 'a', 1: 'b', 2: 'c', length: 3 };

console.log(Array.prototype.join.call(arrayLike, '-')); // a-b-c
console.log(Array.prototype.map.call(arrayLike, (x) => x.toUpperCase())); // [ 'A', 'B', 'C' ]
console.log(Array.from(arrayLike)); // [ 'a', 'b', 'c' ]
```

Современный вариант проще: `Array.from` превращает псевдомассив в настоящий массив. Заимствование полезно, когда копировать данные не нужно.

## Типичные ошибки

- Думать, что у примитива есть собственные свойства: `'abc'.foo = 1` молча ничего не сохранит (в строгом режиме будет `TypeError`).
- Использовать `new Number`/`new String` и сравнивать через `===` или проверять в `if`.
- Добавлять в `Object.prototype` перечислимые свойства: они «протекут» во все `for...in`.
- Выдумывать собственные методы для `Array.prototype`, пересекающиеся с будущими стандартами.

## Коротко

- Вершина иерархии — `Object.prototype`; прототип `Array.prototype`, `Function.prototype`, `Date.prototype` и других — тоже он.
- Методы примитивов берутся из `String.prototype`, `Number.prototype`, `Boolean.prototype` через временную обёртку.
- `null` и `undefined` не имеют обёрток: обращение к их свойствам — `TypeError`.
- Встроенные прототипы расширяют только ради полифилов, и только если метода ещё нет.
- Методы можно заимствовать через `call`, подставляя псевдомассив в качестве `this`; проще — `Array.from`.

---

*Тема по мотивам статьи [«Встроенные прототипы»](https://learn.javascript.ru/native-prototypes) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
