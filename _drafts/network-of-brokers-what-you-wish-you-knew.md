---
layout: post
title: 'ActiveMQ Network of Brokers: What you wish you knew, Part 1'
author: Oscar Smith-Sieger
tags:
- NoB
- Network of Brokers
- ActiveMQ
- Clustering
- What you wish you knew 
---

## Introduction

[Apache ActiveMQ](https://activemq.apache.org/components/classic/) is an [open-source](https://github.com/apache/activemq) message broker written in Java, with a poorly-understood horizontal scaling mechanism called [Network of Brokers](https://activemq.apache.org/components/classic/documentation/networks-of-brokers) (NoB). Hopefully this will answer everything you wish you knew about NoB!

There’s quite a lot you can do running a single ActiveMQ broker, but it has its limits. Vertical scaling only gets us so far, which is when we should reach for horizontal scaling. The native mechanism ActiveMQ provides for this is Network of Brokers (NoB).

When people think of “horizontal scaling” they have a lot of different ideas, namely about what exactly is being scaled. Some people want availability, some durability, some want storage, you name it. There’s even different names people use, like “clustering”. Sometimes these mean the same things, sometimes not. The approach ActiveMQ takes with NoB is different than what most people expect. Despite this, NoB is actually quite simple and analogous to something many already understand. Keep an open mind and you should be fine!

## What Network of Brokers is

Network of Brokers (NoB) implements a [store-and-forward](https://en.wikipedia.org/wiki/Store_and_forward) network. The name comes from the fact that messages are stored by a single node, then forwarded to some destination. You may be familiar with this concept if you know about IP networking, specifically packet-switched networks. If you are, try to apply the same general mental model.

To build on the “store-and-forward” framework, we say a broker “stores” a message when it’s held in the broker’s storage. Then to “forward” is when a broker dispatches a message to a consumer. If this seems a bit simple… that’s because it is!

But maybe that’s a confusing mental model? Here’s another one: we can think of the broker as consuming messages (”storing”) from external producers, while producing messages (”forwarding”) to external consumers. In this sense, it’s not “store-and-forward”, rather it’s “consume-and-produce”. This is the “broker-centric” model, where you can think of an external client as a limited broker (one that typically only produces or only consumes). 

In either model, we can think of a single ActiveMQ broker as an NoB with a single node. Adding nodes provides new paths for a message to flow through, but from a single broker’s perspective they aren’t much different from external producers/consumers. NoB connections themselves are one-way: a source broker forwards (produces) messages to a destination (consuming) broker. To allow two brokers to freely pass messages back and forth (a “duplex” connection), simply create two one-way connections in opposite directions. With a duplex connection brokers can logically produce/consume from each other. ActiveMQ actually provides a convenience to create duplex connections automatically, but it’s still just two one-way connections under the hood.

Without external producers/consumers (”clients”) the NoB network will lie inert and do nothing. It’s only when clients try to consume does it create the need for brokers to produce messages along the network. This works via the “demand forwarding” mechanism. When a client in the network wants to consume a message, it creates “demand” in the network. This demand propagates through the network, inducing nodes in the network to dispatch messages to fulfil this demand. Generally speaking, demand will be fulfilled from the broker the client is connected to first. This means there won’t be NoB demand sent if there are local messages available. But in some broker configurations, things may be more complicated. Make sure to test your networks!

## What Network of Brokers isn’t

NoB is **probably not** what you expect when you hear “clustering”. At one point NoB would’ve been called “clustering”, but it’s not what people usually expect these days. Oracle docs on JMS (from 2010) refer to NoB as a [“conventional broker cluster”](https://docs.oracle.com/cd/E19340-01/820-6424/ggsbb/index.html). ActiveMQ documentation does refer to NoB networks as “clusters” in a few places, but isn’t very consistent. In ActiveMQ’s context, “cluster” simply means a set of brokers that interoperate in some way. No further guarantees or expectations can be placed upon a cluster (such as availability, durability, or reliability). It’s all dependent on how it’s configured.

NoB provides no increase to “data availability”. There’s no replication or such to ensure data isn’t on a single node. It’s possible to build abstractions on top of NoB for this, but it’s not provided by ActiveMQ itself. Since NoB is a store-and-forward network, messages only belong to a single node at once. If a single node holds all the messages in your network and that node goes down, your network won’t have any messages left. If that's a concern you can improve the availability of singular nodes to mitigate the issue. But at large enough scale you’ll always want a way to mitigate node failure. Usually this is down to architectural decisions, rather than in-broker solutions.

Due to the demand-forwarding mechanism, NoB doesn’t “load-balance” messages across the storage of the network. If all your producers are producing onto one node and nothing is consuming in the network, the producing node won’t send messages down the network if it starts to get full. Instead, depending on broker configuration, it’ll start to reject messages. Even if the network has 1TB of storage, if the entry node has only 200gb and there’s no demand, you’ve only really got 200gb of storage! It is possible to configure a network such that it will forward messages down the line, but that’s a more complicated story.

## What Network of Brokers can solve

Here’s the hard truth: depending on your system architecture, NoB might provide you nothing. But NoB is the only solution (within ActiveMQ) for true scaling, meaning any system that must handle substantial load should be architected with NoB in mind. It’s also important to step back and see if the scaling can be solved outside ActiveMQ, at another layer of the system. While ActiveMQ can serve in your hot path, it requires the knowledge and understanding that not all are willing to commit. 

NoB can trivially provide the typical “mesh” architecture, where N nodes are all connected to each other. The effect is a client can connect to any node in the network and be able to produce/consume to any other client in the network. This provides an intrinsic benefit of overall availability: if any one node goes down the clients can connect to any other node. But depending on how the clients pick the node to connect to, this can also provide a form of load balancing by spreading work across the network. Creating a duplex connection between each broker in the network is the most straightforward approach for implementation.

But really, the world is your oyster: you can build any network topology you desire with NoB. Fundamentally you can’t overcome the store-and-forward nature: a given message’s availability is never greater than a single nodes. You could build abstractions to duplicate messages if you wanted, but the effort may outweigh the benefit. Understanding the nature of NoB, it’s benefits, and disadvantages, is more important. It’s a tool in your toolbox like any other and not necessarily the best one for every problem. When in doubt, test things out!

## Conclusion

TODO

### Appendix 1: Miscellaneous documentation

- https://activemq.apache.org/components/classic/documentation/networks-of-brokers
- https://activemq.apache.org/components/classic/documentation/how-do-distributed-queues-work
- https://activemq.apache.org/components/classic/documentation/networks-of-brokers#networks-of-brokers-and-advisories