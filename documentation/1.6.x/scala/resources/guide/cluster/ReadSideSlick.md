# Slick Read-Side support

This page is specifically about Lagom's support for relational database read-sides using Slick.  Before reading this, you should familiarize yourself with Lagom's general [[read-side support|ReadSide]] and [[relational database read-side support overview|ReadSideRDBMS]].


## Configuration

Slick support builds on top of Lagom's support for [[storing persistent entities in a relational database|PersistentEntityRDBMS]]. See that guide for instructions on configuring Lagom to use the correct JDBC driver and database URL.

Next we need to configure the Slick mappings for the read-side model. Note that this example is using the `slick.jdbc.H2Profile`. Make sure you import your profile of choice instead.

@[slick-mapping-initial](code/docs/home/scaladsl/persistence/SlickRepos.scala)

## Query the Read-Side Database

Let us first look at how a service implementation can retrieve data from a relational database using Slick.

@[imports](code/docs/home/scaladsl/persistence/SlickReadSideQuery.scala)
@[service-impl](code/docs/home/scaladsl/persistence/SlickReadSideQuery.scala)


Note that a Slick [Database](https://scala-slick.org/doc/3.2.1/api/#slick.jdbc.JdbcBackend$DatabaseDef) is injected in the constructor together with the previously defined `PostSummaryRepository`. Slick's [Database](https://scala-slick.org/doc/3.2.1/api/#slick.jdbc.JdbcBackend$DatabaseDef) allows the execution of the Slick `DBIOAciton` returned by `selectPostSummaries()`. Importantly, it also manages execution of the blocking JDBC calls in a thread pool designed to handle it, which is why it returns a `Future`.

## Update the Read-Side

We need to transform the events generated by the [[Persistent Entities|PersistentEntity]] into database tables that can be queried as illustrated in the previous section. For that we will implement a [ReadSideProcessor](api/index.html#com/lightbend/lagom/scaladsl/persistence/ReadSideProcessor) with assistance from the [SlickReadSide](api/index.html#com/lightbend/lagom/scaladsl/persistence/slick/SlickReadSide) support component. It will consume events produced by persistent entities and update one or more database tables that are optimized for queries.

This is how a [ReadSideProcessor](api/index.html#com/lightbend/lagom/scaladsl/persistence/ReadSideProcessor) class looks like before filling in the implementation details:

@[imports](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)
@[initial](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

You can see that we have injected the Slick read-side support, this will be needed later.

You should already have implemented tagging for your events as described in the [[Read-Side documentation|ReadSide]], so first we'll implement the `aggregateTags` method in our read-side processor stub, like so:

@[tag](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

### Building the read-side handler

The other method on the `ReadSideProcessor` is `buildHandler`.  This is responsible for creating the [ReadSideHandler](api/index.html#com/lightbend/lagom/scaladsl/persistence/ReadSideProcessor.ReadSideHandler) that will handle events.  It also gives the opportunity to run two callbacks, one is a global prepare callback, the other is a regular prepare callback.

[SlickReadSide](api/index.html#com/lightbend/lagom/scaladsl/persistence/jdbc/SlickReadSide) has a `builder` method for creating a builder for these handlers, this builder will create a handler that will automatically manage transactions and handle read-side offsets for you.  It can be created like so:

@[create-builder](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

The argument passed to this method is an identifier for the read-side processor that Lagom should use when it persists the offset. Lagom will store the offsets in a table that it will automatically create itself if it doesn't exist. If you would prefer that Lagom didn't automatically create this table for you, you can turn off this feature by setting `lagom.persistence.jdbc.create-tables.auto=false` in `application.conf`. The DDL for the schema for this table is as follows:

```sql
CREATE TABLE read_side_offsets (
  read_side_id VARCHAR(255), tag VARCHAR(255),
  sequence_offset bigint, time_uuid_offset char(36),
  PRIMARY KEY (read_side_id, tag)
)
```

### Global prepare

The global prepare callback runs at least once across the whole cluster.  It is intended for doing things like creating tables and preparing any data that needs to be available before read side processing starts.  Read side processors may be sharded across many nodes, and so tasks like creating tables should usually only be done from one node.

The global prepare callback is run from an Akka cluster singleton.  It may be run multiple times - every time a new node becomes the new singleton, the callback will be run.  Consequently, the task must be idempotent.  If it fails, it will be run again using an exponential backoff, and the read side processing of the whole cluster will not start until it has run successfully.

Of course, setting a global prepare callback is completely optional, you may prefer to manage database tables manually, but it is very convenient for development and test environments to use this callback to create them for you.

Below is an example method that we've implemented to create tables using Slick DDL generation. Here Slick support for DDL statements is used to create the table only if it does not exists, so that the operation can be idempotent as explained before.

@[slick-mapping-schema](code/docs/home/scaladsl/persistence/SlickRepos.scala)

The best place to define such a method is in your `Model Repository` where we usually add all code related to database operations.

It can then be registered as the global prepare callback in the `buildHandler` method:

@[register-global-prepare](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

### Prepare

In addition to the global prepare callback, there is also a prepare callback that can be specified by calling [builder.setPrepare](api/index.html#com.lightbend.lagom.scaladsl.persistence.slick.SlickReadSide$ReadSideHandlerBuilder@setPrepare). This will be executed once per shard, when the read side processor starts up.

If you read the [[Cassandra read-side support|ReadSideCassandra]] guide, you may have seen this used to prepare database statements for later use. JDBC `PreparedStatement` instances, however, are not guaranteed to be thread-safe, so the prepare callback should not be used for this purpose with relational databases.

Again this callback is optional, and in our example we have no need for a prepare callback, so none is specified.

### Registering your read-side processor

Once you've created your read-side processor, you need to register it with Lagom. This is done using the [`ReadSide`](api/index.html#com/lightbend/lagom/scaladsl/persistence/ReadSide) component:

@[register-event-processor](code/docs/home/scaladsl/persistence/BlogServiceImpl3.scala)

Note that if you are utilizing [[Macwire for dependency injection|DependencyInjection#Wiring-together-a-Lagom-application]], you can simply add the following to your Application Loader:

@[register-event-processor](code/docs/home/scaladsl/persistence/SlickBlogApplicationLoader.scala)

### Event handlers

The event handlers take an event and returns a Slick `DBIOAction`.

Here's an example callback for handling the `PostAdded` event:

@[insert-or-update](code/docs/home/scaladsl/persistence/SlickRepos.scala)
@[post-added](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

This can then be registered with the builder using `setEventHandler`:

@[set-event-handler](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

Once you have finished registering all your event handlers, you can invoke the `build` method and return the built handler:

@[build](code/docs/home/scaladsl/persistence/SlickBlogEventProcessor.scala)

## Application Loader

The Lagom Application loader needs to to be configured for Slick persistence. This can be done by mixing in the `SlickPersistentComponents` class like so:

@[load-components](code/docs/home/scaladsl/persistence/SlickBlogApplicationLoader.scala)