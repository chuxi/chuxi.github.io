---
layout: post
title: Scala Actor
date: 2015-07-03
published: false
categories: Scala
---

## What is Akka?

Akka is a toolkit for developing correct concurrent, fault-tolerant and scalable application.

Akka includes two main models and implements some unique features together:

1. Actor
    - Simple and high level abstractions for concurrency and parallelism
    - Asynchronous, non-blocking and highly performant event-driven programming model.
    - Very lightweight event-driven processes (several million actors per GB of heap memory)
2. Fault Tolerance
    - Supervisor hierarchies with "let-it-crash" semantics
    - Supervisor hierarchies can span over multiple JVMs to provide truly fault-tolerant systems.
    - Excellent for writing highly fault-tolerant systems that self-heal and never stop
3. Location Transparency
4. Persistence

## Concepts

### Actor System

Actors are objects which encapsulate state and behavior, they communicate exclusively by exchanging messages which are placed into the recipient’s mailbox. And actors form a hierarchical structure which is called actor system.

So Actor System is a heavyweight structure that create and organize 1..N actors, treat as the root of all actors in an application.

Actor System has a hierarchy node of "root guardian(/)" as the top level, you can treat the whole structure as a tree. "user guardian(/user)" is the actors user created to implement application functions. Another important branch is "system guardian(/system)", it manages all system-created top-level actors, such as logging listeners. And there are three other guardian actors, "/DeadLetters", "/temp", "remote". Reference to [Top-Level Scopes for Actor Paths](http://doc.akka.io/docs/akka/snapshot/general/addressing.html#Top-Level_Scopes_for_Actor_Paths)

### Actor

An actor is a container for **State**, **Behavior**, a **Mailbox**, **Children** and a **Supervisor Strategy**. All of this is encapsulated behind an **Actor Reference**. Finally, this happens **When an Actor Terminates**.

#### Actor Reference

The Actor Model requires a actor object is shielded from the outside. Therefore, actors are represented to the outside using actor references, which are objects that can be passed around freely and without restriction.

#### State

Actor objects will typically contain some variables which reflect possible states the actor may be in. This can be an explicit state machine, or it could be a counter, set of listeners, pending requests, etc.

#### Behavior

Every time a message is processed, it is matched against the current behavior of the actor. Behavior means a function which defines the actions to be taken in reaction to the message at that point in time, say forward a request if the client is authorized, deny it otherwise. (Become, Unbecome operations)

#### Mailbox

An actor’s purpose is the processing of messages, and these messages were sent to the actor from other actors (or from outside the actor system). The piece which connects sender and receiver is the actor’s mailbox: each actor has exactly one mailbox to which all senders enqueue their messages.

#### Children

Each actor is potentially a supervisor: if it creates children for delegating sub-tasks, it will automatically supervise them. The list of children is maintained within the actor’s context and the actor has access to it.

#### Supervisor Strategy

The final piece of an actor is its strategy for handling faults of its children.

#### When an Actor Terminates

Once an actor terminates, i.e. fails in a way which is not handled by a restart, stops itself or is stopped by its supervisor, it will free up its resources, draining all remaining messages from its mailbox into the system’s “dead letter mailbox” which will forward them to the EventStream as DeadLetters.

### Supervision and Monitoring

Supervision describes a dependency relationship between actors, the supervisor delegates tasks to subordinates and therefore must respond to their failures. This is the basis of fault-tolerance.

In the other side, monitoring is a relationship between actors with no dependency. like siblings and some remote actors. Because we can not access an actor's state except whether it is alive or dead, we only could get a Terminated Message from actor which is watched.

Supervision forms a dependency hierarchy among actors. The Top level hierarchy is the same as actor system.

### Actor References and Paths

We can obtain an actor reference only by two general methods: by creating or b y looking up. Reference to [Actor References, Paths and Addresses](http://doc.akka.io/docs/akka/snapshot/general/addressing.html)

#### Creating Actors

An actor system is typically started by creating actors beneath the guardian actor using the `ActorSystem.actorOf` method and then using `ActorContext.actorOf` from within the created actors to spawn the actor tree.

#### Looking up Actors By Path

Actor references may be looked up using the `ActorSystem.actorSelection` method.

## Actors Programming Basis

```
The Actor Model provides a higher level of abstraction for writing concurrent and distributed systems. It alleviates the developer from having to deal with explicit locking and thread management, making it easier to write correct concurrent and parallel systems

```

In fact, I think the key definition of Actor Model is the words "a higher level of abstraction", which makes it different from java thread model which focus on sequential programming and shared memory access.

Developers need to extends the `Actor` base trait and implements the `receive` method. Reference to [Defining an Actor class](http://doc.akka.io/docs/akka/snapshot/scala/actors.html#Defining_an_Actor_class). And Props is a configuration class to specify creation of an actor. By Props we can create immutable actor objects.

As described, developers use Props of some Actor class to create an Actor Reference.

```scala

// ActorSystem is a heavy object: create only one per application
val system = ActorSystem("mySystem")
val myActor = system.actorOf(Props[MyActor], "myactor2")

```

or

```scala

class FirstActor extends Actor {
  val child = context.actorOf(Props[MyActor], name = "myChild")
  // plus some behavior ...
}

```

So the following question would be: How to define a MyActor?

### Actor APIs

Actor is a trait with only one abstract method: `receive`. In addition, it offers:
- self
- sender
- context
- supervisorStrategy

Besides, you can override some **hook methods** predefined in Actor trait to implement some different control function during actor lifecycle.

```scala

def preStart(): Unit = ()

def postStop(): Unit = ()

def preRestart(reason: Throwable, message: Option[Any]): Unit = {
  context.children foreach { child ⇒
    context.unwatch(child)
    context.stop(child)
  }
  postStop()
}

def postRestart(reason: Throwable): Unit = {
  preStart()
}

```

#### Lifecycle of an Actor

I find that actor lifecycle is very simple because it shields internal state from the outside and communicate by immutable messages. However, it is complicated again because developer needs to be careful with the internal actor process, which is controlled by Actor API hooks.

![](/assets/2015-07-02-01.png)

An actor is determined by path and UID, so when `actorSelection()` is called, you get the actorRef which occupied the path, maybe not the same actorRef which has the same UID. Besides, wec can find that during restart process, the mailbox is not changed.

[More explainations](http://doc.akka.io/docs/akka/snapshot/scala/actors.html#Actor_Lifecycle)

#### Actor Start

#### Actor Restart

#### Actor Stop
