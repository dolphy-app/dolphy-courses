# Декораторы, call и apply

Функции в JavaScript — значения, их можно передавать и возвращать. Это позволяет «оборачивать» одну функцию в другую, добавляя поведение: кэш, логирование, подсчёт вызовов. Такая обёртка называется декоратором. Чтобы писать её правильно и не терять `this`, нужны методы `call` и `apply`.

## Декоратор-кэш

Допустим, есть «тяжёлая» функция, результат которой зависит только от аргумента. Вместо того чтобы менять её, обернём в декоратор `cacheResult`, который запоминает ответы:

```js
function slowSquare(x) {
  console.log('считаю ' + x);
  return x * x;
}

function cacheResult(func) {
  const cache = new Map();
  return function (x) {
    if (cache.has(x)) return cache.get(x);
    const result = func(x);
    cache.set(x, result);
    return result;
  };
}

const fastSquare = cacheResult(slowSquare);
console.log(fastSquare(4)); // считаю 4, затем 16
console.log(fastSquare(4)); // 16 (из кэша, без вывода «считаю»)
```

Плюсы: исходная функция не тронута, декоратор можно применить к любой функции, а несколько декораторов можно вкладывать друг в друга.

## Проблема с this

Тот же декоратор ломается на методах объекта. Внутри обёртки `func(x)` вызывается как обычная функция, поэтому `this` у неё — `undefined` в строгом режиме (будет `TypeError` при обращении к полю) или глобальный объект в обычном режиме (поле не найдётся):

```js
const worker = {
  factor: 3,
  triple(x) {
    return x * this.factor;
  },
};

worker.triple = cacheResult(worker.triple);
console.log(worker.triple(2)); // NaN (в строгом режиме — TypeError)
```

Ошибка в том, что обёртка не передаёт `this` дальше. Для этого и нужен `call`.

## func.call(context, ...args)

Метод `call` вызывает функцию, явно задавая `this` первым аргументом, а остальные аргументы передаёт как обычно:

```js
function showName(greeting) {
  return greeting + ', ' + this.name;
}
const ann = { name: 'Анна' };
const bob = { name: 'Борис' };

console.log(showName.call(ann, 'Привет')); // Привет, Анна
console.log(showName.call(bob, 'Пока')); // Пока, Борис
```

Теперь исправим декоратор: обёртка станет обычной функцией (не стрелочной!), возьмёт свой `this` и передаст его исходной функции:

```js
function cacheMethod(func) {
  const cache = new Map();
  return function (x) {
    if (cache.has(x)) return cache.get(x);
    const result = func.call(this, x);
    cache.set(x, result);
    return result;
  };
}
```

Проверим на новом объекте:

```js
const calc = {
  factor: 3,
  triple(x) {
    return x * this.factor;
  },
};
calc.triple = cacheMethod(calc.triple);
console.log(calc.triple(2)); // 6
console.log(calc.triple(2)); // 6
```

## Несколько аргументов: func.apply и spread

Если функция принимает несколько аргументов, обёртка должна передавать их все. Есть два способа. Современный — остаточные параметры и spread:

```js
function forward(func) {
  return function (...args) {
    return func.call(this, ...args);
  };
}
```

Старый, но по-прежнему встречающийся — `apply`, который принимает аргументы **одним массивом** (или массивоподобным объектом):

```js
function sum(a, b, c) {
  return a + b + c + this.base;
}
const ctx = { base: 100 };
console.log(sum.call(ctx, 1, 2, 3)); // 106
console.log(sum.apply(ctx, [1, 2, 3])); // 106
```

Оба метода вызывают функцию немедленно и возвращают её результат — в отличие от `bind`, который создаёт новую функцию (о нём — в следующем уроке). Единственное различие: `call` принимает аргументы списком, `apply` — массивом. Если аргументы уже лежат в массиве, удобнее `apply` или `call` со spread. Запомнить можно так: **a**pply — **a**rray.

Передачу `this` и всех аргументов дальше вместе с возвратом результата называют **транзитным вызовом** (call forwarding). Это основа любого декоратора.

## Заимствование метода

Через `call` можно применить метод одного объекта к другому. Классический пример — взять метод массива для массивоподобного объекта:

```js
const like = { 0: 'a', 1: 'b', 2: 'c', length: 3 };
const joined = Array.prototype.join.call(like, '-');
console.log(joined); // a-b-c
```

Метод `join` читает `this[0]`, `this[1]`, … и `this.length`, поэтому ему всё равно, настоящий ли это массив. Это называется заимствованием метода.

## Декоратор с хешированием нескольких аргументов

Ключом кэша для нескольких аргументов может стать строка, склеенная из них. Ограничение: разные значения могут дать одну строку, например `1` и `'1'`, поэтому такая хеш-функция должна подходить задаче:

```js
function cacheBy(func, hash) {
  const cache = new Map();
  return function (...args) {
    const key = hash(args);
    if (cache.has(key)) return cache.get(key);
    const result = func.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const add = cacheBy((a, b) => a + b, (args) => args.join(','));
console.log(add(1, 2)); // 3
```

## Свойства функции пропадают

Обёртка — это новая функция, у неё нет свойств исходной (`name` другой, пользовательские поля потеряны). Если код рассчитывает на них, их нужно перенести вручную, иначе декоратор станет «непрозрачным».

## Типичные ошибки

- Обёртка — стрелочная функция, поэтому `this` берётся снаружи, а не от вызывающего.
- Потерян `return`: обёртка вызывает функцию, но не возвращает результат.
- Передан только первый аргумент.
- Забыли, что кэш по `Map` с ключами-объектами сравнивает ссылки, а не содержимое.

## Коротко

- Декоратор — функция, которая принимает функцию и возвращает обёртку с новым поведением.
- `func.call(ctx, a, b)` вызывает `func` с заданным `this`; `func.apply(ctx, [a, b])` делает то же с массивом аргументов.
- Обёртка должна быть обычной функцией, передавать `this` и все аргументы и возвращать результат.
- Через `call` можно заимствовать методы, например `Array.prototype.join.call(arrayLike, sep)`.
- Обёртка не наследует свойства исходной функции.

---

*Тема по мотивам статьи [«Декораторы и переадресация вызова, call/apply»](https://learn.javascript.ru/call-apply-decorators) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
