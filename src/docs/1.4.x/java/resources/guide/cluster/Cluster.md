# Cluster

Instances of the same service may run on multiple nodes, for scalability and redundancy. Nodes may be physical or virtual machines, grouped in a cluster.

The underlying clustering technology is [Akka Cluster](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java).

If instances of a service need to know about each other, they must join the same cluster. Within a cluster, services may use the [[Persistence|PersistentEntity]] and [[Publish-Subscribe|PubSub]] modules of Lagom.

## Dependency

The clustering feature is already included if you are using either of the [[persistence|PersistentEntity]] or [[pubsub|PubSub#Dependency]] modules.

If you want to enable it without those modules, add the following dependency your project's build.

In Maven:

```xml
<dependency>
    <groupId>com.lightbend.lagom</groupId>
    <artifactId>lagom-javadsl-cluster_${scala.binary.version}</artifactId>
    <version>${lagom.version}</version>
</dependency>
```

In sbt:

@[cluster-dependency](code/build-cluster.sbt)

## Cluster composition

A cluster should only span nodes that are running the same service.

You could imagine using cluster features across different services, but we recommend against that, because it would couple the services too tightly. Different services should only interact with each other through each service's API.

## Joining

A service instance joins a cluster when the service starts up. In development you are typically only running the service on one cluster node. No explicit joining is necessary; the [[Lagom Development Environment|DevEnvironment]] handles it automatically. In production, you need to implement the joining yourself as follows.

First, define some initial contact points of the cluster, so-called seed nodes. You can define seed nodes in `application.conf`:

    akka.cluster.seed-nodes = [
      "akka.tcp://MyService@host1:2552",
      "akka.tcp://MyService@host2:2552"]

Alternatively, this can be defined as Java system properties when starting the JVM:

    -Dakka.cluster.seed-nodes.0=akka.tcp://MyService@host1:2552
    -Dakka.cluster.seed-nodes.1=akka.tcp://MyService@host2:2552

The node that is configured first in the list of `seed-nodes` is special. Only that node that will join itself. It is used for bootstrapping the cluster.

The reason for the special first seed node is to avoid forming separated islands when starting from an empty cluster. If the first seed node is restarted and there is an existing cluster it will try to join the other seed nodes, i.e. it will join the existing cluster.

You can read more about cluster joining in the [Akka documentation](https://doc.akka.io/docs/akka/2.5/cluster-usage.html?language=java#joining-to-seed-nodes).

## Downing

When operating a Lagom service cluster you must consider how to handle network partitions (a.k.a. split brain scenarios) and machine crashes (including JVM and hardware failures). This is crucial for correct behavior when using [[Persistent Entities|PersistentEntity]]. Persistent entities must be single-writers, i.e. there must only be one active entity with a given entity identity. If the cluster is split in two halves and the wrong downing strategy is used there will be active entities with the same identifiers in both clusters, writing to the same database. That will result in corrupt data.

The naïve approach is to remove an unreachable node from the cluster membership after a timeout. This works great for crashes and short transient network partitions, but not for long network partitions. Both sides of the network partition will see the other side as unreachable and after a while remove it from its cluster membership. Since this happens on both sides the result is that two separate disconnected clusters have been created. This approach is provided by the opt-in (off by default) auto-down feature in the OSS version of Akka Cluster. Because of this auto-down should not be used in production systems.

**We strongly recommend against using the auto-down feature of Akka Cluster.**

A pre-packaged solution for the downing problem is provided by [Split Brain Resolver](https://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html), which is part of the [Lightbend Platform](https://www.lightbend.com/lightbend-platform). The `keep-majority` strategy is configured to be enabled by default if you use Lagom with the Split Brain Resolver.

See [[Using Lightbend Platform with Lagom|LightbendPlatform]] and the [Split Brain Resolver documentation](https://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html) for instructions on how to enable it in the build of your project.

Even if you don't use the commercial Lightbend Platform, you should still read & understand the concepts behind [Split Brain Resolver](https://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html) to ensure that your solution handles the concerns described there.
