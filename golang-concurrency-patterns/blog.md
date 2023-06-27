---
title: Golang Concurrency Patterns
published: false
description: 
tags: 
cover_image:
canonical_url:
---

# Golang Concurrency Patterns

## 1. Publish/Subscribe Server

```golang
type PubSub interface{
    Publish(e Event)

    Subscribe(c chan<- Event)

    Cancel(c chan<- Event)
}
```

The publish method publishes event e to all current subscriptions.

Subscribe registers c to receive future events. All subscribers receive events in the same order, and that order respects program order: if Publish(e1) happens before Publish(e2), subscribers receive e1 before e2.

Cancel cancels the prior subscription of channel c. After any pending already-subscribed events have been sent on c, the server will signal that the subscription is cancelled by closing c.

Channels in Go are a fundamental feature for facilitating communication and synchronization between goroutines. They provide a way to send and receive values, allowing goroutines to coordinate their actions and share data.

`c chan<- Event` indicates that the channel c will only receive objects of type `Event` on it.

Following is an implementation using mutex:

```golang
type Server struct {
    mu sync.Mutex
    sub map[chan<- Event]bool
}

func (s *Server) Init() {
    s.sub = make(map[chan<- Event]bool)
}

func (s *Server) Publish(e Event) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for c := range s.sub {
        c <- e
    }   
}

func (s *Server) Subscribe(c chan<- Event) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.sub[c] {
        panic("pubsub: already subscribed")
    }
    s.sub[c] = true
}

func (s *Server) Cancel(c chan<- Event) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if !s.sub[c] {
        panic("pubsub: not subscribed")
    }
    close(c)
    delete(s.sub, c)
}

func helper(in <-chan Event, out chan<- Event) {
    var q []Event
    for {
        select {
            case e := <-in:
                q = append(q, e)
            case out <- q[0]:
                q = q[1:]
        }
    }
}
```
In Go, a mutex (short for mutual exclusion) is a synchronization primitive used to protect shared resources from concurrent access. It allows only one goroutine to access the protected resource at a time. The mutex provides two methods, Lock and Unlock, to acquire and release the lock, respectively. When a goroutine acquires the lock, other goroutines attempting to acquire it will block until it is released, ensuring exclusive access to the resource. Mutexes are a simple and effective way to prevent race conditions in concurrent Go programs.

The `defer` statement in Go is used to schedule a function call to be executed when the surrounding function completes. The deferred function calls are executed in a last-in-first-out order, ensuring that resources are properly cleaned up. In tha above code, `defer` is used for unlocking mutexes.

The `select` statement is used to handle multiple channel operations concurrently. It allows us to wait for communication on multiple channels and perform different actions based on the first channel that is ready to communicate. This enables us to handle non-blocking operations and select the case that can proceed, providing a powerful mechanism for concurrent programming.

```golang
type subReq struct{
    c chan<- Event
    ok chan bool
}

type Server struct {
    publish chan Event
    subscribe chan subReq
    cancel chan subReq
}

func (s *Server) Publish(e Event){
    s.publish <- e
}

func (s *Server) Subscribe(c chan<- Event){
    r := subReq{c: c, ok: make(chan bool)}
    s.subscribe <- r
    if !<-r.ok {
        panic("pubsub: already subscribed")
    }
}

func (s *Server) Cancel(c chan<- Event) {
    r := subReq{c: c, ok: make(chan bool)}
    s.cancel <- r
    if !<-r.ok {
        panic("pubsub: not subscribed")
    }
}


func (s *Server) Init() {
    s.publish = make(chan Event)
    s.subscribe = make(chan subReq)
    s.cancel = make(chan subReq)
    go s.loop()
}

func (s *Server) loop() {
    sub := make(map[chan<- Event]bool)
    for {
        select {
            case e := <-s.publish:
                for c := range sub {
                    c <- e
                }
            case r := <-s.subscribe:
                if sub[r.c] {
                    r.ok <- false
                    break
                }
                sub[r.c] = true
                r.ok <- true
            case c := <-s.cancel:
                if !sub[c.c] {
                    c.ok <- false
                    break
                }
                close(c.c)
                delete(sub, c.c)
                c.ok <- true
            }
    }
}
```
Goroutines in Go are lightweight, independently executing functions that run concurrently with other goroutines. They provide a simple and efficient way to achieve concurrency in Go programs. Goroutines are multiplexed onto a smaller number of operating system threads, allowing many goroutines to run concurrently and efficiently utilize available CPU resources.

In the above code, the mutexes are converted into goroutines as is makes the program clearer. The loop function runs on a different thread, and it contains the logic to process the events coming on different channels.

