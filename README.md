# Senior Full Stack Interview Prep — Volume 1 (v2, expanded)
## JavaScript · TypeScript · React · Next.js

**Prepared for:** Ansh Doshi — Senior Full Stack / Senior Software Engineer roles, India (~₹20 LPA)
**Accuracy baseline:** September 2026. Next.js 16.x (16.3 is the current minor), React 19.2.x, TypeScript 5.x.

**What's new in v2:**
- Every follow-up question now has a full answer.
- Side concepts (e.g. shallow vs deep copy, WeakMap, structural sharing, circuit breakers) have their own **Deep dive** blocks with examples.
- Explanations are longer and use plain-language analogies.
- A few v1 inaccuracies are corrected:
  - JS-4 now covers the strict-mode `this` behavior.
  - REACT-7 now explains why `useOptimistic` reverts unless the server data refreshes.
  - NEXT-5 now has exact CVE patch versions and hosting scope.

---

## How to use this file

**Structure of every question:**
1. **Short answer.** What you say in the interview, in 30–60 seconds.
2. **Explanation.** Why it works, in plain language.
3. **Deep dive.** Side concepts the interviewer can branch into.
4. **Example / Code.** TypeScript unless stated otherwise.
5. **Production use.** Where this appears in real systems, often your own resume.
6. **Common mistakes.** What weak candidates say.
7. **Follow-up Q&A.** Likely next questions, *with answers*.

**Difficulty:** 🟢 Basic · 🟡 Intermediate · 🟠 Advanced · 🔴 Senior/Production
**[RESUME]** = tied to your resume. Highest-probability questions.

**How to practice:** Read the short answer, close the file, say it out loud, then answer each follow-up without looking. If you can't explain a follow-up in your own words, you don't know it yet.

**Answer formula for every question:** *definition → why it matters → example → trade-off.* The trade-off at the end is what separates senior answers from mid-level ones.

**Series roadmap:**
- **Vol 1:** JS, TS, React, Next.js (this file)
- **Vol 2:** Node.js, Express, REST APIs, Authentication & Security
- **Vol 3:** PostgreSQL, MongoDB, Performance & Scalability, Debugging
- **Vol 4:** AWS, Docker, Kubernetes, Kafka, Distributed Systems, System Design
- **Vol 5:** AI / LLM / RAG, DSA coding set, Behavioral & Resume deep-dive

---

# PART A — JAVASCRIPT

### JS-1. `var` vs `let` vs `const`, hoisting, and the Temporal Dead Zone 🟢

**Short answer:**
- `var` is function-scoped, hoisted, and initialized to `undefined`.
- `let` and `const` are block-scoped. They are also hoisted, but they remain uninitialized until the declaration line executes. Accessing them earlier throws a `ReferenceError`. That window is the Temporal Dead Zone (TDZ).
- `const` prevents *reassigning the variable*. It does not make the value immutable.

**Explanation:**
Before running any scope, the engine does a setup pass and registers every declaration in that scope. That registration is "hoisting." The difference is what each declaration holds before its line runs:
- `var` is filled with `undefined` immediately. Reading it early silently gives `undefined`, which hides bugs.
- `let`/`const` are registered but locked. Reading them early throws, which exposes the bug immediately.
- Function declarations are hoisted *with their body*, so you can call them before they appear in the code. Function *expressions* assigned to `let`/`const` are in the TDZ like any other `let`/`const`.

Think of a hotel check-in. `var` gives you a room key with an empty room behind it. `let`/`const` reserve the room but won't hand you the key until check-in time.

**Scope difference:**
- *Function scope* (`var`): the variable is visible everywhere inside the enclosing function, including outside the `if` or `for` block where it was declared.
- *Block scope* (`let`/`const`): the variable exists only inside the nearest `{ }`.

```js
function demo() {
  if (true) { var a = 1; let b = 2; }
  console.log(a); // 1  — var leaks out of the if-block
  console.log(b); // ReferenceError — b only lived inside the block
}
```

**Example (run each snippet separately; the TDZ error stops execution):**
```js
console.log(x); // undefined
var x = 1;

console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 2;

const user = { name: "Ansh" };
user.name = "A";   // ✅ allowed: mutating the object
// user = {};      // ❌ TypeError: Assignment to constant variable
```

**The classic loop question:**
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i)); // 3 3 3
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j)); // 0 1 2
```
- **`var` version:** there is only **one** `i` for the whole loop. The callbacks run after the loop finishes, when `i` is already 3, so all three read that same `i`.
- **`let` version:** the spec creates a **new binding of `j` for every iteration**. Each callback closes over its own copy.

**Deep dive — making objects truly immutable:**
`Object.freeze` blocks adding, removing, or changing properties, but it's **shallow**:
```js
const cfg = Object.freeze({ db: { host: "a" } });
cfg.db = {};          // ignored (throws in strict mode)
cfg.db.host = "b";    // ✅ still works — nested object is NOT frozen

function deepFreeze(obj) {
  for (const v of Object.values(obj)) {
    if (v && typeof v === "object" && !Object.isFrozen(v)) deepFreeze(v);
  }
  return Object.freeze(obj);
}
```
In TypeScript, `readonly` and `as const` give *compile-time* immutability with zero runtime cost. `Object.freeze` gives *runtime* immutability.

**Production use:**
- Lint rules like `no-var` and `prefer-const` are standard.
- The `var`-in-loop bug appears in retry schedulers and batch timers written in older code.

**Common mistakes:**
- Saying "`let` is not hoisted."
- Saying "`const` means immutable."
- Not knowing that `typeof undeclaredVar` returns `"undefined"`, while `typeof` on a TDZ variable throws.

**Follow-up Q&A:**

**Q1. Why does the `var` loop print `3 3 3`?**
Because `var` creates a single function-scoped `i`. All three arrow functions close over the same variable. `setTimeout` callbacks are macrotasks that run after the synchronous loop completes, and by then `i` has been incremented to 3, which is also the value that ended the loop.

**Q2. Fix it without using `let`.**
Create a new scope per iteration, or pass the value in:
```js
// IIFE: each call gets its own parameter `n`
for (var i = 0; i < 3; i++) {
  (function (n) { setTimeout(() => console.log(n)); })(i);
}
// setTimeout forwards extra args to the callback
for (var i = 0; i < 3; i++) setTimeout((n) => console.log(n), 0, i);
// bind pre-fills the argument
for (var i = 0; i < 3; i++) setTimeout(console.log.bind(null, i));
```

**Q3. Is `const` faster than `let`?**
Not in any way that matters. Choose `const` for *readability*: it tells the reader the binding never changes.

**Q4. What is hoisted for a `class` declaration?**
Classes behave like `let`: they're hoisted but in the TDZ, so `new Foo()` before `class Foo {}` throws.

---

### JS-2. Closures — what, why, and a real use 🟢→🟡

**Short answer:** A closure is a function bundled with references to the variables of the scope it was created in. The function keeps access to those variables even after the outer function has returned. Closures enable private state, function factories, memoization, debounce/throttle, and how React hooks see props and state.

**Explanation:**
When a function is created, it stores a hidden link to its surrounding scope (the *lexical environment*). "Lexical" means *where the code is written*, not where it's called from. When the function later runs, variable lookup goes: its own locals → the captured outer scope → further outward → global.

Analogy: a closure is a backpack. When a function leaves the room where it was created, it takes a backpack holding references to the variables it needs.

**Key detail:** closures capture *variables* (references), not *values*. If the variable changes later, the closure sees the new value:
```js
let n = 1;
const read = () => n;
n = 2;
read(); // 2 — not 1
```

**Example — private state with a factory:**
```ts
function createRateLimiter(maxPerMinute: number) {
  let count = 0;                    // private: no outside code can touch this
  let windowStart = Date.now();
  return function allow(): boolean {
    const now = Date.now();
    if (now - windowStart >= 60_000) { count = 0; windowStart = now; }
    if (count >= maxPerMinute) return false;
    count++;
    return true;
  };
}
const allowOtp = createRateLimiter(5);
allowOtp(); // true ... sixth call in the same minute → false
```

**Deep dive — memoization with a closure:**
```ts
function memoize<A extends string | number, R>(fn: (a: A) => R) {
  const cache = new Map<A, R>();          // captured by the returned function
  return (a: A): R => {
    if (cache.has(a)) return cache.get(a)!;
    const r = fn(a);
    cache.set(a, r);
    return r;
  };
}
```
This cache grows forever. In a long-running server, that's a memory leak (see JS-8).

**Production use:**
- Middleware factories: `requireRole("admin")` returns a handler that remembers `"admin"`.
- Rate limiters, caches, and debounce functions.
- React: every render creates new closures over that render's props and state, which explains the stale-closure bug (REACT-3).

**Common mistakes:**
- Defining a closure as "a function inside a function." That's nesting. The key point is *retained access after the outer function returns*.
- Not realizing closures keep captured objects alive in memory.

**Follow-up Q&A:**

**Q1. How can closures cause memory leaks?**
Anything a live closure can reach can't be garbage-collected. Example: an event listener closure that references a large array. If the listener is never removed, the array lives forever. The same applies to a timer callback that's never cleared, or a module-level cache that only grows. Fix: remove listeners, clear timers, bound caches (LRU), and avoid capturing large objects you don't need.

**Q2. Would `createRateLimiter` work on Vercel serverless or multiple Node instances?**
No. Each instance has its own memory and therefore its own `count`. With 10 instances, the effective limit becomes up to 10× higher. Instances also restart often, which resets the counter. For a global limit, use shared storage: a Redis `INCR` with expiry, or a database row. This is exactly how OTP send limits should work. **[RESUME]**

**Q3. What does this print, and why?**
```js
function counter() { let c = 0; return { inc: () => ++c, get: () => c }; }
const a = counter(), b = counter();
a.inc(); a.inc(); b.inc();
console.log(a.get(), b.get()); // 2 1
```
Each call to `counter()` creates a new scope, so `a` and `b` have independent `c` variables.

**Q4. Closure vs class for private state?**
Both work. Classes now have true private fields (`#count`), which are clearer when there are multiple methods and you want `instanceof`. Closures are lighter for a single returned function. Mention both, and say either is fine when used consistently.

---

### JS-3. The Event Loop — predict the output 🟡 (asked constantly)

**Short answer:** JavaScript executes on one call stack. Each loop cycle:
1. Run the current synchronous code to completion.
2. Drain **all** microtasks: Promise `.then`/`.catch`/`.finally` callbacks, `await` continuations, `queueMicrotask`.
3. The browser may render.
4. Take **one** macrotask (timers, I/O callbacks, UI events, `MessageChannel`).
5. Repeat.

**Explanation:**
The event loop is a single worker with two queues:
- **Microtasks** are an express lane. The worker clears the entire express lane before taking the next regular task.
- **Macrotasks** (the "task queue") are the regular lane.

`await x` pauses the async function and schedules the rest of it as a microtask once `x` settles. The code *before* the first `await` runs synchronously.

**Example:**
```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
queueMicrotask(() => console.log("4"));
(async () => {
  console.log("5");
  await null;
  console.log("6");
})();
console.log("7");
// Output: 1 5 7 3 4 6 2
```

**Step by step:**

| Step | Action | Printed | Microtask queue | Macrotask queue |
|---|---|---|---|---|
| 1 | log 1 | 1 | – | – |
| 2 | setTimeout | – | – | [2] |
| 3 | `.then` scheduled | – | [3] | [2] |
| 4 | queueMicrotask | – | [3, 4] | [2] |
| 5 | async fn body runs synchronously | 5 | [3, 4] | [2] |
| 6 | `await null` schedules continuation | – | [3, 4, 6] | [2] |
| 7 | log 7 | 7 | … | … |
| 8 | drain microtasks | 3 4 6 | – | [2] |
| 9 | one macrotask | 2 | – | – |

**Deep dive — why "blocking the event loop" matters:**
- **Browser:** while your code runs, nothing else happens. No clicks, no typing, no painting. Any task over 50 ms is a "long task" and hurts INP (responsiveness).
- **Node:** one server process handles all requests on one thread. A 2-second CPU loop means *every* user waits 2 seconds.
- **Typical blockers:** `JSON.parse` on huge payloads, sorting big arrays, synchronous crypto or `fs` calls, catastrophic regexes (ReDoS).

**Deep dive — microtask starvation:**
```js
function loop() { Promise.resolve().then(loop); } // never yields to macrotasks
loop(); // page freezes: microtasks keep refilling, timers and rendering never run
```

**Production use:** Understanding why the UI freezes during a CSV export, or why one heavy API endpoint slows down all other endpoints on the same Node process.

**Common mistakes:**
- Thinking `setTimeout(fn, 0)` runs immediately. It runs after all current sync code and microtasks, and browsers clamp nested timers to at least ~4 ms.
- Thinking an `async` function is entirely asynchronous.

**Follow-up Q&A:**

**Q1. Where does Node's `process.nextTick` fit?**
- In Node, the `nextTick` queue is processed *before* the Promise microtask queue after each operation completes. So in a CommonJS script, `Promise.resolve().then(A); process.nextTick(B)` prints B then A.
- In ES modules, ordering can differ because module evaluation itself runs asynchronously. Don't depend on it.
- Overusing `nextTick` can starve I/O. Prefer `queueMicrotask` or `setImmediate`. (Full Node event-loop phases are in Vol 2.)

**Q2. How would you process 1 lakh records in the browser without freezing the UI?**
Three options, from simplest to strongest:
1. **Chunk and yield:** process ~500 items, then yield to the event loop so the browser can paint and handle input, then continue.
2. **`scheduler.yield()`:** a newer API built for this; check browser support and fall back to `setTimeout`.
3. **Web Worker:** move the computation to another thread entirely. Best for heavy CPU work.

```ts
async function processInChunks<T>(items: T[], fn: (t: T) => void, chunk = 500) {
  for (let i = 0; i < items.length; i += chunk) {
    items.slice(i, i + chunk).forEach(fn);
    await new Promise((r) => setTimeout(r, 0));   // yield: lets the browser paint and handle clicks
  }
}
```

**Q3. What is a macrotask vs a microtask, in one sentence each?**
- Macrotask: a unit of work the event loop picks up one at a time (a timer, an I/O callback, a click event).
- Microtask: a small follow-up job that runs immediately after the current task, before anything else (promise callbacks).

**Q4. Is JavaScript single-threaded?**
Your JavaScript code runs on one thread per realm. The runtime around it is not: browsers have network, rendering, and worker threads, and Node uses libuv's thread pool for file system, DNS, crypto, and zlib. That's how a single-threaded language does concurrent I/O.

---

### JS-4. `this` binding and arrow functions 🟡

**Short answer:** For regular functions, `this` is determined by **how the function is called**. There are four rules, in priority order:
1. `new Foo()` → the new object.
2. `fn.call(obj)` / `apply` / `bind` → `obj`.
3. `obj.method()` → `obj`.
4. A plain call `fn()` → `undefined` in strict mode (ES modules and classes are always strict), or `globalThis` in sloppy mode.

**Arrow functions have no `this` of their own.** They use the `this` of the surrounding scope at the time they were *created*, and `call`/`bind` can't change it.

**Explanation:**
Think of `this` as "who's asking." For regular functions, you find it at the call site: look left of the dot. `user.greet()` → `this` is `user`. If nothing is left of the dot, rule 4 applies. Arrow functions don't have their own "who's asking"; they borrow it from the surrounding code.

**Example:**
```js
"use strict";
const svc = {
  name: "billing",
  regular() { return this?.name; },
  arrow: () => this?.name,                    // `this` from module scope → undefined
  later() { setTimeout(function () { console.log(this?.name); }, 0); },  // undefined
  laterArrow() { setTimeout(() => console.log(this.name), 0); },        // "billing"
};

svc.regular();             // "billing" (rule 3)
const detached = svc.regular;
detached();                // undefined (rule 4: no object left of the dot)
svc.arrow();               // undefined (arrow ignores the call site)

const bound = svc.regular.bind({ name: "search" });
bound();                   // "search"
bound.call({ name: "x" }); // still "search": the first bind wins
```
**v1 correction:** calling a *detached* `laterArrow` would throw a `TypeError` in strict mode (reading `.name` of `undefined`). In sloppy mode it would read `globalThis.name` instead. Always state which mode you're assuming.

**Deep dive — `call` vs `apply` vs `bind`:**
- `call(thisArg, a, b)`: invoke now, arguments listed individually.
- `apply(thisArg, [a, b])`: invoke now, arguments as an array.
- `bind(thisArg, a)`: don't invoke. Returns a **new function** with `this` (and optionally leading arguments) fixed permanently.

**Production use — the lost-`this` bug in classes:**
```ts
class Uploader {
  private count = 0;
  onProgress() { this.count++; }                  // `this` depends on the caller
  onProgressSafe = () => { this.count++; };       // class-field arrow: always this instance
}
const u = new Uploader();
emitter.on("progress", u.onProgress);      // ❌ `this` is not `u` when emitter calls it
emitter.on("progress", u.onProgressSafe);  // ✅
```

**Common mistakes:**
- "Arrow functions bind `this` to the object they're in." Object literals don't create a `this` scope.
- Using arrow functions as prototype or object methods that need the object.

**Follow-up Q&A:**

**Q1. What does `bind` return? Can a bound function be re-bound?**
It returns a new "bound function." Re-binding it creates another wrapper, but `this` stays the value from the *first* bind. Only the pre-filled arguments can be extended.

**Q2. Why can't arrow functions be used with `new`?**
They have no `[[Construct]]` internal method and no `prototype` property, so `new arrow()` throws a `TypeError`. They also have no `arguments` object; use rest parameters (`...args`) instead.

**Q3. What is `this` inside a class static method?**
The class constructor itself, when called as `MyClass.staticMethod()`.

**Q4. When would you deliberately prefer a regular function over an arrow?**
- Object or prototype methods that need `this`.
- Functions called with `new`.
- DOM handlers where you want `this` to be the element.
- Anything that needs `arguments`.

---

### JS-5. `Promise.all` vs `allSettled` vs `race` vs `any` 🟡

**Short answer:**

| Method | Resolves when | Rejects when | Use for |
|---|---|---|---|
| `all` | all fulfill → array of values (input order) | **first** rejection | Data that's all mandatory |
| `allSettled` | all settle → `{status, value \| reason}[]` | never | Dashboards, bulk jobs with partial failure |
| `race` | first to *settle* (success or failure) | first to settle is a rejection | Timeouts (legacy pattern) |
| `any` | first to *fulfill* | all reject → `AggregateError` | Fastest mirror/replica, fallbacks |

**Explanation:** These are "combinators": they combine several promises into one. The key question is how each one treats failure.
- `all` is strict: one failure fails the whole thing.
- `allSettled` is a report card: it waits for everyone and tells you each outcome.
- `race` is about *time*: whoever settles first wins, success or failure.
- `any` is optimistic: the first success wins, and it only fails if every single one fails.

**Important:** none of these **cancel** the losing promises. A promise is a *result container*, not a handle to the work. The HTTP requests, DB queries, and LLM calls keep running and costing money.

**Example:**
```ts
// Dashboard: one failing widget shouldn't blank the page
const [profileR, jobsR, creditsR] = await Promise.allSettled([
  getProfile(id), getJobs(id), getCredits(id),
]);
const profile = profileR.status === "fulfilled" ? profileR.value : null;
if (creditsR.status === "rejected") logger.warn({ err: creditsR.reason }, "credits unavailable");
```

**Deep dive — real cancellation with `AbortController`:**
```ts
// Modern timeout: actually aborts the network request
const res = await fetch(url, { signal: AbortSignal.timeout(5_000) });
// On timeout, fetch rejects with a DOMException named "TimeoutError";
// on controller.abort() it's "AbortError"

// Combine a user "cancel" with a timeout
const userCancel = new AbortController();
const signal = AbortSignal.any([userCancel.signal, AbortSignal.timeout(5_000)]);
await fetch(url, { signal });
```

**Deep dive — why the old `race` timeout is weaker:**
```ts
const withTimeout = <T,>(p: Promise<T>, ms: number) =>
  Promise.race([p, new Promise<never>((_, rej) => setTimeout(() => rej(new Error("timeout")), ms))]);
```
- Your code stops waiting, but the request keeps running in the background.
- The timer isn't cleared on success, so it keeps a handle alive in Node.

Prefer an `AbortSignal` whenever the underlying API accepts one (fetch, most DB drivers, the Anthropic/OpenAI SDKs).

