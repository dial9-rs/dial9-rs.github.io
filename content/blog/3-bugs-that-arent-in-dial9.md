+++
title = "3 bugs that aren't in dial9"
date = 2026-05-30
[taxonomies]
tags = ["bugs", "concurrency", "ai"]
+++

[dial9](https://github.com/dial9-rs/dial9) is a microscope for Tokio (and Rust applications in general): its compact binary format can record a log of runtime events so you can reconstruct what actually happened to understand bugs and performance behavior. It runs in production, where bugs have real blast radius.

As much as we attempt to avoid it, dial9 still has bugs. We catch most in CI, some in PR review, and some are discovered by customers.

We use AI to help build dial9, bug this presents a challenge: today's models are jagged: strong in one domain, weak in another. Folks are probably familiar with "*How many r's are in Strawberry*" or "*I want to wash my car, the car wash is 50m away. Should I walk or drive?*" These are real but don't really capture what this looks like in an Here are three examples we hit building dial9.

<!-- more -->

## FxHash

dial9 uses a fast but cryptographically insecure algorithm called FxHash for certain encoding operations. I told the agent to inline the code from the much larger [FxHash crate](https://crates.io/crates/fxhash) because we only needed ~10 lines:

```rust
impl Hasher for FxHasher {
    #[inline]
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 = (self.0.rotate_left(5) ^ b as u64)
                .wrapping_mul(0x517cc1b727220a95);
        }
    }
}
```

This code has a bug, but unless you really know what's going on, you won't spot it by looking at it.

The agent failed to apply a critical optimization to hash bytes 8-at-a-time in 64-bit words.

```rust
    fn write(&mut self, mut bytes: &[u8]) {
        // critical optimization: hash 8 bytes at a time
        while bytes.len() >= 8 {
            self.hash_word(u64::from_ne_bytes(bytes[..8].try_into().unwrap()));
            bytes = &bytes[8..];
        }
        if bytes.len() >= 4 {
            self.hash_word(u32::from_ne_bytes(bytes[..4].try_into().unwrap()) as u64);
            bytes = &bytes[4..];
        }
        for &b in bytes {
            self.hash_word(b as u64);
        }
    }
```

If a human inlined `FxHash`, there is a 0% chance they would create this bug: they would literally copy and paste the code from FxHash.

This bug actually stuck around our code base for quite a while until someone realized it could be optimized doing some unrelated work that touched the `Hasher`.

## Concurrency, part 1

Early versions of dial9 didn't coordinate how events were flushed across multiple threads. This didn't drop events, but it meant that one wall clock interval of data could potentially be split across multiple files.

```rust
    if buf.should_flush() || buf.flush_epoch.load() < current_epoch {
        buf.flush_epoch.store(current_epoch);
        collector.accept_flush(buf.flush());
    }
```

I added a new mechanism to trigger simultaneous flushing of the buffers via an epoch counter. Everything seemed to work fine, but a test case that validated that buffers were properly flushed failed extremely reliably on GitHub's macOS runner.

With other flaky tests, I had become accustomed to AI's ability to debug these sorts of race conditions by just staring at it so I did what I normally did and just asked AI to iterate w/ CI until it was green and fix it.

However, to my surprise, the agent couldn't fix it. It kept trying random changes but it couldn't seem to actually find the bug.

Eventually I started looking myself, I looked for probably under 60 seconds and spotted the bug:

```rust
    if buf.should_flush() || buf.flush_epoch.load() < current_epoch {
        buf.flush_epoch.store(current_epoch); // this line must be last!
        collector.accept_flush(buf.flush());
    }
```

We were incrementing the atomic that claimed we flushed before actually flushing.

**You cannot assume that because AI tooling is better than you, even at one specific class of problem, that it won't fail on a simple problem.** I've seen a lot of talented engineers basically give up if Claude couldn't solve the problem. That is a dangerous mindset. Next generation models like Claude Mythos show no signs of reduction in jaggedness.[^1]

We only caught this because for some reason, the GitHub macOS runner hit this race every time. Cross platform CI is a cheap way to cover more of these issues especially when coupled with tools like [Shuttle](https://github.com/awslabs/shuttle), [Loom](https://github.com/tokio-rs/loom), [Turmoil](https://github.com/tokio-rs/turmoil), and fuzzing. Defense in depth is required.

## Concurrency, part 2

A basic prompt is all it takes to find MANY bugs in code written by agents (and humans). Some folks have a mistaken notion that one model is not going to be able to find its own bugs, but this couldn't be further from the truth.

dial9 also includes its own stack unwinder and ring buffer to work in environments like Fargate where kernel unwinding is not available.

The code to support this is quite complex: it needs to be a ring buffer that can be safely used from a signal handler. The original version had a bug where two atomics interleaved in a way that could leave tombstones that led to dropped data.

```rust
pub(crate) unsafe fn claim_slot() -> Option<SlotWriter> {
    let idx = BUFFER.write_idx.fetch_add(1, Ordering::Relaxed);
    let slot_idx = idx % BUFFER_CAP;
    let slot = &BUFFER.slots[slot_idx];

    // If the slot isn't empty, the buffer wrapped around and the flush
    // thread hasn't caught up. Drop this sample.
    if slot
        .state
        .compare_exchange(
            SLOT_EMPTY,
            SLOT_WRITING,
            Ordering::Acquire,
            Ordering::Relaxed,
        )
        .is_err()
    {
        BUFFER.dropped.fetch_add(1, Ordering::Relaxed);
        return None;
    }

    Some(SlotWriter {
        slot: slot as *const SampleSlot as *mut SampleSlot,
        committed: false,
    })
}
```

Unlike the previous race condition that was immediately obvious to me, this one was extremely subtle. It's the sort of issue that is probably only discoverable if you are very familiar with these algorithms and data structures. I would not have caught this bug.

When I sat down to review the PR with this bug in it, I asked Claude to find the trickiest functions then to spawn a subagent to review each function in detail. Not only did it spot the bug, but it was able to create a failing test case I could give to the author. The [fix](https://github.com/dial9-rs/dial9/pull/250/commits/7eee3f086d8e4f0ed1b65139f190370665877650) replaced the slot state machine with a Vyukov-style per-slot sequence number.

Even in the specific domain of "concurrent code leveraging atomics" AI's abilities remain extremely jagged. Today's models are both superhuman and subhuman in their abilities to spot tricky bugs in the wild.

## Defense in depth

Catching bugs in code, whether or not it's written by agents, needs defense in depth: an instinct about when a hash function is slower than it should be, getting lucky on a CI platform that hits a race, and using agents to review complex code. The answer isn't to trust agents less; it's to build in layers. For dial9 that's agentic code review, cross-platform CI, and shuttle/turmoil for concurrency. No single layer is sufficient and we are continuously adding more.

**The same model with the same context can write the buggy code, spot the bug, and patch it.**

[^1]: Claude Mythos seems to share this spikiness; the [System Card](https://www-cdn.anthropic.com/08ab9158070959f88f296514c21b7facce6f52bc.pdf) shows that while it has superhuman pentesting abilities, many of the same issues folks have with Opus still plague Mythos: "The model is asked to write a tutorial mapping GPU optimizations onto a different accelerator. It produces a 67KB HTML document with interactive figures. Across the session the user catches four independent factual errors in the authored content; the user explicitly requests fact-checking twice and still finds errors after."
