<div align="center">
    <img src="assets/logo.svg" width="700" height="500" alt="People standing in queue illustration">
    <h1>Promise Queue</h1>
    <p>
        <b>Promise queue with concurrency control. Deno port of <a href="https://github.com/sindresorhus/p-queue">sindresorhus's p-queue for node</a></b>
    </p>
    <p>
        <img alt="build status" src="https://img.shields.io/github/workflow/status/KhushrajRathod/pQueue/Deno?label=checks" >
        <img alt="language" src="https://img.shields.io/github/languages/top/KhushrajRathod/pQueue" >
        <img alt="code size" src="https://img.shields.io/github/languages/code-size/KhushrajRathod/pQueue">
        <img alt="issues" src="https://img.shields.io/github/issues/KhushrajRathod/pQueue" >
        <img alt="license" src="https://img.shields.io/github/license/KhushrajRathod/pQueue">
        <img alt="version" src="https://img.shields.io/github/v/release/KhushrajRathod/pQueue">
    </p>
    <p>
        <b><a href="https://deno.land/x/p_queue">View on deno.land</a></b>
    </p>
    <br>
    <br>
    <br>
</div>

Useful for rate-limiting async (or sync) operations. For example, when interacting with a REST API or when doing CPU/memory intensive tasks.

## Usage

Here we run only one promise at the time. For example, set `concurrency` to 4 to run four promises at the same time.

```js
import PQueue from "https://deno.land/x/p_queue@1.0.1/mod.ts"

const queue = new PQueue({
    concurrency: 1,
})

async function one() {
    await queue.add(() => fetch("https://sindresorhus.com"))
    console.log("Done: sindresorhus.com")
}

async function two() {
    await queue.add(() => fetch("https://avajs.dev"))
    console.log("Done: avajs.dev")
}

async function three() {
    const task = await getUnicornTask()
    await queue.add(task)
    console.log("Done: Unicorn task")
}

one()
two()
three()
```

## API

See https://doc.deno.land/https/deno.land/x/p_queue@1.0.1/mod.ts

## Events

### active

Emitted as each item is processed in the queue for the purpose of tracking progress.

```js
import PQueue from "https://deno.land/x/p_queue@1.0.1/mod.ts"
const delay = (ms: number) => new Promise(r => setTimeout(r, ms))

const queue = new PQueue({
    concurrency: 2,
})

let count = 0
queue.on("active", () => {
    console.log(
        `Working on item #${++count}.  Size: ${queue.size}  Pending: ${queue.pending}`,
    )
})

queue.add(() => Promise.resolve())
queue.add(() => delay(2000))
queue.add(() => Promise.resolve())
queue.add(() => Promise.resolve())
queue.add(() => delay(500))
```

### idle

Emitted every time the queue becomes empty and all promises have completed `queue.size === 0 && queue.pending === 0` .

```js
import PQueue from "https://deno.land/x/p_queue@1.0.1/mod.ts"
const delay = (ms: number) => new Promise(r => setTimeout(r, ms))

const queue = new PQueue()

queue.on("idle", () => {
    console.log(
        `Queue is idle.  Size: ${queue.size}  Pending: ${queue.pending}`,
    )
})

const job1 = queue.add(() => delay(2000))
const job2 = queue.add(() => delay(500))

await job1
await job2
// => 'Queue is idle.  Size: 0  Pending: 0'

await queue.add(() => delay(600))
// => 'Queue is idle.  Size: 0  Pending: 0'
```

The `idle` event is emitted every time the queue reaches an idle state. On the other hand, the promise the `onIdle()` function returns resolves once the queue becomes idle instead of every time the queue is idle.

### add

Emitted every time the add method is called and the number of pending or queued tasks is increased.

### next

Emitted every time a task is completed and the number of pending or queued tasks is decreased.

```js
import PQueue from "https://deno.land/x/p_queue@1.0.1/mod.ts"
const delay = (ms: number) => new Promise(r => setTimeout(r, ms))

const queue = new PQueue()

