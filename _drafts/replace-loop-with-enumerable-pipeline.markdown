---
layout: post
title: "Replace Loop With Enumerable Pipeline"
date: 2016-01-04T14:52:16-05:00
categories: ruby
---

I wanted to talk about a refactoring I've found useful a couple of times over
the course of the last year.  I have, at various points, found myself
needing to transform JSON data from a remote endpoint.  Sometimes, the remote
takes the form of a messaging system; other times it's a paginated REST
endpoint.  From a processing perspective, both behave the same:  I get a chunk
of raw data to transform, I process each record, and then request the next
batch.

This is frequently done procedurally, from inside an each loop.  I have found
it better to break apart the steps of that procedure, and build up a number of
`Enumerator`s and enumerable filters, to achieve that result.  I find this
style to be easier to understand, as it leans more strongly on Ruby's idoms,
while providing better encapsulation.

<!-- more -->

## An Example

As an example, the process of fetching and processing a batch of `kafka`
messages may look something like this:

```ruby
class Processor
  def call(client)
    client.fetch.each do |messages|
      messages.each do |raw|
        data    = JSON.parse(raw)
        message = build_message_from(data)

        next unless valid?(message)
        process(message.body)
      end
    end
  end
end

client    = initialize_client
processor = Processor.new

loop { processor.call(client) }
```

The primary ruby kafka client, [Poseidon][], fetches messages in a chunk;
each call to fetch will return an array of raw message data. (line 3,4)

Because this is shared data over a network, each message comes back in the
form of a JSON encoded string[^1] which we need to decode, (line 5).

Since messages may be coming from any source, we want to
validate the message, to ensure that it's in our expected formats. (line 8)

Potentially, we might also want to skip over messages that represent events we
don't care about, which would be an extra step between lines 8 and 9.

At last, we perform the actual processing on the message received (line 9).

To keep this going through more than one batch, we run the whole thing inside
an infinite loop, so that each chunk of messages gets processed[^2].

[poseidon]: http://github.com/bpot/poseidon

[^1]: This does not _have_ to be true; it's possible to send more-or-less any
      payload in the body of a kafka message. We find it wiser to stick with
      loosely typed encodings, rather than binary represenatations, however.

[^2]: `client.fetch` is a blocking call, and only returns when there are
      actual messages to process.

### So ... what's wrong with that?


Assuming the helper methods are defined, the above code _works_, and it's a
reasonable implementation.  However, I find it to be awkward to test.  The
input to the process message is an object responding to `fetch` and returning
an array of strings.  Easy enough to double in our tests, but the is a lot to
have to check for:

  * what happens when fetch raises an error?
  * what happens when a message is invalid JSON?
  * what happens when the message cannot be decoded into our internal message
    object representation?
  * what consititudes a valid or invalid message -- we have to check those
    various boundries.
  * If we _are_ filtering to specific message structures, we need to check
    that the filtering works as expected.
  * Failures in any of the above steps should probably _not_ prevent
    evaluating the next message.
  * lastly, we need to check how our valid input behaves within the business
    logic.
  * Failures in processing may end up handled differently, depending on the
    nature of the failure.

Many of those are exceptional conditions, and we need to handle the exceptions
within the processing loop.  There are a _number_ of different cases we need
to catch, and each one gets handled differently -- if only from a logging and
reporting perspective.  All of those circumstances need to be manufactured as
a JSON-string, and fed into the method as a double-response.

Because it is a sequence of method calls, exceptions need to be handled from
within the `each` loop (line 3) so that the we may stop processing the broken
message, while still handling the rest of the messages.  The interior loop ends up with
a very high ratio of exception-handler-to-code, concealing the logic.

All of that _before_ we start having to manufuacture different inputs to
excercise the business logic of the process method:

```ruby
def call(client)
  client.fetch.each do |messages|
    messages.each do |raw|
      begin
        data    = JSON.parse(raw)
        message = build_message_from(data)

        next unless valid?(message)
        process(message.body)
      rescue InvalidJSONError => e
        # what to do
      rescue MessageConstructionError => e
        # ...
      rescue ProcessCollaboratorOffline => e
        # ...
      rescue SomeOtherProcessFailure => e
        # ....
      end
    end
  end
end
```


These are all classic signs of a violation of Single Responsibility.  We have
a method that is responsible for fetching, decoding, filtering and processing
each indvidual message.