**Production use:**
- `allSettled` for bulk SMS/notification sends, so you can report "48 sent, 2 failed."
- `all` when rendering a page that truly needs every piece.
- `any` for querying multiple providers and taking the first good answer.

**Common mistakes:**
- "`Promise.all` runs things in parallel." The *work* is already started when you create the promises; `all` just waits. Creating promises inside a sequential `for await` loop is what makes things sequential.
- Assuming failure cancels the other promises.

**Follow-up Q&A:**

**Q1. How do you truly cancel a `fetch`?**
Pass a `signal` from an `AbortController` and call `controller.abort()`, or use `AbortSignal.timeout(ms)`. The browser closes the request, and the promise rejects with an `AbortError`/`TimeoutError` (handle these separately from real errors). On the server side, check `req.signal` or the connection-closed event to stop work for a client that left.

**Q2. Implement `allSettled`.**
```ts
function allSettled<T>(ps: Iterable<T | PromiseLike<T>>): Promise<PromiseSettledResult<Awaited<T>>[]> {
  return Promise.all(
    Array.from(ps, (p) =>
      Promise.resolve(p).then(
        (value) => ({ status: "fulfilled" as const, value }),
        (reason) => ({ status: "rejected" as const, reason }),
      ),
    ),
  ) as Promise<PromiseSettledResult<Awaited<T>>[]>;
}
```
Trick: convert every rejection into a *fulfilled* result object, so `Promise.all` never sees a failure.

**Q3. What happens to a rejected promise inside `Promise.all` after the first rejection?**
It's ignored by `all` (already settled), but it's still considered *handled*, so there's no unhandled-rejection warning.

**Q4. Sequential vs parallel awaits — what's the difference?**
```ts
for (const id of ids) await load(id);         // sequential: total = sum of durations
await Promise.all(ids.map(load));             // parallel: total ≈ slowest (but unbounded concurrency!)
```
With 1,000 ids, the parallel version fires 1,000 requests at once. Use a concurrency limit (JS-11).

---

### JS-6. Prototypes and inheritance 🟡

**Short answer:** Every object has an internal link, `[[Prototype]]`, to another object (or `null`). When you read a property that isn't on the object itself, JS walks up this *prototype chain*. `class`/`extends` is syntax over the same mechanism: methods live on `Class.prototype` and are shared by all instances.

**Explanation:**
- Reading `dog.speak` means: check `dog` → then `Dog.prototype` → then `Animal.prototype` → then `Object.prototype` → then `null` (not found, `undefined`).
- *Writing* a property always creates it on the object itself (it "shadows" the inherited one). It never modifies the prototype.

**Two names people confuse:**
- `obj.__proto__` (or `Object.getPrototypeOf(obj)`): the object's actual link upward.
- `Fn.prototype`: a property on *constructor functions*. It becomes the `__proto__` of objects created with `new Fn()`.

**Example:**
```js
class Animal { speak() { return "..."; } }
class Dog extends Animal { speak() { return "woof"; } }
const d = new Dog();

Object.getPrototypeOf(d) === Dog.prototype;                    // true
Object.getPrototypeOf(Dog.prototype) === Animal.prototype;     // true
d.hasOwnProperty("speak");                                     // false — it's inherited
```

**Deep dive — prototype pollution (a real security bug class):**
```js
// naive deep merge on untrusted JSON
function merge(target, src) {
  for (const k in src) {
    if (typeof src[k] === "object" && src[k] !== null) merge(target[k] ??= {}, src[k]);
    else target[k] = src[k];
  }
}
merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'));
({}).isAdmin; // true  ← EVERY object now "has" isAdmin
```
**Why it works:** `JSON.parse` creates `"__proto__"` as a normal key. When `merge` reads `target["__proto__"]`, it gets `Object.prototype` and writes onto it.

**Defenses:**
- Validate input shape with a schema (Zod strips unknown keys by default).
- Skip the keys `__proto__`, `constructor`, and `prototype` in merges.
- Use `Object.create(null)` or `Map` for dictionaries built from user input.
- Keep libraries patched (lodash had CVEs for exactly this).

**Common mistakes:**
- Saying "JS classes work like Java classes." They're prototype-based; methods are shared objects, not copied per instance.
- Mixing up `__proto__` and `.prototype`.

**Follow-up Q&A:**

**Q1. How do you prevent prototype pollution?**
Schema validation at the boundary, safe merge utilities that reject dangerous keys, `Map` or null-prototype objects for user-keyed data, and up-to-date dependencies. As defense in depth, `Object.freeze(Object.prototype)` can be used in some apps, but it can break libraries.

**Q2. What does `Object.create(null)` give you?**
An object with **no** prototype: no `toString`, no `hasOwnProperty`, no inherited keys. It's a safe dictionary. Use `Object.hasOwn(obj, k)` to check keys.

**Q3. `instanceof` vs `typeof`?**
- `typeof` returns a primitive type tag (`"object"`, `"function"`, …). Note `typeof null === "object"`, a historical bug.
- `instanceof` checks whether `Ctor.prototype` appears anywhere in the object's prototype chain. It can fail across realms (e.g., an array from an iframe), which is why `Array.isArray` exists.

---

### JS-7. Shallow copy vs deep copy — and `structuredClone` 🟡

**Short answer:**
- A **shallow copy** creates a new outer object, but nested objects and arrays are *shared* with the original.
- A **deep copy** recursively copies everything, so nothing is shared.
- Tools: spread/`Object.assign`/`Array.from`/`slice` are shallow. `structuredClone` is a proper deep clone for data. `JSON.parse(JSON.stringify())` is a lossy deep clone.

**Explanation:**
Variables holding objects store a **reference** (an address), not the object itself.
```js
const a = { x: 1 };
const b = a;       // NOT a copy — both point to the same object
b.x = 2;
a.x;               // 2
```
Analogy: a shallow copy is a new folder containing the *same* documents (shortcuts to the originals). A deep copy photocopies every document inside the folder too.

**Shallow copy in action:**
```js
const state = { name: "Ansh", filters: { city: "Ahmedabad", skills: ["React"] } };
const shallow = { ...state };

shallow.name = "A";                     // ✅ top level is independent
state.name;                             // "Ansh"

shallow.filters.city = "Pune";          // ❌ nested object is SHARED
state.filters.city;                     // "Pune"  ← original changed!
shallow.filters === state.filters;      // true
```

**Deep copy options compared:**

| Method | Nested independent? | Date | Map/Set | undefined | Functions | Circular refs | Class prototype |
|---|---|---|---|---|---|---|---|
| `{...obj}` / `Object.assign` | ❌ | shared ref | shared ref | ✅ kept | ✅ kept (shared) | n/a | ❌ plain object |
| `JSON.parse(JSON.stringify(x))` | ✅ | ❌ becomes string | ❌ becomes `{}` | ❌ key dropped | ❌ dropped | ❌ **throws** | ❌ lost |
| `structuredClone(x)` | ✅ | ✅ | ✅ | ✅ | ❌ **throws** DataCloneError | ✅ | ❌ lost (plain object) |
| lodash `cloneDeep` | ✅ | ✅ | ✅ | ✅ | kept as reference | ✅ | ✅ keeps prototype |

```js
const src = { at: new Date(), tags: new Set(["a"]), note: undefined };
JSON.parse(JSON.stringify(src)); // { at: "2026-09-24T...Z", tags: {} }  → note gone, Date is a string, Set empty
structuredClone(src);            // { at: Date, tags: Set{"a"}, note: undefined } ✅
```

**Deep dive — structural sharing (what React and Redux actually do):**
Deep-cloning an entire state tree on every update is slow and wasteful. Instead, copy **only the path that changed** and reuse every untouched branch:
```js
const next = {
  ...state,                               // new root
  filters: { ...state.filters, city: "Surat" },  // new filters object (changed path)
};
next.filters !== state.filters;           // true  → React sees a change here
next.profile === state.profile;           // true  → untouched branch reused (cheap, and memoized children skip re-rendering)
```
Immer (used inside Redux Toolkit) lets you write "mutating" code on a draft and produces this structurally shared result automatically:
```ts
import { produce } from "immer";
const next = produce(state, (draft) => { draft.filters.city = "Surat"; });
```

**Production use:**
- React state updates.
- Undo/redo snapshots (`structuredClone`).
- Copying config before modifying it per tenant.
- Sending data to Web Workers (`postMessage` uses the same structured clone algorithm).

**Common mistakes:**
- Using spread and assuming a deep copy. This is the #1 cause of "the other screen changed too" bugs.
- Using `JSON` cloning on data with Dates, then comparing dates as strings.
- Deep-cloning huge objects on every keystroke.

**Follow-up Q&A:**

**Q1. What is structural sharing and why does React depend on it?**
It means creating a new object only for the changed path and reusing references everywhere else. React (and `React.memo`, `useMemo`, Redux selectors) detect changes by *reference comparison*, which is O(1). Changed paths get new references, so React sees the change. Unchanged branches keep old references, so memoized components skip work. Deep cloning everything would give *every* branch a new reference, and everything would re-render.

**Q2. Write a deep clone function.**
```ts
function deepClone<T>(value: T, seen = new WeakMap<object, unknown>()): T {
  if (value === null || typeof value !== "object") return value;          // primitives
  if (seen.has(value as object)) return seen.get(value as object) as T;   // circular refs
  if (value instanceof Date) return new Date(value.getTime()) as T;
  if (value instanceof Map) {
    const m = new Map(); seen.set(value, m);
    value.forEach((v, k) => m.set(deepClone(k, seen), deepClone(v, seen)));
    return m as T;
  }
  if (value instanceof Set) {
    const s = new Set(); seen.set(value, s);
    value.forEach((v) => s.add(deepClone(v, seen)));
    return s as T;
  }
  const out: any = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value as object, out);
  for (const key of Reflect.ownKeys(value as object)) {       // includes symbol keys
    out[key] = deepClone((value as any)[key], seen);
  }
  return out;
}
```
Points to mention out loud:
- the base case for primitives
- the `WeakMap` for circular references
- special types (Date, Map, Set)
- preserving the prototype
- symbol keys

In real code you'd just use `structuredClone`.

**Q3. Is `Array.prototype.slice()` or `[...arr]` a deep copy?**
No. Both are shallow. `[[1],[2]]` copied with spread still shares the inner arrays.

