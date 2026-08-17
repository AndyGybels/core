# Rust Recorder Walking Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A custom integration that shadows Home Assistant's built-in `recorder` and routes the states write path (`states`, `states_meta`, `state_attributes` tables) through a Rust engine, with everything else inherited from the stock Python `Recorder`.

**Architecture:** A new standalone project (`ha-rust-recorder`) containing (a) a PyO3/maturin Rust extension `ha_recorder_engine` with a dedicated SQLite writer thread, and (b) `custom_components/recorder/` defining `RustRecorder(Recorder)` — a subclass of the built-in recorder that overrides only `_process_state_changed_event_into_session` (enqueue to Rust) and `_commit_event_session_or_retry` (flush Rust after the Python commit). Shadowing only changes which integration is *set up*; `homeassistant.components.recorder` imports still resolve to the built-in package, so all consumers (`get_instance`, history, statistics, logbook, `sql`) keep working unchanged against the same SQLite file (WAL mode). Attribute serialization and hashing stay in Python (orjson + fnv, already C-speed) so dedup stays byte-compatible; Rust receives ready-to-insert primitives.

**Tech Stack:** Rust (stable), PyO3 with abi3, maturin, rusqlite (bundled), lru crate; Python 3.13+, `homeassistant` + `pytest-homeassistant-custom-component` as dev dependencies.

## Global Constraints

- New project directory: `/Users/andygybels/Projects/ha-rust-recorder` (NOT inside the core-hass checkout; the spec forbids forking core).
- Supported database: SQLite only. On a non-SQLite `db_url`, setup must delegate to the stock recorder (spec: "detects a non-SQLite db_url and declines to set up").
- Supported schema version: exactly `53` (from `homeassistant/components/recorder/db_schema.py::SCHEMA_VERSION` at time of writing). On mismatch, delegate to the stock recorder.
- Pin dev dependencies to the HA version being validated against: `homeassistant==2026.8.*`.
- Migrations, non-state events, purge scheduling, statistics generation, and all queries remain the inherited Python implementation in this plan.
- The custom integration's manifest MUST contain a `version` key (HA requires it for custom integrations).
- Every Python test function parameter is type-annotated with concrete types (`HomeAssistant`, not `Any`).
- Commit after every task; conventional-commit style messages (`feat:`, `test:`, `chore:`).

---

### Task 1: Project scaffold with a callable Rust extension

**Files:**
- Create: `pyproject.toml`, `Cargo.toml`, `rust/src/lib.rs`, `.gitignore`, `tests/__init__.py`, `tests/test_import.py`

**Interfaces:**
- Produces: importable module `ha_recorder_engine` with `engine_version() -> str`.

- [ ] **Step 1: Initialize project and git**

```bash
mkdir -p /Users/andygybels/Projects/ha-rust-recorder && cd /Users/andygybels/Projects/ha-rust-recorder
git init -b main
uv venv --python 3.13 .venv
printf '.venv/\ntarget/\n__pycache__/\n*.egg-info/\ndist/\n' > .gitignore
```

- [ ] **Step 2: Write build configuration**

`pyproject.toml`:
```toml
[build-system]
requires = ["maturin>=1.7,<2.0"]
build-backend = "maturin"

[project]
name = "ha-recorder-engine"
version = "0.1.0"
requires-python = ">=3.13"

[tool.maturin]
module-name = "ha_recorder_engine"
manifest-path = "Cargo.toml"

[dependency-groups]
dev = [
    "homeassistant==2026.8.*",
    "pytest-homeassistant-custom-component",
    "pytest",
    "pytest-asyncio",
]
```

`Cargo.toml`:
```toml
[package]
name = "ha_recorder_engine"
version = "0.1.0"
edition = "2021"

[lib]
name = "ha_recorder_engine"
crate-type = ["cdylib", "rlib"]
path = "rust/src/lib.rs"

[dependencies]
pyo3 = { version = "0.23", features = ["abi3-py313"] }
rusqlite = { version = "0.32", features = ["bundled"] }
lru = "0.12"

[profile.release]
lto = true
```

- [ ] **Step 3: Write the failing test**

`tests/test_import.py`:
```python
"""Smoke test: the Rust extension builds and imports."""


def test_engine_version() -> None:
    import ha_recorder_engine

    assert ha_recorder_engine.engine_version() == "0.1.0"
```

- [ ] **Step 4: Run test to verify it fails**

Run: `uv run --group dev pytest tests/test_import.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ha_recorder_engine'`

- [ ] **Step 5: Write minimal implementation**

`rust/src/lib.rs`:
```rust
use pyo3::prelude::*;

#[pyfunction]
fn engine_version() -> &'static str {
    env!("CARGO_PKG_VERSION")
}

#[pymodule]
fn ha_recorder_engine(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(engine_version, m)?)?;
    Ok(())
}
```

- [ ] **Step 6: Build and run test to verify it passes**

Run: `uv run --group dev maturin develop && uv run --group dev pytest tests/test_import.py -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "chore: scaffold maturin/PyO3 project with importable engine module"
```

---

### Task 2: Pass-through custom component that shadows the built-in recorder

This is the riskiest assumption in the whole design (custom integration overrides built-in; consumers unaffected) — prove it before writing any engine code.

**Files:**
- Create: `custom_components/recorder/manifest.json`, `custom_components/recorder/__init__.py`, `custom_components/recorder/const.py`, `tests/conftest.py`, `tests/test_shadow.py`

**Interfaces:**
- Consumes: built-in `homeassistant.components.recorder` (`async_setup`, `get_instance`, `Recorder`, `DOMAIN`).
- Produces: a loadable custom `recorder` integration whose `async_setup` delegates to the built-in one; `const.py` exposes `ENGINE_SUPPORTED_SCHEMA_VERSION = 53`.

- [ ] **Step 1: Write the failing test**

`tests/conftest.py`:
```python
"""Shared fixtures. pytest-homeassistant-custom-component makes custom_components/ loadable."""

import pytest


@pytest.fixture(autouse=True)
def auto_enable_custom_integrations(enable_custom_integrations: None) -> None:
    """Enable loading custom integrations in all tests."""
```

