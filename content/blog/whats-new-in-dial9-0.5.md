+++
title = "What's new in dial9 0.5?"
date = 2026-08-04
authors = ["Russell Cohen"]
+++

In dial9's early days it was exclusively a tool to help understand what Tokio was doing. But it turns out that if you build a tool just for Tokio, it actually has [everything you need for it to be a general purpose flight recorder][flight-recorder]. dial9 0.5 makes this vision a reality.

If you aren't coming from 0.3 and just want to know what's supported, dial9 is a flight recorder that supports the following data sources out of the box (and new data sources can be added externally):

1. **Tokio**: poll-start, poll-stop, worker park, worker unpark, scheduling delay, and [task dumps][task-dumps] (`taskdump`, Linux only) — [docs][tokio-events], [example][example-simple]
2. **Heap** (`memory-profiling`): sampled allocations, including liveset tracking for leak detection — [docs][memory-docs], [example][example-memory]
3. **Profiling** (`cpu-profiling`): framepointer unwinding based profiler on Linux and Android with fallback to the ctimer API on containerized environments — [docs][cpu-docs], [example][example-cpu]
4. **Tracing** (`tracing-layer`): `tracing` spans, entered and closed — [docs][tracing-layer-docs], [example][example-tracing]
5. **metrique** (`metrique-sink`): unit-of-work events & spans with per-request context — [docs][metrique-sink-docs], [example][example-metrique]
6. **Linux Kernel Resource Usage** (`process-resource`, unix): rss, page faults, and the rest of `getrusage` — [docs][process-docs], [example][example-process]
7. **Socket Accept Queues** (`linux-socket`, Linux only): TCP accept-queue depth — [docs][socket-docs], [example][example-socket]

### The main crate is now [`dial9`][dial9-crate] (was previously `dial9-tokio-telemetry`)
Tokio is now "just one more source" for dial9. So instead of using `dial9-tokio-telemetry`, your application will now depend on [`dial9`][dial9-crate] and enable the `tokio` feature.

```toml
[dependencies]
dial9 = { version = "0.5.0-rc2", features = ["tokio"] }
```

The facade ships no default features, so a library dependency pulls in no viewer or CLI weight. Add the source features from the table above as you need them, plus `worker-s3` if you want the S3 upload stage.

This also means that the macro is now [`#[dial9::main]`][dial9-main] and you can [`dial9::spawn`][dial9-spawn] a new task. To launch with the least code possible use [`recorder_from_env`](todo-link):
```rust
use dial9::Dial9TokioHandle;
// Load configuration via environment variables:
#[dial9::main(config = dial9::recorder_from_env)]
async fn main() {
    let handle = Dial9TokioHandle::current();
    handle.spawn(async { /* wake events tracked when enabled */ }).await.unwrap();
}
```

### The `dial9` viewer now supports aggregated spans, flamegraphs, and Tokio stats
The dial9 viewer does _technically_ still work as a HTML-only webpage, but if you run it via `dial9 serve` and your traces are in S3 (or anywhere that implements the [`StorageBackend`][storage-backend] trait), the dial9 viewer can produce [flamegraphs, flamegraph diffs, and Tokio stats][aggregator-doc] across long time ranges and multiple hosts.

This works without any offline aggregation; when you open a flamegraph, it starts loading a deterministic sampling of your uploaded segments. As more segments are loaded, the flamegraph (or span histogram, or Tokio stats) will be incrementally refined, but usually, the changes after say, 2-3% of the data is loaded is pretty minor as long as the sampling is uniform.

Its also suitable for being hosted on your own infrastruture (behind auth, of course).

```bash
cargo binstall dial9@0.5.0-rc2

# Aggregate over a local directory of traces
dial9 serve --local --agg-source-dir /tmp/dial9-traces

# ...or over S3
dial9 serve --local --bucket my-traces --prefix traces/ --agg
```

