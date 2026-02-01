# Spring Integration Introduction Guide

## Contents
- [EIP](#EIP)
- [Main EIP patterns implemented in Spring Integration](#main-eip-patterns-implemented-in-spring-integration)
- [Main terms](#main-terms)
    - [Message](#message)
    - [Message Channel](#message-channel)
    - [Message Endpoint](#message-endpoint)
    - [Correlation with JMS terms](#coreraltion-with-jms-terms)
- [Spring Integration configuration overview](#spring-integration-configuration-overview)
- [Main patterns examples](#main-patterns-examples)
    - [Channels](#channels)
    - [Handler](#handler)
    - [Transformer](#transformer)
    - [Router](#router)
    - [Filter](#filter)
    - [Splitter]
    - [Aggregator]
    - [JMS integration]
    - [HeaderEnricher]
    - [Delayer]
    - [Lambda and method reference]
- [Examples]
- [Sources]

---

## EIP

![EIP](EIP.jpg)

There is a book called **Enterprise Integration Patterns** by Martin Fowler that describes patterns for integrating enterprise applications.  
The focus is on asynchronous communication and queue-based integration.

The book describes several hundred patterns. Implementations of EIP include:
- Spring Integration
- Apache Camel
- Mule
- NiFi

This guide considers only **Spring Integration**.

---

## Main EIP patterns implemented in Spring Integration

### Patterns
- Endpoint
- Channel (Point-to-point and Publish-subscribe)
- Aggregator
- Filter
- Transformer
- Control Bus

### Integrations
- REST / HTTP
- FTP / SFTP
- WebServices (SOAP and REST)
- TCP / UDP
- JMS
- RabbitMQ
- Email
- DB

---

## Main terms

Spring Integration is one of the implementations of EIP and an implementation of the **pipes-and-filters architecture**.

The idea of this architecture is to split complex processing into a number of independent elements that can be reused when needed.  
Each processing step happens inside a filter component. Data is passed through channels.

In Spring Integration:
- **Message Endpoint** acts as a filter
- **Message Channel** acts as a channel
- **Message** is transferred through channels and delivered to endpoints

---

## Message

![Message](message.jpg)

In Spring Integration, a `Message` is a structure consisting of:
- **Payload** – can contain any Java object
- **Headers** – metadata

Headers usually contain mandatory elements such as:
- `id`
- `timestamp`
- `correlationId`

Any arbitrary key-value pairs can be stored in headers.

A `Message` can be considered a direct analogue of `javax.jms.Message`.

---

## Message Channel

![Channels](channels.jpg)

A message channel represents a *pipe* that connects different handlers.

- A **producer** sends messages to a channel
- A **consumer** receives messages from a channel

Channels:
- Reduce coupling between components
- Provide convenient interception and monitoring points

### Channel types

- **Point-to-point**
    - Each message can be consumed by only one consumer
- **Publish-subscribe**
    - Each message is delivered to all subscribers

Spring Integration supports both models.

A message channel is analogous to a **JMS Destination**.

---

## Message Endpoint

A Message Endpoint represents a *filter* whose main purpose is to:
- Process a message or its headers
- Perform side effects

Common Message Endpoints:
- Message Transformer
- Message Filter
- Message Router
- Splitter
- Aggregator
- Channel Adapter

---

## Correlation with JMS terms

| EIP                    | JMS                        |
|------------------------|----------------------------|
| Message Channel        | Destination                |
| Point-to-point channel | Queue                      |
| Publish-subscribe      | Topic                      |
| Message                | Message                    |
| Message Endpoint       | Producer / Consumer        |

---

## Spring Integration configuration overview

Spring Integration can be configured using:
- XML
- Java configuration
- Java DSL

Below is the same process described in different ways:

1. Read a message from `INT.QUEUE.IN`
2. Multiply the payload by `8`
3. Invoke `SaveRequestHandler`
4. Send the response to `INT.QUEUE.OUT`

---

## XML configuration

```xml
<?xml version="1.0" encoding="UTF-8" ?>
---
```
## Java configuration
```java
@Bean
public JmsMessageDrivenEndpoint startEndpoint() {
    DefaultMessageListenerContainer container = new DefaultMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.setDestinationName("INT.QUEUE.IN");
    container.setBeanName("startContainer");

    ChannelPublishingJmsMessageListener listener =
            new ChannelPublishingJmsMessageListener();
    listener.setRequestChannel(transformRequestChannel());

    return new JmsMessageDrivenEndpoint(container, listener);
}
```
Java DSL
```java
@Bean 
public IntegrationFlow myFlow() {
    return IntegrationFlow
        .from(Jms.inboundAdapter(connectionFactory)
        .destination("INT.QUEUE.IN"))
        .transform(Integer.class, num -> num * 8)
        .handle(new SaveRequestHandler())
        .handle(Jms.outboundAdapter(connectionFactory)
        .destination("INT.QUEUE.OUT"))
        .get();
}
```
DSL allows describing integration flows in a very compact way.

Key points:
- Flow starts with `IntegrationFlows.from()`
- Channels are created automatically
- Flows can be split and reused
- Each flow has `<flowName>.input` channel

# Main patterns examples
## Channels
### DirectChannel
Default channel type in Spring Integration.
Point-to-point with no additional logic.
```java
@Bean
public MessageChannel myChannel() {
    return new DirectChannel();
}
```
### PublishSubscribeChannel
Publish-subscribe semantics.
Each message is delivered to all subscribers.
```java
@Bean
public MessageChannel myChannel() {
    return new PublishSubscribeChannel();
}
```
Thread pool configuration:
```java
@Bean
public Executor executor() {
    return Executors.newFixedThreadPool(5);
}

@Bean
public MessageChannel pubSubChannel() {
    return new PublishSubscribeChannel(executor());
}
```
## Handler
Base component for:
- Message processing
- Side actions (DB, queues, etc.)

### Reply-producing handler
```java
public class MyHandler
        extends AbstractReplyProducingMessageHandler {

    @Override
    protected Object handleRequestMessage(Message<?> message) {
        return message.getPayload();
    }
}
```
### Terminal handler
```java
public class MyHandler
        extends AbstractMessageProducingHandler {

    @Override
    protected void handleMessageInternal(Message<?> message) {
        // terminal logic
    }
}
```
## Transformer

Used to explicitly transform messages.

### Payload-only transformer
```java
public class MyTransformer
        extends AbstractPayloadTransformer<String, Integer> {

    @Override
    protected Integer transformPayload(String payload) {
        return Integer.parseInt(payload);
    }
}
```
### Header-aware transformer
```java
public class MyTransformer extends AbstractTransformer {

    @Override
    protected Object doTransform(Message<?> message) {
        return message;
    }
}
```
## Router

Routes messages based on payload or headers.
```java
.route(Integer.class, a -> a > 0, spec -> spec
    .channelMapping(true, "channel1")
    .channelMapping(false, "channel2"))
```
Supports:
- Channel routing
- Sub-flow routing
- Default flows
- Return to parent flow

## Filter
Filters messages based on conditions.
```java
.filter(String.class, Objects::nonNull)
.filter(String.class, s -> s.length() > 3,
        spec -> spec.discardChannel("discardChannel"))
```
Custom filter class:
```java
public class MyFilter {
    public boolean resolve(Integer payload) {
        return payload > 5;
    }
}
```
# Sources
- Enterprise Integration Patterns
- Spring Integration documentation