## Assignment 1

### `isLocked()` for `testAndSet` lock

The idea is to replace boolean locked state with the thread ID:

```java
class TTASLock {
  AtomicLong lockOwner = new AtomicLong(0);
  
  public boolean isLocked() {
    return lockOwner.get() == Thread.currentThread().getId();
  }
  
  public void lock() {
    long tid = Thread.currentThread().getId();    
    if (lockOwner.get() == tid) {
      // Already locked --- todo support reentrant locking
      return;
    }
    while (true) {
      while (lockOwner.get() > 0) { }
      if (lockOwner.compareAndSet(0, tid)) {
        // successfully locked
        return;
      }
    }
  }
  
  public void unlock() {
    long tid = Thread.currentThread.getId();
    lockOwner.compareAndSet(tid, 0);
  }
}
```

Unfortunately, the lock is actually no longer a TTAS lock, but rather a "TCAS" lock, i.e. it uses a more powerful CAS operation. This design isn't going to work with testAndSet(), because if several threads at once execute it, there is no guarantee on the final state of lockOwner.

### `isLocked()` for CLH lock

The solution is trivial: the thread owns the lock iff its local node is locked.

```java
class CLHLock {
    public void isLocked() {
        return tlNode.get().locked;
    }
}
```

To show that it's correct, consider the last lock/unlock operation preceding the call to `isLocked()` in this thread. If it was `lock()`, then this thread must be holding the lock, because otherwise it could not be calling `isLocked()` since it would be spinning in `lock()`. But in that case `node.locked` was set to `true` by `lock()`. If it was `unlock()` or none, then the thread cannot hold the lock; but the method set `node.locked` to `false`.

### `isLocked()` for MSC lock

```java
class MSCLock implements Lock {
  AtomicReference<Qnode> tail = new AtomicReference<>();
  ThreadLocal<Qnode> tlNode = ThreadLocal.withInitial(Qnode::new);

  public boolean isLocked() {
    Qnode qnode = tlNode.get();
    Qnode next = qnode.next;
    return next == null ? tail.get() == qnode : next.locked && !qnode.locked;
  }
  
  // The rest is unchanged
}
```

#### Correctness

1. Case `next == null`. Consider point in time after doing `tail.get()`. Lemma: `tail.get() == qnode` iff we own the lock.
  - If `tail.get() == qnode`,  we're at the end of the queue. This is the state after we've exited `lock()`, and no other thread has attempted to acquire the lock yet. We know that because that thread will first need to do CAS on `tail`.
  - Conversely, suppose that we own the lock. `next == null` means that either no other thread has attempted to grab the lock, or it is in the process of doing that (inside `lock()`). TODO


The method considers two cases when we can hold the lock.

1. We are at the end of the queue.
    - If `next == null` and `tail.get() == qnode` are both true, that means that we're holding the lock: that's the state after exiting `lock()`. It's in fact "if and only if": `next` is never set to `null`, if it's null that means that it was never modified, that is, no other thread tried to acquire the lock since we left `lock()`.  
    - Can any of these two expressions be false in this case, while we're still holding the lock? Note that if `next` is not null, then we're definitely not at the end of the queue. So the only case left to consider is when `tail.get() != qnode`. If it's `null`, that means that  


### A generic solution

The problem can be solved for any lock by holding isLocked state in a separate ThreadLocal boolean variable. It is set to true when lock() is succeeded before the method returns, and set to false wen unlock() is successful (in MSC lock that happens after a busy wait) before the method returns. That's great, but we are storing more state compared to the specific solutions given above.


## Assignment 2

The problem is that this implementation is prone to deadlock. Here's how: let's have two threads, A and B, and let's denote the corresponding thread local nodes as nodeA, nodeB.

1. Thread A acquires lock. Linked list: `nodeA = tail`.
2. Thread B attempts to acquire lock, gets preempted while spinning on line 8. Linked list: `nodeA -> nodeB = tail`.
3. Thread A releases lock. Linked list: `nodeA -> nodeB = tail`.
4. Thread A attempts to acquire lock. After line 7, linked list looks like this: `nodeA -> nodeB -> nodeA = tail` -  a deadlock! Thread A will spin on nodeB, which is never going to be unlocked.

MSC lock as given in the lecture always allocates a new node. Let's suppose the task is to consider a modified implementation that reuses the lock:

```java
class MCSLock implements Lock {
  AtomicReference<Qnode> tail = new AtomicReference<>();
  ThreadLocal<Qnode> tlNode = ThreadLocal.withInitial(Qnode::new);
  
  public void lock() {
    Qnode qnode = tlNode.get();
    Qnode pred = tail.getAndSet(qnode);
    if (pred != null) {
       qnode.locked = true;
       pred.next = qnode;
       while (qnode.locked) {}
    }
  } 
  
  public void unlock() {
    Qnode qnode = tlNode.get();
    if (qnode.next == null) {
      if (tail.CAS(qnode, null))
        return;
      while (qnode.next == null) {}
    }
    qnode.next.locked = false;
    qnode.next = null;
  }
} 
```

Let's consider the sequence of states of the nodes in the linked list using the same sequence as above.
 
| Action | nodeA | nodeB | tail |
|--------|-------|-------|------|
| 1. Thread A acquires lock | locked=false, next=null | - | nodeA |
| 2. Thread B attemps to acquire lock, gets preempted while spinning | locked=false, next=nodeB | locked=true, next=null | nodeB | 
| 3. Thread A releases lock | locked=false, next=null | locked=false, next=null | nodeB |
| 4. Thread A acquires lock | locked=true, next=null | locked=false, next=nodeA | nodeA |

MSC works because threads are spinning on their nodes, whereas in CLH they are spinning on the predecessor node, which allows for cycles in the wait list. With MSC design, cycles of this kind are impossible.  