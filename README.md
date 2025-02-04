# broadcast

![CI](https://github.com/teivah/broadcast/actions/workflows/ci.yml/badge.svg)
[![Go Report Card](https://goreportcard.com/badge/github.com/teivah/broadcast)](https://goreportcard.com/report/github.com/teivah/broadcast)

Notifications broadcaster in Go

## What?

`broadcast` is a library that allows sending repeated notifications to multiple goroutines with guaranteed delivery.

## Why?

### Why not Channels?

The standard way to handle notifications is via a `chan struct{}`. However, sending a message to a channel is received by a single goroutine. 

The only operation that is broadcasted to multiple goroutines is a channel closure. Yet, if the channel is closed, there's no way to send a message again.

❌ Repeated notifications to multiple goroutines

✅ Guaranteed delivery

### Why not sync.Cond?

`sync.Cond` is the standard solution based on condition variables to set up containers of goroutines waiting for a specific condition.

There's one caveat to keep in mind, though: the `Broadcast()` method doesn't guarantee that a goroutine will receive the notification. Indeed, the notification will be lost if the listener goroutine isn't waiting on the `Wait()` method.

✅ Repeated notifications to multiple goroutines

❌ Guaranteed delivery

## How?

### Step by Step

First, we need to create a `Relay`:

```go
relay := broadcast.NewRelay()
```

Once a `Relay` is created, we can create a new listener using the `Listen` method. As the `broadcast` library relies internally on channels, it accepts a capacity:

````go
list := relay.Listener(1) // Create a new listener based on a channel with a one capacity
````

A `Relay` can send a notification in three different manners:
* `Notify()`: block until a notification is sent to all the listeners
* `NotifyCtx(context.Context)`: send a notification to all the listeners unless the provided context times out or is canceled
* `Broadcast()`: send a notification to all the listeners in a non-blocking manner; guaranteed delivery isn't guaranteed

On the `Listener` side, we can access the internal channel using `Ch`:

```go
<-list.Ch() // Wait on a notification
```

We can close a `Listener` and a `Relay` using `Close`:

```go
list.Close() 
relay.Close()
```

Closing a `Relay` and `Listener`s can be done concurrently in a safe manner.

### Example

```go
relay := broadcast.NewRelay() // Create a relay
defer relay.Close()

list1 := relay.Listener(1) // Create a listener with one capacity
defer list1.Close()
list2 := relay.Listener(1) // Create a listener with one capacity
defer list2.Close()

// Listener goroutines
f := func(i int, list *broadcast.Listener) {
    for range list.Ch() { // Waits for receiving notifications
        fmt.Printf("listener %d has received a notification\n", i)
    }
}
go f(1, list1)
go f(2, list2)

// Notifier goroutine
for i := 0; i < 5; i++ {
time.Sleep(time.Second)
    relay.Notify() // Send notifications with guaranteed delivery
    ctx, _ := context.WithTimeout(context.Background(), time.Second)
    relay.NotifyCtx(ctx) // Send notifications but cancel after 1 second
    relay.Broadcast()    // Send notifications without guaranteed delivery
}
```
