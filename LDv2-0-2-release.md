# LiveDict v2.0.2 Documentation

**TTL-based, Persistent / Ephemeral Python Dictionary with Hook Callbacks and Sandbox**

LiveDict is a flexible key-value store that provides **synchronous and asynchronous interfaces**, optional persistence, **sandboxed execution for callbacks**, and multiple backend options (memory, SQLite, Redis).

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Key Features](#key-features)
4. [Backends](#backends)
5. [API Usage](#api-usage)

   * [Synchronous LiveDict](#synchronous-livedict)
   * [Asynchronous LiveDict](#asynchronous-livedict)
   * [TTL / Expiry](#ttl--expiry)
   * [Callbacks](#callbacks)
   * [Sandbox](#sandbox)
   * [Per-Key Locking](#per-key-locking)
6. [Example Usage](#example-usage)
7. [Changelog](#changelog)
8. [Contributing](#contributing)
9. [License](#license)

---

## Overview

LiveDict provides a **dictionary-like interface** in Python with TTL, optional persistence, callback hooks, and sandboxing support.

* Works in **synchronous** and **asynchronous** modes.
* Backends supported: **memory**, **SQLite**, **Redis**.
* Sandbox ensures **callback safety** and prevents untrusted code from crashing the main store.

---

## Installation

Minimum supported Python: **3.8+**

```bash
pip install livedict
```

---

## Key Features

* TTL-based expiry of keys with **automatic removal**.
* **Hook callbacks** for `set`, `expire`, and `access` events.
* Sandbox module to safely execute **user-provided callbacks**.
* Per-key **locking** for safe concurrent access.
* **Multiple backends**: memory, SQLite, Redis.
* **Synchronous & Asynchronous** APIs.
* Callbacks now run correctly with **no duplication or re-entrancy** issues (fixed in v2.0.2).

---

## Backends

| Backend | Mode Supported | Notes                                      |
| ------- | -------------- | ------------------------------------------ |
| memory  | sync/async     | In-memory only; fast and ephemeral         |
| SQLite  | sync/async     | Optional file-backed persistence           |
| Redis   | sync/async     | Requires Redis server; async uses executor |

---

## API Usage

### Synchronous LiveDict

```python
from livedict import LiveDict

live = LiveDict(work_mode='sync', backend='memory')
live.set('foo', 123)
print(live.get('foo'))  # 123
live.delete('foo')
```

---

### Asynchronous LiveDict

```python
import asyncio
from livedict import LiveDict

async def main():
    live = LiveDict(work_mode='async', backend='memory')
    await live.set('bar', 'hello')
    print(await live.get('bar'))

asyncio.run(main())
```

---

### TTL / Expiry

```python
import time
from livedict import LiveDict

live = LiveDict(work_mode='sync', backend='memory')
live.set('temp', 'expire-me', ttl=2)
print(live.get('temp'))  # 'expire-me'
time.sleep(3)
print(live.get('temp'))  # None
```

---

### Callbacks

Callbacks are executed **on events** like `set` or `expire`.

```python
def on_set(key, val):
    print(f'Set {key} = {val}')

async def on_expire(key, val):
    print(f'{key} expired')

cb1 = live.register_callback('set', on_set)
cb2 = live.register_callback('expire', on_expire, is_async=True)
```

* **Disable/Enable callback**:
  `live.set_callback_enabled(cb1, False)`
* **Unregister callback**:
  `live.unregister_callback(cb2)`

#### Async multiple callbacks workaround

```python
async def main_callback(key, val):
    await asyncio.gather(async_cb1(key, val), async_cb2(key, val))

live.register_callback('expire', main_callback, is_async=True)
```

> **Note (v2.0.2):** Fixed duplicate callback invocation caused by redundant async loop handling in `_trigger_event`.

---

### Sandbox

Sandboxed execution prevents **exceptions or slow callbacks** from affecting the main store.

```python
import time
from livedict import LiveDict

def bad_callback(key, val):
    raise RuntimeError('Oops!')

def slow_callback(key, val):
    time.sleep(3)

live = LiveDict(work_mode='sync', backend='memory')
live.register_callback('set', bad_callback)
live.register_callback('set', slow_callback, timeout=1.0)
live.set('x', 99)  # Exceptions handled safely
```

---

### Per-Key Locking

```python
from livedict import LiveDict

live = LiveDict(work_mode='sync', backend='memory')
live.set('counter', 1)

if live.lock('counter', timeout=1):
    try:
        val = live.get('counter')
        live.set('counter', val + 1)
    finally:
        live.unlock('counter')

print(live.get('counter'))  # Updated safely
```

---

## Example Usage

See `test_sync.py` and `test_async.py` in the repository for complete demonstrations of:

* Sync & Async usage
* TTL / Expiry
* Callback behavior (with sandbox)
* Locking & concurrency
* SQLite & Redis backend examples

---

## Changelog

### [2.0.2] – 2025-10-05

**Changed**

* Fixed duplicate callback invocation caused by redundant async retry handling in `_trigger_event`.

---

### [2.0.1] – 2025-10-05

**Changed**

* Removed incorrect dependency `sqlite3` from `setup.py`.
  (SQLite is bundled with Python ≥ 3.6.)

---

### [2.0.0] – 2025-10-04

**Added**

* Comprehensive Google-style docstrings across all modules.
* Sandbox functionality is active and maintained within the package.

**Changed**

* Version bumped to `2.0.0`.
* Packaging files updated for PyPI release.

**Removed**

* Cryptography-related code removed.
  Encryption and signing must now be handled externally.

**Notes**

* Sandbox behavior is active; users should test under their own runtime conditions.

---

## Contributing

* Follow **Google-style docstrings**.
* Run `pdoc` or Sphinx (with napoleon) to generate docs.
* Submit issues or PRs on GitHub.

---

## License

GNU Affero General Public License v3.0 © 2025 Hemanshu Vaidya  
[![GitHub license](https://img.shields.io/badge/license-AGPL--3.0-blue.svg)](https://github.com/hemanshu03/LiveDict/blob/main/LICENSE)

> **LiveDict** – TTL-based Python key-value store by [hemanshu03](https://github.com/hemanshu03)
