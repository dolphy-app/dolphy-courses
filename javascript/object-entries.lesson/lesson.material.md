# Object.keys, values, entries

Объект — набор пар «имя свойства — значение». Чтобы пройтись по этим парам, посчитать сумму значений или преобразовать объект, обычно переводят его в массив и применяют привычные `map`, `filter`, `reduce`. Для этого есть три статических метода `Object.keys`, `Object.values` и `Object.entries`, а обратно в объект превращает `Object.fromEntries`. После урока вы сможете обрабатывать объекты так же уверенно, как массивы.

## Три метода

Все три принимают объект и возвращают **новый массив**:

| Метод | Результат |
|---|---|
| `Object.keys(obj)` | массив имён свойств (строки) |
| `Object.values(obj)` | массив значений |
| `Object.entries(obj)` | массив пар `[имя, значение]` |

```js
const prices = { apple: 10, pear: 20, plum: 5 };

console.log(Object.keys(prices));    // ['apple', 'pear', 'plum']
console.log(Object.values(prices));  // [10, 20, 5]
console.log(Object.entries(prices)); // [['apple', 10], ['pear', 20], ['plum', 5]]
```

Обратите внимание на синтаксис: вызывается `Object.keys(obj)`, а не `obj.keys()`. Дело в общности: объект — самая универсальная структура, и у него может быть собственное свойство `keys`, которое не должно конфликтовать с методом языка. Поэтому утилиты вынесены в `Object`.

Не путайте с `Map`: у неё `map.keys()` возвращает итератор, а `Object.keys` — настоящий массив.

## Что попадает в результат

Эти методы возвращают только то, что:

- принадлежит самому объекту (унаследованные свойства игнорируются);
- имеет строковое имя (символьные ключи пропускаются);
- является перечисляемым (`enumerable`).

```js
const base = { inherited: 1 };
const obj = Object.create(base);
obj.own = 2;
obj[Symbol('hidden')] = 3;

console.log(Object.keys(obj)); // ['own']

for (const key in obj) {
  console.log(key); // 'own', затем 'inherited'
}
```

Цикл `for...in` в отличие от `Object.keys` заходит и в прототипы, поэтому в современном коде для перебора собственных свойств предпочитают `Object.keys/values/entries`. Символы можно получить через `Object.getOwnPropertySymbols`, а все собственные строковые ключи, включая неперечисляемые, — через `Object.getOwnPropertyNames`.

## Порядок свойств

Порядок в результате не всегда равен порядку записи. Целочисленные ключи (`'0'`, `'1'`, `'42'`) идут первыми по возрастанию, остальные строки — в порядке создания:

```js
const obj = { b: 2, a: 1, 2: 'x', 1: 'y' };
console.log(Object.keys(obj));   // ['1', '2', 'b', 'a']
console.log(Object.values(obj)); // ['y', 'x', 2, 1]
```

Если вам нужен свой порядок, сортируйте массив ключей явно или используйте `Map`.

## Работа с массивами и строками

`Object.keys/values/entries` можно применять и к другим значениям: массивы дадут индексы-строки, строки — посимвольные пары. А вот `null` и `undefined` приведут к `TypeError`; примитивы-числа вернут пустой массив:

```js
console.log(Object.keys([7, 8]));    // ['0', '1']
console.log(Object.entries('ab'));   // [['0', 'a'], ['1', 'b']]
console.log(Object.keys(5));         // []
// Object.keys(null) -> TypeError
```

## Перебор и деструктуризация пар

`Object.entries` удобно использовать с `for...of` и деструктуризацией пары:

```js
for (const [name, price] of Object.entries({ tea: 3, milk: 2 })) {
  console.log(name, price); // tea 3, затем milk 2
}
```

Суммирование значений:

```js
const total = Object.values(prices).reduce((sum, n) => sum + n, 0);
console.log(total); // 35
```

Результат — копия данных в виде массива: изменение этого массива объект не затрагивает (`Object.values(obj).push(5)` оставит `obj` прежним), но вложенные объекты в нём — те же самые ссылки.

## Object.fromEntries: обратно в объект

`Object.fromEntries(iterable)` принимает массив (или любой итерируемый набор) пар и собирает объект. При повторяющемся ключе побеждает последняя пара.

```js
console.log(Object.fromEntries([['a', 1], ['b', 2]])); // { a: 1, b: 2 }
console.log(Object.fromEntries([['a', 1], ['a', 2]]));  // { a: 2 }
console.log(Object.fromEntries(new Map([['x', 1]])));   // { x: 1 }
```

Связка `entries → массивные методы → fromEntries` позволяет преобразовывать объекты:

```js
const doubled = Object.fromEntries(
  Object.entries(prices).map(([name, price]) => [name, price * 2])
);
console.log(doubled); // { apple: 20, pear: 40, plum: 10 }

const cheap = Object.fromEntries(
  Object.entries(prices).filter(([, price]) => price < 15)
);
console.log(cheap); // { apple: 10, plum: 5 }
```

Оба шага не меняют исходный объект — результат всегда новый.

## Типичные ошибки

- Вызов `obj.keys()` вместо `Object.keys(obj)`.
- Ожидание, что символьные ключи попадут в результат.
- Надежда на порядок «как написано» при числовых ключах.
- Передача `null`/`undefined` в эти методы без проверки.
- Забытый возврат пары в `map`: нужно `[key, value]`, а не просто значение — иначе `fromEntries` выбросит ошибку.

## Коротко

- `Object.keys`, `Object.values`, `Object.entries` возвращают массивы собственных перечисляемых строковых свойств.
- Символы и унаследованные свойства пропускаются; `for...in` захватывает и унаследованные.
- Целочисленные ключи идут первыми по возрастанию.
- `Object.fromEntries` собирает объект из пар; последняя пара с тем же ключом побеждает.
- Конструкция `Object.fromEntries(Object.entries(obj).map(...))` — стандартный способ преобразовать объект.

---

*Тема по мотивам статьи [«Object.keys, values, entries»](https://learn.javascript.ru/keys-values-entries) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
