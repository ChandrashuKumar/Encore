# Encore — Specification

Encore is a backend service that shows which seconds of a video people watch again. Video players send it position reports, it counts how many times each second of each video was watched, and it serves that curve back to the players, in the same way that a video site draws a "most replayed" graph above the progress bar.

It is built for one hard situation: a premiere, when a single video goes from no viewers to hundreds of thousands of viewers in about a minute, and every one of those viewers is sending reports for the same video.

This document states what Encore must do (the functional requirements), how well it must do it (the non-functional requirements), the limits on how it may be built, and the stack it is built on. The build order and the phase gates are in `ROADMAP.md`. Design decisions and the alternatives that were considered are in `DECISIONS.md`. Measured results are in `BENCHMARKS.md`.

---

## 1. What Encore is

Encore is a standalone service with one input and one output.

**Input:** a position ping from a video player, sent every 10 seconds of playback, which says "viewer V is at second P of video X".

**Output:** for any video, the curve, which is an array with one number per second of the video, where each number is how many times that second was watched.

Everything else about a video platform, such as storage, playback, accounts, and recommendations, is assumed to exist elsewhere. Encore only needs the ids.

---

## 2. Why this problem

The problem becomes hard during a premiere, for four reasons:

- All of the traffic goes to one video, so the work has to be handled for that single video rather than spread across many.
- The traffic climbs from near zero to its peak in about 60 seconds, so capacity has to be in place before the climb begins.
- Every player on that video reads the same curve, thousands of times per second.
- The premiere is the moment when the curve matters most, so the service has to stay up under exactly this load.

A design that writes every ping to a database row will slow down, build up a queue of waiting requests, and then stop serving. Encore must accept pings quickly without waiting on a database, it must count them correctly even when parts of the system crash, and it must reject excess load immediately so that a queue can never build.

---

## 3. What Encore is not

- Encore does not store, stream, catalogue, or play videos.
- Encore computes one curve per video. It does not produce dashboards or per-viewer histories.
- Encore does not make recommendations.
- Encore accepts the viewer id in a ping exactly as it was given, and does no authentication of its own.
- Encore is a backend only. A small reference client is provided to show the contract, and any player can follow that contract.

---

## 4. Functional requirements

### FR-1 — Ping ingestion

Encore accepts a ping with the following fields:

| Field | Type | Meaning |
|---|---|---|
| `viewerId` | string | The viewer, or the viewing session |
| `videoId` | string | The video |
| `position` | integer | The playback position, in whole seconds |
| `sequence` | integer | Starts at 1 for each session and increases by 1 on every ping that is sent |
| `clientTs` | integer | The player's clock when the ping was sent, in milliseconds since the epoch |

The ingest endpoint validates the ping, publishes it to a durable log, and responds with `202 Accepted`. The only system it talks to while handling the request is the log. An invalid ping receives `400`. If the log is unavailable, the endpoint waits for at most a fixed timeout and then responds with `503`.

### FR-2 — Client contract

The player must follow three rules:

1. It sends a ping each time 10 more seconds of the video have been played. The interval is counted in video time, so while the video is paused no ping is sent, and at any playback speed two consecutive pings are always 10 video-seconds apart.
2. It increases `sequence` by 1 on every ping. If a ping is retried because the response was lost, the retry carries the same `sequence` as the original.
3. The ping is the only message the player sends. Encore detects seeks, pauses, and replays from the positions in the pings.

A player that follows the contract gets correct counts. If a player sends fewer pings than the contract asks for, the only effect is that some watched seconds go uncounted; every count that is produced is still correct.

### FR-3 — Counting rules

Counting runs in a worker that reads pings from the log. For each (viewer, video) pair, the worker keeps a record of the last position it saw, the last sequence it saw, and the time of the last ping. The record expires 1 hour after the last ping.

On each ping, the worker computes `gap = position − lastPosition`, and then exactly one of these rules applies:

| Rule | Condition | Action |
|---|---|---|
| R1 | `sequence` is less than or equal to the last sequence seen | Ignore the ping. It is a retry or a re-delivery. |
| R2 | No record exists for this viewer and video | Count nothing. Save the position and the sequence. |
| R3 | `gap` is between 8 and 12, inclusive | Add 1 to `watched` for every second from `lastPosition` up to, but not including, `position`. Save the position and the sequence. |
| R4 | Any other gap (above 12, negative, or zero) | Count nothing. Save the position and the sequence. |

