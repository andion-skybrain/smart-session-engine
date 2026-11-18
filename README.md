# Smart Session Engine

A session-based Python engine for session management, task
scheduling, and behaviour simulation. It shows how to run
long-lived sessions safely: one session per IP, proxy rotation with
cooldowns, retries with backoff, and a small learning component that
picks the best timing strategy.

**This is a design showcase, not a code release.** This repository
documents the architecture and the components of the Smart Session
Engine. The full source code is not published here yet.

The engine works with in-memory mock clients and a local-page
browser client that only opens localhost or `file://` URLs.

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
tests/                        pytest suite, including the session conflict test
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

## Testing

A pytest suite covers the engine: the scheduler, the proxy manager,
the behaviour engine, and the retry logic, including an integration
test for the one-session-per-IP rule.
