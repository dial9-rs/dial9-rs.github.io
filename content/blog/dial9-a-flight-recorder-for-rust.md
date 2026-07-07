+++
title = "dial9: A flight recorder for Rust"
date = 2026-06-04
authors = ["Russell Cohen"]
+++

> Diagrams by Fable, text by me.

Many performance and correctness problems are easy to solve with enough data. If you can see an exhaustive
ordered log of everything that happened, hard problems tend to become shallow. Trying to figure out what is different about P99 requests vs. P50? If you record all spans from your application, kernel events, and have always-on profiling, it makes it possible to analyze the traces to understand what's going on. These traces are designed to be analyzed programmatically, not just by squinting at a flamegraph. But that's a lot of data.

How do you make it economical to record 1M events per second in prod and get the data off the host for post-hoc analysis? Once you have it, you can get all these data sources together in one place.

Making this possible is the goal of dial9.

### dial9 vs. other formats

How much space and time does it take to record an event?

```json
{ "event": "PollStart", "timestamp": 12323134125, "task_id": 10 }
```

How long does it take to record that event to your logs or tracing? If you're using
`tracing`, it's about 1µs per event. OTel using gRPC `LogRecord` is quicker, closer to 300ns. The amount of data you produce is also a problem. If you want to record every `poll`, you quickly find yourself in the 100Khz-1Mhz 
regime; if you wrote that out to JSON, you'd be emitting almost 1GB / minute. Even if you could encode it efficiently, you'd still have to
be able to get it off the host to analyze it.

With `dial9`, that same event is around 4B and takes 20ns to emit.

{{ serialize_toggle() }}

{{ big_picture() }}

dial9 is designed at each layer to protect the application at the cost of losing profiling data. If data is being produced faster than it can be uploaded, the files rotate out of the ring buffer before they are consumed. If data is being produced faster than the central collector can write it, the oldest batches are dropped. If sources are producing data faster than they can be drained, that data is dropped.

### Data sources

Everything in a dial9 trace is just an event, recorded by pluggable sources:

- **The Tokio runtime**: every poll start/end, wake, task spawn & termination, worker park/unpark, and queue-depth sample
- **A CPU profiler** ([`perf-self-profile`](https://github.com/dial9-rs/dial9/tree/main/perf-self-profile)): sampled stacks, symbolized in the background after the segment is sealed
- **The allocator**: allocation and free events
- **The OS**: process resource usage, TCP accept-queue depth
- **Your application**: anything you `#[derive(TraceEvent)]` on, plus manual spans

You can record as much (or as little) data as you want.

Because the format is self-describing, a new source doesn't need coordination with the viewer: it registers its schemas at runtime and its events show up in the trace.

### The dial9 trace format


The dial9 trace format tries to solve these design problems:

- Fast to encode: dial9 itself should never be the bottleneck.
- Simple to decode: A basic decoder is ~100 lines of dependency free JS.
- Self describing: Everything you need is in the trace file
- Open ended: New event types can be added, even at runtime.
- Small: The amount of data stored should be reasonably compact, especially after compression

{{ trace_format() }}

The [spec](https://github.com/dial9-rs/dial9/blob/main/dial9-trace-format/SPEC.md) for the trace format has a lot more details, but the key idea is that
the trace format contains `SCHEMA` frames that register a schema and a schema id. Then, events that come from
that schema reference the schema id and can write their fields out in order.

**Pools make it space and memory efficient**: The format also has built-in `String` pools and `StackFrame` pools to make it efficient to store
strings and stack traces, both of which are frequently repeated. These pools are thread-local so we can have interning without lock contention at the cost of duplicated pools in the trace file.

**Nanosecond precision timestamps in 3B**: 24-bit nanosecond delta timestamps allow nanosecond-level precision (which would normally take 64 bits) in 24 bits instead.

**Mergeable**: Individual segments of the trace format can be concatenated back to back to form a larger, valid trace file. This allows thread-local buffers that
encode the trace data in-place that then write `~1MB` blocks of data into a central buffer on a much slower cadence.

This effectively removes all lock contention from recording data but we'll get more into that in a moment.


To get data into the trace file you can `#[derive(TraceEvent)]`:

```rust
#[derive(Debug, TraceEvent)]
#[traceevent(wire_slot)] // optimization to pre-allocate a schema id
pub struct PollStartEvent {
    /// Timestamp in nanoseconds.
    #[traceevent(timestamp)]
    pub timestamp_ns: u64,
    /// Worker thread index.
    pub worker_id: WorkerId,
    /// Local queue depth (capped to u8).
    pub local_queue: u8,
    /// Task being polled.
    pub task_id: TaskId,
    /// Interned spawn location.
    pub spawn_loc: InternedString,
}
```

This is how all the data sources in dial9 record data.


### `dial9-core`: The event bus and central collector
The trace format just gets data from structured form to binary; it doesn't solve the problem of getting a recorded event
off the host.

{{ event_bus() }}

`dial9` creates a thread-local buffer for each thread that's recording events. Currently, each TL buffer has 1MB of pre-allocated storage.

The next challenge is getting data from the thread-local buffers into the central buffer where they will eventually be processed
and sent off host.

An early version of dial9 rotated solely based on size. However, this has inherent problems: If a thread is slow to produce data then it will
emit less frequently. If segments are uploaded off host every thirty seconds, you will end up with segments that are not "wall clock aligned"
because a segment from a slow-producer thread will cover many logical segments.

To solve this, dial9 uses a global epoch counter and a mutex on each TL buffer. Every 5ms, the flush thread wakes up to drain sources and flush available data. When it's time to rotate to the next file, the epoch counter is incremented. Then 5ms later, any buffers that _still_ haven't flushed are intrusively flushed by acquiring their lock and draining their data.

This works because [uncontended mutexes are fast](https://www.youtube.com/watch?v=tND-wBBZ8RY). Each TL buffer checks this counter each time an event is pushed. If the counter has increased, it self-flushes its data to the central buffer (an `ArrayQueue`).

This means that busy buffers never need to contend for their own lock (because they are the only ones to ever acquire it). Only relatively idle buffers (where Mutex contention is fine anyway!) need to ever contend for their own mutex.


### Finishing traces & getting them off host

{{ segment_pipeline() }}

Once the data is recorded, we still need to get it off the host. Once an individual file has been sealed, it's processed by the background pipeline to symbolize and persist it. The pipeline is pluggable and includes a default S3 destination.

### Analyzing the data
dial9 includes a few ways out of the box to analyze the data:

**The dial9 viewer**: The dial9 viewer is a Rust program that can be run locally or hosted that serves files from S3 and renders them to look at. This is often what you need, but it's sometimes helpful to go deeper.

**The dial9 toolkit**: Because the traces are designed to be analyzed, dial9 ships with a toolkit of `js` files that can do basic analysis of the traces. Because they're just code, people and agents can tweak them to help dive deep and answer specific questions. You can write a custom analysis to diff one host against another or look at CPU samples that are only impact requests to a specific API endpoint.
