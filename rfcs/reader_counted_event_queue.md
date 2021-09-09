# Feature Name: `reader_counted_event_queue`

## Summary

Eliminate 2 frame event limitation, eliminate need in ManualEventReader. Without performance sacrifice.

Achieve through integration of https://github.com/tower120/rc_event_queue.

## Motivation

Current Event implementation may be considered as logically dangerous, since systems that rely on events may simply 
 not receive messages. And this could happen silently. ManualEventReader solution ... is not ideal.

This RFC attempts to solve this problem completely.

## User-facing explanation

It looks like, there should be no changes to current state of affairs.
Except that events will survive 2 frames window.

## Implementation strategy

[How "reader counted event queue" works.](https://github.com/tower120/rc_event_queue/blob/master/doc/principal-of-operation.md)

`rc_event_queue` should work in `AUTO_CLEANUP = false` mode. Meaning that we will call `cleanup` manually.
For each EventType - make corresponding EventQueue resource. 

EventType can be non-cloneable.

For each system that read - make EventReader, and store it as system local resource. Get `EventReader::iter()` before each
system run, pass that Iterator to the system. Alternatively, pass `EventReader`, so user can make iterator himself.

For each system that write - store `Arc<EventQueue>` as system local resource. Write with `EventQueue::extend`. For this,
use `small_vec<EventType>`, or `vec<EventType>` as local resource. Provide user with means of calling `EventQueue::clear` - 
this additional functionality may come in handy, if user decides to clear event queue, and accessing to previous messages
may be dangerous (Like level reloaded, and resource/entity pointed in event message are no longer valid).

Maintenance (reclaim memory):
* Each frame - for each `EventQueue` call `cleanup`. 
* **[SAFETY MEASURE]** Each Nth frame - check length of each `EventQueue`. If it is too big, truncate first chunks/clear (this will not free memory yet). 
  Next (this will move all EventReaders to new position immediately):
  - Either. On all associated EventReaders should be called `update_position()` (EventReaders should be presumably "queried" from systems locals, that read that EventType). 
  - Either. Each frame, in scheduler, for systems with EventReaders, that not run,  `update_position()` should be called.
    This is very fast operation.

If there is time window, where nothing happens, and we just like wait for v-sync, it is a good place to put maintenance there. 
Maintenance can be skipped few frames, and sliced between several frames. But I think this should not be necessary, since I expect
maintenance to be very fast.

## Drawbacks

Potential implementation complexity of calling EventReaders `update_position()` in maintenance [SAFETY MEASURE] part.
But it is _only_ needed to guarantee that queue will not get 100Mb's len. But if that happened - almost certainly 
something wrong with game logic. Maybe we should not try to save the day, and just panic instead?

## Rationale and alternatives

It is possible to use [rc_event_queue](https://github.com/tower120/rc_event_queue) without engine integration.
But ergonomic will be much better, having first class support.

https://github.com/bevyengine/rfcs/pull/17 tries to solve the same problem, but in somewhat different way. I think 
[rc_event_queue](https://github.com/tower120/rc_event_queue) - based implementation should be faster in all operations. Since
for read - only atomic read is needed; for write - Mutex used, instead of RwLock; cleanup is faster due to chunk-based nature, and being O(1).
Also, `cleanup` is safe to call during system run, since readers does not lock, and writer locks is short - so it is safe to
have long run systems which could took several frames to finish their run. It is also safe to run multiple writers at once, 
along with readers too! _It also can work, without need to manually call `cleanup` (with `AUTO_CLEANUP = true`)._ 

## Unresolved questions

- Should we truncate queue, or panic on reaching queue's length limit?
- Do we need `clear` functionality? I personally think, yes.

## Future possibilities

Maybe it would be nice to have soft and hard queue length limits:
- On reaching soft limit callback/special error event? will be triggered.
- On reaching hard limit event queue will be trimmed to some size. _Or panic?_