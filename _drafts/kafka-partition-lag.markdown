---
layout: post
title: "Experience Report: Kafka Partition Lag"
date: 2016-08-15T14:45:06-04:00
categories: postmortem
comments: true
tags: kafka, ops
---

<style>
  aside {
    background: #eeeeee;
    padding: 1rem 30px;
    margin: 1rem -30px;
    font-size: smaller;
    color: #422;
  }
</style>

Recently, I and a number of colleagues spent the better part of a week chasing
down some baffling behavior in a kafka consumer. After a routine[^routine]
cluster upgrade, we observed that _one_ of the partitions in a deal publication
topic was lagging further and further behind, negatively affecting our
production processes.

[^routine]: Not actually all that routine, as we blanked out all of our existing
            topics, chasing a clean start.  This had the effect of changing
            the number of partitions on this topic.

By the end of the week, we'd chased down the issue, and have determined that
it was the result of the confluence of a number of factors which had been
lying dormant until the right combination of circumstances arose. I think the
combination is subtle and interesting enough to peel back the curtain a bit
and see what it was that bit us.

<aside markdown='block'>
### TL; DR

  Very large messages can interact in pathological ways with blocking consumer
  fetches in a consumer that has claimed multiple partitions. Either increase
  the `max_bytes` when fetching a message batch, or reduce the `socket_timeout`,
  the time spent waiting on a message batch to return.
</aside>

## A quick Kafka primer

Kafka is a distributed message log, where the brokers themselves are relatively
simple.  Each message topic is divided into partitions, which guarantee that the 
messages will be sent to consumers in the same order as they were received.

A producer can "control" where a message ends up by setting a partition key;
this is used to deterministically assign the message to a partition.  For the
topics in question, we use the deal id. Messages about deal `2342565` will
_always_ go to partition 4.  By ensuring that all messages for a deal go to
the same partition, we prevent a scenario where an old change, processed late,
can overwrite the newest values.

When processing a topic, each partition is claimed by a single consumer, which 
will process messages in order.  If there are 12 partitions, and 4 consumers,
then a properly balanced consumer group will have each consumer processing 3
partitions.  In order for a topic to retain it's ordering guarantees, the
consumer must process those messages _sequentially_.

Because each topic may be handled by only one consumer, the fundamental unit
of parallelization for a kafka topic is partitions.  If you have more
consumers than partitions, the leftover consumers will simply sit idle, waiting
for a rebalance.

## Our setup

As a part of our internal service architecture, we run a small kafka 0.8
cluster with a number of topics that broadcast changes made to our deals.  This
separates the services responsible for providing deal data to our public-facing
web and mobile interfaces are disconnected from the applications that
allow our internal teams to edit those deals.

A number of consumer processes are run on internal servers, and publish
the contents of the `deal_updated` topic into the catalog service.  These consumers 
are ruby daemons use the <tt>[poseidon_cluster]</tt> gem[^deprecated] to interact with 
the remote brokers.  The consumer uses a simple consumer group protocol,
where each partition claimed by a consumer is processed in a single thread,
round-robin fashion.

As near as I'm aware, outside of running in ruby, which is suffering from some
weak driver support at the moment, this is a bog-standard kafka 0.8 setup.
We've been running this configuration, without issue, for over a year.

Well, that is until ...

[poseidon_cluster]: https://github.com/bsm/poseidon_cluster

[^deprecated]: Yes, I know the gem is marked deprecated.  It was not when
               we wrote the consumers that use it.  This is one of the reasons
               our cluster is still on kafka 0.8.

## Lagging

Sometime late on the Monday following our cluster upgrade, we get a confused
inquiry from our internal users:  It seems that a small number of deals were
not reflecting changes on their respective preview pages.  After some digging
around, we determined that a single partition was lagging behind, with
nearly 2000 unprocessed messages.

Some further investigation of partition 7, and we noticed that most of the
messages stuck in the partition queue had one of a small set of deal IDs.
Parallel investigation determined that there were no relevant code changes
made on any of the systems involved.  Whatever broke was related, in some way,
to the cluster upgrade over the weekend.

After the first day's investigations, we had determined:

 * this particular consumer was using an outdated version of our kafka consumer
   gem.
 * The partition in question had a backlog consisting primarily of messages
   relating to the same small set of deal IDs
 * _no other_ topics were adversely affected.  Whatever the issue was, it
   was strictly related to this topic, or these consumer applications.
 * There were no code changes made to the applications.

It was, however, nearing the end of the day.  Hoping we just had a horrible
clump related to the weekend cluster outage, we forwarded the consumer's
progress pointer to the end of the partition, and force republished the deals
that needed to make it through the message queue.

Our freshly caught up partition remained caught up for less than an hour.


## I could swear I've seen this before

