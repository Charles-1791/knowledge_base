# Under the Hood: Go Channel and Select Internals
## Golang Channel
```golang
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	timer    *timer // timer feeding this chan
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
For simplicity, we only discuss buffered channels here.

Conceptually, a golang channel consists of three core components:
1. A circular buffer that stores elements
2. Two queues (linked lists) holding blocked goroutines(*runtime2.sudog)
3. A mutex for synchronization. 

Goroutines that send values to a channel are called senders, and those that receive values are receivers.

When the buffer is empty, a receiver blocks until another goroutine sends an element (or closes the channel). Similarly, when the buffer is full, a sender blocks until a receiver retrieves an element.

So how is such blocking mechanism implemented? 

Here comes the functionality of two queues - `sendq` and `recvq`. 

- When a sender tries to push an element into a full channel, the sender gets blocked by its corresponding `P`(logical processor) and is shifted to the rear of the `sendq`. 

- When a receiver tries to fetch an element, it first pops one from the circular buffer (if available). At the same time, if there’s a waiting sender in sendq, the channel dequeues it, stores its value in the buffer, and wakes up the sender’s goroutine.

## Select Clause
The select mechanism in golang is particularly intriguing. Here’s a high-level overview of how it works internally:

1. Evaluate expressions for send-to-channel cases.
2. Sort and generate a constant lock order and polling order of channels.
3. Lock all the channels one by one following the order specified in step 2.
4. Try each case arm(try-push or try-pop to channel) one by one following the polling order. If any case succeeds, or there is a default branch, unlock all channels by the inverse order and execute the case branch, go to step 8. Otherwise, go to step 5.
```golang
for _, casei := range pollorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c

    if casi >= nsends { // receive from a channel
        sg = c.sendq.dequeue() // check if there is a sender waiting in queue
        if sg != nil {
            goto recv // accept an element from a sender
        }
        if c.qcount > 0 { // when try pop fails, the sender queue must be empty, we further check whether is buffer is empty.
            goto bufrecv // accept an element from buffer
        }
        if c.closed != 0 {
            goto rclose // if the channel is closed, accept zero value
        }
    } else { // send to a channel
        if raceenabled {
            racereadpc(c.raceaddr(), casePC(casi), chansendpc)
        }
        if c.closed != 0 {
            goto sclose // send to a closed channel cause panic
        }
        sg = c.recvq.dequeue() 
        if sg != nil { // check if there is a receiver waiting for an element
            goto send // send to a receiver
        }
        if c.qcount < c.dataqsiz { // when there is no receiver in queue, check whether the buffer is full
            goto bufsend // the buffer is not full, send the value to buffer
        }
    }
}

if !block { // check if there is a default case
    selunlock(scases, lockorder)
    casi = -1
    goto retc
}
```

5. Enqueue the current goroutine to each channel's waiting queue. 
- For receive cases → append to recvq.
- For send cases → append to sendq.
```golang
// pass 2 - enqueue on all chans
gp = getg()
if gp.waiting != nil {
    throw("gp.waiting != nil")
}
nextp = &gp.waiting
for _, casei := range lockorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c
    sg := acquireSudog()
    sg.g = gp
    sg.isSelect = true
    // No stack splits between assigning elem and enqueuing
    // sg on gp.waiting where copystack can find it.
    sg.elem = cas.elem
    sg.releasetime = 0
    if t0 != 0 {
        sg.releasetime = -1
    }
    sg.c = c
    // Construct waiting list in lock order.
    *nextp = sg
    nextp = &sg.waitlink

    if casi < nsends {
        c.sendq.enqueue(sg)
    } else {
        c.recvq.enqueue(sg)
    }

    if c.timer != nil {
        blockTimerChan(c)
    }
}
```
6. Blocking the current goroutine and unlock all channels in reverse order. 
7. When the goroutine is awakened by a channel operation, it must have already been removed from that channel’s waiting queue by the operation that triggered the wake-up. Upon resuming, the goroutine reacquires all channel locks, removes itself from the waiting queues of all other channels involved in the select, and then proceeds to execute the branch corresponding to the channel operation that caused the wake-up.
```golang
for _, casei := range lockorder {
    k = &scases[casei]
    if k.c.timer != nil {
        unblockTimerChan(k.c)
    }
    if sg == sglist {
        // sg has already been dequeued by the G that woke us up.
        casi = int(casei)
        cas = k
        caseSuccess = sglist.success
        if sglist.releasetime > 0 {
            caseReleaseTime = sglist.releasetime
        }
    } else {
        c = k.c
        if int(casei) < nsends {
            c.sendq.dequeueSudoG(sglist)
        } else {
            c.recvq.dequeueSudoG(sglist)
        }
    }
    sgnext = sglist.waitlink
    sglist.waitlink = nil
    releaseSudog(sglist)
    sglist = sgnext
}
```
8. Done.

You may have noticed that race condition may occur when two channels attempt to wake up the same blocked select concurrently. Go runtime solves this with an atomic variable stored in the goroutine’s g structure, named `selectDone`.  

When a channel operation tries to remove a goroutine from its waiting queue, it first checks whether the goroutine is currently participating in a select.

If so, it attempts to atomically set selectDone from 0 to 1 using a compare-and-swap (CAS)

Since CAS is atomic, only one channel can succeed. The successful one is responsible for waking the blocked goroutine, while others back off once they see that selectDone is already set.

The following is the source code.

```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
	lock(&c.lock)

	if c.closed != 0 {
		// ...
	} else {
		// Just found waiting sender with not closed.
		if sg := c.sendq.dequeue(); sg != nil {
			// Found a waiting sender. If buffer is size 0, receive value
			// directly from sender. Otherwise, receive from head of queue
			// and add sender's value to the tail of the queue (both map to
			// the same buffer slot because the queue is full).
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}
    // ...
}

func (q *waitq) dequeue() *sudog {
	for {
		// ...

		// if a goroutine was put on this queue because of a
		// select, there is a small window between the goroutine
		// being woken up by a different case and it grabbing the
		// channel locks. Once it has the lock
		// it removes itself from the queue, so we won't see it after that.
		// We use a flag in the G struct to tell us when someone
		// else has won the race to signal this goroutine but the goroutine
		// hasn't removed itself from the queue yet.
		if sgp.isSelect && !sgp.g.selectDone.CompareAndSwap(0, 1) {
			continue
		}
		return sgp
	}
}


func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// ...
	goready(gp, skip+1)
}

```






