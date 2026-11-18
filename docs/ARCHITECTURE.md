# Smart Session Engine: architecture notes

This document describes the design of the Smart Session Engine in
more depth than the README. It is written from the working engine,
which is covered by a pytest suite (26 passed, 2 browser tests
skipped). The full source code is not published in this repository
yet. This is a design showcase only.

## Design goals

1. One session per IP. A session is one unit of work bound to one
   proxy, meaning one IP address. Two sessions must never share an IP.
2. Failure is a normal path. Proxies die, work items fail, sessions
   fail. Each failure is handled, recorded, and recovered from, not
   hidden.
3. Small components with one job each. The scheduler schedules, the
   proxy manager manages proxies, the client talks to a platform, the
   behaviour engine decides timing. Nothing does two jobs.
4. Testable without any real platform. Mock clients and an injectable
   clock mean the whole engine runs in tests with no network.

## TaskScheduler

`TaskScheduler` is the centre of the engine. It models the scheduling
of work across sessions.

A session is a small record: a session id, a proxy, a client, and
counters for tasks completed, skipped, and failed. The scheduler keeps
two maps, sessions by id and IPs by session, plus simple metrics
counters for sessions started and failed and tasks completed, skipped,
and failed.

Its methods:

- `start_session(session_id)` acquires a proxy from the proxy manager,
  refuses with a `SessionConflictError` if that IP is already in use,
  builds a client through the factory, connects it, and records the
  session.
- `run_task(session_id, behavior=None)` fetches one work item from the
  session client and submits the result. When a behaviour engine is
  given, it supplies the delay and the skip decision and learns from
  the outcome.
- `end_session(session_id)` closes the client and releases the proxy.
- `fail_session(session_id)` cools the proxy down through the proxy
  manager and closes the session.
- `get_metrics()` returns a copy of the counters.

The one-session-per-IP rule is enforced in two places: the proxy
manager in strict mode will not hand out an IP that is already in
use, and the scheduler checks again before starting a session. That
double check is deliberate. It is the fix for the hardest bug in the
project: two sessions shared one IP and both failed. The rule is
covered by an integration test.

## Composition over inheritance

The scheduler uses composition. It holds a proxy manager and a client
factory, and `run_task` takes a behaviour engine as an argument. Each
of those has its own responsibility and its own lifecycle, and each
can be swapped on its own. In tests, the factory builds mock clients,
so the scheduler can be tested without any real platform.

Inheritance would be the wrong tool here, because a scheduler is not
a kind of proxy manager or a kind of client. Inheriting from them
would tie unrelated responsibilities together and make each part
harder to test alone.

The one place inheritance is used is `SessionConflictError`, which
extends `RuntimeError`. That is a true is-a relationship, an error
type, not shared behaviour. The rule is simple: inherit for a shared
type, compose for shared work.

## PlatformClient and the factory

`PlatformClient` is one abstract base class that defines the client
interface: connect, fetch work, submit work, close. `MockMediaClient`
and `MockTaskClient` implement it with canned in-memory work, and
`PlatformClientFactory` builds a client by name.

This gives polymorphism: the scheduler calls the same methods no
matter which client it holds, so a new source only needs a new client
class and one factory entry. The sanitised version of the engine uses
only local clients, so nothing in it talks to a real external
platform. A separate local browser client opens only localhost or
`file://` URLs.

## ProxyManager

`ProxyManager` hands out proxies, rotates them, and tracks cooldowns.
In strict mode it will not hand out a proxy whose IP is already in
use. A proxy that fails is reported and cooled down before it can be
used again. The clock is injectable, so cooldowns can be tested
without waiting in real time. The proxies in the configuration are
test-net placeholders, not real proxies.

## BehaviorSimulationEngine

The behaviour engine decides how a session behaves: how long to wait
between tasks and whether to skip a task. It uses lognormal delays
for human-like timing and three strategies, patient, balanced, and
quick, each with its own delay and skip rate. An epsilon-greedy
bandit learns which strategy earns the best outcome and shifts
toward it over time. The engine records the delay instead of sleeping
through it, so tests stay fast.

## Retry and configuration

Retries use a decorator with exponential backoff,
`@retry(tries=3, delay=1, backoff=2)`. Settings for the engine, the
proxies, the behaviour, and the retries live in one YAML config file,
loaded by a small config module. The engine also runs in Docker.

## Honest limits

- The engine is a completed engineering project, not a production
  service. No deployment scale is claimed for it.
- The test figures describe the version that was tested: 26 passed,
  2 browser tests skipped because Playwright browsers were not
  installed in that environment.
- The sanitised version uses local and mock clients only. It
  demonstrates the engineering patterns, and it makes no claim about
  real-world behaviour on any external platform.