So, we knew that we had a number of seriously chatty deals.  Investigation of
the specific deals in question lead us to discover that these deals were also
_large_, weighing in at just under 800kb of serialized JSON. Because of a web
of `after-save` callbacks in the management application, they also tended to
get published many times in a row.  We would see a run of 2-300 copies of the
same deal in the lagging partition, presumably with subtle differences in
each step.


<aside markdown='block'>
### Callback hell
I understand **why** the publication was tied together via ActiveRecord
callbacks; it was the same fear that paralyzed me at this point: "what if we
miss an update point?"  The production application is one of our original
monorails, dragged kicking and screaming through a number of framework
upgrades.  To say that it features a variety of code styles and half-completed
attempts to impose structure is to put it mildly.


Still, the time to search through an application and find the locations
to _intentionally_ publish updates would have been while the feature was in
development -- schedule pressure be damned -- because at that point,
_nothing depends on it_.

I am upgrading my opinion of AR lifecycle callbacks from: "poor idea" to 
"the devil's playground".
</aside>


So, a partition flooded by runs of messages applying to the same deal ID. 
We've also figured out another piece of the puzzle, about what has changed
with _this_ topic:  As one of our original topics, it had been generated with
a very small number of partitions, an out-of-the-box default of 3.  In the time
since, we've updated that default to 12.  When we upgraded the cluster over
the weekend, we wiped out the existing topics, and our `deal_changed` topic
got recreated with a larger number of partitions, which was when our problems
started.

There's a hint in that observation, and we missed it for another two days.

Our assumption was that we got unlucky.  Two of these very clumpy deals just
_happened_ to end up landing on the same partition; if we could break them up,
then two different consumers could be dedicated to the process of burning
through that backlog, rather than just the one.

The way that kafka determines what partition a message goes onto more or less
boils down to generating a hash from the partition key, and bucketing that in one
of the available partitions:

```
  partition = message.partition_key.hash % parititon_count
```

Armed with that idea, we decided to create a few new partitions, and see if
that would split up our clumps, and give the consumers the space they needed to
burn down their backlog.  So, we bumped from 12 to 20 partitions[^partition_count], gave
everything a chance to rebalance and work through the new distribution.

The good news: we broke up the clump.

The bad news:  Now we had _2_ lagging partitions.

We did, however, see glimmers of hope in the lag numbers, as they _were_
slowly going down.  It was nearing the end of the day, and with our production
team not likely to be updating deals in the off-hours, we figured the queues
would burn down overnight.

It has been said that software developers are some of the most optimistic
people on the planet:  "Surely, I've fixed that bug now!" and, of course,
our sorry history of project estimation.  I think it's how we defend our
sanity.

You might guess that our backlog was _not_ burned off overnight.

It was around this time, I think, that I started making jokes about Cthulhu
using my brainpan as a coffee mug.

[^partition_count]: Something we were not able to do until we'd performed the
                    previous weekend's cluster upgrade ...

## ETOO_MANY_MESSAGES

One of the major pain points we were seeing was the repeated duplication of
a message for the same deal, over and over.  We knew this was the result of
an after_save hook, but we weren't expecting it to continue firing during the
night, which is why there was almost no progress made on the backlog when
we checked in the next morning.

It turns out that we have a cron job that, for certain types of deals, runs
hourly, and updates every option that deal has. We attempted to avoid sending
so many messages by adding the nasty-hack of storing a last-sent hash of each
message in a global cache, and not resending the message if there was
no change.

This had approximately zero impact.

Further investigation revealed that this cron job shut off each individual
option in the deal, and then selectively turning some of them back on again,
based on the results of a web-service call.  The (biggest) problem deal had
roughly 600 options, and serialized as an 800kb json blob.

This cron job was saving each option individually.  Twice, for most -- not
every option gets turned back on.  We were, essentially, sending 1000 messages,
weighing in at 800kb each ... roughly 800 megabytes of updates to the same
deal.  Every. Single. Hour.

Well.  No wonder that partition is lagging behind.

I rigged up a shut-off block that turns off the `republish` after-save hook for
anything run inside the block, then altered the cron task to explicitly
republish _once_, when it's finished all of it's processing.  

No more data-bomb; things should catch up and we _should_ be smooth from there
... only, this wasn't a new deal, and it's presumably been doing this for a
while, so why did it _suddenly_ start killing our consumers?  There's still
more to find here.

## We're missing 30 seconds

Day three was spent adding instrumentation.  I brought the consumer up to date
with our kafka consumer gem, to catch up with some of the logging fixes that
were present. Then, for good measure, I cut a new release that added some
additional graphite instrumentation around message size and how long it takes
to communicate with ZooKeeper.  Brian performed his splunk wizardry, and Chris
began a process of attempting to teach Newrelic how to read our consumers.

At this point, we had logs and stats, and we **knew** that, somehow, we were
losing 10-30 seconds between messages on this topic. We just didn't know where
they were coming from. Every tracked and instrumented interaction showed normal
time.