`tests/test_shadow.py`:
```python
"""Prove custom_components/recorder shadows the built-in and stays API-compatible."""

from homeassistant.components import recorder
from homeassistant.core import HomeAssistant
from homeassistant.loader import async_get_integration
from homeassistant.setup import async_setup_component
from pytest_homeassistant_custom_component.components.recorder.common import (
    async_wait_recording_done,
)


async def test_custom_recorder_shadows_builtin(hass: HomeAssistant) -> None:
    integration = await async_get_integration(hass, recorder.DOMAIN)
    assert not integration.is_built_in

    assert await async_setup_component(
        hass, recorder.DOMAIN, {recorder.DOMAIN: {"db_url": "sqlite://"}}
    )
    await async_wait_recording_done(hass)

    hass.states.async_set("sensor.test", "42")
    await async_wait_recording_done(hass)
    assert recorder.get_instance(hass) is not None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run --group dev pytest tests/test_shadow.py -v`
Expected: FAIL — integration `recorder` not found as custom (no manifest yet).

- [ ] **Step 3: Write minimal implementation**

`custom_components/recorder/manifest.json`:
```json
{
  "domain": "recorder",
  "name": "Recorder (Rust engine)",
  "version": "0.1.0",
  "documentation": "https://github.com/andygybels/ha-rust-recorder",
  "codeowners": ["@andygybels"],
  "dependencies": ["http"],
  "iot_class": "local_push",
  "requirements": []
}
```

`custom_components/recorder/const.py`:
```python
"""Constants for the Rust-backed recorder."""

ENGINE_SUPPORTED_SCHEMA_VERSION = 53
```

`custom_components/recorder/__init__.py`:
```python
"""Rust-backed recorder: shadows the built-in recorder integration.

Imports of homeassistant.components.recorder still resolve to the built-in
package; this custom integration only takes over component *setup*.
For now it is a pure pass-through.
"""

from homeassistant.components.recorder import (  # noqa: F401  (re-export surface)
    CONFIG_SCHEMA,
    async_setup as _builtin_async_setup,
)
from homeassistant.core import HomeAssistant
from homeassistant.helpers.typing import ConfigType


async def async_setup(hass: HomeAssistant, config: ConfigType) -> bool:
    """Set up the recorder by delegating to the built-in implementation."""
    return await _builtin_async_setup(hass, config)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run --group dev pytest tests/test_shadow.py -v`
Expected: PASS. If `is_built_in` is True, the harness did not pick up `custom_components/` — check that tests run from the project root and `enable_custom_integrations` is active; do not proceed until the shadow genuinely loads.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: pass-through custom recorder integration shadowing the built-in"
```

---

### Task 3: Schema fixture for Rust tests

**Files:**
- Create: `scripts/gen_schema.py`, `fixtures/schema_53.sql`, `rust/src/test_util.rs` (test-only helper)

**Interfaces:**
- Produces: `fixtures/schema_53.sql` (checked in; full DDL of an HA schema-53 SQLite DB) and Rust test helper `test_util::open_test_db() -> rusqlite::Connection`.

- [ ] **Step 1: Write the schema dump script**

`scripts/gen_schema.py`:
```python
"""Generate fixtures/schema_53.sql from HA's SQLAlchemy models."""

import sqlite3
from pathlib import Path
import tempfile

from sqlalchemy import create_engine

from homeassistant.components.recorder.db_schema import SCHEMA_VERSION, Base

assert SCHEMA_VERSION == 53, f"Update fixture + const for schema {SCHEMA_VERSION}"

with tempfile.TemporaryDirectory() as tmp:
    db = Path(tmp) / "schema.db"
    engine = create_engine(f"sqlite:///{db}")
    Base.metadata.create_all(engine)
    engine.dispose()
    ddl = "\n".join(
        line for line in sqlite3.connect(db).iterdump() if "INSERT INTO" not in line
    )

out = Path(__file__).parent.parent / "fixtures" / "schema_53.sql"
out.parent.mkdir(exist_ok=True)
out.write_text(ddl + "\n")
print(f"wrote {out}")
```

- [ ] **Step 2: Generate and sanity-check the fixture**

Run: `uv run --group dev python scripts/gen_schema.py && grep -c "CREATE TABLE" fixtures/schema_53.sql`
Expected: file written; CREATE TABLE count ≥ 10 (states, states_meta, state_attributes, events, event_data, event_types, statistics, statistics_short_term, statistics_meta, recorder_runs, schema_changes, migration_changes).

- [ ] **Step 3: Write the failing Rust test**

Append to `rust/src/lib.rs`:
```rust
#[cfg(test)]
mod test_util;
```

`rust/src/test_util.rs`:
```rust
use rusqlite::Connection;

pub fn open_test_db() -> Connection {
    let conn = Connection::open_in_memory().unwrap();
    conn.execute_batch(include_str!("../../fixtures/schema_53.sql"))
        .unwrap();
    conn
}

#[test]
fn schema_fixture_loads() {
    let conn = open_test_db();
    let n: i64 = conn
        .query_row("SELECT count(*) FROM sqlite_master WHERE type='table' AND name='states'", [], |r| r.get(0))
        .unwrap();
    assert_eq!(n, 1);
}
```

- [ ] **Step 4: Run Rust tests**

Run: `cargo test`
Expected: `schema_fixture_loads` PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "test: schema-53 SQLite fixture and Rust test harness"
```

---

### Task 4: Rust write engine (states path)

**Files:**
- Create: `rust/src/engine.rs`, `rust/src/writer.rs`
- Modify: `rust/src/lib.rs`

**Interfaces:**
- Consumes: `test_util::open_test_db()` (tests); schema-53 tables.
- Produces (Rust, consumed by Task 5 bindings):
  - `struct StateWrite { entity_id: String, state: Option<String>, shared_attrs: Option<Vec<u8>>, attrs_hash: i64, last_updated_ts: f64, last_changed_ts: Option<f64>, last_reported_ts: Option<f64>, old_last_reported_ts: Option<f64>, context_id_bin: Option<Vec<u8>>, context_user_id_bin: Option<Vec<u8>>, context_parent_id_bin: Option<Vec<u8>>, origin_idx: i64, entity_removed: bool }`
  - `Engine::open(path: &str) -> Result<Engine>`; `Engine::enqueue(w: StateWrite)`; `Engine::flush() -> Result<()>` (blocking, returns after commit); `Engine::clear_caches()`; `Engine::shutdown()`.

