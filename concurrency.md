# Phaenon — Concurrency design (draft 0.1)

Status: **withdrawn 2026-11-08** (roadmap D25, §17): the syntax is not in the language until PHP
core has a scheduler, and this document is the specification it will be mapped onto then.
Reviewed 2026-11-01 (decisions §15) · M5. Companion: `language-overview.md` §4–5,
`roadmap.md` D6 and §3 (M5), `functions.md` (closures and their lowering), `collections.md`
(method syntax, the stdlib written in Phaenon).

This document is the design. §14 lists the questions the review settled, §15 the decisions. Where it departs from what the roadmap assumed
(D6's "stdlib loop", `tequila/nats` as the first adopter) it says so and why.

---

## 0. One paragraph

Phaenon gets structured concurrency with two pieces of syntax: a `concurrently { … }` block
that owns every task started inside it, and `spawn expr`, which starts one. A task is a PHP
Fiber scheduled by the Revolt event loop, and its handle is amp's `Future<T>`, so what Phaenon
adds is the **structure**, not another asynchronous world: a block cannot be left while one of
its tasks runs, the first failure cancels its siblings and is rethrown where the block ends, and
cancellation and deadlines flow down the scope. No function is marked `async`: any function can
be called from a task, and a function that uses a block inside is called like any other, from a
plain request handler as well as from inside another block. The compiled PHP is amp v3 code a
person would write — `Scope::run(function () use (…) { … Scope::spawn(fn () => …) … })` — so it
composes with amp's libraries, runs in a Symfony console process and under FrankenPHP, and
passes PHPStan at level max.

## 1. The facts that shape it

What the dogfood codebase does today, read on 2026-11-01:

- **`tq-cli` is the platform's one concurrent program.** amp v3.1 over Revolt 1.0 (http-client,
  http-server, websocket-client, `Amp\delay`), entered from Symfony console commands with no
  `EventLoop::run()` anywhere — amp runs the loop inside `await()` when the main fiber waits.
  Its fan-out is `Api::concurrently(array $jobs)`, twelve call sites, implemented as
  `await(array_map(async(...), $jobs))`: a job that throws fails the batch while the other jobs
  keep running, finish unobserved, and may still be writing when the caller has moved on.
  `ManifestFlow` combines a `DeferredFuture`, a local HTTP server and a `TimeoutCancellation`
  by hand. `TestInfra` polls with `usleep()`, which would freeze the loop if it ever ran inside a
  fan-out.
- **`tequila/nats` is synchronous by decision.** Its architecture decision record (2026-08)
  chose a sans-I/O core, non-blocking sockets with `stream_select` against a deadline, and an
  event-pumped `Connection` whose every call "returns or throws" — because Messenger, the
  change-stream streamers and a FrankenPHP request handler want exactly that, and because
  Fibers/Revolt would have brought "a runtime nobody else in core uses". The `Transport`
  interface is the seam it left for "a Fiber/Revolt-backed transport, a drop-in later".
- **`core` and the services built on it** run on FrankenPHP worker mode and in Messenger workers;
  no Fibers, no amp.
- **PHP 8.5 has Fibers and nothing above them**: no scheduler, no awaitable, no `async` in the
  language. Revolt is the event loop amp v3 and (through a bridge) ReactPHP share. Two loops in
  one process cannot share a socket wait: a Phaenon-owned loop would be a fork of Revolt that
  excluded amp's libraries, which is to say excluded the only non-blocking I/O PHP has.

Consequences for the design: **the loop is Revolt, the awaitable is amp's `Future`, the
runtime is amp v3**, and Phaenon's contribution is syntax, structure and checking on top. Two
roadmap assumptions change: D6 said "purely syntax + stdlib loop" — the loop is not Phaenon's;
and M5 named `tequila/nats` as "the natural first adopter" — the natural first adopter is
`tq-cli`, which already runs on amp and has the fan-outs, while `nats` contributes the Revolt
transport (§9.3) rather than a rewrite.

## 2. Goals and non-goals

### Goals

1. **Structured.** A task lives no longer than the block that started it. Leaving the block by
   `return`, by its last statement or by an exception waits for every task. A failure in one
   task cancels the others and surfaces at the block; nothing fails silently in the background.
2. **No function colouring.** There is no `async fun`, no `await`-only-inside-`async`. A method
   that fans out inside is a method; its caller cannot tell.
