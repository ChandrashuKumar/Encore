# Encore — Decisions

This file records every design decision that shapes Encore, together with the alternatives that were considered and the reason each one was set aside. Each entry answers the question "why is it built this way?" so that the answer exists in writing before anyone has to ask it.

Entries are numbered in the order they were made. A status of **Settled** means the decision stands. A status of **Open** means the decision was made on an estimate and will be confirmed or changed once a measurement exists; the measurement that will settle it is named in the entry.

Requirements are referred to by their numbers in `SPEC.md` (FR-n and NFR-n).

---

## Index

| # | Decision |
|---|---|
| D-001 | Encore is one feature, delivered as a standalone service |
| D-002 | The premiere is the design target |
| D-003 | Pings go to a log first; the request path touches no database |
| D-004 | The client reports positions only; the server works out everything else |
| D-005 | Pings are sent every 10 seconds of video time |
| D-006 | A per-session sequence number makes counting idempotent |
| D-007 | The counting rules live in one pure function |
| D-008 | Watched seconds are credited as a half-open range with a tolerance of 8 to 12 |
| D-009 | Version 1 counts watched seconds only |
| D-010 | The log key is the video id plus a number derived from the viewer id |
| D-011 | Redis holds the live counters, one hash per video |
| D-012 | The worker batches increments and writes each batch in one transaction |
| D-013 | Viewer records live in Redis |
| D-014 | PostgreSQL holds a durable copy, written by a periodic flush with checkpoints |
| D-015 | After Redis is lost, the worker rewinds to the last checkpoint |
| D-016 | The log is the only record of raw pings |
| D-017 | Curves are served from an in-process snapshot refreshed every 60 seconds |
| D-018 | Counts are exact |
| D-019 | Overload is handled by priority tiers that reject immediately |
| D-020 | Rate limits exist at the edge and per instance |
| D-021 | A semaphore bounds how many requests wait on the log at once |
| D-022 | The publish to the log has a fixed timeout |
| D-023 | Capacity is added from a schedule ahead of each premiere |
| D-024 | Latency targets were fixed before the first measurement |
| D-025 | Worker lag is measured from the log's own message timestamp |
| D-026 | Java and Spring Boot with virtual threads |
| D-027 | k6 generates load; the single-machine limitation is stated in every result |
| D-028 | Network faults are injected with Toxiproxy in addition to killing containers |

---

## D-001 · Encore is one feature, delivered as a standalone service · **Settled**

**Context.** A video platform has many parts: storage, streaming, accounts, recommendations, comments, playback. Building several of them produces a broad system in which no single part is done well enough to prove anything.

**Decision.** Encore implements exactly one feature, the "most replayed" curve, and it is built so that any platform could plug it in by sending position pings and reading curves. It needs only viewer ids and video ids from the platform.

**Why.** A narrow boundary makes every claim testable. The whole input is one message type, so the load generator can produce realistic traffic without a platform behind it, and every failure drill can be run against the complete system.

**Alternatives considered.** A fuller platform backend was considered and set aside because it would spread effort across parts that have no bearing on the premiere problem.

---

## D-002 · The premiere is the design target · **Settled**

**Context.** A service that only has to handle steady traffic can be built with a request handler that writes to a database. The interesting engineering appears when traffic changes faster than the system can react.

**Decision.** Every design choice is judged against one scenario: a single video that goes from no viewers to the full audience in about 60 seconds, with all traffic going to that video.

**Why.** The scenario is real for any platform that releases content at a fixed time, it has a known start time (which allows preparation), and it concentrates traffic on one key (which defeats the usual way of spreading load across many keys). It therefore exercises the write path, the hot-key problem, and the read path at the same time.

**Alternatives considered.** A steady high load with traffic spread across many videos was considered. It is easier and less informative, because partitioning by video solves it on its own.

---

## D-003 · Pings go to a log first; the request path touches no database · **Settled**

**Context.** The simplest write path takes a ping, updates counters in a database, and then responds. Under the premiere load, every request would wait on the database, requests would queue, and the queue would grow until the service stopped responding. Supports FR-1.

