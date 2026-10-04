# Классы: базовый синтаксис

Конструкторы и `F.prototype` позволяют описывать «типы» объектов, но выглядит это громоздко: данные в одном месте, методы дописываются отдельно. Синтаксис `class` собирает всё в один блок. После урока вы сможете описывать классы с конструктором, методами, геттерами и полями, и будете понимать, что под капотом это всё те же прототипы.

## Базовый синтаксис

```js
class User {
  constructor(name) {
    this.name = name;
  }

  sayHi() {
    return `Привет, ${this.name}!`;
  }
}

const user = new User('Анна');

console.log(user.sayHi()); // Привет, Анна!
console.log(typeof User); // function
console.log(User.prototype.sayHi === user.sayHi); // true
console.log(Object.getOwnPropertyNames(User.prototype)); // [ 'constructor', 'sayHi' ]
```

`new User('Анна')` вызывает `constructor`, который заполняет новый объект. Методы, записанные в теле класса, лежат в `User.prototype` — как если бы вы написали `User.prototype.sayHi = ...`. Между методами в теле класса запятых нет (в литерале объекта они нужны, и это частая путаница).

## Что такое class на самом деле

Класс — это особый вид функции. `class User` создаёт функцию `User` с кодом из `constructor` (или пустую, если конструктора нет) и записывает методы в `User.prototype`. Но отличия от «функции-конструктора» есть, и они существенны:

- вызвать класс без `new` нельзя — будет `TypeError`;
- методы класса неперечислимы: `for...in` их не покажет;
- код внутри класса всегда выполняется в строгом режиме;
- класс не поднимается как функция: его нельзя использовать до объявления (как `let`).

```js
class Box {
  open() {}
}

try {
  Box();
} catch (error) {
  console.log(error.message); // Class constructor Box cannot be invoked without 'new'
}

console.log(Object.keys(Box.prototype)); // []

try {
  new Later();
} catch (error) {
  console.log(error.name); // ReferenceError
}
class Later {}
```

## Class Expression

Как и функции, классы можно объявлять выражением, в том числе именованным. Имя видно только внутри класса.

```js
const Point = class Dot {
  who() {
    return Dot.name;
  }
};

console.log(Point.name); // Dot
console.log(new Point().who()); // Dot
console.log(typeof Dot); // undefined
```

Класс можно вернуть из функции — получится «фабрика классов»:

```js
function makeGreeter(word) {
  return class {
    greet(name) {
      return `${word}, ${name}!`;
    }
  };
}

const Hello = makeGreeter('Привет');
console.log(new Hello().greet('мир')); // Привет, мир!
```

## Геттеры и сеттеры

Внутри класса работают привычные `get` и `set`. Они тоже оказываются в прототипе.

```js
class Rect {
  constructor(width, height) {
    this.width = width;
    this.height = height;
  }

  get area() {
    return this.width * this.height;
  }

  set side(value) {
    this.width = value;
    this.height = value;
  }
}

const rect = new Rect(2, 5);
console.log(rect.area); // 10
rect.side = 3;
console.log(rect.area); // 9
```

Вычисляемые имена методов задаются квадратными скобками:

```js
class Menu {
  ['open' + 'Door']() {
    return 'дверь открыта';
  }
  *[Symbol.iterator]() {
    yield 'a';
    yield 'b';
  }
}

console.log(new Menu().openDoor()); // дверь открыта
console.log([...new Menu()]); // [ 'a', 'b' ]
```

## Поля класса

Помимо методов, в теле класса можно объявить **поля** — они создаются как собственные свойства каждого экземпляра перед выполнением конструктора.

```js
class Counter {
  count = 0;
  step = 1;

  tick() {
    this.count += this.step;
    return this;
  }
}

const a = new Counter();
const b = new Counter();
a.tick().tick();

console.log(a.count); // 2
console.log(b.count); // 0
console.log(Object.keys(a)); // [ 'count', 'step' ]
console.log(Object.hasOwn(Counter.prototype, 'count')); // false
```

Поля всегда собственные для экземпляра, а не общие в прототипе, поэтому для каждого создаётся свой массив или объект.

### Потеря this и стрелочные поля

Метод класса, переданный как колбэк, теряет `this`. Одно из решений — поле со стрелочной функцией: она создаётся для каждого экземпляра и запоминает `this`.

```js
class Button {
  label = 'ОК';
  getLabel() {
    return this?.label;
  }
  getLabelBound = () => this.label;
}

const btn = new Button();
const lost = btn.getLabel;
const kept = btn.getLabelBound;

console.log(lost()); // undefined
console.log(kept()); // ОК
```

Плата за такое решение: функция создаётся заново для каждого объекта, а не лежит в общем прототипе.

## Типичные ошибки

> [!WARNING]
>
> - Ставить запятые между методами класса — это синтаксическая ошибка.
> - Вызывать класс без `new`.
> - Пытаться использовать класс до объявления.
> - Ожидать, что поле, объявленное в классе, окажется в прототипе: оно собственное.
> - Забывать, что метод, переданный как колбэк, остаётся без `this`.

## Коротко

> [!TIP]
>
> - `class` — синтаксис поверх функций и прототипов: методы кладутся в `Class.prototype`, конструктор — это сама функция.
> - Класс нельзя вызвать без `new`, его методы неперечислимы, а тело работает в строгом режиме.
> - Классы бывают в виде объявления и выражения; можно возвращать класс из функции.
> - Внутри допустимы `get`/`set`, вычисляемые имена и генераторы.
> - Поля класса создаются в каждом экземпляре; стрелочные поля решают потерю `this`, но не экономят память.

---

*Тема по мотивам статьи [«Класс: базовый синтаксис»](https://learn.javascript.ru/class) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