3. **Output a person would write** (D9, D11): amp v3 calls, no synthesised variables, every
   identifier the programmer's; PHPStan level max on the output; line N of the `.php` is line N
   of the `.phn`.
4. **Checked.** The mistakes structure exists to prevent are compile errors: a task started
   outside a block, a handle smuggled out of its block, a loop-freezing sleep inside one.
5. **Synchronous entry points stay synchronous.** A Symfony command, a FrankenPHP request, a
   Messenger handler use a block without starting a loop; from their point of view the block
   is a call that returns or throws, like today's `Connection::request()`.

### Non-goals (v1)

- Channels, pipelines, queues: amp's `Pipeline` and `Amp\Sync` are usable as libraries; a
  Phaenon spelling may come once the block is in use.
- Parallelism: Fibers are one thread. CPU-bound work stays in processes.
- Making blocking PHP non-blocking: PDO, the mongodb extension, `file_get_contents` block the
  loop inside a task as they block anything else; §9.4 says what the design does about it.
- A Phaenon event loop, actors, supervision trees, async generators.

## 3. The model

- A **task** is a Fiber the Revolt loop schedules. `spawn` starts one; it runs until it
  completes or until it waits — on I/O through an amp library, on another task's result, on
  `Scope::sleep()`. Waiting suspends the Fiber and lets the loop run the others. Scheduling is
  cooperative, as in every Fiber runtime.
- A **scope** is what a `concurrently` block opens. It owns the tasks spawned lexically inside
  it, carries a `Cancellation` (its own, composed with the enclosing scope's and with its
  deadline), and ends only when every task has completed.
- A **handle** is `Amp\Future<T>`, the value of `spawn`. `->await(): T` gives the result or
  throws the task's exception. `map`, `catch`, `finally`, `ignore` are amp's and work.
- The **body** of a block runs in the fiber that reached it — the main fiber included. No new
  fiber is created for the body, so `$this`, locals and the call stack are the caller's; the
  first wait inside is what starts the loop turning. That is how a synchronous program enters
  concurrency without an `EventLoop::run()`.
- **Nesting** is dynamic as well as lexical: a task may open its own block (a child scope); a
  method called from a task may open one. A child scope's failure is the task's exception, and
  propagates through the parent scope like any other.

## 4. Spelling

### 4.1 `concurrently { … }`

```php
fun profile(UserId $id): Profile
{
    return concurrently {
        $user = spawn $this->users->find($id);           // Future<User>
        $orders = spawn $this->orders->forUser($id);     // Future<list<Order>>

        return new Profile($user->await(), $orders->await());
    };
}
```

```php
concurrently {
    spawn $this->mailer->send($welcome);
    spawn $this->audit->record($signup);
}
// both have completed here — or the block threw
```

```php
$exists = concurrently (timeout: 30.0) {
    return Scope::all($probes);
};

concurrently (cancellation: $shutdown) {       // a Cancellation the caller holds
    …
}
```

- The block is an **expression** when it is used as one and a **statement** otherwise. Its
  value is what `return` inside it returns; a block without `return` is `void`. `return`
  inside the block leaves the block, never the enclosing function — the block is a closure in
  the output (§6), and the spelling says so.
- The optional parenthesised list after the keyword is a PHP argument list: `timeout: float`
  seconds, `cancellation: Cancellation`. It compiles to the named arguments of `Scope::run`.
- When the block ends — by `return`, by its last statement, or by an exception — every task
  spawned in it has completed. Tasks the body never awaited are awaited there; the first
  failure among them is thrown (§5).
- `break` and `continue` cannot cross the block, `yield` cannot appear in it, `goto` cannot
  enter or leave it (checker, §7). Inside, everything else is ordinary code: loops, `try`,
  nested blocks, method calls that open blocks of their own.

### 4.2 `spawn`

```php
$user   = spawn $this->users->find($id);       // Future<User>
$sent   = spawn $this->mailer->send($mail);    // Future<void>
$report = spawn {                              // several statements: a block task
    $rows = $this->load($id);
    return Report::of($rows);
};
```

- `spawn expr` starts `expr` as a task of the innermost enclosing `concurrently` block and
  evaluates to its handle, `Future<T>` for an `expr` of type `T` (`Future<void>` for a `void` call).
  `spawn { … }` is the same with a statement body; its `return` is the task's result.
- `expr` is captured the way an arrow function captures: by value, with `$this` available.
  Variables it reads are read at spawn time; assignments inside a `spawn { }` body do not reach
  the enclosing scope (the checker reports a write to an outer variable, §7).