**Decision.** The ingest endpoint publishes the ping to Kafka and responds `202` as soon as the broker confirms the write. A separate worker reads the log and updates counters at its own pace.

**Why.** Appending to a log is a sequential write and is fast under any load. The response time of the ingest endpoint then depends only on the log, and a slow or failed counter store or durable store has no effect on whether pings are accepted. The log is also a complete, ordered record, which makes recounting (NFR-5) and rebuilding (FR-9) possible.

**Alternatives considered.** Writing directly to Redis from the request handler was considered. It is faster than a database but it still ties the response to Redis, so a Redis outage would refuse pings and there would be no record to rebuild from. A write-ahead queue in the application's own memory was considered and set aside because a crash would lose it.

---

## D-004 · The client reports positions only; the server works out everything else · **Settled**

**Context.** A player knows when the user seeks, pauses, or replays, and it could send an event for each. The server would then trust those events. Supports FR-2 and FR-3.

**Decision.** The only message a player sends is its current position. Seeks, pauses, and replays are inferred by the server from the gap between consecutive positions.

**Why.** The interpretation of what a viewer did lives in one place, the server, so a change to the rules is a server change and reaches every installed client at once. Old client versions keep producing correct data. Any player that can report a position can integrate, and the ingest endpoint has one message schema to validate. A client that sends wrong positions can only distort one sample at a time, whereas a client that sends a wrong seek event could claim a whole watched range.

**Alternatives considered.** Explicit events (`seek from A to B`, `pause`, `play`) were considered. They would make the worker simpler and would allow a seek to carry the position the viewer left from. They were set aside because they move the meaning of the data into client code that cannot be updated on demand, and because event-based protocols need ordering rules (a seek must follow a play, a pause must have a matching play) that both sides can get wrong.

**Consequences.** The worker keeps a small record per (viewer, video), which costs one Redis read per ping (see D-013). A long buffering stall that pushes a ping outside the normal gap is read as a skip; this is a documented edge case.

---

## D-005 · Pings are sent every 10 seconds of video time · **Settled**

**Context.** The ping interval sets the traffic volume and the resolution of what the server knows. Supports FR-2.

**Decision.** A player sends a ping each time 10 more seconds of the video have been played. The interval is measured in video time, so a paused player sends nothing and a player at 2x speed still sends pings 10 video-seconds apart.

**Why.** One ping per second per viewer would be ten times the traffic for no gain, because the server can credit the seconds between two pings when the gap is normal. Measuring in video time removes playback speed from the counting rules entirely: a normal gap is always about 10, whatever the speed.

**Alternatives considered.** A real-time interval was considered. It would require the ping to carry the playback rate and the worker to scale the expected gap by it, and a paused player would keep sending pings with no new information.

---

## D-006 · A per-session sequence number makes counting idempotent · **Settled**

**Context.** The same ping can reach the worker more than once: a player retries when a response is lost, and Kafka re-delivers messages after a worker crash because it guarantees delivery at least once. Two pings from one viewer can also arrive out of order if they take different network paths. Supports FR-3 (rule R1) and NFR-6.

**Decision.** Every ping carries a sequence number that the player increases by one per ping within a session, and a retry carries the same number as the original. The worker ignores any ping whose sequence is less than or equal to the last sequence it stored for that viewer and video.

**Why.** One integer comparison handles retries, re-deliveries, and reordering without the worker needing to know which of the three happened. Any ping that is ignored can only cause under-counting of at most one stretch; it can never cause a second credit. The viewer record and the counters are written in one atomic step, so a re-delivered batch finds the stored sequence already advanced and is ignored in full.

**Alternatives considered.** Deduplicating by position was considered and set aside because a genuine rewind to a position seen before would be discarded as a duplicate. Kafka's exactly-once transactions were considered and set aside because they add coordination cost on every message and still leave client retries unhandled. A table of seen message ids was considered and set aside because the sequence number gives the same protection with a single stored integer per viewer.

---

## D-007 · The counting rules live in one pure function · **Settled**

