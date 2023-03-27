# Redis Work Queue

A work queue, on top of a redis database, with implementations in Python, Rust, Go and C#.

This provides no method of tracking the outcome of work items. This is fairly simple to implement
yourself (just store the result in the redis database with a key derived from the work item id). If
you want a more fully-featured system for managing jobs, see our [Collection Manager](#).

Implementations in other languages are welcome, open a PR!

## Examples

### Setup

### Adding jobs

### Claiming and completing jobs

## More details

All the implementations share the same operations, on the same core types, these are:

### Items

Items in the work queue consist of an `id`, a string, and some `data`, arbitrary bytes.

For convenience, the IDs are often randomly generated UUIDs, however they can be customized.
Another item with the same ID as a previous item shouldn't be added until the previous item has been
completed.

### Adding an item

*Python: [`WorkQueue.add_item`](#), Rust: [`WorkQueue::add_item`](#), Go: [`WorkQueue.AddItem`](#)*

Adding an item is exactly what it sounds like! It adds an item to the work queue. It will then
either be in the queue or being processed (before coming back to the queue if the processing fails)
until the job is completed.

### Leasing an item

*Python: [`WorkQueue.lease`](#), Rust: [`WorkQueue::lease`](#), Go: [`WorkQueue.Lease`](#)*

Workers wanting to receive a job and complete it must start by obtaining a lease.

When requesting a lease, you exchange an expiry time for an item. The worker should then complete
the item before the expiry time by calling `complete`. If `complete` isn't called in time, it's
assumed that the worker died and the item is returned to the queue for another worker to pick up.

This means that a worker can receive a job that another worker has already partially or fully
completed (and then died before calling `complete`) or even for two workers to be simultaneously
working on the same job if the lease expiry was too short (try to avoid this if possible!). It's
therefore important that workers are written in a way that won't cause problems if a worker starts
again after another worker has already fully or partially completed the task, or is working on it at
the same time. This allows a fully resilient system.

The work queue cannot loose track of a job once it's been added, so, as long as workers keep
successfully working, a job will always be run to completion (even if it is run multiple times in
that process).

If you're unhappy about jobs being run more than once, see [But I never want my job to run more than
once](#).

#### Storing the result of a work item

The work queue provides no method of tracking the outcome of work items. This is fairly simple to
implement yourself (just store the result in the redis database with a key derived from the work
item id). If you want a more fully-featured system for managing jobs, see our [Collection
Manager](#).

#### Handling errors

If an error occurs and the job should be retried, later on, by the same or different worker, then
the worker should **not** call `complete` and should obtain another lease and work on the next item,
ignoring the one it was previously processing. When the previous lease expires, it will be returned
to the work queue and will be retried. For example:

```python
while True:
    job = work_queue.lease(100)
    # ... do some work ...
    if should_try_again_later:
        # Don't call complete, just get another lease
        continue
    # ... finish the work ...
    work_queue.complete(job)
```

If an error occurs that means the job shouldn't be retried, you should send this error to the
correct place (perhaps the same place you put your results) and then call `complete`. The job then
won't be run again.

#### But I never want my job to run more than once

Before following the instructions below, you should think really hard about the title statement. If
the job can't run more than once then, and the worker dies during the work, the work will be left
incomplete, forever... *and ever...* <small>*and ever...*</small> (unless you have your own error
recovery system)

It's possible to write almost all jobs in a way which allows it to be restarted if a worker node
dies. If you can it's probably worth the effort!

##### I still think I want my job to only ever possibly run once

If this is the case, you should call `complete` (**and check the return value**) immediately
after obtaining the lease.

For example, in Python:
```python
job = queue.lease(1000)
if queue.complete(job):
    # This will only run once, per job, ever, even if the worker dies
    foo(job)
```

This works because `complete` returns `true` *iff* it is the worker that completed the job. So while
`lease` may return the same job many times, `complete(job)` will return `true` only once per job.

### Completing an item

*Python: [`WorkQueue.complete`](#), Rust: [`WorkQueue::complete`](#), Go: [`WorkQueue.Complete`](#)*

Complete marks a job as completed and removes it from the work queue. After `complete` has been
called, no workers will receive this job again.

`complete` returns a boolean indicating if this worker was the first worker to call `complete`. So,
while `lease` might give the same job to multiple workers, `complete` will return `true` for only
one worker.

#### Storing the result

See [Storing the result of a work item](#)

### Cleaning

#### Light cleaning

*Python: [`WorkQueue.light_clean`](#), Rust: [`WorkQueue::light_clean`](#), no Go or C#
implementation*

When a worker dies while processing a job, or abandons a job, the job is left in the processing
state until it expires. The role of *light cleaning* is to move these jobs back to the main work
queue so another worker can pick them up.

The interval *light cleaning* should be run on should be approximately equal to the shortest lease
time you use.

#### Deep cleaning

*Python: [`WorkQueue.deep_clean`](#), Rust: [`WorkQueue::deep_clean`](#), no Go or C#
implementation*

In addition to this, a worker dying in the middle of a call to `complete` can leave database items
that are no longer associated with an active job. The job of a *deep clean* is to iterate over these
keys and make sure the database is clean.

It's very rare that deep cleaning is needed, but it can happen if you get really unlucky, so it
should be run automatically but infrequently.

The cleaning process we provide runs this every 6 hours by default.

#### Cleaning process

When there are many workers of different types, it's simpler just to have a dedicated process
running the cleaning. We provide a simple cleaner, both in Python and Rust.

### Other operations

#### Getting the queue length

*Python: [`WorkQueue.queue_len`](#), Rust: [`WorkQueue::queue_len`](#), Go: [`WorkQueue.QueueLen`](#)*

#### Getting the number of leased items

*Python: [`WorkQueue.processing`](#), Rust: [`WorkQueue::processing`](#), Go: [`WorkQueue.Processing`](#)*

This includes items being worked on and abandoned items (see [Handling errors](#)) yet to be
returned to the main queue.

## Under the hood
