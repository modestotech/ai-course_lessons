# Celery: Architecture, Layers and Technical Debt

Celery is a distributed task queue for Python. This checkout is **5.7.0b1**, requires Python ≥3.10, and is BSD-licensed. An application publishes task messages to a broker such as RabbitMQ or Redis. Worker processes consume and run those tasks, and can optionally write the results to a result backend. The package is about 46k lines of Python in `celery/`, with tests in `t/`.

## Architecture in layers

```
┌───────────────────────────────────────────────────────────────┐
│ 6. CLI / entry points     celery/bin, celery/apps             │  click commands: worker, beat, multi, inspect…
├───────────────────────────────────────────────────────────────┤
│ 5. Runtime services       celery/worker, celery/beat.py,      │  long-running processes
│                           celery/events                       │
├───────────────────────────────────────────────────────────────┤
│ 4. Concurrency            celery/concurrency                  │  prefork / threads / gevent / eventlet / solo
├───────────────────────────────────────────────────────────────┤
│ 3. Workflow & results     celery/canvas.py, celery/result.py, │  chain/group/chord, AsyncResult,
│                           celery/backends                     │  ~18 result-store backends
├───────────────────────────────────────────────────────────────┤
│ 2. Application core       celery/app (Celery, Task, amqp,     │  config, registry, routing, tracing
│                           trace, routes, defaults, control)   │
├───────────────────────────────────────────────────────────────┤
│ 1. Foundations            celery/utils, local.py, _state.py,  │  proxies, global state, OS/daemon,
│                           platforms.py, bootsteps.py, signals │  dependency-graph startup
├───────────────────────────────────────────────────────────────┤
│ 0. External libs          kombu (messaging), billiard (mp     │  sister projects, maintained by the
│                           fork), vine (promises), click       │  same org
└───────────────────────────────────────────────────────────────┘
```

### 0. External libraries

Celery doesn't talk to brokers directly. All transport work (AMQP, Redis, SQS and so on) goes through **kombu**. **billiard** is Celery's own fork of `multiprocessing`, and the prefork pool depends on it. **vine** supplies the promise and `barrier` primitives used for async result handling. This split is the most important boundary in the design: Celery handles task semantics and kombu handles the wire.

### 1. Foundations

- **`local.py`**: `Proxy` and `PromiseProxy` objects that resolve lazily. `current_app` and `current_task`, and much of the public API, are built on them.
- **`_state.py`**: process-global and thread-local state, such as the default app, the current app and the task stack.
- **`bootsteps.py`**: a small dependency-graph framework. Worker components declare `requires=` and are started and stopped in topological order. This is the main extension point inside the worker.
- **`platforms.py`**: daemonization, pidfiles, signals, and switching uid/gid.
- **`signals.py`**: hooks like `task_prerun` and `worker_ready` that let users extend Celery without subclassing.

### 2. Application core (`celery/app`)

- **`base.py` → `Celery`**: the central object, holding configuration, the task registry and lazy finalization. `@app.task` creates a `Task` subclass **bound to the app** (`app.Task` is generated per app via `subclass_with_self`).
- **`task.py`**: the `Task` class, with `delay`, `apply_async`, `retry` and the `Request` context.
- **`amqp.py`**: turns a call into a protocol v2 message (headers, body, routing).
- **`routes.py`**: decides which queue a task goes to.
- **`trace.py`**: the **hot path on the worker side**. `build_tracer` produces a closure that runs the task and handles state transitions, signals, result storage, callbacks and errbacks. It is heavily optimized and hard to read.
- **`control.py`**: remote control and inspection (`inspect active`, `revoke` and so on) over broadcast "pidbox" queues.

### 3. Workflows and results

- **`canvas.py`** (2,600 lines, the largest module): `Signature`, `chain`, `group`, `chord`, `chunks`, and the stamping API. These are dict subclasses, so they serialize directly into messages.
- **`backends/`**: `base.py` defines `BaseBackend`, `KeyValueStoreBackend` and `BaseKeyValueStoreBackend`. About 18 implementations sit on top (Redis, database/SQLAlchemy, RPC, Mongo, S3, DynamoDB, Cassandra, Elasticsearch and others). `asynchronous.py` adds push-based result consumption for the Redis and RPC backends.
- **`result.py`**: `AsyncResult` and `GroupResult`, the client-side handles.

### 4. Concurrency (`celery/concurrency`)

These are pluggable execution pools behind the `BasePool` interface. `prefork.py` with **`asynpool.py`** is the default and the most complex, at about 1,500 lines. It does event-loop-driven I/O with child processes over pipes, built on billiard. The other pools are `thread`, `gevent`, `eventlet` and `solo`.

### 5. Runtime services

