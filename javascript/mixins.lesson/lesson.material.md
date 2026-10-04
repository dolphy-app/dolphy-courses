# Примеси

У класса в JavaScript может быть только один родитель. А что, если нужно добавить поведение из двух независимых источников — например, «умеет сообщать о событиях» и «умеет сериализоваться в строку»? Для этого придуман шаблон «примесь» (mixin): набор методов, который «подмешивают» в класс, не делая его родителем. В уроке разберём простую примесь на объекте, примесь с `super` и функциональные примеси — «классы-фабрики».

## Простая примесь

Примесь — это обычный объект с методами. Скопировать их в прототип класса можно через `Object.assign`.

```js
const sayHiMixin = {
  sayHi() {
    return `Привет, ${this.name}!`;
  },
  sayBye() {
    return `Пока, ${this.name}!`;
  },
};

class User {
  constructor(name) {
    this.name = name;
  }
}

Object.assign(User.prototype, sayHiMixin);

const user = new User('Анна');

console.log(user.sayHi()); // Привет, Анна!
console.log(Object.hasOwn(User.prototype, 'sayBye')); // true
console.log(user instanceof User); // true
```

Методы теперь лежат в `User.prototype` и доступны всем экземплярам. Копирование идёт значениями методов, а не ссылками на примесь, так что последующие изменения `sayHiMixin` на уже подмешанные методы не влияют.

Примесь может использовать другие примеси и вызывать методы класса через `this`. Например, примесь событий:

```js
const eventMixin = {
  on(name, handler) {
    this._handlers ??= {};
    (this._handlers[name] ??= []).push(handler);
  },
  trigger(name, ...args) {
    (this._handlers?.[name] ?? []).forEach((h) => h.apply(this, args));
  },
};

class Menu {
  choose(value) {
    this.trigger('select', value);
  }
}
Object.assign(Menu.prototype, eventMixin);

const menu = new Menu();
const log = [];
menu.on('select', (value) => log.push(value));
menu.choose('кофе');
menu.choose('чай');

console.log(log); // [ 'кофе', 'чай' ]
```

Здесь `this._handlers` создаётся на экземпляре, а не на прототипе — иначе все меню делили бы один набор обработчиков.

## Ограничения Object.assign

`Object.assign` копирует только собственные перечислимые свойства. Методы класса неперечислимы, поэтому сделать примесью класс таким способом не получится.

```js
class Walker {
  walk() {
    return 'иду';
  }
}
class Person {}

Object.assign(Person.prototype, Walker.prototype);
console.log(typeof new Person().walk); // undefined
```

Кроме того, геттеры и сеттеры в примеси будут вызваны во время копирования, и в целевой объект попадут их значения. Если нужны и они, копируйте через дескрипторы:

```js
const mixin = {
  get label() {
    return `№${this.id}`;
  },
};
class Item {
  constructor(id) {
    this.id = id;
  }
}

Object.defineProperties(Item.prototype, Object.getOwnPropertyDescriptors(mixin));

console.log(new Item(7).label); // №7
```

## Примесь с super

Если примесь — объект с `super`, то `super` привязывается к `[[HomeObject]]` — самому объекту примеси, а не классу, в который его скопировали. Поэтому в примесях можно использовать `Object.setPrototypeOf(mixin, Parent)` для явной связи.

```js
const base = {
  hi() {
    return 'base';
  },
};
const mixin = {
  __proto__: base,
  hi() {
    return super.hi() + '+mixin';
  },
};
class Greeter {}
Object.assign(Greeter.prototype, mixin);

console.log(new Greeter().hi()); // base+mixin
```

Метод ищет `super` через прототип самой примеси (`base`) — независимо от того, в какой класс его скопировали. Это свойство надо помнить: оно и полезно, и неожиданно.

## Функциональные примеси

Чаще всего в реальном коде примесью называют **функцию, принимающую базовый класс и возвращающую расширенный**. Тогда `super` работает естественно, а порядок применения задаёт цепочку наследования.

```js
const Serializable = (Base) =>
  class extends Base {
    serialize() {
      return JSON.stringify(this);
    }
  };

const Comparable = (Base) =>
  class extends Base {
    compareTo(other) {
      return this.value - other.value;
    }
  };

class Point {
  constructor(value) {
    this.value = value;
  }
}
class SmartPoint extends Serializable(Comparable(Point)) {}

const a = new SmartPoint(3);
const b = new SmartPoint(5);

console.log(a.serialize()); // {"value":3}
console.log(a.compareTo(b)); // -2
console.log(a instanceof Point); // true
console.log(a instanceof SmartPoint); // true
```

Преимущества такого подхода: работают `super` и `instanceof` по цепочке, примесь можно применить к любому классу, а несколько примесей выстраиваются в понятный порядок. Неудобство: примесь в цепочке — это скрытые промежуточные классы, и проверить, что «подмешано именно `Serializable`», `instanceof` не позволит. Для этого можно определить `Symbol.hasInstance` или пометить объект флагом.

Можно написать вспомогательную функцию для сборки нескольких примесей по порядку:

```js
const mix = (Base, ...mixins) => mixins.reduce((cls, m) => m(cls), Base);

class Plain {}
const Loud = (B) => class extends B { shout() { return 'ААА'; } };
const Quiet = (B) => class extends B { whisper() { return 'тсс'; } };

const Both = mix(Plain, Loud, Quiet);
const x = new Both();

console.log(x.shout(), x.whisper()); // ААА тсс
console.log(x instanceof Plain); // true
```

## Конфликты имён

Если две примеси определяют метод с одним именем, победит та, что применена позже (при `Object.assign` — позже в списке; при функциональных примесях — внешняя). Чтобы не затирать чужое молча, используйте `super` в функциональных примесях или давайте методам уникальные имена.

```js
const A = (B) => class extends B { who() { return 'A'; } };
const C = (B) => class extends B { who() { return super.who() + 'C'; } };

class Root {}
class Final extends C(A(Root)) {}

console.log(new Final().who()); // AC
```

## Типичные ошибки

> [!WARNING]
>
> - Пытаться «подмешать» класс через `Object.assign(Target.prototype, Source.prototype)`: методы класса неперечислимы и не скопируются.
> - Хранить изменяемое состояние в самой примеси: оно станет общим. Состояние кладут в `this`.
> - Забывать про порядок: при конфликте имён последняя примесь перекрывает предыдущие.
> - Ожидать, что `instanceof` проверит наличие примеси.

## Коротко

> [!TIP]
>
> - Примесь — набор методов, который добавляют в класс без наследования; так обходят «один родитель».
> - Простая форма — `Object.assign(Class.prototype, mixin)`; геттеры и сеттеры копируйте через дескрипторы.
> - Состояние должно жить на экземпляре (`this`), а не в примеси.
> - Функциональные примеси `(Base) => class extends Base {}` дают рабочий `super` и понятный порядок.
> - При совпадении имён побеждает позднее применённая примесь.

---

*Тема по мотивам статьи [«Примеси»](https://learn.javascript.ru/mixins) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
