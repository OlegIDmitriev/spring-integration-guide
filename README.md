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
    - [Splitter](#splitter)
    - [Aggregator](#aggregator)
    - [JMS integration](#jms-integration)
    - [HeaderEnricher](#headerenricher)
    - [Delayer](#delayer)
    - [Lambda and method reference](#lambda-and-method-reference)
- [Sources](#sources)

---

## EIP

![EIP](/img/01_eip.png)
`source: https://www.enterpriseintegrationpatterns.com/patterns/messaging/index.html`

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

![Message](/img/02_message.png)

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

![Channel](/img/03_channel.png)

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
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns="http://www.springframework.org/schema/integration"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns:beans="http://www.springframework.org/schema/beans"
             xmlns:jms="http://www.springframework.org/schema/integration/jms"
             xsi:schemaLocation="http://www.springframework.org/schema/beans
            https://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/integration
            https://www.springframework.org/schema/integration/spring-integration.xsd
            http://www.springframework.org/schema/integration/jms
            https://www.springframework.org/schema/integration/jms/spring-integration-jms.xsd
            ">

    <jms:message-driven-channel-adapter id="jmsIn"
                                        destination="INT.QUEUE.IN"
                                        channel="transformRequestChannel"/>

    <channel id="transformRequestChannel"/>
    <transformer input-channel="transformRequestChannel" output-channel="saveRequestChannel" expression="payload * 8" />
    <channel id="saveRequestChannel"/>
    <service-activator input-channel="saveRequestChannel"
                       output-channel="sendAnswerChannel"
                       ref="saveRequestHandler"
                       method="handle"/>
    <channel id="sendAnswerChannel"/>

    <jms:outbound-channel-adapter id="jmsout" channel="sendAnswerChannel" destination="INT.QUEUE.OUT"/>

</beans:beans>
```
## Java configuration
```java
@Bean
public JmsMessageDrivenEndpoint startEndpoint() {
    var container = new DefaultMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.setDestinationName("INT.QUEUE.IN");
    container.setBeanName("startContainer");

    var listener = new ChannelPublishingJmsMessageListener();
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
![Transformer](/img/04_transformer.png)

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
![Router](/img/05_router.png)

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
![Filter](/img/06_filter.png)

Filters messages based on conditions.
In a basic case, a filter is similar to a stream filter: it allows only those messages that satisfy a condition to pass further. In a more advanced scenario, a filter can route messages that do not meet the condition to a side channel or a separate flow instead of the main one.

Simple filters can be implemented using lambda functions or method references:
```java
// Filter 1: only strings that are not null will be sent to output channel
// Filter 2: only strings with non-zero length will be sent to output channel
// Filter 3: strings whose length is less than or equal to 3 will be sent to the discardChannel
// Filter 4: strings that are not equal to "World" will be sent to flow1
// The String.class argument allows you to explicitly specify the payload type of the incoming message

@Bean
public IntegrationFlow filterFlow() {        
    return f -> f
            .filter(Objects::nonNull)
            .filter(String.class, s -> s.length() > 0)
            .filter(String.class, s -> s.length() > 3, spec -> spec.discardChannel("discardChannel"))
            .filter("World"::equals, spec -> spec.discardFlow(flow1()))
            ;
}
```
`Custom filter class`
```java
public class MyFilter {
    public boolean resolve(@Payload Integer payload) {
        return payload > 5;
    }
}
```
`Integration config`
```java
@Bean
public IntegrationFlow filterFlow() {
    return f -> f
            .filter(new MyFilter(), "filter", spec -> spec.discardChannel("discardChannel"))
            ;
}
```

## Splitter
![Splitter](/img/07_splitter.png)

A splitter allows you to split 1 incoming message into multiple outgoing messages. In the basic scenario, it receives a message whose payload contains a collection or an Iterable of N elements; as output, it sends N individual messages—one for each element of the original collection. Each outgoing message inherits the headers from the original one and is assigned 3 new headers: correlationId — the group identifier, the same for all elements of the original collection; sequenceSize — the number of elements in the collection; sequenceNumber — the ordinal number of the element. If the original message already contains these headers (for example, if several splitters are configured sequentially in a flow), then an additional header sequenceDetails is assigned, where the same data is stored as a list of lists: [["uuid1", size1, number1], ["uuid2", size2, number2]].
```java
// one message is received as input, with a list in the payload: List.of(1, 2, 3, 4, 5)
// as output, we get 5 individual messages: 1, 2, 3, 4, 5
@Bean
public IntegrationFlow splitterFlow1() {
    return f -> f
        .split();
}


// If we pass an object as input, one of whose fields contains a collection: {"list": [1, 2, 3]}
// then we can pass this collection to the splitter
// PayloadType.class — the class that contains the field list of type List
@Bean
public IntegrationFlow splitterFlow2() {
    return f -> f
        .split(PayloadType.class, payload -> payload.getList());
}

// Or use a method reference
@Bean
public IntegrationFlow splitterFlow2() {
    return f -> f
        .split(PayloadType.class, PayloadType::getList);
}
```

If the function is too complex, you can extend the AbstractMessageSplitter class. It is important that the implemented method returns an Iterable or a Message whose payload contains an Iterable.
```java
import org.springframework.integration.splitter.AbstractMessageSplitter;
import org.springframework.messaging.Message;

import java.util.List;

public class MySplitter extends AbstractMessageSplitter {
    @Override
    protected Object splitMessage(Message<?> message) {
        var list = (List<String>) message.getPayload();
        return list;
    }
}
```
```java
@Bean
public IntegrationFlow splitterFlow() {
    return f -> f.split(new MySplitter());
}
```
<Add configuration tasks for splitWith()>

## Aggregator
![Aggregator](/img/08_aggregator.png)

An aggregator is usually used together with a splitter—when we want to first split an original message into individual elements, perform some action on each of them, and then collect them back. How the aggregator works: when a message is split, each message is assigned 3 headers: correlationId, sequenceSize, sequenceNumber. Upon receiving a message, the aggregator stores it and checks whether the number of stored messages in the group (correlationId) is equal to the group size (sequenceSize). If this condition is met, it sends 1 message to the aggregator’s output channel whose payload contains a collection of the group’s messages, and whose headers are aggregated from the group messages. If the condition is not met, it does nothing.
At the same time, you can configure:
* an arbitrary strategy that determines when a group is considered fully processed
* a timeout for waiting for all messages in the group
* processing of messages after all messages in the group have been received
* different types of storage for message persistence (including persistent ones)


***Basic implementation:***
```java
// an incoming message has a payload containing a list: [1, 2, 3]
// Step 1: split the incoming message into individual ones: 1, 2, 3
// Step 2: multiply the number in the payload of each message by 3
// Step 3: collect 3 messages into one, whose payload is a list: [3, 6, 9]

@Bean
public IntegrationFlow splitAndAggregate() {
    return f -> f
        .split()
        .transform(Integer.class, num -> num * 3)
        .aggregate();
}
```
***Advanced implementation***
`MyAggregator.java`
```java
import org.springframework.messaging.Message;

import java.util.List;
import java.util.stream.Collectors;

public class MyAggregator {
    public MyResponse aggregate(List<Message<?>> messages) {
        List<Integer> list = messages.stream()
            .map(m -> (Integer) m.getPayload())
            .toList();

        return new MyResponse(list);
    }
    
    public static class MyResponse {
        private final List<Integer> list;
        
        public MyResponse(List<Integer> list) {
            this.list = list;
        }
        
        public List<Integer> getList() {
            return list;
        }
    }
}
```
`Config.java`
```java
@Bean
public IntegrationFlow splitAndAggregate(JdbcMessageStore messageStore) {
    return f -> f
        .split()
        .transform(Integer.class, num -> num * 3)
        .aggregate(spec -> spec
            .correlationStrategy(m -> m.getHeaders().get("correlationKey"))
            .releaseStrategy(g -> g.size() > 10)
            .processor(new MyAggregator(), "aggregate")
            .messageStore(messageStore)
        );
}
```
<An alternative approach should be described here/>


## JMS Integration
Reading/sending a message to a JMS queue
Using inbound/outbound adapters:
```java
// read a message from the queue INT.QUEUE.IN
// multiply the payload by 2
// send the response to the queue INT.QUEUE.OUT
@Bean
public IntegrationFlow jmsFlow1() {
    return IntegrationFlow.from(
                    Jms.inboundAdapter(connectionFactory).destination("INT.QUEUE.IN")
            )
            .transform(Integer.class, num -> num * 2)
            .handle(Jms.outoundAdapter(connectionFactory).destination("INT.QUEUE.OUT"));
}

// advanced version
@Bean
public IntegrationFlow jmsFlow2() {
    return IntegrationFlow.from(
                    Jms.inboundAdapter(connectionFactory)
                            .destination("INT.QUEUE.IN")
                            .messageSelector("header='HeaderValue'")
                            .configureJmsTemplate(j -> j.jmsMessageConverter(new SimpleMessageConverter()))
                            .headerMapper(new DefaultJmsHeaderMapper()),
                    e -> e.poller(customPoller())
            )
            .transform(Integer.class, num -> num * 2)
            .handle(
                    Jms.outoundAdapter(connectionFactory)
                            .destinationExpression("headers['replyTo'] ?: 'INT.QUEUE.OUT'"));
}
```
MessageDrivenChannelAdapter
```java
@Bean
public IntegrationFlow jmsFlow1() {
    return IntegrationFlows.from(
                    Jms.messageDrivenChannelAdapter(
                            Jms
                                    .container(connectionFactory, "INT.QUEUE.IN")
                                    .backOff(new ExponentialBackOff())))
            .transform(Integer.class, num -> num * 2)
            .handle(
                    Jms.outoundAdapter(connectionFactory)
                            .destination("INT.QUEUE.OUT"));
}
```
JmsOutboundGateway
<Add information>
```java
// Asynchronous processing
// Read a message from the queue INT.QUEUE.IN
// send it to the external inbound queue INT.EXTERNAL.IN
// read the response from the external outbound queue INT.EXTERNAL.OUT
// log this message
@Bean
public IntegrationFlow simpleFlow1() {
    return IntegrationFlow.from(
                    Jms.inboundAdapter(connectionFactory).destination("INT.QUEUE.IN")
            )
            .handle(asyncGateway())
            .handle(Jms.outoundAdapter(connectionFactory).destination("INT.QUEUE.OUT"));
}

private JmsOutboundGateway asyncGateway() {
    var gateway = new JmsOutboundGateway();
    gateway.setConnectionFactory(connectionFactory);
    gateway.setRequestDestinationName("INT.EXTERNAL.IN");
    gateway.setReplyDestinationName("INT.EXTERNAL.OUT");
    gateway.setAsync(true);
    gateway.useReplyContainer(true);
    gateway.setCorrelationKey("correlationKey");
    gateway.setReceiveTimeout(0L);

    return gateway;
}

// Synchronous processing
@Bean
public IntegrationFlow simpleFlow2() {
    return IntegrationFlow.from(
                    Jms.inboundAdapter(connectionFactory).destination("INT.QUEUE.IN")
            )
            .handle(syncGateway())
            .handle(Jms.outoundAdapter(connectionFactory).destination("INT.QUEUE.OUT"));
}

private JmsOutboundGateway syncGateway() {
    var gateway = new JmsOutboundGateway();
    gateway.setConnectionFactory(connectionFactory);
    gateway.setRequestDestinationName("INT.EXTERNAL.IN");
    gateway.setReplyDestinationName("INT.EXTERNAL.OUT");
    gateway.setAsync(false);
    gateway.setCorrelationKey("correlationKey");
    gateway.setReceiveTimeout(120_000L);

    return gateway;
}

// With object serialization to XML and deserialization of the response
@Bean
public IntegrationFlow simpleFlow3() {
    return IntegrationFlow.from(
                    Jms.inboundAdapter(connectionFactory).destination("INT.QUEUE.IN")
            )
            .handle(asyncGateway())
            .handle(Jms.outoundAdapter(connectionFactory).destination("INT.QUEUE.OUT"));
}

private JmsOutboundGateway xmlSerializeGateway() {
    var gateway = new JmsOutboundGateway();
    gateway.setConnectionFactory(connectionFactory);
    gateway.setRequestDestinationName("INT.EXTERNAL.IN");
    gateway.setReplyDestinationName("INT.EXTERNAL.OUT");
    gateway.setAsync(true);
    gateway.useReplyContainer(true);
    gateway.setCorrelationKey("correlationKey");
    gateway.setReceiveTimeout(0L);

    return gateway;
}

private MessageConverter xmlMessageConverter() {
    var marshaller = new Jaxb2Marshaller();
    marshaller.setMarshallerProperties(Map.of(
            Marshaller.JAXB_FORMATTED_OUTPUT, true,
            Marshaller.JAXB_FRAGMENT, true,
            Marshaller.JAXB_ENCODING, "UTF-8"
    ));

    marshaller.setClassesToBeBound(RequestXml.class, ResponseXml.class); //рутовые JAXB классы запроса и ответа вншнего сервиса
    marshaller.setPackagesToScan("com.dmitriev.oleg.external.service.jaxb");

    var messageConverter = new MarshallingMessageConverter();
    messageConverter.setTargetType(MessageType.TEXT);
    messageConverter.setMarshaller(marshaller);
    messageConverter.setUnmarshaller(marshaller);
    return messageConverter;
}
```
## HeaderEnricher
```java
@Bean
public IntegrationFlow simpleFlow() {
    return f -> f
            .enrichHeaders(spec -> spec
                .header("header1", "headerValue")                 // add header1 with value headerValue
                .headerExpression("header2", "payload.id")        // add header2 with value from the id field of the message payload
                .headerExpression("header3", "headers.messageId", true) // add header3 with the value of the messageId header
            );
}
```
## Delayer
If you need to delay the processing of a message, you can use the delayer component. You can set a fixed delay or take it from a header. In order not to lose waiting messages after a restart, you need to configure a persistent messageStore.
```java
@Bean
public IntegrationFlow simpleFlow() {
    return f -> f
        .delay(spec -> spec
            .groupId("waitGroupId")
            .defaultDelay(3_000L) // 3s
            .messageStore(messageStore)
        );
}
```
## Lambda and Method Reference
For simple actions, you can directly use lambda functions and method references inside a flow.
```java
// an incoming message has a payload of 1
// the output channel prints 16

@Bean
public IntegrationFlow flow() {
    return f -> f
        .transform(Integer.class, num -> num * 8)
        .filter(Integer.class, num -> num > 0)
        .handle(Integer.class, (p, h) -> p * 2) // p - payload, h - headers
        .transform(Object::toString)
        .handle(System.out::println);
}
```
# Sources
- https://docs.spring.io/spring-integration/reference/
- https://www.baeldung.com/spring-integration