Semantics replicated from `Recorder._process_state_changed_event_into_session` (core.py:1088):
- `shared_attrs == None` → skip the row entirely (Python passes None when serialization failed).
- old-state chaining: per-entity map of last inserted `state_id` → set `old_state_id`; if `old_last_reported_ts` is set, `UPDATE states SET last_reported_ts=? WHERE state_id=<old id>` in the same transaction.
- `entity_removed` → `state = NULL`; drop the entity from the chain map; if the entity has no `states_meta` row yet, skip the insert.
- `states_meta`: HashMap cache, on miss SELECT then INSERT.
- `state_attributes`: LRU (cap 2048) keyed by attrs bytes → `attributes_id`; on miss `SELECT attributes_id FROM state_attributes WHERE hash=?1 AND shared_attrs=?2`, else INSERT with the Python-provided hash.
- Legacy columns (`entity_id`, `attributes`, `event_id`, `last_changed`, `last_updated`, `context_id`, `context_user_id`, `context_parent_id`) are left NULL, matching `States.from_event`.

- [ ] **Step 1: Write the failing Rust tests**

Append to `rust/src/engine.rs` (tests module; engine code comes in Step 3):
```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn w(entity: &str, state: &str, attrs: &str, ts: f64) -> StateWrite {
        StateWrite {
            entity_id: entity.into(),
            state: Some(state.into()),
            shared_attrs: Some(attrs.as_bytes().to_vec()),
            attrs_hash: 1234,
            last_updated_ts: ts,
            last_changed_ts: None,
            last_reported_ts: None,
            old_last_reported_ts: None,
            context_id_bin: Some(vec![1u8; 16]),
            context_user_id_bin: None,
            context_parent_id_bin: None,
            origin_idx: 0,
            entity_removed: false,
        }
    }

    #[test]
    fn writes_state_with_meta_and_attrs() {
        let (engine, conn) = Engine::open_for_test();
        engine.enqueue(w("sensor.a", "1", "{\"unit\":\"W\"}", 100.0));
        engine.flush().unwrap();
        let (state, meta_id, attrs_id): (String, i64, i64) = conn
            .query_row(
                "SELECT state, metadata_id, attributes_id FROM states",
                [],
                |r| Ok((r.get(0)?, r.get(1)?, r.get(2)?)),
            )
            .unwrap();
        assert_eq!(state, "1");
        let entity: String = conn
            .query_row("SELECT entity_id FROM states_meta WHERE metadata_id=?1", [meta_id], |r| r.get(0))
            .unwrap();
        assert_eq!(entity, "sensor.a");
        let attrs: String = conn
            .query_row("SELECT shared_attrs FROM state_attributes WHERE attributes_id=?1", [attrs_id], |r| r.get(0))
            .unwrap();
        assert_eq!(attrs, "{\"unit\":\"W\"}");
    }

    #[test]
    fn dedups_attrs_and_chains_old_state() {
        let (engine, conn) = Engine::open_for_test();
        engine.enqueue(w("sensor.a", "1", "{\"unit\":\"W\"}", 100.0));
        engine.enqueue(w("sensor.a", "2", "{\"unit\":\"W\"}", 101.0));
        engine.flush().unwrap();
        let attrs_rows: i64 = conn.query_row("SELECT count(*) FROM state_attributes", [], |r| r.get(0)).unwrap();
        assert_eq!(attrs_rows, 1);
        let old: Option<i64> = conn
            .query_row("SELECT old_state_id FROM states WHERE state='2'", [], |r| r.get(0))
            .unwrap();
        let first: i64 = conn
            .query_row("SELECT state_id FROM states WHERE state='1'", [], |r| r.get(0))
            .unwrap();
        assert_eq!(old, Some(first));
    }

    #[test]
    fn entity_removed_writes_null_and_breaks_chain() {
        let (engine, conn) = Engine::open_for_test();
        engine.enqueue(w("sensor.a", "1", "{}", 100.0));
        let mut removed = w("sensor.a", "", "{}", 101.0);
        removed.state = None;
        removed.entity_removed = true;
        engine.enqueue(removed);
        engine.enqueue(w("sensor.a", "3", "{}", 102.0));
        engine.flush().unwrap();
        let null_states: i64 = conn
            .query_row("SELECT count(*) FROM states WHERE state IS NULL", [], |r| r.get(0))
            .unwrap();
        assert_eq!(null_states, 1);
        let old: Option<i64> = conn
            .query_row("SELECT old_state_id FROM states WHERE state='3'", [], |r| r.get(0))
            .unwrap();
        assert_eq!(old, None);
    }

    #[test]
    fn updates_old_row_last_reported() {
        let (engine, conn) = Engine::open_for_test();
        engine.enqueue(w("sensor.a", "1", "{}", 100.0));
        let mut second = w("sensor.a", "2", "{}", 101.0);
        second.old_last_reported_ts = Some(100.5);
        engine.enqueue(second);
        engine.flush().unwrap();
        let lr: Option<f64> = conn
            .query_row("SELECT last_reported_ts FROM states WHERE state='1'", [], |r| r.get(0))
            .unwrap();
        assert_eq!(lr, Some(100.5));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test`
Expected: compile FAILURE (`Engine` not defined).

- [ ] **Step 3: Implement the engine**

