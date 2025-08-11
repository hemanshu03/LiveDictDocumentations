# LiveDict - Release v1.0.0 Documentation

**Version:** 1.0.0 (release)

A compact, production-minded, in-memory encrypted ephemeral key-value store with optional persistence backends.

---

## Overview

LiveDict is a synchronous, TTL-based encrypted key-value store designed for speed, simplicity, and safe persistence. Its primary runtime store is memory (fast), with optional persistence into SQLite, filesystem-backed object files, or Redis. LiveDict uses AES-GCM when `cryptography` is available and provides a deterministic XOR+Base64 fallback for test or restricted environments.

Key design principles:

* Keep the in-memory store authoritative and tiny; persistence backends are mirrors for durability and recovery.
* Default behaviour: keys are expirable (TTL) unless explicitly set to never-expire.
* Efficient expiry scheduling using a min-heap and a background monitor thread that executes expiry hooks outside the lock.
* Safe backend interaction: persistence failures are non-fatal.

This document covers usage, API, configuration models, backends, and operational notes.

---

## Quick start

```python
from livedict import LiveDict

# simple default LiveDict
d = LiveDict()

# store a JSON-serializable object for 60 seconds
d.set("foo", {"x": 1}, ttl=60)

# get value back
print(d.get("foo"))  # -> {'x': 1}

# delete
d.delete("foo")

# stop background monitor when shutting down
d.stop()
```

For an asyncio-friendly wrapper use `AsyncLiveDict` which offloads calls to a threadpool:

```python
from livedict import AsyncLiveDict

ad = AsyncLiveDict()
await ad.set("k", {"v":1}, ttl=10)
val = await ad.get("k")
await ad.stop()
```

---

## Installation & Dependencies

LiveDict core is a pure-Python module with optional third-party dependencies for enhanced features.

Required:

* Python 3.8+
* `pydantic` (for configuration models and validation)

Optional (for production-grade features):

* `cryptography` — enables AES-GCM encryption (recommended). Without it, LiveDict falls back to a deterministic XOR+Base64 adapter (useful for tests but not secure for production).
* `PyYAML` — for YAML builder support.
* `redis` — if the Redis backend is to be used.

Install examples:

```bash
pip install livedict
```

---

## Main components

### CipherAdapter

* Purpose: pluggable encryption abstraction.
* Behavior: uses AES-GCM from `cryptography` when available; otherwise uses deterministic XOR + Base64.
* Important: to persist encrypted blobs across restarts and be able to decrypt them, use a stable `CipherAdapter` (same keys and `version_tag`). The encrypted blob format for AES-GCM is `version_tag || nonce(12) || ciphertext`.

API:

* `CipherAdapter(keys: Optional[List[bytes]] = None, version_tag: bytes = b"LD1")`
* `encrypt(plaintext: bytes) -> bytes`
* `decrypt(blob: bytes) -> Optional[bytes]`

Security note: the fallback XOR adapter is not secure — only suitable for tests.

### LiveDict (synchronous)

* Mapping-like API: supports `__getitem__`, `__setitem__`, `__delitem__`, `__contains__`.
* Core methods: `set`, `get`, `delete`, `keys`, `items`, `add_listener`, `stop`.
* TTL semantics: default TTL comes from `config.common.default_ttl`; non-expiring keys use a sentinel negative value.
* Hooks: `on_access` (runs on `get`) and `on_expire` (runs via monitor thread when key expires). Hooks run synchronously in the runtime thread — be careful to offload heavy work.
* Listeners: synchronous functions invoked after each `set()` with `("{bucket}:{key}", value)`.
* Limits: configurable `max_total_keys` and `max_keys_per_bucket` to avoid unbounded memory growth.

Important methods (short):

* `set(key, value, ttl=None, key_expirable=None, on_access=None, on_expire=None, bucket=None, backend=None, persist=False)`

  * `persist=True` and `backend` provided will mirror to the chosen backend (non-fatal if persistence fails).
* `get(key, token=None, bucket=None, backend=None)`

  * If not present in-memory and `backend` provided, LiveDict attempts to rehydrate from the backend.
* `delete(key, persist=False, backend=None)`
* `keys(bucket=None)`
* `items(bucket=None)` — enumerates values via `get()` (may execute `on_access` hooks).
* `stop()` — request monitor thread to stop and join briefly; call during shutdown.

Thread-safety: LiveDict uses an `RLock` plus a `Condition` for heap/monitor coordination. The monitor thread manages expiries and calls `on_expire` hooks outside the lock.

---

### Backends

Backends implement the `BackendBase` interface with `set`, `get`, `delete`, `keys`, `cleanup`.

Provided implementations:

* `InMemoryBackend` — lightweight dict used for tests or when in-memory persistence mirror is desired.
* `SQLiteBackend` — file-backed persistence using a short-lived sqlite3 connection per operation (avoids long-held file locks and works safely on Windows). Use DSN `sqlite:///path/to.db` or just a path. Table schema: `(bucket, key, value BLOB, expire_at REAL)`.
* `FileBackend` — stores each object as `<base_dir>/<bucket>/<key>.entry` where the first line is JSON metadata (`expire_at`) and subsequent bytes are ciphertext.
* `RedisBackend` — thin redis-py wrapper storing `bucket:key` with TTL; requires `redis` package.