- `spawn` is legal only **lexically inside** a `concurrently` block — directly, or inside an
  arrow function or closure written there (`$ids->map(fn ($id) => spawn $this->load($id))`
  is the intended fan-out spelling). The runtime also checks: a closure that carried a `spawn`
  out of its block and runs later throws a `LogicException`.
- A task spawned from inside another task's body expression belongs to the same block (the
  block's scope is ambient in the task's fiber), so lexical nesting and dynamic nesting agree.

### 4.3 Fan-out helpers

Two functions on the runtime class cover the homogeneous case without handles:

```php
// map<string, array<string, int>>: keys preserved, failures as in a block
$indexes = Scope::map($repos, fn (string $repo) => $this->rulesets->index($this->token, "repos/{$this->org}/$repo", 'Repository'));

// inside a block, when the tasks are already spawned
$results = Scope::all($tasks);                 // iterable<K, Future<T>> → array<K, T>
```

`Scope::map(iterable<K, T>, Closure(T): R): array<K, R>` opens a scope of its own, spawns
one task per element, joins and returns the results under the same keys — exactly what
`tq-cli`'s `Api::concurrently` does, with structure. `Scope::all` is amp's keyed
`Future\await` under the ambient cancellation. Both are written in Phaenon in the runtime
(`stdlib/runtime/Scope.phn`), with `concurrently` inside, as `Seq` and `Map` are.

`Scope::first(iterable<Future<T>>): T` (the first to complete, the rest cancelled) is listed for
completeness; whether v1 ships it is an open question (§14).

### 4.4 Cancellation and time

- `Scope::cancellation(): Cancellation` is the current task's scope cancellation (the
  innermost block's, composed with its parents' and its deadline). Code hands it to amp APIs
  that accept one: `$client->request($request, Scope::cancellation())`.
- `Scope::sleep(float $seconds): void` is `Amp\delay()` under the ambient cancellation: it
  wakes early, with `CancelledException`, when the scope is cancelled. `sleep()`, `usleep()`,
  `time_nanosleep()`, `time_sleep_until()` inside a block are compile errors (§7): they freeze
  the loop and every task with it.
- `Scope::checkpoint(): void` throws `CancelledException` when the current scope is cancelled —
  for a long CPU loop that has no wait of its own; `Scope::cancelled(): bool` asks without
  throwing.
- A deadline is the block's `timeout:`; on expiry the scope is cancelled and the block throws
  what amp's `TimeoutCancellation` reports: a `CancelledException` whose previous exception is
  the `TimeoutException`. No Phaenon exception types: both are amp's, and a `catch` written for
  amp code keeps working.

## 5. Semantics

| At the block's end | What happens |
|---|---|
| The body returned (or ran out) | The scope awaits every task not yet awaited. If all succeeded, the block yields the body's value |
| A task failed, before or after the body finished | The scope is **failed**: every other task is cancelled, the body is cancelled at its next scope-aware wait (`->await()` of a sibling, `Scope::sleep`, `Scope::all`, an amp call given `Scope::cancellation()`), the scope waits until every task has finished, then throws the **first** failure. A later failure that is not a `CancelledException` is chained as `previous`; the siblings' `CancelledException`s are dropped |
| The body threw | Every task is cancelled and awaited; the body's exception is rethrown, task failures chained as above |
| The deadline passed, or the given `cancellation` fired | As a failure, with amp's `CancelledException` (its previous exception the `TimeoutException` for a deadline) |

Rules that follow:

- **A task's failure fails its block, whether or not the body awaited the handle.** `->await()`
  on a failed task throws the task's exception too (amp's semantics) and the body may catch it,
  but the scope still fails when it ends. When a failure is an answer — an existence probe that
  gets a 404 — the task catches it itself and returns the answer: `spawn $this->exists($repo)`
  with the `try` inside `exists()`. This is the Trio and Kotlin rule; Java's plain
  `StructuredTaskScope` (failures only where observed) was considered and rejected as the rule
  under which a forgotten `await` loses an exception. A supervising variant (`supervise: true`)
  is not in v1.