`rust/src/writer.rs` — writer thread owning the connection:
```rust
use std::collections::HashMap;
use std::num::NonZeroUsize;
use std::sync::mpsc::{Receiver, Sender};

use lru::LruCache;
use rusqlite::{params, Connection};

use crate::engine::StateWrite;

pub enum Msg {
    Write(StateWrite),
    Flush(Sender<Result<(), String>>),
    ClearCaches,
    Shutdown(Sender<Result<(), String>>),
}

pub struct Writer {
    conn: Connection,
    buffer: Vec<StateWrite>,
    meta_ids: HashMap<String, i64>,
    last_state_id: HashMap<String, i64>,
    attr_ids: LruCache<Vec<u8>, i64>,
}

const MAX_BUFFER: usize = 4096;

impl Writer {
    pub fn new(conn: Connection) -> Self {
        conn.execute_batch(
            "PRAGMA journal_mode=WAL;
             PRAGMA busy_timeout=10000;
             PRAGMA synchronous=NORMAL;
             PRAGMA foreign_keys=OFF;",
        )
        .expect("pragmas");
        Self {
            conn,
            buffer: Vec::new(),
            meta_ids: HashMap::new(),
            last_state_id: HashMap::new(),
            attr_ids: LruCache::new(NonZeroUsize::new(2048).unwrap()),
        }
    }

    pub fn run(mut self, rx: Receiver<Msg>) {
        while let Ok(msg) = rx.recv() {
            match msg {
                Msg::Write(w) => {
                    self.buffer.push(w);
                    if self.buffer.len() >= MAX_BUFFER {
                        let _ = self.flush();
                    }
                }
                Msg::Flush(ack) => {
                    let _ = ack.send(self.flush());
                }
                Msg::ClearCaches => {
                    self.meta_ids.clear();
                    self.last_state_id.clear();
                    self.attr_ids.clear();
                }
                Msg::Shutdown(ack) => {
                    let _ = ack.send(self.flush());
                    return;
                }
            }
        }
    }

    fn flush(&mut self) -> Result<(), String> {
        if self.buffer.is_empty() {
            return Ok(());
        }
        let writes = std::mem::take(&mut self.buffer);
        let tx = self.conn.transaction().map_err(|e| e.to_string())?;
        for w in &writes {
            Self::apply(&tx, &mut self.meta_ids, &mut self.last_state_id, &mut self.attr_ids, w)
                .map_err(|e| e.to_string())?;
        }
        tx.commit().map_err(|e| e.to_string())
    }

    fn apply(
        tx: &rusqlite::Transaction<'_>,
        meta_ids: &mut HashMap<String, i64>,
        last_state_id: &mut HashMap<String, i64>,
        attr_ids: &mut LruCache<Vec<u8>, i64>,
        w: &StateWrite,
    ) -> rusqlite::Result<()> {
        // Serialization failed on the Python side: skip, matching stock behavior.
        let Some(shared_attrs) = &w.shared_attrs else { return Ok(()) };

        // states_meta lookup / insert
        let meta_id = match meta_ids.get(&w.entity_id) {
            Some(id) => Some(*id),
            None => {
                let found: Option<i64> = tx
                    .query_row(
                        "SELECT metadata_id FROM states_meta WHERE entity_id=?1",
                        params![w.entity_id],
                        |r| r.get(0),
                    )
                    .map(Some)
                    .or_else(|e| if e == rusqlite::Error::QueryReturnedNoRows { Ok(None) } else { Err(e) })?;
                match found {
                    Some(id) => {
                        meta_ids.insert(w.entity_id.clone(), id);
                        Some(id)
                    }
                    None if w.entity_removed => None, // renamed/never existed: skip row
                    None => {
                        tx.execute(
                            "INSERT INTO states_meta (entity_id) VALUES (?1)",
                            params![w.entity_id],
                        )?;
                        let id = tx.last_insert_rowid();
                        meta_ids.insert(w.entity_id.clone(), id);
                        Some(id)
                    }
                }
            }
        };
        let Some(meta_id) = meta_id else { return Ok(()) };

        // state_attributes dedup (hash computed on the Python side)
        let attrs_id = match attr_ids.get(shared_attrs) {
            Some(id) => *id,
            None => {
                let found: Option<i64> = tx
                    .query_row(
                        "SELECT attributes_id FROM state_attributes WHERE hash=?1 AND shared_attrs=?2",
                        params![w.attrs_hash, std::str::from_utf8(shared_attrs).unwrap_or("")],
                        |r| r.get(0),
                    )
                    .map(Some)
                    .or_else(|e| if e == rusqlite::Error::QueryReturnedNoRows { Ok(None) } else { Err(e) })?;
                let id = match found {
                    Some(id) => id,
                    None => {
                        tx.execute(
                            "INSERT INTO state_attributes (hash, shared_attrs) VALUES (?1, ?2)",
                            params![w.attrs_hash, std::str::from_utf8(shared_attrs).unwrap_or("")],
                        )?;
                        tx.last_insert_rowid()
                    }
                };
                attr_ids.put(shared_attrs.clone(), id);
                id
            }
        };

        // old-state chaining + last_reported backfill
        let old_state_id = last_state_id.get(&w.entity_id).copied();
        if let (Some(old_id), Some(old_lr)) = (old_state_id, w.old_last_reported_ts) {
            tx.execute(
                "UPDATE states SET last_reported_ts=?1 WHERE state_id=?2",
                params![old_lr, old_id],
            )?;
        }

        tx.execute(
            "INSERT INTO states (state, metadata_id, attributes_id, old_state_id,
                last_updated_ts, last_changed_ts, last_reported_ts,
                context_id_bin, context_user_id_bin, context_parent_id_bin, origin_idx)
             VALUES (?1,?2,?3,?4,?5,?6,?7,?8,?9,?10,?11)",
            params![
                w.state, meta_id, attrs_id, old_state_id,
                w.last_updated_ts, w.last_changed_ts, w.last_reported_ts,
                w.context_id_bin, w.context_user_id_bin, w.context_parent_id_bin, w.origin_idx
            ],
        )?;
        if w.entity_removed {
            last_state_id.remove(&w.entity_id);
        } else {
            last_state_id.insert(w.entity_id.clone(), tx.last_insert_rowid());
        }
        Ok(())
    }
}
```

`rust/src/engine.rs` — public handle:
```rust
use std::sync::mpsc::{channel, Sender};
use std::thread::JoinHandle;

use rusqlite::Connection;

use crate::writer::{Msg, Writer};

#[derive(Debug, Clone)]
pub struct StateWrite {
    pub entity_id: String,
    pub state: Option<String>,
    pub shared_attrs: Option<Vec<u8>>,
    pub attrs_hash: i64,
    pub last_updated_ts: f64,
    pub last_changed_ts: Option<f64>,
    pub last_reported_ts: Option<f64>,
    pub old_last_reported_ts: Option<f64>,
    pub context_id_bin: Option<Vec<u8>>,
    pub context_user_id_bin: Option<Vec<u8>>,
    pub context_parent_id_bin: Option<Vec<u8>>,
    pub origin_idx: i64,
    pub entity_removed: bool,
}

pub struct Engine {
    tx: Sender<Msg>,
    handle: Option<JoinHandle<()>>,
}

impl Engine {
    pub fn open(path: &str) -> Result<Self, String> {
        let conn = Connection::open(path).map_err(|e| e.to_string())?;
        Ok(Self::from_conn(conn))
    }

    fn from_conn(conn: Connection) -> Self {
        let (tx, rx) = channel();
        let writer = Writer::new(conn);
        let handle = std::thread::Builder::new()
            .name("ha-recorder-engine".into())
            .spawn(move || writer.run(rx))
            .expect("spawn writer thread");
        Self { tx, handle: Some(handle) }
    }

    #[cfg(test)]
    pub fn open_for_test() -> (Self, Connection) {
        // Shared in-memory DB so the test can inspect what the writer committed.
        let uri = format!("file:test_{:p}?mode=memory&cache=shared", &std::ptr::null::<u8>());
        let keeper = Connection::open(&uri).unwrap();
        keeper
            .execute_batch(include_str!("../../fixtures/schema_53.sql"))
            .unwrap();
        let writer_conn = Connection::open(&uri).unwrap();
        (Self::from_conn(writer_conn), keeper)
    }

    pub fn enqueue(&self, w: StateWrite) {
        let _ = self.tx.send(Msg::Write(w));
    }

    pub fn flush(&self) -> Result<(), String> {
        let (ack_tx, ack_rx) = channel();
        self.tx.send(Msg::Flush(ack_tx)).map_err(|e| e.to_string())?;
        ack_rx.recv().map_err(|e| e.to_string())?
    }

    pub fn clear_caches(&self) {
        let _ = self.tx.send(Msg::ClearCaches);
    }

    pub fn shutdown(&mut self) -> Result<(), String> {
        let (ack_tx, ack_rx) = channel();
        self.tx.send(Msg::Shutdown(ack_tx)).map_err(|e| e.to_string())?;
        let res = ack_rx.recv().map_err(|e| e.to_string())?;
        if let Some(h) = self.handle.take() {
            let _ = h.join();
        }
        res
    }
}
```