Persistence behavior notes:

* When `persist=True` on `set()` or `delete()` the backend operation is attempted but failures are intentionally non-fatal.
* `SQLiteBackend.get()` includes a tolerant fallback: if a (bucket,key) pair is missing it will search other buckets for the most recent non-expired row for the same key.

---

## Configuration models (Pydantic)

Key models exposed:

* `BucketPolicyModel` — controls whether per-key buckets are used, default bucket behaviour and enforcement.
* `InMemoryBackendModel` — limits and mirror settings.
* `SQLBackendModel`, `RedisBackendModel`, `NoSQLBackendModel`, `ExternalObjectStoreModel` — backend-specific options.
* `StorageBackendsModel` — aggregate storage config.
* `CommonModel` — common options like `default_ttl`, `serializer`, `encryption_enabled`, `encryption_version_tag`.
* `MonitorModel` — tuning for heap rebuilds and monitor.
* `LiveDictConfig` — root config.

Builders:

* `LiveDictJSONBuilder` — load from JSON path/string/dict.
* `LiveDictYAMLBuilder` — load from YAML string/path (requires PyYAML).
* `LiveDictPresetBuilder` — small presets (`simple`, `bucketed`, `durable_sql`, `hybrid_cache_db`).
* `LiveDictEnvBuilder` — build from `LIVEDICT_` prefixed environment variables.
* `LiveDictFluentBuilder` — programmatic DSL to construct a config.
* `LiveDictInteractiveBuilder` — interactive CLI wizard (stdin) used in interactive contexts.

Factory helpers:

* `LiveDict.from_config(cfg, cipher=None, extra_backends=None)`
* `LiveDict.from_preset(name)`
* Convenience top-level functions: `from_preset`, `from_json`, `from_yaml`.

---

## Async wrapper (AsyncLiveDict)

`AsyncLiveDict` wraps the synchronous `LiveDict` by offloading calls to an executor via `loop.run_in_executor`.

* Use when you want an asyncio-friendly API but accept that the store remains thread-backed and that callbacks are still synchronous.
* Pass a custom executor to control concurrency (recommended in high-load situations).
* Cancellation: canceling the awaiting coroutine does not cancel the underlying thread work.

---

## Hooks and async coordination

* `on_expire` and `on_access` run synchronously in the monitor thread or caller thread. If you need them to perform async work, schedule coroutines via `loop.call_soon_threadsafe(asyncio.create_task, coro())` from the hook.
* Avoid long-running/blocking logic inside hooks — they run inside monitor or caller thread and can block expiry processing.

---

## Limits, monitoring and heap maintenance

* Expiries are scheduled on a min-heap `(expire_at, id)`. Each stored `Entry` has a numeric id to handle duplicates and stale heap entries.
* The monitor thread wakes for the nearest expiry and executes hook callbacks outside the lock.
* The heap can be rebuilt when the number of stale heap entries exceeds a threshold controlled by `config.monitor.heap_rebuild_ratio`.

---

## Operational notes & best practices

* Use a stable `CipherAdapter` when persisting to disk across restarts (same key(s) and `version_tag`).
* Prefer `cryptography` in production. The fallback adapter is for tests only.
* For multi-process durability use `SQLiteBackend` or a shared object store; remember LiveDict's in-memory store is authoritative until mirrored.
* Limit key counts with `max_total_keys` and `max_keys_per_bucket` to avoid memory exhaustion.
* Call `stop()` on shutdown to cleanly stop the monitor thread.

---

## Troubleshooting

* Missing `pydantic` -> ImportError at module init: `pip install pydantic`.
* Decryption fails after restart -> ensure `CipherAdapter` keys and `version_tag` match those used when data was encrypted.
* Monitor thread not stopping -> call `stop()` and ensure program waits briefly; monitor joins with a short timeout.

---

## Example: durable configuration (SQLite mirror)

```python
from livedict import LiveDict, LiveDictFluentBuilder

cfg = (
    LiveDictFluentBuilder()
    .preset("durable_sql")
    .with_sql("sqlite:///./data/livedict.db")
    .with_defaults(ttl=300)
    .preview()
)

ld = LiveDict.from_config(cfg)
ld.set("session:123", {"user_id": 42}, ttl=600, persist=True, backend="sql")
```

---

## Changelog for Release v1.0.0

* Initial release of LiveDict core module.
* AES-GCM encryption with deterministic fallback.
* SQLite, file, Redis and in-memory backends.
* Heap-based expiry monitor with on\_expire/on\_access hooks.
* Async wrapper `AsyncLiveDict`.
* Pydantic-based configuration models and multiple builder helpers.

---

## License & authorship

MIT License. © 2025 LiveDict. All rights reserved.

---

*End of release documentation for release version 1.0.0.*



