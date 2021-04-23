# Feature Name: `automatic_event_tracking`

## Summary

This feature would change Events to track EventReader access. Any EventReader used in a system would automatically subscribe to its associated event type. The EventReaders would then access a shared resource in order to track which events have been read by all subscribed EventReaders. The read events could then be safely discarded, without worrying about an EventReader having missed any.

## Motivation

The current double buffering approach is error prone when used in systems with run criteria. If it takes longer than two frames for the run criteria to be met, both buffers will have been cleared. The issue can be circumvented by clearing events after they have been accessed by all subscribed EventReaders.

## Draft Implementation

https://github.com/bevyengine/bevy/pull/1776

## Guide-level explanation

*TBD, may not require api changes*

## Reference-level explanation

Events will need a way to tell which part of the buffer has been read by all relevant EventReaders. The EventReaders will need to track this information on each read in a way that allows the information to be accessed by an update system. This update system can then find the oldest event that has been read by all EventReaders, and discard all events up to that point in the buffer. Since the EventReaders are not components, they will need to do this using data available to SystemParams. Each Event type will also need its own EventReader / Subscription types. Ideally they will be able to access the shared subscription / read events data without blocking on write access, so that they can continue to read events in parallel.

## Drawbacks

- Currently uses an RwLock for the shared EventReader related data (this can be changed to use commands if Commands are fixed to work when in SystemParams.)
- Might be too opaque
- Can end up with unbounded memory usage if the user has a system with an EventReader or creates a ManualEventReader that is never used. (Could maybe warn about this in debug builds?)

## Rationale and alternatives

This would involve minimal / no api changes, and would allow us to avoid any lost events. If the user forgets to run Events::update, the current method of making sure events are not lost can lead to unbounded memory usage.

## Unresolved questions

- ~Should subscriptions occur on first read, or as soon as a system with an EventReader is added? The former could lead to an EventReader missing out on events prior to the first time the system was run, the latter would have a more complicated implementation~ Not actually that complicated after all
- Should EventReaders only update their status when they've iterated through the events, or should it happen whenever the system is executed?
- Should ManualEventReader be kept? It might no longer be necessary / useful, and would potentially have a messier API or would suffer from the original issues around skipped events.
  - It seems like there are still some cases where it's easier to use that pattern, so maybe the real question is how should the api for those work? The draft currently has them using string ids to keep track of subscriptions.
- Is there a way to just get all relevant EventReaders and iterate through them instead of having a shared resource?
  - This wouldn't work with the current draft ManualEventReader implementation, unless they had a shared resource separate from how EventReaders were accessed.
- Is there a way to have ManualEventReaders track where they were called from to get rid of the name / string requirement?
- Is there a straightforward way to get rid of the RwLock on the EventReader event counts data?
  - Commands inside of SystemParams don't currently work, but either fixing that or implementing similarly postponed updates would fix this issue.
