+++
author = "Vladimir Lukiyanov"
title = "Exploring Redis streams with Akka streams"
date = "2020-11-27"
lastmod = "2020-12-01"
description = "A short introduction to consuming and producing messages to Redis streams using Akka streams"
tags = [
    "akka", "redis", "streams", "scala", "exploration"
]
+++

Redis is an in-memory database and Akka streams is a popular reactive streams implementation for Scala. In this post we explore a simple end-to-end toy example to demonstrate using Redis streams in Akka streams via the popular Java library [Lettuce](https://github.com/lettuce-io/lettuce-core). The repo [vlukiyanov/akka-redis-streams-example](https://github.com/vlukiyanov/akka-redis-streams-example) contains the code for the example references below.

The Redis streams data structure implements a persistent log with support for consumer groups, and is designed to fan out messages to multiple consumers. There is a [good but lengthy introduction to Redis streams on the Redis website](https://redis.io/topics/streams-intro) though roughly speaking there are three key operations:

* **Adding messages to the stream** using the [XADD](https://redis.io/commands/xadd) operation. Each message has a unique ID, which is usually generated automatically by the Redis server and returned to the caller.
* **Reading messages from the stream** using [XREAD](https://redis.io/commands/xread) and [XREADGROUP](https://redis.io/commands/xreadgroup) operations. There are various modes, from just reading a given message by ID, to reading message using consumer groups.
* **Dealing with consumer groups, consumers, and acknowledging messages** using operations like [XACK](https://redis.io/commands/xack), [XGROUP](https://redis.io/commands/xgroup), and [XPENDING](https://redis.io/commands/xgroup). Consumer groups ensure guarantees around delivery, for example the ability to inspect messages that were consumed but not acknowledged by consumers - properties that are essential for building reliable systems. 
  
Let's start with a rough outline of our simple app:

{{<center>}}
{{<mermaid>}}
graph LR;
    A(Source of messages) --> B(XADD) --> C(Sink ignore)
{{< /mermaid >}}
{{</center>}}

{{<center>}}
{{<mermaid>}}
graph LR;
    E(Source of XREADGROUP) --> F(Sink XACK)
{{< /mermaid >}}
{{</center>}}

We will optimise this app for throughput. To do this we imitate multiple simultaneous consumers and producers, in particular using parallelism with `.mapAsync(4)` to saturate Redis. These optimisations are unlikely to be relevant in production. For production, for example for Akka HTTP projects, note that Lettuce also has a [reactive interface](https://github.com/lettuce-io/lettuce-core/wiki/Reactive-API-(5.0)) using [Project Reactor](http://projectreactor.io/). This interface can be used within Akka Streams via the [reactive streams interop](https://doc.akka.io/docs/akka/current/stream/reactive-streams-interop.html).

# Setting up

To start with we need a local Redis server. The easiest way is using Docker, where some variation of `docker run --name redis-local -p 6379:6379 -d redis` will start a local server on port `6379`; to change the port number to e.g. `6380` use `-p 6380:6379`.

# Producing messages

To producer will take messages we have defined locally in our code and publish them to Redis streams. Each instantiation publishes a `Map[String, String]` to a given stream using the [XADD](https://redis.io/commands/xadd) operation. This will then add the input to the stream with an ID based on the timestamp, which is then returned; the process can be modelled as a `Flow[Map[String, String], String, NotUsed]`. A simple implementation:

```scala
object RedisStreamsFlow {
  def create(redis: RedisCommands[String, String],
             stream: String): Flow[Map[String, String], String, NotUsed] = {
    Flow
      .apply[Map[String, String]]
      .map(elem => redis.xadd(stream, elem.asJava))
  }
}
```

This is simple but not always performant, a more performant version can be implemented using pipelining. To do this compress the `Flow` using the `groupedWithin` method, send the groups in `Seq[Map[String, String]]` chunks, creating `Seq[String]`, and finally apply `.mapConcat(identity)` to flatten the results into instances of `String`.

```scala
object RedisStreamsFlow {
  def create(redis: RedisAsyncCommands[String, String],
             stream: String): Flow[Map[String, String], String, NotUsed] = {
    redis.setAutoFlushCommands(false)
    Flow
      .apply[Map[String, String]]
      .groupedWithin(1000, 1.second)
      .mapAsync(4) { elems =>
        {
          val futures =
            elems.map(elem => redis.xadd(stream, elem.asJava).asScala)
          redis.flushCommands()
          Future.sequence(futures)
        }
      }
      .mapConcat(identity)
  }
}
```

In the above `redis.setAutoFlushCommands(false)` disables Lettuce's automatic flushing, as discussed in [Lettuce documentation](https://lettuce.io/core/release/reference/#_pipelining_and_command_flushing), and instead pipelines the commands manually. Periodically `redis.flushCommands()` invokes writing to the transport after many commands have been added - increasing throughput. 

The complete code for this is in [`RedisStreamsFlow.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/api/RedisStreamsFlow.scala) and [`ProducerExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/ProducerExample.scala).

# Consuming messages

There are many ways to consume messages using Redis streams. To make the most of the capabilities that stream offers we need to use consumer groups. Consumer groups are discussed in the [introduction on the Redis website](https://redis.io/topics/streams-intro). Lettuce's [XREADGROUP](https://redis.io/commands/xreadgroup) implementation takes a consumer group, consumer name, an offset configuration for the reading the stream, and returns a list of `io.lettuce.core.StreamMessage`; a `StreamMessage` stores a string `id` and a `Map[String, String]` representing the `body` of the message consumed. We can model the consumer as a `Source[StreamMessage[String, String], Cancellable]`. An simple implementation:

```scala
object RedisStreamsSource {
  def create(
      redis: RedisAsyncCommands[String, String],
      stream: String,
      group: String,
      consumer: String): Source[StreamMessage[String, String], Cancellable] = {
    Source
      .tick(0 millisecond, 100 millisecond, Done)
      .mapAsync(4) { _ =>
        redis
          .xreadgroup(
            Consumer.from(group, consumer),
            XReadArgs.Builder.count(100000),
            XReadArgs.StreamOffset.lastConsumed(stream)
          )
          .asScala
      }
      .mapConcat(_.asScala.toList)
  }
}
```

This will poll Redis at some fixed interval, then flatten the lists of messages using `.mapConcat(_.asScala.toList)`. The complete code for this is in [`RedisStreamsSource.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/api/RedisStreamsSource.scala) and [`ConsumerExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/ConsumerExample.scala).

# Acknowledging messages

When a consumer in a consumer group has successfully processed a message it should indicate this by sending an [XACK](https://redis.io/commands/xack); sending [XACK](https://redis.io/commands/xack) removes the message from the pending list. As a message is uniquely determined by a string id this process can be modelled as a `Sink[String, NotUsed]`. A simple implementation:

```scala
object RedisStreamsAckSink {
  def create(redis: RedisAsyncCommands[String, String],
             group: String,
             stream: String): Sink[String, NotUsed] = {
    Flow
      .apply[String]
      .mapAsync(4) { messageId =>
        redis.xack(stream, group, messageId).asScala
      }
      .to(Sink.ignore)
  }
}
```

# Threading it all together

We're now able to create a full example which simultaneously produces and consumes messages, afterwards acknowledging them. Start with the producer, this includes a snippet from https://stackoverflow.com/a/49279641 for rate measurement:

```scala
val redisStreamsFlow = RedisStreamsFlow.create(asyncCommands, "testStream")
val messageSource = Source
   .repeat(Map("key" -> "test"))
   .limit(10_000_000)

messageSource
  .via(redisStreamsFlow)
  .conflateWithSeed(_ => 0) { case (acc, _) => acc + 1 }
  .zip(Source.tick(1.second, 1.second, NotUsed))
  .map(_._1)
  .toMat(Sink.foreach(i => println(s"$i elements/second producer")))(Keep.right)
  .run()
```

Next consume the produced messages, acknowledging them as we go along:

```scala
val redisStreamsSource = RedisStreamsSource.create(asyncCommands,
  "testStream",
  "testGroup",
  "testConsumer")

val redisStreamAckSink = RedisStreamsAckSink.create(asyncCommands, "testGroup", "testStream")

redisStreamsSource
  .map(_.getId)
  .to(redisStreamAckSink)
  .run()
```

Of course a more useful example would actually do something with the messages before acknowledging them. The complete code for this is in [`AckExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/AckExample.scala).

Running this code we eventually see that all the message have been acknowledged, this can be verified by running [XPENDING](https://redis.io/commands/xpending):

```
127.0.0.1:6379> XPENDING testStream testGroup
1) (integer) 0
2) (nil)
3) (nil)
4) (nil)
```

The above means our code was successful - we have the example app. This can be used as a springboard for exploring other features in Redis streams, and possibly applications for this in-memory log data structure (though note Lettuce's [reactive interface](https://github.com/lettuce-io/lettuce-core/wiki/Reactive-API-(5.0)) - this will be more suitable for production).