- **Cancellation is cooperative**, and the design says so rather than pretending otherwise.
  Revolt keeps one `Suspension` per fiber, which a scope could throw into from outside; but
  amp's `Future::await()` keeps its completion callback subscribed after such a throw and would
  resume the fiber a second time, at whatever it is waiting on by then. So a cancelled task is
  interrupted only where it waits through something that honours its cancellation: Phaenon's
  own waits (`->await()` of a handle is amp's and takes a `Cancellation`; `Scope::sleep`,
  `Scope::all`, `Scope::map`, `Scope::checkpoint` take the ambient one), and amp APIs the task
  passed `Scope::cancellation()` to. A task in the middle of an amp HTTP request it did not hand
  a cancellation to runs to the end of that request before it notices. The block waits for it:
  structure is kept, latency is the price of the omission, and the omission is visible in the
  code (no cancellation argument).
- **Order.** Tasks start when the body next waits, in spawn order (amp's `async` queues the
  fiber; the body keeps running until its first wait). The body is a participant, not a
  supervisor: it may do work of its own between spawns.
- **Re-entrancy.** A block inside a task is a child scope with its own cancellation, composed
  with the parent's: cancelling the parent cancels the child. A block inside a method called
  from the main fiber in a request handler is a root scope.
- **After the block**, the handles are complete. Reading one is harmless (the value, or the
  exception already thrown by the block). The checker still refuses to let them out (§7):
  a handle that outlives its block is the shape of the bug structure exists to prevent.

## 6. Compilation

The emitter lowers both constructs to calls on `\Phaenon\Concurrent\Scope`, imported with
`use` per D11:

| Source | Output |
|---|---|
| `concurrently {` … `}` (statement) | `Scope::run(function () use (…) {` … `});` |
| `$x = concurrently {` … `};` (expression) | `$x = Scope::run(function () use (…) {` … `});` |
| `concurrently (timeout: 5.0) {` … `}` | `Scope::run(function () use (…) {` … `}, timeout: 5.0);` |
| `spawn expr` | `Scope::spawn(fn () => expr)` |
| `spawn {` … `}` | `Scope::spawn(function () use (…) {` … `})` |

- **The `use` clause is computed**: every variable of the enclosing function the body reads is
  captured by value; a variable the body assigns and the enclosing function reads afterwards is
  captured by reference (`&$total`). That is the clause a person writes for the same closure;
  the checker's flow analysis (M2) already knows both sets. `$this` binds as in any closure.
- **Line preservation holds** (`emit` invariant): the opening keyword and brace become
  `Scope::run(function () use (…) {` on the same line, the closing brace `});` — with the
  options after it when there are any. Nothing inside moves; `spawn expr` is an inline
  rewrite.
- **Types travel through PHPStan's generics**, not through docblocks: `Scope::spawn` is
  `@template T @param Closure(): T @return Future<T>`, `Scope::run` is
  `@template R @param Closure(): R @return R`, so `$user` is `Future<User>` and the block's
  value is `Profile` with nothing written in the output. The emitter writes no return type on
  the closures, as it writes none on constant-expression closures (`functions.md` §2).
- Every identifier in the output is the programmer's (D9): `Scope` is a runtime class, the
  closures are the lowering of blocks the programmer wrote, the `use` lists name the
  programmer's variables.

The example of §4.1 compiles to:

```php
public function profile(UserId $id): Profile
{
    return Scope::run(function () use ($id) {
        $user = Scope::spawn(fn () => $this->users->find($id));
        $orders = Scope::spawn(fn () => $this->orders->forUser($id));

        return new Profile($user->await(), $orders->await());
    });
}
```

## 7. Checker

Proposed codes, in the `P2xxx` family (types and flow) as a new hundred:

| Code | Error | Where |
|---|---|---|
| `P2601` | `spawn` outside a `concurrently` block | lexical: not inside a block (nested closures count as inside) |
| `P2602` | a task handle leaves its block | the block's value type contains `Future` that a `spawn` in the block produced; a `spawn` result assigned to a by-reference captured variable, a property, or a static |
| `P2603` | control flow crosses a `concurrently` or `spawn` block | `break`/`continue` to a loop outside, `yield`, `goto` in or out; a `return` with a value in a statement block, whose value nobody reads (§14 asks whether this last case stays an error) |
| `P2604` | a `spawn` body mutates a variable of the enclosing scope | `spawn { $count++; }`, `$total += …`, `$rows[] = …`, `$x = $x . 'y'`: the body is a by-value closure in another fiber and changes a copy; return the value instead. A plain `$x = …` to a name that exists outside is a local of the task, as in any closure |
| `P2605` | blocking sleep inside a block | `sleep`, `usleep`, `time_nanosleep`, `time_sleep_until` in a block or spawn body, nested closures included; `Scope::sleep` instead |
| `P2606` | a bad block option | an unnamed option, an unknown one, `timeout:` that is not `int\|float`, `cancellation:` that is not an `Amp\Cancellation` |

