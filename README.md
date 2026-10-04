# Dolphy courses

Open course library for [Dolphy](https://github.com/dolphy-app). Each top-level directory is a course in the
KnowledgeBase format (`course_manifest.json`, `<lesson>.lesson/` directories).

## Courses

| Directory     | Course id    | Language | Contents                                                                 |
| ------------- | ------------ | -------- | ------------------------------------------------------------------------ |
| `javascript/` | `javascript` | Russian  | The JavaScript language: from variables to async, generators and modules |

## Use in Dolphy

Courses screen → **Add from Git** → `https://github.com/dolphy-app/dolphy-courses`. The branch field can stay empty
(default branch). Update the course later from **Settings → Library → Repositories**.

The `javascript` course uses two exercise types shipped with the app: `dolphy.choice` (multiple choice) and
`dolphy.js` (code checked by running tests). Every exercise is checked automatically.

## Validate

With the monorepo of Dolphy checked out:

```sh
pnpm -F @dolphy-app/engine engine-cli validate <path-to-this-repo> --run-checks --extensions <dolphy>/apps/desktop/extensions
```

## Exercise format (short)

`dolphy.js` exercises keep the tests and the reference solution inside the exercise front matter, so a course works
wherever the repository is mounted:

```yaml
---
engine:
  exercise:
    type: dolphy.js
    spec:
      starter: |
        function sum(a, b) {}
      tests: |
        test('adds numbers', () => assert.equal(sum(1, 2), 3));
      reference: |
        function sum(a, b) { return a + b; }
---
Write `sum(a, b)`.
```

Available in `tests`: `test(name, fn)`, `assert` (`node:assert/strict`), `logs` (the learner's `console.log` lines),
`sleep(ms)`. Learner code and tests run in a fresh Node.js `vm` context with a timeout.

## License

CC BY-NC 4.0, see `LICENSE` and `NOTICE` (the course is derived from a non-commercial tutorial).
