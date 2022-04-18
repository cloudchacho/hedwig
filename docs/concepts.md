# Concepts

These are the primary concepts to know about when dealing with messaging systems.

## Message

A message is simply a packet of information. The information can be anything - data, metadata, log. Hedwig provides
a mechanism for transferring this information from one system to another. A system can be a unix process / an app in a
distributed system. The semantics of the information are up to you, Hedwig doesn't enforce a particular meaning - 
only the schema to which your data must conform. Some possible messages are defined below. In addition to data, messages
have some metadata associated with them: id, creation timestamp, publisher name, schema, and headers.

### Events

The most common use case is to notify other systems that may be interested in events. For example, your User 
Management app can publish a ``user-created`` message notification to all your apps. As publishers and consumers
are loosely coupled, this separation of concerns is very effective in ensuring a stable ecosystem. It's recommended
that the data only includes control plane information, such as entity identifiers, and not data plan information, such
as entity properties.

### Asynchronous APIs

Hedwig may be used to call APIs asynchronously. The contract is enforced by your infrastructure by connecting topics
to queues, and payload is validated using the schema you define. Response is a delivered using a separate message if
required.

## Publishers / Consumers

Messages are published by systems which are called publishers and consumed by systems which we call consumers. A 
system can be both consumer and publisher for different kinds of messages. In Hedwig eco-system, publishers are
de-coupled from consumers - publishers aren't aware of consumers, and won't even know if there are any consumers (of 
course publishing messages no one listens to, isn't inherently useful). There may be several consumers interested in
a specific message. All consumers receive a _copy_ of the message.

## Topics

Topic are a common heading under which messages are published. Its utility is being able to isolate different kinds 
of messages so that a system can request messages of a specific kind.

## Queues

A queue is a holding area where messages intended for a specific system are stored before they are consumed. 
Messages must be explicitly deleted after they've been processed, or they live in the queue until a specified 
timeout is reached. If a consumer fails to process messages after a finite number of times, they are moved to a 
different queue called the dead letter queue.

## Schema

A schema defines the constraints of the data that your message can contain. For example, you may want to define that 
your messages are valid JSON values with a specific key. Hedwig provides schema validation so that your application 
doesn't have to think about whether messages are valid.
