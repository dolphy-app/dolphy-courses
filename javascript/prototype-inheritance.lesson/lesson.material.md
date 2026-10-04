# Прототипное наследование

Объекты в JavaScript умеют «заимствовать» свойства и методы у других объектов. Этот механизм называется прототипным наследованием, и на нём построено всё остальное: конструкторы, классы, встроенные типы вроде массивов. После урока вы будете понимать, как движок ищет свойство по цепочке прототипов и почему запись ведёт себя не так, как чтение.

## Скрытое свойство [[Prototype]]

У каждого объекта есть внутренняя ссылка `[[Prototype]]`: либо на другой объект, либо `null`. Если прочитать свойство, которого в самом объекте нет, движок идёт по этой ссылке и ищет дальше. Так продолжается, пока свойство не найдено или цепочка не закончилась.

Прочитать и изменить ссылку можно через `Object.getPrototypeOf` и `Object.setPrototypeOf`. Исторически есть и «геттер/сеттер» `__proto__`: он работает, но считается устаревшим способом доступа, в новом коде лучше использовать функции.

```js
const animal = {
  eats: true,
  walk() {
    return 'идёт';
  },
};
const rabbit = { jumps: true };

Object.setPrototypeOf(rabbit, animal);

console.log(rabbit.eats); // true
console.log(rabbit.walk()); // идёт
console.log(Object.getPrototypeOf(rabbit) === animal); // true
console.log(Object.hasOwn(rabbit, 'eats')); // false
```

`rabbit.eats` не лежит в `rabbit`, но находится у `animal`, поэтому чтение успешно. Цепочка может быть длиннее: у `animal` тоже есть свой прототип (по умолчанию `Object.prototype`), а у него — `null`. Это конец цепочки.

```js
const a = { fromA: 1 };
const b = { fromB: 2 };
const c = { fromC: 3 };
Object.setPrototypeOf(b, a);
Object.setPrototypeOf(c, b);

console.log(c.fromA + c.fromB + c.fromC); // 6
console.log(c.missing); // undefined
```

Есть два ограничения. Во-первых, циклы запрещены: попытка замкнуть цепочку вызовет `TypeError`. Во-вторых, `[[Prototype]]` — это либо объект, либо `null`; примитив при присваивании через `setPrototypeOf` будет отвергнут.

```js
const x = {};
const y = Object.create(x);
try {
  Object.setPrototypeOf(x, y);
} catch (error) {
  console.log(error.name); // TypeError
}
```

## Запись не ходит по цепочке

Чтение использует прототипы, а вот запись и удаление работают только с самим объектом. Присваивание создаёт собственное свойство, даже если у прототипа такое уже есть, а прототип остаётся нетронутым.

```js
const base = { color: 'green' };
const item = Object.create(base);

item.color = 'red';

console.log(item.color); // red
console.log(base.color); // green
delete item.color;
console.log(item.color); // green
```

После `delete` собственное свойство исчезло, и снова стало «видно» свойство прототипа.

### Исключение: аксессоры

Если в прототипе найден геттер/сеттер, присваивание вызывает сеттер, а не создаёт собственное свойство. Но внутри сеттера `this` — это объект, на котором вызвали операцию, а не тот, где аксессор описан. Поэтому данные, записанные через `this`, оказываются у «наследника».

```js
const user = {
  name: 'Анна',
  surname: 'Иванова',
  get fullName() {
    return `${this.name} ${this.surname}`;
  },
  set fullName(value) {
    [this.name, this.surname] = value.split(' ');
  },
};
const admin = { __proto__: user, isAdmin: true };

console.log(admin.fullName); // Анна Иванова
admin.fullName = 'Пётр Сидоров';
console.log(admin.fullName); // Пётр Сидоров
console.log(user.fullName); // Анна Иванова
```

## Значение this

Метод, найденный в прототипе, вызывается с `this` равным объекту **перед точкой**, а не владельцу метода. Значит, методы общие, а состояние — у каждого объекта своё.

```js
const stack = {
  push(value) {
    this.items ??= [];
    this.items.push(value);
    return this.items.length;
  },
};
const s1 = Object.create(stack);
const s2 = Object.create(stack);
s1.push(1);
s1.push(2);
s2.push(10);

console.log(s1.items); // [ 1, 2 ]
console.log(s2.items); // [ 10 ]
console.log(stack.items); // undefined
```

Обратите внимание на оператор `??=`: массив создаётся в том объекте, на котором вызван метод. Если бы массив был свойством самого `stack` и мы делали `this.items.push(...)`, то изменение шло бы в общий массив — запись по цепочке не ходит, но **мутация общего объекта** видна всем.

```js
const shared = { tags: [] };
const one = Object.create(shared);
const two = Object.create(shared);
one.tags.push('a');

console.log(two.tags); // [ 'a' ]
console.log(Object.hasOwn(one, 'tags')); // false
```

## Перебор и наследуемые свойства

`for...in` обходит и собственные, и унаследованные **перечислимые** свойства. А вот `Object.keys`, `Object.values`, `Object.entries` и `JSON.stringify` смотрят только на собственные.

```js
const parent = { inherited: 1 };
const child = Object.create(parent);
child.own = 2;

const forIn = [];
for (const key in child) forIn.push(key);

console.log(forIn); // [ 'own', 'inherited' ]
console.log(Object.keys(child)); // [ 'own' ]
console.log(Object.hasOwn(child, 'inherited')); // false
console.log('inherited' in child); // true
```

Оператор `in` проверяет всю цепочку, `Object.hasOwn(obj, key)` — только сам объект. Методы, которые вы видите у `{}` (`toString`, `hasOwnProperty`), не попадают в `for...in`, потому что они неперечислимые.

## Типичные ошибки

- Считать, что присваивание `child.value = 1` изменит прототип. Оно создаёт собственное свойство `child`.
- Забыть, что объекты и массивы в прототипе общие: `child.list.push(x)` меняет данные для всех наследников.
- Пытаться замкнуть цепочку в цикл или назначить прототипом примитив.
- Менять прототип «на лету» у уже используемого объекта через `setPrototypeOf`: это замедляет код в движках, лучше задавать прототип при создании.

## Коротко

- `[[Prototype]]` — ссылка на другой объект или `null`; чтение свойства идёт по цепочке до первого совпадения.
- Читать и менять ссылку нужно через `Object.getPrototypeOf` / `Object.setPrototypeOf`; `__proto__` — устаревший доступ.
- Запись и `delete` действуют только на сам объект; исключение — унаследованный сеттер, который вызывается с `this` текущего объекта.
- `this` в методе — объект перед точкой, поэтому методы общие, а состояние раздельное.
- `for...in` захватывает унаследованные перечислимые свойства, `Object.keys` и `Object.hasOwn` — только собственные.

---

*Тема по мотивам статьи [«Прототипное наследование»](https://learn.javascript.ru/prototype-inheritance) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
