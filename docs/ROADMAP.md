# Encore — Roadmap

This document sets the order in which Encore is built and the condition that must be met before each phase is considered finished. The requirements it refers to (FR-n and NFR-n) are defined in `SPEC.md`. The reasons behind the design are in `DECISIONS.md` (D-n). The numbers that each phase produces are recorded in `BENCHMARKS.md`.

---

## The gate rule

Each phase ends with an exit condition. A phase is finished when its exit condition is met and the result is recorded, or when the exit condition has been measured, missed, and the miss has been recorded with the measured number and the diagnosed reason. In both cases the next phase may begin. A phase is never marked finished on the basis of an untested belief that it works.

A missed exit condition stays recorded as missed. If a later phase fixes the cause, the fix is recorded as a new result next to the old one, and the old one stays in the record.

---

## Testing policy

Encore is verified in three ways, and each is used for the thing it is good at.

**Unit tests** cover the counting rules, which are one pure function. Every rule in FR-3 and every row of the worked example in FR-3 has a test. These tests run in milliseconds and are the complete proof that the rules behave as specified, because the function they test is the function the worker runs.

**Load tests** cover every performance and throughput requirement. Each one is a k6 script in `load/` with a fixed traffic shape. The script's own output is pasted into `BENCHMARKS.md` without editing.

**Scripted drills** cover every failure requirement. Each one is a shell script in `drills/` that applies a fault (kills a container, adds network latency, forces a crash) while a load test is running, and then runs a checker. The checker is a script that compares the system's counts with an exact recount from the log and prints a verdict and the numbers behind it.

There are no mocked integration tests. Each component is tested by running the real component under load and recording what happened.

---

## Test catalogue

The phases below refer to these tests by number.

| # | Name | What it does | Pass condition |
|---|---|---|---|
| T-1 | Single-viewer end to end | One simulated viewer watches, skips, and rewinds; the curve is read afterwards | The curve matches the worked example in FR-3 exactly |
| T-2 | Freshness | A ping is sent for a second that no other viewer is watching, and the curve is polled until that second changes; run at idle and under T-4's load | Time until visible is recorded for both conditions; under load it is within worker lag + 60 s (NFR-4) |
| T-3 | Recount | After any load run, a tool reads every ping from the log, applies the counting rules, and compares the result with Redis and PostgreSQL | Difference is zero (NFR-5) |
| T-4 | Admission off versus on | 50,000 pings per second to one video, climbing from zero over 60 s and holding for 60 s; run once with all limits disabled and once with them enabled | Both runs produce a table of p99, maximum, and the share of 202, 429, and other responses; the enabled run meets NFR-1 and NFR-2 |
| T-5 | Read latency | 2,000 curve reads per second for one video while T-4's load runs | p99 at most 20 ms (NFR-3) |
| T-6 | Hot video | T-4's load, run four times: partition spread off and batching off; spread on and batching off; spread off and batching on; both on | A table of throughput, worker lag, and Redis commands per second for each combination (NFR-8) |
| T-7 | Replay | The curve is exported; Redis and PostgreSQL are emptied; the worker is reset to the start of the log; after it finishes, the curve is exported again | The two exports are identical; rebuild time is recorded (FR-9) |
| T-8 | Duplicates | Every ping is sent two or three times with the same sequence number | Zero extra counts against T-3 (NFR-6) |
| T-9 | Viewer behaviour | Simulated viewers watch, skip forward, rewind, and pause in random patterns | Skipped seconds have zero credit from those viewers; rewatched seconds have the correct count (FR-3) |
| T-10 | Redis killed under load | During T-4's load, the Redis container is killed for 60 s and then restarted | At least 99% of pings receive 202 during the outage; curve reads keep answering; after recovery, T-3 passes except for the viewer-record loss stated in NFR-7 (FR-11, FR-12) |
| T-11 | PostgreSQL killed under load | Same as T-10, for PostgreSQL | Pings receive 202; counting continues; after recovery PostgreSQL matches Redis (FR-11) |
| T-12 | Worker killed mid-backlog | Load is sent with the worker stopped so that a backlog builds; the worker is started and killed with `kill -9` halfway through the backlog, then restarted | The number of re-delivered pings is recorded; T-3 passes afterwards (NFR-6) |
| T-13 | Pre-scaling | A premiere is registered three minutes ahead with an expected peak rate; the controller is watched | Extra workers are healthy at least 10 s before `startsAt`; idle capacity-seconds are recorded (NFR-9) |
| T-14 | Slow Redis | During T-4's load, Toxiproxy adds 500 ms of latency on the Redis path, then 10% packet loss | Pings keep receiving 202; the read path's circuit breaker opens; recovery time is recorded (FR-11) |
| T-15 | Slow PostgreSQL | Same as T-14, for PostgreSQL | Pings keep receiving 202; the flush backs off and resumes; recovery time is recorded (FR-11) |