The rules produce the following behaviour:

- Normal viewing credits every second exactly once. Because the range in R3 ends just before `position`, the second at the boundary between two consecutive pings is credited by only one of them.
- A forward skip leaves the skipped seconds uncounted.
- A rewind ping credits nothing at the moment it arrives. The next normal ping after it credits the seconds from the rewound position onward, so a stretch that is watched a second time is counted a second time. This is the complete mechanism for replays.
- A pause credits nothing.
- A retried ping, and a ping that the log delivers a second time after a worker crash, have no effect, because R1 ignores them. This depends on the viewer record and the counts being written in one atomic step, so that after a crash either both are updated or neither is.
- The range of 8 to 12 allows for player timers that fire slightly early or slightly late.

A worked example for one viewer:

| Ping (position, sequence) | Rule | Effect on `watched` |
|---|---|---|
| 10, #1 | R2 | none |
| 20, #2 | R3 | seconds 10–19 +1 |
| 20, #2 (retry) | R1 | none |
| 30, #3 | R3 | seconds 20–29 +1 |
| 20, #4 (rewind) | R4 | none |
| 30, #5 | R3 | seconds 20–29 +1 (now 2) |
| 300, #6 (skip) | R4 | none |
| 310, #7 | R3 | seconds 300–309 +1 |

The final result is that seconds 10–19 were watched once, seconds 20–29 were watched twice, seconds 300–309 were watched once, and every other second stays at zero.

The rules are implemented as one pure function, meaning a function that takes the ping and the viewer record as arguments and returns the decision, with no database or network access inside it. The worker calls this function for every ping, and this function is the only place in the system where the rules exist.

### FR-4 — Validation

A ping is rejected with `400` if any field is missing, if `position` is negative or greater than the video's known duration, or if `sequence` is less than 1. The video's duration is supplied when the video is first registered (see FR-7). When the duration is unknown, the maximum allowed length is used.

### FR-5 — Curve read

`GET /v1/curve/{videoId}` returns the curve in this shape:

```json
{
  "videoId": "abc",
  "duration": 10800,
  "bucketSeconds": 1,
  "watched": [0, 0, 3, 7, 12, ...],
  "snapshotAt": "2026-10-10T12:00:00Z"
}
```

`snapshotAt` is the time at which the returned curve was built. An optional `bucket=N` query parameter returns the curve with the counts summed into N-second buckets. A video that has received no pings returns `200` with a curve of all zeros.

### FR-6 — Peaks read

`GET /v1/curve/{videoId}/peaks?n=5` returns the `n` seconds that have the highest `watched` values, together with their counts. It is computed from the same snapshot that FR-5 serves.

### FR-7 — Premiere registration

`POST /v1/premieres` registers a scheduled premiere with the fields `videoId`, `startsAt`, `expectedPeakPingsPerSecond`, and optionally `durationSeconds` for the video. `GET /v1/premieres/upcoming?withinSeconds=N` lists the premieres that start within the next N seconds. These endpoints exist for the scaling controller (see FR-10), and no rate limit applies to them.

### FR-8 — Priority admission

Every request belongs to a tier. When the system is overloaded, the lower tiers are rejected before the higher ones. A rejection is an immediate `429` with the header `Retry-After: 1`.

| Tier | Endpoints | Rejected |
|---|---|---|
| `PING_WRITE` | `POST /v1/pings` | last |
| `CURVE_READ` | `GET /v1/curve/{id}` | second |
| `PEAKS_READ` | `GET /v1/curve/{id}/peaks` | first |
| none | `/actuator/*`, `/v1/premieres/*` | never |

The limit exists in two places. Each service instance has its own limit, which protects that instance's CPU. The edge in front of all of the instances has a separate limit for the whole fleet, so that the total amount of admitted traffic stays the same when more instances are added.

### FR-9 — Rebuild from the log

If the counter store and the durable store are both empty, and the log is replayed from its beginning, the result must be exactly the same counts that the live system produced. Every piece of state that lives outside the log must be rebuildable from the log.