Somewhere in here, we got to playing with message fetch defaults, and an
explanation on the consumer is in order:

In order not to be an excessively chatty protocol, the consumer pulls messages
from kafka in chunks.  It usually works out to around 50 or so messages, pulled
from the kafka server at once.  The chunk is processed, message-by-message
until completed, then a chunk is pulled from the next partition.  By
round-robining through partitions like this, it prevents any one partition
from totally blocking the consumer.

So, with that in mind, and looking at message sizes ... we realized that, for
these problem messages, a chunk would really only consist of a single deal.  By
increasing the maximum fetch size, we got the consumer to pull down around
10-20 of these larger messages at one time.

That did it. Our burnoff rate suddenly picked up, and trend-lines indicated
we'd get through the lag-induced backlog in about 10-12 hours.  Combined with
the fix of not bombing the system, we should be in the clear now.

Problem solved.   Only ....

## One more 'why?'

We were still missing 30 seconds or so, every run, that we couldn't account
for.  The runs were now more efficient, but there was still a nagging
suspicion that we weren't quite there yet.

Brian started looking at various configurations, and eventually homed in on 
a networking timeout, set to 10 seconds.  It didn't look like a likely
candidate -- we were both quite skeptical -- but, wouldn't hurt to lower it.

So, we dropped the timeout from 10 seconds to 1, and deployed .... and soon,
the lagging messages started falling like dominoes.  Our 12 hour backlog,
already a hard-fought win, was chewed up and spat out in about 20 minutes.


So ... it's a timeout.  Which is when the light goes on. _This_ is what
changed, why we went from a perfectly functional system to something that
just could not keep up, with exactly zero code changes.

A week prior, the topic had been configured with 3 partitions.  The consumers
are run on 6 separate worker boxes, and since a partition can only be claimed
by a single consumer ... we really had 3 active consumers, and 3 dormant ones,
doing nothing.  Each of those active consumers would just happily chew through
it's claimed partition, and outside of a few disk space alarms earlier in
the year, we were generally unaware just _how_ chatty some of our
messaging had become.

Fast forward to after the topic rebuild; We now have 12 topics -- later 20 --
being consumed by those same 6 boxes.  Now, the consumers are no longer sitting
idle, and they are no longer dedicated to a single partition.  Assuming a
normal balance, Consumer A is pulling data from partitions 3 and 10.  It 
pulls a chunk of messages from partition 3, burns through them, and then 
grabs the next batch from partition 10.

Only, when partition 10 doesn't have any messages, it blocks.  To avoid heavy,
constant network chatter, the consumer will block for 10 seconds, waiting for
the server to accumulate enough messages to fill a chunk.  If, after the timeout,
there are no messages, the consumer shrugs, and moves on to the next partition.

Ordinarily, this doesn't matter much.  However, when you get into a
pathological case -- a heavily unbalanced set of partitions, made even worse
through unbalanced message sizes ... you can get into a case where the
round-robin blocking is _actively harmful_ ... we had 10 completely up to date
topics, which would _block_ while waiting for any new messages to come in,
before finally moving back to the choked up topic, which would only yield a 
handful of messages because of block size limits.  When we added partitions to
try to force a redistribution, we actually made things _worse_ by inserting
additional blocking calls into the round-robin cycle.

## Lessons?

Ultimately, it took four of us four days to crack this one open, and make it
all the way down to root cause.  A lot of that is because chasing down an
intersection of small configuration differences can be difficult, generally, 
and made more difficult when not everyone is familiar with the system in
question.  This one involved a lot more kafka esoterica than I had prepared
anyone to deal with ... but, there are now several more people with a
functioning knowledge of the guts of our kafka infrastructure, so that's a
good thing.

We're still backfilling some additional instrumentation, but our graphite stats,
and splunk logs were invaluable tools for getting into the running systems and
being able to see what was going on.  A little bit of foresight can save you
from a _lot_ of blind alleys when the panic sets in.

My read of the causes of the situation is largely that the old configuration,
where we "had no problems" was doing an effective job of masking a problem that
we suffered at a deeper level.  The Real Problem was the unbalanced partitions
and the large message size.  There is little to be done about the message size;
in this case, it's inherent to the datagrams we're pushing. 

The sheer number of repeat messages, however, can and should be addressed.
Relying on after-save hooks allows you to be remarkably un-intentional.
Sure, this guarantees that you will never forget to synchronize your changes ...
but it also encourages carelessness.  I am of the firm opinion that you
should be mindful of when and how you talk to external systems[^mindful], and
tying that conversation into a database lifecycle hook is the _opposite_ of
mindful.  We got bit.

[^mindful]: This includes your database, by the way.  The cron job described
  above is an all-too-common example of the cavalier way we treat the `save`
  methods.