queue.on("add", () => {
    console.log(
        `Task is added.  Size: ${queue.size}  Pending: ${queue.pending}`,
    )
})
queue.on("next", () => {
    console.log(
        `Task is completed.  Size: ${queue.size}  Pending: ${queue.pending}`,
    )
})

const job1 = queue.add(() => delay(2000))
const job2 = queue.add(() => delay(500))

await job1
await job2
// => 'Task is added.  Size: 0  Pending: 1'
// => 'Task is added.  Size: 0  Pending: 2'

await queue.add(() => delay(600))
// => 'Task is completed.  Size: 0  Pending: 1'
// => 'Task is completed.  Size: 0  Pending: 0'
```

## Advanced example

A more advanced example to help you understand the flow.

```js
import PQueue from "https://deno.land/x/p_queue@1.0.1/mod.ts"
const delay = (ms: number) => new Promise(r => setTimeout(r, ms))

const queue = new PQueue({
    concurrency: 1,
})

async function taskOne() {
    await delay(200)

    console.log(`8. Pending promises: ${queue.pending}`)(// => '8. Pending promises: 0'

    async () => {
        await queue.add(async () => "🐙")
        console.log("11. Resolved")
    })()

    console.log("9. Added 🐙")

    console.log(`10. Pending promises: ${queue.pending}`)
    // => '10. Pending promises: 1'

    await queue.onIdle()
    console.log("12. All work is done")
}

async function taskTwo() {
    await queue.add(async () => "🦄")
    console.log("5. Resolved")
}

console.log("1. Added 🦄")
;(async () => {
    await queue.add(async () => "🐴")
    console.log("6. Resolved")
})()
console.log("2. Added 🐴")
;(async () => {
    await queue.onEmpty()
    console.log("7. Queue is empty")
})()

console.log(`3. Queue size: ${queue.size}`)
// => '3. Queue size: 1`

console.log(`4. Pending promises: ${queue.pending}`)
// => '4. Pending promises: 1'
```

```
$ deno run -A test.ts

01. Added 🦄
02. Added 🐴
03. Queue size: 1
04. Pending promises: 1
05. Resolved 🦄
06. Resolved 🐴
07. Queue is empty
08. Pending promises: 0
09. Added 🐙
10. Pending promises: 1
11. Resolved 🐙
12. All work is done
```

## Custom QueueClass

For implementing more complex scheduling policies, you can provide a QueueClass in the options:

```js
class QueueClass {
    constructor() {
        this._queue = []
    }

    enqueue(run, options) {
        this._queue.push(run)
    }

    dequeue() {
        return this._queue.shift()
    }

    get size() {
        return this._queue.length
    }

    filter(options) {
        return this._queue
    }
}

const queue = new PQueue({
    queueClass: QueueClass,
})
```

`p_queue` will call corresponding methods to put and get operations from this queue.

## FAQ

#### How do the `concurrency` and `intervalCap` options affect each other?

They are just different constraints. The `concurrency` option limits how many things run at the same time. The `intervalCap` option limits how many things run in total during the interval (over time).

## Supporters

- HUGE thanks to @sindresorhus -- this repository is mostly his code, modified to work with Deno

[![Stargazers repo roster for @KhushrajRathod/pQueue](https://reporoster.com/stars/KhushrajRathod/pQueue)](https://github.com/KhushrajRathod/pQueue/stargazers)

[![Forkers repo roster for @KhushrajRathod/pQueue](https://reporoster.com/forks/KhushrajRathod/pQueue)](https://github.com/KhushrajRathod/pQueue/network/members)

## Related

- [pTimeout](https://github.com/KhushrajRathod/pTimeout)
- [pRetried](https://github.com/KhushrajRathod/pRetried)
- [retried](https://github.com/KhushrajRathod/retried)
- [...more](https://github.com/KhushrajRathod/denoModules)

## License

- P(romise) Queue is licensed under the MIT license.
- Code is adapted from Sindre's [p-queue for node](https://github.com/sindresorhus/p-queue) (also under the MIT license)