`P2603`'s `return`-with-a-value case: a statement block whose body says `return $x;` computes a
value nobody reads, and is most likely a programmer who meant to return from the method (C7).
A `return;` without a value leaves the block early and is fine. The compiler knows the amp API
through its own embedded declarations (`stdlib/concurrent/amp/`), so a project that forgot to
require `phaenon/concurrent` fails at runtime, not at `phn check`; a check against the
project's `composer.json` is an open item.

Type rules: `spawn e : Future<T>` for `e : T`, `Future<void>` for a `void` call (PHPStan infers the same from the lowered arrow function); `concurrently { … } :
R` for the join of the block's `return` types, `void` without one; `timeout:` is `float|int`,
`cancellation:` is `Amp\Cancellation`. The checker types the lowered closure's `use` list for
its own flow analysis, so a variable assigned only inside a block is, after the block, what the
block made it.

## 8. The runtime: `phaenon/concurrent`

A second Composer package beside `phaenon/runtime`, namespace `Phaenon\Concurrent`, requiring
`amphp/amp ^3` (which brings `revolt/event-loop`). It is separate so that `phaenon/runtime`
stays dependency-free: `core` uses `Seq::map` on every page and has no reason to install an
event loop. A project that writes `concurrently` requires `phaenon/concurrent`; `phn check`
says so when it is missing (§7).

```php
final class Scope
{
    /** @template R  @param Closure(): R $body  @return R */
    static fun run(Closure $body, float|int|null $timeout = null, ?Cancellation $cancellation = null): mixed;
    /** @template T  @param Closure(): T $task  @return Future<T> */
    static fun spawn(Closure $task): Future;
    /** @template K of array-key  @template T  @param iterable<K, Future<T>> $tasks  @return array<K, T> */
    static fun all(iterable $tasks): array;
    /** @template K of array-key  @template T  @template R  @param iterable<K, T> $items  @param Closure(T): R $task  @return array<K, R> */
    static fun map(iterable $items, Closure $task): array;
    static fun cancellation(): Cancellation;
    static fun sleep(float $seconds): void;
    static fun checkpoint(): void;
    static fun cancelled(): bool;
}
```

Implementation notes (for the builder, not the reviewer): a per-fiber stack of open scopes
(`WeakMap<Fiber, list<Scope>>`, the main fiber keyed separately as Revolt keys it); `spawn`
wraps amp's `async()` so that the child fiber starts with the parent scope on its stack and the
scope records the handle; each scope owns a `DeferredCancellation` composed
(`CompositeCancellation`) with its parent's and, with a `timeout:`, a `TimeoutCancellation`;
joining iterates the handles in completion order (`Future::iterate`) so that a failure is seen
as early as it happens, cancels the rest and drains them; the scope keeps every handle it ever
spawned, so that an abort during a join still waits for the tasks of that round. The class is
written in Phaenon (`stdlib/concurrent/Scope.phn`, compiled into the package like `Seq`), which
makes it the first dogfood of its own syntax; its semantics are tested against the real amp and
Revolt in the package's PHPUnit suite.

## 9. Interop

### 9.1 amp

Handles **are** amp Futures, cancellation **is** amp's: Phaenon code and amp code call each
other without adapters. A block inside an amp request handler is a scope like any other; a
`Future` from `Amp\async()` can be awaited inside a block (it is not owned by the scope, and the
checker does not pretend otherwise: `P2602` is about `spawn` results).

### 9.2 Symfony, FrankenPHP, Messenger

Entry is synchronous (§3): a console command or a request handler calls a method that fans out
inside and gets a value or an exception. Revolt's driver is a process-global that survives
across FrankenPHP worker-mode requests, as the NATS connection does. **Verified 2026-11-01**
(C9): a worker script (`frankenphp_handle_request` in a loop, two worker threads, FrankenPHP
1.2.2 / PHP 8.3.10, amp 3.1.3) whose handler ran four tasks per request — three `delay()`s of
5–15 ms and a socket connect to the server itself — joined with `Future\await`, under 400
requests at concurrency 8: 400 correct answers, a median of 16.6 ms per request (the longest
delay, so the tasks ran concurrently), the loop's registered-callback count flat at 1, a
loop-level `EventLoop::repeat` timer ticking on across requests (1 → 66), memory flat, the
handler in the main fiber. The driver persists between requests and the next request's first
wait resumes it. Messenger handlers are plain calls.

