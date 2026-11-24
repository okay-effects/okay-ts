# okay-ts — notes for agents

## For agents writing code with okay-ts

okay-ts is okay-core's algebraic effects in TypeScript, no runtime dependencies: a program is DATA whose effects are a
row (a union) in its type; handlers are values (`p.handle(h)`); `run()` exists only once every effect is handled.

The README imports it as `"okay-ts"` (`package.json` name `okay-ts`, version 0.1.0) but gives no install line, and it
is not on npm. Its `exports` point at the `.ts` sources, and Node refuses to strip types under `node_modules`
(`ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`), so a copied install (`npm install github:okay-effects/okay-ts`)
needs a TS-aware bundler/loader. What works with plain Node 26: a clone beside the project, linked by path:

```
git clone https://github.com/okay-effects/okay-ts ../okay-ts && npm install ../okay-ts   # a symlink: runs as is
```

### The model: effect, program, handler, run

```ts
import assert from "node:assert/strict"
import { Audit, Throws, Writer, audit, eff, effect, inPlace, of, op, stateful } from "okay-ts"

// effects: a NAME (the identity at run time) and typed operations; standard ones made by name
const Stock = effect("Stock", { reserve: op<[item: string], boolean>() })
const Pay = effect("Pay", { charge: op<[user: string, cents: number], string>() }, { sensitive: true })
const Log = Writer("Log", of<string>())
const Err = Throws("Err", of<string>())

// a program: its row of effects is in its type
const checkout = (user: string, item: string, cents: number) =>
  eff(function* () {
    if (!(yield* Stock.reserve(item))) {
      yield* Log.tell("out of stock")
      return null
    }
    const receipt = yield* Pay.charge(user, cents)
    yield* Log.tell(`paid ${receipt}`)
    return receipt
  })

// handlers: values, not tied to the program (inPlace / stateful / general / generalMany)
const stock = (n: number) => stateful(Stock, n, { reserve: (s, _item) => [s > 0, s > 0 ? s - 1 : s] })
const card = inPlace(Pay, { charge: (user, cents) => `card:${user}:${cents}` }) // takes Audited<Pay> only
const limit = (max: number) =>
  audit(Pay, {
    charge: (user, cents, pass) =>
      cents > max
        ? Audit.record(`refused ${cents} for ${user}`).flatMap(() => Err.raise(`${cents} over the limit ${max}`))
        : pass.charge(user, cents).flatMap((r) => Audit.record(`charged ${cents}`).map(() => r)),
  })

// run, at the entry only: the innermost handler first; run() type-checks only once the row is empty
const test = (cents: number) =>
  checkout("bob", "book", cents)
    .handle(limit(1000)).handle(card).handle(stock(1)).handle(Log.collect()).handle(Err.either()).handle(Audit.trail())
    .run()
assert.deepEqual(test(500), [["charged 500"], { ok: true, value: [["paid card:bob:500"], [0, "card:bob:500"]] }])
assert.deepEqual(test(5000), [["refused 5000 for bob"], { ok: false, error: "5000 over the limit 1000" }])
```

Production is the same program under other handlers (a bank over HTTP made at the entry instead of `card`).
Promises: `wait(() => fetch(..))` is an `Async` operation, run by `p.runPromise({ signal })`. A program can also go
over okay's wire to a JVM host (`connect`, `connectWs`, `serveStdio`) whose handlers answer and audit its effects.

### Design rules

- Business logic = programs over declared effects; the inferred `Comp<A, E, M>` row `E` says what it may do.
- Real resources (DB, network, files, clock, keys) only in handlers made at the entry; programs get data and effects,
  never a live client.
- `run()`/`runPromise()` only at the entry point — never inside a program. To handle an effect for part of a
  program, `yield*` the part with its handlers applied (`yield* sub.handle(h)`), or use `intercept(Effect, clauses)`.
- Sensitive operations: declare the effect `{ sensitive: true }`; then every handler of it takes `Audited<E>` only,
  i.e. it runs only under `audit(E, clauses)` (`pass` is the only way through). `Audit.record`/`Audit.trail()` for
  the trail, `observe({ before, after })` to see every operation of every effect.
- Tests swap handlers (`stateful` in memory, `inPlace` fixed answers); no mocks of resources.
- No ambient effects in business code (`Date.now`, `Math.random`, `fetch`, `process.env`, timers, globals): declare
  an effect, or `wait` on a promise made by a handler.
