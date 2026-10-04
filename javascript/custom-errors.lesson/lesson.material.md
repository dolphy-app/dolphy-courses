# Пользовательские ошибки

Встроенных `Error`, `TypeError` и `RangeError` хватает не всегда. Когда функция чтения данных может упасть из-за неверного формата, отсутствующего поля или сетевого сбоя, вызывающему коду нужно отличать эти случаи друг от друга. Для этого создают собственные классы ошибок, наследуясь от `Error`. После урока вы сможете строить иерархию ошибок, различать их через `instanceof` и оборачивать низкоуровневые ошибки в понятные.

## Расширяем Error

Свой класс ошибки — обычное наследование. Главное — вызвать `super(message)`, чтобы у объекта появились `message` и `stack`, и выставить `name`:

```js
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
  }
}

const e = new ValidationError('Неверное значение');
console.log(e.name); // ValidationError
console.log(e.message); // Неверное значение
console.log(e instanceof ValidationError); // true
console.log(e instanceof Error); // true
console.log(String(e)); // ValidationError: Неверное значение
```

Зачем выставлять `name`? Если его не задать, он будет унаследован от `Error.prototype`, то есть `"Error"`, и в логах вы увидите безликое `Error: ...`:

```js
class Plain extends Error {}
const p = new Plain('x');
console.log(p.name); // Error
console.log(String(p)); // Error: x
```

Класс при этом всё равно настоящий: `p instanceof Plain` даст `true`. Разница только в подписи.

Способ выставлять имя автоматически — читать его из конструктора. Тогда в наследниках не нужно повторять присваивание:

```js
class AppError extends Error {
  constructor(message) {
    super(message);
    this.name = this.constructor.name;
  }
}
```

Привязка к `constructor.name` ломается, если код минифицируют и имена классов сокращаются. Для кода, который не проходит минификацию (серверный код на Node.js, как правило), это безопасно; иначе задавайте имя строкой.

Обратите внимание: `this.name = ...` создаёт собственное свойство экземпляра, поэтому оно попадёт в `Object.keys(error)` и в `JSON.stringify(error)`. Встроенные `message` и `stack` — несчётные, их там нет.

```js
class PropertyRequiredError extends AppError {
  constructor(property) {
    super('Нет свойства: ' + property);
    this.property = property;
  }
}

const err = new PropertyRequiredError('age');
console.log(err.name); // PropertyRequiredError
console.log(err.property); // age
console.log(JSON.stringify(err)); // {"name":"PropertyRequiredError","property":"age"}
```

## Иерархия и instanceof

Сила наследования в том, что обработчик может выбирать уровень детализации. Одна ветка ловит конкретную ошибку, другая — любую из семейства, третья — всё остальное:

```js
class HttpError extends Error {
  constructor(status, message) {
    super(message ?? 'HTTP ' + status);
    this.name = 'HttpError';
    this.status = status;
  }
}

class NotFoundError extends HttpError {
  constructor() {
    super(404, 'Not found');
    this.name = 'NotFoundError';
  }
}

function describe(err) {
  if (err instanceof NotFoundError) return 'not found';
  if (err instanceof HttpError) return 'http error';
  return 'unknown';
}

console.log(describe(new NotFoundError())); // not found
console.log(describe(new HttpError(500))); // http error
console.log(describe(new Error('x'))); // unknown
```

Порядок проверок важен: сначала более конкретный класс, потом более общий. Если поставить `HttpError` первым, `NotFoundError` никогда до своей ветки не дойдёт, ведь он тоже `HttpError`. Не используйте для различия ошибок текст `message` — его легко поменять, а `instanceof` проверяет сам тип.

## Обёртывание исключений

Представьте функцию `readUser(json)`. Внутри она вызывает `JSON.parse`, и тот может бросить `SyntaxError`. Но вызывающему коду неинтересно, что именно парсилось: ему нужно знать, что чтение не удалось. Поэтому низкоуровневую ошибку «оборачивают» в свою, более высокоуровневую, сохраняя исходную как причину. Для этого у `Error` есть второй аргумент с опцией `cause`:

```js
class ReadError extends Error {
  constructor(message, options) {
    super(message, options);
    this.name = 'ReadError';
  }
}

function readUser(json) {
  try {
    return JSON.parse(json);
  } catch (err) {
    throw new ReadError('Не удалось прочитать пользователя', { cause: err });
  }
}

try {
  readUser('{');
} catch (err) {
  console.log(err.name); // ReadError
  console.log(err.cause.name); // SyntaxError
  console.log(err.cause instanceof SyntaxError); // true
}
```

Свойство `cause` не перечисляется в `Object.keys`, но хранится в объекте, и по цепочке `err.cause.cause...` можно дойти до первопричины. Раньше (и в старых окружениях) авторы сохраняли исходную ошибку вручную, например в `this.cause`; в современном Node.js и браузерах удобнее стандартная опция.

Обёртки дают два преимущества. Во-первых, внешний код зависит только от ваших классов, а не от деталей реализации (завтра вместо `JSON.parse` будет другой парсер, а `catch` останется прежним). Во-вторых, в журнале остаётся полная история: что случилось на уровне задачи и что — на уровне низкоуровневого вызова.

## Наследуем от встроенных подтипов

Можно наследоваться и от `SyntaxError`, `TypeError` и других. Тогда экземпляр проходит проверки и на ваш класс, и на родительский встроенный:

```js
class FormatError extends SyntaxError {
  constructor(message) {
    super(message);
    this.name = 'FormatError';
  }
}

const f = new FormatError('плохой формат');
console.log(f instanceof FormatError); // true
console.log(f instanceof SyntaxError); // true
console.log(f instanceof Error); // true
```

Это полезно, когда существующий код уже умеет обрабатывать `SyntaxError`, а вы хотите дать ему более точную подпись.

## Типичные ошибки

- Забыть `super(message)` в конструкторе: сначала будет `ReferenceError` при обращении к `this`, а если `super` вызвать без сообщения, `message` останется пустым.
- Не задать `name`: в логах все ваши ошибки выглядят как `Error`.
- Различать ошибки по тексту сообщения вместо `instanceof` или специального поля (`code`, `status`).
- Ловить родительский класс раньше дочернего в цепочке `if`.
- Проглотить исходную ошибку при оборачивании и потерять причину: передавайте `{ cause: err }`.

## Коротко

- Свой класс ошибки — `class MyError extends Error`, в конструкторе `super(message)` и `this.name`.
- Иерархия ошибок позволяет обрабатывать их на нужном уровне через `instanceof`; сначала проверяйте дочерние классы.
- Дополнительные данные (код, статус, свойство) храните в полях ошибки.
- Оборачивайте низкоуровневые ошибки в свои и передавайте исходную через `{ cause }`.
- Можно наследоваться и от встроенных `SyntaxError`, `TypeError` и т. д.

---

*Тема по мотивам статьи [«Пользовательские ошибки»](https://learn.javascript.ru/custom-errors) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
