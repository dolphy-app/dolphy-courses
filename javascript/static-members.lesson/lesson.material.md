# Статические свойства и методы

Бывает, что функциональность относится к классу в целом, а не к отдельному объекту: фабрика, сравнение двух экземпляров, счётчик созданных объектов. Для этого есть статические члены — свойства и методы, которые принадлежат самому классу. В уроке разберём их синтаксис, наследование, а также статические блоки инициализации.

## Статические методы

Ключевое слово `static` перед методом кладёт его не в `Class.prototype`, а в сам класс (функцию). Вызывают его как `Class.method()`, и экземпляры его не видят.

```js
class Version {
  constructor(major, minor) {
    this.major = major;
    this.minor = minor;
  }

  static parse(text) {
    const [major, minor] = text.split('.').map(Number);
    return new Version(major, minor);
  }

  static compare(a, b) {
    return a.major - b.major || a.minor - b.minor;
  }
}

const v1 = Version.parse('1.4');
const v2 = Version.parse('1.10');

console.log(Version.compare(v1, v2) < 0); // true
console.log(typeof v1.parse); // undefined
console.log(Object.hasOwn(Version, 'parse')); // true
```

Статические методы часто служат фабриками (`Version.parse`), служебными функциями, привязанными к типу (`Array.isArray`, `Object.keys`, `Promise.all`), или местом для сравнения и сортировки экземпляров.

Если нужно вызвать статический метод из экземпляра, пишут `this.constructor.method()` — так сработает версия того класса, объектом которого является `this`:

```js
class Tool {
  static label() {
    return 'tool';
  }
  describe() {
    return this.constructor.label();
  }
}
class Hammer extends Tool {
  static label() {
    return 'hammer';
  }
}

console.log(new Tool().describe()); // tool
console.log(new Hammer().describe()); // hammer
```

## Статические свойства

Свойство можно объявить прямо в теле класса со словом `static`. Оно будет одно на весь класс.

```js
class Order {
  static count = 0;
  static TAX = 0.2;

  constructor() {
    Order.count += 1;
    this.id = Order.count;
  }
}

new Order();
new Order();
const third = new Order();

console.log(Order.count); // 3
console.log(third.id); // 3
console.log(third.count); // undefined
console.log(Order.TAX); // 0.2
```

Обратите внимание на `Order.count += 1` внутри конструктора: к статическому свойству обращаются через имя класса (или `this.constructor`), а не через `this` — у экземпляра такого свойства нет.

Статическое свойство можно инициализировать результатом выражения; внутри инициализатора `this` — это сам класс:

```js
class Config {
  static defaults = { retries: 3 };
  static retries = this.defaults.retries * 2;
}

console.log(Config.retries); // 6
```

## Статические блоки

Если для подготовки статических данных нужен целый код, а не одно выражение, используется блок `static { ... }`. Он выполняется один раз при создании класса, `this` в нём — класс.

```js
class Registry {
  static #items = new Map();
  static size;

  static {
    Registry.#items.set('a', 1);
    Registry.#items.set('b', 2);
    this.size = Registry.#items.size;
  }

  static get(key) {
    return Registry.#items.get(key);
  }
}

console.log(Registry.size); // 2
console.log(Registry.get('b')); // 2
```

## Наследование статических членов

Когда класс расширяет другой, функция-наследник получает в `[[Prototype]]` родителя. Поэтому статические методы и свойства наследуются так же, как обычные — по цепочке прототипов.

```js
class Animal {
  static planet = 'Земля';
  static compare(a, b) {
    return a.speed - b.speed;
  }
  constructor(name, speed) {
    this.name = name;
    this.speed = speed;
  }
}
class Rabbit extends Animal {}

const animals = [new Rabbit('быстрый', 30), new Rabbit('медленный', 5)];
animals.sort(Rabbit.compare);

console.log(animals.map((a) => a.name)); // [ 'медленный', 'быстрый' ]
console.log(Rabbit.planet); // Земля
console.log(Object.hasOwn(Rabbit, 'planet')); // false
console.log(Object.getPrototypeOf(Rabbit) === Animal); // true
```

Важная тонкость: статическое **свойство** примитивного типа наследуется как чтение, но запись через наследника создаёт собственное свойство и родителя не меняет. А вот изменяемый объект остаётся общим.

```js
class Base {
  static hits = 0;
  static log = [];
  static hit() {
    this.hits += 1;
    this.log.push(this.name);
  }
}
class Child extends Base {}

Base.hit();
Child.hit();

console.log(Base.hits); // 1
console.log(Child.hits); // 2
console.log(Object.hasOwn(Child, 'hits')); // true
console.log(Base.log); // [ 'Base', 'Child' ]
```

`Child.hit()` выполняет `this.hits += 1`, где `this` — `Child`: читает `1` у `Base`, а записывает `2` уже в `Child`. Массив же `log` один и тот же. Учитывайте это при написании счётчиков в иерархиях.

## Встроенные классы

У встроенных классов статических членов много: `Array.from`, `Number.isInteger`, `Date.now`. Но между собой они статику не наследуют: цепочка `Array.prototype` → `Object.prototype` относится к экземплярам, а сами функции `Array` и `Object` связаны иначе — обе имеют прототипом `Function.prototype`.

```js
console.log(Object.getPrototypeOf(Array) === Function.prototype); // true
console.log(typeof Array.keys); // undefined
console.log(typeof Object.keys); // function
```

А вот в пользовательских классах с `extends` статика наследуется всегда.

## Типичные ошибки

> [!WARNING]
>
> - Вызывать статический метод у экземпляра: `obj.parse()` — `TypeError`.
> - Обращаться к статическому свойству через `this` в обычном методе.
> - Ожидать, что `static count` в наследнике изменит счётчик родителя: запись создаст собственное свойство.
> - Класть в статическое поле изменяемый объект и удивляться, что он общий для всех наследников.

## Коротко

> [!TIP]
>
> - `static` помещает метод или поле в сам класс, а не в прототип экземпляров.
> - К статике обращаются через имя класса или `this.constructor` из метода экземпляра.
> - Внутри статических методов и блоков `this` — класс (или наследник, от имени которого вызвали).
> - Статические члены наследуются: `Child` имеет `Parent` в `[[Prototype]]`.
> - Запись примитива через наследника создаёт его собственную копию, изменяемые объекты остаются общими.

---

*Тема по мотивам статьи [«Статические свойства и методы»](https://learn.javascript.ru/static-properties-methods) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