---

## Phase 0 — The counting rules · *FR-3 · D-007, D-008*

**Goal.** The rules exist as code and are proven correct before anything is connected to them.

**Builds.**
- The `Ping` and `ViewerRecord` types.
- One pure function that takes a ping and the viewer's record (or no record) and returns the list of seconds to credit and the new record.
- A unit test for each rule R1 to R4, a test for each row of the worked example in FR-3, and tests for the edge cases: a gap of exactly 8, exactly 12, 7, and 13; a negative gap; a zero gap; a retried sequence; a sequence lower than the stored one; a first ping with no record.

**Exit condition.** Every test passes, and the function has no dependency on any I/O library.

**Records.** Nothing in `BENCHMARKS.md`.

---

## Phase 1 — Ingest, log, worker, read · *FR-1, FR-2, FR-4, FR-5, FR-6 · D-003, D-004, D-005, D-006, D-010, D-011, D-013, D-017, D-022, D-025*

**Goal.** A ping sent to the service shows up in the curve read from the service.

**Builds.**
- The ingest endpoint with validation, publishing to Kafka with the key from D-010, and the fixed publish timeout from D-022.
- The Kafka topic with 12 partitions and 7-day retention.
- The worker as a Kafka consumer with manual offset commit, calling the Phase 0 function for each ping and writing counts and viewer records to Redis.
- The read endpoint for the curve and the peaks, served from a Caffeine snapshot with a 60-second expiry, built from Redis.
- The worker-lag metric, computed from the Kafka message timestamp (D-025).
- The reference client: a small script that behaves as a player following FR-2.
- Docker Compose with Kafka, Redis, the service, and nothing else yet.

The worker in this phase sends one Redis command per credited second and holds no in-memory batch. Batching is added in Phase 4 so that its effect can be measured.

**Tests.** T-1 and T-2 at idle.

**Exit condition.** T-1 passes. T-2 at idle produces a recorded number.

**Records.** The idle freshness number from T-2.

---

## Phase 2 — Durability and recovery · *FR-9, FR-12, NFR-5, NFR-7 · D-014, D-015, D-016, D-018*

**Goal.** Counts survive the loss of any store, and the log can rebuild everything.

**Builds.**
- PostgreSQL with the `curve`, `flush_checkpoint`, `video`, and `premiere` tables.
- The flush job: every 10 seconds, copy the counters of every changed video from Redis to PostgreSQL and record the Kafka offsets covered.
- Cold load: when the worker or the read path touches a video that Redis does not have, its counters are loaded from PostgreSQL first.
- Rewind recovery: when the worker starts and finds Redis empty, it resets each partition to the checkpointed offset and re-processes from there.
- The read path's fallback from Redis to PostgreSQL, behind a circuit breaker on the Redis client.
- The recount tool used by T-3.

**Tests.** T-3 after a moderate load run, T-7, and T-11.

**Exit condition.** T-3 reports a difference of zero. T-7 produces identical exports. T-11 passes.

**Records.** The T-7 rebuild time.

---

## Phase 3 — Load and the headline comparison · *FR-8, NFR-1, NFR-2, NFR-3, NFR-12 · D-019, D-020, D-021, D-024, D-027* · **HEADLINE**

**Goal.** The service survives the premiere load with limits on, and visibly fails without them.

**Builds.**
- Per-instance tier limits as token buckets, with the tier assignment from FR-8.
- nginx in front of the service, with the same tier limits at the fleet level and upstream discovery through Docker DNS.
- The semaphore on in-flight Kafka publishes, sized from the measured Kafka confirm time (D-021).
- A configuration switch that disables every limit, used only by T-4's first run.
- The k6 scripts for T-4 and T-5.

**Tests.** T-4, T-5, and T-2 under load.

**Exit condition.** The T-4 table exists for both runs. The enabled run meets NFR-1 and NFR-2, or the miss is recorded with the number and the reason. T-5 meets NFR-3 or the miss is recorded.

**Records.** The T-4 table, which is the first table in `README.md`. The measured Kafka confirm time and the resulting semaphore size, which settles the Open status of D-021. The T-5 and T-2 numbers.

---