Update `rust/src/lib.rs` to declare the modules:
```rust
pub mod engine;
pub mod writer;
```
(keep the existing `engine_version` pyfunction and `#[cfg(test)] mod test_util;`).

Note: the test-only in-memory URI trick needs the writer connection opened with the same shared-cache URI; if rusqlite flags are needed use `Connection::open_with_flags(&uri, OpenFlags::SQLITE_OPEN_READ_WRITE | OpenFlags::SQLITE_OPEN_CREATE | OpenFlags::SQLITE_OPEN_URI)`. If shared-cache proves flaky, switch `open_for_test` to a `tempfile` on disk — behavior identical to production (file DB) and no test semantics lost.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cargo test`
Expected: all 4 engine tests + schema fixture test PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: Rust write engine for the states path with dedup and old-state chaining"
```

---

### Task 5: PyO3 bindings for the engine

**Files:**
- Modify: `rust/src/lib.rs`
- Create: `tests/test_bindings.py`

**Interfaces:**
- Consumes: `engine::{Engine, StateWrite}` from Task 4.
- Produces (Python API, consumed by Task 6):
  - `ha_recorder_engine.RecorderEngine(db_path: str)`
  - `.enqueue_state(entity_id: str, state: str | None, shared_attrs: bytes | None, attrs_hash: int, last_updated_ts: float, last_changed_ts: float | None, last_reported_ts: float | None, old_last_reported_ts: float | None, context_id_bin: bytes | None, context_user_id_bin: bytes | None, context_parent_id_bin: bytes | None, origin_idx: int, entity_removed: bool) -> None`
  - `.flush() -> None` (raises `RuntimeError` on write failure)
  - `.clear_caches() -> None`
  - `.shutdown() -> None`

- [ ] **Step 1: Write the failing test**

`tests/test_bindings.py`:
```python
"""The PyO3 bindings write real rows readable via sqlite3."""

import sqlite3
from pathlib import Path

import ha_recorder_engine


def _make_db(tmp_path: Path) -> Path:
    db = tmp_path / "test.db"
    schema = Path(__file__).parent.parent / "fixtures" / "schema_53.sql"
    conn = sqlite3.connect(db)
    conn.executescript(schema.read_text())
    conn.close()
    return db


def test_enqueue_flush_roundtrip(tmp_path: Path) -> None:
    db = _make_db(tmp_path)
    engine = ha_recorder_engine.RecorderEngine(str(db))
    engine.enqueue_state(
        "sensor.power", "42", b'{"unit_of_measurement":"W"}', 99, 100.0,
        None, None, None, b"\x01" * 16, None, None, 0, False,
    )
    engine.flush()
    engine.shutdown()

    conn = sqlite3.connect(db)
    row = conn.execute(
        "SELECT s.state, m.entity_id, a.shared_attrs FROM states s"
        " JOIN states_meta m ON s.metadata_id = m.metadata_id"
        " JOIN state_attributes a ON s.attributes_id = a.attributes_id"
    ).fetchone()
    assert row == ("42", "sensor.power", '{"unit_of_measurement":"W"}')
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run --group dev maturin develop && uv run --group dev pytest tests/test_bindings.py -v`
Expected: FAIL with `AttributeError: module 'ha_recorder_engine' has no attribute 'RecorderEngine'`

- [ ] **Step 3: Implement the bindings**