**Q4. How do you compare two objects for equality?**
- `===` compares references.
- For *value* equality you need a deep-equal function (lodash `isEqual`, `node:util`'s `isDeepStrictEqual`).
- In React, avoid deep equality in hot paths. Rely on immutable updates so references are enough.

---

### JS-8. Memory leaks in JavaScript applications 🟠

**Short answer:** A leak is memory your program no longer needs but still *references*, so the garbage collector can't free it. The classic causes:
1. Forgotten event listeners and subscriptions
2. Uncleared `setInterval`/timers
3. Unbounded caches and module-level Maps
4. Closures capturing large objects
5. Detached DOM nodes still referenced from JS

**Explanation:**
JavaScript's garbage collector frees objects that are **unreachable** from "roots": globals, the current call stack, active closures, and registered handlers. It doesn't know whether you'll *use* an object again, only whether you *could reach* it. So a leak is always "something still points to it."

**Example — unbounded cache → bounded LRU:**
```ts
// ❌ grows forever in a long-running Node process
const cache = new Map<string, Result>();

// ✅ LRU: evict least-recently-used when full (Map keeps insertion order)
class LRU<K, V> {
  private m = new Map<K, V>();
  constructor(private max: number) {}
  get(k: K): V | undefined {
    const v = this.m.get(k);
    if (v !== undefined) { this.m.delete(k); this.m.set(k, v); } // move to "most recent"
    return v;
  }
  set(k: K, v: V) {
    this.m.delete(k);
    this.m.set(k, v);
    if (this.m.size > this.max) this.m.delete(this.m.keys().next().value!); // oldest = first key
  }
}
```

**Example — React listener leak:**
```tsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);  // without this: leak + setState after unmount
}, []);
```

**How to diagnose (the senior part):**

*Browser — the three-snapshot technique:*
1. Open DevTools → Memory → take heap snapshot A.
2. Perform the suspected action (open/close a modal) about 10 times, then take snapshot B.
3. Use the Comparison view and sort by "# Delta." Look for objects whose count grew by roughly 10×.
4. Search for "Detached" to find DOM nodes that were removed from the page but are still referenced.
5. The "Retainers" panel shows *who* is holding the object. That's your bug.

*Node:*
1. Graph `process.memoryUsage().heapUsed` over time under steady load. A sawtooth that trends upward is a leak; a flat sawtooth is normal GC.
2. Take heap snapshots with `node --inspect` + Chrome DevTools, or `--heapsnapshot-signal=SIGUSR2` in production-like environments.
3. Compare snapshots the same way as in the browser.
4. Reproduce with a load test (autocannon/k6) instead of guessing.

**Deep dive — `WeakMap`, `WeakRef`, `FinalizationRegistry`:**
- **`WeakMap`**: keys must be objects, and the map **doesn't keep keys alive**. When the key object is garbage-collected, its entry disappears. Ideal for attaching metadata to objects you don't own, like caching computed data per DOM node or per request object. It isn't iterable and has no `.size`, because entries can vanish at any time.
- **`WeakRef`**: holds an object without preventing collection; `.deref()` returns it or `undefined`. Rarely needed. GC timing is unpredictable, so never build correctness on it.

**Production use:**
- A recruiter keeps the SPA open all day, and each navigation leaks a subscription. By evening the tab uses 2 GB.
- A Node container slowly grows until Kubernetes OOM-kills it (exit code 137), and it restarts every few hours. The restart hides the leak.

**Common mistakes:**
- "Restarting fixes it" with no root cause.
- Confusing high memory with a leak. Caches using memory by design are fine if bounded.
- Calling `global.gc()` in production code.

**Follow-up Q&A:**

**Q1. When would you use a `WeakMap`?**
When you need to associate data with an object whose lifetime you don't control, without extending that lifetime. Examples: memoizing a heavy computation per object, private data per class instance (before `#private` fields existed), tracking which DOM nodes already have a listener attached.

**Q2. How would you catch a leak before production?**
- **Soak tests:** run a realistic load for 1–2 hours in staging and check that memory plateaus.
- **Memory panels** in monitoring with alerts on sustained growth.
- **Lint rules** that flag missing effect cleanup.
- **Code review** attention on module-level mutable state in server code.

**Q3. Does a closure always keep the entire outer scope alive?**
Engines like V8 optimize to keep only the variables the closure actually uses, but variables shared between closures created in the same scope are kept together. Practical rule: don't create long-lived callbacks in scopes holding huge temporary objects.

**Q4. Your Node API's memory grows 50 MB/hour. What are your first three checks?**
1. Module-level Maps, arrays, or caches that only grow (per-user or per-request entries).
2. Listeners added per request and never removed (e.g., `emitter.on` inside a handler).
3. Timers or intervals created per request and never cleared.

Then take heap snapshots to confirm rather than guess.

---

### JS-9. [CODING] Debounce and throttle 🟡 (very common)

**Short answer:**
- **Debounce:** wait until calls *stop* for N ms, then run once with the latest arguments. Use for search-as-you-type, autosave, window-resize-end.
- **Throttle:** run at most once every N ms while calls keep coming. Use for scroll handlers, mouse-move, analytics pings, rate-limited buttons.

**Explanation (lift analogy):**
- **Debounce** is a lift door that waits 3 seconds after the *last* person enters before closing. Every new person resets the wait.
- **Throttle** is a lift that departs every 30 seconds no matter how many people keep arriving.

Timeline, with a call on every 100 ms tick for 1 second and N = 300 ms:
- Debounce fires **once**, 300 ms after the last call.
- Throttle (leading + trailing) fires at 0 ms, ~300, ~600, ~900, plus a final trailing call.

**Approach:** Keep timer state in a closure (JS-2).
- Debounce: clear and restart the timer on every call.
- Throttle: record the last run time and ignore calls inside the window, but schedule one *trailing* call so the final event isn't lost.

```ts
export function debounce<A extends unknown[]>(fn: (...args: A) => void, wait: number) {
  let t: ReturnType<typeof setTimeout> | undefined;
  const debounced = (...args: A) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), wait);
  };
  debounced.cancel = () => clearTimeout(t);
  return debounced;
}

export function throttle<A extends unknown[]>(fn: (...args: A) => void, wait: number) {
  let last = 0;
  let trailing: ReturnType<typeof setTimeout> | undefined;
  let lastArgs: A;
  return (...args: A) => {
    lastArgs = args;                              // trailing call uses the LATEST args
    const remaining = wait - (Date.now() - last);
    if (remaining <= 0) {
      clearTimeout(trailing); trailing = undefined;
      last = Date.now(); fn(...args);             // leading edge
    } else if (!trailing) {
      trailing = setTimeout(() => {
        last = Date.now(); trailing = undefined; fn(...lastArgs);   // trailing edge
      }, remaining);
    }
  };
}
```

**Complexity:** O(1) time and space per call.

**Edge cases to mention:**
- Cancel on unmount (otherwise the callback runs after the component is gone).
- The trailing call must use the *latest* args (e.g., final scroll position).
- `this` binding if used as an object method (use `fn.apply(this, args)` in a regular function version).
- `wait = 0` still defers to the next macrotask.

**Deep dive — debounce in React (the common bug):**
```tsx
// ❌ new debounced function each render → new timer each render → never debounces
function Search() {
  const onChange = debounce((q: string) => search(q), 300);
}

// ✅ stable across renders
function Search() {
  const searchRef = useRef(search); searchRef.current = search;       // always latest search fn
  const debounced = useMemo(() => debounce((q: string) => searchRef.current(q), 300), []);
  useEffect(() => () => debounced.cancel(), [debounced]);              // cleanup on unmount
  return <input onChange={(e) => debounced(e.target.value)} />;
}
```
Often simpler: debounce the *value* instead of the function (see `useDebouncedValue` in REACT-12).

**Production use [RESUME]:** Recruiter candidate search on Typesense. Debounce at ~300 ms, *and* abort in-flight requests. Debouncing reduces request count; aborting prevents out-of-order results.

**Common mistakes:**
- Mixing up the two definitions.
- Creating the debounced function inside render.
- Believing debounce prevents race conditions.

**Follow-up Q&A:**

**Q1. Add a `leading` option to debounce.**
```ts
function debounceLeading<A extends unknown[]>(fn: (...a: A) => void, wait: number) {
  let t: ReturnType<typeof setTimeout> | undefined;
  return (...args: A) => {
    if (!t) fn(...args);                                   // fire immediately on first call
    clearTimeout(t);
    t = setTimeout(() => { t = undefined; }, wait);        // reopen after quiet period
  };
}
```
Use case: a "Submit" button. Act on the first click and ignore rapid double-clicks.

**Q2. Why doesn't debouncing solve race conditions?**
Debounce controls *when requests start*, not *when they finish*. The user types "rea", pauses 300 ms, request A starts. Then types "react", pauses, request B starts. If A is slower, it arrives after B and overwrites the results. The fix is to abort the previous request or ignore stale responses.

**Q3. Throttle vs `requestAnimationFrame` for scroll?**
For work that updates visuals, `requestAnimationFrame` syncs to the display's frame rate (~16 ms at 60 Hz) and skips frames when the tab is hidden. That's better than a fixed-time throttle. Also consider `IntersectionObserver` instead of scroll handlers for "is this element visible" logic.

---

### JS-10. [CODING] Implement `Promise.all` 🟡

**Problem:** Write `promiseAll(items)` with the same behavior as `Promise.all`:
- Accepts any iterable of values or promises.
- Resolves with results in **input order**.
- Rejects on the first rejection.
- Resolves immediately for an empty input.

**Approach:**
1. Convert the iterable to an array and preallocate the results.
2. Wrap each item with `Promise.resolve` so plain values and thenables work.
3. Store each result at its **original index** and count completions.
4. When the count equals the length, resolve. Pass `reject` directly for the first failure.

```ts
export function promiseAll<T>(items: Iterable<T | PromiseLike<T>>): Promise<Awaited<T>[]> {
  return new Promise((resolve, reject) => {
    const arr = Array.from(items);
    const results = new Array(arr.length);
    let done = 0;
    if (arr.length === 0) return resolve([]);
    arr.forEach((item, i) => {
      Promise.resolve(item).then((value) => {
        results[i] = value;                         // index, not push → order preserved
        if (++done === arr.length) resolve(results);
      }, reject);                                   // first rejection wins; later ones are no-ops
    });
  });
}
```

**Complexity:** O(n) time, O(n) space.

**Edge cases:**
- Empty input resolves to `[]`. **Without this check it hangs forever.**
- Non-promise values.
- Thenables (objects with a `.then`).
- Duplicate promises in the input.
- Multiple rejections: only the first matters, because a promise can settle only once.

**Explanation — why the counter and not `results.length`?**
Arrays with preassigned indexes can have a `length` equal to n while slots are still empty. A separate counter is the only reliable completion signal.

**Common mistakes:**
- `results.push(value)`, which orders by completion time, not input order.
- Forgetting the empty case.
- `await`-ing items in a loop, which is sequential and defeats the purpose.

**Follow-up Q&A:**

**Q1. Implement `Promise.race`.**
```ts
function promiseRace<T>(items: Iterable<T | PromiseLike<T>>): Promise<Awaited<T>> {
  return new Promise((resolve, reject) => {
    for (const item of items) Promise.resolve(item).then(resolve, reject);
  });
}
```
An empty input stays pending forever. That matches the spec.

**Q2. Implement `Promise.any`.**
Resolve on the first fulfillment. Count rejections, and if all reject, reject with `new AggregateError(errors, "All promises were rejected")`. An empty input rejects immediately.

**Q3. Implement it with a concurrency limit.**
See JS-11. That's the version interviewers use to separate mid-level from senior candidates.

---

### JS-11. [CODING] Run async tasks with a concurrency limit 🟠 [RESUME]

**Problem:** Process N async tasks with at most K running at once, and return results in input order. Context: your Gemini matching is "rate-limited." How do you score 500 candidates with only 5 LLM calls in flight?

**Explanation:**
- `Promise.all(items.map(fn))` starts **all** 500 calls instantly. You'll hit HTTP 429 (Too Many Requests), timeouts, and cost spikes.
- The fix is a *worker pool*: start K workers, and each worker loops, pulling the next item index until none are left. A worker only takes a new task when its previous one is done, so at most K are ever in flight.
- The shared counter `next++` is safe without locks because JS runs one piece of synchronous code at a time. No two workers can execute `next++` simultaneously; they only interleave at `await` points.

```ts
export async function mapWithConcurrency<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T, index: number) => Promise<R>,
): Promise<R[]> {
  if (!Number.isInteger(limit) || limit < 1) throw new RangeError("limit must be >= 1");
  const results = new Array<R>(items.length);
  let next = 0;

  async function worker() {
    while (true) {
      const i = next++;
      if (i >= items.length) return;
      results[i] = await fn(items[i], i);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

// usage
const scores = await mapWithConcurrency(candidates, 5, (c) => scoreCandidate(jd, c));
```

**Complexity:**
- Time: O(n) dispatch overhead. Wall-clock time ≈ ⌈n/K⌉ × average task duration when durations are similar.
- Space: O(n) for results plus O(K) active workers.

**Edge cases:**
- Empty input returns `[]`.
- K > n: spawn only n workers.
- K invalid: throw.
- **Error behavior:** the first rejection rejects the outer promise, *but the other workers keep running*. For bulk jobs, catch inside `fn` and return a result object.

**Deep dive — tolerate failures instead of failing fast:**
```ts
type Result<R> = { ok: true; value: R } | { ok: false; error: unknown };

const results = await mapWithConcurrency(candidates, 5, async (c): Promise<Result<Score>> => {
  try { return { ok: true, value: await scoreCandidate(jd, c) }; }
  catch (error) { return { ok: false, error }; }
});
const failed = results.filter((r) => !r.ok).length;   // report "495 scored, 5 failed"
```

**Deep dive — concurrency limit vs rate limit (key senior distinction):**
- **Concurrency limit:** max requests *in flight at the same moment* (e.g., 5).
- **Rate limit:** max requests *per time window* (e.g., 60/minute) or tokens per minute.

If each call takes 200 ms, a concurrency of 5 can still send ~1,500 requests per minute and break a 60 RPM quota. LLM providers enforce both request-rate and token-rate limits, so production code often needs a pool *and* a token bucket:
```ts
class TokenBucket {
  private tokens: number; private last = Date.now();
  constructor(private capacity: number, private refillPerSec: number) { this.tokens = capacity; }
  async take() {
    while (true) {
      const now = Date.now();
      this.tokens = Math.min(this.capacity, this.tokens + ((now - this.last) / 1000) * this.refillPerSec);
      this.last = now;
      if (this.tokens >= 1) { this.tokens -= 1; return; }
      await new Promise((r) => setTimeout(r, ((1 - this.tokens) / this.refillPerSec) * 1000));
    }
  }
}
const bucket = new TokenBucket(10, 1);               // burst 10, then 1 request/second (60 RPM)
await mapWithConcurrency(candidates, 5, async (c) => { await bucket.take(); return scoreCandidate(jd, c); });
```

**Common mistakes:**
- `Promise.all(items.map(fn))` for large N.
- Batching in fixed chunks of 5 with `Promise.all` per chunk. That works, but each chunk waits for its slowest task. A pool keeps all K slots busy.
- Adding a mutex around `next++`.

**Follow-up Q&A:**

**Q1. How do you add retries?**
Wrap `fn` with `retry()` from JS-12: `(c) => retry(() => scoreCandidate(jd, c), { shouldRetry: isTransient })`. Retries occupy a pool slot, which is correct because they are real in-flight load.

**Q2. How do you enforce the limit across multiple server instances or Vercel functions?**
In-memory pools only limit *one* process. For a global limit, use shared state:
- A Redis-based limiter (a sliding window, or a semaphore implemented with a counter plus expiry).
- Or, better at scale, put jobs on a **queue** (SQS, BullMQ, Kafka) and control concurrency by the number of consumers. (Vol 4.)

**Q3. How would you make it cancellable?**
Pass an `AbortSignal`. Each worker checks `signal.aborted` before taking the next item and passes the signal into `fn` (e.g., into `fetch`).

**Q4. Chunking vs pool — when is chunking acceptable?**
When tasks take similar durations, or when the downstream API has a batch endpoint (e.g., Typesense `import` with 500 documents per call). Batch endpoints beat many single calls.

---

### JS-12. [CODING] Retry with exponential backoff and jitter 🟠

**Short answer:**
- Retry only **transient** failures: network errors, timeouts, HTTP 429, 502/503/504.
- Retry only **idempotent** operations, or operations protected by an idempotency key.
- Wait `min(cap, base × 2^attempt)`, randomized ("jitter") so clients don't retry in sync.
- Set a max attempt count and an overall deadline.

**Explanation:**
- **Exponential:** each wait doubles (200 ms → 400 → 800 …), giving an overloaded service room to recover.
- **Jitter:** if 10,000 clients fail at the same moment and all retry after exactly 400 ms, they hit the recovering server as one synchronized wave (the "thundering herd"). Randomizing each wait spreads the load out.
- **Idempotent:** doing it twice has the same effect as doing it once. `SET status = 'active'` is idempotent. `credits = credits - 10` is **not**.

```ts
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export async function retry<T>(
  fn: (attempt: number) => Promise<T>,
  opts: { retries?: number; baseMs?: number; maxMs?: number; shouldRetry?: (e: unknown) => boolean } = {},
): Promise<T> {
  const { retries = 3, baseMs = 200, maxMs = 5_000, shouldRetry = () => true } = opts;
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      if (attempt >= retries || !shouldRetry(err)) throw err;
      const cap = Math.min(maxMs, baseMs * 2 ** attempt);
      await sleep(Math.random() * cap);             // "full jitter": uniform in [0, cap)
    }
  }
}

export const isTransient = (e: any) =>
  e?.name === "TimeoutError" || e?.code === "ECONNRESET" ||
  e?.status === 429 || e?.status === 502 || e?.status === 503 || e?.status === 504;
```

**Edge cases:**
- Honor the `Retry-After` header on 429/503; the server is telling you exactly how long to wait.
- **Never retry** 400 (bad input), 401/403 (auth), 404, or 422. They will fail again.
- Enforce an overall deadline so retries don't exceed the caller's timeout.
- Log every retry with the attempt number, for debugging.

**Deep dive — idempotency keys (how to safely retry writes):**
The client generates a unique key per *logical operation* and sends it with every attempt. The server stores the key with the result and returns the stored result for duplicates.
```sql
CREATE TABLE credit_charges (
  idempotency_key TEXT PRIMARY KEY,           -- unique → second insert fails
  dealer_id UUID NOT NULL,
  amount INT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
-- charge only if this key hasn't been seen:
INSERT INTO credit_charges (idempotency_key, dealer_id, amount)
VALUES ($1, $2, $3) ON CONFLICT (idempotency_key) DO NOTHING RETURNING *;
```
If no row is returned, the charge already happened, so return the original result instead of charging again. Razorpay, Stripe, and most payment APIs support idempotency keys for this reason.

**Deep dive — circuit breaker:**
Retries help with *brief* blips. During a *long* outage, retries just add load and slow every user down. A circuit breaker tracks failures:
- **Closed** (normal): requests flow through.
- **Open**: after N failures in a window, fail fast for a cooldown period *without calling* the service.
- **Half-open**: after the cooldown, let one trial request through. Success closes the breaker; failure reopens it.
```ts
class CircuitBreaker {
  private failures = 0; private openedAt = 0;
  constructor(private threshold = 5, private cooldownMs = 30_000) {}
  async exec<T>(fn: () => Promise<T>): Promise<T> {
    if (this.failures >= this.threshold) {
      if (Date.now() - this.openedAt < this.cooldownMs) throw new Error("circuit open");
      // cooldown over → half-open: let calls through as a trial
      // (simplified: production breakers allow only ONE trial call at a time)
    }
    try {
      const r = await fn();
      this.failures = 0;                                        // success → close
      return r;
    } catch (e) {
      this.failures++;
      if (this.failures >= this.threshold) this.openedAt = Date.now();  // trip or re-trip
      throw e;
    }
  }
}
```
Production libraries: `opossum` for Node. Service meshes can do this at the infrastructure level.

**Production use [RESUME]:**
- Third-party vehicle/data lookups, SMS/OTP providers, Gemini calls, Razorpay verification.
- Your "charge credits only for successful paid lookups" rule is exactly where retries plus idempotency matter. A retried lookup must never be billed twice.

**Common mistakes:**
- Retrying non-idempotent writes (double charges, duplicate SMS).
- **Retry amplification:** retries at the browser, API, and worker layers multiply. 3 × 3 × 3 = 27 attempts per user action during an outage.
- No jitter.

**Follow-up Q&A:**

**Q1. What's a circuit breaker and how does it relate to retries?**
Retries handle short, random failures. A circuit breaker handles sustained failure by stopping calls entirely for a cooldown period. You use both: retries *inside* a closed breaker, and fast failure when it's open, often with a fallback like cached data or a "try later" message.

**Q2. How does this relate to your credit billing design?** **[RESUME]**
Two rules:
- Charge only when the lookup result is a confirmed success (model outcomes as a discriminated union; see TS-4).
- Make the charge idempotent with a key derived from the lookup request ID, so retries, double-clicks, or duplicate cron runs can't double-bill.

Prepare to explain what *you* actually did to prevent double charges.

**Q3. Where should retries live — client or server?**
Usually at *one* layer, closest to the failing dependency: in the server code that calls the third-party API. Clients should retry only safe reads or idempotent requests, with a small budget.

**Q4. What's "full jitter" vs "equal jitter"?**
- Full jitter: `random(0, cap)`. It spreads load most.
- Equal jitter: `cap/2 + random(0, cap/2)`. It guarantees a minimum wait.

AWS's architecture guidance popularized full jitter as a good default.

---

# PART B — TYPESCRIPT

### TS-1. `any` vs `unknown` vs `never` 🟢

**Short answer:**
- `any` disables type checking. You can do anything with it, and errors surface at runtime.
- `unknown` is "some value, type not known yet." You must *narrow* it before using it. It's the safe version of `any`.
- `never` is "no value can exist here." It's the type of functions that never return, and of branches that are impossible after exhaustive checks.

**Explanation — a mental model using sets:**
- `unknown` is the set of *all* values (the top type). Everything is assignable to it, but you can't do anything with it until you prove what it is.
- `never` is the *empty* set (the bottom type). It's assignable to everything, and nothing (except `never`) is assignable to it.
- `any` is an escape hatch that breaks the rules in both directions, which is why it's dangerous.

**Example:**
```ts
function toUpper(input: unknown) {
  // input.toUpperCase();                     // ❌ compile error: 'input' is of type 'unknown'
  if (typeof input === "string") return input.toUpperCase();   // ✅ narrowed to string
  throw new Error("expected string");
}

function fail(msg: string): never { throw new Error(msg); }    // never returns

function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}
```

**Deep dive — how `any` spreads silently:**
```ts
const data: any = JSON.parse(body);   // JSON.parse returns any
const id = data.user.id;              // any — no error even if user is undefined
const total = id * 2;                 // any — the "virus" spread to total
```
With `unknown`, you'd be forced to validate first. Enable `strict: true` and lint rules like `@typescript-eslint/no-explicit-any` and `no-unsafe-*` to keep `any` out.

**Production use:**
- `catch (e)` variables are `unknown` under `strict` (via `useUnknownInCatchVariables`).
- `fetch().json()` returns `Promise<any>`. Treat the result as `unknown` and validate it (TS-7).
- LLM JSON output is always `unknown`.

**Common mistakes:**
- Using `any` to make errors go away.
- `catch (e: any) { e.message }` crashes when something throws a string.

**Follow-up Q&A:**

**Q1. Where does `never` appear automatically?**
- After exhausting a union (e.g., in the `default` of a complete `switch`).
- Intersections of incompatible types (`string & number`).
- Functions that always throw or loop forever.
- Conditional types that filter members out (`Exclude<"a" | "b", "a" | "b">` is `never`).

**Q2. How do you safely read an error message from `unknown`?**
```ts
function errorMessage(e: unknown): string {
  if (e instanceof Error) return e.message;
  if (typeof e === "string") return e;
  try { return JSON.stringify(e); } catch { return String(e); }
}
```

**Q3. `unknown` vs `{}` vs `object`?**
- `{}` means any non-null, non-undefined value, including primitives like `5`.
- `object` means non-primitive values only.
- `unknown` includes everything, even `null` and `undefined`.

For "I don't know what this is," use `unknown`.

---

### TS-2. `interface` vs `type` 🟢

**Short answer:** Both can describe object shapes, and for plain objects they're mostly interchangeable. Differences:
- **Only `type`** can express unions, tuples, primitives, mapped types, and conditional types.
- **Only `interface`** supports *declaration merging* (two declarations with the same name combine), which is how you augment library types.
- `interface extends` reports conflicts clearly and is cached better by the compiler in large type hierarchies. With `&` intersections, conflicting properties can silently become `never`.

**Rule of thumb:** `interface` for object contracts meant to be extended, `type` for unions and computed types. Consistency matters more than the choice.

```ts
interface User { id: string; name: string }
interface Admin extends User { permissions: string[] }

type Status = "active" | "suspended";                 // union → type only
type Pair = [lat: number, lng: number];               // tuple → type only
type WithTimestamps<T> = T & { createdAt: Date };     // generic helper

// declaration merging: augment a library's type
declare module "next-auth" {
  interface Session { dealerId: string }
}
```

**Deep dive — structural typing:**
TypeScript compares types by **shape**, not by name. If two types have the same properties, they're compatible:
```ts
type UserId = string;
type DealerId = string;
function getDealer(id: DealerId) {}
const userId: UserId = "u_123";
getDealer(userId);   // ✅ compiles — both are just string. This is a real bug source.
```

**Deep dive — branded types (the fix):**
```ts
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type DealerId = Brand<string, "DealerId">;

const asDealerId = (s: string) => s as DealerId;      // create only at trusted boundaries (after validation)
function getDealer(id: DealerId) {}
getDealer("u_123" as UserId);  // ❌ compile error now
```
Zod supports this with `.brand<"DealerId">()`.

**Common mistakes:**
- Claiming one is "always better."
- Not knowing about declaration merging, which is how types like `Express.Request` get a `user` field added.

**Follow-up Q&A:**

**Q1. What is structural typing, and what problem does it cause?**
Compatibility is based on shape, not declared name. It's flexible, since you don't need `implements` everywhere. But semantically different values with the same underlying type (`UserId` vs `DealerId`, `Rupees` vs `Paise`) can be mixed up silently. Branded types fix that at compile time.

**Q2. How do you add `user` to Express's `Request` type?**
```ts
declare global {
  namespace Express { interface Request { user?: { id: string; role: Role; dealerId: string } } }
}
```
This works because `interface` merges.

**Q3. Can a class `implements` a `type`?**
Yes, if the type is an object type (not a union). `implements` only checks the shape at compile time; it doesn't add anything at runtime.

---

### TS-3. Generics with constraints 🟡

**Short answer:** Generics are type parameters. They let a function or type work with many types while **preserving the relationship** between input and output. `extends` constrains what's allowed, and `keyof`/indexed access (`T[K]`) connect keys to their value types.

**Explanation:** Without generics you'd pick between `any` (no safety) and duplicating code per type. With a generic, the caller's type flows through: "whatever type goes in, the matching type comes out."

```ts
function first<T>(arr: T[]): T | undefined { return arr[0]; }
first([1, 2]);        // number | undefined
first(["a"]);         // string | undefined

function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const c = { name: "Ravi", exp: 4 };
getProp(c, "exp");        // number
// getProp(c, "salary");  // ❌ '"salary"' is not assignable to '"name" | "exp"'
```

**Example — typed API client (one function, correct type per endpoint):**
```ts
type Endpoints = {
  "/candidates": { id: string; name: string }[];
  "/credits": { balance: number };
};
async function api<P extends keyof Endpoints>(path: P): Promise<Endpoints[P]> {
  const res = await fetch(path);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()) as Endpoints[P];     // ⚠ still a trust boundary: validate in real code
}
const credits = await api("/credits");            // { balance: number }
```

**Deep dive — generic defaults and inference:**
```ts
type ApiResponse<T = unknown, E = { message: string }> =
  | { ok: true; data: T }
  | { ok: false; error: E };
```
Let TypeScript *infer* type arguments where possible. Write them explicitly only when inference can't work (e.g., `useState<User | null>(null)`, where `null` alone would infer the type `null`).

**Production use:**
- Typed API clients and repositories (`Repository<T>`).
- Reusable components: TanStack Table's `ColumnDef<TData>`, generic `Select<T>` components.
- Result types.

**Common mistakes:**
- `function f<T>(x: T): any`, where the generic contributes nothing.
- Over-generic code nobody can read.

**Follow-up Q&A:**

**Q1. What does `as` do at runtime?**
Nothing. It's a compile-time assertion ("trust me"). It doesn't convert or check anything. `res.json() as User` compiles even if the server returns an error object, so the bug shows up later as `undefined` somewhere. Assertions are acceptable only after you've validated the data yourself.

**Q2. Write a generic `groupBy`.**
```ts
function groupBy<T, K extends PropertyKey>(items: readonly T[], keyFn: (t: T) => K): Record<K, T[]> {
  const out = {} as Record<K, T[]>;
  for (const item of items) (out[keyFn(item)] ??= []).push(item);
  return out;
}
groupBy(candidates, (c) => c.city);   // Record<string, Candidate[]>
```
(Modern runtimes also have a built-in `Object.groupBy`.)

**Q3. What's a conditional type?**
A type-level if/else: `T extends U ? X : Y`. Example: `type ElementOf<T> = T extends (infer E)[] ? E : never;` `infer` captures a part of a type.

---

### TS-4. Discriminated unions and exhaustive checking 🟡 (high value)

**Short answer:** A discriminated (tagged) union is a union of object types that share a literal-typed field, the **discriminant** (e.g., `status`). Checking that field narrows the type automatically, so only the valid fields for that case are accessible. A `default` branch that calls `assertNever` turns "forgot to handle a new case" into a compile error.

**Explanation:** It encodes "exactly one of these shapes" directly in the type, so impossible states can't be represented. You don't need to check `if (data && !error && !loading)`; the tag tells you exactly which fields exist.

```ts
type LookupResult =
  | { status: "success"; data: VehicleInfo; costCredits: number }
  | { status: "not_found" }
  | { status: "provider_error"; retryable: boolean };

function creditsToCharge(r: LookupResult): number {
  switch (r.status) {
    case "success":        return r.costCredits;   // r.data is accessible ONLY here
    case "not_found":      return 0;
    case "provider_error": return 0;
    default:               return assertNever(r);  // add "rate_limited" later → compile error here
  }
}
```

**Deep dive — "make impossible states unrepresentable":**
```ts
// ❌ booleans allow nonsense: { loading: true, error: "x", data: {...} } compiles
type Bad<T> = { loading: boolean; error?: string; data?: T };

// ✅ only valid combinations exist
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };
```
The same idea applies to backend workflows. For a job with statuses `draft → pending_approval → published`, model each state with the fields it actually has (e.g., `approvedBy` exists only after approval).

**Production use [RESUME]:**
- Your billing rule, "charge credits only for successful paid lookups": modeling outcomes as a union makes it impossible to forget a case when a provider adds a new status.
- Your 6-stage recruitment pipeline with human approval is also a natural state machine.

**Common mistakes:**
- A discriminant typed as plain `string` instead of literal types. Narrowing doesn't work then.
- Destructuring before narrowing, which loses the narrowing in older TS versions.

**Follow-up Q&A:**

**Q1. How would you model API request state in React?**
With the `RequestState<T>` union above, rendering by `switch (state.status)`. TanStack Query already exposes a similar discriminant (`status: "pending" | "error" | "success"`).

**Q2. What happens in `assertNever` at runtime if bad data arrives?**
It throws with the unexpected value. That's the right behavior: data from the network *can* contain a status your code doesn't know, even if TypeScript says it can't. It's a type guarantee and a runtime alarm at the same time.

**Q3. How do you model the recruitment pipeline stages safely?**
Use a union of stage objects, plus a transition function that only allows valid moves:
```ts
type Stage = "sourced" | "screened" | "interview" | "offer" | "hired" | "rejected";
const allowed: Record<Stage, readonly Stage[]> = {
  sourced: ["screened", "rejected"], screened: ["interview", "rejected"],
  interview: ["offer", "rejected"], offer: ["hired", "rejected"], hired: [], rejected: [],
};
function canMove(from: Stage, to: Stage) { return allowed[from].includes(to); }
```
Enforce the same rule on the server, and ideally with a database constraint or trigger too. The stage names here are illustrative; use your real six.

---

### TS-5. Utility types and mapped types 🟡

**Short answer:** Utility types transform existing types instead of duplicating them:
- `Partial`, `Required`, `Readonly`
- `Pick`, `Omit`, `Record`
- `Exclude`, `Extract`, `NonNullable`
- `ReturnType`, `Parameters`, `Awaited`

They're built with **mapped types** (`{ [K in keyof T]: ... }`) and conditional types.

```ts
type Candidate = { id: string; name: string; phone: string; city: string; createdAt: Date };

type CandidatePatch = Partial<Omit<Candidate, "id" | "createdAt">>;   // PATCH body
type PublicCandidate = Omit<Candidate, "phone">;                      // PII-free shape
type RolePerms = Record<"admin" | "hr" | "viewer", string[]>;

type Nullable<T> = { [K in keyof T]: T[K] | null };                   // custom mapped type
type Loaded = Awaited<ReturnType<typeof loadCandidate>>;              // derive, don't duplicate
```

**Explanation — how a mapped type works:**
`{ [K in keyof T]: T[K] | null }` means: "for every key K of T, create a property K whose type is T's type for K, or null." `Partial<T>` is the same thing with `?` added: `{ [K in keyof T]?: T[K] }`.

**Deep dive — types don't remove data (a real data-leak bug):**
```ts
function toPublic(c: Candidate): PublicCandidate {
  return c;   // ✅ compiles! Extra properties are allowed when assigning a variable
}
res.json(toPublic(candidate));   // ❌ phone number is sent to the client anyway
```
TypeScript's type says `phone` isn't there, but the object still has it. Fix: build the output object explicitly, or parse it through a Zod schema, which strips unknown keys:
```ts
const PublicCandidateSchema = z.object({ id: z.string(), name: z.string(), city: z.string() });
res.json(PublicCandidateSchema.parse(candidate));    // phone removed at runtime
```
Or select only the needed columns in the query (Prisma `select`), which is also faster.

**Production use:** PATCH DTOs, PII-safe response types, types derived from Prisma models and Zod schemas.

**Common mistakes:**
- Believing `Omit`/`Pick` change runtime objects.
- Duplicating types by hand, which then drift apart.

**Follow-up Q&A:**

**Q1. Write `DeepPartial<T>`.**
```ts
type DeepPartial<T> =
  T extends (...args: any[]) => any ? T :
  T extends readonly (infer U)[] ? readonly DeepPartial<U>[] :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;
```
Handle functions and arrays explicitly, then recurse into objects.

**Q2. `Omit` vs `Exclude`?**
- `Omit<T, K>` removes **properties** from an object type.
- `Exclude<U, M>` removes **members from a union**. Example: `Exclude<"a" | "b" | "c", "a">` is `"b" | "c"`.

**Q3. Why did the `toPublic` function compile?**
TypeScript's *excess property check* only applies to **object literals written directly** in an assignment or argument. A variable holding a wider object is allowed. That's structural typing: having *more* properties is compatible.

---

### TS-6. Narrowing, type guards, and `satisfies` 🟠

**Short answer:**
- **Narrowing:** TypeScript refines a union type based on control flow, using `typeof`, `instanceof`, `in`, equality checks, truthiness, and discriminants.
- **Custom type guards** (`x is T`) teach it new checks.
- **Assertion functions** (`asserts x is T`) narrow by throwing.
- **`satisfies`** validates a value against a type *without widening* the value's inferred type.

```ts
function isApiError(e: unknown): e is { status: number; message: string } {
  return typeof e === "object" && e !== null && "status" in e && "message" in e
    && typeof (e as any).status === "number";
}

function assertDefined<T>(v: T, msg: string): asserts v is NonNullable<T> {
  if (v == null) throw new Error(msg);
}
const user = await findUser(id);
assertDefined(user, "user not found");
user.name;   // narrowed to non-null after the assertion
```

**`satisfies` — why it exists:**
```ts
type RouteCfg = { path: string; roles: readonly string[] };

const A: Record<string, RouteCfg> = { dashboard: { path: "/d", roles: ["admin"] } };
type KA = keyof typeof A;  // string — the specific key names are lost

const B = { dashboard: { path: "/d", roles: ["admin"] } } satisfies Record<string, RouteCfg>;
type KB = keyof typeof B;  // "dashboard" — checked AND precise
```

**Explanation:**
- A type annotation says "treat this value as this type," which throws away detail.
- `satisfies` says "check this value against the type, but remember exactly what it is."
- Combining `as const satisfies X` gives readonly literal types *and* validation.

**Common mistakes:**
- Type guards that lie. TypeScript trusts your `is` claim completely and never verifies the function body.
- Using `as` where `satisfies` would catch real errors.

**Follow-up Q&A:**

**Q1. `as const` vs `satisfies`?**
- `as const` makes everything `readonly` with literal types (`"admin"` instead of `string`). It doesn't validate against any type.
- `satisfies` validates without widening.

They're often used together: `const ROLES = ["admin", "hr"] as const satisfies readonly Role[]`.

**Q2. Why does narrowing sometimes "reset" inside callbacks?**
```ts
let user: User | null = getUser();
if (user) {
  setTimeout(() => user.name);   // error: 'user' is possibly null — it's a `let` and could be reassigned
}
```
TypeScript can't prove a mutable variable wasn't changed before the callback runs. Copy it to a `const` first. (Newer TS versions preserve narrowing in more cases for `const` and unmodified variables.)

**Q3. What does the `in` operator narrow?**
`"data" in result` narrows a union to the members that declare a `data` property. It's useful when there's no discriminant field.

---

### TS-7. "TypeScript types don't exist at runtime." How do you protect API boundaries? 🟠 [RESUME]

**Short answer:** TypeScript types are erased at compile time; the emitted JavaScript has no types. So anything crossing a **trust boundary** can be any shape at runtime:
- HTTP request bodies, query params, headers
- Environment variables
- Third-party API responses
- Webhooks
- Database JSON columns
- **LLM output**

Validate it with a runtime schema (Zod), and **derive the TypeScript type from the schema** so validation and types can't drift apart.

```ts
import { z } from "zod";

const CreateJobSchema = z.object({
  title: z.string().trim().min(3).max(120),
  salaryMin: z.number().int().nonnegative(),
  salaryMax: z.number().int().nonnegative(),
}).refine((d) => d.salaryMax >= d.salaryMin, { message: "salaryMax < salaryMin", path: ["salaryMax"] });

type CreateJobInput = z.infer<typeof CreateJobSchema>;   // single source of truth

export async function createJob(raw: unknown, session: Session) {
  const parsed = CreateJobSchema.safeParse(raw);
  if (!parsed.success) return { ok: false as const, errors: parsed.error.flatten() };
  // dealerId comes from the session — never from the body
  return db.job.create({ data: { ...parsed.data, dealerId: session.dealerId } });
}

// Fail fast at boot instead of failing at 2 AM
const Env = z.object({
  DATABASE_URL: z.string().url(),
  CRON_SECRET: z.string().min(16),
  GEMINI_API_KEY: z.string().min(1),
});
export const env = Env.parse(process.env);
```

**Explanation:**
- *Trust boundary:* the point where data enters your system from something you don't control.
- Inside the boundary, after validation, types are reliable. Outside it, they're guesses.
- Deriving types with `z.infer` means changing the schema updates the type automatically.

**Deep dive — `parse` vs `safeParse`, and what Zod does to unknown keys:**
- `parse` throws a `ZodError`. Use it when invalid data means a bug, like env config at startup.
- `safeParse` returns `{ success, data | error }`. Use it for user input, where invalid data is normal and needs a 400/422 response.
- Zod object schemas **strip unknown keys by default**. Extra fields like `isAdmin: true` are silently dropped, which helps prevent mass-assignment attacks. `.strict()` rejects unknown keys instead; `.passthrough()` keeps them. (Exact method names vary slightly between Zod 3 and 4, so check the docs for your version.)

**Production use [RESUME]:**
- React Hook Form + Zod: the *same* schema validates on the client (UX) and the server (security).
- LLM output validation for your resume-extraction and matching pipelines (Vol 5).

**Common mistakes:**
- Client-only validation.
- `const body = (await req.json()) as CreateJobInput`, which is a lie to the compiler.
- Validating the shape but not business rules (min ≤ max).

**Follow-up Q&A:**

**Q1. Where exactly do you validate in a Next.js app?**
- Every Server Action (its arguments).
- Every Route Handler (body, query, params).
- Webhook payloads (after verifying the signature).
- Environment variables at boot.
- Responses from third-party APIs and LLMs before using them.

**Q2. Should you trust `dealerId` from the request body?**
Never. Anyone can change it. Tenant identity must come from the authenticated session, and the resource being accessed must be checked against it. Trusting a client-sent tenant ID is an IDOR (Insecure Direct Object Reference) vulnerability: change the ID, read another dealer's data.

**Q3. What is mass assignment?**
Passing the whole request body into a DB write: `db.user.update({ data: req.body })`. An attacker adds `role: "super_admin"` and it gets saved. Fix: whitelist fields with a schema, and never spread raw input into writes.

**Q4. What's the performance cost of validation?**
Small for normal payloads (microseconds to low milliseconds). It matters only for very large arrays in hot paths. Validate once at the boundary, not repeatedly inside.

---

### TS-8. [RESUME] Type-safe RBAC permissions 🔴

**Question:** "You built a 7-role RBAC system with dealer-namespaced permissions and fail-closed enforcement. How would you model it in TypeScript, and does typing make it secure?"

**Short answer:**
- Model permissions as a union generated from `as const` arrays or template literal types, and map roles to permissions with a `Record`. Typos then become compile errors.
- **Types don't make it secure.** Security comes from a server-side check that runs on every request and is **fail-closed**: an unknown role, a missing permission, a missing session, or any error all mean *deny*.
- Tenant isolation (which dealer's data) is a **separate, mandatory** condition on top of the role check.

```ts
const RESOURCES = ["candidate", "job", "billing", "user"] as const;
const ACTIONS = ["read", "write", "delete"] as const;
type Permission = `${(typeof RESOURCES)[number]}:${(typeof ACTIONS)[number]}`; // "candidate:read" | ...

type Role = "super_admin" | "dealer_owner" | "dealer_admin" | "hr" | "recruiter" | "viewer" | "support";

const ROLE_PERMISSIONS: Record<Role, readonly Permission[]> = {
  super_admin:  ["candidate:read", "candidate:write", "job:write", "billing:read", "user:write"],
  dealer_owner: ["candidate:read", "job:write", "billing:read", "user:write"],
  dealer_admin: ["candidate:read", "job:write", "user:write"],
  hr:           ["candidate:read", "candidate:write", "job:write"],
  recruiter:    ["candidate:read", "job:write"],
  viewer:       ["candidate:read"],
  support:      ["candidate:read", "billing:read"],
};

export function can(
  user: { role: string; dealerId: string } | null,
  perm: Permission,
  resourceDealerId: string,
): boolean {
  if (!user) return false;                                   // no session → deny
  const perms = ROLE_PERMISSIONS[user.role as Role];         // role from DB/JWT may be anything
  if (!perms) return false;                                  // unknown role → deny
  if (user.role !== "super_admin" && user.dealerId !== resourceDealerId) return false; // tenant isolation
  return perms.includes(perm);
}
```
⚠ The role names are **illustrative**. Use your actual 7 roles and explain *why* each exists.

**Explanation:**
- **Template literal types:** `` `${Resource}:${Action}` `` generates every combination automatically. Add a resource and all its permissions exist.
- **Fail-closed vs fail-open:**
  ```ts
  // FAIL-OPEN BUG: if perms is undefined (unknown role), this returns true
  if (perms && !perms.includes(perm)) return false;
  return true;
  ```
  Always structure checks so the *default* path is `return false`.
- **RBAC vs tenant scoping:** RBAC answers "can a recruiter write jobs?" Tenant scoping answers "can *this* recruiter write *this dealer's* jobs?" Missing the second check is the most common multi-tenant breach.

**Deep dive — where the check runs (defense in depth):**
1. **UI:** hide buttons. This is UX only, not security.
2. **Proxy:** coarse "is logged in" redirects (NEXT-5).
3. **Server Action / Route Handler:** `can(...)` on every mutation and read.
4. **Data layer:** every query includes `WHERE dealer_id = $sessionDealerId`.
5. **Database (optional):** Postgres Row-Level Security as a last line of defense (Vol 3).

**Production use [RESUME]:** Your 1,000+ dealer platform. Tenant isolation failure there means one dealership seeing another's candidates, a severe trust and legal issue (India's DPDP Act covers personal data).

**Common mistakes:**
- UI-only checks.
- Trusting the role from a client-editable place.
- Using TS `enum`s. (`as const` unions are preferred: zero runtime code, better with template literal types.)
- Scattering checks everywhere instead of centralizing them.

**Follow-up Q&A:**

**Q1. RBAC vs ABAC — when do roles stop being enough?**
- **RBAC:** permissions depend on the role.
- **ABAC:** permissions depend on attributes of the user, resource, and context. Example: "a recruiter can edit a job only if they created it and it's still in draft," or "only during business hours."

Signs you need ABAC: role explosion (`recruiter_ahmedabad_draft_only`) and conditions based on ownership or state. A common hybrid is RBAC for coarse permissions plus a few attribute checks in code. Libraries like CASL help.

**Q2. How do you test fail-closed behavior?**
Table-driven tests over every role × permission × same-tenant/other-tenant combination, plus edge cases: null user, unknown role string, a removed role, a malformed dealer ID. Assert that anything not explicitly allowed is denied.
```ts
for (const role of ALL_ROLES) for (const perm of ALL_PERMS) {
  test(`${role} ${perm} other-tenant is denied`, () => {
    if (role === "super_admin") return;
    expect(can({ role, dealerId: "A" }, perm, "B")).toBe(false);
  });
}
```

**Q3. How do you add a role without redeploying?**
Store role → permission mappings in the database and cache them briefly. Keep the *permission list* in code, since permissions correspond to code paths. Validate database role data against the code's permission union at load time, and deny unknown values.

**Q4. If you use JWTs, what happens when a user's role is downgraded?**
The JWT still carries the old role until it expires. Options:
- Short token lifetime plus refresh tokens.
- Look up the role server-side on each request (cached briefly).
- Maintain a token version per user in the database and reject older tokens.

(Details in Vol 2.)

**Q5. What is Postgres Row-Level Security and why add it?**
Database-enforced policies such as `USING (dealer_id = current_setting('app.dealer_id')::uuid)`, so even a buggy query without a `WHERE` clause can't cross tenants. It's defense in depth. The trade-offs are complexity, connection-pool session handling, and query debugging. Supabase uses RLS heavily. (Vol 3.)

---

# PART C — REACT (React 19.x)

### REACT-1. How does rendering and reconciliation work? Why do `key`s matter? 🟢

**Short answer:** When state changes, React works in two phases:
1. **Render phase:** React calls your component functions to produce a new element tree. This is pure computation, with no DOM changes.
2. **Commit phase:** React **reconciles** (diffs) the new tree against the previous one and applies only the necessary DOM changes. Then it runs effects.

`key`s identify list items across renders, so React can match each item to its previous self and keep its state and DOM node.

**Explanation:**
"Render" ≠ "update the DOM." A component can re-render (its function runs again) and produce identical output, in which case React touches nothing in the DOM. Re-renders cost JavaScript CPU time; DOM commits cost layout and paint.

**Diffing heuristics that make this fast (O(n) instead of O(n³)):**
- Different element type at the same position (`<div>` → `<span>`, or `ComponentA` → `ComponentB`) → React destroys the whole subtree and builds a new one. **All state below is lost.**
- Same type → React keeps the DOM node or component instance and updates changed props.
- Lists → matched by `key`, not by position.

**The index-key bug, concretely:**
```tsx
// rows = [Asha, Bhavin, Chirag], each row has an uncontrolled <input> note
{rows.map((r, i) => <CandidateRow key={i} row={r} />)}
```
1. The user types "Call back" into Asha's row (key 0).
2. The user deletes Asha. Rows are now [Bhavin, Chirag] with keys 0 and 1.
3. React sees that key 0 still exists and reuses its DOM input, **which still holds "Call back"**. That note now appears next to Bhavin.

Fix: `key={r.id}`.

**When are index keys fine?** When the list is static: never reordered, filtered, or inserted into, and items have no state.

**Deep dive — using `key` to reset state on purpose:**
```tsx
<CandidateEditor key={candidateId} candidateId={candidateId} />
```
Switching candidates changes the key, so React unmounts the old editor and mounts a fresh one with clean form state. No `useEffect` "reset" logic needed.

**Common mistakes:**
- `key={Math.random()}` or `key={Date.now()}`. Every render remounts every row, losing focus and state and killing performance.
- Keys that are unique only globally but not stable (e.g., generated on each fetch).

**Follow-up Q&A:**

**Q1. What triggers a re-render?**
1. Its own state changes (`setState` with a different value by `Object.is`).
2. Its parent re-renders (by default all children re-render, even with identical props).
3. A context it consumes changes value.

"Props changed" isn't a separate trigger; props only change *because* the parent re-rendered. `React.memo` is what makes a child skip re-rendering when its props are equal.

**Q2. Does calling `setState` with the same value re-render?**
React bails out if the new value is identical (`Object.is`). It may still run the component once more before bailing out in some cases, but it won't re-render children or commit.

**Q3. What's the Virtual DOM, and is it why React is fast?**
It's a lightweight in-memory description of the UI (React elements). It isn't inherently faster than hand-written DOM code. Its value is letting you write declarative UI while React computes minimal updates. Frameworks like Svelte and Solid skip the virtual DOM entirely.

**Q4. Must keys be globally unique?**
No. They must be unique among **siblings** in the same list.

---

### REACT-2. State updates: batching, functional updates, immutability 🟢

**Short answer:**
- `setState` **schedules** an update; the variable in your current render doesn't change.
- Since React 18, multiple updates are **automatically batched** into one re-render everywhere: event handlers, promises, timeouts, native events.
- When the next state depends on the previous one, use the **functional updater** (`setX(prev => …)`).
- State must be updated **immutably** (new references) so React can detect changes.

**Explanation — why `setN(n + 1)` three times adds only 1:**
Inside one render, `n` is a constant, a snapshot. All three calls compute `0 + 1` and queue "set to 1."
The functional form queues *functions*, which React applies in order: 0→1→2→3.

```tsx
function Counter() {
  const [n, setN] = useState(0);
  const wrong = () => { setN(n + 1); setN(n + 1); setN(n + 1); };          // → 1
  const right = () => { setN(p => p + 1); setN(p => p + 1); setN(p => p + 1); }; // → 3
  const log = () => { setN(n + 1); console.log(n); };                        // logs OLD n
  return <button onClick={right}>{n}</button>;
}
```

**Immutability:**
```tsx
// ❌ mutation: same array reference → React bails out → UI doesn't update
filters.cities.push("Surat"); setFilters(filters);

// ✅ new references along the changed path (structural sharing, see JS-7)
setFilters(prev => ({ ...prev, cities: [...prev.cities, "Surat"] }));

// ✅ updating one item in a list
setRows(prev => prev.map(r => (r.id === id ? { ...r, shortlisted: true } : r)));
// ✅ removing
setRows(prev => prev.filter(r => r.id !== id));
```

**Deep dive — `flushSync` (the rare escape hatch):** forces React to apply an update synchronously, e.g., to measure DOM right after a state change or to scroll to a just-added item. Use sparingly; it defeats batching.

**Common mistakes:**
- Reading state right after setting it and expecting the new value.
- Mutating nested objects.
- Storing derived values in state (e.g., `fullName`) instead of computing them during render.

**Follow-up Q&A:**

**Q1. Why does React require immutability?**
Change detection uses reference comparison (`Object.is`), which is O(1). Deep comparison would be expensive on every update. Immutable updates also enable time-travel debugging, safe concurrent rendering, and memoization.

**Q2. `useState` vs `useReducer` — when to switch?**
Switch to `useReducer` when:
- the next state depends on multiple fields,
- there are many related transitions (a multi-step form, a wizard), or
- you want the update logic testable outside the component.

A reducer is a pure function `(state, action) => newState`, which pairs well with discriminated-union actions (TS-4).

**Q3. What is derived state, and why avoid storing it?**
Anything computable from existing props or state (a filtered list, a total count). Storing it creates two sources of truth that can go out of sync. Compute it during render, and use `useMemo` only if it's expensive.

---

### REACT-3. `useEffect`: dependencies, cleanup, and race conditions 🟡 (asked everywhere)

**Short answer:**
- `useEffect` synchronizes a component with an **external system**: network, subscriptions, timers, browser APIs, non-React widgets.
- It runs **after** the commit (after paint in most cases).
- The dependency array must list every reactive value (props, state, values derived from them) the effect reads.
- The cleanup function runs **before the effect re-runs** and **on unmount**.
- For async work, cleanup must cancel or ignore stale results.

**Explanation — think "synchronize," not "lifecycle":**
Don't think "run on mount." Think: "keep the outside world in sync with these values. When they change, undo the old sync (cleanup) and do the new one."

**Race condition, step by step:**
1. The user opens candidate A; a fetch for A starts.
2. The user quickly opens candidate B; a fetch for B starts.
3. B's response arrives first → the screen shows B.
4. A's slower response arrives → **the screen shows A's data under B's name.**

Fix: cancel A's request when `id` changes.
```tsx
function CandidateProfile({ id }: { id: string }) {
  const [data, setData] = useState<Candidate | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    setData(null); setError(null);
    fetch(`/api/candidates/${id}`, { signal: controller.signal })
      .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then(setData)
      .catch(err => { if (err.name !== "AbortError") setError(err.message); });
    return () => controller.abort();          // id changed or unmounted → cancel the stale request
  }, [id]);

  if (error) return <ErrorState msg={error} />;
  return data ? <Profile c={data} /> : <Skeleton />;
}
```
An alternative for APIs without abort support: a local `let ignore = false;` flag, set to `true` in cleanup and checked before `setData`.

**Deep dive — stale closures:**
```tsx
const [count, setCount] = useState(0);
useEffect(() => {
  const t = setInterval(() => setCount(count + 1), 1000);  // `count` captured as 0 forever → stuck at 1
  return () => clearInterval(t);
}, []);                                                     // deps lie: effect uses `count`

// ✅ fix: functional update → no dependency on count
useEffect(() => {
  const t = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(t);
}, []);
```
Why: each render's effect captures that render's variables (JS-2). With `[]`, only the first render's effect exists, so it sees `count = 0` forever.

**Deep dive — StrictMode double-invocation:**
In development only, React 18+ mounts → unmounts → remounts each component to reveal missing cleanup. Seeing two requests in dev means *add cleanup*. It doesn't happen in production.

**Deep dive — "You might not need an effect":**

| Instead of an effect for… | Do this |
|---|---|
| Computing a filtered list from props/state | Compute during render (`useMemo` if expensive) |
| Resetting state when a prop changes | Change the component's `key` |
| Responding to a user click (POST request, analytics) | Do it in the event handler |
| Syncing one state variable from another | Derive it; remove the duplicate state |
| Fetching data in a large app | TanStack Query or a Server Component |

**Common mistakes:**
- Disabling `react-hooks/exhaustive-deps`.
- Objects or functions in deps that change every render, causing infinite loops. Define them inside the effect or memoize them.
- `async` directly as the effect function (it returns a Promise, not a cleanup). Define an async function inside instead.

**Follow-up Q&A:**

**Q1. `useEffect` vs `useLayoutEffect`?**
- `useLayoutEffect` runs **synchronously after DOM mutations but before the browser paints**. Use it to measure layout (element size, position) and adjust before the user sees anything, which avoids flicker (e.g., positioning a tooltip).
- It blocks painting, so keep it short.
- `useEffect` runs after paint and is right for everything else.

**Q2. Why can't the effect callback be `async`?**
An effect must return either nothing or a cleanup *function*. An `async` function always returns a Promise, which React would treat as an invalid cleanup.
```tsx
useEffect(() => { (async () => { /* await ... */ })(); }, [id]);
```

**Q3. An effect causes an infinite loop. What's happening?**
The effect sets state → re-render → a dependency has a *new reference* (object, array, or function created during render) → the effect runs again → sets state again… Fixes:
- Move the object inside the effect.
- Memoize it.
- Depend on primitive fields (`user.id` instead of `user`).
- Use a functional update.

**Q4. What's `useEffectEvent` (React 19.2)?**
It creates an "effect event": a function that always sees the latest props and state but **isn't a dependency**. Example: an effect connects to a chat room when `roomId` changes, and on connect it logs with the current `theme`. Wrapping the logging in `useEffectEvent` means changing `theme` doesn't reconnect.

---

### REACT-4. `useMemo`, `useCallback`, `React.memo` — and the React Compiler 🟡

**Short answer:**
- **`React.memo(Component)`** skips re-rendering if every prop is shallow-equal (`Object.is`) to the previous props.
- **`useMemo(fn, deps)`** caches a computed value between renders.
- **`useCallback(fn, deps)`** caches a function's identity; it's the same as `useMemo(() => fn, deps)`.
- They help only when (1) a computation is genuinely expensive, or (2) a stable reference is needed: props to a memoized child, effect dependencies, context values.
- The **React Compiler** (stable; supported in Next.js 16 via `reactCompiler: true`) inserts memoization automatically at build time, making most manual memoization unnecessary in new code.

**Explanation — why functions break memo:**
Every render creates new function and object instances. `() => {} !== () => {}`. So a memoized child that receives `onSelect={() => …}` sees a "changed" prop every time, and `React.memo` does nothing.

```tsx
const CandidateTable = React.memo(function CandidateTable(
  { rows, onSelect }: { rows: Row[]; onSelect: (id: string) => void },
) { /* renders 500 rows */ return null; });

function Page({ all }: { all: Row[] }) {
  const [query, setQuery] = useState("");
  const [theme, setTheme] = useState("light");
  const rows = useMemo(() => all.filter(r => r.name.toLowerCase().includes(query.toLowerCase())), [all, query]);
  const onSelect = useCallback((id: string) => router.push(`/c/${id}`), []);
  // toggling `theme` re-renders Page, but CandidateTable skips: rows & onSelect references unchanged
  return <CandidateTable rows={rows} onSelect={onSelect} />;
}
```

**Deep dive — the cost side:**
Memoization isn't free:
- Dependency comparison runs on every render.
- Cached values are held in memory.
- The code gets noisier.

For cheap computations (a small `.map`), the overhead can exceed the savings. Measure first.

**Deep dive — "move state down" beats memo:**
If typing in a search box re-renders a heavy sibling, often the best fix is structural. Move the search state into a smaller `<SearchBox>` component so the heavy component isn't in the re-rendering subtree at all. The same goes for passing heavy parts as `children`: they're created by the parent and aren't re-created when the child's state changes.

**Common mistakes:**
- Wrapping everything in `useMemo`/`useCallback`.
- Memoizing a child but passing new inline objects (`style={{…}}`, `options={[…]}`).
- Missing dependencies, which give stale values.

**Follow-up Q&A:**

**Q1. How do you prove a memoization helped?**
React DevTools Profiler: record the interaction before and after, then compare commit durations and "why did this render." Also look for long tasks in the Chrome Performance panel. No measurement means no claim.

**Q2. What does the React Compiler require from your code?**
Code that follows the **Rules of React**:
- Components and hooks are pure during render.
- Props and state are never mutated.
- Hooks are called unconditionally at the top level.

The compiler skips components it can't prove safe. `eslint-plugin-react-hooks` flags violations.

**Q3. `useMemo` for correctness vs performance?**
React treats `useMemo` as a performance hint and may discard cached values (in theory). Don't rely on it to guarantee one-time execution. For a truly stable, one-time value, use `useRef` or `useState(() => init)`.

---

### REACT-5. Controlled vs uncontrolled components — and large forms 🟡

**Short answer:**
- **Controlled:** the input's value lives in React state (`value` + `onChange`). React is the source of truth. Easy to validate and transform per keystroke, but each keystroke re-renders.
- **Uncontrolled:** the DOM keeps the value. You read it via a `ref` or `FormData` when needed. Fewer re-renders.
- React Hook Form uses uncontrolled inputs with subscriptions, so typing in one field doesn't re-render the whole form. That's why it scales to large forms.

```tsx
// Controlled
const [name, setName] = useState("");
<input value={name} onChange={e => setName(e.target.value)} />

// Uncontrolled + FormData (also works with React 19 form actions)
<form action={async (fd: FormData) => { await save(String(fd.get("name"))); }}>
  <input name="name" defaultValue="" />
</form>
```

**React Hook Form + Zod:**
```tsx
const schema = z.object({
  name: z.string().trim().min(2, "Name too short"),
  phone: z.string().regex(/^[6-9]\d{9}$/, "Enter a valid 10-digit mobile"),   // illustrative Indian mobile rule
});
type Form = z.infer<typeof schema>;

function CandidateForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<Form>({ resolver: zodResolver(schema) });
  return (
    <form onSubmit={handleSubmit(async (d) => { await saveCandidate(d); })}>
      <input {...register("name")} aria-invalid={!!errors.name} />
      {errors.name && <p role="alert">{errors.name.message}</p>}
      <input {...register("phone")} inputMode="numeric" autoComplete="tel" />
      {errors.phone && <p role="alert">{errors.phone.message}</p>}
      <button disabled={isSubmitting}>Save</button>
    </form>
  );
}
```

**Deep dive — the controlled/uncontrolled switch warning:**
`<input value={user?.name} />` starts as `undefined` (uncontrolled) and becomes a string (controlled), so React warns. Always pass a defined value: `value={user?.name ?? ""}`.

**Production use [RESUME]:** Your stack lists React Hook Form and Zod. Share the schema with the server action so validation rules exist in one place.

**Common mistakes:**
- Client-side validation only.
- Not disabling submit during the request.
- Controlled inputs for a 40-field form with heavy parents.

**Follow-up Q&A:**

**Q1. How do you prevent duplicate submissions *server-side*?**
Disabling the button handles only honest double-clicks. The server needs its own protection:
- A **unique constraint** on the natural key (e.g., `UNIQUE(dealer_id, phone)` for candidates), or
- An **idempotency key** generated when the form mounts and sent with the submission; the server stores it and ignores repeats (JS-12).

**Q2. When would you still choose controlled inputs?**
When you need to react to every keystroke: live formatting (masking a phone number), dependent fields, instant availability checks, or character counters.

**Q3. How do you show server-side validation errors in React Hook Form?**
Return field errors from the server and call `setError("phone", { message: "Already exists" })` for each, or with React 19 actions, return an error state from `useActionState` (REACT-7).

---

### REACT-6. Context API performance problem 🟡

**Short answer:**
- Context is a dependency-injection mechanism, not a state manager.
- When a Provider's `value` changes (by reference), **every** component using that context re-renders, even if it uses only a piece that didn't change.
- Fixes:
  - Memoize the value object.
  - Split contexts (state vs actions, or by domain).
  - For frequently changing shared state, use a store with **selectors** (Redux Toolkit `useSelector`, Zustand), so components subscribe only to the slice they use.

**Explanation:**
```tsx
<AuthCtx.Provider value={{ user, logout }}>   // ← new object on EVERY provider render
```
Even if `user` didn't change, `{...}` is a new reference → all consumers re-render. Context has no concept of "which field did you read."

```tsx
const AuthStateCtx = createContext<User | null>(null);
const AuthActionsCtx = createContext<{ logout(): void } | null>(null);

function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const actions = useMemo(() => ({ logout: () => setUser(null) }), []); // stable forever
  return (
    <AuthActionsCtx.Provider value={actions}>
      <AuthStateCtx.Provider value={user}>{children}</AuthStateCtx.Provider>
    </AuthActionsCtx.Provider>
  );
}
// A LogoutButton reading only AuthActionsCtx never re-renders when `user` changes.
```

**Deep dive — how selector-based stores avoid this:**
Zustand and Redux use `useSyncExternalStore`. Each component provides a selector (`state => state.credits.balance`). On any store change, the store runs every subscriber's selector and compares the *result* with its previous result (`Object.is`). Only components whose selected slice changed re-render.
```ts
const balance = useStore(s => s.credits.balance);   // re-renders only when balance changes
```

**Good uses of context:** theme, locale (next-intl), the current user, feature flags, dependency injection of clients. These are values that change rarely.

**Common mistakes:**
- One giant `AppContext` holding everything.
- Putting fast-changing values (input text, mouse position, timers) in context.

**Follow-up Q&A:**

**Q1. How do `useSelector` and Zustand avoid the context problem?**
They keep state *outside* React and subscribe components via `useSyncExternalStore` with selectors. Only a change in the *selected* value causes a re-render. React-Redux also passes the store through context, but the store object itself never changes; updates flow via subscriptions.

**Q2. Does wrapping consumers in `React.memo` help?**
No. Context changes bypass `memo`. A memoized component that calls `useContext` still re-renders when that context's value changes. `memo` only helps for children that *don't* consume the context.

**Q3. Where do you keep server data, then?**
Not in context and not in Redux. Use TanStack Query or Server Components (REACT-9).

---

### REACT-7. React 19: Actions, `useActionState`, `useOptimistic`, `use` 🟠 (current version, likely asked)

**Short answer:** React 19 made async mutations first-class. An **Action** is an async function run inside a transition, so React tracks its pending state, errors, and optimistic updates automatically.
- **`useActionState(action, initialState)`** returns `[state, formAction, isPending]`. The action receives `(previousState, payload)` and returns the new state.
- **`useOptimistic(value, reducer?)`** shows a temporary value while an action is in progress. When the action finishes, the optimistic value is discarded and the real `value` is shown.
- **`useFormStatus()`** (from `react-dom`) lets a child, like a submit button, read the pending state of its parent `<form>`.
- **`use(promiseOrContext)`** reads a promise (suspending until it resolves) or a context. Unlike hooks, it *can* be called inside conditions and loops.
- **`ref` is a regular prop** on function components; `forwardRef` is no longer needed.
- `<form action={fn}>` accepts a function and resets uncontrolled fields after a successful submit.

```tsx
"use client";
import { useActionState, useOptimistic } from "react";
import { toggleShortlist } from "./actions";   // Server Action (see NEXT-4)

type State = { error?: string };

export function ShortlistButton({ candidateId, shortlisted }: { candidateId: string; shortlisted: boolean }) {
  const [optimisticOn, setOptimisticOn] = useOptimistic(shortlisted);
  const [state, formAction, isPending] = useActionState<State, FormData>(async () => {
    setOptimisticOn(!optimisticOn);                    // instant feedback
    const res = await toggleShortlist(candidateId);    // server action also refreshes data (updateTag)
    return res.ok ? {} : { error: "Could not update. Try again." };
  }, {});

  return (
    <form action={formAction}>
      <button disabled={isPending} aria-pressed={optimisticOn}>
        {optimisticOn ? "★ Shortlisted" : "☆ Shortlist"}
      </button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

**Explanation — v1 correction on how `useOptimistic` actually settles:**
The optimistic value **always** disappears when the action ends. What shows afterward is whatever `shortlisted` prop the component has *then*.
- **On success:** the Server Action must make the new server data flow back, e.g., by calling `updateTag(...)`/`refresh()` or `revalidatePath(...)` so the Server Component re-renders with `shortlisted={true}`. Without that, the button flips back even though the database changed.
- **On failure:** the prop is unchanged, so the UI reverts automatically. That's the "free rollback."

**Deep dive — `use()` with Suspense:**
```tsx
// Server Component starts the fetch (doesn't await), passes the promise down
export default function Page() {
  const statsPromise = getDealerStats();
  return <Suspense fallback={<Skel />}><Stats statsPromise={statsPromise} /></Suspense>;
}
// Client Component
"use client";
function Stats({ statsPromise }: { statsPromise: Promise<Stats> }) {
  const stats = use(statsPromise);    // suspends until resolved
  return <div>{stats.total}</div>;
}
```

**Production use:** Shortlist toggles, approve/reject buttons for AI-generated outputs (your human-approval step **[RESUME]**), status changes, inline edits.

**Common mistakes:**
- Optimistic UI for operations likely to fail, or ones that involve money. For credit charges and payments, show a pending state instead.
- Forgetting to refresh server data, so the UI reverts on success.
- Creating a new promise inside a Client Component render and passing it to `use()`. A new promise every render suspends forever. Promises should come from a Server Component or a cache.

**Follow-up Q&A:**

**Q1. When would you *not* use optimistic updates?**
- Payments and credit deductions.
- Operations with a high failure rate (strict server validation, conflicts).
- Irreversible actions (sending an SMS to a candidate).
- When the server computes the result (e.g., a generated ID or price the client can't predict).

**Q2. What are React 19.2's `<Activity>` and `useEffectEvent`?**
- `<Activity mode="hidden">` hides a subtree while **preserving its state and DOM** and deferring its updates. Examples: keep a tab's scroll position and form input when switching tabs, or pre-render a likely next screen.
- `useEffectEvent`: see REACT-3 Q4.

**Q3. How does `useFormStatus` work?**
It must be called from a component rendered *inside* a `<form>`. It returns `{ pending, data, method, action }` for that parent form, so a reusable `<SubmitButton>` can show a spinner without prop drilling.

**Q4. Does `useActionState` work without JavaScript?**
With Server Actions and a `permalink` argument, forms can submit before hydration (progressive enhancement). Without JS, the form posts natively and the page re-renders with the returned state.

---

### REACT-8. Concurrent rendering: `useTransition`, `useDeferredValue`, Suspense 🟠

**Short answer:** Concurrent rendering lets React **interrupt** a render in progress to handle something more urgent, then resume or restart.
- **`useTransition`** returns `[isPending, startTransition]`. Updates inside `startTransition` are low-priority and interruptible, so typing and clicking stay responsive during heavy re-renders.
- **`useDeferredValue(value)`** returns a copy of the value that "lags behind" during heavy updates. It's the same idea, for when you don't own the state update.
- **Suspense** shows a fallback while children are waiting (lazy-loaded code, data via `use()`, streamed server content).

**Explanation:**
Before concurrency, a render was all-or-nothing. Typing into a box that filters 10,000 rows blocked every keystroke until the whole list re-rendered.
With a deferred value, React first renders the urgent update (the input text) and immediately paints it. Then it renders the list with the new query in the background. If the user types again before that finishes, React throws away the stale work and starts over with the newest value.

```tsx
function SearchPage({ rows }: { rows: Row[] }) {
  const [text, setText] = useState("");
  const deferred = useDeferredValue(text);
  const filtered = useMemo(() => heavyFilter(rows, deferred), [rows, deferred]);
  const isStale = text !== deferred;
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <MemoTable rows={filtered} />        {/* must be memoized, or it re-renders with every keystroke anyway */}
      </div>
    </>
  );
}