- **`worker/worker.py`** is `WorkController`, assembled from bootsteps.
- **`worker/consumer/`** is the `Consumer` blueprint, also bootsteps: connection, mingle, gossip, heartbeat, events, tasks, control and delayed delivery. It receives messages, and **`strategy.py`** turns them into **`request.py` → `Request`**, which goes to the pool.
- **`worker/loops.py`** runs the async event loop (kombu's `Hub`) or a synchronous fallback.
- **`beat.py`** is the periodic scheduler. It keeps a shelve-backed schedule and gets cron and solar schedules from `schedules.py`.
- **`events/`** covers the event dispatcher and receiver, the in-memory cluster `State`, and a curses monitor.

### 6. Entry points

`bin/` holds the click-based CLI. `apps/worker.py`, `apps/beat.py` and `apps/multi.py` are the process "applications" that the CLI launches.

### End-to-end flow of a task

1. `task.delay()` → `app.send_task`
2. `amqp.as_task_v2` builds the message
3. kombu `Producer.publish` → broker
4. The worker's `Consumer` receives it → `strategy` → `Request`
5. `pool.apply_async` → the child process runs `trace.build_tracer(...)`
6. `backend.store_result`
7. The client calls `AsyncResult.get()`

## Technical debt

### 1. Python 2 and Celery 3 compatibility leftovers

- There are 74 TODO/FIXME/XXX markers in `celery/`, and many are `# XXX compat` aliases: `subtask = signature` and `maybe_subtask` (`celery/canvas.py:2581`, `:2611`), `BaseTask = Task` (`celery/app/task.py:1290`), `subtask_from_request` (`celery/app/task.py:765`), `RetryTaskError` and `SystemTerminate` (`celery/exceptions.py`), `PIDFile` (`celery/platforms.py:250`), and the `publisher` property (`celery/events/dispatcher.py:262`).
- Several say "Remove in Celery 6.0" (`celery/apps/worker.py:153`, `celery/app/utils.py:208`).
- There are about 31 deprecation-warning sites.
- Some exceptions are labelled `# XXX Unused` (`celery/exceptions.py:241`).
- Old-style config names are still mapped to new ones in `app/defaults.py` and `app/utils.py`.

### 2. Canvas complexity

`canvas.py` is the debt hotspot, with 12 markers:

- Seven methods repeat the comment "XXX chord is also a class in outer scope" because the `chord` and `_chord` names shadow each other.
- Other comments read like open questions from the maintainers:
  - "TODO figure out why we are always cloning before freeze" (`canvas.py:1153`)
  - "TODO why isn't this asserting is_last_task == False?" (`canvas.py:1312`)
  - "Not sure if this is still used anywhere… Consider removing" (`canvas.py:1716`)
- The chord, group and chain interactions, chord upgrades and stamping are where most long-standing bug reports come from.

### 3. Global and implicit state

- `_state.py`, `current_app` proxies and lazy `PromiseProxy` objects make the import order and the question "which app am I using?" implicit. About 21 modules depend on the current or default app.
- Because `Task` classes are generated per app, introspection and type checking are hard, and so is testing in isolation.
- Pickle-era design choices, such as signatures being `dict` subclasses, limit how far the API can be refactored.

### 4. Very large, multi-purpose modules

| File | Lines |
|---|---|
| `canvas.py` | 2,611 |
| `app/base.py` | 1,736 |
| `concurrency/asynpool.py` | 1,489 |
| `backends/base.py` | 1,442 |
| `app/task.py` | 1,290 |
| `result.py` | 1,166 |
| `beat.py` | 1,147 |

`asynpool.py` and `app/trace.py` are written for speed (local-variable caching, closures, manual fd juggling), so they are fragile to change.

### 5. Typing is minimal

mypy runs only on about 9 files or dirs (`pyproject.toml`, `[tool.mypy] files`: states, signals, fixups, schedules and a few others) with `strict = false` and `follow_imports = "skip"`. The core (app, canvas, worker, backends) is effectively untyped.

### 6. A wide integration surface with uneven maintenance

- There are about 18 result backends. Several are niche (ArangoDB, Couchbase, CouchDB, Consul, CosmosDB) and get little use or CI.
- Four concurrency models are supported, and eventlet is largely abandoned upstream.
- A lot of behaviour depends on the broker (Redis vs AMQP visibility timeouts, for example), and this is spread across Celery and kombu.

### 7. Coupling to sister projects

- Features often require coordinated releases of kombu, billiard and vine. This release pins `kombu>=5.7.0b1`.
- billiard is a long-lived fork of the standard library's `multiprocessing` and has to be maintained separately.

### 8. Smaller items

- The repo `TODO` file just points to GitHub issues.
- There are about 15 `global` statements and about 25 `noqa`/`type: ignore` suppressions.
- Config is only partly applied in places, for example "XXX will not show finalized configuration" (`app/base.py:277`).

## Where to look first for refactoring value

1. **`canvas.py`**: fix the shadowed `chord` names, resolve the open TODOs, and remove dead code.
2. **Compat aliases**: delete the ones scheduled for removal in Celery 6.0.
3. **Typing**: extend mypy coverage to `app/` and `result.py`.
