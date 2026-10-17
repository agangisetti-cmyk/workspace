# Jest and Vitest: A Learning Guide for the CUID LWC Migration

A learning guide for the CUID LWC unit-test migration. Written for an EM: enough depth to follow design reviews and PRs, no more.

## Contents

- [TL;DR](#tldr)
- **Part 1: Basics**
  - [What a unit test runner does](#what-a-unit-test-runner-does)
  - [Jest in 5 minutes](#jest-in-5-minutes)
  - [Vitest in 5 minutes](#vitest-in-5-minutes)
  - [Side-by-side comparison](#side-by-side-comparison)
- **Part 2: LWC**
  - [How LWC tests run today (Jest)](#how-lwc-tests-run-today-jest)
  - [How they run on Vitest](#how-they-run-on-vitest)
  - [Migration cheat sheet](#migration-cheat-sheet)
  - [Gotchas that bite](#gotchas-that-bite)
- **Part 3: Our work**
  - [Series A to G decoded](#series-a-to-g-decoded)
  - [Glossary](#glossary)
- **Part 4: Practice**
  - [Hands-on exercises](#hands-on-exercises-about-60-minutes-total)
  - [Resources](#resources)

---

## TL;DR

- **Jest** (Meta, 2014; now OpenJS Foundation) is the long-time default JavaScript test runner. It was built for CommonJS (`require`) and transforms source through Babel.
- **Vitest** (from the Vite team, 2021) is a newer runner built on Vite. It runs ES modules (`import`) natively, has TypeScript support built in, and keeps an API that is mostly compatible with Jest (`describe`, `it`, `expect`, with `vi.*` replacing `jest.*`).
- LWC is ESM-native, so with Vitest we stop converting ESM to CommonJS just so tests can run. Core already has Bazel and Vitest infrastructure, so we would run one modern runner instead of two.
- Most of the work is **not** renaming `jest.fn` to `vi.fn`. The hard part is rebuilding the LWC-specific machinery that `@lwc/jest-*` gave us for free: compiling components, resolving `@salesforce/*` imports, wire adapters, sa11y and snapshots.

---

# Part 1: Basics

## What a unit test runner does

Every runner does the same five jobs. Keeping this list in mind makes the migration easy to follow, because each series of work items replaces one or more of these jobs for LWC.

| # | Job | Example | Where it shows up in our series |
|---|---|---|---|
| 1 | **Find** test files | `**/__tests__/*.spec.js` | C4 opt-in gate, C6 suite selection |
| 2 | **Transform** source into runnable JS | Compile an LWC `.html` template and `.css` into JS | B2, B3, B4, C1 |
| 3 | **Resolve** imports | `@salesforce/label/c.greeting` becomes a stub value | A2 to A9, D1 to D3 |
| 4 | **Run** tests in an environment | jsdom (a fake browser DOM) in an isolated worker | B5, C7, C8, A10 |
| 5 | **Assert and report** | `expect(...)`, snapshots, JUnit XML, a11y checks | B6, E0 to E2, F1, F2 |

## Jest in 5 minutes

```js
// sum.js
export function sum(a, b) { return a + b; }

// sum.test.js
import { sum } from './sum';

describe('sum', () => {
  it('adds numbers', () => {
    expect(sum(1, 2)).toBe(3);
  });
});
```

Key ideas:

- **Globals.** `describe`, `it`, `expect` and `jest` are available everywhere without importing them.
- **Mocks.** `jest.fn()` creates a fake function, `jest.spyOn(obj, 'm')` wraps a real method, and `jest.mock('module')` replaces a whole module.
- **Hoisting.** Babel moves `jest.mock()` calls to the top of the file so they run before imports.
- **Transforms.** Jest runs files through a "transformer" (usually Babel) that outputs CommonJS. Running ESM natively is still experimental in Jest.
- **Config.** `jest.config.js` with `moduleNameMapper` (regex-based import rewriting), `setupFilesAfterEnv`, `testEnvironment: 'jsdom'` and `snapshotSerializers`.
- **Presets.** A preset bundles config. LWC ships `@lwc/jest-preset`.

### Mocks and timers

```js
import { fetchUser } from './api';
import { greet } from './greet';

jest.mock('./api');                      // auto-mock every export

it('greets the fetched user', async () => {
  fetchUser.mockResolvedValue({ name: 'Ajay' });
  await expect(greet()).resolves.toBe('Hello, Ajay');
  expect(fetchUser).toHaveBeenCalledTimes(1);
});

it('debounces', () => {
  jest.useFakeTimers();
  const cb = jest.fn();
  debounce(cb, 300)();
  jest.advanceTimersByTime(300);
  expect(cb).toHaveBeenCalled();
});
```

## Vitest in 5 minutes

```js
// sum.test.js
import { describe, it, expect } from 'vitest';   // explicit by default
import { sum } from './sum';

describe('sum', () => {
  it('adds numbers', () => {
    expect(sum(1, 2)).toBe(3);
  });
});
```

Key ideas:

- **Built on Vite.** Vitest reuses Vite's transform pipeline and plugin system, which is Rollup-compatible. A Rollup plugin like `@lwc/rollup-plugin` can therefore compile LWC for tests, and that is the basis of our B and C series.
- **Native ESM.** Code runs as real ES modules, with no conversion to CommonJS.
- **TypeScript built in** (esbuild strips the types), so no Babel setup is needed.
- **Jest-like API.** `vi.fn`, `vi.spyOn`, `vi.mock` and `vi.useFakeTimers`. Globals are optional (`test.globals: true`).
- **Watch mode** knows the module graph and reruns only the tests affected by a change.
- **Pools and isolation.** Tests run in worker threads or forked processes. `isolate: true` gives every test file a fresh module registry (see C7).
- **Config.** `vitest.config.js` uses `defineConfig({ plugins, resolve.alias, test: { environment, setupFiles, ... } })` (see B1).

### Same test as above, Vitest style

```js
import { it, expect, vi } from 'vitest';
import { fetchUser } from './api';
import { greet } from './greet';

vi.mock('./api');                        // also hoisted, by Vitest itself

it('greets the fetched user', async () => {
  fetchUser.mockResolvedValue({ name: 'Ajay' });
  await expect(greet()).resolves.toBe('Hello, Ajay');
  expect(fetchUser).toHaveBeenCalledTimes(1);
});

it('debounces', () => {
  vi.useFakeTimers();
  const cb = vi.fn();
  debounce(cb, 300)();
  vi.advanceTimersByTime(300);
  expect(cb).toHaveBeenCalled();
});
```

For most tests the change really is this small. The hard part is the LWC-specific infrastructure behind the tests.

## Side-by-side comparison

| Dimension | Jest | Vitest | Why it matters for Core LWC |
|---|---|---|---|
| Module system | CommonJS first. ESM is experimental. | ESM native | LWC source is ESM. Fewer transform layers means fewer cases where tests behave differently from the code they test. |
| Transform pipeline | Babel or custom transformers (`@lwc/jest-transformer`) | Vite plugins (Rollup-compatible), esbuild | We can reuse `@lwc/rollup-plugin`, the same compiler path used to build LWC, instead of a separate Jest-only transformer. |
| TypeScript | Needs Babel or ts-jest | Built in (type-stripping) | Fits the TypeScript Enablement track (see B4). |
| Speed | Mature, but transform and startup costs add up at scale | Usually faster, especially in watch mode | Not guaranteed in jsdom-heavy suites with full isolation. The epic requires benchmarks showing no material regression. |
| API | `jest.*`, globals by default | `vi.*`, explicit imports by default | Most changes are mechanical, so a codemod can do them. |
| Mocking | Mature. `jest.requireActual` is synchronous. Supports `{ virtual: true }`. | `vi.importActual` is **async**. No virtual mocks (the resolver has to handle them). | The main source of non-mechanical migration work (A2, A8, A9, A11). |
| Snapshots | Mature, with custom serializers | Compatible, but small format differences exist | LWC CSS scope tokens make snapshots unstable unless they are handled (E series). |
| Ecosystem | Huge. Every library has a Jest integration. | Large and growing fast | Some integrations had to be vendored or rebuilt (sa11y in F1, wire-service util in G1). |
| Maturity and risk | Very stable, slower to evolve | Fast-moving, with occasional breaking majors | We need version pinning and an upgrade owner. |

> **Is Vitest better?** Vitest is more modern and fits our stack better (ESM, Vite and Rollup, TypeScript, and the Bazel and Vitest infrastructure we already have). It is not automatically faster or safer. The migration only pays off if we keep parity (mocks, wire, a11y, snapshots, CI reporting) and show that tests run at least as fast. That is why the epic has explicit parity and performance gates.

---

# Part 2: LWC

## How LWC tests run today (Jest)

```js
import { createElement } from 'lwc';
import Greeting from 'c/greeting';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';
import LABEL from '@salesforce/label/c.greetingLabel';

jest.mock('@salesforce/apex/AccountController.getAccounts',
  () => ({ default: jest.fn() }), { virtual: true });

describe('c-greeting', () => {
  afterEach(() => { document.body.innerHTML = ''; });

  it('renders the name', async () => {
    const el = createElement('c-greeting', { is: Greeting });
    el.name = 'Ajay';
    document.body.appendChild(el);
    await Promise.resolve();                        // let LWC re-render
    expect(el.shadowRoot.querySelector('p').textContent).toBe('Hello, Ajay');
    await expect(el).toBeAccessible();              // sa11y matcher
  });
});
```

Several packages do work behind the scenes to make this test run:

| Package | What it does | Vitest replacement (series) |
|---|---|---|
| `@lwc/jest-preset` | Bundles the config below | Our new **preset** package (A1, B1) |
| `@lwc/jest-transformer` | Compiles LWC `.js/.html/.css` and stubs `@salesforce/*` imports | `@lwc/rollup-plugin` plus our **plugin** (A2 to A9, B2, B3, C1) |
| `@lwc/jest-resolver` | Resolves `c/foo` and namespace module specifiers | Plugin resolveId plus aliases (D1 to D3) |
| `@lwc/jest-serializer` | Clean, stable snapshots of shadow DOM | E0 to E2 |
| `@salesforce/wire-service-jest-util` | Emits mock data through `@wire` | B7, A10, G1 |
| `@sa11y/jest` | `toBeAccessible()` matcher and automatic checks | `@sa11y/vitest` (F1, F2) |
| Core `ui-tests.bzl` | Bazel rules that run Jest in CI | A third branch for Vitest (C4, C5, C6) |

## How they run on Vitest

```js
// vitest.config.js (simplified idea of what the preset produces)
import { defineConfig } from 'vitest/config';
import lwc from '@lwc/rollup-plugin';
import sfImports from '<our-plugin>';     // resolves @salesforce/* to stubs and spies

export default defineConfig({
  plugins: [sfImports(), lwc({ /* compiler options matching Jest (B3) */ })],
  resolve: { alias: [ /* converted moduleNameMapper, order preserved (D1 to D3) */ ] },
  test: {
    environment: 'jsdom',
    setupFiles: ['lwc-runtime-setup.js', 'sa11y-vitest-setup.js'],   // B5, F2
    isolate: true, pool: 'forks',                                    // C7
    snapshotSerializers: ['lwc-serializer.js'],                     // E1
  },
});
```

```js
// The same component test after migration
import { describe, it, expect, vi, afterEach } from 'vitest';
import { createElement } from 'lwc';
import Greeting from 'c/greeting';

vi.mock('@salesforce/apex/AccountController.getAccounts',
  () => ({ default: vi.fn() }));     // no "virtual": the plugin makes the module resolvable

// ...test body is unchanged...
```

## Migration cheat sheet

| Jest | Vitest | Mechanical? |
|---|---|---|
| `jest.fn()` | `vi.fn()` | Yes |
| `jest.spyOn(o, 'm')` | `vi.spyOn(o, 'm')` | Yes |
| `jest.mock('m', factory)` | `vi.mock('m', factory)` | Mostly. Factories must return a `default` key explicitly for default exports. |
| `jest.mock('m', f, { virtual: true })` | `vi.mock('m', f)` plus a resolver that knows `m` | No. Needs plugin support (A2). |
| `jest.requireActual('m')` | `await vi.importActual('m')` | No. The call is async, so the factory must become async. |
| `jest.useFakeTimers()` / `advanceTimersByTime` | `vi.useFakeTimers()` / same names | Yes (minor default differences) |
| `jest.clearAllMocks / resetAllMocks / restoreAllMocks` | `vi.clearAllMocks / resetAllMocks / restoreAllMocks` | Yes, but `restore` semantics differ slightly |
| Variables referenced inside a `jest.mock` factory | `vi.hoisted(() => ...)` | No |
| `moduleNameMapper: { '^c/(.*)$': '<rootDir>/c/$1' }` | `resolve.alias: [{ find: /^c\/(.*)$/, replacement: '.../c/$1' }]` | Mostly. First match wins, so order matters (D3). |
| `setupFilesAfterEnv` | `test.setupFiles` | Yes |
| `testEnvironment: 'jsdom'` | `test.environment: 'jsdom'` | Yes |
| Globals (`describe`, `it`...) | Import from `'vitest'` or set `globals: true` | Yes |

## Gotchas that bite

<details open>
<summary><b>1. Async <code>importActual</code></b></summary>

`jest.requireActual` is synchronous, while `vi.importActual` returns a Promise. Partial mocks have to become `vi.mock('m', async (orig) => ({ ...(await orig()), foo: vi.fn() }))`. A codemod can do this, but it should flag unusual cases instead of guessing.

</details>

<details>
<summary><b>2. Module identity under isolation (A10)</b></summary>

With `isolate: true`, each test file gets fresh copies of modules. If the wire adapter registered by the test and the one the component imports end up as two different objects, mock data never reaches the component. The test can still pass, just for the wrong reason. A10 exists to prevent this.

</details>

<details>
<summary><b>3. Tests that silently pass</b></summary>

If runtime setup fails, or a promise rejects with nothing awaiting it, a test can go green without checking anything. The epic requires these cases to **fail**. The C8 leak-probe test and the G3 end-to-end gate guard against this.

</details>

<details>
<summary><b>4. Compiler version drift (C1, C2)</b></summary>

The Jest path and the Vitest path must compile LWC with the same compiler version and options (B3). If they don't, migrated tests check slightly different code. C1 bumps `@lwc/rollup-plugin` from 8.18.2 to 9.x, and C2 handles consumers broken by that bump.

</details>

<details>
<summary><b>5. Snapshot churn (E series)</b></summary>

LWC adds generated CSS scope tokens to markup. If the tokens are not stable across runs, every snapshot changes on every run.

</details>

<details>
<summary><b>6. Ordered regex aliases (D3)</b></summary>

Jest's `moduleNameMapper` is an ordered list of regexes with `$1` capture groups, and the first match wins. The D3 "capture-group resolver shim" reproduces that exactly in Vite.

</details>

---

# Part 3: Our work

## Series A to G decoded

Each letter is one layer of the runner. The layers have to be built in roughly this order.

| Series | Layer | What it delivers | Status (GUS, Oct 16) |
|---|---|---|---|
| **A** (A1 to A9) | Import resolution | Scaffolds the plugin and preset npm packages. Resolves every `@salesforce/*` import (labels, resourceUrl, permissions, user, client, i18n, schema, site, apex) to stubs or spies. | Closed (Sept epic). A10 and A11 carried into Nov. |
| **B** (B1 to B7) | Compile and boot | `defineConfig` preset, LWC rollup-plugin wiring, compiler options matching Jest, TypeScript type-stripping, runtime load order, custom matchers, `@wire` interception hook | Ready for Review (Oct epic) |
| **C** (C1 to C8) | Core and Bazel integration | Compiler bump, `validations/vitest-lwc/` engine dir, per-package opt-in in `ui-tests.bzl`, Jest and Vitest coexistence target, CI critical-suite selection, isolation, leak probe | New (Nov epic) |
| **D** (D1 to D3) | Module aliasing | `lightning/*` stubs, static aliases, regex capture-group resolver | New |
| **E** (E0 to E2) | Snapshots | Scope-token registry, serializer, stable CSS tokens | New |
| **F** (F1, F2) | Accessibility | Vendored `@sa11y/vitest` plus a setup file | New |
| **G** (G1 to G3) | Validation gate | Validates the wire-service util and `import.meta.env`. End-to-end framework gate with no codemod yet. | New |

## Glossary

| Term | Meaning |
|---|---|
| ESM / CJS | ES Modules (`import`/`export`) and CommonJS (`require`/`module.exports`) |
| jsdom | A JavaScript implementation of the browser DOM, used to render components without a real browser |
| Preset | A reusable bundle of test config that a package extends |
| Plugin (Vite/Rollup) | A hook into resolve, load and transform, used to compile LWC and to fake `@salesforce/*` modules |
| Synthetic shadow | LWC's polyfill for shadow DOM, which Core still relies on and which has to be initialized before tests run |
| Wire adapter | The LWC `@wire` data provider. Tests emit fake data through a test adapter. |
| sa11y | Salesforce's accessibility test library (axe-core based), which provides `toBeAccessible()` |
| Codemod | A script that rewrites source code automatically (for example `jest.fn` to `vi.fn`) across thousands of files |
| Isolation / pool | Whether each test file gets a fresh module graph, and whether it runs in threads or forked processes |
| Quarantine | The CI mechanism that sidelines known-flaky tests so they don't block merges |

---

# Part 4: Practice

## Hands-on exercises (about 60 minutes total)

1. **Jest hello world (10 min).** `mkdir t1 && cd t1 && npm init -y && npm i -D jest`. Write `sum.test.js` (use `require`) and run `npx jest`.
2. **Vitest hello world (10 min).** `mkdir t2 && cd t2 && npm init -y && npm i -D vitest`. Write the same test with `import` and run `npx vitest`. Edit the source file and watch it rerun right away.
3. **Mock migration (15 min).** In Jest, write a partial mock with `jest.requireActual`. Port it to Vitest and see why the factory has to become `async`.
4. **LWC on Jest (15 min).** `npx degit trailheadapps/lwc-recipes` (or any SFDX project) and run `npm run test:unit`. Read one wire-adapter test.
5. **Read our code (10 min).** Open the B-series PRs (B1 preset, B2 rollup wiring) and map each file to the "5 jobs" table at the top of this page.

## Resources

- Jest docs: [jestjs.io/docs/getting-started](https://jestjs.io/docs/getting-started) (read Getting Started, Mock Functions, Timer Mocks, Snapshot Testing)
- Vitest guide: [vitest.dev/guide](https://vitest.dev/guide/) (read Features, Mocking, Test Environment, Improving Performance)
- **Vitest's official "Migrating from Jest" page:** [vitest.dev/guide/migration](https://vitest.dev/guide/migration.html), the single most useful page for this project
- LWC Jest tooling source: [github.com/salesforce/lwc-test](https://github.com/salesforce/lwc-test) (preset, transformer, resolver, serializer)
- sa11y: [github.com/salesforce/sa11y](https://github.com/salesforce/sa11y) (includes the Jest and Vitest integrations)
- LWC sample tests: [github.com/trailheadapps/lwc-recipes](https://github.com/trailheadapps/lwc-recipes)

---

*Generated 2026-10-16 from GUS epics a3QEE000002gDhF2AU, a3QEE000002iPcf2AE and a3QEE000002aaS12AI. Status reflects GUS on that date.*
