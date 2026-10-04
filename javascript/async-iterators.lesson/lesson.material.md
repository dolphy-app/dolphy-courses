# Асинхронные итераторы и генераторы

Обычные итераторы и генераторы хороши, когда следующее значение можно получить сразу. Но данные часто приходят «порциями со временем»: страницы ответа API, строки из потока, события. Асинхронные итераторы и генераторы позволяют описать такой источник как последовательность и перебирать её циклом `for await..of`, не думая про вложенные промисы и колбэки.

## Асинхронный итератор: что меняется

Чтобы объект можно было перебирать асинхронно, нужны три изменения по сравнению с обычным итерируемым объектом:

1. Метод называется `Symbol.asyncIterator`, а не `Symbol.iterator`.
2. Его `next()` возвращает **промис**, который выполняется значением `{ value, done }`.
3. Перебирают такой объект циклом `for await (const x of obj)`.

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

const range = {
  from: 1,
  to: 3,
  [Symbol.asyncIterator]() {
    let current = this.from;
    const last = this.to;
    return {
      async next() {
        await sleep(5); // имитация ожидания данных
        if (current <= last) return { done: false, value: current++ };
        return { done: true, value: undefined };
      },
    };
  },
};

(async () => {
  for await (const v of range) console.log(v); // 1, затем 2, затем 3
})();
```

Метод `next()` необязательно объявлять `async`: достаточно, чтобы он вернул промис. `async` просто удобен, потому что можно ставить `await` и возвращать обычный объект — он сам обернётся в промис.

Цикл `for await` вызывает `range[Symbol.asyncIterator]()` один раз, затем на каждой итерации вызывает `next()`, дожидается промиса и смотрит на `done`. Следующий `next()` не вызывается, пока тело цикла не завершилось, так что обработка идёт последовательно.

| | Итераторы | Асинхронные итераторы |
|---|---|---|
| Метод получения итератора | `Symbol.iterator` | `Symbol.asyncIterator` |
| `next()` возвращает | `{ value, done }` | промис с `{ value, done }` |
| Цикл | `for..of` | `for await..of` |

`for await` можно использовать только внутри `async`-функции (в модулях — ещё и на верхнем уровне). Синхронные конструкции вроде `...` и `Array.from` ищут `Symbol.iterator` и с асинхронным итерируемым объектом не работают:

```js
// range — объект из примера выше
try {
  [...range];
} catch (e) {
  console.log(e.constructor.name, e.message); // TypeError range is not iterable
}
```

Для сбора значений в массив в Node.js 22 есть `Array.fromAsync(asyncIterable)`, возвращающий промис массива.

## `for await` и обычные итерируемые объекты

`for await..of` умеет работать и с **синхронными** итерируемыми объектами: каждое значение он «разворачивает» через `await`. Поэтому по массиву промисов можно пройти по очереди:

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

(async () => {
  for await (const v of [1, Promise.resolve(2), sleep(1).then(() => 3)]) {
    console.log(v); // 1, затем 2, затем 3
  }
})();
```

Если один из промисов отклонён, цикл бросит исключение в точке итерации, и его можно перехватить обычным `try..catch`.

## Асинхронные генераторы

Писать объект с `next()` вручную долго. Асинхронный генератор делает то же самое короче: добавьте `async` к `function*`, и внутри тела можно использовать и `await`, и `yield`.

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function* seq(from, to) {
  for (let i = from; i <= to; i++) {
    await sleep(5);
    yield i;
  }
}

(async () => {
  for await (const v of seq(1, 3)) console.log(v); // 1, 2, 3
})();
```

Метод `next()` асинхронного генератора возвращает промис, поэтому при ручной работе его нужно ждать:

```js
async function* one() {
  yield 'a';
}

(async () => {
  const g = one();
  console.log(await g.next()); // { value: 'a', done: false }
  console.log(await g.next()); // { value: undefined, done: true }
})();
```

Как и у обычного генератора, вызов `one()` не запускает тело; оно стартует при первом `next()`. Причём до первого `await` или `yield` код выполняется синхронно прямо внутри вызова `next()`:

```js
async function* g() {
  console.log('start');
  yield 1;
}

