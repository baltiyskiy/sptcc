## 0. Is the algorithm wait-free or just lock-free?

The algorithm is wait-free, since every thread makes a bounded number of steps regardless of other threads' action or inaction: O(1) in `enqueue()`,  and O(n) in `dequeue()` where `n` is the counter value, read only once.

## 1. Is the algorithm linearizable?

The algorithm is not linearizable because of the same property that makes it wait-free. Since counter value is read only once in `dequeue()`, this function cannot see the effect of a concurrent `enqueue()`, and can return `null` even if the queue is not empty. To show that this leads to non-linearizability, here's a sequence of operations where a `dequeue()` cannot be linearized (numbers indicate time):

1. enqueue(x) starts
2. enqueue(x) finishes
3. dequeue1 starts
4. enqueue(y) starts
5. enqueue(y) finishes
6. dequeue2 starts
7. dequeue2 finishes, returns x
8. dequeue1 finishes, returns `null`

This execution takes place if step 4 happens after `dequeue()` reads the counter but before it  enters the loop. It observes value 1, since enqueue(x) has already finished on step 2. Then assume that this thread is preempted, and resumes execution after step 7. Since `x` has been dequeued, it observes `null` as a result of `swap`, and the loop then finishes, the function returns `null` -- even though there is `y` at index `1`. 

Now we prove that we cannot linearize `dequeue1`. Indeed, if we could, the linearization point would lie between 3 and 8. But at no point the queue is empty: after step 2, it contains `x`, after step 5, it contains `x, y`, after step 7, it contains `y`.

## 2. How do we fix the algorithm?

To fix, we must re-read the counter on each iteration in `dequeue()`:

```
for (k:=0; k < c; ++k) {
    v:=swap(vals[k],null)
    if (v ≠ null)
        return v
}
return null
```

Notice how the algorithm becomes lock-free and not wait-free, since an individual thread cannot always have progress if other threads keep incrementing the counter in `enqueue()`.