**Context.** The rules that turn a ping into credited seconds are the core of the service. If they exist in two places, the tested version and the running version can drift apart. Supports FR-3.

**Decision.** The rules are one function that takes the ping and the viewer record and returns the seconds to credit and the new record. It performs no input or output. The worker calls this function for every ping, and the unit tests exercise this same function.

**Why.** The function can be tested for every rule and every edge case in milliseconds, and whatever the tests prove is exactly what runs in production. Replay (FR-9) is guaranteed to reproduce the same counts because the function is deterministic.

**Alternatives considered.** Expressing the rules in a SQL statement (an upsert with a condition) was considered. It would make the database enforce the rule atomically, but it would leave the Java version of the rule untested in production and the SQL version untested in unit tests.

---

## D-008 · Watched seconds are credited as a half-open range with a tolerance of 8 to 12 · **Settled**

**Context.** With pings 10 seconds apart, the worker learns the viewer's position only at those moments and must decide what happened in between. Supports FR-3 (rule R3).

**Decision.** When the gap between consecutive positions is between 8 and 12 inclusive, the worker credits every second from the previous position up to but excluding the new position. Any other gap credits nothing.

**Why.** The half-open range gives each boundary second to exactly one pair of pings, so a viewer who watches continuously credits every second exactly once. The tolerance of plus or minus 2 seconds absorbs player timers that fire slightly early or late. A gap above 12 cannot be explained by continuous playback at the agreed interval, so it is a skip, and crediting the seconds in between would be wrong.

**Alternatives considered.** Requiring a gap of exactly 10 was considered and set aside because ordinary timer drift would be misread as a skip. Crediting only the single second named in each ping was considered and set aside because the curve would then be sampled at 10-second resolution and would under-count by a factor of ten.

---

## D-009 · Version 1 counts watched seconds only · **Settled for v1**

**Context.** A viewer who seeks to a particular second and then pauses there has shown interest in that second, but under the counting rules that second gets no credit, because credit is given only when the next normal ping proves the viewer kept watching.

**Decision.** Version 1 keeps a single counter per second, `watched`. A second counter for seek targets, and the extra client ping that would be needed to see a seek followed by a pause, are recorded as future work in `SPEC.md`.

**Why.** The single counter is enough to prove every promise in the specification, and every added signal brings its own client contract change, its own noise-handling rule, and its own test. Shipping the core with measurements first keeps the project honest about what has actually been proven.

**Alternatives considered.** Adding a `landed` counter in version 1 was considered. It would make the curve more expressive, and it remains the first candidate for version 2.

---

## D-010 · The log key is the video id plus a number derived from the viewer id · **Settled**

**Context.** Kafka processes each partition with one consumer thread, and messages with the same key always go to the same partition. The counting rules need each viewer's pings to be processed in order. During a premiere, one video carries almost all of the traffic. Supports NFR-8.

**Decision.** The key of each message is the video id followed by a number from 0 to 7 computed as a hash of the viewer id modulo 8.

**Why.** A viewer always produces the same key for the same video, so that viewer's pings always go to one partition and are processed in order by one thread. Different viewers of the same video produce different numbers, so the video's traffic is spread across up to 8 partitions and up to 8 threads can process the premiere video at the same time. The rules compare a viewer only with that same viewer's previous ping, so ordering across different viewers is never needed.

**Alternatives considered.** Keying by video id alone was considered. It keeps every ping for a video in one partition, which is more order than the rules need, and it limits the premiere video to one thread. Keying by viewer id alone was considered. It spreads load evenly but it mixes every video into every partition, which makes per-video operations such as the flush less efficient and gives no way to reason about one video's throughput.

**Consequences.** The number 8 is a starting value. It bounds the parallelism available to one video and will be revisited if the hot-video measurement shows a single video saturating 8 threads.

---

## D-011 · Redis holds the live counters, one hash per video · **Settled**

**Context.** At 50,000 pings per second with up to 10 seconds credited per ping, the counters receive up to 500,000 increments per second, almost all on one video. Supports NFR-1 and NFR-8.

**Decision.** The live counters are kept in Redis as one hash per video, with the second number as the field and the count as the value. The worker increments fields in this hash.