We can modify the helper function to process the events for different subscribers independently as follows:
```golang
func helper(in <-chan Event, out chan<- Event) {
    var q []Event
    // wait till input channel is closed and all the events inside q are processed
    for in != nil && len(q) > 0 {
        // Decide whether and what to send.
        var sendOut chan<- Event
        var next Event
        if len(q) > 0 {
            sendOut = out
            next = q[0]
        }
        select {
            case e, ok := <-in:
                if !ok {
                    in = nil // stop receiving from in
                    break
                }
                q = append(q, e)
            case sendOut <- next:
                q = q[1:]
        }
    }
    close(out)
}
```

The main loop can be modified as follows to let independent concerns run independently.
```golang
func (s *Server) loop() {
    // map from output channel to input channel
    sub := make(map[chan<- Event]chan<- Event)
    for {
        select {
            case e := <-s.publish:
                // publish new event to all the input channels of the subscribers
                for _, h := range sub {
                    h <- e
                }
            case r := <-s.subscribe:
                if sub[r.c] != nil {
                    r.ok <- false
                    break
                }
                // add new subscriber to the publisher by creating a new input channel
                // and starting a new goroutine for the subscriber to process events
                h = make(chan Event)
                go helper(h, r.c)
                sub[r.c] = h
                r.ok <- true
            case c := <-s.cancel:
                if sub[c.c] == nil {
                    r.ok <- false
                    break
                }
                // close the input channel of the subscriber
                    close(sub[r.c])
                delete(sub, r.c)
                r.ok <- true
        }
    }
}
```

## Work Scheduler

Given a list of servers, and number of tasks, schedule them.

```golang
func Schedule(servers []string, numTask int, call func(srv string, task int))
```

We will use buffered channel of length equal to number of servers as a concurrent blocking queue to store the idle server ids. Also, we will use goroutines to let the tasks run independently.

```golang
func Schedule(servers []string, numTask int, call func(srv string, task int)) {
    idle := make(chan string, len(servers))
    for _, srv := range servers {
        idle <- srv
    }
    for task := 0; task < numTask; task++ {
        go func() {
            srv := <-idle
            call(srv, task)
            idle <- srv
        }()
    }   
}
```

The above code has a data race problem. In Go, when we use a loop variable inside a closure, the closure captures the variable itself and not its value. This can lead to unexpected behavior because the variable may have a different value by the time the closure executes. We modify the code as follows to create a copy of task that won't be affected by the subsequent iterations of the loop, ensuring that the correct value is used when the closure executes.

```golang
for task := 0; task < numTask; task++ {
    task := task
    go func(task2 int) {
        srv := <-idle
        call(srv, task)
        idle <- srv
    }()
}
```

In the above code, we start a lot of goroutines(if there are a large number of tasks) at once, even though it may take a long time for the servers to start processing those tasks. So we take the code `srv := <- idle` out of the goroutine. This will ensure that the goroutine is started only when there is an idle server available to process that task.

```golang
for task := 0; task < numTask; task++ {
    task := task
    srv := <-idle
    go func(task2 int) {
        call(srv, task)
        idle <- srv
    }()
}
```

In the following code, we run all the servers as different goroutines waiting for new tasks coming on the work channel, and processing them in parallel.

```golang
func Schedule(servers chan string, numTask int, call func(srv string, task int)) {
    work := make(chan int)
    done := make(chan bool)
    runTasks := func(srv string) {
        for task := range work {
            call(srv, task)
            done <- true
        }
    }
    go func() {
        // new servers come on channels, so we do not need to know the number of servers available
        for _, srv := range servers {
            go runTasks(srv)
        }
    }()
    
    for task := 0; task < numTask; task++ {
        work <- task
    }
    // close work channel to signal to the servers that no more tasks will be sent
    close(work)
    
    for i := 0; i < numTask; i++ {
        <-done
    }
}
```

There is an issue with the above code...it can lead to a DEADLOCK!

Consider the case when number of servers are lesser that the number of tasks(which is the usual case in real life). The servers goroutines try to communicate with the main thread that they have completed processing the task via the `done` channel. However, the main thread is waiting for the server goroutines to accept new tasks via the `work` channel, and is not available to process the done requests coming on the `done` channel.

One way to fix this is to create a separate goroutine for sending tasks to the servers.

```golang
go func() {
    for task := 0; task < numTask; task++{
        work <- task
    }
    close(work)
}()
```

In this way, the main thread does not wait till all tasks have sent to servers before starting to process the done requests.

The easiest way to avoid the deadlock is to make the work channel big enough so that sending the tasks on the `work` channel is never blocked which will prevent the deadlock.

```golang
//make the work channel a buffered channel of length numTask
work := make(chan int)   =>  work := make(chan int, numTask)
```

If some of the tasks timeout while processing, we can add them back to the work channel. We need to move the `close(work)` at the end to avoid premature closing of the work channel before all the tasks have been processed successfully.