(`--local` here means "render logs human-readably for a workstation" — the trace source is `--agg-source-dir` or `--bucket`. There's also `dial9 serve --simulator` if you just want to poke at a synthetic trace.)

### Trigger mode lets you upload only anomolous data
Production applications using dial9 often produce upwards of 1MB/second. This is a lot to store especially if the data doesn't have anything interesting. dial9 now supports a rotating ring buffer of data that is only flushed when triggered from the application. For example, if your application hits an anomolous condition or has a sudden increase in load, you can trigger a dump to be flushed to disk s3, or any other destination.

[`with_dump_trigger`][with-dump-trigger] gates *when* the pipeline runs, not *what* it does — whatever pipeline you'd have run continuously is the pipeline a dump runs.

```rust
use std::time::Duration;
use dial9::{Dial9Handle, DiskBuffer, Dial9HandleTokioExt, RecorderPipelineExt, TokioAttachOptions};

#[dial9::main(config = || {
    let writer = DiskBuffer::builder()
        .base_path("/tmp/dial9-traces")
        .max_total_size(10 * 1024 * 1024)
        .build()?;

    let recorder = dial9::recorder(writer)
        // Whatever pipeline you'd run continuously. `with_dump_trigger`
        // only changes *when* it runs.
        .with_custom_pipeline(|p| p.gzip().write_back_to("/tmp/dial9-dumps"))
        // Coalesce a re-tripping watcher's burst into one dump.
        .with_dump_trigger(|t| t.debounce(Duration::from_secs(30)))
        .build();

    let mut builder = tokio::runtime::Builder::new_multi_thread();
    builder.enable_all().worker_threads(2);
    let runtime = recorder
        .handle()
        .attach_tokio_runtime(builder, TokioAttachOptions::default())?;
    Ok((recorder, runtime))
})]
async fn main() {
    // Reach the trigger through the ambient handle from any runtime thread:
    // a monitor task, a panic hook, a `/dump` handler.
    let trigger = Dial9Handle::current()
        .dump_trigger()
        .expect("on-demand mode enabled");

    // ... your app runs; segments accumulate in the ring, pipeline stays parked ...

    // Something looks wrong: keep it.
    let receipt = trigger
        .dump_current_data()
        .with_metadata("reason", "idle-ratio-drop")
        .await?;

    println!("dump {} captured {} segments", receipt.dump_id, receipt.segments_processed);
}
```


See [`on_trigger_dump.rs`][example-trigger] and [`on_trigger_dump_windows.rs`][example-trigger-windows] for examples.

### In-memory buffer skips writing to disk
The buffering of in-flight tracefiles has been overhauled to now have two options:
- [`DiskBuffer`][disk-buffer]: Same as dial9 0.3.0, a directory on disk holds trace files in flight prior to being uploaded
- [`MemoryBuffer`][memory-buffer]: A new mode that buffers traces in memory. In practice, this does not increase memory usage since the trace needed to be loaded into memory to symbolize it anyway. The only downside is that traces are not durable in the event of an application panic or crash. This is the recommended mode for most applications.

### CPU profiling now works on Android!
Thanks to [@nickrobinson][nickrobinson] for [landing an upstream PR to `libc`][libc-pr] & [updating dial9][android-pr]. dial9 now works great end-to-end on Android! Android needs special handling because the Android runtime owns `SIGSEGV` through `libsigchain`, so the PR adds the platform-specific signal and context handling needed for safe frame-pointer unwinding. It's been running at Ditto behind a feature flag on production Android devices.

### Everything is a [`Source`][source-trait]
In dial9 0.3, Tokio and CPU profilers were both deeply integrated to provide data into dial9. In 0.5 they're [ordinary `Source` implementations][tokio-source-pr] on a plain `Recorder`, which means the profiling features no longer pull in `tokio` at all, and you can plug in your own source without touching dial9's internals. A `Source` can also contribute a stage to the segment pipeline — that's how enabling CPU profiling automatically wires up symbolization without the caller doing anything.

### Memory profiling liveset tracking is now much faster
dial9's memory profiler previously wrote a sampled set of allocations and _all_ frees into a ring buffer which the flush thread then drained. The problem was that writing all frees into one `crossbeam` ArrayQueue became a bottleneck for some applications. [The new memory profiler][memory-profiling-pr] uses `scc` on the allocator side to filter allocations
prior to writing them into the ring buffer. This makes free-set tracking much more usable in production environments (but you should obviously still benchmark it for your application!) Since the size of the hashmap is only dependent on the sampled liveset, running at very low sampling rates should result in very low performance overhead.

### metrique integration for events & spans

[`dial9-metrique`][dial9-metrique] contains a [`Dial9Stream`][dial9-stream] that can send [metrique][metrique] events directly to your dial9 traces ([full example][metrics-service-example]):

```rust
let metrics_join = ServiceMetrics::attach_to_stream(Dial9Stream::tee(
    recorder.handle(),
    LocalFormat::new(OutputStyle::Pretty).output_to(request_metrics_file),
));
```

Then you can emit metrics to `ServiceMetrics` and the same records will go both to your main output and dial9.

### Process resource usage & Socket Accept Queues (now opt-in)
Kernel resource usage sampling (rss, page faults) is now behind the `process-resource` feature, and [socket accept queue sampling][socket-pr] is behind `linux-socket`. **Note for upgraders:** rusage sampling was on by default on Unix in 0.3, so if you want to keep those events you need to enable the feature explicitly.

[flight-recorder]: https://dial9-rs.github.io/blog/dial9-a-flight-recorder-for-rust/
[dial9-crate]: https://crates.io/crates/dial9
[tokio-events]: https://docs.rs/dial9/0.5.0-rc2/dial9/analysis/analysis_events/index.html
[task-dumps]: https://docs.rs/dial9/0.5.0-rc2/dial9/struct.TaskDumpConfig.html
[example-simple]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/simple_workload.rs
[memory-docs]: https://docs.rs/dial9/0.5.0-rc2/dial9/memory/index.html
[example-memory]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/examples/memory-local/src/main.rs
[cpu-docs]: https://docs.rs/dial9/0.5.0-rc2/dial9/cpu/index.html
[example-cpu]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/cpu_profile_workload.rs
[tracing-layer-docs]: https://docs.rs/dial9/0.5.0-rc2/dial9/tracing_layer/struct.Dial9TracingLayer.html
[example-tracing]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/tracing_sleep.rs
[metrique-sink-docs]: https://docs.rs/dial9-metrique/0.5.0-rc2/dial9_metrique/
[example-metrique]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/metrique_metrics.rs
[process-docs]: https://docs.rs/dial9/0.5.0-rc2/dial9/process/index.html
[example-process]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/examples/metrics-service/src/main.rs#L210
[socket-docs]: https://docs.rs/dial9/0.5.0-rc2/dial9/socket/index.html
[example-socket]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/examples/metrics-service/src/main.rs#L228
[dial9-main]: https://docs.rs/dial9/0.5.0-rc2/dial9/attr.main.html
[dial9-spawn]: https://docs.rs/dial9/0.5.0-rc2/dial9/fn.spawn.html
[source-trait]: https://docs.rs/dial9/0.5.0-rc2/dial9/core/trait.Source.html
[storage-backend]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9-viewer/src/storage.rs#L85
[aggregator-doc]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/aggregator.md
[with-dump-trigger]: https://docs.rs/dial9/0.5.0-rc2/dial9/struct.RecorderBuilder.html#method.with_dump_trigger
[dump-id]: https://docs.rs/dial9/0.5.0-rc2/dial9/core/dump/struct.DumpId.html
[example-trigger]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/on_trigger_dump.rs
[example-trigger-windows]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/dial9/examples/on_trigger_dump_windows.rs
[disk-buffer]: https://docs.rs/dial9/0.5.0-rc2/dial9/type.DiskBuffer.html
[memory-buffer]: https://docs.rs/dial9/0.5.0-rc2/dial9/type.MemoryBuffer.html
[nickrobinson]: https://github.com/nickrobinson
[libc-pr]: https://github.com/rust-lang/libc/pull/5108
[android-pr]: https://github.com/dial9-rs/dial9/pull/685
[tokio-source-pr]: https://github.com/dial9-rs/dial9/pull/698
[memory-profiling-pr]: https://github.com/dial9-rs/dial9/pull/495
[dial9-metrique]: https://docs.rs/dial9-metrique/0.5.0-rc2/dial9_metrique/
[dial9-stream]: https://docs.rs/dial9-metrique/0.5.0-rc2/dial9_metrique/struct.Dial9Stream.html
[dial9-context]: https://docs.rs/dial9-metrique/0.5.0-rc2/dial9_metrique/struct.Dial9Context.html
[metrique]: https://docs.rs/metrique
[metrics-service-example]: https://github.com/dial9-rs/dial9/blob/dial9-v0.5.0-rc2/examples/metrics-service/src/main.rs#L276-L279
[metrique-pr]: https://github.com/dial9-rs/dial9/pull/723
[socket-pr]: https://github.com/dial9-rs/dial9/pull/506
[schedstat-pr]: https://github.com/dial9-rs/dial9/pull/619