### FR-10 — Pre-scaling from the schedule

A controller reads the premiere schedule and starts additional worker capacity ahead of each premiere's `startsAt`. It removes the extra capacity after a fixed delay past the start time. The lead time is derived from the measured startup time of a worker plus a safety margin, and the controller records how many idle capacity-seconds it spent waiting for the premiere to begin.

### FR-11 — Behaviour when a component is lost

When any single supporting component is lost, pings continue to be accepted and curves continue to be served:

| Lost component | Keeps working | Degrades |
|---|---|---|
| Counter store | Pings are accepted. Curves are read from the last snapshot, and after that from the durable store. | Counts stop advancing until the store recovers. |
| Durable store | Pings are accepted. Counting continues. Curves are read. | The durability lag grows until the store recovers. |
| Worker | Pings are accepted. Curves are read. | Counts stop advancing and the backlog grows. |
| One service instance | Everything continues through the remaining instances. | Capacity is reduced. |

If the log itself is lost, pings are refused with the `503` described in FR-1, and curves continue to be served.

### FR-12 — Recovery converges to the log

After any component is restored, the counts must converge to what a full recount of the log gives. In particular, after the counter store is lost, the worker rewinds to the log position that was recorded by the last successful durable write, and it re-processes everything from that point onward.

---

## 5. Non-functional requirements

### NFR-1 — Write latency under the premiere load

With admission enabled, and with 50,000 pings per second offered to a single video, climbing from zero over 60 seconds and then holding for 60 seconds, the 99th-percentile response time for accepted (`202`) pings is at most 100 ms.

### NFR-2 — Write availability under the premiere load

Under the same load, every response from the service is either `202` or `429`.

### NFR-3 — Read latency

For a video that is already in the snapshot cache, `GET /v1/curve/{id}` serves 2,000 requests per second with a 99th-percentile response time of at most 20 ms.

### NFR-4 — Freshness

A ping is reflected in the curve within (worker lag + 60 seconds). Worker lag is the time from the moment a ping enters the log to the moment the worker processes it. It is calculated from the timestamp that the log attaches to each message when the message is published. It is measured both at idle and under the premiere load. The 60 seconds is the snapshot refresh interval.

### NFR-5 — Accuracy

After the worker has drained its backlog, the live counts are equal to an exact recount that is computed directly from the log. The difference is zero.

### NFR-6 — Idempotence

Retried pings and re-delivered pings produce zero extra counts. The number of re-deliveries after a forced worker crash is measured and reported, and their effect on the counts is shown to be zero.

### NFR-7 — Durability

Counts survive a restart of every component. The amount of count data that is at risk from an abrupt loss of the counter store is bounded by the durable-write interval, and FR-12 recovers it. The one accepted loss is the viewer records: after the counter store is lost, each active viewer's first ping has no previous record, so rule R2 applies to it, and at most one ping interval of watched time per active viewer goes uncounted. This is stated as a known limit.

### NFR-8 — Throughput for a single hot video

During a premiere, one video receives almost all of the traffic. The system must be able to process that video with several worker threads at the same time, and it must send far fewer commands to the counter store than the number of seconds it credits.

To allow several threads, the pings for a video are spread across several log partitions. Each ping is keyed with the video id together with a number that is derived from the viewer id. A given viewer therefore always lands on the same partition, so that viewer's pings stay in order, while different viewers of the same video are spread across different partitions.

To reduce the number of commands sent to the counter store, the worker reads pings in groups, adds up all of the increments for each (video, second) pair in memory, and then sends one command per changed second.

The effect of each of these two measures is reported as a pair of measurements, one with the measure switched off and one with it switched on.

### NFR-9 — Pre-scaling timeliness

The extra worker capacity that the controller starts is healthy and receiving work at least 10 seconds before the premiere's `startsAt`. The idle capacity-seconds spent waiting are reported as the cost of pre-scaling.

### NFR-10 — Observability

Every quantity in this section is exposed as a metric: request latency histograms per endpoint, admitted and rejected counts per tier, worker lag per partition, counter-store operations per second, snapshot age, durable-write lag, and a counter of re-delivered pings. A dashboard shows all of them live.

### NFR-11 — Honest reporting