// useTransition for tab switching
const [isPending, startTransition] = useTransition();
<button onClick={() => startTransition(() => setTab("analytics"))}>Analytics {isPending && "…"}</button>
```

**Common mistakes:**
- Thinking transitions make work *faster*. Total work is the same; only the responsiveness improves.
- Using `useDeferredValue` to reduce **network** calls. It doesn't debounce requests.
- Forgetting to memoize the heavy child, which removes the benefit.

**Follow-up Q&A:**

**Q1. `useDeferredValue` vs debounce — which for server search?**
- **Server search** (network cost, rate limits): debounce the query and abort stale requests. Debounce reduces the number of requests.
- **Client-side filtering or heavy rendering** (CPU cost): `useDeferredValue`. It adapts automatically to device speed, with no fixed delay.

Both can be combined.

**Q2. Can you put an async function inside `startTransition`?**
In React 19, yes. Async transitions are what Actions are. `isPending` stays true until the async work completes. Note that state updates after an `await` inside the transition may need their own `startTransition` wrap, depending on the version; check the docs.

**Q3. What does Suspense do when a transition triggers a suspend?**
During a transition, React keeps showing the **old** UI instead of flipping back to the fallback. That avoids jarring spinner flashes when navigating. Outside a transition, the nearest Suspense fallback is shown.

---

### REACT-9. Server state vs client state: TanStack Query vs Redux 🟠 [RESUME]

**Short answer:**
- **Server state** is data owned by the backend that the UI keeps a *cached copy* of: candidates, jobs, credit balances. It can go stale, needs refetching, deduplication, retries, pagination, and invalidation after mutations. **TanStack Query** is built for exactly this.
- **Client state** is owned by the UI: modal open/closed, selected rows, the draft of a multi-step form, theme. `useState`, `useReducer`, Redux Toolkit, or Zustand.
- Putting server data into Redux means hand-writing loading flags, caching, invalidation, and refetch logic, which TanStack Query already does correctly.

```tsx
const candidateKeys = {
  all: ["candidates"] as const,
  lists: () => [...candidateKeys.all, "list"] as const,
  list: (f: Filters) => [...candidateKeys.lists(), f] as const,
  detail: (id: string) => [...candidateKeys.all, "detail", id] as const,
};

