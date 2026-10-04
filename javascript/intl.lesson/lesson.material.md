# Intl: форматирование и сравнение по правилам языка

Дата `03/05/2024` в США — 5 марта, а в Великобритании — 3 мая. Число `1,234.5` в Германии пишется `1.234,5`. Слово «файл» после числа меняет форму: 1 файл, 2 файла, 5 файлов. Писать такие правила руками для каждого языка нереально, поэтому в JavaScript есть встроенное пространство имён `Intl` (Internationalization API) — набор классов, знающих правила форматирования для сотен локалей.

## Локаль и общий шаблон

Все классы `Intl` устроены одинаково: конструктор принимает **локаль** (строку BCP 47 вроде `'en-US'`, `'ru'`, `'de-DE'`, либо массив запасных вариантов) и **объект опций**, а у экземпляра есть рабочие методы (`format`, `compare`, `select` и т. д.) и `resolvedOptions()`, показывающий, какие настройки реально применились.

```js
const nf = new Intl.NumberFormat('en-US');
console.log(nf.format(1234567.891)); // 1,234,567.891
console.log(nf.resolvedOptions().locale); // en-US
```

Локаль нужно указывать явно. Если не указать, будет использована локаль среды (в браузере — язык пользователя, в Node — системная), и результат станет зависеть от машины. Неверный тег (`'en_US'` с подчёркиванием) приводит к `RangeError`. Проверить допустимость можно через `Intl.getCanonicalLocales('EN-us')` (вернёт `['en-US']`) и `supportedLocalesOf`.

Экземпляр создаётся дорого, а использовать его можно многократно, поэтому в цикле создавайте форматтер один раз.

## Intl.NumberFormat: числа, валюты, проценты

```js
console.log(new Intl.NumberFormat('en-US').format(1234567.891)); // 1,234,567.891
console.log(new Intl.NumberFormat('de-DE').format(1234567.891)); // 1.234.567,891
console.log(new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(1234.5)); // $1,234.50
console.log(new Intl.NumberFormat('en-US', { style: 'percent' }).format(0.256)); // 26%
console.log(new Intl.NumberFormat('en-US', { notation: 'compact' }).format(1234567)); // 1.2M
console.log(new Intl.NumberFormat('en-US', { style: 'unit', unit: 'kilometer-per-hour' }).format(50)); // 50 km/h
```

Важные опции:

- `style`: `'decimal'` (по умолчанию), `'currency'`, `'percent'`, `'unit'`. Для `'currency'` обязательна опция `currency` (код ISO 4217, например `'EUR'`): без неё будет `TypeError`, с недопустимым кодом (`'US'`) — `RangeError`.
- `minimumFractionDigits`, `maximumFractionDigits` — знаки после запятой. `{ minimumFractionDigits: 2 }` превращает `5` в `5.00`.
- `maximumSignificantDigits` — значащие цифры: `123456` с лимитом 3 станет `123,000`.
- `minimumIntegerDigits` — дополнение нулями: `7` → `007`.
- `useGrouping: false` — убрать разделители групп.
- `signDisplay: 'always'` — всегда показывать знак (`+5`).

Округление — «половина от нуля»: `2.5` → `3`, `3.5` → `4` (не банковское). Ещё одна особенность: многие локали используют в качестве разделителей **неразрывные пробелы** (в русской локали — `U+00A0`, во французской — узкий `U+202F`). Они выглядят как пробелы, но `===` с обычным пробелом не сработает. В тестах и сравнении строк приводите их так: `s.replace(/\s/g, ' ')`. И не завязывайте логику на точный формат вывода — он может отличаться между версиями ICU.

Метод `formatToParts` возвращает массив частей (`{ type, value }`), если нужно стилизовать, например, только дробную часть. Есть он и у `DateTimeFormat`.

## Intl.DateTimeFormat: даты и время

```js
const d = new Date(Date.UTC(2024, 2, 5, 14, 7, 9));
const opts = { timeZone: 'UTC' };
console.log(new Intl.DateTimeFormat('en-US', opts).format(d)); // 3/5/2024
console.log(new Intl.DateTimeFormat('ru-RU', opts).format(d)); // 05.03.2024
console.log(new Intl.DateTimeFormat('de-DE', opts).format(d)); // 5.3.2024
console.log(new Intl.DateTimeFormat('en-US', { ...opts, year: 'numeric', month: 'long', day: 'numeric' }).format(d)); // March 5, 2024
console.log(new Intl.DateTimeFormat('en-US', { ...opts, weekday: 'long' }).format(d)); // Tuesday
console.log(new Intl.DateTimeFormat('en-US', { ...opts, hour: '2-digit', minute: '2-digit', hour12: false }).format(d)); // 14:07
console.log(new Intl.DateTimeFormat('en-US', { timeZone: 'Asia/Tokyo', hour: 'numeric', hour12: false }).format(d)); // 23
```

Главная идея: вы указываете **какие компоненты** хотите увидеть (`year`, `month`, `day`, `weekday`, `hour`, `minute`, `second`, `timeZoneName`) и в каком виде (`'numeric'`, `'2-digit'`, `'short'`, `'long'`), а порядок и разделители подбирает локаль. Есть и готовые наборы: `dateStyle` и `timeStyle` (`'full'`, `'long'`, `'medium'`, `'short'`); их нельзя смешивать с отдельными компонентами.

Опция `timeZone` задаёт часовой пояс (IANA-имя вроде `'Europe/Moscow'` или `'UTC'`). Без неё берётся пояс системы, поэтому для воспроизводимости в тестах и на сервере указывайте её явно. Месяц в `Date` нумеруется с нуля, но `Intl` это не затрагивает — он принимает готовый объект даты.

