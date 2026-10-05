# Расширение встроенных классов

Встроенные классы — `Array`, `Map`, `Error`, `Promise` и другие — тоже можно расширять через `extends`. Так создают специализированные коллекции и собственные типы ошибок. У этой возможности есть нюансы: что возвращают унаследованные методы, зачем нужен `Symbol.species` и почему у встроенных классов нет статического наследования.

## Расширяем Array

```js
class PowerArray extends Array {
  isEmpty() {
    return this.length === 0;
  }
}

const arr = new PowerArray(1, 2, 5, 10, 50);

console.log(arr.isEmpty()); // false
console.log(arr.length); // 5
console.log(arr instanceof Array); // true
console.log(Array.isArray(arr)); // true
```

Наш массив — настоящий: `length`, индексы и все методы работают, потому что объект создаётся «экзотическим» встроенным конструктором `Array`, а не обычным.

Неожиданность в том, что методы, создающие новый массив (`map`, `filter`, `slice`, `flat`, `concat`), возвращают объект **того же класса**, что и исходный. Внутри они вызывают `new this.constructor(...)`.

```js
class PowerArray extends Array {
  isEmpty() {
    return this.length === 0;
  }
}

const arr = new PowerArray(1, 2, 5, 10, 50);
const filtered = arr.filter((x) => x >= 10);

console.log(filtered.constructor === PowerArray); // true
console.log(filtered.isEmpty()); // false
console.log(filtered instanceof PowerArray); // true
console.log(arr.slice(0, 2) instanceof PowerArray); // true
```

Обычно это удобно: после `filter` можно сразу вызвать `isEmpty`. Но иногда нужно вернуть обычный массив.

## Symbol.species

Статический геттер `Symbol.species` сообщает методам, какой конструктор использовать для результата. По умолчанию он возвращает `this`, то есть сам класс. Переопределив его, мы заставим `map`/`filter` создавать, например, обычные массивы.

```js
class PowerArray extends Array {
  static get [Symbol.species]() {
    return Array;
  }
  isEmpty() {
    return this.length === 0;
  }
}

const arr = new PowerArray(1, 2, 5, 10, 50);
const filtered = arr.filter((x) => x >= 10);

console.log(filtered.isEmpty); // undefined
console.log(filtered instanceof PowerArray); // false
console.log(Array.isArray(filtered)); // true
```

Для `Map`, `Set`, `Promise` и `RegExp` этот механизм работает аналогично там, где такие методы создают новые объекты (например, `Promise.prototype.then`).

## Свои ошибки

Самый частый случай расширения встроенных классов — собственные типы ошибок. Наследуясь от `Error`, вы получаете стек вызовов, `message` и совместимость с `instanceof`.

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

try {
  throw new ValidationError('Поле пустое', 'email');
} catch (error) {
  console.log(error instanceof ValidationError); // true
  console.log(error instanceof Error); // true
  console.log(`${error.name}: ${error.message}`); // ValidationError: Поле пустое
  console.log(error.field); // email
  console.log(typeof error.stack); // string
}
```

`name` нужно задать вручную, иначе в сообщениях будет `Error`. Обычная практика — базовый класс прикладных ошибок, от которого наследуют остальные; тогда `catch` может отличать «свои» ошибки от случайных.

```js
class AppError extends Error {
  constructor(message, options) {
    super(message, options);
    this.name = this.constructor.name;
  }
}
class NotFoundError extends AppError {}

const err = new NotFoundError('нет страницы', { cause: 404 });

console.log(err.name); // NotFoundError
console.log(err.cause); // 404
console.log(err instanceof AppError); // true
```

Второй аргумент `super(message, options)` с полем `cause` — стандартный способ приложить исходную причину.

## Расширяем Map и другие

Аналогично работают `Map`, `Set`, `Promise` и прочие классы. В наследнике можно переопределять методы и вызывать родительские через `super`.

```js
class DefaultMap extends Map {
  constructor(factory, entries) {
    super(entries);
    this.factory = factory;
  }
  get(key) {
    if (!this.has(key)) {
      this.set(key, this.factory());
    }
    return super.get(key);
  }
}

const groups = new DefaultMap(() => []);
groups.get('a').push(1);
groups.get('a').push(2);

console.log(groups.get('a')); // [ 1, 2 ]
console.log(groups.size); // 1
console.log(groups instanceof Map); // true
```

## Статические методы наследников

Сами встроенные классы статику друг у друга не наследуют: `Array` и `Object` — независимые функции, и `Array.keys` не существует. Но для вашего наследника всё работает как для обычного `extends`: статические методы родителя доступны, а `from` и `of` создают объекты именно вашего класса.

```js
class PowerArray extends Array {}

const made = PowerArray.from([1, 2, 3]);

console.log(made instanceof PowerArray); // true
console.log(PowerArray.of(7) instanceof PowerArray); // true
```

## Типичные ошибки

> [!WARNING]
>
> - Забыть, что `map` и `filter` вернут экземпляр вашего класса, и не ожидать там нужные поля: конструктор будет вызван с одним числом (длиной), а не с вашими параметрами.
> - Менять сигнатуру конструктора `Array` и ломать внутренние вызовы `new this.constructor(n)`.
> - Не задать `name` у класса ошибки.
> - Не вызвать `super(message)` в конструкторе ошибки: сообщение потеряется.

## Коротко

> [!TIP]
>
> - Встроенные классы расширяются через `extends` и остаются настоящими (`Array.isArray`, `instanceof`).
> - Методы вроде `map`, `filter`, `slice` создают результат через `this.constructor`, поэтому возвращают объект вашего класса.
> - `static get [Symbol.species]()` позволяет выбрать другой конструктор для результата.
> - Для своих ошибок наследуйтесь от `Error`, вызывайте `super(message)` и задавайте `name`.
> - Не меняйте сигнатуру конструктора встроенного класса несовместимым образом.

---

*Тема по мотивам статьи [«Расширение встроенных классов»](https://learn.javascript.ru/extend-natives) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
