# Fabrik - Roblox Async & Reactive Library

[![Static Badge](https://img.shields.io/badge/build-v1.0.0-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

Fabrik is a full-stack async and reactive library for Roblox. It provides promises, raw thread control, named task scheduling, signals, cooldowns, reactive atoms, molecules, organisms, queues, locks, memoization, latches, batching, and conditions -- all unified under one module with a consistent chainable API.

---

## Table of Contents

* [Features](#features)
* [Installation](#installation)
* [Quick Start](#quick-start)
* [Core Concepts](#core-concepts)
* [API Reference](#api-reference)
* [Exported Types](#exported-types)
* [Contact](#contact)

---

## Features

* **Promise API:** `.next()`, `.catch()`, `.conclude()` -- a clean chainable surface with no raw coroutine leaks.
* **Thread Control:** Spawn, defer, delay, sleep, pause, resume, cancel, and wrap threads without touching `coroutine` or `task` directly.
* **Named Task Registry:** Define tasks by name, dispatch them immediately or deferred, pool them with a max concurrency, chain them, cancel them.
* **Persistent Queue:** A sequential queue that lives across dispatches with pause, resume, and drain events.
* **Lock:** Prevents two tasks from running against the same resource at the same time via `acquire`, `release`, and `withLock`.
* **Memo:** Caches the result of an async task by key with a timeout -- duplicate calls for the same key while one is in-flight collapse into one.
* **Condition:** Poll a value or condition on an interval, resolves when it changes or meets a predicate.
* **Signal:** Promise-aware signals with `filter`, `map`, `merge`, `pipe`, `toTask`, and `promiseWhere`.
* **Cooldown:** Chainable cooldown primitives with `auth`, `throttle`, `debounce`, and direct promise integration.
* **Atom:** Reactive single values with `onChange`, `promise`, and `derive` for computed atoms.
* **Molecule:** Composed atom groups with `patch`, `get`, and unified change propagation.
* **Organism:** Composed molecule groups forming a full reactive state tree.
* **Batch:** Group multiple atom mutations and flush listeners once at the end of a frame.
* **Latch:** Holds all waiting threads until opened -- all of them release at the same time.

---

## Installation

Place the Fabrik module into ReplicatedStorage, then require it wherever needed. Fabrik is safe to use on both server and client -- task dispatch, thread control, atoms, signals, and reactive primitives work in both environments.

```lua
local Fabrik = require(ReplicatedStorage.Fabrik)
```

---

## Quick Start

```lua
local Fabrik = require(ReplicatedStorage.Fabrik)

-- define a task
Fabrik.task("loadPlayer", function(userId)
    return Keep.load(userId)
end)

-- dispatch and chain
Fabrik.run("loadPlayer", player.UserId)
    :next(function(data)
        print("loaded:", data.coins)
    end)
    :catch(function(err)
        warn("failed:", err)
    end)
    :conclude(function()
        print("always runs")
    end)

-- reactive atom
local hp = Fabrik.atom(100)

hp:onChange(function(new, old)
    if new <= 0 then
        Fabrik.run("onDeath", player)
    end
end)

-- signal gated by cooldown
local hitSignal  = Fabrik.signal.new()
local attackCd   = Fabrik.cooldown.new(1)

hitSignal
    :filter(function() return attackCd:isReady() end)
    :next(function(amount)
        attackCd:start()
        Fabrik.batch(function()
            hp:set(hp:get() - amount)
        end)
    end)
```

---

## Core Concepts

### Promises
Fabrik's promise surface is intentionally constrained. Nothing from the internal implementation leaks out -- the only API you ever touch is `.next()`, `.catch()`, `.conclude()`, and `.cancel()`. Every async operation in Fabrik returns one of these.

### Threads vs Tasks
`Fabrik.thread` is raw coroutine/task control -- spawn a thread, sleep it, cancel it. `Fabrik.task` is a named registry on top of threads -- define work by name, dispatch it with arguments, track its handle, pool multiple dispatches with a concurrency cap.

### Reactive Primitives
Atoms are single reactive values. Molecules group atoms. Organisms group molecules. Every level propagates changes upward and exposes the same `onChange`, `promise`, `get`, and `patch` interface. Use `Fabrik.batch` to group mutations and fire listeners once.

### Signals
Fabrik signals are not plain event emitters. Every signal can be transformed with `filter`, `map`, and `merge` to produce new signals without any intermediate listeners. `promiseWhere` converts a signal into a one-shot promise that resolves only when a predicate passes.

### Latches
A latch holds all waiting threads until it opens. Threads calling `latch:await()` block until someone calls `latch:open()`, at which point every waiter releases at the same time. Unlike a signal, a latch that is already open will let new arrivals through immediately -- until `latch:reset()` closes it again.

---

## API Reference

---

### Fabrik.promise

#### Constructor
```lua
Fabrik.promise(fn: (resolve, reject) -> ()) -> Promise
```
Creates a new promise. The executor receives `resolve` and `reject` functions. Errors thrown inside the executor automatically call `reject`.

#### Static Methods

```lua
Fabrik.promise.resolve(value)             -- already-resolved promise
Fabrik.promise.reject(err)               -- already-rejected promise
Fabrik.promise.all({ promises })         -- resolves when all resolve, rejects on first failure
Fabrik.promise.any({ promises })         -- resolves with the first to resolve
Fabrik.promise.race({ promises })        -- resolves or rejects with the first to settle
Fabrik.promise.allSettled({ promises })  -- resolves with all outcomes regardless of failure
Fabrik.promise.delay(seconds)            -- resolves after n seconds
Fabrik.promise.retry(fn, times, backoff?) -- retries fn up to n times, optional exponential backoff
Fabrik.promise.timeout(promise, seconds) -- rejects promise if not resolved in time
Fabrik.promise.fromSignal(signal)        -- one-shot promise that resolves on next signal fire
Fabrik.promise.fromEvent(signal, predicate?) -- same but with optional filter
Fabrik.promise.each(list, fn, concurrency?) -- maps list through async fn with optional concurrency cap
```

#### Instance Methods

```lua
promise:next(fn)      -- runs fn on resolve, returns a new promise
promise:catch(fn)     -- runs fn on reject, returns a new promise
promise:conclude(fn)  -- always runs regardless of outcome
promise:cancel()      -- cancels if still pending
promise:await()       -- yields current thread, returns (ok, value)
```

---

### Fabrik.thread

Raw coroutine and task control. All methods map closely to Luau's `task` and `coroutine` globals but with consistent error capture and clean cancellation.

```lua
Fabrik.thread.spawn(fn)           -- fresh thread, errors captured
Fabrik.thread.defer(fn)           -- runs on next frame
Fabrik.thread.delay(n, fn)        -- runs after n seconds
Fabrik.thread.sleep(n)            -- yields current thread for n seconds
Fabrik.thread.pause()             -- yields current thread indefinitely
Fabrik.thread.resume(thread)      -- resumes a paused thread
Fabrik.thread.cancel(thread)      -- kills a thread cleanly
Fabrik.thread.current()           -- returns the running thread
Fabrik.thread.wrap(fn)            -- error-safe coroutine.wrap equivalent
Fabrik.thread.interval(n, fn)     -- repeating tick, returns a cancellable handle
Fabrik.thread.once(signal, fn)    -- spawns a thread that waits for one signal fire
Fabrik.thread.race(threads)       -- first thread to finish wins, rest are cancelled
```

#### interval handle
```lua
local handle = Fabrik.thread.interval(0.5, fn)
handle:cancel()
```

---

### Fabrik.task

Named task registry with scheduling, pooling, chaining, and cancellation.

#### Defining Tasks

```lua
Fabrik.task("name", function(...) end)
```

#### Dispatch Methods

```lua
Fabrik.run(name, ...)      -- immediate dispatch, returns a handle
Fabrik.defer(name, ...)    -- deferred to next frame, returns a handle
Fabrik.queue(name, ...)    -- adds to sequential queue, returns a handle
Fabrik.spawn(name, ...)    -- fresh thread dispatch, returns a handle
```

#### Pooling and Chaining

```lua
-- parallel with concurrency cap
Fabrik.pool(limit, {
    Fabrik.run("task", arg1),
    Fabrik.run("task", arg2),
}):next(function(results) end)

-- sequential, pipes results
Fabrik.chain(
    Fabrik.run("fetch", id),
    Fabrik.run("process"),
    Fabrik.run("save")
)

-- batch dispatch -- single promise for a group
Fabrik.task.batch({
    Fabrik.run("taskA"),
    Fabrik.run("taskB"),
}):next(function(results) end)

-- recurring task
Fabrik.task.schedule("name", interval)
```

#### Task Handle

```lua
handle:await()             -- yield until done
handle:next(fn)            -- promise interface
handle:catch(fn)
handle:conclude(fn)
Fabrik.cancel(handle)      -- cancel in-flight task
Fabrik.task.status(handle) -- "pending" | "running" | "done" | "cancelled" | "failed"
```

#### Error Handling

```lua
Fabrik.task.onError(function(err, name)
    warn("uncaught task error in", name, err)
end)
```

---

### Fabrik.queue

A persistent sequential queue that lives across dispatches.

```lua
local q = Fabrik.queue.new({ concurrency = 1, priority = false })

q:push(fn)        -- add work
q:pause()         -- halt processing
q:resume()        -- continue processing
q:clear()         -- remove all pending work
q:size()          -- number of pending items
q.drained:connect(fn) -- fires when queue empties
```

---

### Fabrik.lock

Prevents two tasks from running against the same resource at the same time.

```lua
local lock = Fabrik.lock.new()

lock:acquire():next(function()
    -- exclusive access
    lock:release()
end)

-- auto-release pattern
lock:withLock(function()
    -- runs exclusively, releases on return or error
end)
```

---

### Fabrik.memo

Caches the result of an async task by key. Results expire after a set timeout. Duplicate calls for the same key while one is already running collapse into one -- the handler only runs once.

```lua
local cached = Fabrik.memo.new(function(userId)
    return Keep.load(userId)
end, { timeout = 30 })

cached(userId):next(function(data)
    print(data.coins)
end)

cached:invalidate(userId)  -- force a fresh fetch next call
cached:clear()             -- wipe all cached values
```

---

### Fabrik.condition

Poll a value or condition on a configurable interval. Returns a promise.

```lua
-- resolves when condition is true
Fabrik.condition.until(function()
    return game.Players.NumPlayers >= 4
end):next(function()
    startRound()
end)

-- resolves on next change
Fabrik.condition.onChange(function()
    return workspace.RoundActive.Value
end):next(function(newValue)
    print("changed to:", newValue)
end)

-- optional config
Fabrik.condition.until(fn, { interval = 0.5 })
```

---

### Fabrik.signal

Promise-aware signals with transformation support.

#### Construction

```lua
local signal = Fabrik.signal.new()
```

#### Core Methods

```lua
signal:connect(fn)     -- returns a disconnect function
signal:once(fn)        -- fires once then auto-disconnects
signal:fire(...)       -- fires all listeners
signal:destroy()       -- cleans up all connections
```

#### Transformation -- returns a new signal

```lua
signal:filter(fn)           -- only forwards fires where predicate returns true
signal:map(fn)              -- transforms fired values before forwarding
signal:merge(otherSignal)   -- new signal that fires when either fires
signal:pipe(otherSignal)    -- forwards all fires to another signal
```

#### Promise Integration

```lua
signal:promise()              -- resolves on next fire
signal:promiseWhere(fn)       -- resolves only when predicate passes
```

#### Task Integration

```lua
signal:toTask("taskName")     -- every fire dispatches the named task with fired args
```

---

### Fabrik.cooldown

Chainable cooldowns with promise and task integration.

```lua
local cd = Fabrik.cooldown.new(duration)

cd:start()          -- starts the cooldown timer
cd:reset()          -- resets without starting
cd:isReady()        -- true if cooldown has expired
cd:remaining()      -- seconds remaining

cd:promise()        -- resolves when cooldown expires
cd:andThen(fn)      -- runs fn when ready, chains directly
cd:auth(task)       -- dispatches task only if ready, resets after
cd:throttle(fn)     -- wraps fn -- calls during cooldown are silently dropped
cd:debounce(fn)     -- wraps fn -- resets timer on each call, fires after cooldown
```

---

### Fabrik.atom

A single reactive value.

```lua
local value = Fabrik.atom(initial)

value:get()                        -- read current value
value:set(newValue)                -- write new value
value:onChange(function(new, old) end)  -- fires on every change
value:promise()                    -- resolves on next change
value:derive(fn)                   -- returns a new computed atom
```

#### Derived Atoms

```lua
local isDead = hp:derive(function(v)
    return v <= 0
end)

isDead:onChange(function(dead)
    if dead then onDeath() end
end)
```

---

### Fabrik.molecule

A group of named atoms treated as a single reactive unit.

```lua
local stats = Fabrik.molecule({
    hp      = Fabrik.atom(100),
    shield  = Fabrik.atom(50),
    stamina = Fabrik.atom(100),
})

stats.hp:set(80)

stats:get()                              -- snapshot of all values
stats:patch({ hp = 80, stamina = 90 })  -- batch set multiple fields
stats:onChange(function(key, new, old) end)
stats:promise()                          -- resolves on any inner atom change
```

---

### Fabrik.organism

A group of named molecules forming a reactive state tree.

```lua
local player = Fabrik.organism({
    stats    = statsMolecule,
    position = Fabrik.molecule({ x = Fabrik.atom(0), z = Fabrik.atom(0) }),
})

player:get()       -- deep snapshot of all molecules and atoms
player:patch({...})
player:onChange(function(molecule, key, new, old) end)
player:promise()   -- resolves on any change anywhere in the tree
```

---

### Fabrik.batch

Groups multiple atom or molecule mutations and flushes all `onChange` listeners once after the block completes. Without batching, each `:set()` fires its own listeners immediately.

```lua
Fabrik.batch(function()
    hp:set(50)
    shield:set(0)
    stamina:set(100)
end)
-- onChange listeners fire once here, not three times above
```

---

### Fabrik.latch

Holds all waiting threads until opened. All of them release at the same time. New arrivals on an already-open latch pass through immediately -- until `reset()` is called.

```lua
local latch = Fabrik.latch.new()

-- multiple threads waiting
Fabrik.thread.spawn(function()
    latch:await()
    print("released")
end)

latch:open()       -- releases all current and future waiters
latch:reset()      -- closes the latch again
latch:isOpen()     -- boolean
latch:promise()    -- resolves when latch opens
```

---

## Exported Types

```lua
export type Promise<T> = {
    next:     (self: Promise<T>, cb: (T) -> any) -> Promise<any>,
    catch:    (self: Promise<T>, cb: (any) -> ()) -> Promise<T>,
    conclude: (self: Promise<T>, cb: () -> ()) -> Promise<T>,
    cancel:   (self: Promise<T>) -> (),
    await:    (self: Promise<T>) -> (boolean, T),
}

export type Signal<T> = {
    connect:      (self: Signal<T>, cb: (T) -> ()) -> () -> (),
    once:         (self: Signal<T>, cb: (T) -> ()) -> (),
    fire:         (self: Signal<T>, ...any) -> (),
    filter:       (self: Signal<T>, fn: (T) -> boolean) -> Signal<T>,
    map:          (self: Signal<T>, fn: (T) -> any) -> Signal<any>,
    merge:        (self: Signal<T>, other: Signal<any>) -> Signal<any>,
    pipe:         (self: Signal<T>, other: Signal<any>) -> (),
    promise:      (self: Signal<T>) -> Promise<T>,
    promiseWhere: (self: Signal<T>, fn: (T) -> boolean) -> Promise<T>,
    toTask:       (self: Signal<T>, name: string) -> (),
    destroy:      (self: Signal<T>) -> (),
}

export type Atom<T> = {
    get:      (self: Atom<T>) -> T,
    set:      (self: Atom<T>, value: T) -> (),
    onChange: (self: Atom<T>, cb: (T, T) -> ()) -> () -> (),
    promise:  (self: Atom<T>) -> Promise<T>,
    derive:   (self: Atom<T>, fn: (T) -> any) -> Atom<any>,
}

export type Cooldown = {
    start:    (self: Cooldown) -> (),
    reset:    (self: Cooldown) -> (),
    isReady:  (self: Cooldown) -> boolean,
    remaining:(self: Cooldown) -> number,
    promise:  (self: Cooldown) -> Promise<()>,
    andThen:  (self: Cooldown, fn: () -> ()) -> Cooldown,
    auth:     (self: Cooldown, task: any) -> (),
    throttle: (self: Cooldown, fn: (...any) -> ()) -> (...any) -> (),
    debounce: (self: Cooldown, fn: (...any) -> ()) -> (...any) -> (),
}

export type TaskHandle = {
    next:    (self: TaskHandle, cb: (...any) -> ()) -> TaskHandle,
    catch:   (self: TaskHandle, cb: (any) -> ()) -> TaskHandle,
    conclude:(self: TaskHandle, cb: () -> ()) -> TaskHandle,
    await:   (self: TaskHandle) -> (boolean, ...any),
}

export type TaskStatus = "pending" | "running" | "done" | "cancelled" | "failed"

export type Latch = {
    await:   (self: Latch) -> (),
    open:    (self: Latch) -> (),
    reset:   (self: Latch) -> (),
    isOpen:  (self: Latch) -> boolean,
    promise: (self: Latch) -> Promise<()>,
}
```

All types are exported from `Fabrik/Types.lua` and re-exported by the main module.

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X (Twitter) | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Last Updated: May 8, 2026*

---