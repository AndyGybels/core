# Rust-backed recorder drop-in — design

**Date:** 2026-08-17
**Status:** Approved design, pending implementation plan
**Context:** Milestone 1 of a long-term effort to move Home Assistant's hot system
components to Rust for performance and memory wins on small hardware (Raspberry Pi
class), while keeping full backwards compatibility with Python integrations.

## Goal

Replace the recorder component's engine with a Rust implementation (PyO3 extension
module) that is invisible to the rest of Home Assistant:

- Same `recorder` domain and public Python API.
- Same SQLite schema and database file.
- Existing recorder/history test suite passes against it.
- Measurable wins on a Raspberry Pi: lower RSS, lower sustained CPU per event,
  lower history-query latency.

Non-goal: changing the core event bus, state machine, or any device integration.
Those remain untouched Python. (A Rust state machine is a candidate for a later
milestone, *after* two or more Rust subsystems exist to share it — not before.)

## Why the recorder first

- It is the hottest subsystem on small hardware: per-event object churn on the
  write path, and history/statistics queries that cause memory spikes and swap.
- It is cleanly separable: packaged as a component with a defined setup entry
  point, consuming the rest of the system through a single event-bus subscription.
- Its win does not depend on a Rust core. Events cross the FFI boundary once;
  everything after that (batching, writes, purge, statistics, query serialization)
  can be Python-free.

## Deployment shape

Ship as a **custom integration that shadows the built-in `recorder`**
(`custom_components/recorder/`). Home Assistant loads custom integrations with
precedence over built-ins, so:

- No fork of core; installable on a stock HA instance.
- Upstream tracking is done by pinning each release of the drop-in to the
  recorder schema version (and HA version range) it was validated against.
- Rollback is "delete the custom component" — the built-in recorder takes over
  the same database file.

## Architecture

### Rust owns (the hot paths)

- **Write path end-to-end:** one event-stream subscription crossing the FFI
  boundary; batching; the commit-interval loop; SQLite writes; state-attribute
  and event-data deduplication; purge and repack; short-term (5-minute) and
  long-term (hourly) statistics generation. All Python-free after the boundary.
- **Hot read paths:** `history.get_significant_states` and the statistics query
  APIs, implemented in Rust.
- **Websocket serialization:** history/statistics websocket responses serialized
  Rust-straight-to-JSON bytes, never materializing Python objects. This is the
  largest single win for interactive use (history graphs, energy dashboard).

### Python keeps (the risky or cold paths)

- **Schema migrations:** the existing Python migration code runs at startup,
  before the Rust engine takes over. Migrations are the riskiest recorder code
  and the least hot; porting them buys nothing.
- **Long-tail consumers:** `util.session_scope`, the `db_schema` SQLAlchemy
  models (imported by logbook), and the `sql` integration's user-defined queries
  keep working untouched. The SQLite file and schema are unchanged; these
  consumers open reader connections alongside Rust's writer connection
  (WAL mode).
- **The Python facade:** `homeassistant.components.recorder`'s public module
  surface (`get_instance`, `history`, `statistics`, `filters`, constants)
  remains a thin Python layer delegating to the Rust module.

## Compatibility surface (measured in this repo)

- ~20 core components declare a recorder dependency. Deep consumers: `history`,
  `logbook`, `energy`, `sensor` (long-term statistics), `sql`, `statistics`,
  `filter`, `history_stats`.
- ~9 integrations (`opower`, `tibber`, `elvia`, `mill`, `solaredge`,
  `anglian_water`, `ista_ecotrend`, `suez_water`, `kitchen_sink`) push data via
  `async_add_external_statistics` / `async_import_statistics`. These two entry
  points must be preserved exactly (signature and semantics).
- Public API imports observed across components: `get_instance`, `history`,
  `CONF_DB_URL`, `SupportedDialect`, `DOMAIN`, `filters.Filters`,
  `filters.like_domain_matchers`, `statistics.{StatisticsRow, StatisticData,
  StatisticMetaData, StatisticMeanType}`, `models.*` converters,
  `util.session_scope`, `db_schema.{Events, EventData, EventTypes}`.
- The recorder's registered websocket commands.
- The SQLite schema itself is a de-facto public contract: the `sql` integration
  exposes it to user queries, `db_schema` models are imported by other
  components, and external tools read the database file.
- The spec of record for behavior is the existing test suite:
  `tests/components/recorder/` + `tests/components/history/` (~41,700 lines).

## Scope cut: SQLite only

The stock recorder also supports MariaDB/PostgreSQL via SQLAlchemy. Milestone 1
detects a non-SQLite `db_url` and declines to set up, letting the built-in
recorder load instead. Raspberry Pi users — the target audience — are on the
SQLite default.

## Phasing inside the milestone

1. **Rust write engine** behind the existing Python facade; all queries still
   run through the existing Python code against the same database. Shippable and
   benchmarkable on its own.
2. **History and statistics queries** move to Rust.
3. **Direct Rust→JSON websocket serialization** for history/statistics responses.

Each phase is independently shippable and validated against the test suite
before the next begins.

## Validation

- Run the existing recorder + history test suites against the drop-in; failures
  are compat bugs by definition.
- A small on-device benchmark harness (Raspberry Pi): RSS, sustained events/sec,
  history-query latency — stock recorder vs. drop-in, per phase.

## Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Upstream schema version bumps | Pin each release to a validated schema/HA version range; reuse upstream's own Python migrations at startup. |
| Exact-behavior edges (attribute dedup, ULID event IDs, commit-interval semantics) | The 41,700-line test suite is the spec; run it per phase. |
| SQLite writer/reader contention between the Rust writer and Python reader connections | WAL mode plus the same busy-timeout settings the stock recorder uses today. |
| FFI boundary cost per event | Single subscription, minimal per-event conversion (extract primitives once); everything downstream is Rust. |

## Next step

Implementation plan via the writing-plans process, starting with phase 1
(Rust write engine).