## Phase 4 — The hot video · *NFR-8 · D-010, D-012*

**Goal.** One video receiving all traffic is processed by several threads and does not overwhelm Redis.

**Builds.**
- The worker's batch loop: read a batch of pings, apply the rules in order against an in-memory copy of each viewer's record, sum the increments per (video, second), and write the sums and the records to Redis in one pipelined transaction before committing the offset.
- A configuration switch for the partition key: video id only, or video id with the viewer-derived spread.
- A configuration switch for batching: one command per credited second, or batched.

**Tests.** T-6, then T-3 after each of its four runs.

**Exit condition.** The T-6 table exists with all four combinations, and T-3 reports zero difference for each.

**Records.** The T-6 table, which is the second table in `README.md`. If a single video saturates the 8-way spread, the spread value is revisited and D-010 is updated.

---

## Phase 5 — Failure drills · *FR-11, NFR-6, NFR-13 · D-006, D-015*

**Goal.** Every row of FR-11 has a scripted drill with a recorded result.

**Builds.**
- The drill scripts in `drills/`: kill Redis, kill PostgreSQL, kill the worker with `kill -9`, each with a fixed timeline.
- The checker that runs after each drill and prints the verdict from T-3 together with the share of pings accepted during the fault.
- The k6 scripts for T-8 and T-9.

**Tests.** T-8, T-9, T-10, T-12. T-11 is re-run under full load.

**Exit condition.** T-8 and T-9 pass. T-10, T-11, and T-12 each have a recorded result, with the number of re-delivered pings from T-12 and the share of pings accepted during each outage.

**Records.** One section per drill in `BENCHMARKS.md`. This is the end of the core scope: at this point every functional requirement except FR-7 and FR-10 is demonstrable.

---

## Phase 6 — Pre-scaling · *FR-7, FR-10, NFR-9 · D-023*

**Goal.** Worker capacity is in place before a premiere begins.

**Builds.**
- The premiere endpoints for registering and listing premieres.
- The scaling controller as a separate small program that polls the upcoming premieres, computes the needed number of workers from the expected peak rate and the measured per-worker throughput from Phase 4, scales the worker service with Docker Compose a fixed lead time before `startsAt`, and scales it back after a fixed delay.
- Measurement of worker startup time, which sets the lead time.

**Tests.** T-13.

**Exit condition.** T-13 shows extra workers healthy at least 10 seconds before `startsAt`.

**Records.** The lead-time arithmetic, the idle capacity-seconds, and the measured startup time.

---

## Phase 7 — Network faults, dashboard, demo · *NFR-10, NFR-11, NFR-13 · D-028*

**Goal.** The project can be shown to a stranger in fifteen minutes.

**Builds.**
- Toxiproxy between the service and Redis, and between the service and PostgreSQL, with drill scripts for T-14 and T-15.
- The Grafana dashboard showing every metric named in NFR-10.
- A demo control panel: a small web page with a slider for offered load, buttons that run each drill, and the dashboard embedded.
- `README.md` with the two headline tables, the architecture, the drill results, and the limitations.
- `BENCHMARKS.md` with the raw output of every test.

**Tests.** T-14 and T-15.

**Exit condition.** Both drills have recorded results. A reader can start the system with one command, run one premiere, kill one component from the panel, and watch the dashboard react.

**Records.** The T-14 and T-15 results. The final state of every Open decision.

---

## What is demonstrable after each phase

| After phase | What can be shown |
|---|---|
| 0 | The rules pass their tests |
| 1 | A ping becomes a curve |
| 2 | Wipe both stores, replay the log, get the same curve |
| 3 | The headline table: limits off collapses, limits on holds |
| 4 | The hot-video table |
| 5 | Kill any store or the worker under load and keep counting correctly |
| 6 | A premiere scales itself up before it starts |
| 7 | The fifteen-minute demo |

---

## Explicitly cut

The following were considered and are outside this roadmap. Each is noted so that a reader knows it was a choice.

- A second counter for seek targets, and the extra client ping it requires (D-009). This is the first candidate for a later version.
- Explicit seek, pause, and play events from the client (D-004).
- Authentication of viewers. The viewer id is accepted as given.
- A Redis cluster or Redis replication. A single Redis instance is used, and its loss is handled by D-015.
- Kafka replication. The topic has a replication factor of 1 on a single broker.
- A second machine for the load generator. The single-machine limitation is stated in every result (D-027). If a second machine becomes available, Phases 3 and 4 are re-run there and both sets of numbers are kept.
- Approximate counting of any kind (D-018).