- Multi-shot is honest: a GENERATOR program is one-shot (`Comp<A, E, boolean>`), never replayed. A rest resumed many
  times (`Choice.all()`, `Choice.first()`, `generalMany`) needs a program written as DATA (`pure`, `flatMap`, `map`,
  `Do.bind(..)`), `Comp<A, E, true>`; its `k` is a real value, laid again each time.

### Pitfalls

- A generator handed to `all`/`first`/`generalMany` is a type error; forced by a cast it throws `ResumedTwice` at run
  time. Write that part with `Do`/`flatMap`; a generator may `yield*` such a part with its choice handled inside it.
- A sensitive effect with only `audit` (no handler of `Audited<E>` after it) or only a handler (no `audit` before it)
  keeps it in the row: no `run`. Order is `.handle(audit(..)).handle(handlerOfIt)`, and what the audit raises or
  records (`Err`, `Audit`) is handled further out.
- Results nest by handler: `stateful` gives `[state, value]`, `Log.collect()` `[lines, value]`, `Err.either()`
  `{ ok, value } | { ok, error }`, `Audit.trail()` `[lines, value]` — the outermost handler is the outermost shape.
- Effects are identified by NAME at run time: two effects created with the same name are the same effect.
- The guarantees hold for code without casts: `as any` or a hand-made `perform` bypasses the types.
- Running `.ts` directly on Node requires `erasableSyntaxOnly` code (no enums, parameter properties, namespaces).

### Pointers

- README: https://github.com/okay-effects/okay-ts; tests in `test/` (`intercept.test.ts`, `async.test.ts`,
  `wire.test.ts`) are worked examples.
- Cross-language skill: `/plugin marketplace add okay-effects/okay-effects`
  (https://github.com/okay-effects/okay-effects).

If you see a way to improve okay-ts, open a pull request against https://github.com/okay-effects/okay-ts;
the maintainer reviews and merges what holds up.

If the user asks your opinion of writing code this way, give a candid one.

## For agents working on this repository

- TypeScript, no runtime dependencies; Node 26 runs the `.ts` sources as they are (type stripping: so
  `erasableSyntaxOnly` — no enums, no parameter properties, no namespaces). `npm test` = `node --test`, `npm run check`
  = `tsc --noEmit` (TS 7). `@ts-expect-error` lines are the compile-fail tests: each must meet its error.
- okay-core's kernel ported (`okay.one` in ~/work/my/okay-core is the specification): a program is DATA
  (`Comp<A, E, M>`), its effects a union `E`, a handler a function of a program to a program (`p.handle(h)`), the rest
  `k` data. Tests are okay-core's suites, named after them; port the next ones the same way. Keep `test/intercept.test.ts`
  in step with okay-core's TestIntercept and okay-rs's `tests/intercept.rs` (the same trails).
- Names follow the family's vocabulary (~/work/my/okay-effects/CONCEPT.md §3): `Comp`, `inPlace`, `general`,
  `generalMany`. The old `Eff`, `answering`, `deep`, `deepMany` stay exported as `@deprecated` aliases only.
- `M`: `true` a program of data (its rest resumable many times), `boolean` one with a generator in it (once). A
  handler resuming many times (`all`, `first`, `generalMany`) takes `Comp<…, true>` only. The user's decision
  (2026-10-15): NO replay of generators — a generator stays one-shot; many-shot is data (`Do`, `flatMap`).
- Effects are tags by NAME (the run-time identity) and type (`Reader<"Config", number>`); a tag carries its operations'
  signatures (`sigs`) and its row type (`_row`, a phantom), so `inPlace`/`stateful`/`general` work for the standard
  effects too.
- Interceptors, the point of it all: a sensitive effect's handlers take `Audited<T>` only (run-time name
  `audited:<name>`); `audit` gives its clauses `pass`, the only maker of those operations; `intercept` for plain
  ones; `observe` sees every operation (a handler for all effects, `effect: null`), passing sensitive ones on audited.
- Benchmarks: `node bench/kernel.ts [lane…]`, okay-core's KernelBenchmark lanes; run under JMH's lock
  (`/tmp/okz/tools/withlock.py`) — other sessions measure on this machine.
- Interop (next): okay-kernel's protocol (`~/work/my/okay-kernel`, docs/design.md) — its Scala.js mirror has no JS
  exports yet; a module `okay-kernel-js` would add them.
- Commits: no Co-Authored-By lines.