Every target in this section is measured, and the measured number is recorded whether or not it meets the target. A target that is missed stays recorded as missed, together with the number and the diagnosed reason. The targets in this document stay fixed while the measurements are being made.

### NFR-12 — Resource envelope

The whole system, including the load generator, runs on one developer machine under Docker Compose, with memory limits on every container. The load generator may alternatively run on a second machine, and when it does, the results say so.

### NFR-13 — Operability

One command starts everything. One command adds or removes worker capacity. Every failure drill, whether it kills a component under load or adds network latency or packet loss between components, is a script that runs with one command.

---

## 6. Fixed numbers

| Name | Value |
|---|---|
| Ping interval | 10 s of playback |
| Normal gap range (R3) | 8–12 s |
| Viewer record expiry | 1 h after the last ping |
| Durable-write interval | 10 s |
| Snapshot refresh interval | 60 s |
| Counter expiry for idle videos | 24 h without writes |
| Log retention | 7 days |
| Log partitions | 12 |
| Per-video spread across partitions | 8 |
| Maximum video length | 21,600 s (6 h) |
| Pre-scale lead time | 30 s before `startsAt` |
| Scale-down delay | 180 s after `startsAt` |

These are the values that the requirements are measured against. If any of them is changed after measurement, the change is recorded in `DECISIONS.md` with the old value, the new value, and the measurement that caused the change.

---

## 7. Design constraints

These are rules about how the system may be built. They limit the design rather than describe a behaviour.

- The ping request path talks only to the log. Every database read and write happens in the worker, the flush job, or the read path.
- All counts are exact integers, as NFR-5 requires.
- A request that is over a limit is rejected immediately with `429`.
- The counting rules exist in exactly one place, as one pure function, and every component that needs the rules calls that function.
- The log is the only record of raw pings. The durable store holds counters, checkpoints, and the premiere schedule, and any recount reads the log.

---

## 8. Stack — decided

| Component | Choice | Role |
|---|---|---|
| Service | Java 21, Spring Boot | Ingest API, worker, read API, admission control |
| Log | Apache Kafka | Durable, ordered record of every ping; the source of truth |
| Counter store | Redis | Live `watched` counters (one hash per video, with one field per second) and the viewer records |
| Durable store | PostgreSQL | Durable copy of the counters, log-position checkpoints, and the premiere schedule |
| Snapshot cache | Caffeine (in-process) | The built curve for each video, refreshed every 60 s and served to all readers |
| Edge | nginx | Spreads load across instances, holds the fleet-wide admission limits, and discovers new instances through Docker DNS |
| Load generation | k6 | Premiere-shaped traffic, viewer behaviour scripts, and duplicate and retry scripts |
| Fault injection | Toxiproxy | Adds latency and packet loss between the service and its stores |
| Metrics | Micrometer, Prometheus, Grafana | Everything that NFR-10 requires |
| Orchestration | Docker Compose | One-command start, scaling, and memory limits |

The reasons for each choice, and the alternatives that were considered, are in `DECISIONS.md`.

---

## 9. End state

Encore is complete when all of the following are true:

1. Every functional requirement from FR-1 through FR-12 is implemented and can be demonstrated.
2. Every non-functional requirement from NFR-1 through NFR-13 has a measured result in `BENCHMARKS.md`, whether that result met the target or missed it.
3. The three headline comparisons exist as tables: admission off versus on under the premiere load; hot-video throughput before and after partition spreading and write batching; and the behaviour of the system with each supporting component killed under load.
4. A reader with no prior context can start the system with one command, run one premiere, kill one component, and watch the dashboard react, all within fifteen minutes.

---

## 10. Glossary

- **Ping** — one position report from a player.
- **Curve** — for one video, the array of `watched` counts, with one entry per second.
- **Watched** — for one second of one video, the number of times a viewer played through that second.
- **Premiere** — a scheduled moment at which one video receives a sudden, large audience.
- **Shed** — to reject a request immediately with `429`.
- **Tier** — a priority group of endpoints that share one limit.
- **Snapshot** — a curve that is built once and served unchanged until the next refresh.
- **Rewind** — moving the worker's log position back to an earlier point so that pings are processed again.
- **Worker lag** — the time between a ping entering the log and the worker processing it.
