+++
title = "Principles for fast Tokio applications"
date = 2026-09-13
authors = ["Russell Cohen"]
draft = true
+++

> I'm on my way back from RustConf. At the Unconf, we had a productive discussion about debugging and benchmarking async applications. Many interesting insights were shared. I'm attempting to enumerate some of them here, along with some of my own experiences. This is the first draft of what I hope can become a living document of best practices. Feel free to file an issue or open a PR!
>
> — Russell

There are few hard-and-fast rules for writing code that performs well on Tokio runtimes; the answer to so many questions is "it depends." The performance of a workload depends on what else is running on the runtime at that moment. This is why so many problems only show up in production! Writing async applications that perform well is a balance between fairness and batching, contention and isolation.

This post lays out some general principles and covers exceptions where I can. It assumes basic familiarity with Tokio's work-stealing runtime; a high-level summary is included in [the appendix](#appendix-a-mental-model-for-tokio-in-four-bullet-points).

## General principles

### First, determine whether you have a problem

If you start looking for red flags in a Tokio application, you will find them. Almost every real application I have seen has polls [(the time between `.await` when the code yields back to the runtime)](#appendix-a-mental-model-for-tokio-in-four-bullet-points) much longer than the 10-100 microseconds Alice recommends in her excellent post [What is Blocking?](https://ryhl.io/blog/async-what-is-blocking/). These problems may or may not affect the application metrics or behavior you actually care about (see: [long polls can be fine sometimes](#blocking-the-executor-can-be-fine-sometimes)). It is important to work backward from a real metric you are trying to improve. For example, an application can have long polls that are completely benign; "fixing" them will not measurably impact user-facing metrics.

In the overwhelming majority of problems I have come across, the issue was in the application code itself, often in the interaction between multiple components of a distributed system (and not actually in Tokio). dial9 has given a lot of visibility into Tokio; at least as often as it finds a Tokio problem, it actually clearly demonstrates the _lack_ of one (which gives folks the confidence to search elsewhere!) Of course, sometimes it is a Tokio problem.

### Split for latency, batch for throughput

#### Yield more frequently to optimize for latency

Low latency across many requests requires fairness between connections.

Consider Redis (or any application that supports request pipelining). A naive implementation will read data directly off the connection while more data is available. If requests are pipelined though, that data is going to be `Ready` for the entire pipelined request all at once. This creates both long polls and unfairness between clients.

Throughput is largely unchanged: the same number of requests are processed. Latency, however, changes dramatically because one entire pipeline can wait behind another. Explicitly yielding after each request can reduce latency by roughly 10× in this example. You can do even better by yielding only after several consecutive immediately-ready reads.

```rust
async fn handle_conn(&mut self) -> crate::Result<()> {
    while !self.shutdown.is_shutdown() {
        // If the connection has buffered data, this can repeatedly return
        // Poll::Ready without yielding back to the runtime.
        let frame = tokio::select! {
            res = self.connection.read_frame() => res?,
            _ = self.shutdown.recv() => {
                return Ok(());
            }
        };

        execute_command(&self.db, &mut self.connection, frame).await?;

        // To improve fairness:
        // tokio::task::yield_now().await;
    }
}
```

<figure class="align-center" style="width: min(1200px, calc(100vw - 32px)); max-width: none; margin-left: 50%; transform: translateX(-50%);">
  <a href="/blog/principles-for-fast-tokio-applications/mini-redis-latency-distribution.png" aria-label="Open the full-size mini-Redis latency distribution">
    <img src="/blog/principles-for-fast-tokio-applications/mini-redis-latency-distribution.png" alt="Two mini-Redis pipeline latency distributions showing that adaptive yielding reduces p50 latency from 0.967 to 0.105 milliseconds and p99 latency from 2.548 to 0.320 milliseconds" width="3000" height="1800" style="width: 100%; height: auto;" loading="lazy" decoding="async">
  </a>
  <figcaption><p>Yielding after four consecutive immediately-ready reads makes pipelined requests much fairer without giving up batching entirely.</p></figcaption>
</figure>

**How do I know if I have this problem?**

- P99 is much greater than P50.
- Polls take longer than the work inside them should require.
- Many spans fall inside a single poll.

#### Batch work to amortize overhead

Fairness is not free. The more useful work you can do per runtime event—changing tasks, polling, moving between workers, or changing threads—the more efficient your application can be.

Perhaps the best example is `tokio::fs`. I sometimes go so far as to say that "`tokio::fs` is considered harmful." Without [`io_uring`](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.enable_io_uring), Tokio runs each filesystem operation on the blocking pool. Each call to [`spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html) also has a cost, and every runtime has a shared blocking pool.

If you know you will perform a series of filesystem operations—or any blocking work—batch them into the largest sensible blocking segment. In some cases, a dedicated OS thread is a better fit.

This principle applies anywhere you interact with Tokio. If you know you will send work to the [global queue](https://docs.rs/tokio/latest/tokio/runtime/index.html#multi-threaded-runtime-behavior-at-the-time-of-writing), batching can amortize that coordination too.

**How do I know if I have this problem?**

- Tokio APIs such as `spawn_blocking` consume noticeable time in flamegraphs.
- A tight loop performs many individually small filesystem or blocking operations.
- Throughput improves when the same work is grouped into larger units.

### Beware global resources

The Tokio runtime schedules work on workers: dedicated threads that poll ready tasks. Workers scale across cores, but some runtime resources still require shared coordination.

The blocking pool is currently[^blocking-queue] a global resource. Under high enough rates, pushing work onto the blocking queue becomes a bottleneck and `spawn_blocking` can become visible in flamegraphs. I have seen negative performance effects at roughly 50,000 blocking tasks per second on a 32-core host; your mileage will vary. `spawn_blocking` is not a magic fix for every piece of blocking or CPU-heavy code. For short, bounded work, it may be faster to let Tokio's workers and work stealing handle it, but, as always "it depends."

Tokio also has a global task queue. Tasks land there when local worker queues overflow, which is usually rare, or when work is scheduled from outside a runtime worker, which can be common in some applications. One example is a channel whose sender runs on a non-Tokio thread.

**How do I know if I have this problem?**

- Runtime-wide operations such as `spawn_blocking` are prominent in flamegraphs.
- The global queue is consistently deep. In a healthy application it should generally stay close to empty; in a saturated application, it can take a long time to drain.

### Be extremely careful with blocking mutexes

One of the easiest ways to stall an entire runtime is to block a worker on a contended synchronous mutex and then hold that mutex for a long time.

A pattern I have seen in practice is a metrics registry stored behind a mutex or read-write lock. If a flush holds the lock while doing expensive work, every Tokio worker may eventually schedule a task that tries to record a metric and blocks on the same lock. The runtime can grind to a halt as its workers become blocked. In severe cases, no worker remains available to drive I/O.

Keep synchronous-mutex critical sections in async applications extremely short and bounded. Do not hold the lock while flushing, performing I/O, or awaiting another future. An async-aware mutex prevents the worker thread itself from blocking, but it does not make long or highly contended critical sections cheap.

**How do I know if I have this problem?**

- P99 spikes on a predictable interval—for example, once a minute when a background flush runs.
- In dial9, many tasks suddenly become blocked and off-CPU for a nontrivial duration.

<figure class="align-center" style="width: min(1200px, calc(100vw - 32px)); max-width: none; margin-left: 50%; transform: translateX(-50%);">
  <a href="/blog/principles-for-fast-tokio-applications/lock-contention.png" aria-label="Open the full-size lock-contention trace">
    <img src="/blog/principles-for-fast-tokio-applications/lock-contention.png" alt="dial9 trace showing all four Tokio workers blocked by mutex contention, followed by kernel scheduling delays and a sudden drop in active tasks" width="1600" height="1000" style="width: 100%; height: auto;" loading="lazy" decoding="async">
  </a>
  <figcaption><p>A contended blocking mutex stalls all four runtime workers at once.</p></figcaption>
</figure>

### Constrain parallelism—usually

Tokio can happily spawn far more tasks than the rest of your system can handle. Accidentally opening 3,000 concurrent connections to S3 because a workload fanned out an unbounded number of tasks is very common.

The answer is boring: limit concurrency. Fancy adaptive algorithms are sometimes appropriate, but a [`Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) is often enough.

### Isolate Tokio workers from other threads

Tokio's design relies on workers waking quickly. However, if the operating system is highly loaded, it may take 10–20 ms—or more—for the kernel to schedule a worker after Tokio attempts to wake it. If you measure P99 latency in single-digit milliseconds, this is a disaster. I've observed this during incremental migrations from Java to Rust at Amazon, where both processes ran on the same host and the Rust process gradually took on more of the work.

The less work the Java process did, the faster the Rust process became, even as it handled more work. This effect is even stronger when the other applications use a large number of threads.

The most basic solution is to use `cgroups` or related APIs to pin the Tokio workers and other code to separate CPU cores.

The same issue can arise from other Rust threads. Background threads such as those used by `tracing_appender` can sometimes do more than 100 ms of work without yielding the CPU. If Tokio attempts to wake a worker during this time, that worker may be delayed until the kernel preempts the other thread.

If you see this happening, the solution is the same: pin noncritical background work to its own core and move Tokio workers to other cores. **You rarely need every core for Tokio, and reserving cores for other work tends to improve latency.**

**How do I know I have this problem?**

- dial9 shows a kernel scheduling delay between a worker-unpark event and the worker actually running.

## Tricks for when you know better

The patterns in this section are not generally the right thing to do, but sometimes they are exactly what a workload needs.

### Blocking the executor can be fine—sometimes

In an idealized async application, all work would happen in tiny bursts with frequent yields back to Tokio. The real world does not always work that way, and tiny bursts are not necessarily the fastest way to run software. Batching work can be more efficient.

In practice, long polls are not always a problem. Under light load, Tokio's work stealing can compensate when one worker is occupied for longer than usual. That starts to break down under two conditions:

1. The Tokio runtime is heavily loaded and spare worker capacity does not exist.
2. The operating system is heavily loaded, so unparking workers is frequently delayed.

In both cases, stealing work takes longer. If work is not stolen quickly enough, core runtime maintenance—such as driving I/O—may not happen frequently enough to maintain low latency.

**Important note!** This advice does not apply if you are utilizing things like `tokio::join!` and `tokio::select!` that utilize in-task concurrency. Within a single task, there is no work stealing; if you block the executor, nothing else running _on that task_ can make progress. This sometimes manifests as unexpected timeouts and generally bad latency.

### Use multiple runtimes to isolate workloads by priority

The strongest isolation comes from assigning work to separate runtimes and pinning those runtimes to dedicated cores. Many network services have both latency-sensitive work and lower-priority background work. Putting them on separate runtimes creates a scheduling boundary between the two.

You can also set OS-level niceness when the runtime threads start. See dial9's [multiple-runtime example](https://github.com/dial9-rs/dial9/blob/main/dial9/examples/multi_runtime.rs) and Tokio's [`on_thread_start`](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.on_thread_start) hook.

### Spin to keep control

> This is a very advanced tactic for chasing latency measured in microseconds. I don't reccommend reaching for this first, but it can definitely work.

Every time you yield back to the Tokio scheduler—or Tokio parks a worker thread and yields it to the operating system—you create a chance for that work to be delayed when it wakes again.

For extremely latency-sensitive work, one option is to intentionally spin for a short preset period, maybe 50 microseconds, rather than yield while waiting for the next piece of useful work. This consumes a core and can harm neighboring workloads, so it is probably wrong for most applications. Under carefully controlled conditions, however, it can be the right tradeoff.

## Appendix: A mental model for Tokio in four bullet points

- Rust futures make incremental progress between await points. These active sections are called polls, after the [`Future::poll`](https://doc.rust-lang.org/stable/std/future/trait.Future.html#tymethod.poll) method.
- When futures are not being polled, they are idle and waiting for an executor to run them again. A good executor polls a future only when it has work to do.
- Tokio runs *N* workers, usually one per available core. Each worker has a local queue. When a queue overflows or work cannot be added to a local queue, the task goes to the [global queue](https://docs.rs/tokio/latest/tokio/runtime/index.html#multi-threaded-runtime-behavior-at-the-time-of-writing).
- When one worker's queue backs up, another worker can steal work from it—if the runtime detects the imbalance and another worker has capacity.

[^blocking-queue]: [Tokio 1.52.0](https://github.com/tokio-rs/tokio/releases/tag/tokio-1.52.0) briefly shipped a sharded blocking queue, but [1.52.1 reverted it](https://github.com/tokio-rs/tokio/releases/tag/tokio-1.52.1) after a regression that could cause `spawn_blocking` to hang. Tokio [PR #8337](https://github.com/tokio-rs/tokio/pull/8337) later re-landed the sharded queue as an unstable feature that is disabled by default.