### 9.3 `tequila/nats`

The ADR's drop-in: a `RevoltTransport` on `Amp\Socket` — `read($timeout)` is a socket read under
a `TimeoutCancellation`, `write` likewise, `connect`/`upgradeTls` amp's connectors. `Connection`
is unchanged and synchronous callers see no difference, since an `await` from the main fiber
runs the loop; inside a block, a NATS request suspends instead of freezing every other task
behind `stream_select`. One real problem comes with it: **two tasks pumping one connection**.
`pumpUntil` was written for a single caller; two fibers in it would both read the socket, and
amp's socket refuses a second concurrent read. The transport needs a single reader — a pump task
owned by the connection, or a mutex around the pump — which is a change to `Connection`, not to
the transport. That is the real engineering of the nats part, and §14 asks whether it is M5's.

### 9.4 Blocking PHP

A task that calls PDO, the mongodb extension, `file_get_contents`, or a `stream_select` client
such as nats' default transport blocks the loop for the duration of that call: the program stays
correct and ceases to be concurrent for that moment. No colouring means the checker cannot see
it, and the design accepts that; `P2605` catches the sleeps, which are the one blocking call
with no reason to exist inside a block. The documentation says plainly which Tequila libraries
suspend (amp http, nats on the Revolt transport) and which block (mongodb-odm, PDO).

## 10. Migration

`phn migrate` rewrites nothing automatically: there is no PHP spelling to lift. The guide names
the hand rewrites — `Api::concurrently($jobs)` becomes `Scope::map` or a block,
`await(array_map(async(...), …))` likewise, a `usleep` poll becomes `Scope::sleep` — and the
dogfood (§12) does them.

## 11. Editor

PhpStorm's parser knows neither keyword, and the plugin's lexer-level technique (`fun` →
`function`) cannot add the closing `})`. A workable approximation for M5: the lexer turns
`concurrently` and its option list into a comment token, so the braces parse as a bare statement
group, and `spawn` into a comment, so `$user = $this->users->find($id)` is typed `User` — wrong
by a `Future<>`, navigable and completable. The expression form (`return concurrently {`) stays
a syntax error in the editor until a PSI-level block exists (M6, with the LSP). The plugin's
limitations list gets the entry.

## 12. Dogfood

- **`tq-cli`** (585 files, PHPStan level 10, amp already): migrate to `.phn` on a local
  `phaenon-dogfood` branch, replace the twelve `Api::concurrently` sites with `Scope::map` or
  blocks, `TestInfra`'s `usleep` polls with `Scope::sleep`, `ManifestFlow` with a block holding
  the server task and the callback wait under `timeout:`; its suite, PHPStan and cs must pass on
  the compiled output. This is where the design meets real fan-outs and real amp.
- **`tequila/nats`**: the `RevoltTransport` and an integration test that drives one connection
  from a block (two concurrent requests), which forces the single-reader change of §9.3. Whether
  that lands in M5 or waits for a Tequila-side decision is §14's question.
- The FrankenPHP check of §9.2: a throwaway worker-mode service that fans out per request, run
  for a few hundred requests.

## 13. Alternatives considered

