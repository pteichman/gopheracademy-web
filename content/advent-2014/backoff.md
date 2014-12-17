+++
author = ["Peter Teichman"]
date = "2014-12-17T19:30:00+00:00"
title = "Simple backoff"
series = ["Advent 2014"]
+++

There comes a time in the life of many programs when you need to
maintain a persistent connection to a server. When that server goes
down (as it will), you'll need to reconnect.

Reconnecting usually means two things: you should slowly increase the
time you're waiting between attempts, and you should slightly
randomize those times in order to avoid the
[thundering herd problem](http://en.wikipedia.org/wiki/Thundering_herd_problem).

There are several libraries for Go that implement backoff with delay,
and their APIs are similar to this:

```
type Backoff struct {
     attempt int
     ...
}

func New() *Backoff {
     ...
}

func (b *Backoff) Try(f func() error) {
     // Perform some delay based on the current b.attempt, likely using time.Sleep().

     if err := f(); err == nil {
         // Reset the backoff counter on success.
         b.Reset()
     } else {
         b.attempt++
     }
}

func (b *Backoff) Reset() {
     attempt = 0
}
```

I've found this approach to be limiting for a few reasons:

It turns control of my function over to the backoff package, making it
more difficult to reason about program flow.

It doesn't allow you to reset a backoff process mid-try if you get a
signal that you can retry from zero.

It doesn't allow you to easily change behavior between fatal and
retryable errors in `f()`.

Any outside state must be updated in the body of `f()`, another blow
to reasoning about program flow.

I've used the below as an alternative in a couple of projects now:

```
package backoff

import "time"

// BackoffPolicy implements backoff policy,  randomizing its delays and
// saturating at its last value.
type BackoffPolicy struct {
	Millis []int
}

// defaultBackoff is a backoff policy ranging up to 5s.
var Default = BackoffPolicy{
	[]int{0, 10, 10, 100, 100, 500, 500, 3000, 3000, 5000},
}

// duration returns the time duration of the n'th wait cycle in its
// backoff policy. This is backoff.millis[n], randomized to avoid
// thundering herds.
func (b BackoffPolicy) Duration(n int) time.Duration {
	if n >= len(b.Millis) {
		n = len(b.Millis) - 1
	}

	return time.Duration(jitter(b.Millis[n])) * time.Millisecond
}

// jitter returns a random integer uniformly distributed in the range
// [0.5 * millis .. 1.5 * millis]
func jitter(millis int) int {
	if millis == 0 {
		return 0
	}

	return millis/2 + rand.Intn(millis)
}
```

With typical use looking like this:

```
func retryConn() {
     var attempt int

     for {
          time.Sleep(backoff.Default.Duration(attempt))

          conn, err := makeConnection()
          if err != nil {
              // Maybe log the error.
              attempt++
              continue
          }

          // Do your thing with conn; log any eventual errors.

          attempt = 0
     }     
}
```

You can find real life usage in [this pumpWrites
function](https://github.com/spacemonkeygo/monitor/blob/c18860ccb55edc52e761551989e78a605ff58bb2/trace/scribe.go#L66),
which maintains a persistent connection to a Scribe server for writing
call trace data.

This has several advantages:

The calling function has complete control over program flow. It can
use time.Sleep() to backoff for simplicity, or it can use time.After()
and select if other signals are in play.

So if you have a "done" channel to signal that retries should stop:

```
func retryConn(done <-chan struct{}) {
     var attempt int

     for {
         // Replace the time.Sleep above with this block.
         select {
         case <-done:
              return
         case <-time.After(backoff.Default.Duration(attempt)):
         }

         [ ... ]
     }
}

```

It allows you to reset the backoff counter in the middle of a retry if
you have an out-of-band reset signal (e.g. ZooKeeper notifying you the
server is back up):

```
func retryConn(reset, done <-chan struct{}) {
     var attempt int

loop:
     for {
         // Replace the time.Sleep above with this block.
         select {
         case <-done:
              return
         case <-reset:
              attempt = 0
              continue loop         
         case <-time.After(backoff.Default.Duration(attempt)):
         }

         [ ... ]
     }
}
```

The backoff policy itself is data only and holds no backoff state,
allowing safe concurrent use with no need for allocations.

All state, including the connection itself and the attempt counter, is
held entirely in the retryConn function. You can reason about the
state of the connection without looking anywhere else. The attempt
counter is simply an integer, not a property hidden behind an opaque
struct.

If a fatal error occurs when dealing with `conn`, you can easily
return that from retryConn without disrupting another process.

Most important, and this is a thing I love about programming in Go, is
that the function *looks like what it does*. There's no hidden state
or magic behavior.

I've always found this array-based backoff sufficient, but if you want
to calculate the delays, you can elevate `Duration()` into an
interface and implement that however you'd like:

```
type Backoff interface {
     Duration(n int) time.Duration
}

type RandomBackoff struct {}

func (b RandomBackoff) Duration(n int) time.Duration {
     // This is a terrible idea.
     return time.Duration(rand.Int63())
}
```

And always remember to seed your random number generator!
