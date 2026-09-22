# Python Async & Concurrency --- Interview Notes

> Topics 4.1--4.9\
> Focus: Python Backend + AI/Agentic AI interviews

## Table of Contents

1.  [4.1 Concurrency Fundamentals](#41-concurrency-fundamentals)
2.  [4.2 GIL](#42-gil)
3.  [4.3 Threading](#43-threading)
4.  [4.4 Multiprocessing](#44-multiprocessing)
5.  [4.5 Asyncio Fundamentals & Event
    Loop](#45-asyncio-fundamentals--event-loop)
6.  [4.6 Coroutines, Tasks & Futures](#46-coroutines-tasks--futures)
7.  [4.7 gather, wait & as_completed](#47-gather-wait--as_completed)
8.  [4.8 Timeouts & Cancellation](#48-timeouts--cancellation)
9.  [4.9 Async Synchronization](#49-async-synchronization)
10. [Interview Traps](#interview-traps)
11. [AI/Backend Mental Model](#aibackend-mental-model)

------------------------------------------------------------------------

# 4.1 Concurrency Fundamentals

## Concurrency vs Parallelism

### Concurrency

Multiple tasks make progress during overlapping time periods.

They do not necessarily execute at exactly the same time.

``` text
Task A:  ── work ── wait for I/O ───────── work ──
Task B:              ── work ── wait ── work ─────
```

### Parallelism

Multiple tasks execute simultaneously on different CPU execution
resources.

``` text
Core 1:  ───────── Task A ─────────
Core 2:  ───────── Task B ─────────
```

**Interview definition:**

> Concurrency is about dealing with multiple tasks at the same time;
> parallelism is about executing multiple tasks at the same time.

## I/O-Bound vs CPU-Bound

### I/O-bound

The program spends significant time waiting for:

-   Network
-   Database
-   Filesystem
-   External APIs
-   MCP calls
-   Vector databases

Common approaches:

-   `asyncio`
-   Threading

### CPU-bound

The program spends significant time computing:

-   Image processing
-   Heavy numerical computation
-   CPU-heavy parsing
-   Pure Python algorithms

Common approaches:

-   Multiprocessing
-   Native libraries that release the GIL

## Choosing an Approach

  Approach          Best suited for
  ----------------- ---------------------------------------------
  `asyncio`         Large number of asynchronous I/O operations
  Threading         Blocking/synchronous I/O
  Multiprocessing   CPU-heavy Python work

------------------------------------------------------------------------

# 4.2 GIL

The **Global Interpreter Lock (GIL)** in traditional/current standard
CPython execution allows only one thread at a time to execute Python
bytecode within a given interpreter.

## Important consequences

CPU-bound pure-Python threads do not achieve true parallel execution
within the same interpreter because of the GIL.

However:

> **The GIL does NOT make threading useless.**

Blocking I/O can release the GIL, so threads are useful for I/O-bound
work.

Native extensions can also release the GIL.

## CPU-bound Example

``` python
def calculate():
    # CPU-heavy pure Python
    ...
```

Using many threads does not necessarily provide CPU parallelism.

For CPU-heavy pure Python, multiprocessing is usually more appropriate.

``` python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=8) as executor:
    ...
```

Each process has:

-   Its own Python interpreter
-   Its own memory
-   Its own GIL

Therefore multiple processes can execute Python code in parallel.

## GIL != Synchronization

A common interview trap:

> "Because of the GIL, Python code cannot have race conditions."

**False.**

The GIL protects interpreter execution at a lower level. It does **not**
make a multi-step application operation atomic.

For example:

``` python
counter += 1
```

is logically:

``` text
read counter
add 1
write counter
```

Two threads can interleave these operations.

Use an appropriate synchronization primitive such as `threading.Lock`.

------------------------------------------------------------------------

# 4.3 Threading

A thread is an execution path inside a process.

Threads in the same process share:

-   Memory
-   Global state
-   File descriptors
-   Process resources

but each thread has its own execution state/stack.

## Creating a Thread

``` python
import threading

def worker():
    print("working")

t = threading.Thread(target=worker)

t.start()
t.join()
```

### `start()` vs `run()`

``` python
t.start()
```

Starts execution in a separate thread.

``` python
t.run()
```

Directly invokes the target in the current thread. It does **not**
create a new thread.

### `join()`

``` python
t.join()
```

Waits for the thread to finish.

## Race Condition

Consider:

``` python
counter += 1
```

Conceptually:

``` text
read counter
add 1
write counter
```

Two threads can interleave these operations and lose updates.

### Protecting a Critical Section

``` python
lock = threading.Lock()

with lock:
    counter += 1
```

Only one thread can execute the protected section at a time.

## Thread Synchronization Primitives

### Lock

Allows one thread at a time.

``` python
lock = threading.Lock()

with lock:
    ...
```

### RLock

A **reentrant lock**.

The same thread can acquire the lock multiple times without deadlocking
itself.

### Semaphore

Allows up to `N` concurrent entrants.

``` python
sem = threading.Semaphore(5)
```

### Event

Used for signaling between threads.

## ThreadPoolExecutor

Useful when you have many blocking I/O operations and want bounded
worker threads.

``` python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as executor:
    ...
```

## Threading Production Problems

-   Too many threads
-   Memory overhead
-   Context switching
-   Race conditions
-   Deadlocks
-   Lock contention
-   GIL limitations for CPU-bound pure Python

------------------------------------------------------------------------

# 4.4 Multiprocessing

Multiprocessing uses separate OS processes.

Each process has:

-   Separate memory
-   Separate Python interpreter
-   Separate GIL
-   Independent execution state

This allows CPU-bound pure Python work to execute in parallel.

## Process Pool

``` python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=8) as executor:
    ...
```

For an 8-core machine, a worker count around the number of available
cores can be a reasonable starting point, but the correct value depends
on workload.

### Important

8 cores does **not** mean the machine can only have 8 processes.

You can have many processes, but only a limited number can execute
simultaneously according to available CPU execution resources.

A process pool keeps a bounded number of workers and queues remaining
work.

## Process Memory

Processes normally do not share regular Python memory.

Communication can use:

-   `multiprocessing.Queue`
-   `Pipe`
-   Shared memory
-   `Value`
-   `Array`
-   `Manager`

## Threading vs Multiprocessing

  Feature                           Threading              Multiprocessing
  --------------------------------- ---------------------- --------------------------
  Memory                            Shared                 Separate
  GIL                               Same interpreter GIL   Separate GIL per process
  CPU parallelism for pure Python   Limited                Yes
  I/O                               Excellent              Usually unnecessary
  Creation overhead                 Lower                  Higher
  Communication                     Easier                 IPC required

------------------------------------------------------------------------

# 4.5 Asyncio Fundamentals & Event Loop

`asyncio` is Python's framework for asynchronous I/O.

Core concepts:

``` text
Coroutine
    ↓
Task
    ↓
Event Loop
    ↓
Await / I/O
```

## Coroutine Function

``` python
async def fetch():
    return "data"
```

Calling:

``` python
coro = fetch()
```

creates a **coroutine object**.

The function body does not execute simply because the function was
called.

## `await`

``` python
result = await fetch()
```

The current coroutine waits for the awaited operation.

If the operation needs to wait for I/O, the current coroutine can
suspend and the event loop can run other ready tasks.

### Critical point

> **`await` does not automatically mean concurrency.**

## Sequential Async Code

``` python
async def main():
    await task("A")
    await task("B")
```

If each task takes 2 seconds:

``` text
A:  |----2s----|
B:              |----2s----|

Total ≈ 4s
```

B isn't started until A completes.

## Concurrent Async Code

``` python
async def main():
    await asyncio.gather(
        task("A"),
        task("B")
    )
```

``` text
A:  |----2s----|
B:  |----2s----|

Total ≈ 2s
```

## Event Loop

The event loop:

1.  Finds ready work.
2.  Runs a coroutine.
3.  The coroutine reaches an `await` that suspends it.
4.  The event loop runs another ready task.
5.  When the I/O/timer completes, the suspended task becomes runnable
    again.

Conceptually:

``` text
Task A
  ↓
await I/O
  ↓
suspend
  ↓
Event Loop
  ↓
Task B
  ↓
...
```

Asyncio normally uses cooperative scheduling.

## `asyncio.run()`

Typical top-level entry point:

``` python
asyncio.run(main())
```

It manages the event loop for the top-level async program.

## Blocking Code Trap

This is bad inside an event-loop thread:

``` python
async def work():
    time.sleep(5)
```

`time.sleep()` blocks the event-loop thread.

Prefer:

``` python
async def work():
    await asyncio.sleep(5)
```

`asyncio.sleep()` allows the event loop to run other tasks.

------------------------------------------------------------------------

# 4.6 Coroutines, Tasks & Futures

## Coroutine

``` python
async def work():
    return 100

coro = work()
```

`coro` is a coroutine object.

It represents async work but is not independently scheduled as a Task.

## Task

``` python
task = asyncio.create_task(work())
```

A Task schedules a coroutine to run on the event loop.

Mental model:

``` text
Coroutine
    ↓
create_task()
    ↓
Task
    ↓
Event Loop
```

### Why `create_task()` matters

``` python
async def main():
    task_a = asyncio.create_task(a())
    task_b = asyncio.create_task(b())

    await task_a
    await task_b
```

Both A and B have already been scheduled.

`await task_a` means:

> The current coroutine (`main`) waits for Task A.

It does **not** prevent the event loop from running Task B.

## Future

A Future represents a result that will become available later.

Mental model:

> **Coroutine = async work**\
> **Task = scheduled coroutine**\
> **Future = placeholder for a future result**

------------------------------------------------------------------------

# 4.7 `gather()`, `wait()` & `as_completed()`

## `asyncio.gather()`

Runs multiple awaitables concurrently and collects their results.

``` python
results = await asyncio.gather(
    a(),
    b(),
    c()
)
```

If:

``` text
A → 3s
B → 1s
C → 2s
```

completion order:

``` text
B → C → A
```

But:

``` python
results
```

is:

``` python
["A", "B", "C"]
```

because `gather()` preserves **input order**.

### Return type

``` python
type(results)
# list
```

## Exceptions in `gather()`

Default:

``` python
await asyncio.gather(a(), b(), c())
```

If one awaitable raises, the exception is propagated to the caller.

Other operations are **not automatically cancelled simply because one
raised**.

## `return_exceptions=True`

``` python
results = await asyncio.gather(
    a(),
    b(),
    c(),
    return_exceptions=True
)
```

Possible result:

``` python
[
    "A",
    ValueError("B failed"),
    "C"
]
```

Useful when individual failures should be handled independently.

## `asyncio.as_completed()`

Processes results in **completion order**.

``` python
for future in asyncio.as_completed(tasks):
    result = await future
```

If:

``` text
B → 1s
C → 2s
A → 3s
```

processing happens:

``` text
B → C → A
```

Useful when you want to process a result immediately instead of waiting
for all operations.

## `asyncio.wait()`

``` python
done, pending = await asyncio.wait(tasks)
```

Returns:

-   `done` → completed Tasks
-   `pending` → Tasks still running

### `FIRST_COMPLETED`

``` python
done, pending = await asyncio.wait(
    tasks,
    return_when=asyncio.FIRST_COMPLETED
)
```

Returns as soon as any Task finishes.

**Important:** first completed does not necessarily mean first
successful.

A task that raises an exception has also completed.

You can cancel pending work:

``` python
for task in pending:
    task.cancel()
```

## Comparison

  ----------------------------------------------------------------------------
                      `gather()`        `as_completed()`   `wait()`
  ------------------- ----------------- ------------------ -------------------
  Main purpose        Collect results   Process as they    Inspect task state
                                        finish             

  Result order        Input order       Completion order   Task objects

  Early processing    No                Yes                Yes

  `FIRST_COMPLETED`   No                Natural completion Yes
                                        processing         

  Returns             List              Completion         `(done, pending)`
                                        iterator           
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

# 4.8 Timeouts & Cancellation

Timeouts prevent slow operations from consuming resources indefinitely.

## `asyncio.wait_for()`

``` python
result = await asyncio.wait_for(
    call_api(),
    timeout=3
)
```

If the operation doesn't finish within 3 seconds, `TimeoutError` is
raised.

### Cancellation Chain

``` text
Timeout
   ↓
wait_for() cancels underlying operation
   ↓
operation receives CancelledError
   ↓
operation performs cleanup and re-raises
   ↓
wait_for() raises TimeoutError to caller
```

Example:

``` python
async def call_api():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("API cancelled")
        raise

try:
    await asyncio.wait_for(
        call_api(),
        timeout=2
    )
except asyncio.TimeoutError:
    print("Timeout")
```

Output:

``` text
API cancelled
Timeout
```

### Important clarification

`CancelledError` is not magically converted by Python.

`wait_for()` internally handles the cancellation caused by its own
timeout and then raises `TimeoutError` to its caller.

So:

``` text
Inside call_api()
    → CancelledError

Outside wait_for()
    → TimeoutError
```

## Explicit Cancellation

``` python
task = asyncio.create_task(work())

task.cancel()
```

The coroutine can handle cancellation:

``` python
try:
    await work()
except asyncio.CancelledError:
    cleanup()
    raise
```

Generally, don't silently swallow `CancelledError`.

## `asyncio.timeout()`

Modern Python also supports:

``` python
async with asyncio.timeout(3):
    result = await call_api()
```

Useful for applying a timeout to a block/scope.

## Timeout + `gather()`

``` python
results = await asyncio.wait_for(
    asyncio.gather(
        call_db(),
        call_vector_db(),
        call_mcp()
    ),
    timeout=5
)
```

This establishes an overall deadline for the concurrent operation.

### Production principle

Avoid unnecessary work continuing after the caller has already timed
out.

------------------------------------------------------------------------

# 4.9 Async Synchronization

## `asyncio.Lock`

Protects shared state from concurrent coroutine access.

Example race:

``` python
counter = 0

async def increment():
    global counter

    current = counter
    await asyncio.sleep(0)
    counter = current + 1
```

Two coroutines can both read `0` before either writes.

Possible final result:

``` text
counter = 1
```

instead of:

``` text
counter = 2
```

## Fix with Lock

``` python
lock = asyncio.Lock()

async def increment():
    global counter

    async with lock:
        current = counter
        await asyncio.sleep(0)
        counter = current + 1
```

Only one Task can enter the critical section at a time.

## Critical Section

The protected code:

``` python
async with lock:
    ...
```

is the critical section.

Keep it as small as practical.

Avoid unnecessarily holding the lock during slow I/O:

``` python
# Usually bad
async with lock:
    await external_api()
    update_state()
```

Prefer:

``` python
result = await external_api()

async with lock:
    update_state(result)
```

if the API call doesn't require protection.

------------------------------------------------------------------------

## `asyncio.Semaphore`

Controls the maximum number of concurrent operations.

``` python
sem = asyncio.Semaphore(3)
```

Use:

``` python
async with sem:
    await call_api()
```

At most 3 tasks can be inside simultaneously.

### Example

10 calls, semaphore capacity 3, each call takes 2 seconds:

``` text
3 calls → 2s
3 calls → 2s
3 calls → 2s
1 call  → 2s

Total ≈ 8s
```

General approximation for equal-duration work:

``` text
ceil(N / concurrency_limit) × task_duration
```

### Important nuance

The semaphore doesn't necessarily create literal fixed batches.

As soon as one currently running operation finishes, another waiting
operation can acquire the released permit.

## Lock vs Semaphore

``` text
Lock
  → capacity 1
  → protect shared state

Semaphore(N)
  → capacity N
  → control concurrency
```

Typical Semaphore use cases:

-   External APIs
-   MCP calls
-   LLM calls
-   Vector DB queries
-   Database operations
-   Connection/resource limits

------------------------------------------------------------------------

## `asyncio.Event`

An Event is a shared signal/flag.

``` python
event = asyncio.Event()
```

Initially:

``` python
event.is_set()
# False
```

Wait:

``` python
await event.wait()
```

Signal:

``` python
event.set()
```

### Example: Model Initialization

``` python
model_ready = asyncio.Event()

async def request_handler():
    await model_ready.wait()
    return await model.generate(...)

async def load_model():
    await load_model_from_disk()
    model_ready.set()
```

Requests arriving before the model is ready wait.

When:

``` python
model_ready.set()
```

is called, waiting requests can proceed.

### Important

Once the Event is set, it stays set until:

``` python
event.clear()
```

Therefore:

``` python
await model_ready.wait()
```

returns immediately while the Event remains set.

## Event Mental Model

``` text
OFF
 ↓
wait() → blocked

set()
 ↓
ON
 ↓
waiting tasks wake
 ↓
future wait() → immediate

clear()
 ↓
OFF
 ↓
future wait() → blocked
```

------------------------------------------------------------------------

## `asyncio.Condition`

A Condition is useful when Tasks need to coordinate around **shared
state**.

Mental model:

``` text
Condition
   =
Lock + wait/notify mechanism
```

## Producer-Consumer Example

``` python
queue = []
condition = asyncio.Condition()
```

### Consumer

``` python
async def consumer():
    async with condition:

        while not queue:
            await condition.wait()

        item = queue.pop(0)

        print("Consumed:", item)
```

### Producer

``` python
async def producer(item):
    async with condition:

        queue.append(item)

        condition.notify()
```

### Flow

Initially:

``` text
queue = []

Consumer
   ↓
queue empty
   ↓
condition.wait()
   ↓
WAITING
```

Producer:

``` text
queue.append(job)
       ↓
condition.notify()
       ↓
consumer becomes runnable
```

When scheduled, the consumer resumes from the `await condition.wait()`
point.

It then checks:

``` python
while not queue:
```

again.

## Why `while`, not `if`?

Prefer:

``` python
while not queue:
    await condition.wait()
```

instead of:

``` python
if not queue:
    await condition.wait()
```

Because notification means:

> Something changed; check the condition again.

It does not guarantee that the desired state is still true when the
coroutine resumes.

## Condition Lifecycle

``` text
condition.wait()
      ↓
release lock + wait
      ↓
notify()
      ↓
become runnable
      ↓
event loop schedules coroutine
      ↓
re-acquire lock
      ↓
re-check condition
      ↓
continue OR wait again
```

A coroutine can therefore wake up and then wait again if the condition
is still false.

------------------------------------------------------------------------

# Event vs Condition

## Event

Use when you need a simple signal/state flag.

Example:

> "Wait until the model is ready."

``` python
await model_ready.wait()
```

## Condition

Use when tasks need to coordinate around shared state.

Example:

> "Wait until the job queue is non-empty."

``` python
while not queue:
    await condition.wait()
```

### Interview answer

> I would use an Event for a simple signal such as initialization
> completing. I would use a Condition when multiple tasks need to
> coordinate around shared mutable state and wait until a particular
> condition becomes true.

------------------------------------------------------------------------

# Interview Traps

## 1. Does `await` automatically mean concurrency?

No.

``` python
await a()
await b()
```

is sequential.

------------------------------------------------------------------------

## 2. Does asyncio mean parallelism?

No.

Asyncio primarily provides concurrent I/O.

CPU-heavy Python code can still block the event-loop thread.

------------------------------------------------------------------------

## 3. Does the GIL make threading useless?

No.

Threads are useful for blocking I/O.

The GIL primarily limits CPU-bound pure-Python execution within the same
interpreter.

------------------------------------------------------------------------

## 4. Does one event-loop thread mean no race conditions?

No.

Coroutines can interleave at `await` points.

Use `asyncio.Lock` when shared state needs protection.

------------------------------------------------------------------------

## 5. Does `create_task()` execute the coroutine immediately?

It schedules the coroutine as a Task.

The event loop executes it when it gets a chance.

------------------------------------------------------------------------

## 6. Does `gather()` return results in completion order?

No.

It preserves input order.

``` text
Completion:
B → C → A

gather():
[A, B, C]
```

------------------------------------------------------------------------

## 7. Does `FIRST_COMPLETED` mean first successful result?

No.

It means the first Task to finish.

A Task that raises an exception has also completed.

------------------------------------------------------------------------

## 8. Does `wait_for()` simply stop waiting?

No.

When its timeout expires, it cancels the underlying operation and
reports the timeout to its caller.

------------------------------------------------------------------------

## 9. Lock vs Semaphore

``` text
Lock       → one Task
Semaphore  → up to N Tasks
```

------------------------------------------------------------------------

## 10. Event vs Condition

``` text
Event
→ signal / flag

Condition
→ coordinate around shared state
```

------------------------------------------------------------------------

# AI/Backend Mental Model

A production AI backend may look conceptually like:

``` text
                    API Request
                         |
                         v
                 +---------------+
                 | Async Handler  |
                 +-------+-------+
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      PostgreSQL      Vector DB        MCP
          |              |              |
          +--------------+--------------+
                         |
                  asyncio.gather()
                         |
                   timeout/deadline
                         |
                  semaphore limits
                         |
                  result aggregation
```

Relevant asyncio tools:

``` text
asyncio
  |
  +-- gather()       → concurrent operations
  +-- wait()         → task state / racing
  +-- as_completed() → process results as they finish
  +-- timeout()      → deadline
  +-- cancellation   → stop useless work
  +-- Semaphore      → bounded concurrency
  +-- Lock           → shared-state protection
  +-- Event          → readiness signal
  +-- Condition      → state-based coordination
```

------------------------------------------------------------------------

# One-Line Revision Sheet

``` text
Concurrency      → overlapping progress
Parallelism      → simultaneous execution

GIL              → one thread executes Python bytecode at a time per interpreter
Threading        → shared memory; useful for blocking I/O
Multiprocessing  → separate memory/interpreters; CPU parallelism

Coroutine        → async work
Task             → scheduled coroutine
Future           → placeholder for future result

await            → suspend current coroutine when awaiting
create_task()    → schedule coroutine as Task
gather()         → concurrent execution + results in input order
as_completed()   → process results in completion order
wait()           → get done/pending Tasks

wait_for()       → timeout + cancellation of underlying awaitable
CancelledError   → cancellation request
TimeoutError     → deadline exceeded

Lock             → one Task in critical section
Semaphore(N)     → max N concurrent Tasks
Event            → readiness signal
Condition        → wait/notify around shared state
```

# High-Value Backend Pattern

For independent downstream I/O:

``` python
async def handle_request():
    async with semaphore:
        results = await asyncio.gather(
            query_db(),
            search_vector_db(),
            call_mcp()
        )

    return results
```

For a deadline:

``` python
async def handle_request():
    async with asyncio.timeout(5):
        return await asyncio.gather(
            query_db(),
            search_vector_db(),
            call_mcp()
        )
```

A production implementation will additionally need appropriate:

-   Timeouts
-   Cancellation handling
-   Concurrency limits
-   Connection pools
-   Rate limits
-   Error handling
-   Backpressure
-   Observability