**Why.** Redis performs small in-memory increments far faster than a relational database performs row updates, and a hash keeps all of one video's counters under one key so that the read path can fetch a whole curve with a single command.

**Alternatives considered.** Keeping the counters in PostgreSQL was considered and set aside because row-level updates at this rate on one video's rows would serialise on locks and saturate the disk. One Redis key per (video, second) was considered and set aside because reading a curve would then take thousands of commands.

**Consequences.** Redis is in-memory, so it needs a durable copy (D-014) and a recovery path (D-015).

---

## D-012 · The worker batches increments and writes each batch in one transaction · **Settled**

**Context.** Sending one Redis command per credited second would mean up to 500,000 commands per second, which is beyond what a single Redis instance handles. Supports NFR-8.

**Decision.** The worker reads a batch of pings from Kafka, applies the counting rules to each one in order, adds up all of the resulting increments per (video, second) in memory, and then sends the summed increments and the updated viewer records to Redis as one pipelined transaction. The Kafka offset for the batch is committed only after the transaction succeeds.

**Why.** During a premiere, thousands of viewers are at roughly the same position, so most increments in a batch land on the same few seconds and collapse into a few commands. The single transaction makes the counters and the viewer records change together, which is what lets rule R1 ignore a re-delivered batch safely.

**Alternatives considered.** A Lua script in Redis that applies the rules and increments atomically was considered. It would move the rules out of the pure function (against D-007) and would still execute one script per ping.

**Consequences.** Within one batch, the worker must apply pings for the same viewer in order against an in-memory copy of that viewer's record, so that the second ping sees the first ping's update.

---

## D-013 · Viewer records live in Redis · **Settled**

**Context.** The worker needs each viewer's last position and sequence on every ping. Supports FR-3.

**Decision.** Each (viewer, video) record is a small Redis hash with a 1-hour expiry that is refreshed on every write.

**Why.** Kafka reassigns partitions between worker instances whenever instances are added or removed, including during pre-scaling. A record kept only in one instance's memory would be lost at every reassignment, and every active viewer on that partition would fall into rule R2 and lose a stretch. Redis is shared by all worker instances, so whichever instance picks up a partition finds the records already there.

**Alternatives considered.** An in-process map was considered and set aside for the reason above. A Kafka Streams local state store with changelog topics was considered. It solves the reassignment problem but it brings a second framework and its own operational model into a project whose point is to show the mechanics plainly.

**Consequences.** One Redis read per ping, reduced to one multi-key read per batch by D-012. The reads are a measured cost in the hot-video benchmark.

---

## D-014 · PostgreSQL holds a durable copy, written by a periodic flush with checkpoints · **Settled**

**Context.** Redis is memory-only and will eventually drop old videos. The curve for a video must survive restarts and remain available for the life of the video. Supports NFR-7 and FR-11.

**Decision.** Every 10 seconds, a flush job copies the counters of every video that changed since the last flush from Redis into a PostgreSQL table with one row per (video, second), and records in a checkpoint table the Kafka offset per partition up to which the counts are fully reflected.

**Why.** The flush turns a stream of hundreds of thousands of increments per second into a batch of upserts every 10 seconds, which PostgreSQL handles comfortably. The checkpoint ties the durable state to a precise position in the log, which is what makes recovery exact (D-015). The read path can serve any video from PostgreSQL when Redis does not have it.

**Alternatives considered.** Writing through to PostgreSQL on every increment was considered and set aside because it reintroduces the database bottleneck that D-003 removed. Running with Redis persistence alone and no PostgreSQL was considered and set aside because Redis would then have to hold every video forever and a single Redis failure would be the only line of defence.

---

## D-015 · After Redis is lost, the worker rewinds to the last checkpoint · **Settled**

**Context.** If Redis loses its data, every increment since the last flush is gone from the live counters, while the log still holds every ping. Supports FR-12.

**Decision.** When the worker starts and finds Redis empty, it loads each video's counters from PostgreSQL on first touch, reads the checkpoint table, and resets each partition's Kafka position to the recorded offset. It then re-processes every ping from that offset onward.