const it = g();
console.log('created');
const p = it.next();
console.log('called');
p.then((r) => console.log(r.value));
// created
// start
// called
// 1
```

Значение из `return` асинхронного генератора, как и в обычном, цикл `for await` не отдаёт. Если после `yield` стоит промис, генератор сам дождётся его значения: `yield Promise.resolve('p')` отдаст `'p'`. Вызовы `next()`, сделанные подряд без ожидания, ставятся генератором в очередь и обрабатываются по порядку.

Делегирование `yield*` тоже работает: внутри асинхронного генератора оно перебирает и асинхронные, и синхронные итерируемые объекты.

```js
async function* a() {
  yield 1;
}

async function* b() {
  yield* a();
  yield* [2, 3];
}

(async () => {
  const out = [];
  for await (const v of b()) out.push(v);
  console.log(out); // [ 1, 2, 3 ]
})();
```

## Объект, асинхронно перебираемый через генератор

Так же как `*[Symbol.iterator]()` упрощает обычные итерируемые объекты, `async *[Symbol.asyncIterator]()` упрощает асинхронные. Каждый новый `for await` получает свежий генератор, поэтому такой объект можно перебирать повторно:

```js
const feed = {
  async *[Symbol.asyncIterator]() {
    yield 'x';
    yield 'y';
  },
};

(async () => {
  const out = [];
  for await (const v of feed) out.push(v);
  for await (const v of feed) out.push(v);
  console.log(out); // [ 'x', 'y', 'x', 'y' ]
})();
```

## Практика: постраничная выдача

Типичная задача — API отдаёт данные страницами и сообщает, откуда взять следующую. Асинхронный генератор прячет эту механику: снаружи видна просто последовательность элементов.

```js
const pages = [[1, 2], [3], []];
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// имитация запроса: возвращает страницу и номер следующей (или null)
async function fetchPage(i) {
  await sleep(1);
  return { items: pages[i] ?? [], next: i + 1 < pages.length ? i + 1 : null };
}

async function* allItems() {
  let cursor = 0;
  while (cursor !== null) {
    const { items, next } = await fetchPage(cursor);
    yield* items;
    cursor = next;
  }
}

(async () => {
  const out = [];
  for await (const v of allItems()) out.push(v);
  console.log(out); // [ 1, 2, 3 ]
})();
```

Следующая страница запрашивается только тогда, когда потребитель дочитал предыдущую. Если выйти из цикла по `break`, лишних запросов не будет.

## Остановка и ошибки

Если цикл `for await` прерван (`break`, `return`, исключение в теле), у итератора вызывается `return()`, и в асинхронном генераторе выполняются блоки `finally` — можно закрывать соединения и освобождать ресурсы:

```js
async function* res() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log('closed');
  }
}

(async () => {
  for await (const v of res()) {
    console.log(v); // 1
    break;
  }
  // closed
})();
```

Ошибка внутри асинхронного генератора превращается в отклонение промиса от `next()`, и `for await` пробрасывает её как обычное исключение:

```js
async function* bad() {
  yield 1;
  throw new Error('fail');
}

(async () => {
  try {
    for await (const v of bad()) console.log(v); // 1
  } catch (e) {
    console.log('caught', e.message); // caught fail
  }
})();
```

## Типичные ошибки

- Использовать `for..of` вместо `for await..of` для асинхронного генератора: получите `TypeError ... is not iterable`.
- Применять `[...asyncIterable]` и `Array.from(asyncIterable)` — они синхронные. Нужен цикл или `Array.fromAsync`.
- Забыть `await` перед `next()` при ручном обходе: вы получите промис, а не объект `{ value, done }`.
- Ждать, что `for await` обработает значения параллельно. Он последователен: следующая итерация начнётся после завершения тела цикла.
- Делать `yield` внутри колбэка (`forEach`, `then`): `yield` работает только непосредственно в теле генератора.
- Рассчитывать на значение из `return` — цикл его не отдаёт.

## Коротко

- Асинхронно итерируемый объект имеет `Symbol.asyncIterator`; его `next()` возвращает промис с `{ value, done }`; перебор — `for await..of`.
- Асинхронный генератор — `async function*`: в нём можно и `await`, и `yield`; его `next()` возвращает промис.
- `for await` умеет перебирать и синхронные итерируемые объекты, разворачивая промисы по одному.
- Асинхронный генератор подходит для постраничных данных и потоков: источник читается лениво, по мере потребления.
- `break` вызывает `return()` и запускает `finally`; ошибка внутри генератора выходит из `for await` как исключение.

---

*Тема по мотивам статьи [«Асинхронные итераторы и генераторы»](https://learn.javascript.ru/async-iterators-generators) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