const { data, isPending, isError, error } = useQuery({
  queryKey: candidateKeys.list(filters),
  queryFn: ({ signal }) => fetchCandidates(filters, signal),   // signal: auto-abort when key changes/unmount
  staleTime: 30_000,
});

const qc = useQueryClient();
const updateMutation = useMutation({
  mutationFn: updateCandidate,
  onSuccess: (_data, vars) => {
    qc.invalidateQueries({ queryKey: candidateKeys.detail(vars.id) });
    qc.invalidateQueries({ queryKey: candidateKeys.lists() });   // prefix match: all list variants
  },
});
```

**Explanation — the key vocabulary:**

| Term | Meaning |
|---|---|
| Query key | The cache ID. Any variable affecting the result **must** be in it. |
| `staleTime` | How long data counts as fresh. Default 0: immediately stale, so refetch on mount/focus. |
| `gcTime` | How long *unused* cache entries stay in memory. Default 5 min. Named `cacheTime` before v5. |
| Invalidation | Mark matching queries stale and refetch the active ones. |
| Deduplication | Two components requesting the same key share one network request. |

**Deep dive — optimistic updates with TanStack Query:**
```ts
useMutation({
  mutationFn: toggleShortlist,
  onMutate: async (id) => {
    await qc.cancelQueries({ queryKey: candidateKeys.detail(id) });
    const prev = qc.getQueryData<Candidate>(candidateKeys.detail(id));
    qc.setQueryData<Candidate>(candidateKeys.detail(id), (c) => c && { ...c, shortlisted: !c.shortlisted });
    return { prev };                                  // snapshot for rollback
  },
  onError: (_e, id, ctx) => qc.setQueryData(candidateKeys.detail(id), ctx?.prev),   // rollback
  onSettled: (_d, _e, id) => qc.invalidateQueries({ queryKey: candidateKeys.detail(id) }), // resync
});
```

**Common mistakes:**
- Keeping `staleTime: 0` everywhere, which causes refetch storms on window focus.
- Unstructured keys like `["data"]`.
- Copying query data into `useState`, which creates two sources of truth.

**Follow-up Q&A:**

**Q1. You list both Redux Toolkit and TanStack Query. What did each own?** **[RESUME]**
Prepare a *true* answer. A strong typical split: TanStack Query owned all API data (lists, details, mutations, invalidation), and Redux Toolkit owned cross-page UI/client state (auth session info in the SPA, multi-step wizard drafts, global UI preferences). If at N2N you used RTK Query instead, say so. It's Redux Toolkit's own server-cache tool with the same concepts.

**Q2. How do you implement pagination and infinite scroll?**
- Page-based: put the page number in the query key, and use `placeholderData: keepPreviousData` so the old page stays visible while the next loads.
- Infinite scroll: `useInfiniteQuery` with `getNextPageParam` from a **cursor** (e.g., the last item's `created_at` + `id`). Cursor pagination beats offset for large, changing tables (Vol 3).

**Q3. Server Components exist now. Do you still need TanStack Query?**
Less often. Server Components fetch initial data on the server with no client cache needed. TanStack Query remains valuable for highly interactive client-side data: infinite lists, polling, optimistic mutations, live filters, and data shared across many client components. Many apps use both: server-render the first page, then TanStack Query takes over on the client.

**Q4. What happens to in-flight requests when the query key changes?**
If your `queryFn` uses the provided `signal`, TanStack Query aborts the obsolete request. The cache is keyed, so an old response can never overwrite the new key's data anyway.

---

### REACT-10. Error boundaries 🟠

**Short answer:**
- An error boundary catches errors thrown **during rendering, in lifecycle methods, and in constructors** of its child tree, and renders a fallback UI instead of unmounting the whole app.
- They're still implemented as class components (`static getDerivedStateFromError` + `componentDidCatch`); most teams use the `react-error-boundary` library.
- They do **not** catch errors in: event handlers, async code (`setTimeout`, promises after `await`), server-side rendering, or the boundary itself.

```tsx
import { ErrorBoundary } from "react-error-boundary";