| Alternative | Why not |
|---|---|
| A Phaenon event loop (D6's "stdlib loop") | A second loop in a process cannot share socket waits with amp's; Phaenon would own a fork of Revolt and exclude amp's libraries, the only non-blocking I/O PHP has |
| A Phaenon `Task<T>` wrapper around `Future<T>` | A type in between would need unwrapping at every amp boundary and buy nothing: the structure lives in the scope, not in the handle |
| `await` as a keyword | `->await()` is a method on a value the programmer holds; a keyword would read as colouring and add nothing the method lacks. `Scope::all` covers the list case |
| A named scope: `concurrently ($tasks) { $tasks->spawn(…) }` | Explicit, and what Java and Swift do — but the name carries no information, every line pays it, and `spawn` in a nested arrow function would need the name threaded through. The ambient scope is lexically unambiguous (`P2601`) and runtime-checked for the escaped-closure case |
| Lowering to `try { … } finally { $scope->close(); }` with no closure | Needs a variable the programmer did not write (D9) and cannot tell, in `finally`, whether the body threw — which decides between awaiting the tasks and cancelling them. The closure lowering is what `Scope::run` needs to know both |
| Forced cancellation (throw into the task's suspension) | Unsafe with amp's `Future::await`, which keeps its completion callback and would resume the fiber a second time (§5). Cooperative cancellation is what amp's own libraries implement |
| Failure only where observed (Java's plain `StructuredTaskScope`) | A forgotten `await` would lose an exception; the Trio/Kotlin rule keeps "a block throws iff something inside it threw and was not caught where it was thrown" |
| `tequila/nats` rewritten on Fibers as the first target | Its ADR argues the opposite for good reasons, and it has no fan-out to express; the transport is its part |

## 14. Open questions for the review

1. **The runtime is amp v3 / Revolt**, not a Phaenon loop — revising D6's wording. Agreed?
2. **Spelling**: `concurrently { }` and `spawn` (alternatives: `go`, `launch`, `start`);
   `->await()` as a method, no `await` keyword; options as `concurrently (timeout: 5.0) { }`.
3. **The failure rule** (§5): a task's failure fails the block even when the body awaited and
   caught it. The alternative is "fails unless observed".
4. **A second package `phaenon/concurrent`** with the amp requirement, keeping
   `phaenon/runtime` dependency-free — or one package that pulls amp into every Phaenon project.
5. **Expression blocks**: `return concurrently { … };` is new to PHP's shape. The alternative is
   statement-only blocks with by-reference captures of the variables assigned inside, which
   keeps `return` meaning what it means in PHP — at the price of `use (&$user, &$orders)` in the
   output and the loss of the block's result type.
6. **`Scope::first`** in v1, or only `all`/`map`.
7. **`P2603` for a `return` with a value in a statement block**: an error, or dropped.
8. **Dogfood target**: `tq-cli` first (with the twelve fan-outs), `nats` limited to the Revolt
   transport and the single-reader change — or the roadmap's order.
9. **The FrankenPHP verification** (§9.2) before or after the compiler work.
10. **Diagnostic numbering**: a new hundred `P26xx` in the `P2xxx` family, or a fifth family.

## 15. Decisions

Taken at the review of 2026-11-01, which closed every question of §14. `roadmap.md` §1 has D6
revised accordingly and D16 for the block. Where the draft above and this table differ, the
table wins.

| # | Decision | Why |
|---|---|---|
| C1 | The runtime is amp v3 over Revolt; Phaenon ships no event loop. Revises D6's "stdlib loop" (2026-11-01) | §1, §13: a second loop cannot share socket waits with amp's libraries |
| C2 | The handle is `Amp\Future<T>` and waiting is its method, `->await()`; no `await` keyword (2026-11-01) | A method on a value the programmer holds reads as PHP and shows its type in the editor; a keyword would read as colouring |
| C3 | Blocks are expressions: `return concurrently { … };`, in the shape of `return match (…) { … };` (2026-11-01) | §14 q5: the result comes out as a value with its own type, no by-reference captures in the output |
| C4 | A second package `phaenon/concurrent` requires `amphp/amp`; `phaenon/runtime` stays dependency-free (2026-11-01) | §8: `core` uses `Seq::map` on every page and has no reason to install an event loop |
| C5 | A task's failure fails its block, whether or not the body awaited the handle and caught the exception; a failure that is an answer is caught inside the task (2026-11-01) | Exceptions are never lost (Trio, Kotlin, and Java's finalized `StructuredTaskScope.open()` with `awaitAllSuccessfulOrThrow`); "fails unless observed" would need a Phaenon handle type, since `Amp\Future` is final and `await()` gives the scope no hook. A `Scope::settle()` that returns results and failures side by side can be added as a library helper when a use site asks |
| C6 | No `Scope::first` in v1; amp's `awaitFirst` and `awaitAny` stay callable inside a block (2026-11-01) | Two meanings (first to complete, first to succeed), losers that run until their next cancellation-aware wait, and no use site in Tequila |
| C7 | `return;` leaves a block early; `return expr;` in a block whose value nobody reads is `P2603` (2026-11-01) | The one place closure semantics can surprise: a programmer who meant to return from the method |
| C8 | Dogfood order: `tq-cli` first (the twelve fan-outs, suite and PHPStan on the output), then `tequila/nats` with the `RevoltTransport` and the single-reader change to `Connection`, inside M5 (2026-11-01) | tq-cli is where the design meets real amp code; the nats part is a Tequila design change that tq-cli's result may inform |
| C9 | The FrankenPHP worker-mode check of §9.2 comes first, time-boxed (2026-11-01) | If Revolt's loop misbehaves between requests, §9.2 and the runtime change; better known before the emitter exists |
| C10 | Diagnostics are `P26xx`, a new hundred in the types-and-flow family (2026-11-01) | Every rule is a flow or structure rule; a fifth family for six codes is noise |

## 16. Decisions made while building M5

Taken between the review and the close of M5 (2026-11-01). Where the text above differs, this
table wins.

| Topic | Decision | Why |
|---|---|---|
| Deadlines | A block that times out throws what amp's `TimeoutCancellation` reports: an `Amp\CancelledException` whose previous exception is the `Amp\TimeoutException` | A Phaenon-specific exception would be one more type to catch around amp code that already handles amp's |
| `P2604` | Only a mutation of an outer variable inside a `spawn` body is reported (`++`, compound assignment, an element write, `$x = $x …`); a plain rebinding that shadows it is allowed | The first golden reported `$n = …` inside a task, which is a new local in the closure the task compiles to, not a race |
| `P2606` | A block's options are named only: `timeout:` (int or float seconds) and `cancellation:` (`Amp\Cancellation`) | The options map one to one onto `Scope::run`'s parameters; positional options would read as a call |
| The `use` clause | The emitter captures by value what the body reads before writing and the scope defines before the block; by reference what the body assigns and the scope reads first after it; arrow-function parameters shadow | The first rule (everything mentioned, by reference) produced `&$path`, `&$repo` on loop variables in tq-cli; the refined one writes what a person writes |
| The runtime's source | `phaenon/concurrent`'s `Scope` is written in Phaenon (`stdlib/concurrent/Scope.phn`, with a `concurrently` block inside) and compiled into the package; the compiler reads the same file as the API's declaration | The class is the first dogfood of its own syntax, and the declaration cannot drift from the code |
| amp declarations | The compiler embeds the amp 3.1 and Revolt 1.0 signatures it relies on and loads them ahead of a project's vendor copy | The checker must know `Future<T>` in a project whose vendor directory is not installed yet; a missing method is a declaration to add |
| `tequila/nats` | The connection has one socket reader at a time, and an opt-in reader task per connection is that reader made permanent; `RevoltTransport` is a suggested dependency (merged upstream, the client's ADR amended with it) | C8 named a "single-reader change"; one reader at a time keeps worker mode's "nothing runs between requests" by default, and the reader task serves long-running amp processes without a second code path |
| Editor | The PhpStorm plugin approximates blocks at the token level: a statement block parses as a block, `spawn expr` as `expr`; the expression forms wait for a PSI element (M6) | The plugin erases Phaenon syntax in front of PhpStorm's parser; a block that yields a value has no PHP shape to erase to |

## 17. Withdrawn (2026-11-08)

Roadmap D25. The syntax is not in the language; this document stays as the specification of what
it means, for the day PHP core has a scheduler to map it onto.

| Topic | Decision | Why |
|---|---|---|
| **The syntax goes** | `concurrently { }` and `spawn` leave the parser, the checker (`P2601`–`P2606` retired and reserved, never reused), the emitter and the editors; `concurrently` and `spawn` are not reserved words, so a PHP function of either name compiles. `phaenon/concurrent` is archived on GitHub and abandoned on Packagist; the amp and Revolt declarations leave the compiler, which reads a project's vendor copy of amp like any other package. | D24: the compiled output depends on nothing but PHP. A block compiled to three production dependencies the compiler chose — `phaenon/concurrent`, amp, Revolt — in a project that may run another loop today or a better library tomorrow, and an outside reader of the output named that first. |
| **Not a pluggable target** | A project naming what a block compiles to was weighed and rejected. | Cancellation, failure propagation and the handle's type would be whatever the target does; the checker could promise none of §5, and two projects' blocks would mean two things. |
| **amp as amp** | A program that wants amp writes amp: `async()`, `await()`, `Future`, `TimeoutCancellation`, checked from the package's own declarations. tq-cli's fifteen blocks go back to `Api::concurrently()`. | Honest about the dependency, and no worse than the block for the checker: amp's signatures are PHPStan-typed. |
| **What brings it back** | A PHP release with a scheduler, an awaitable and cancellation of its own — the True Async work or a successor. The model of §3–§5 and the capture rules of §16 carry over; §6 is rewritten for the native primitives, and the dependency count stays at zero. Not a runtime of Phaenon's own (roadmap §3, Phase 3). | Structured concurrency needs the runtime's cooperation, and the runtime that can give it without a dependency is PHP. |