Replace the `#[pymodule]` section of `rust/src/lib.rs` with:
```rust
use pyo3::exceptions::PyRuntimeError;
use pyo3::prelude::*;

use crate::engine::{Engine, StateWrite};

#[pyclass]
struct RecorderEngine {
    inner: Option<Engine>,
}

#[pymethods]
impl RecorderEngine {
    #[new]
    fn new(db_path: &str) -> PyResult<Self> {
        let inner = Engine::open(db_path).map_err(PyRuntimeError::new_err)?;
        Ok(Self { inner: Some(inner) })
    }

    #[allow(clippy::too_many_arguments)]
    #[pyo3(signature = (entity_id, state, shared_attrs, attrs_hash, last_updated_ts,
        last_changed_ts, last_reported_ts, old_last_reported_ts,
        context_id_bin, context_user_id_bin, context_parent_id_bin,
        origin_idx, entity_removed))]
    fn enqueue_state(
        &self,
        entity_id: String,
        state: Option<String>,
        shared_attrs: Option<Vec<u8>>,
        attrs_hash: i64,
        last_updated_ts: f64,
        last_changed_ts: Option<f64>,
        last_reported_ts: Option<f64>,
        old_last_reported_ts: Option<f64>,
        context_id_bin: Option<Vec<u8>>,
        context_user_id_bin: Option<Vec<u8>>,
        context_parent_id_bin: Option<Vec<u8>>,
        origin_idx: i64,
        entity_removed: bool,
    ) -> PyResult<()> {
        let engine = self.inner.as_ref().ok_or_else(|| PyRuntimeError::new_err("engine shut down"))?;
        engine.enqueue(StateWrite {
            entity_id, state, shared_attrs, attrs_hash, last_updated_ts,
            last_changed_ts, last_reported_ts, old_last_reported_ts,
            context_id_bin, context_user_id_bin, context_parent_id_bin,
            origin_idx, entity_removed,
        });
        Ok(())
    }

    fn flush(&self, py: Python<'_>) -> PyResult<()> {
        let engine = self.inner.as_ref().ok_or_else(|| PyRuntimeError::new_err("engine shut down"))?;
        py.allow_threads(|| engine.flush()).map_err(PyRuntimeError::new_err)
    }

    fn clear_caches(&self) -> PyResult<()> {
        let engine = self.inner.as_ref().ok_or_else(|| PyRuntimeError::new_err("engine shut down"))?;
        engine.clear_caches();
        Ok(())
    }

    fn shutdown(&mut self, py: Python<'_>) -> PyResult<()> {
        if let Some(mut engine) = self.inner.take() {
            py.allow_threads(move || engine.shutdown()).map_err(PyRuntimeError::new_err)?;
        }
        Ok(())
    }
}

#[pyfunction]
fn engine_version() -> &'static str {
    env!("CARGO_PKG_VERSION")
}

#[pymodule]
fn ha_recorder_engine(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(engine_version, m)?)?;
    m.add_class::<RecorderEngine>()?;
    Ok(())
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run --group dev maturin develop && uv run --group dev pytest tests/ -v && cargo test`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: PyO3 RecorderEngine bindings"
```

---

### Task 6: RustRecorder subclass wired into the custom component

**Files:**
- Create: `custom_components/recorder/rust_recorder.py`
- Modify: `custom_components/recorder/__init__.py`
- Create: `tests/test_rust_recorder.py`

**Interfaces:**
- Consumes: `ha_recorder_engine.RecorderEngine` (Task 5); built-in `Recorder`, `States`, `StateAttributes`, managers.
- Produces: `RustRecorder(Recorder)`; custom `async_setup` that instantiates it (SQLite + schema 53 only, else stock delegation).

Key wiring facts from the built-in `Recorder` (core.py):
- `_setup_recorder()` runs connection + migration; the engine must open only after migration is done and only then take over state writes. Hook: override `_activate_and_set_db_ready` (core.py:805) — call `super()`, then create the engine. Until the engine exists, `_process_state_changed_event_into_session` falls back to `super()`, so pre-migration startup events follow the stock path.
- `_commit_event_session_or_retry()` commits the SQLAlchemy session; overriding it and flushing the engine afterwards makes `async_wait_recording_done` (which drives a commit) see Rust rows too.
- `Recorder.run()` (core.py:673) calls `self._shutdown()` (core.py:1475) on the recorder thread when the loop exits — override `_shutdown` and call `engine.shutdown()` after `super()._shutdown()`, so the engine's final flush happens after the stock final commit.
- Purge tasks run through `queue_task`; after `PurgeTask`/`PurgeEntitiesTask` complete, `states_meta` rows may be gone → call `engine.clear_caches()`. Hook: override `_process_one_task_or_event_or_recover` and clear caches when the task is a purge task.

- [ ] **Step 1: Write the failing tests**

`tests/test_rust_recorder.py`:
```python
"""End-to-end: states recorded through the Rust engine, readable via stock query paths."""

from homeassistant.components import recorder
from homeassistant.components.recorder.db_schema import StateAttributes, States, StatesMeta
from homeassistant.components.recorder.util import session_scope
from homeassistant.core import HomeAssistant
from homeassistant.setup import async_setup_component
from pytest_homeassistant_custom_component.components.recorder.common import (
    async_wait_recording_done,
)

from custom_components.recorder.rust_recorder import RustRecorder


async def _setup(hass: HomeAssistant, tmp_path) -> None:
    db_url = f"sqlite:///{tmp_path}/rust_test.db"
    assert await async_setup_component(
        hass, recorder.DOMAIN, {recorder.DOMAIN: {"db_url": db_url}}
    )
    await async_wait_recording_done(hass)


async def test_instance_is_rust_recorder(hass: HomeAssistant, tmp_path) -> None:
    await _setup(hass, tmp_path)
    assert isinstance(recorder.get_instance(hass), RustRecorder)


async def test_states_written_and_deduped(hass: HomeAssistant, tmp_path) -> None:
    await _setup(hass, tmp_path)
    hass.states.async_set("sensor.power", "1", {"unit_of_measurement": "W"})
    hass.states.async_set("sensor.power", "2", {"unit_of_measurement": "W"})
    await async_wait_recording_done(hass)

    instance = recorder.get_instance(hass)

    def _read() -> tuple[int, int, int]:
        with session_scope(hass=hass, read_only=True) as session:
            states = session.query(States).all()
            metas = session.query(StatesMeta).filter(StatesMeta.entity_id == "sensor.power").count()
            attrs = session.query(StateAttributes).count()
            assert states[1].old_state_id == states[0].state_id
            return (len(states), metas, attrs)

    n_states, n_meta, n_attrs = await instance.async_add_executor_job(_read)
    assert n_states == 2
    assert n_meta == 1
    assert n_attrs == 1


