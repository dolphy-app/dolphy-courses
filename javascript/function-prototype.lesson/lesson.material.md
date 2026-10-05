# F.prototype

В прошлом уроке прототип мы задавали вручную. Но обычно объекты создают через `new Constructor()`, и прототип выставляется автоматически. Откуда движок его берёт? Из свойства `prototype` функции-конструктора. В уроке разберёмся, как это работает, чем `F.prototype` отличается от `[[Prototype]]` и что такое `constructor`.

## Свойство prototype у функции

Если вызвать функцию через `new`, то у созданного объекта `[[Prototype]]` будет равен `F.prototype` на момент вызова. Обратите внимание: это обычное свойство функции с именем `prototype`, а не скрытая ссылка. Просто у `new` есть соглашение читать именно его.

```js
const animal = {
  eats: true,
};

function Rabbit(name) {
  this.name = name;
}
Rabbit.prototype = animal;

const rabbit = new Rabbit('Белый');

console.log(rabbit.eats); // true
console.log(Object.getPrototypeOf(rabbit) === animal); // true
console.log(Object.hasOwn(rabbit, 'eats')); // false
```

Важная деталь: прототип берётся в момент `new`. Если позже присвоить `Rabbit.prototype` другой объект, то уже созданные кролики останутся со старым прототипом, а новые получат новый. А вот если не заменять объект, а изменять его (добавлять методы), то поменяется всё сразу, ведь прототип — общий.

```js
function User() {}
const first = new User();

User.prototype.hello = function () {
  return 'hi';
};
console.log(first.hello()); // hi

const oldProto = User.prototype;
User.prototype = { bye() { return 'bye'; } };
const second = new User();

console.log(typeof first.bye); // undefined
console.log(typeof second.bye); // function
console.log(Object.getPrototypeOf(first) === oldProto); // true
```

## F.prototype по умолчанию и constructor

У каждой обычной функции (не стрелочной) свойство `prototype` уже есть: это объект с единственным собственным свойством `constructor`, которое ссылается на саму функцию.

```js
function Rabbit() {}

console.log(Rabbit.prototype.constructor === Rabbit); // true
console.log(Object.keys(Rabbit.prototype)); // []
console.log(Object.getOwnPropertyNames(Rabbit.prototype)); // [ 'constructor' ]
```

Свойство `constructor` неперечислимое, поэтому в `for...in` его не видно. Объекты, созданные через `new Rabbit()`, получают `constructor` по цепочке:

```js
function Rabbit(name) {
  this.name = name;
}
const r = new Rabbit('Серый');
const r2 = new r.constructor('Чёрный');

console.log(r.constructor === Rabbit); // true
console.log(r2.name); // Чёрный
console.log(Object.hasOwn(r, 'constructor')); // false
```

Такой приём — создать «такой же» объект, не зная имени конструктора, — работает, пока `constructor` указывает куда надо. А вот JavaScript сам этого не гарантирует: свойство ни на что не влияет в `new` и `instanceof`, это просто обычное поле.

### Ловушка: замена prototype целиком

Если записать в `F.prototype` новый объектный литерал, `constructor` пропадёт, потому что литерал — обычный объект, и его `constructor` берётся из `Object.prototype`.

```js
function Rabbit() {}
Rabbit.prototype = { jumps: true };

const r = new Rabbit();
console.log(r.constructor === Rabbit); // false
console.log(r.constructor === Object); // true
```

Есть два способа сохранить свойство. Первый — дополнять существующий объект, а не заменять его. Второй — вернуть `constructor` вручную. Для точной копии поведения лучше сделать его неперечислимым.

```js
function Rabbit() {}
Rabbit.prototype = { jumps: true };
Object.defineProperty(Rabbit.prototype, 'constructor', {
  value: Rabbit,
  writable: true,
  configurable: true,
});

console.log(new Rabbit().constructor === Rabbit); // true
console.log(Object.keys(Rabbit.prototype)); // [ 'jumps' ]
```

## Методы в прототипе экономят память

Методы, объявленные внутри конструктора, создаются заново для каждого объекта. Методы в `F.prototype` существуют в единственном экземпляре.

```js
function A() {
  this.say = function () {};
}
function B() {}
B.prototype.say = function () {};

console.log(new A().say === new A().say); // false
console.log(new B().say === new B().say); // true
```

Вариант с прототипом экономнее по памяти и позволяет менять поведение для всех объектов разом. Состояние (данные конкретного объекта) при этом по-прежнему хранится в самом объекте, внутри конструктора.

## Стрелочные функции и prototype

У стрелочных функций и у методов в сокращённой записи (`{ m() {} }`) нет свойства `prototype`, и вызвать их через `new` нельзя.

```js
const arrow = () => {};
const obj = { method() {} };

console.log(arrow.prototype); // undefined
console.log(obj.method.prototype); // undefined
try {
  new arrow();
} catch (error) {
  console.log(error.name); // TypeError
}
```

## Типичные ошибки

> [!WARNING]
>
> - Путать `F.prototype` и `[[Prototype]]`. Первое — свойство функции, из которого `new` берёт значение; второе — внутренняя ссылка объекта. Для функции `F` верно, что `Object.getPrototypeOf(F) === Function.prototype`, а вовсе не `F.prototype`.
> - Присваивать `F.prototype = {...}` после создания объектов и ожидать, что они изменятся.
> - Заменять `prototype` литералом и терять `constructor`.
> - Записывать изменяемые данные (массивы, объекты) в `F.prototype`: они станут общими для всех экземпляров.

## Коротко

> [!TIP]
>
> - `new F()` ставит новому объекту `[[Prototype]] = F.prototype` (значение берётся в момент вызова).
> - У обычной функции `F.prototype` по умолчанию содержит только неперечислимый `constructor`, ссылающийся на `F`.
> - Замена `F.prototype` объектом ломает `constructor` и не затрагивает ранее созданные объекты; добавление свойств — видно всем.
> - Методы в `F.prototype` существуют в одном экземпляре; данные конкретного объекта хранятся в нём самом.
> - У стрелочных функций и сокращённых методов `prototype` нет, `new` к ним неприменим.

---

*Тема по мотивам статьи [«F.prototype»](https://learn.javascript.ru/function-prototype) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
