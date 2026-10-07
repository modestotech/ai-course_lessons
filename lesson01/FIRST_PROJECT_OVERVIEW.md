# Celery: Project Overview

## What it is

This repository is **Celery**, an open-source **distributed task queue** for Python. You use it to run work outside the normal request/response cycle of an application, either in the background, on other machines, or on a schedule.

- **Version in this checkout:** `5.7.0b1` (codename *Collider*), a beta release
- **Python support:** 3.10+ (`setup.py` → `python_requires=">=3.10"`)
- **License:** BSD 3-Clause
- **Upstream:** https://github.com/celery/celery · Docs: https://docs.celeryq.dev/

## The core idea

```
 ┌──────────┐   task message   ┌──────────┐   delivers   ┌──────────┐
 │  Client  │ ───────────────▶ │  Broker  │ ───────────▶ │  Worker  │
 │ (your app)│                 │(RabbitMQ,│              │ (celery  │
 └──────────┘                  │ Redis…)  │              │  worker) │
      ▲                        └──────────┘              └────┬─────┘
      │                 result (optional)                      │
      └──────────────────── Result backend ◀───────────────────┘
                           (Redis, DB, S3…)
```

1. Your application defines **tasks**, which are ordinary Python functions decorated with `@app.task`.
2. When you call `task.delay(...)`, Celery serializes the call into a **message** and sends it to a **broker** (usually RabbitMQ or Redis).
3. One or more **worker** processes take messages from the broker and run the tasks.
4. If you need the return value, the worker can save it in a **result backend**, where the client reads it through an `AsyncResult`.

Because workers and brokers can be scaled independently, Celery supports horizontal scaling, high availability and retries.

### Minimal example

```python
from celery import Celery

app = Celery('hello', broker='amqp://guest@localhost//')

@app.task
def hello():
    return 'hello world'
```

```bash
celery -A hello worker --loglevel=INFO   # start a worker
```

```python
hello.delay()                            # enqueue from your app
```

## Key features

| Area | What Celery offers |
|---|---|
| **Brokers (transports)** | RabbitMQ, Redis, Amazon SQS, Google Pub/Sub, and more through [Kombu](https://github.com/celery/kombu) |
| **Concurrency pools** | prefork (multiprocessing), threads, eventlet, gevent, solo |
| **Result backends** | Redis, RPC/AMQP, SQLAlchemy, Django ORM, MongoDB, Cassandra, Elasticsearch, DynamoDB, S3, GCS, Azure, Couchbase, CouchDB, ArangoDB, Consul, memcached, filesystem |
| **Workflows ("canvas")** | `chain`, `group`, `chord`, `chunks`, `signature` for composing tasks |
| **Scheduling** | `celery beat` for periodic tasks (intervals, crontab, solar) |
| **Reliability** | Retries, autoretry, acks-late, time limits, rate limits |
| **Monitoring** | Event stream, remote control (`inspect`, `control`), works with tools such as Flower |
| **Serialization** | JSON, pickle, YAML, msgpack, plus optional compression and message signing |
| **Framework integration** | Django fixups, Pydantic support for task arguments |

## Repository layout

```
celery/            The library itself
├── app/           The Celery app object, Task base class, config defaults, routing, tracing
├── worker/        Worker implementation: consumer, request handling, autoscaling, heartbeats
├── concurrency/   Pool implementations (prefork/asynpool, thread, eventlet, gevent, solo)
├── backends/      Result backend implementations (one module per store)
├── bin/           CLI commands: `celery worker`, `beat`, `inspect`, `call`, `purge`, `multi`, ...
├── events/        Event dispatch/receiving and the curses monitor
├── security/      Message signing and serializer security
├── fixups/        Framework integrations (e.g. Django)
├── contrib/       Extras: pytest plugin, testing helpers, migration tools, abortable tasks
├── utils/         Shared helpers
├── canvas.py      Workflow primitives (chain/group/chord/signature)
├── beat.py        Periodic-task scheduler
├── schedules.py   crontab / interval / solar schedule types
├── result.py      AsyncResult, GroupResult
├── bootsteps.py   Dependency-ordered startup/shutdown system used by the worker
└── signals.py     Hooks such as task_prerun, task_success, worker_ready

t/                 Test suite
├── unit/          Fast, isolated unit tests
├── integration/   Tests against real brokers and backends
├── smoke/         End-to-end smoke tests (pytest-celery / containers)
└── benchmarks/    Performance benchmarks

docs/              Sphinx documentation (the source for docs.celeryq.dev)
examples/          Example projects: Django, eventlet/gevent, periodic tasks, quorum queues, security, stamping, ...
docker/            Docker dev environment (Dockerfile + docker-compose)
helm-chart/        Helm chart for deploying Celery workers on Kubernetes
requirements/      Pinned dependency sets (default, test, docs, and extras per backend)
Changelog.rst      Release notes
```

## How the pieces fit together

- **`Celery` app** (`celery/app/base.py`): the entry point. It holds the configuration, the task registry and the connections to the broker and backend.
- **Tasks** (`celery/app/task.py`): the `Task` class wraps your function and provides `delay()`, `apply_async()`, `retry()` and other methods.
- **Messaging**: Celery uses **Kombu**, a sister project, for the transport layer, so every broker sits behind one shared interface.
- **Worker** (`celery/worker/`): built from *bootsteps*. The consumer receives messages, a strategy turns them into `Request` objects, and the execution pool runs them.
- **Beat** (`celery/beat.py`): a separate process that sends scheduled tasks to the broker when they are due.

## Typical use cases

- Sending email or notifications without blocking a web request
- Image and video processing, report generation, and data pipelines
- Calling slow or unreliable third-party APIs, with retries
- Periodic jobs that replace cron (cleanup, syncing, aggregation)
- Spreading CPU-heavy work across many machines

## Working on this repo

```bash
pip install -e . -r requirements/test.txt
pytest t/unit                    # run unit tests
tox                              # run the full test matrix
docker compose -f docker/docker-compose.yml up   # full dev environment
```

See `CONTRIBUTING.rst` for the contribution guidelines.
