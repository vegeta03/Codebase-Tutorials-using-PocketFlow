# Chapter 6: Duration

Building on [Chapter 5: Options](05_options.md), we now turn to **Duration**—the abstraction that centralizes all timing logic in RxGo. Whether you’re driving `Interval`, `Timer`, `Debounce` or `Throttle`, `Duration` ensures consistent unit handling, validation, and seamless integration with custom schedulers or test harnesses.

---

## 1. Motivation & Central Use Case

Modern reactive pipelines often need fine-grained control over time:

- Emitting events at a steady beat (e.g. sampling sensors every second)  
- Deferring actions until a pause in upstream events (e.g. auto-save when user stops typing)  
- Limiting throughput (e.g. rate-limiting API calls to one per 500 ms)  

Without a central timing primitive, every operator would reinvent `time.Duration` parsing, default units, context cancellation, and testing support. Enter **Duration**—think of it as a **metronome** in music, dictating the beat of your stream.

### Central Use Case: Auto-Save Form

Imagine a text-editor that should auto-save only when the user pauses typing for at least 500 ms, and never more often than once every 5 s (to avoid excessive I/O).

1. **Debounce** keystroke events by 500 ms → wait for “silence.”  
2. **Throttle** the resulting save trigger to at most one per 5 s.  
3. **Trigger** the save action downstream.

We’ll show how `Duration` drives both `Debounce` and `Throttle` to solve this scenario cleanly.

---

## 2. Key Concepts

1. **Duration interface**  

   ```go
   type Duration interface {
     duration() time.Duration
   }
   ```

   A simple wrapper over `time.Duration` that operators call via `d.duration()`.

2. **WithDuration**  

   ```go
   func WithDuration(d time.Duration) Duration
   ```

   Factory that creates a `Duration`. This is your standard metronome tick.

3. **Infinite**  

   ```go
   // Infinite represents an infinite wait time
   var Infinite int64 = -1
   ```

   Some operators use `Infinite` to disable timeouts or back-off loops.

4. **CausalityDuration & timeCausality** (testing)  
   A special `Duration` implementation that drives a fake clock.  

   ```go
   // timeCausality lets you inject a sequence of events + ticks.
   func timeCausality(elems ...interface{}) (context.Context, Observable, Duration)
   ```

   - `elems` is a mix of data values, `tick` markers, or `error` values.  
   - Each call to `duration()` runs the next element’s function (pushing to a channel) and returns a tiny or large wait.  
   - Great for deterministic unit tests without real delays.

5. **Integration**  
   All time-based operators (`Interval`, `Timer`, `Debounce`, `Throttle`) accept a `Duration`, call `duration()` internally, and wire into your pipeline using RxGo’s usual `Option` and `Observable` plumbing.

---

## 3. Using Duration in RxGo

Below are the most common patterns for time-based operators. We’ll build the **auto-save** example step by step.

### 3.1. Debounce: Wait for Silence

```go
package main

import (
  "fmt"
  "time"
  "github.com/reactivex/rxgo/v2"
)

func main() {
  // Simulate keystroke events on a channel
  keystrokes := make(chan rxgo.Item)
  go func() {
    for _, k := range []string{"h", "e", "l", "l", "o"} {
      keystrokes <- rxgo.Of(k)
      time.Sleep(100 * time.Millisecond)  // rapid typing
    }
    close(keystrokes)
  }()

  // Debounce: wait 500ms of silence
  autoSaveTrig := rxgo.FromEventSource(keystrokes).
    Debounce(rxgo.WithDuration(500 * time.Millisecond))

  // Subscribe and print when save should happen
  for item := range autoSaveTrig.Observe() {
    fmt.Println("Auto-save at:", item.V)
  }
}
```

Explanation

1. We stream raw keystrokes via `FromEventSource`.  
2. `Debounce(WithDuration(500ms))` resets its internal timer on each keystroke.  
3. Only when 500 ms passes with no new input does `Debounce` emit the **last** keystroke—triggering an auto-save.

### 3.2. Throttle: Limit Rate

```go
saved := autoSaveTrig.
  Throttle(rxgo.WithDuration(5 * time.Second)).
  Map(func(_ context.Context, _ interface{}) (interface{}, error) {
    performSave()           // your I/O logic
    return "saved", nil
  })

// Fire and forget: we only care about side-effects
<-saved.Run()
fmt.Println("Auto-save pipeline completed")
```

Explanation

- `Throttle(5s)` ensures that even if `Debounce` fires more often, we only run `performSave()` at most once every 5 s.
- Downstream `Map` performs the actual save; `Run()` kicks off the sequence and waits for completion.

### 3.3. Interval & Timer

For periodic or one-off scheduling:

```go
// Emit a heartbeat every second (hot source)
heartbeats := rxgo.Interval(rxgo.WithDuration(1 * time.Second))

// Emit a single timeout after 10 seconds
timeout := rxgo.Timer(rxgo.WithDuration(10 * time.Second))

// Merge them
merged := rxgo.MergeObservable([]rxgo.Observable{heartbeats, timeout})
for item := range merged.Observe() {
  fmt.Println("Event:", item.V)
}
```

