# Smart Session Engine

A session-based Python engine for session management, task
scheduling, and human-like behaviour simulation. It shows how to run
long-lived sessions safely: one session per IP, proxy rotation with
cooldowns, retries with backoff, and a small learning component that
picks the best timing strategy.

**This is a design showcase, not a code release.** This repository
documents the architecture, the components, and the testing of the
Smart Session Engine. The full source code is not published here yet.

**This is not a streaming automation tool.** It does not log in to,
play, or automate Spotify, Apple Music, or any music streaming
service. It has no account handling and no credentials for any such
service. The only clients are in-memory mocks and a local-page browser
client that only opens localhost or `file://` URLs. It exists to
demonstrate the engineering patterns, not to fake plays or streams.

## Architecture

The engine is built from small components with one job each:

```
core/task_scheduler.py      TaskScheduler: sessions, one session per IP
behavior/simulation_engine.py  BehaviorSimulationEngine: timing, skip
                              patterns, epsilon-greedy bandit
platforms/base.py           PlatformClient (abstract base), two mock
                              clients, and a factory
platforms/browser.py        LocalBrowserClient: Playwright, local URLs only
devices/rotation.py         ProxyManager: rotation, cooldowns, strict mode
utils/retry.py              @retry(tries=3, delay=1, backoff=2)
utils/config.py             YAML config loader
config.yaml                   engine, proxy, behaviour, and retry settings
tests/                        pytest suite, including bug story B
```

- **TaskScheduler** starts sessions, binds each one to a single proxy
  (one IP), runs tasks through a platform client, and records simple
  metrics (sessions started and failed, tasks completed, skipped, and
  failed). It refuses to start a second session on an IP that is
  already in use.
- **BehaviorSimulationEngine** waits a human-like amount of time
  (lognormal delays) and sometimes skips a task instead of completing
  it. An epsilon-greedy bandit learns which timing strategy (patient,
  balanced, quick) earns the best outcome, so the engine improves its
  choice over time.
- **PlatformClient** is one abstract base class. `MockMediaClient` and
  `MockTaskClient` implement it with canned in-memory work, and
  `PlatformClientFactory` builds a client by name. A new source only
  needs a new client class and one factory entry.
- **ProxyManager** hands out proxies that are not cooling down and, in
  strict mode, not already in use. A proxy that fails is cooled down
  before it can be used again.
- **Retry** is a decorator with exponential backoff: `@retry(tries=3,
  delay=1, backoff=2)`.

See `docs/ARCHITECTURE.md` for a deeper walk through the design,
including the TaskScheduler class, why the engine uses composition
over inheritance, and how the components are tested.

## Testing

A pytest suite covers the engine: 26 tests passed, 2 browser tests
skipped because Playwright browsers were not installed in the test
environment. The suite includes an integration test for the session
conflict fix described below.

## Interview talking points

**Problem.** Long-lived sessions fail in many ways: a login expires, a
proxy dies, a page changes. Running several sessions at once makes it
worse, because two sessions can end up sharing one IP and both fail.

**Built.** A session-based engine in Python: a TaskScheduler plus a
behaviour engine plus platform clients on one abstract base class, in
Docker, with a retry decorator with backoff, proxy rotation with
cooldowns, and one session per IP. A small bandit learns which timing
strategy works best. The whole thing is covered by a pytest suite.

**Hard part.** Resilience: making failure a normal path. Two sessions
shared one IP and both failed. The fix was to enforce one session per
IP with strict proxy mode, and to cool down failed proxies before they
are used again. That rule is now enforced in two places, the proxy
manager and the scheduler, and it is covered by an integration test.