We could try testing the helper methods individually, triggering `#sends` for
those private functions, but that skips over our exception handling behavior.
It may _work_ for iterating over inputs to the different messages, but our
tests are yelling at us.

## A less coupled way

If we've started looking at testing each individual helper method, we've
already started grasping towards a realization -- each step of this procedure
is an essentially discreet step in the chain.  Inputs are complicated because
they are handled sequentially, within the same block, and share a _lot_ of
knowledge about the rest of the steps.


### Simplify the input

One thing that complicated the tests above was that all of the inputs were to
be structured as a test double, responding to the `fetch` method, and
returning an array of strings.  Let's unwrap that:

```ruby
class Processor
  def call(messages)
    messages.each do |raw|
      # ...
    end
  end
end

client    = initialize_client
processor = Processor.new

loop { processor.call(client.fetch) }
```

In one sense, we've made the `Processor` dumber -- it no longer explicitly
knows how to interact with our message source.  On the other hand, by making
it _dumber_, we've made it more _flexible_.  We've gone from a very fixed
protcol to something that is much more widely supported.  Our tests can go
from using `processor.call(double(fetch: input_array))` to  just
`processor.call(input_array)`

### Map and reject

Having the hint that input takes an `Enumerable`, we note that each step of
the processing loop is either feeding into the next step, or skipping
forward:

```ruby
class Processor
  def call(messages)
    messages.to_enum.lazy
      .map{|raw| JSON.parse(raw) }
      .map{|data| build_message_from(data) }
      .select{|message| valid?(message) }
      .each{|message| process(message) }
  end
end
```

`to_enum.lazy` converts the input to a lazy enumerator, which will yield a
single record at a time, sending each record completely through the chain
before starting the next.

This _looks_ better, but that's because I've left out the exception handling
again.  We're still saddled with the need for a large sequence of rescue
blocks -- and, worse, there's actually a regression here.  If we add the
rescue block to the end of the `call` method, the first exception will
terminate the sequence, rather than moving to the next record.

Actually getting the exceptions together might look more like this:

```ruby
class Processor
  def call(messages)

    from_json = ->(raw) do
      JSON.parse(raw)
    rescue InvalidJSONError => e
       # ...
    end

    build_message = -> (data) do
      build_message_from(data)
    rescue MessageConstructionError => e
      # ...
    end

    messages.to_enum.lazy
      .map(from_json).compact
      .map(build_message).compact
      .select{|m| valid? m }
      .each{|m| process m }
  end
end
```

I've moved the process exception handling into the process method, because
that's really where it belongs.  That doesn't really help this _not_ look like
a mess, but this excercise does help bundle the exceptions with their source.
I find the required `compact` to be a huge red flag however.

### Enter Enumerator

The reason that the `compact` is required above is that an entry that triggers
an exception will show up in the `map` stream as a `nil`.  After handling that
exception, what we really want to do is skip over it so that the rest of the
sequence doesn't have to deal with it.

An `Enumerator` lets us control what is yielded, and suggests we move our
rescue encapsulation out of the calling method:

```ruby
class MessageMapper
  def messages(source)
    Enumerator.new do |e|
      source.each do |raw|
        e << build_message_from JSON.parse(raw)
      rescue InvalidJSONError => e
        # ...
      rescue MessageConstructionErrror => e
        # ...
      end
    end
  end
end


class Processor
  def call(messages)
    mesages.to_enum.lazy
      .select{|m| valid? m  }
      .each{|m| process m }
  end
end

client    = initialize_client
processor = Processor.new
mapper    = MessageMapper.new

loop { processor.call mapper.messages(client.fetch) }
```

When `MessageMapper` encounters an error, the rescue blocks kick in as
before.  However, this time, the failing entry is _not_ yielded to the 
subsequent entries.  The enumerator provided by `MessageMapper#messages` will
yield _only_ valid messages.  We have effectively built a generator that takes
an Enumerable of n JSON strings, and returns an Enumerator of 0-n Messages.

I'm not going to squint hard enough to see JSON decoding and rebuilding the
message object as separate concerns, so they are combined in this mapper.
Tests for various bad-message input scenarios may be applied to the
MessageMapper alone, while business-logic and message validation tests may be
written using the _actual message objects_ rather than stringified JSON.  This
makes those tests markedly less noisy.

If the valid message filtering becomes more complicated, that too becomes a
prime candidate for conversion to a Enumerator.  Another interesting exercise
may be to build an infinite generator around the `client.fetch`, to replace
the outermost inifinite loop.