**Why.** Every ping after the checkpoint had its effect only in Redis, which is now gone, so re-processing those pings adds each effect exactly once. Every ping before the checkpoint is already reflected in PostgreSQL. The result converges to the exact log with no operator judgement involved.

**Alternatives considered.** Relying on Redis append-only persistence alone was considered. It bounds the loss to about one second but it is a property of one component's configuration rather than of the system, and it does nothing when the Redis volume itself is lost. It is enabled in addition, as a second line of defence. Accepting the loss of up to 10 seconds of counts was considered and set aside because the accuracy requirement is a difference of zero.

**Consequences.** The viewer records are also lost with Redis and are not checkpointed, so each active viewer's first ping after recovery falls into rule R2 and one stretch per viewer goes uncounted. This is the one accepted loss and it is stated in NFR-7.

---

## D-016 · The log is the only record of raw pings · **Settled**

**Context.** A table of every ping in PostgreSQL would allow recounts with SQL, at the cost of one row insert per ping.

**Decision.** Raw pings are kept only in Kafka, with a retention of 7 days. Recounts and replays read Kafka.

**Why.** One insert per ping would put the same write load on PostgreSQL that D-003 removed from the request path, only one step later. Kafka already holds every ping in order, and a recount tool that reads Kafka is the same code path that the worker uses, which is a stronger check than a separate SQL query.

**Alternatives considered.** A partitioned append-only table was considered. It is a reasonable audit log for lower volumes and was set aside at the premiere rate.

**Consequences.** A recount is only possible within the retention window. Any video that needs a recount after 7 days relies on the PostgreSQL copy.

---

## D-017 · Curves are served from an in-process snapshot refreshed every 60 seconds · **Settled**

**Context.** During a premiere, every player on the video fetches the same curve, and the curve for a 3-hour video has 10,800 values. Supports NFR-3 and FR-11.

**Decision.** The read path builds the curve once per video per instance, holds it in a Caffeine cache for 60 seconds, and serves the same array to every request in that window. The build reads the whole hash from Redis in one command, or the rows from PostgreSQL when Redis does not have the video.

**Why.** Thousands of reads per second of a 10,800-field hash would make Redis the read bottleneck. A snapshot costs one Redis read per minute per instance regardless of how many players ask. When Redis is down, the last snapshot keeps serving until it expires, and after that the build falls through to PostgreSQL.

**Alternatives considered.** Reading Redis on every request was considered and set aside for the load reason. Pushing updated curves into the cache from the worker was considered and set aside because it couples the worker to every read instance and gives at most a few seconds of extra freshness for a curve that changes slowly.

**Consequences.** A ping becomes visible to readers within worker lag plus 60 seconds, which is the freshness requirement in NFR-4. The 60-second value is a starting point and will be revisited with the measured read load.

---

## D-018 · Counts are exact · **Settled**

**Context.** Approximate structures such as HyperLogLog or count-min sketches use less memory and are common in counting systems.

**Decision.** Every count is an exact integer.

**Why.** The data model is small enough that exactness costs nothing: one integer per second per video. Exact counts allow the accuracy requirement to be a difference of zero against a recount, which is a stronger and simpler claim than a bounded error. Exactness is also what makes replay (FR-9) produce byte-identical output.

**Alternatives considered.** Approximate counting was considered and set aside because the memory it saves is not needed and the error it introduces would weaken every accuracy claim.

---

## D-019 · Overload is handled by priority tiers that reject immediately · **Settled**

**Context.** Under the premiere load, more requests arrive than the system can serve. A system that accepts all of them lets a queue build, and a queued request wastes a thread and memory for seconds before timing out. Supports FR-8, NFR-1 and NFR-2.

**Decision.** Requests are grouped into tiers with separate limits. A request that exceeds its tier's limit is rejected at once with `429` and `Retry-After: 1`. Ping writes are the highest tier and are rejected last; peak reads are the lowest and are rejected first.

