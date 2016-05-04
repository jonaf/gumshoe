![build status](https://api.travis-ci.org/lwoodson/gumshoe.svg?branch=master)

# GumShoe
A "gumshoe" is an old slang term for private detective or investigator.  Gumshoe
the library is a mechanism for emitting arbitrary events from code for
auditing and analysis.  Searching these events, you can see exactly what is
happening in your code in production.  GumShoe provides the interface for
emitting these events in a JSON format, what consumes them and places them
somewhere for auditing and analysis is left to other tools.

## Events
Events are an occurence of something in your code.  They have a name, a
timestamp and arbitrary data that is pertinent to your application and the
specific event.  GumShoe puts no restrictions on you here.  The events will be
converted to JSON before they are emitted.

## Contexts
Contexts have a name and store data that is included in events.  While a context
is established, any data within the context is included in all events emitted
while the context is active.  For instance, your context might include a
userName field.  All events emitted while the context is active will include
the userName field populated with the value in the context.

Contexts are stackable, with events emitted during a context being reflective
of all of the data within the stack.  For instance, if you had a customer
and wanted to process all of their items, you might start a context for
the customer, and then push a context for each of the items.  Events emitted
will include the customer data from the customer context and item data for
each of the items.

The first time a context is established, a stream UUID is given or created and
stored with the context.  All subsequent events will include this stream
UUID.  You can thus find one event, take its stream UUID, search again on
the UUID and see all related events in the stream.

Start and stop events are automatically emitted when you enter/leave a
context.  The time in milliseconds between these is tracked and included in the
stopped event as execution_time.

Contexts are thread local, and thus will play nice with multi-threaded
applications.

## DSL
A DSL is provided for easily using GumShoe.  This varies slightly from
language to language, but in general its use is as follows:

```python
customer = get_customer()
context("customer_processing").put("customer", customer.id).start()

for items in items_of(client):
  context("item_processing").put("item", item.id).start()
  # do something
  emit("did something")
  # do something else
  emit("did something else")
  context().stop()

context("exporting_file").start()
# build file
# push file
context().stopAll()
```

If the customer 1 had two items with ids 1 and 2, this would result in 12 events
as follows:

```
{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing"],
  "type": "started",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "started",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "did something",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "did something else",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "finished",
  "timestamp": "2011-12-03T10:15:30Z",
  "execution_time": 100,
  "customer": 1,
  "item": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "started",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 2
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "did something",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 2
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "did something else",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1,
  "item": 2
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "item_processing"],
  "type": "finished",
  "timestamp": "2011-12-03T10:15:30Z",
  "execution_time": 100,
  "customer": 1,
  "item": 2
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "exporting_file"],
  "type": "started",
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing", "exporting_file"],
  "type": "finished",
  "execution_time": 200,
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1
}

{
  "stream_id": "123e4567-e89b-12d3-a456-426655440000",
  "context": ["customer_processing"],
  "type": "finished",
  "execution_time": 400,
  "timestamp": "2011-12-03T10:15:30Z",
  "customer": 1
}
```

If using a file logger, these will be emitted one object per line.  These
examples in the logged format can be found [here](example.json).

## Decorators
Decorators are used to decorate all events within an application with data
before they are emitted.  This can be used to add environment information,
such as the hostname where the application is running that emitted the
event.  The default decorator adds the hostname, thread name and process
user to all events.  You can define your own decorator and plug it into
GumShoe to suit your needs.

## Emitters
Emitters are what broadcast events.  There are three offered out of the box:
a standard out emitter, a dev null emitter and a log file emitter.  The
log file emitter is useful to analyze the audit events using a log aggregation
tool like the elasticsearch, logstash and kibana stack.  You can write your
own emitter and plug it into GumShoe.

## Filters
Filters are used to filter events.  If they evaluate to false, the event
will not be emitted.  By default, no filters are in place, but you can
define your own and add them to GumShoe for an application.  This is
useful if you want to throttle the number of events emitted or for sampling.

## Configuration
To override the out of the box defaults for decorators, emitters and filters,
a configuration must be provided to GumShoe.

## Implementations
There are two implementations currently provided --
[Java](java/README.md) and [Python](python/README.md).
