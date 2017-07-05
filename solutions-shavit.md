## Assignment 1

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