**Why.** A fast rejection costs the server almost nothing and tells the client exactly when to retry. The tiering means that under overload the service degrades in a chosen order: the ping, which is the data, keeps flowing while reads that only decorate the player are shed. Each tier's limit is a token bucket, which allows a short burst and then enforces the rate.

**Alternatives considered.** Queuing with backpressure was considered and set aside because a queue is where latency grows. A single global limit was considered and set aside because it would shed pings and reads in arrival order, with no control over which data is lost.

---

## D-020 · Rate limits exist at the edge and per instance · **Settled**

**Context.** The service runs as several instances behind nginx, and the number of instances changes during pre-scaling. Supports FR-8.

**Decision.** Each instance applies its own tier limits, sized to protect that instance's CPU. nginx applies the same three tier limits for the whole fleet, sized to what the system as a whole can take.

**Why.** A limit inside an instance multiplies when instances are added: three instances with 8,000 pings per second each admit 24,000 pings per second in total, which may be more than Kafka, Redis, and the worker can handle. nginx is the one component that exists once, so a limit placed there is the system's true ceiling. The per-instance limit is still needed so that one instance receiving more than its share is protected on its own.

**Alternatives considered.** A shared limiter backed by Redis was considered. It gives one global count but puts a Redis round trip on every request and makes the limiter unavailable when Redis is down.

---

## D-021 · A semaphore bounds how many requests wait on the log at once · **Settled, size Open**

**Context.** With virtual threads, the application can hold a very large number of requests in flight at the same time. A tier limit bounds the arrival rate, but it does nothing about how many requests are waiting inside the Kafka publish when Kafka becomes slow. The number waiting equals the arrival rate multiplied by the wait time, so a slow Kafka turns a steady rate into an unbounded pile of waiting requests. Supports NFR-2.

**Decision.** The ingest endpoint takes a permit from a semaphore before publishing and returns it after the broker confirms. When no permit is free, the request is rejected with `429`.

**Why.** The semaphore caps concurrency, which is the quantity that grows when a dependency slows down. It turns "Kafka is slow" into fast rejections instead of thread and memory exhaustion.

**Sizing.** The permit count is derived from a measurement: the admitted ping rate multiplied by the 95th-percentile Kafka confirm time under the premiere load, multiplied by a safety factor of about five. The initial value is 1,000 and is marked Open until that measurement exists.

**Alternatives considered.** Relying on a bounded platform-thread pool was considered. It bounds concurrency as a side effect, but it queues excess requests instead of rejecting them, and it is removed by the switch to virtual threads (D-026).

---

## D-022 · The publish to the log has a fixed timeout · **Settled**

**Context.** A Kafka client that cannot reach the broker will, by default, keep trying for about a minute before it reports failure. During that time, the ingest request is held open. Supports FR-1 and FR-11.

**Decision.** The publish call waits for at most a fixed timeout (initially 2 seconds) and then the endpoint responds `503`.

**Why.** A held-open request consumes a semaphore permit and a thread for the whole minute, and the client learns nothing until then. A bounded `503` releases resources and tells the client to retry; the retry carries the same sequence number, so no count is affected.

**Alternatives considered.** Keeping the default was considered and set aside because a broker outage would then make every ingest request hang for a minute.

---

## D-023 · Capacity is added from a schedule ahead of each premiere · **Settled**

**Context.** Autoscaling that reacts to load measures the load, decides to scale, starts new containers, and waits for them to become healthy. That takes longer than the 60-second climb of a premiere. Supports FR-10 and NFR-9.

**Decision.** Premieres are registered with a start time and an expected peak rate. A controller polls the schedule and starts the needed worker capacity a fixed lead time before the start, then removes it a fixed delay after the start. The lead time is the sum of the poll interval, the measured container startup time, and a safety margin.

**Why.** The start time is known in advance, so the system can be ready rather than catching up. The controller also records the idle capacity-seconds spent waiting, which makes the cost of early scaling a measured number.

**Alternatives considered.** Reactive autoscaling on CPU or queue depth was considered and set aside because its reaction time is longer than the event it would react to. Running at peak capacity permanently was considered and set aside because its cost is the thing pre-scaling exists to avoid.