- `Interval` pushes `0,1,2,…` on every tick.
- `Timer` emits just one value after the specified duration.

---

## 4. Under the Hood

### 4.1. Step-by-Step Sequence

Below is what happens when you call, for example, `rxgo.Interval(d)` and subscribe:

```mermaid
sequenceDiagram
  participant Dev      as Developer
  participant Factory  as Interval()
  participant Opts     as parseOptions
  participant Chron    as Duration
  participant GoTimer  as time.After
  participant OutCh    as chan Item
  participant Subscriber

  Dev->>Factory: Interval(WithDuration(1s))
  Factory->>Opts: parseOptions(defaultOpts)
  Opts-->>Factory: funcOption
  Factory->>Chron: holds Duration
  Factory-->>Dev: returns ObservableImpl

  Dev->>ObservableImpl: Observe()
  ObservableImpl->>Opts: merge options
  Opts-->>ObservableImpl: channel & ctx
  ObservableImpl->>OutCh: allocate channel
  ObservableImpl->>GoTimer: go func() { for { <-time.After(Chron.duration()); send } }
  GoTimer->>OutCh: emit 0,1,2…
  Subscriber->>OutCh: range-loop items
```

### 4.2. Core Implementation (duration.go)

```go
package rxgo

import "time"

// Duration is the timing abstraction used by interval/timer/throttle/debounce.
type Duration interface {
  duration() time.Duration
}

// duration wraps a Go time.Duration
type duration struct {
  d time.Duration
}

func (d *duration) duration() time.Duration {
  return d.d
}

// WithDuration constructs a Duration from a time.Duration.
func WithDuration(d time.Duration) Duration {
  return &duration{d: d}
}

// Infinite can be used by operators that support an unbounded wait.
var Infinite int64 = -1
```

Operators will simply call:

```go
wait := myDuration.duration()
// e.g. <-time.After(wait)
```

### 4.3. CausalityDuration for Testing

In `duration.go` you’ll also find a testing helper:

```go
// A sentinel for “tick” markers.
var tick = struct{}{}

// execution holds a callback and whether it's a tick event.
type execution struct {
  f      func()
  isTick bool
}

// causalityDuration implements Duration with a scripted sequence.
type causalityDuration struct {
  fs []execution
}

// timeCausality builds a controllable Observable+Duration pair.
func timeCausality(elems ...interface{}) (context.Context, Observable, Duration) {
  ch := make(chan Item, 1)
  fs := make([]execution, len(elems)+1)
  ctx, cancel := context.WithCancel(context.Background())

  for i, elem := range elems {
    if elem == tick {
      fs[i] = execution{f: func() {}, isTick: true}
    } else {
      // on value or error, push to channel
      ev := elem
      fs[i] = execution{
        f: func() { ch <- Of(ev) },
        isTick: false,
      }
    }
  }
  // final execution cancels the context
  fs[len(elems)] = execution{
    f:      func() { cancel() },
    isTick: false,
  }

  return ctx, FromChannel(ch), &causalityDuration{fs: fs}
}

// duration returns the next wait time and executes its callback.
func (d *causalityDuration) duration() time.Duration {
  pop := d.fs[0]
  pop.f()
  d.fs = d.fs[1:]
  if pop.isTick {
    return time.Nanosecond   // effectively no-delay tick
  }
  return time.Minute        // simulate long waiting
}
```

**How to use in tests**:

```go
ctx, source, dur := timeCausality("A", tick, "B", "C", tick)
debounced := source.Debounce(dur).Observe(rxgo.WithContext(ctx))
for item := range debounced {
  fmt.Println("Debounced:", item.V)
}
```

Each call to `dur.duration()` drives either a data push or a tiny “tick,” so you control exactly when events fire without real sleeps.

---

## 5. Real-World Analogy

Think of **Duration** as the conductor’s baton in an orchestra:

- **WithDuration** sets the tempo (e.g. 120 bpm → 500 ms between beats).  
- **Interval** is like the metronome clicking on every beat.  
- **Debounce** waits for a rest before cueing the next section.  
- **Throttle** ensures no musician plays more than one note per bar.  
- **timeCausality** lets you **rehearse** in slow-motion or jump bars, controlling exactly when the music plays—perfect for testing.

---

## 6. Summary & Next Steps

In this chapter we learned:

- The **Duration** interface and `WithDuration` factory.  
- How `Interval`, `Timer`, `Debounce`, and `Throttle` consume `Duration`.  
- The **metronome** analogy for timing.  
- The `timeCausality` helper for deterministic, sleep-free testing.

Armed with `Duration`, you can precisely orchestrate time-based operators in your reactive pipelines. Next up, we’ll explore how RxGo represents individual data points with [Chapter 7: Item](07_item.md). Enjoy!

---

Generated by fork of [AI Codebase Knowledge Builder](https://github.com/vegeta03/PocketFlow-Tutorial-Codebase-Knowledge.git)
