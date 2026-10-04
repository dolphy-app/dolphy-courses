# Наследование классов

Класс может расширять другой класс: получать его методы и добавлять свои. Ключевое слово — `extends`. В уроке разберём, как работают `extends`, `super` и переопределение методов, а главное — почему конструктор наследника обязан вызвать `super()` до обращения к `this`.

## extends

```js
class Animal {
  constructor(name) {
    this.name = name;
    this.speed = 0;
  }

  run(speed) {
    this.speed = speed;
    return `${this.name} бежит со скоростью ${speed}`;
  }

  stop() {
    this.speed = 0;
    return `${this.name} стоит`;
  }
}

class Rabbit extends Animal {
  hide() {
    return `${this.name} прячется`;
  }
}

const rabbit = new Rabbit('Белый');

console.log(rabbit.run(5)); // Белый бежит со скоростью 5
console.log(rabbit.hide()); // Белый прячется
console.log(Object.getPrototypeOf(Rabbit.prototype) === Animal.prototype); // true
console.log(Object.getPrototypeOf(Rabbit) === Animal); // true
```

`extends` делает две вещи. Прототипом `Rabbit.prototype` становится `Animal.prototype`: так экземпляры находят унаследованные методы. И прототипом самой функции `Rabbit` становится `Animal`: так наследуются статические члены (о них — в отдельном уроке).

После `extends` допустимо любое выражение, возвращающее конструктор: `class A extends mixin(Base)` или `class B extends makeBase()`.

## Переопределение методов и super

Если в наследнике объявить метод с тем же именем, он «перекроет» родительский. Чтобы не терять родительское поведение, а расширить его, используйте `super.method(...)`.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  stop() {
    return `${this.name} стоит`;
  }
}
class Rabbit extends Animal {
  stop() {
    return `${super.stop()}, затаился`;
  }
}

console.log(new Rabbit('Серый').stop()); // Серый стоит, затаился
```

`super.method()` ищет метод в родительском прототипе, но вызывает его с текущим `this`. Стрелочные функции собственного `super` не имеют — они берут его из внешнего метода:

```js
class Base {
  hi() {
    return 'base';
  }
}
class Child extends Base {
  hi() {
    const later = () => super.hi() + '+child';
    return later();
  }
}

console.log(new Child().hi()); // base+child
```

## Конструктор наследника

Если у наследника нет своего конструктора, JavaScript создаёт его автоматически и передаёт все аргументы родителю:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}
class Rabbit extends Animal {}

console.log(new Rabbit('Заяц').name); // Заяц
```

Если конструктор свой, он **обязан** вызвать `super(...)` до первого обращения к `this`. Причина в устройстве языка: в наследующем классе объект создаёт родительский конструктор, а не `new`. Пока `super()` не выполнен, `this` ещё не существует.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}
class Rabbit extends Animal {
  constructor(name, earLength) {
    super(name);
    this.earLength = earLength;
  }
}

const r = new Rabbit('Белый', 15);
console.log(r.name, r.earLength); // Белый 15

class Broken extends Animal {
  constructor() {
    this.x = 1;
  }
}
try {
  new Broken();
} catch (error) {
  console.log(error.name); // ReferenceError
}
```

Если конструктор наследника вообще не вызвал `super()` и не вернул объект, получится `ReferenceError` при завершении.

### Поля класса и порядок инициализации

Поля базового класса создаются в начале его конструктора. Поля наследника — сразу после возврата из `super()`. Поэтому, если родительский конструктор вызывает переопределённый метод, он увидит ещё не инициализированные поля потомка.

```js
class Base {
  constructor() {
    console.log(this.describe()); // x=undefined
  }
  describe() {
    return 'base';
  }
}
class Derived extends Base {
  x = 5;
  describe() {
    return `x=${this.x}`;
  }
}

const d = new Derived();
console.log(d.describe()); // x=5
```

Вывод: не вызывайте переопределяемые методы из конструктора родителя, если они зависят от состояния потомка.

## Полиморфизм и цепочка super

Благодаря `extends` код может работать с «любым животным», не зная конкретного класса: каждый наследник по-своему реализует нужный метод.

```js
class Shape {
  area() {
    return 0;
  }
  describe() {
    return `${this.constructor.name}: ${this.area()}`;
  }
}
class Square extends Shape {
  constructor(side) {
    super();
    this.side = side;
  }
  area() {
    return this.side ** 2;
  }
}
class Circle extends Shape {
  constructor(r) {
    super();
    this.r = r;
  }
  area() {
    return Math.round(Math.PI * this.r ** 2);
  }
}

console.log([new Square(3), new Circle(2)].map((s) => s.describe())); // [ 'Square: 9', 'Circle: 13' ]
```

## Типичные ошибки

- Обратиться к `this` в конструкторе наследника до `super()`.
- Забыть передать в `super(...)` аргументы, нужные родителю.
- Забыть `super.method()` и потерять родительскую логику.
- Вызывать в родительском конструкторе методы, зависящие от полей потомка.
- Ставить `extends` на объект, не являющийся конструктором: будет `TypeError`.

## Коротко

- `class B extends A` связывает `B.prototype` с `A.prototype` и `B` с `A` (для статических членов).
- Метод наследника перекрывает родительский; `super.method()` вызывает родительскую версию с текущим `this`.
- В конструкторе наследника `super(...)` вызывается до использования `this`; без явного конструктора аргументы передаются родителю автоматически.
- Поля наследника появляются после `super()`.
- Хорошее наследование — «является»: наследник можно использовать везде, где ожидается родитель.

---

*Тема по мотивам статьи [«Наследование классов»](https://learn.javascript.ru/class-inheritance) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