<ErrorBoundary
  fallbackRender={({ error, resetErrorBoundary }) => (
    <div role="alert">
      Credits widget failed to load. <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  )}
  onError={(error, info) => reportToMonitoring(error, info.componentStack)}
  resetKeys={[dealerId]}            // auto-reset when the dealer changes
>
  <CreditsWidget />
</ErrorBoundary>
```

**Explanation:**
Without a boundary, an uncaught render error **unmounts the entire React root**, leaving a blank white screen. React deliberately removes corrupted UI rather than show wrong data. Boundaries let you contain the blast radius.

**Deep dive — granularity strategy:**
- **Root boundary:** "Something went wrong, reload" as the last resort.
- **Route/page boundary:** Next.js `error.tsx` per route segment.
- **Widget boundary:** each independent dashboard card, so one broken chart doesn't take down the page.

**Deep dive — getting event-handler and async errors into a boundary:**
```tsx
const { showBoundary } = useErrorBoundary();   // from react-error-boundary
async function onExport() {
  try { await exportCsv(); }
  catch (e) { showBoundary(e); }               // or set local error state for inline messages
}
```
React 19 also adds root-level `onCaughtError` / `onUncaughtError` / `onRecoverableError` options in `createRoot` for centralized reporting.

**Common mistakes:**
- One boundary at the root only.
- Swallowing errors without reporting to monitoring (Sentry etc.).
- Expecting a boundary to catch a failed `fetch` in a click handler.

**Follow-up Q&A:**

**Q1. How do you catch errors from event handlers?**
Use `try/catch` inside the handler. Then either show an inline error via state or rethrow into the boundary with `showBoundary`. Report to monitoring either way.

**Q2. What about unhandled promise rejections?**
- Browser: a global `window.addEventListener("unhandledrejection", ...)` reports them.
- Node: `process.on("unhandledRejection", ...)`. Since Node 15 the default is to crash the process, which is usually correct. Log, then let the orchestrator restart it.

**Q3. How does `error.tsx` work in Next.js?**
It's a Client Component that wraps a route segment in an error boundary. It receives `error` and `reset` props. `global-error.tsx` covers the root layout. In production, server error messages are sanitized and a `digest` is provided that matches server logs.

---

### REACT-11. "A recruiter says the candidate list page is slow." Walk me through it. 🔴 [RESUME]

**Short answer:** The *process* is what's being graded: clarify → measure → identify the bottleneck → fix the cause → verify.

**Step 1 — Clarify the symptom.** "Slow" means different things:
- Slow to **load** (first view)?
- Slow when **typing** in the search box?
- Janky when **scrolling**?
- Slow **after applying filters**?
- For everyone, or only some dealers (with more data), or on low-end Android devices?

**Step 2 — Measure before changing anything:**
- **React DevTools Profiler:** which components render, how long they take, and why ("props changed," "parent rendered," "hooks changed").
- **Chrome Performance panel:** long tasks (> 50 ms), scripting vs layout vs paint time.
- **Network tab:** request waterfalls, payload sizes, time to first byte (TTFB).
- **Real-user monitoring:** Core Web Vitals (LCP for load, INP for interaction, CLS for layout shift), segmented by device.

**Step 3 — Fix by cause:**

| Symptom | Likely cause | Fix |
|---|---|---|
| Scroll jank, 1,000+ rows | Too many DOM nodes | Virtualization (TanStack Virtual) |
| Typing lags | Whole table re-renders per keystroke | Move state down, memoize table, `useDeferredValue`, debounce the query |
| Slow first load | Large JS bundle, sequential requests | Code-split (`next/dynamic`, `React.lazy`), server-render the first page, parallelize fetches |
| Slow after filter | API/DB | Check server timing: Typesense latency, missing Postgres index, N+1 queries |
| Large payload | Fetching all fields for all rows | Paginate, select only needed columns, compress |

**Step 4 — Verify:** re-profile, compare before/after numbers, and watch production metrics after release.

**Virtualization example:**
```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ rows }: { rows: Row[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const v = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,        // estimated row height in px
    overscan: 8,                   // extra rows above/below viewport for smooth scrolling
  });
  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: v.getTotalSize(), position: "relative" }}>
        {v.getVirtualItems().map((item) => (
          <div key={rows[item.index].id}
               style={{ position: "absolute", top: 0, left: 0, width: "100%", transform: `translateY(${item.start}px)` }}>
            <CandidateRow row={rows[item.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Explanation — how virtualization works:** Only the rows visible in the scroll viewport (plus a few extra) are in the DOM. A tall spacer div keeps the scrollbar correct. As you scroll, the same small set of DOM nodes is reused for different data. So 10,000 rows cost about as much as 20.

**Common mistakes:**
- Jumping to "add `useMemo`" without profiling.
- Blaming React for a 2-second API call.
- Optimizing on a fast MacBook while recruiters use budget Android phones. Use CPU throttling in DevTools.

**Follow-up Q&A:**

**Q1. What is INP?**
Interaction to Next Paint: a Core Web Vital measuring responsiveness. It looks at the latency from user interactions (click, tap, key press) until the next frame is painted, reporting roughly the worst interactions. Good is ≤ 200 ms. It replaced First Input Delay (FID) as a Core Web Vital in March 2024. Long tasks on the main thread are the usual cause.

**Q2. What are the trade-offs of virtualization?**
- Browser Ctrl+F can't find rows that aren't rendered.
- Accessibility needs care: screen readers see only the rendered rows (use proper ARIA row counts).
- Variable row heights need measurement.
- Print and "select all" behave differently.

Alternatives: pagination, or CSS `content-visibility: auto` for moderate lists.

**Q3. The Profiler shows `CandidateRow` re-rendering 500 times per keystroke. Fix?**
1. Move the search input state into its own component, or
2. Wrap `CandidateRow` in `React.memo` and ensure its props are stable (pass the row object and a memoized callback, not inline functions), and
3. Defer the filtered list with `useDeferredValue`.

Then re-profile to confirm.

**Q4. How do you find what's in a large JS bundle?**
Use a bundle analyzer (Next.js supports `@next/bundle-analyzer`). Look for heavy libraries (full lodash, moment, charting libs) loaded on pages that don't need them. Fix with dynamic imports, lighter alternatives, or moving work to Server Components so it never ships to the client.

---

### REACT-12. [CODING] `useCandidateSearch` hook: debounce, cancel, keep previous results 🔴

**Requirements:**
1. Debounce input by 300 ms.
2. Cancel stale requests.
3. Keep showing previous results while new ones load (no flicker).
4. Don't query for fewer than 2 characters.
5. Filters must be part of the cache identity.

```tsx
import { useEffect, useState } from "react";
import { useQuery, keepPreviousData } from "@tanstack/react-query";

class HttpError extends Error {
  constructor(public status: number) { super(`HTTP ${status}`); }
}

export function useDebouncedValue<T>(value: T, ms = 300): T {
  const [v, setV] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setV(value), ms);
    return () => clearTimeout(t);            // new keystroke → cancel pending update = debounce
  }, [value, ms]);
  return v;
}

export function useCandidateSearch(term: string, filters: Filters) {
  const debounced = useDebouncedValue(term.trim(), 300);
  return useQuery({
    queryKey: ["candidate-search", debounced, filters],
    queryFn: async ({ signal }) => {
      const qs = new URLSearchParams({ q: debounced, ...toParams(filters) });
      const res = await fetch(`/api/search?${qs}`, { signal });   // aborted when key changes
      if (!res.ok) throw new HttpError(res.status);
      return (await res.json()) as SearchResponse;
    },
    enabled: debounced.length >= 2,
    placeholderData: keepPreviousData,        // show old results while new ones load
    staleTime: 15_000,                        // re-typing the same query within 15s → instant from cache
    retry: (count, err) => count < 2 && !(err instanceof HttpError && err.status < 500), // never retry 4xx
  });
}

// usage
function SearchBox() {
  const [term, setTerm] = useState("");
  const { data, isFetching, isError } = useCandidateSearch(term, { city: "Ahmedabad" });
  return (
    <>
      <input value={term} onChange={(e) => setTerm(e.target.value)} placeholder="Search candidates" />
      {isFetching && <Spinner />}
      {isError && <p role="alert">Search unavailable. Try again.</p>}
      <Results items={data?.hits ?? []} />
    </>
  );
}
```
The typed `HttpError` lets `retry` distinguish client errors (4xx, never retried) from transient server/network failures (retried up to 2 times).

**Explanation — how each requirement is met:**

| Requirement | Mechanism |
|---|---|
| Debounce | `useDebouncedValue`: the effect cleanup cancels the timer on each keystroke |
| Cancel stale | TanStack Query aborts `signal` when the key changes |
| No flicker | `keepPreviousData` |
| Min length | `enabled` flag |
| Filters in identity | Filters in `queryKey`, so each filter combination has its own cache entry |
| No out-of-order bugs | Cache is per key; an old response can't overwrite the new key |

**Complexity:** One request per settled input instead of one per keystroke. Repeated queries are served from cache.

**Edge cases:**
- Whitespace-only input (trimmed).
- Gujarati/Hindi names (`URLSearchParams` encodes them).
- The user clears the input: `enabled` becomes false and `data` becomes undefined, so show an empty state rather than stale hits.
- 429 from the server: show a message; don't retry aggressively.
- Very fast typers on slow networks.

**Follow-up Q&A:**

**Q1. Why put `filters` in the query key?**
The key is the cache identity. Without filters in it, changing "Ahmedabad" → "Surat" would return cached Ahmedabad results for the same text. Rule: *every input that changes the result belongs in the key.*

**Q2. `filters` is an object. Won't a new object each render break caching?**
No. TanStack Query hashes keys **deterministically by value** (stable JSON-like hashing with sorted object keys), not by reference. `{city:"A"}` created twice matches the same entry.

**Q3. How would you add search analytics without firing on every keystroke?**
Track only the *debounced* term, and only when results arrive (`useEffect` on `data` + `debounced`). Throttle the tracking call to be safe, and never send PII such as phone numbers in analytics events.

**Q4. How would the backend protect itself from this endpoint?**
- Rate limit per user.
- Cap page size.
- Validate the query length.
- Use Typesense's own limits.
- Cache popular queries briefly.
- Scope every search to the user's dealer (tenant isolation applies to search too). **[RESUME]**

---

# PART D — NEXT.JS (App Router, Next.js 16.x)

**What changed in Next.js 16 (interviewers test whether you're current):**

| Area | Before (≤15) | Next.js 16 |
|---|---|---|
| Bundler | Webpack default | **Turbopack default** |
| Request interception | `middleware.ts` | **`proxy.ts`** (Node.js runtime); middleware filename deprecated |
| Caching | Implicit and confusing defaults | **Opt-in** via Cache Components (`cacheComponents: true`) + `"use cache"` |
| Request APIs | Sync in 14; async with compat warnings in 15 | `params`, `searchParams`, `cookies()`, `headers()` **must be awaited** |
| Tag invalidation | `revalidateTag(tag)` | `revalidateTag(tag, profile)`; new `updateTag`, `refresh` |
| Tooling | `next lint` | `next lint` removed; run ESLint/Biome directly |
| Runtime | Node 18+ | **Node 20.9+** |
| React Compiler | Experimental | Stable support (`reactCompiler: true`) |

**After 16.0:**
- 16.2 (March 2026) added a stable Adapter API for deploying on other platforms.
- 16.3 (August 2026) is the current minor, focused on instant navigations.
- Next.js moved to scheduled monthly security releases in July 2026. Keep patch versions current.

---

### NEXT-1. Server Components vs Client Components 🟢

**Short answer:**
- **Server Components** (the default in `app/`) run **only on the server**. Their code is never sent to the browser. They can be `async`, query the database, and read secrets directly. They can't use state, effects, event handlers, or browser APIs.
- **Client Components** start with `"use client"`. They're **pre-rendered to HTML on the server** for the first load, then **hydrated** in the browser, where their JS makes them interactive.
- `"use client"` marks a **boundary**: that file and everything it imports become part of the client bundle.
- Props passed from Server → Client Components must be **serializable**: plain objects, arrays, strings, numbers, Dates, Maps/Sets, Promises, JSX. Functions can't be passed, except Server Actions.

**Explanation — the "two worlds" model:**
Picture a restaurant:
- The kitchen (server) prepares most of the meal: data fetching and HTML.
- Only the parts the customer interacts with at the table (client) need a waiter present: buttons, forms, dropdowns.

Server Components keep heavy dependencies (DB clients, markdown parsers, date libraries) out of the browser bundle entirely.

**Composition pattern — Server Components inside Client Components via `children`:**
```tsx
// app/dashboard/page.tsx — Server Component
import { Collapsible } from "./collapsible";   // "use client"

export default async function Dashboard() {
  const dealerId = await currentDealerId();
  const stats = await db.stats.forDealer(dealerId);    // direct DB access, no API round trip
  return (
    <Collapsible title="Credits">
      <CreditsSummary stats={stats} />   {/* still a Server Component: rendered on server, zero client JS */}
    </Collapsible>
  );
}

// app/dashboard/collapsible.tsx
"use client";
export function Collapsible({ title, children }: { title: string; children: React.ReactNode }) {
  const [open, setOpen] = useState(true);
  return <section><button onClick={() => setOpen(o => !o)}>{title}</button>{open && children}</section>;
}
```
Why this works: the server renders `<CreditsSummary>` into a serialized result *before* passing it as `children`. The Client Component just places it; it never imports its code.

**Deep dive — preventing secret leaks with `server-only`:**
```ts
// lib/db.ts
import "server-only";          // build error if any Client Component imports this file
export const db = new PrismaClient();
```
Also: only env vars prefixed `NEXT_PUBLIC_` are exposed to the browser. Never prefix secrets.

**Deep dive — what the "RSC payload" is:**
When navigating client-side, the server sends a compact serialized description of the Server Component tree (the "Flight" format), not HTML. React merges it into the existing page without a full reload, preserving client state. (This protocol was the attack surface in React2Shell; see NEXT-4.)

**Common mistakes:**
- `"use client"` at the top of a page "to make it work," which ships the whole page as JS.
- Importing server code (DB, secrets) into client files.
- Passing a function prop from a Server to a Client Component.

**Follow-up Q&A:**

**Q1. Are Client Components rendered on the server?**
Yes. On the initial page load they're pre-rendered to HTML on the server (for fast first paint and SEO), then hydrated in the browser. "Client Component" means "also runs in the browser," not "only runs in the browser." So browser APIs (`window`) must only be accessed in effects or event handlers.

**Q2. Can a Client Component import a Server Component?**
No. Once you're inside the client boundary, imported components become client code. The way to nest server UI inside client UI is to pass it as `children` or other JSX props from a Server Component parent.

**Q3. Where should `"use client"` go?**
As deep in the tree as possible: on the smallest interactive leaf, like a `LikeButton` or a `FilterDropdown`, not on the layout or page. This keeps the client bundle small.

**Q4. How do Server Components improve security?**
Data access and secrets stay on the server, and only rendered results are sent. But every value you pass as a prop to a Client Component **is sent to the browser**. Passing a full user record (with password hash or phone number) to a Client Component leaks it. Pass only the fields needed.

---

### NEXT-2. Rendering strategies: static, dynamic, streaming, ISR, PPR 🟡

**Short answer:**

| Strategy | When HTML is produced | Good for |
|---|---|---|
| Static (prerender) | At build time | Marketing, docs |
| Time-based revalidation (ISR) | At build, regenerated in the background after N seconds | Blog, public job listings |
| On-demand revalidation | Regenerated when you invalidate a tag or path | CMS content after publish |
| Dynamic | On every request | Per-user dashboards |
| Streaming | Shell first; slow parts streamed as ready (Suspense) | Pages with slow widgets |
| **Partial Prerendering** | Static shell served instantly + dynamic "holes" streamed, in one response | Most real pages |

**Next.js 16 model (with Cache Components):**
- Everything is **dynamic by default**.
- You opt into caching with `"use cache"` on data functions or components.
- Dynamic parts must be wrapped in `<Suspense>`.
- Next.js builds a **static shell** from everything cacheable and streams the dynamic parts into it. Partial Prerendering is now part of the Cache Components model rather than a separate flag.

```tsx
export default function JobPage({ params }: { params: Promise<{ id: string }> }) {
  return (
    <>
      <SiteHeader />                                   {/* static */}
      <Suspense fallback={<JobSkeleton />}>
        <JobDescription params={params} />             {/* "use cache" inside → cached per job id */}
      </Suspense>
      <Suspense fallback={<ApplySkeleton />}>
        <ApplyPanel />                                 {/* reads cookies() → dynamic, streamed per request */}
      </Suspense>
    </>
  );
}
```

**Explanation — what streaming means physically:**
The server sends the HTTP response in chunks. The first chunk contains the shell and the fallbacks. As each Suspense boundary's data resolves, the server sends another chunk containing that content plus a tiny script that swaps it into place. The user sees useful content early instead of staring at a blank page until the slowest query finishes.

**Common mistakes:**
- Assuming the Next.js 13/14 caching behavior (`fetch` cached by default).
- One Suspense boundary around the whole page.
- Reading `cookies()` in a layout, which makes everything under it dynamic.

**Follow-up Q&A:**

**Q1. What makes a route or component dynamic?**
- Accessing request-time data: `cookies()`, `headers()`, `searchParams`, `connection()`.
- Uncached data fetching.
- Using non-deterministic values like `Date.now()` or `Math.random()` during render (Next.js 16 asks you to be explicit about these).

With Cache Components enabled, Next.js errors in development if uncached or request-time data is accessed **outside a Suspense boundary**. This forces you to decide: cache it, or stream it.

**Q2. How do you debug why a page isn't static?**
- Read the `next build` output, which marks each route as static/partially prerendered/dynamic.
- Read the dev-time errors pointing at the dynamic access.
- Search for `cookies()`/`headers()` calls in layouts and shared components.

**Q3. SSR vs SSG vs CSR in one line each?**
- **SSR:** HTML generated per request on the server.
- **SSG:** HTML generated once at build time.
- **CSR:** the server sends an empty shell and JavaScript builds the page in the browser. That's worse for first paint and SEO, and fine for authenticated app screens behind a login where SEO doesn't matter.

---

### NEXT-3. Caching in Next.js 16: `"use cache"`, `cacheLife`, `cacheTag`, `revalidateTag`, `updateTag` 🟠

**Short answer:**
1. Enable with `cacheComponents: true` in `next.config.ts`.
2. Add `"use cache"` at the top of an async function or component to cache its result.
3. `cacheLife("hours")` (or `"minutes"`, `"days"`, `"max"`, or a custom profile) sets how long it stays fresh.
4. `cacheTag("job-123")` labels the entry so it can be invalidated.
5. After a mutation:
   - **`updateTag(tag)`**: Server Actions only. It **immediately expires** the entry, and the next read waits for fresh data. This gives **read-your-own-writes**: the user who made the change sees it right away.
   - **`revalidateTag(tag, "max")`**: Server Actions and Route Handlers. It marks the entry stale and serves cached data while refreshing in the background (**stale-while-revalidate**). The single-argument form is deprecated in v16.
   - **`refresh()`**: refreshes uncached data from a Server Action.

```ts
// lib/data.ts
import { cacheLife, cacheTag } from "next/cache";

export async function getJob(jobId: string) {
  "use cache";
  cacheLife("hours");
  cacheTag(`job-${jobId}`, "jobs");
  return db.job.findUnique({ where: { id: jobId } });
}

// app/jobs/actions.ts
"use server";
import { updateTag, revalidateTag } from "next/cache";

export async function updateJobTitle(jobId: string, title: string) {
  const user = await requireUser();
  await assertCan(user, "job:write", jobId);   // authorization FIRST (NEXT-4)
  await db.job.update({ where: { id: jobId }, data: { title } });
  updateTag(`job-${jobId}`);                   // editor sees the new title immediately
  revalidateTag("jobs", "max");                // public listing pages can refresh in the background
}
```

**Explanation — how cache keys work:**
The cache key is derived from the function's identity plus its **arguments** and any **closed-over values**. So `getJob("a")` and `getJob("b")` are separate entries automatically. Arguments must be serializable.

**Deep dive — the per-user data rule (critical for multi-tenant SaaS):**
- You **cannot** call `cookies()` or `headers()` inside a `"use cache"` scope. Next.js errors, because a shared cache must never contain one user's data served to another.
- Read request data *outside* and pass only what the cached function needs as an argument. The argument then becomes part of the key.

```ts
async function DealerStats() {
  const dealerId = await getDealerIdFromSession();   // dynamic, outside cache
  const stats = await getDealerStats(dealerId);      // cached per dealer
  return <Stats data={stats} />;
}
async function getDealerStats(dealerId: string) {
  "use cache";
  cacheTag(`dealer-stats-${dealerId}`);
  cacheLife("minutes");
  return computeStats(dealerId);
}
```
⚠ **Danger:** if you cached `getDealerStats()` *without* the `dealerId` argument (reading the dealer inside some other way), all dealers would share one entry. That's a **cross-tenant data leak**, the worst bug class in your domain.

**Deep dive — `updateTag` vs `revalidateTag` in plain words:**
- `updateTag` = "throw it away now; whoever asks next waits for fresh data." Correct for the person who just edited.
- `revalidateTag(tag, "max")` = "mark it old; keep serving it while a fresh copy is built." Correct for everyone else, where slight staleness is fine and speed matters.

**Production use [RESUME]:** Your "automated analytics cache warming" cron jobs. Be ready to explain *which* cache (Next.js data cache? Postgres materialized views? a Redis/KV store? a table of precomputed aggregates?) and how it's invalidated or refreshed. **Answer only from what you actually built.**

**Common mistakes:**
- Putting `"use cache"` on an entire page component. Pages read params and search params; cache the data functions or child components instead.
- Caching without tags, so nothing can be invalidated.
- Using `revalidateTag` for the editor's own change, so they see stale data and think the save failed.

**Follow-up Q&A:**

**Q1. When do you choose `updateTag` vs `revalidateTag`?**
Ask who needs to see the change, and how fast:
- The user who performed the mutation must see it immediately → `updateTag` (Server Action).
- Other viewers can see it within seconds or minutes → `revalidateTag(tag, "max")`.
- Invalidating from a webhook or Route Handler (e.g., a CMS publish) → `revalidateTag`, since `updateTag` only works in Server Actions.

**Q2. With multiple server instances, where does the cache live?**
- On Vercel, the platform provides a shared cache.
- Self-hosting on multiple instances (Docker/Kubernetes) needs a shared cache handler (e.g., Redis-backed), configured via Next.js's cache handler options. Otherwise each instance has its own cache, and invalidating on one instance leaves others stale.

Mentioning this shows you understand distributed caching.

**Q3. How is `"use cache"` different from React's `cache()`?**
- React `cache()` deduplicates calls **within a single request/render**. It's per-request memoization with nothing persisted.
- `"use cache"` **persists across requests** until expiry or invalidation.

**Q4. What's `cacheLife`'s role vs tags?**
`cacheLife` is **time-based** freshness (stale/revalidate/expire durations). Tags are **event-based** invalidation. Use both: a reasonable lifetime as a safety net, plus tag invalidation on writes.

---

### NEXT-4. Server Actions: how they work and how to secure them 🔴 (security-critical)

**Short answer:**
- A Server Action is an async function marked `"use server"`. It runs on the server and can be invoked from forms or client code as if it were a local function.
- Under the hood, Next.js exposes each action as an **HTTP POST endpoint** identified by an action ID. **Anyone who obtains that ID can call it directly, with any arguments.**
- So every action must, in order:
  1. **authenticate** (who is this?)
  2. **validate** input (Zod)
  3. **authorize** (can this user do this, to *this* resource?)
  4. derive tenant/owner IDs **from the session**, never from arguments

```ts
"use server";
import { z } from "zod";
import { updateTag } from "next/cache";

const Input = z.object({ candidateId: z.string().uuid(), note: z.string().trim().min(1).max(2000) });

export async function addCandidateNote(raw: unknown) {
  const session = await getSession();                                    // 1. authenticate
  if (!session) return { ok: false, error: "UNAUTHENTICATED" } as const;

  const parsed = Input.safeParse(raw);                                   // 2. validate
  if (!parsed.success) return { ok: false, error: "INVALID_INPUT" } as const;

  const candidate = await db.candidate.findUnique({ where: { id: parsed.data.candidateId } });
  if (!candidate || !can(session.user, "candidate:write", candidate.dealerId)) {   // 3. authorize
    return { ok: false, error: "NOT_FOUND" } as const;   // same response for missing and forbidden
  }

  await db.note.create({                                                 // 4. identity from session
    data: { candidateId: candidate.id, authorId: session.user.id, body: parsed.data.note },
  });
  updateTag(`candidate-${candidate.id}`);
  return { ok: true } as const;
}
```

**Explanation — why "hidden in the UI" isn't security:**
If a viewer's UI hides the "Add note" button, the action still exists on the server. An attacker can find action IDs in the client JS bundle or network traffic and replay the POST request with modified arguments. The server must assume every call is hostile.

**Deep dive — built-in protections (know them, don't rely on them alone):**
- Actions accept only **POST**.
- Next.js compares the request's **Origin** header with the **Host** (or `X-Forwarded-Host`) and rejects mismatches, which mitigates CSRF. Extra allowed origins are configured with `serverActions.allowedOrigins`.
- Action IDs are regenerated between builds, and unused actions are removed from the client bundle.
- Values captured in closures (inline actions) are encrypted. But **don't put secrets in closures anyway**.

**Deep dive — React2Shell (CVE-2025-55182):**
- **What:** A **CVSS 10.0, pre-authentication remote code execution** flaw disclosed in December 2025. The React Server Components "Flight" protocol deserialized attacker-controlled request data unsafely.
- **Scope:** It affected the RSC server packages in React **19.0.0, 19.1.0–19.1.1, and 19.2.0** (fixed in **19.0.1, 19.1.2, 19.2.1+**), and every framework that used them, including the Next.js App Router. A single crafted HTTP request could execute code on the server.
- **Senior-level takeaways:**
  1. Your *framework's protocol* is attack surface, not just your own code.
  2. You need a dependency patch process: Dependabot/Renovate, an SLA like "critical CVEs patched within 24–48 h," and a lockfile audit.
  3. Defense in depth limits the blast radius: least-privilege DB credentials, secrets not readable by the app process where possible, egress restrictions, WAF rules.

**Common mistakes:**
- Assuming hidden buttons mean the action is protected.
- Accepting `dealerId`/`userId` as arguments and trusting them.
- Returning raw DB errors or stack traces to the client.
- Using Server Actions for **reading** data. They're designed for mutations, the client dispatches them one at a time, and they can't be cached like GET requests. Fetch data in Server Components or Route Handlers.

**Follow-up Q&A:**

**Q1. Why return NOT_FOUND instead of FORBIDDEN for another tenant's resource?**
FORBIDDEN confirms the resource *exists*, which lets attackers enumerate IDs across tenants. Returning the same response for "doesn't exist" and "not yours" leaks nothing. Log the real reason server-side for auditing.

**Q2. How do you rate-limit a Server Action?**
Inside the action, before doing work: key a counter by user ID (or IP for anonymous actions like OTP requests) in shared storage (Redis `INCR` + `EXPIRE`, or a DB row with a sliding window). Return a friendly error when exceeded. In-memory counters don't work on serverless (JS-2 Q2).

**Q3. Server Action vs API route — can a mobile app call a Server Action?**
Technically the POST exists, but the action ID format is an internal implementation detail that changes between builds. Mobile apps, webhooks, and third parties should call **Route Handlers** with a stable, versioned API contract (NEXT-7).

**Q4. What should a Server Action return on error?**
A typed result object (a discriminated union, TS-4) with safe, user-facing messages and field errors. Log the details server-side with a correlation ID. Throw only for truly unexpected failures, which the nearest `error.tsx` will catch.

**Q5. How do you avoid repeating auth code in every action?**
Centralize it in a **Data Access Layer** or a small wrapper:
```ts
export function authedAction<I, O>(schema: z.ZodType<I>, handler: (input: I, s: Session) => Promise<O>) {
  return async (raw: unknown) => {
    const s = await getSession();
    if (!s) return { ok: false, error: "UNAUTHENTICATED" } as const;
    const p = schema.safeParse(raw);
    if (!p.success) return { ok: false, error: "INVALID_INPUT" } as const;
    return { ok: true, data: await handler(p.data, s) } as const;
  };
}
```
Resource-level authorization (`can(...)`) still happens inside each handler, because only the handler knows which resource is being accessed.

---

### NEXT-5. `proxy.ts` (formerly middleware): what it's for and what NOT to do 🟠

**Short answer:**
- `proxy.ts` (the Next.js 16 name for `middleware.ts`) runs **before a request is routed**. In v16 it runs on the Node.js runtime.
- Good for:
  - redirects and rewrites
  - locale detection (your `next-intl` experience)
  - setting headers (security headers, request IDs)
  - A/B test bucketing
  - **coarse** auth gating ("no session cookie → redirect to /login")
- It should **never be the only authorization layer**. Real authorization belongs next to the data: Server Actions, Route Handlers, and the Data Access Layer.

```ts
// proxy.ts
import { NextResponse, type NextRequest } from "next/server";

export function proxy(req: NextRequest) {
  const hasSession = req.cookies.has("session");
  if (!hasSession && req.nextUrl.pathname.startsWith("/dashboard")) {
    const url = new URL("/login", req.url);
    url.searchParams.set("next", req.nextUrl.pathname);   // validated on the login page (Q1)
    return NextResponse.redirect(url);
  }
  const res = NextResponse.next();
  res.headers.set("x-request-id", crypto.randomUUID());   // correlation ID for logs
  return res;
}

export const config = { matcher: ["/dashboard/:path*"] };  // don't run on static assets
```

**Why "never the only layer" — CVE-2025-29927 (March 2025, CVSS 9.1):**
- **Mechanism:** Next.js used an internal header, `x-middleware-subrequest`, to prevent infinite middleware loops. Because the framework trusted this header without checking where it came from, an attacker could **send it themselves and skip middleware entirely**. Any authorization done only in middleware was bypassed.
- **Patched versions:** 15.2.3, 14.2.25, 13.5.9, 12.3.5.
- **Who was affected:** Self-hosted apps (`next start`, standalone output). Apps hosted on Vercel or Netlify, and static exports, were not affected. For you on Vercel this is a lesson rather than an exposure, but the architectural lesson is universal: **enforce authorization where the data is accessed.**

**Deep dive — the Data Access Layer (DAL) pattern:**
```ts
// lib/dal.ts
import "server-only";
import { cache } from "react";

export const getSessionUser = cache(async () => {        // deduped within a request
  const token = (await cookies()).get("session")?.value;
  return token ? verifySession(token) : null;
});

export async function getCandidateForViewer(candidateId: string) {
  const user = await getSessionUser();
  if (!user) throw new Unauthorized();
  const c = await db.candidate.findFirst({
    where: { id: candidateId, dealerId: user.dealerId },   // tenant scoping inside the query
    select: { id: true, name: true, city: true, stage: true },  // only the fields the UI needs (no PII leak)
  });
  if (!c) throw new NotFound();
  return c;
}
```
Every page and action calls DAL functions, never the DB directly. Auth plus tenant scoping then lives in one auditable, testable place.

**Production use [RESUME]:** Your "fail-closed server-side enforcement" belongs in this layer. If the proxy were bypassed, nothing would leak.

**Common mistakes:**
- DB calls in the proxy on every request. That adds latency to every page, including prefetches.
- An overly broad `matcher` that runs on `_next/static` and images.
- **Open redirect** via an unvalidated `?next=` parameter.

**Follow-up Q&A:**

**Q1. How do you validate the `?next=` redirect safely?**
Allow only **relative paths on your own site**: the value must start with `/` and must **not** start with `//` or `/\` (browsers treat those as protocol-relative, i.e. another domain). The safest option is an allowlist of path prefixes. Otherwise `?next=//evil.com` sends users to a phishing page after they log in.
```ts
function safeNext(n: string | null) {
  return n && n.startsWith("/") && !n.startsWith("//") && !n.startsWith("/\\") ? n : "/dashboard";
}
```

**Q2. What's a DAL, and why centralize it?**
A Data Access Layer is a server-only module that is the **only** code allowed to read or write the database for the UI. Benefits:
- Auth and tenant checks can't be forgotten.
- DTOs return only safe fields.
- A single place to audit and unit test.
- Consistent logging.

**Q3. Proxy vs Server Component auth check — what's the UX difference?**
- The proxy can redirect *before* any rendering, with no flash of protected content. It's cheap if it only checks cookie presence.
- The Server Component or DAL check is the real verification (signature, expiry, revocation, role, tenant).

Use both: the proxy for UX, the DAL for security.

**Q4. Why did Next.js rename middleware to proxy?**
To make its role explicit: a network-boundary layer in front of the app (like a reverse proxy), not general-purpose "middleware" where people were tempted to put business and auth logic.

---

### NEXT-6. Async request APIs in Next.js 16 (`params`, `searchParams`, `cookies`) 🟡

**Short answer:**
- In Next.js 16, `params` and `searchParams` are **Promises** in pages, layouts, route handlers, and metadata functions.
- `cookies()`, `headers()`, and `draftMode()` return Promises and must be `await`ed.
- Next.js 15 allowed synchronous access with deprecation warnings; **16 removed that compatibility**.
- In Client Components, unwrap a params promise with React's `use()`.

```tsx
// Server Component page
export default async function CandidatePage({
  params,
  searchParams,
}: {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ tab?: string }>;
}) {
  const { id } = await params;
  const { tab = "profile" } = await searchParams;
  const token = (await cookies()).get("session")?.value;
  return <CandidateView id={id} tab={tab} />;
}

// Client Component
"use client";
import { use } from "react";
export function Header({ params }: { params: Promise<{ id: string }> }) {
  const { id } = use(params);
  return <h1>Candidate {id}</h1>;
}
```

**Explanation — why Next.js made these async:**
Request-specific data only exists when a real request arrives. Making access explicit and async lets Next.js:
1. Prerender everything that **doesn't** depend on the request into a static shell.
2. Pause at exactly the points that **do** depend on it, and stream those parts.

With synchronous access, Next.js couldn't tell where the static part ends and the dynamic part begins, so a single cookie read deep in the tree made the whole route dynamic.

**Deep dive — typed route props:** Next.js 15.5+ can generate global helper types (`PageProps<'/candidates/[id]'>`, `LayoutProps`) via its typed routes feature, so you don't hand-write the params shape. Mention it as a nice-to-have; check the docs for the exact setup.

**Common mistakes:**
- `params.id` without `await`. In v16 this is a type error, and at runtime you're reading a property of a Promise (`undefined`).
- Awaiting `params` high in a layout just to pass it down, which makes more of the tree dynamic than needed. Pass the promise down and await it where it's used.

**Follow-up Q&A:**

**Q1. How would you upgrade a large app from Next 15 to 16 safely?**
1. Upgrade Node to 20.9+ in CI and prod.
2. Run the official codemod (`npx @next/codemod@canary upgrade latest`), then search for `@next-codemod-error` comments where it couldn't auto-migrate (custom hooks, conditional access).
3. Rename `middleware.ts` → `proxy.ts` and the exported function to `proxy`.
4. Replace single-arg `revalidateTag(tag)` with `revalidateTag(tag, "max")` or `updateTag`.
5. Replace `next lint` in scripts with direct ESLint/Biome.
6. Decide whether to enable `cacheComponents` now or later. It changes caching semantics, so do it route by route with testing.
7. Run E2E tests (Playwright) on critical flows (login/SSO, billing, search), deploy to a preview, compare performance and error metrics, and roll out.

**Q2. Does `await params` make the page slow?**
No. The params are already known; awaiting is essentially free. The async shape is about letting Next.js understand dependencies, not about fetching anything.

**Q3. What does `searchParams` access do to caching?**
Reading `searchParams` makes that part request-specific (dynamic). If only one component needs it, read it there, inside a Suspense boundary, so the rest of the page can stay in the static shell.

---

### NEXT-7. Route Handlers vs Server Actions — when to use which 🟡 [RESUME]

**Short answer:**
- **Server Actions:** mutations triggered from **your own React UI** (forms, buttons). Tight integration: typed calls, `useActionState`, cache invalidation.
- **Route Handlers** (`app/api/**/route.ts`): anything called by something **other than your own React UI**, where you need a stable HTTP contract:
  - webhooks (Razorpay)
  - mobile apps
  - third-party integrations
  - public/partner APIs
  - **cron jobs**
  - file downloads
  - GET endpoints that benefit from HTTP caching

```ts
// app/api/cron/warm-analytics/route.ts
export const dynamic = "force-dynamic";   // ensure the handler actually runs, never a cached response

export async function GET(req: Request) {
  if (req.headers.get("authorization") !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response("Unauthorized", { status: 401 });
  }
  const lock = await tryAcquireLock("warm-analytics", 10 * 60);  // e.g., Redis SET NX EX, or a DB lock row
  if (!lock) return Response.json({ skipped: "already running" });
  try {
    await warmAnalyticsCache();                                   // must be safe to run twice
    return Response.json({ ok: true });
  } finally {
    await lock.release();
  }
}
```
(If `cacheComponents` is enabled, route handlers follow its model; the key point is making sure a cron route isn't served from cache.)

**What Vercel's cron documentation actually says (know this for your resume):**
- Vercel invokes cron jobs as HTTP requests to your route. If a `CRON_SECRET` env var is set, it's **automatically sent as an `Authorization: Bearer …` header**, and your route must verify it.
- Cron delivery can **occasionally invoke the same scheduled run more than once**, and runs can be **missed**. Jobs should therefore be **idempotent and reconciliation-based**. Vercel's own example: "set status to active" is safe, "increment credit by 10" is not.
- Vercel recommends **locks** (against concurrent runs) plus **idempotent reconciliation** (for duplicate or missed runs).
- **Failed cron invocations are not retried**, so your job must catch up on the next run.
- Timing precision depends on the plan (Hobby: once per day, anywhere within the hour; Pro: per-minute).

**Deep dive — Razorpay webhook handler essentials:**
1. Read the **raw body** (`await req.text()`), since the signature is computed over the exact bytes. Parsing JSON first and re-stringifying breaks verification.
2. Verify the HMAC-SHA256 signature from the `X-Razorpay-Signature` header using your webhook secret, with a **constant-time comparison**.
3. **Deduplicate** by event ID, since webhooks can be delivered more than once (unique constraint).
4. Respond 2xx quickly, and do heavy work asynchronously.
```ts
import crypto from "node:crypto";
export async function POST(req: Request) {
  const raw = await req.text();
  const sig = req.headers.get("x-razorpay-signature") ?? "";
  const expected = crypto.createHmac("sha256", process.env.RAZORPAY_WEBHOOK_SECRET!).update(raw).digest("hex");
  const a = Buffer.from(sig), b = Buffer.from(expected);
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) return new Response("bad signature", { status: 400 });
  const event = JSON.parse(raw);
  // idempotency: store event id with a unique constraint; if it already exists → return 200 without reprocessing
  return Response.json({ received: true });
}
```
Confirm header names and the exact event ID field against Razorpay's current docs before quoting them in an interview.

**Common mistakes:**
- Unprotected cron URLs: anyone can trigger expensive jobs or message blasts.
- Non-idempotent cron jobs like "send reminders to everyone due today," which duplicate-sends on a duplicate run.
- Parsing the webhook body before verifying the signature.

**Follow-up Q&A:**

**Q1. How do you make a cron job idempotent?** **[RESUME]**
Base it on **state**, not on "this run":
- Select work by a condition that the job itself changes: `WHERE sent_at IS NULL AND due_at <= now()`.
- **Claim rows atomically** (`UPDATE … SET claimed_by = $run WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING *`), so two overlapping runs can't take the same row.
- Use unique constraints for side effects (one message per candidate per template per day).
- Write "set" operations instead of "increment" operations.

A duplicate run then finds nothing left to do, and a missed run gets picked up by the next one.

**Q2. What happens if a job exceeds the function's max duration?**
The platform kills it mid-run. Unless it was designed in **resumable batches**, some work is half-done. Process in bounded batches (e.g., 500 rows), commit progress per batch, and let the next run continue. For genuinely long work, move to a queue and workers (Vol 4).

**Q3. How do you monitor 34 cron jobs?** **[RESUME]**
- A structured log line per run: job name, run ID, duration, items processed, errors.
- A **heartbeat / dead man's switch**: alert if a job *hasn't* succeeded within its expected interval. Vercel doesn't retry failed runs, so silence is the dangerous failure mode.
- Per-job failure-rate alerts.
- A dashboard of the last success time per job.

Prepare what *you* actually did here.

**Q4. When would you move work off Vercel Cron?**
When you need automatic retries, backpressure, fan-out to many workers, sub-minute latency, long-running tasks, or guaranteed ordering. The move is to a queue (SQS, BullMQ on Redis, Kafka, or a managed queue) with workers. Your "messaging queue" crons are the first candidates.

---

### NEXT-8. Data fetching waterfalls and streaming 🟠

**Short answer:** A **waterfall** is when independent requests run sequentially: `await a(); await b(); await c();`. Total time is the **sum** of latencies. Fixes:
1. Start independent requests together with `Promise.all`, so total time ≈ the **slowest** one.
2. Split slow sections into separate components behind their own `<Suspense>` boundaries, so each streams as soon as it's ready.
3. Avoid parent→child fetch chains where a child can't start until its parent's data arrives. Hoist or preload where possible.

```tsx
// ❌ Sequential: 300 + 400 + 250 = ~950 ms
const dealer  = await getDealer(id);
const jobs    = await getJobs(id);
const credits = await getCredits(id);

// ✅ Parallel: ~400 ms (the slowest)
const [dealer2, jobs2, credits2] = await Promise.all([getDealer(id), getJobs(id), getCredits(id)]);

// ✅✅ Best UX: the shell renders immediately; each widget streams independently
export default function Dashboard({ id }: { id: string }) {
  return (
    <>
      <Suspense fallback={<Skel />}><DealerHeader id={id} /></Suspense>
      <Suspense fallback={<Skel />}><JobsWidget id={id} /></Suspense>
      <Suspense fallback={<Skel />}><CreditsWidget id={id} /></Suspense>
    </>
  );
}
```

**Explanation — `Promise.all` vs Suspense boundaries:**
- `Promise.all` makes the page wait for the **slowest** request before showing *anything* from that component.
- Separate Suspense boundaries show each widget **as soon as its own data is ready**.

Choose based on whether the data must appear together.

**Deep dive — the hidden waterfall: dependent requests:**
```ts
const user = await getUser();               // needed for...
const dealer = await getDealer(user.dealerId);  // ...this. Genuinely dependent → must be sequential
```
Here you can't parallelize. Options:
- Combine them into one query (a JOIN).
- Cache the first lookup (`React.cache` per request).
- Include the needed ID in the session so the first call isn't required.

**Deep dive — request deduplication:**
If `Header` and `Sidebar` both call `getSessionUser()` in the same render, wrap it in React's `cache()` so the lookup happens once per request (see the DAL in NEXT-5).

**Common mistakes:**
- `Promise.all` on requests that depend on each other.
- One Suspense around the entire page, so everything waits for the slowest part.
- Fetching in Client Components with `useEffect` after hydration. That creates a server → client → server waterfall.

**Follow-up Q&A:**

**Q1. What about waterfalls on the database side?**
The **N+1 query problem**: fetch 50 jobs (1 query), then fetch each job's applicant count separately (50 queries). Fix with a JOIN/aggregate in one query, batching (`WHERE job_id = ANY($1)`), or a dataloader. Covered in depth in Vol 3.

**Q2. How does React deduplicate the same data call across components in one render?**
With `cache(fn)` from React: within a single server request, calling the cached function with the same arguments returns the same promise. It resets per request, so there's no cross-user leakage.

**Q3. What if one widget's data source is down?**
Wrap each widget in its own error boundary (`error.tsx` per segment, or `react-error-boundary`) *and* Suspense boundary. The page stays up, the broken widget shows a retry, and monitoring gets the error.

---

### NEXT-9. Next.js on Vercel serverless: production constraints 🔴 [RESUME]

**Question:** "Your platform runs on Vercel. What breaks at scale, and how did you design around it?"

**Short answer — five constraints and their mitigations:**

**1. Functions are stateless and ephemeral.**
- *Constraint:* Instances start and stop based on traffic. Anything stored in memory (caches, rate-limit counters, locks) is **per-instance** and can vanish at any time.
- *Mitigation:* Shared state goes in Postgres, Redis/KV, or the platform cache. In-memory caches are acceptable only as best-effort optimizations.

**2. Database connection exhaustion.**
- *Constraint:* Every concurrent function instance can open its own DB connections. At high concurrency this exceeds Postgres's `max_connections`, and requests fail with "too many connections."
- *Mitigation:* A connection pooler. With Supabase that's its pooler in **transaction mode** (port 6543), plus Prisma configured for poolers (e.g., `?pgbouncer=true`, which disables prepared statements that transaction pooling can't support). Keep the per-instance pool small.

**3. Execution time limits.**
- *Constraint:* Functions have a maximum duration (it depends on plan and configuration, so check your settings rather than quoting a number). Long jobs get killed mid-run.
- *Mitigation:* Resumable batches with checkpoints, or move long work to background workers or queues.

**4. Cold starts.**
- *Constraint:* The first request to an idle instance pays initialization cost: loading code, creating DB clients.
- *Mitigation:* Keep server bundles lean, avoid heavy top-level work, reuse clients across invocations by defining them at module scope, and cache what's safe.

**5. Cron is a scheduler, not a job queue.**
- *Constraint:* Invocations can be duplicated or missed and are not retried on failure (NEXT-7).
- *Mitigation:* Idempotent reconciliation, locks, and heartbeat alerts.

**Explanation — why this is a senior question:**
It tests whether you understand **your own infrastructure's failure modes**. A mid-level answer lists framework features; a senior answer says "here's how it breaks, and here's what I did about it."

**Deep dive — the connection math:**
- If Postgres allows ~100 connections and each function instance's Prisma pool uses 5 connections, 20 warm instances exhaust the database.
- A traffic spike creating 60 instances fails most requests.
- A pooler multiplexes thousands of short client connections onto a small number of real DB connections.

**Deep dive — Prisma client reuse in dev and serverless:**
```ts
// lib/db.ts
import "server-only";
import { PrismaClient } from "@prisma/client";
const g = globalThis as unknown as { prisma?: PrismaClient };
export const db = g.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== "production") g.prisma = db;   // avoid new clients on every hot reload
```

**Common mistakes:**
- "Serverless scales infinitely." Compute does; your database and third-party rate limits don't.
- Using in-memory caches for correctness-critical data.
- Direct (non-pooled) connections from serverless functions.

**Follow-up Q&A:**

**Q1. At what point would you move work off Vercel Cron?**
When you need retries with backoff, backpressure, high fan-out, sub-minute latency, jobs longer than the function limit, or ordering guarantees. Then use a queue plus workers (SQS/BullMQ/Kafka, or Vercel's own queue offering). Keep crons only as *triggers* that enqueue work.

**Q2. How do you monitor 34 crons?**
See NEXT-7 Q3: structured logs per run, heartbeat alerts for missed runs, failure-rate alerts, and a last-success dashboard.

**Q3. Transaction mode vs session mode pooling — what's the difference?**
- **Session mode:** a client holds one real DB connection for its whole session. Session features (prepared statements, `SET`, advisory locks, `LISTEN`) work, but there are fewer effective connections.
- **Transaction mode:** a real connection is assigned only for the duration of each transaction and then returned to the pool. It's far more scalable for serverless, but session-level state doesn't persist between transactions.

That's why Prisma needs pooler-specific settings, and why migrations typically use a **direct** (non-pooled) connection.

**⚠ Prepare from your real system:**
- Which pooler and mode did you use?
- Did you hit connection limits or function timeouts?
- What happened when a cron failed?

One real incident with numbers beats any textbook answer.

---

### NEXT-10. Hydration errors: causes and fixes 🟡

**Short answer:** **Hydration** is React attaching event handlers and state to server-rendered HTML in the browser. It expects the browser's first render to produce **exactly** the same output as the server. If it differs, React reports a hydration error and re-renders that part on the client. You get a warning, a flicker, and wasted work.

**Common causes:**
1. Time and randomness during render: `Date.now()`, `new Date()`, `Math.random()`.
2. Locale or timezone differences: the server runs in UTC while the user is in IST, so `toLocaleString()` differs.
3. Branching on the environment: `typeof window !== "undefined" ? A : B` during render.
4. Invalid HTML nesting: `<div>` inside `<p>`, `<a>` inside `<a>`. The browser's parser "fixes" the HTML, so the DOM no longer matches.
5. Browser extensions injecting attributes (Grammarly, password managers). This is usually harmless but noisy.
6. Data that changed between the server render and hydration.

```tsx
// ❌ server (UTC) and browser (IST) produce different strings
<p>Updated {new Date(updatedAt).toLocaleString()}</p>

// ✅ Option 1: deterministic formatting (explicit locale + timezone on both sides)
const fmt = new Intl.DateTimeFormat("en-IN", { timeZone: "Asia/Kolkata", dateStyle: "medium", timeStyle: "short" });
<p>Updated {fmt.format(new Date(updatedAt))}</p>

// ✅ Option 2: client-only values rendered after mount
function RelativeTime({ iso }: { iso: string }) {
  const [text, setText] = useState<string | null>(null);
  useEffect(() => setText(formatRelative(iso)), [iso]);   // effects don't run on the server
  return <time dateTime={iso}>{text ?? ""}</time>;
}

// ✅ Option 3: skip SSR for a browser-only widget
const Map = dynamic(() => import("./Map"), { ssr: false });   // only allowed inside Client Components
```

**Explanation — why React is strict about this:**
If the server HTML says "Price: ₹500" and the client says "Price: ₹550," silently keeping the wrong text would be a correctness bug. React either patches the mismatch or throws the server HTML away for that subtree, losing the benefit of SSR there.

**Deep dive — `suppressHydrationWarning`:**
It silences the warning for **one element's own text/attributes**, one level deep. It's intended for unavoidable cases like a timestamp. It doesn't fix mismatches in children, and it hides real bugs if overused.

**Common mistakes:**
- Using `suppressHydrationWarning` as a blanket fix.
- Wrapping everything in "mounted" checks, which throws away SSR benefits.
- Ignoring invalid HTML nesting warnings.

**Follow-up Q&A:**

**Q1. Why is `useId` needed for SSR-safe IDs?**
Form labels need IDs (`<label htmlFor={id}>`). Random IDs or incrementing counters differ between server and client, which causes mismatches. `useId` generates IDs based on the component's **position in the tree**, which is identical on both sides.

**Q2. How do you debug a hydration error quickly?**
- In development, React 19 shows a **diff** of server vs client values in the error overlay. Read it first.
- Then search the flagged component for time, randomness, locale, `window` checks, or invalid nesting.
- Reproduce in an incognito window to rule out extensions.

**Q3. A theme toggle (dark/light) causes hydration mismatches. How do you fix it properly?**
The server doesn't know the user's theme preference if it's stored in `localStorage`. Options:
- Store the theme in a **cookie**, so the server renders the correct class.
- Or inject a tiny blocking inline script in `<head>` that sets the class before hydration, and use `suppressHydrationWarning` on `<html>`. This is what libraries like `next-themes` do.

---

# QUICK REVISION — 25 lines to re-read before any interview

1. `let`/`const` are hoisted but in the TDZ; `const` blocks reassignment, not mutation; `Object.freeze` is shallow.
2. Closures capture *variables* (references), not values. In-memory closure state is per instance, not global on serverless.
3. Event loop: sync → **all** microtasks → render → **one** macrotask → repeat. Code before the first `await` is synchronous.
4. `this` comes from the call site; arrows inherit it lexically. A detached method loses `this` (undefined in strict mode).
5. Promise combinators don't cancel anything. `AbortController`/`AbortSignal.timeout` do.
6. Spread is a **shallow** copy; `structuredClone` is deep (no functions); JSON cloning loses Dates, `undefined`, Map/Set.
7. React needs **structural sharing**: new references only on the changed path.
8. A leak is "unneeded but still reachable." Diagnose with 3 heap snapshots and the Comparison view.
9. Debounce = after calls stop; throttle = at most once per window; neither prevents races. Abort does.
10. A concurrency limit ≠ a rate limit. LLM and third-party APIs usually need both.
11. Retry only transient failures on idempotent operations, with capped exponential backoff and jitter. Idempotency keys make writes retry-safe.
12. `unknown` over `any`; validate every trust boundary with Zod; derive types with `z.infer`.
13. Types vanish at runtime: `Omit<T,"phone">` doesn't remove `phone`. Select or parse explicitly.
14. Discriminated unions + `assertNever` make impossible states unrepresentable and new cases impossible to forget.
15. RBAC = what action; tenant scoping = whose data. Both server-side, both fail-closed.
16. Stable keys; `key` changes deliberately reset a component's state.
17. Effects synchronize with external systems; cleanup cancels stale work; the functional updater fixes stale closures.
18. Memoize only where identity or cost matters; measure with the Profiler; the React Compiler handles most of it.
19. Server state → TanStack Query (keys, `staleTime`, invalidation). Client state → `useState`/Redux/Zustand.
20. React 19: Actions, `useActionState`, `useOptimistic` (needs a server data refresh on success), `use()`, ref as a prop.
21. Next 16: Turbopack, `proxy.ts`, async request APIs, opt-in `"use cache"`, Node 20.9+.
22. `updateTag` = read-your-own-writes (Server Actions only); `revalidateTag(tag, "max")` = stale-while-revalidate.
23. A Server Action is a public POST endpoint: authenticate, validate, authorize, derive tenant from the session in every one.
24. Never rely on proxy/middleware alone (CVE-2025-29927); patch RSC dependencies promptly (React2Shell, CVE-2025-55182).
25. Serverless: stateless, pooled DB connections (transaction mode), time limits, idempotent and locked crons (duplicate and missed runs happen; failures aren't retried).

---

**Next:** Volume 2 covers Node.js internals, Express, REST API design, and Authentication & Security (OTP flows, SSO/OIDC, JWT vs sessions, RBAC, OWASP Top 10), including deep dives on your identity migration and SSO race-condition fix.