async def test_history_api_reads_rust_rows(hass: HomeAssistant, tmp_path) -> None:
    from homeassistant.components.recorder import history
    import homeassistant.util.dt as dt_util

    await _setup(hass, tmp_path)
    start = dt_util.utcnow()
    hass.states.async_set("sensor.power", "7", {"unit_of_measurement": "W"})
    await async_wait_recording_done(hass)

    instance = recorder.get_instance(hass)
    result = await instance.async_add_executor_job(
        lambda: history.get_significant_states(hass, start, None, ["sensor.power"])
    )
    assert [s.state for s in result["sensor.power"]] == ["7"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run --group dev pytest tests/test_rust_recorder.py -v`
Expected: FAIL with `ImportError` (`rust_recorder` module missing).

- [ ] **Step 3: Implement RustRecorder**

`custom_components/recorder/rust_recorder.py`:
```python
"""Recorder subclass that routes the states write path through the Rust engine."""

from __future__ import annotations

import logging
from typing import Any

from sqlalchemy.engine import Engine as SAEngine

from ha_recorder_engine import RecorderEngine
from homeassistant.components.recorder.core import Recorder
from homeassistant.components.recorder.db_schema import StateAttributes
from homeassistant.core import Event, EventStateChangedData
from homeassistant.util.ulid import ulid_to_bytes_or_none
from homeassistant.util.uuid import uuid_hex_to_bytes_or_none

_LOGGER = logging.getLogger(__name__)


class RustRecorder(Recorder):
    """Stock recorder with the states write path in Rust."""

    _engine_rs: RecorderEngine | None = None

    def _activate_and_set_db_ready(self, *args: Any, **kwargs: Any) -> None:
        """After migration completes, open the Rust engine on the same DB file."""
        super()._activate_and_set_db_ready(*args, **kwargs)
        db_path = self.db_url.removeprefix("sqlite:///")
        self._engine_rs = RecorderEngine(db_path)
        _LOGGER.info("Rust recorder engine active on %s", db_path)

    def _process_state_changed_event_into_session(
        self, event: Event[EventStateChangedData]
    ) -> None:
        engine = self._engine_rs
        if engine is None:  # pre-migration startup events: stock path
            super()._process_state_changed_event_into_session(event)
            return

        entity_id = event.data["entity_id"]
        new_state = event.data["new_state"]
        old_state = event.data["old_state"]
        entity_removed = new_state is None

        shared_attrs_bytes = self.state_attributes_manager.serialize_from_event(event)
        if entity_id is None or not shared_attrs_bytes:
            return
        attrs_hash = StateAttributes.hash_shared_attrs_bytes(shared_attrs_bytes)

        if entity_removed:
            state_value: str | None = None
            last_updated_ts = event.time_fired_timestamp
            last_changed_ts = last_reported_ts = None
        else:
            state_value = new_state.state
            last_updated_ts = new_state.last_updated_timestamp
            last_changed_ts = (
                None
                if new_state.last_updated == new_state.last_changed
                else new_state.last_changed_timestamp
            )
            last_reported_ts = (
                None
                if new_state.last_updated == new_state.last_reported
                else new_state.last_reported_timestamp
            )

        context = event.context
        engine.enqueue_state(
            entity_id,
            state_value,
            bytes(shared_attrs_bytes),
            attrs_hash,
            last_updated_ts,
            last_changed_ts,
            last_reported_ts,
            old_state.last_reported_timestamp if old_state else None,
            ulid_to_bytes_or_none(context.id),
            uuid_hex_to_bytes_or_none(context.user_id),
            ulid_to_bytes_or_none(context.parent_id),
            event.origin.idx,
            entity_removed,
        )

    def _commit_event_session_or_retry(self) -> None:
        super()._commit_event_session_or_retry()
        if self._engine_rs is not None:
            self._engine_rs.flush()
```

Update `custom_components/recorder/__init__.py` to build a `RustRecorder` instead of delegating. The built-in `async_setup` constructs `Recorder(...)` directly; rather than reimplementing its ~80 lines of config parsing, monkey-patch the class it instantiates for the duration of setup, and fall back to pure delegation when unsupported:

```python
"""Rust-backed recorder: shadows the built-in recorder integration."""

import logging

from homeassistant.components import recorder as builtin_recorder
from homeassistant.components.recorder import CONFIG_SCHEMA  # noqa: F401
from homeassistant.core import HomeAssistant
from homeassistant.helpers.typing import ConfigType

from .rust_recorder import RustRecorder

_LOGGER = logging.getLogger(__name__)


def _db_url(hass: HomeAssistant, config: ConfigType) -> str:
    conf = config.get(builtin_recorder.DOMAIN, {})
    return conf.get("db_url") or builtin_recorder.get_default_url(hass)


async def async_setup(hass: HomeAssistant, config: ConfigType) -> bool:
    """Set up the recorder, using the Rust engine when supported."""
    db_url = _db_url(hass, config)
    if not db_url.startswith("sqlite://"):
        _LOGGER.warning("Rust recorder supports SQLite only; using stock recorder for %s", db_url)
        return await builtin_recorder.async_setup(hass, config)

    core_module = builtin_recorder.core
    original = core_module.Recorder
    core_module.Recorder = RustRecorder
    try:
        return await builtin_recorder.async_setup(hass, config)
    finally:
        core_module.Recorder = original
```

Implementation notes for this step (verify against the pinned HA version, adjust if drifted):
- If `builtin_recorder.async_setup` references `Recorder` by direct import instead of `core.Recorder` attribute access, patch the name in the module where `async_setup` resolves it (check `homeassistant/components/recorder/__init__.py` in the installed HA).
- Schema-version guard: after setup, `RustRecorder._activate_and_set_db_ready` runs post-migration, so the schema equals the pinned HA's `SCHEMA_VERSION`; assert `SCHEMA_VERSION == ENGINE_SUPPORTED_SCHEMA_VERSION` at module import and fall back to stock delegation if it differs.
- In-memory `sqlite://` (no path) is used by many upstream tests but has no file for a second connection — treat it as unsupported (stock delegation) since production Pi targets are file-backed; `tests/test_shadow.py` keeps covering that path.

- [ ] **Step 4: Add engine shutdown and purge-cache hooks**

In `rust_recorder.py`, override the recorder thread's exit hook (`Recorder._shutdown`, called by `run()` when the loop exits) so the engine flushes and joins after the final Python commit:

```python
    def _shutdown(self) -> None:  # runs on the recorder thread
        super()._shutdown()
        if self._engine_rs is not None:
            self._engine_rs.shutdown()
            self._engine_rs = None
```

And clear Rust caches after purge tasks (entity purges can delete `states_meta` rows):

```python
    def _process_one_task_or_event_or_recover(self, task: object) -> None:
        super()._process_one_task_or_event_or_recover(task)
        if self._engine_rs is not None and type(task).__name__ in (
            "PurgeTask",
            "PurgeEntitiesTask",
        ):
            self._engine_rs.clear_caches()
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run --group dev pytest tests/ -v`
Expected: all PASS, including Task 2's shadow test (still exercising the non-file fallback path).

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: RustRecorder subclass routing states writes through the Rust engine"
```

---

### Task 7: Parity test — stock recorder vs. RustRecorder

**Files:**
- Create: `tests/test_parity.py`

**Interfaces:**
- Consumes: everything above; stock `Recorder` via a second HA instance.

- [ ] **Step 1: Write the failing (or initially revealing) test**

`tests/test_parity.py`:
```python
"""Drive identical state sequences through stock and Rust recorders; compare rows."""

from pathlib import Path
import sqlite3
from typing import Any

from homeassistant.components import recorder
from homeassistant.core import HomeAssistant
from homeassistant.setup import async_setup_component
from pytest_homeassistant_custom_component.components.recorder.common import (
    async_wait_recording_done,
)

SEQUENCE: list[tuple[str, str, dict[str, Any]]] = [
    ("sensor.a", "1", {"unit_of_measurement": "W"}),
    ("sensor.a", "2", {"unit_of_measurement": "W"}),
    ("sensor.b", "on", {}),
    ("sensor.a", "2", {"unit_of_measurement": "W", "friendly_name": "A"}),
    ("sensor.b", "off", {}),
]

COMPARE_SQL = """
    SELECT m.entity_id, s.state,
           s.old_state_id IS NOT NULL,
           a.shared_attrs
    FROM states s
    JOIN states_meta m ON s.metadata_id = m.metadata_id
    LEFT JOIN state_attributes a ON s.attributes_id = a.attributes_id
    ORDER BY s.state_id
"""


async def _run_sequence(hass: HomeAssistant, db_path: Path) -> list[tuple]:
    assert await async_setup_component(
        hass, recorder.DOMAIN, {recorder.DOMAIN: {"db_url": f"sqlite:///{db_path}"}}
    )
    await async_wait_recording_done(hass)
    for entity_id, state, attrs in SEQUENCE:
        hass.states.async_set(entity_id, state, attrs)
    await async_wait_recording_done(hass)
    await hass.async_stop()
    return sqlite3.connect(db_path).execute(COMPARE_SQL).fetchall()


async def test_row_parity_with_stock_recorder(
    hass: HomeAssistant, tmp_path: Path, enable_custom_integrations: None
) -> None:
    rust_rows = await _run_sequence(hass, tmp_path / "rust.db")
    assert len(rust_rows) == len(SEQUENCE)
    # Golden expectations derived from the stock recorder's documented write
    # semantics (attribute dedup, old-state chaining). If any of these fail,
    # diff against a stock-recorder DB produced by the same sequence in a
    # core-hass checkout before changing the assertion.
    assert rust_rows[0][:3] == ("sensor.a", "1", 0)
    assert rust_rows[1][:3] == ("sensor.a", "2", 1)
    assert rust_rows[2][:3] == ("sensor.b", "on", 0)
    assert rust_rows[3][:3] == ("sensor.a", "2", 1)
    assert rust_rows[4][:3] == ("sensor.b", "off", 1)
    # dedup: rows 0 and 1 share one attributes row
    conn = sqlite3.connect(tmp_path / "rust.db")
    n_attrs = conn.execute("SELECT count(*) FROM state_attributes").fetchone()[0]
    assert n_attrs == 3  # {"unit..W"}, {}, {"unit..W","friendly_name":"A"}
```

(Note: `{}` attrs — HA state with empty attributes still serializes to a shared
attrs row; if the observed stock behavior differs, encode what stock does, not
what seems sensible. Two HA instances in one pytest session fight over
fixtures, hence golden-row assertions instead of a live A/B in this test.)

- [ ] **Step 2: Run the test**

Run: `uv run --group dev pytest tests/test_parity.py -v`
Expected: PASS. Any failure is a compat bug in Tasks 4–6 — fix there, never by loosening assertions without checking stock behavior first.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "test: row-level parity against stock recorder write semantics"
```

---

### Task 8: Benchmark harness

**Files:**
- Create: `scripts/bench.py`, `BENCHMARKS.md`

**Interfaces:**
- Consumes: the full custom component.

- [ ] **Step 1: Write the benchmark script**

`scripts/bench.py`:
```python
"""Measure sustained state-write throughput and RSS: stock vs Rust recorder.

Usage: python scripts/bench.py [--events 50000] [--rust | --stock]
Writes one line of results to stdout; run each mode in a fresh process.
"""

import argparse
import asyncio
import resource
import sys
import time
from pathlib import Path
import tempfile

from homeassistant import core as ha_core
from homeassistant.components import recorder
from homeassistant.setup import async_setup_component


async def run(events: int, use_rust: bool) -> None:
    tmp = Path(tempfile.mkdtemp())
    hass = ha_core.HomeAssistant(str(tmp))
    if use_rust:
        # make custom_components importable exactly as HA would load it
        hass.config.config_dir = str(Path(__file__).parent.parent)
    await hass.async_start()
    assert await async_setup_component(
        hass, recorder.DOMAIN, {recorder.DOMAIN: {"db_url": f"sqlite:///{tmp}/bench.db"}}
    )
    instance = recorder.get_instance(hass)
    await instance.async_recorder_ready.wait()

    start = time.perf_counter()
    for i in range(events):
        hass.states.async_set("sensor.bench", str(i), {"unit_of_measurement": "W", "i": i % 10})
        if i % 1000 == 0:
            await asyncio.sleep(0)
    await recorder.get_instance(hass).async_block_till_done()
    elapsed = time.perf_counter() - start

    rss_mb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / (1024 * 1024)
    mode = "rust" if use_rust else "stock"
    print(f"{mode}: {events} events in {elapsed:.2f}s = {events / elapsed:.0f} ev/s, peak RSS {rss_mb:.0f} MB")
    await hass.async_stop()


if __name__ == "__main__":
    p = argparse.ArgumentParser()
    p.add_argument("--events", type=int, default=50000)
    p.add_argument("--rust", action="store_true")
    args = p.parse_args()
    sys.exit(asyncio.run(run(args.events, args.rust)))
```

(Implementer note: on macOS `ru_maxrss` is bytes, on Linux it is KiB — divide accordingly, detect via `sys.platform`. If setting `config_dir` post-init is rejected by the HA version, pass the project root as the `HomeAssistant(config_dir=...)` argument for the rust run and a bare temp dir for stock.)

- [ ] **Step 2: Run both modes and record results**

```bash
uv run --group dev python scripts/bench.py --events 50000 > /tmp/stock.txt
uv run --group dev python scripts/bench.py --events 50000 --rust > /tmp/rust.txt
cat /tmp/stock.txt /tmp/rust.txt
```

Expected: both complete; record the two lines (dev-machine numbers; Pi numbers are a later, on-device exercise) in `BENCHMARKS.md` with date, machine, and HA version.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "chore: benchmark harness and first stock-vs-rust numbers"
```

---

## Out of scope for this plan (later plans in phase 1)

- Non-state events path (`events`, `event_data`, `event_types`) in Rust.
- Purge and repack in Rust.
- Statistics generation in Rust.
- Running the core-hass recorder/history test suites against the custom component (needs a core-hass checkout harness; separate plan).
- Raspberry Pi on-device benchmarking and packaging (HACS / prebuilt wheels).

## Verification for the whole plan

```bash
cargo test && uv run --group dev maturin develop && uv run --group dev pytest tests/ -v
```

All green, plus `BENCHMARKS.md` containing a stock-vs-rust comparison line pair.
