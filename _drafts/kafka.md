write to event stream / transaction log
stream must support only 1) append 2) read in sequence
kafka
materialized views consume the stream
need a framework to build materialized views (stream processing framework)
samza does just that
use compaction to keep data storage under control, instead of having a limited history
kappa architecture as opposed to lambda architecture

for compaction to work, we need to follow the contract: for messages w/ duplicate key, only the latest must be preserved. thinking through this...
- channel created (key: channelId) <- bad because now all other streams have to be processed after this one
- channel updated (key: channelId)
- channel content added
- channel content removed
  - merge these into "channel content mutated" (key: channelId-contentId)



ordering!!!
- if all one topic, key has to be channelId. that breaks compaction since we need to preserve multliple messages per channel. maybe there's a way to confgure a more advanced compaction scheme?
- if separate streams, how can you enforce ordering between topics?



also, we need to process messages in order, right? how to enfore that?
- partition key the same for all messages that need to be processed in the same order
- how to ensure order in the mat. view stream consumer workers when distributed?

quetions:
- are all channel events part of a single stream or mutliple streams?
  - if single, can we have disperate message shapes?
  - if multiple, how can we ensure ordering?

why do all of this?
1) better data
	- current databases conflate the concerns of the reader or the writer. now we can optimize for both.
	- events are more useful for analytics (all states, not just final state)
	- point in time snapshot (recovery from human error)
2) materialized views are fully precomputed caches
	- very fast (never a cache miss)
	- don't have to solve the problem of when to invalidate the cache
3) streams everywhere
	- clients subscribe to marialized view changes
	- Meteor, Firebase, ...?

kafka "exactly once" semantics are in the works (wrap multiple messages in a transaction, get confirmation of success and if fail, all are rolled back).

have to be able to handle all versions of a message since we will reprocess them from the beginning of time.


SAMZA
- lots of pieces...



todo:
samza / storm
firebase / meteor
[kafka-node](https://www.npmjs.com/package/kafka-node)
