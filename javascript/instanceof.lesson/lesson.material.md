# Проверка класса: instanceof

Часто нужно узнать, к какому «типу» относится значение, чтобы обработать его по-разному. Для примитивов хватает `typeof`, а для объектов есть `instanceof`. В уроке разберём, как он работает, как его поведение можно изменить через `Symbol.hasInstance`, чем отличается от `isPrototypeOf`, и какой способ проверки выбрать в какой ситуации.

## Оператор instanceof

Выражение `obj instanceof Class` возвращает `true`, если `Class.prototype` находится где-то в цепочке прототипов `obj`. Работает и с классами, и с функциями-конструкторами, и со встроенными типами.

```js
class Rabbit {}
const rabbit = new Rabbit();

console.log(rabbit instanceof Rabbit); // true
console.log(rabbit instanceof Object); // true
console.log([] instanceof Array); // true
console.log([] instanceof Object); // true
console.log(function () {} instanceof Function); // true
console.log('abc' instanceof String); // false
console.log(new String('abc') instanceof String); // true
```

Примитивы никогда не являются экземплярами (`'abc' instanceof String` — `false`). А для наследования проверка срабатывает по всей цепочке: экземпляр потомка — экземпляр и родителя.

```js
class Animal {}
class Dog extends Animal {}
const dog = new Dog();

console.log(dog instanceof Dog); // true
console.log(dog instanceof Animal); // true
console.log(new Animal() instanceof Dog); // false
```

### Как работает алгоритм

Если у класса нет особого метода `Symbol.hasInstance`, движок делает почти такой цикл: берёт `obj.__proto__`, сравнивает с `Class.prototype`, затем идёт выше, пока не найдёт совпадение или не упрётся в `null`.

```js
function myInstanceOf(obj, Class) {
  if (obj === null || typeof obj !== 'object' && typeof obj !== 'function') {
    return false;
  }
  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === Class.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}

class A {}
class B extends A {}

console.log(myInstanceOf(new B(), A)); // true
console.log(myInstanceOf({}, A)); // false
console.log(myInstanceOf(5, Number)); // false
```

Из этого следует, что результат зависит от `Class.prototype` **в момент проверки**. Если подменить `prototype` после создания объекта, `instanceof` перестанет считать его экземпляром.

```js
function Rabbit() {}
const rabbit = new Rabbit();
console.log(rabbit instanceof Rabbit); // true

Rabbit.prototype = {};
console.log(rabbit instanceof Rabbit); // false
```

## Symbol.hasInstance

Если у правого операнда есть статический метод `Symbol.hasInstance`, `instanceof` просто вызывает его и приводит результат к логическому типу. Это позволяет описать собственную логику проверки, например «утиную типизацию».

```js
class CanFly {
  static [Symbol.hasInstance](obj) {
    return typeof obj?.fly === 'function';
  }
}

const bird = { fly() {} };
const stone = {};

console.log(bird instanceof CanFly); // true
console.log(stone instanceof CanFly); // false
```

Большинство функций такого метода в явном виде не имеют: используется встроенная реализация `Function.prototype[Symbol.hasInstance]`, которая и выполняет алгоритм обхода цепочки.

## isPrototypeOf и сравнение прототипов

Метод `A.prototype.isPrototypeOf(obj)` делает то же самое, но без участия конструктора: он принимает любой объект-прототип, в том числе созданный через `Object.create`. Для проверки «строго этот класс, а не потомок» сравнивают прототип напрямую.

```js
const animal = { eats: true };
const rabbit = Object.create(animal);

console.log(animal.isPrototypeOf(rabbit)); // true
console.log(Object.prototype.isPrototypeOf(rabbit)); // true
console.log(Object.getPrototypeOf(rabbit) === animal); // true

class Animal {}
class Dog extends Animal {}
const dog = new Dog();

console.log(dog instanceof Animal); // true
console.log(Object.getPrototypeOf(dog) === Animal.prototype); // false
```

## Другие способы проверки типа

| Способ | Подходит для | Замечание |
|---|---|---|
| `typeof` | примитивов, функций | для `null` даёт `'object'`, для массивов тоже `'object'` |
| `instanceof` | объектов своих классов и встроенных | не работает с примитивами; проблема с несколькими окнами/реалмами |
| `Array.isArray` | массивов | надёжнее `instanceof Array` |
| `Object.prototype.toString.call(x)` | встроенных типов | возвращает `[object Тип]`, можно менять через `Symbol.toStringTag` |
| `#field in obj` | экземпляров конкретного класса | самая строгая проверка для объектов с приватными полями |

`Symbol.toStringTag` позволяет настроить результат `Object.prototype.toString`:

```js
const report = {
  [Symbol.toStringTag]: 'Report',
};

console.log(Object.prototype.toString.call(report)); // [object Report]
console.log({}.toString.call([])); // [object Array]
console.log(Object.prototype.toString.call(null)); // [object Null]
```

Нюанс с несколькими реалмами (в браузере — iframe, в Node.js — модуль `vm`): у каждого реалма свои `Array`, `Object`. Массив, созданный в другом реалме, не является `instanceof Array` текущего, зато `Array.isArray` его распознает.

## Типичные ошибки

> [!WARNING]
>
> - Применять `instanceof` к примитивам и ожидать `true`.
> - Забывать, что подмена `Class.prototype` ломает проверку для старых объектов.
> - Использовать `x instanceof Array` вместо `Array.isArray(x)` для данных из других окон.
> - Путать «является экземпляром» и «создан именно этим конструктором»: `instanceof` смотрит всю цепочку.
> - Проверять правую часть: справа должна быть функция (или объект с `Symbol.hasInstance`), иначе `TypeError`.

## Коротко

> [!TIP]
>
> - `obj instanceof C` истинно, если `C.prototype` присутствует в цепочке прототипов `obj`.
> - Примитивы не являются экземплярами; потомки являются экземплярами родителей.
> - Поведение настраивается статическим методом `Symbol.hasInstance`.
> - `proto.isPrototypeOf(obj)` — то же без конструктора; точный класс проверяют сравнением `Object.getPrototypeOf(obj)`.
> - Для встроенных типов используйте `Array.isArray` и `Object.prototype.toString.call`.

---

*Тема по мотивам статьи [«Проверка класса: "instanceof"»](https://learn.javascript.ru/instanceof) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