Остерегайтесь `hour12` и `hourCycle` — при `hour12: true` вывод содержит `AM`/`PM`, и в новых версиях ICU между временем и `PM` стоит узкий неразрывный пробел.

Методы `toLocaleString`, `toLocaleDateString` и `toLocaleTimeString` у чисел и дат — короткая запись того же: `(1234.5).toLocaleString('en-US')` даёт `1,234.5`. Принимают те же аргументы, но заново создают форматтер при каждом вызове, поэтому для массовой обработки лучше явный `Intl`-объект.

## Intl.Collator: сортировка и сравнение строк

Оператор `<` и сортировка по умолчанию сравнивают кодовые единицы, из-за чего `['ä', 'z', 'a'].sort()` даст `['a', 'z', 'ä']`. Собственные правила языка знает `Collator`:

```js
const de = new Intl.Collator('de');
const sv = new Intl.Collator('sv');
console.log(de.compare('ä', 'z')); // -1 (в немецком ä идёт рядом с a)
console.log(sv.compare('ä', 'z')); // 1 (в шведском ä в конце алфавита)
console.log(['b', 'A', 'a', 'B'].sort(new Intl.Collator('en').compare)); // [ 'a', 'A', 'b', 'B' ]
```

Метод `compare` привязан к экземпляру, его можно сразу передавать в `sort`. Опции:

- `numeric: true` — числа внутри строк сравниваются как числа: `['a10','a2','a1']` → `['a1','a2','a10']`.
- `sensitivity`: `'base'` игнорирует диакритику и регистр (`'a'` и `'á'`, `'a'` и `'A'` равны), `'accent'` учитывает диакритику, но не регистр, `'case'` — регистр, но не диакритику, `'variant'` (по умолчанию для сортировки) учитывает всё.
- `caseFirst`, `ignorePunctuation` — тонкая настройка.

Не забывайте, что `sort` изменяет массив на месте. Если исходный порядок нужен, сначала копируйте: `[...arr].sort(...)`.

## Intl.PluralRules: формы множественного числа

В английском две формы (`one`, `other`), в русском для целых — три (`one`, `few`, `many`), в других языках — до шести. `PluralRules` по числу возвращает категорию, а подставлять слово вы будете сами.

```js
const en = new Intl.PluralRules('en');
console.log(en.select(1), en.select(2), en.select(0)); // one other other
const ru = new Intl.PluralRules('ru');
console.log([0, 1, 2, 5, 11, 21, 22, 25].map((n) => ru.select(n)).join()); // many,one,few,many,many,one,few,many
const ord = new Intl.PluralRules('en', { type: 'ordinal' });
console.log([1, 2, 3, 4, 11, 21].map((n) => ord.select(n)).join()); // one,two,few,other,other,one
```

Категории: `zero`, `one`, `two`, `few`, `many`, `other`. Для порядковых числительных (`type: 'ordinal'`) в английском: 1st, 2nd, 3rd, 4th, 11th, 21st.

## Intl.RelativeTimeFormat и Intl.ListFormat

```js
const rt = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });
console.log(rt.format(-1, 'day'), rt.format(1, 'day'), rt.format(0, 'day')); // yesterday tomorrow today
console.log(rt.format(-3, 'month')); // 3 months ago
console.log(new Intl.RelativeTimeFormat('en').format(-1, 'day')); // 1 day ago
console.log(new Intl.ListFormat('en').format(['a', 'b', 'c'])); // a, b, and c
console.log(new Intl.ListFormat('en', { type: 'disjunction' }).format(['a', 'b', 'c'])); // a, b, or c
```

`RelativeTimeFormat.format(value, unit)` принимает число (отрицательное — прошлое) и единицу (`'second'`, `'minute'`, `'hour'`, `'day'`, `'week'`, `'month'`, `'year'`). Опция `numeric: 'auto'` включает слова «вчера/завтра» вместо «1 день назад». Сама разность дат вычисляется вами.

`ListFormat` склеивает список с правильными союзами: `type: 'conjunction'` («и»), `'disjunction'` («или»).

## Intl.DisplayNames и Intl.Segmenter

`new Intl.DisplayNames('en', { type: 'region' }).of('RU')` вернёт `Russia`; типы `language`, `currency`, `script` аналогичны (`of('de')` → `German`, `of('EUR')` → `Euro`). `Intl.Segmenter` делит текст на графемы, слова и предложения (см. урок про Юникод).

## Типичные ошибки

- Не указывать локаль и `timeZone`: результат «плавает» между машинами.
- Сравнивать результат форматирования с литералом, не учитывая неразрывные пробелы.
- Создавать `new Intl.NumberFormat` в горячем цикле.
- Сортировать `localeCompare`/`Collator` и забывать, что `sort` мутирует массив.
- Путать `style: 'currency'` без `currency` (будет `TypeError`).
- Строить форму слова через `n % 10 === 1` вручную, вместо `PluralRules`.

## Коротко

- `Intl` содержит `NumberFormat`, `DateTimeFormat`, `Collator`, `PluralRules`, `RelativeTimeFormat`, `ListFormat`, `DisplayNames`, `Segmenter`.
- Локаль и часовой пояс задавайте явно; неверный тег локали — `RangeError`.
- Формат определяется набором запрошенных компонентов, а не строкой-шаблоном.
- `Collator` с `numeric` и `sensitivity` даёт правильную сортировку и поиск без учёта регистра/диакритики.
- `PluralRules` возвращает категорию (`one`, `few`, `many`, …), слово подбираете сами.
- Не полагайтесь на точную строку вывода: версии ICU и пробелы могут отличаться.

---

*Тема по мотивам статьи [«Intl: интернационализация в JavaScript»](https://learn.javascript.ru/intl) учебника learn.javascript.ru (Илья Кантор, CC BY-NC); материал переработан.*