---

## D-024 · Latency targets were fixed before the first measurement · **Settled**

**Context.** A target chosen after measuring is whatever the system happened to achieve, and proves nothing. Supports NFR-1, NFR-3 and NFR-11.

**Decision.** The write target (99th-percentile at most 100 ms for accepted pings at 50,000 per second) and the read target (99th-percentile at most 20 ms at 2,000 reads per second from the snapshot) were set before any load test was run, and they stay fixed while the measurements are made.

**Why.** The write path costs a few milliseconds when calm. The 100 ms ceiling is low enough that any queueing in the producer, the broker, or the thread pool breaks it, and high enough that garbage-collection pauses and CPU contention on a single machine do not. The read path on a cache hit costs a millisecond or two, and the 20 ms ceiling is tight enough to fail if the snapshot cache is bypassed. Both are round numbers so that a pass or a miss is informative.

**Consequences.** If a target is missed, the measured number and the reason are recorded. A target may be changed afterwards only with a new entry here that names the measurement behind the change.

---

## D-025 · Worker lag is measured from the log's own message timestamp · **Settled**

**Context.** Worker lag is the time a ping spends in the log before the worker processes it. Computing it needs the time the ping entered the log. Supports NFR-4 and NFR-10.

**Decision.** The worker reads the timestamp that Kafka attaches to each message when it is published and subtracts it from the current time.

**Why.** Kafka attaches a server-side timestamp to every message. Using it adds nothing to the ping schema and keeps the client contract exactly as specified.

**Alternatives considered.** Using the player's `clientTs` was considered and set aside because device clocks are often wrong by seconds or minutes, which would make the lag metric meaningless. Adding a server-side `receivedAt` field to the ping at ingest was considered and set aside because the log's timestamp already provides the same value.

---

## D-026 · Java and Spring Boot with virtual threads · **Settled**

**Context.** The service needs HTTP endpoints, a Kafka consumer, Redis and PostgreSQL clients, and metrics.

**Decision.** Java 21 with Spring Boot, with virtual threads enabled for request handling.

**Why.** Spring Boot provides every integration the service needs with little setup, so effort goes into the design rather than the plumbing. Virtual threads let the ingest endpoint hold many requests waiting on Kafka without the memory cost of a platform thread each, which is what makes a high in-flight count possible and what makes the semaphore (D-021) necessary.

**Alternatives considered.** Go was considered. It would make the worker's batching loop slightly more direct and has a lighter runtime. It was set aside because the Java ecosystem has the more mature Kafka client and the broader set of readers who can review the code.

---

## D-027 · k6 generates load; the single-machine limitation is stated in every result · **Settled**

**Context.** Load tests need a generator that can produce a premiere-shaped climb, simulate viewer behaviour (watching, skipping, rewinding), and send deliberate duplicates. Supports NFR-12.

**Decision.** All load tests are k6 scripts. By default, k6 runs on the same machine as the system under Docker Compose. Every recorded result states where the generator ran.

**Why.** k6 expresses arrival-rate profiles and per-virtual-user behaviour in short scripts and reports latency percentiles directly.

**Consequences.** When the generator shares the machine, it competes for CPU with the service, Kafka, Redis, and PostgreSQL, and the measured numbers understate what the design can do. Running the generator on a second machine removes this effect and is the first improvement to make when one is available.

---

## D-028 · Network faults are injected with Toxiproxy in addition to killing containers · **Settled**

**Context.** Killing a container tests a clean failure. Real failures are more often a dependency that is slow or dropping packets while still appearing to be up. Supports FR-11 and NFR-13.

**Decision.** Toxiproxy sits between the service and Redis, and between the service and PostgreSQL. Drill scripts add latency and packet loss on those paths under load, in addition to the scripts that kill containers.

**Why.** A slow Redis is the case that exercises the timeouts, the circuit breaker on the read path, and the semaphore sizing. A dead Redis exercises none of those, because every call fails immediately.

**Consequences.** Each drill records recovery time, the share of pings accepted during the fault, and whether reads kept being served.