```golang
func Schedule(servers chan string, numTask int, call func(srv string, task int) bool) {
    work := make(chan int, numTask)
    done := make(chan bool)
    runTasks := func(srv string) {
        for task := range work {
            if call(srv, task) {
                done <- true
            } else {
                work <- task
            }
        }
    }

    go func() {
        for _, srv := range servers {
            go runTasks(srv)
        }
    }()

    for task := 0; task < numTask; task++ {
        work <- task
    }

    for i := 0; i < numTask; i++ {
        <-done
    }
    close(work)
}
```

## Replicated Service Client

```golang
type Client interface {
    Init(servers []string, callOne func(string, Args) Reply)

    Call(args Args) Reply
}
```

The `Init` function initializes the client to use the given servers. To make a particular request later, the client can use `callOne(srv, args)`, where `srv` is one of the available servers from the provided list. The `Call` function make a request to an available server. Multiple goroutines can call `Call` concurrently.

```golang
type Client struct {
    servers []string
    callOne func(string, Args) Reply
    mu sync.Mutex
    prefer int
}
func (c *Client) Init(servers []string, callOne func(string, Args) Reply) {
    c.servers = servers
    c.callOne = callOne
}

func (c *Client) Call(args Args) Reply {
    type result struct {
        serverID int
        reply Reply
    }

    const timeout = 1 * time.Second
    t := time.NewTimer(timeout)
    defer t.Stop()  // stop timer when not needed

    done := make(chan result, len(c.servers))
    for id := 0; id < len(c.servers); id++ {
        id := id
        go func() {
            done <- result{id, c.callOne(c.servers[id], args)}
        }()
        
        select {
            case r := <-done:
                return r.reply
            case <-t.C:
                // timeout
                t.Reset(timeout)
        }
    }
    
    r := <-done
    return r.reply
}
```

In the above code, when the `Call` function is called, the client iterates over all the available servers and calls the provided function with that server id in a separate goroutine. If is does not complete its execution before timeout, the next server is tried.

In real life, once the client finds a server that is working, it may want to prefer that server for future calls of the `Call`. This can be done by storing the id of the preferred server in the client. We can modify the `Call` function a follows:

```golang

func (c *Client) Call(args Args) Reply {
    type result struct {
        serverID int
        reply Reply
    }

    const timeout = 1 * time.Second
    t := time.NewTimer(timeout)
    defer t.Stop()  // stop timer when not needed

    done := make(chan result, len(c.servers))
    
    //to avoid data races across concurrent calls of Call
    c.mu.Lock()
    prefer := c.prefer
    c.mu.Unlock()

    var r result
    for off := 0; off < len(c.servers); off++ {
        id := (prefer + off) % len(c.servers)
        go func() {
            done <- result{id, c.callOne(c.servers[id], args)}
        }()
        select {
            case r = <-done:
                goto Done
            case <-t.C:
                // timeout
                t.Reset(timeout)
        }   
    }

    r = <-done
Done:
    c.mu.Lock()
    c.prefer = r.serverID
    c.mu.Unlock()

    return r.reply
} 
```

## Protocol Multiplexer

```golang
type ProtocolMux interface {
    // Init initializes the mux to manage messages to the given service.
    Init(Service)
    // Call makes a request with the given message and returns the reply.
    // Multiple goroutines may call Call concurrently.
    Call(Msg) Msg
}

type Service interface {
    // ReadTag returns the muxing identifier in the request or reply message.
    // Multiple goroutines may call ReadTag concurrently.
    ReadTag(Msg) int64
    // Send sends a request message to the remote service.
    // Send must not be called concurrently with itself.
    Send(Msg)
    // Recv waits for and returns a reply message from the remote service.
    // Recv must not be called concurrently with itself.
    Recv() Msg
}
```

```golang
type Mux struct {
    srv Service
    send chan Msg
    mu sync.Mutex
    pending map[int64]chan<- Msg
}

func (m *Mux) Init(srv Service) {
    m.srv = srv
    m.pending = make(map[int64]chan Msg)
    go m.sendLoop()
    go m.recvLoop()
}

func (m *Mux) sendLoop() {
    for args := range m.send {
        m.srv.Send(args)
    }
}

func (m *Mux) recvLoop() {
    for {
        reply := m.srv.Recv()
        tag := m.srv.Tag(reply)

        m.mu.Lock()
        done := m.pending[tag]
        delete(m.pending, tag)
        m.mu.Unlock()

        if done == nil {
            panic("unexpected reply")
        }
        done <- reply
    }
}

func (m *Mux) Call(args Msg) (reply Msg) {
    tag := m.srv.ReadTag(args)
    done := make(chan Msg, 1)

    m.mu.Lock()
    if m.pending[tag] != nil {
        m.mu.Unlock()
        panic("mux: duplicate call tag")
    }
    m.pending[tag] = done
    m.mu.Unlock()
    
    m.send <- args
    return <-done
}
```

## References

The code is taken from Russ Cox's [slides](https://swtch.com/mit-6824-go-2021.pdf).
