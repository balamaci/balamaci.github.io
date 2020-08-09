+++
author = "Serban Balamaci"
date = 2016-08-10T14:09:03Z
description = ""
draft = false
slug = "reactive-log-processing"
title = "Reactive log stream processing with RxJava - Part I"

+++


### Centralized logs as a data source for realtime data analysis
In the previous blog entries we saw how we can leverage the power of the ELK stack for log collection and analysis of our Java apps.

With the move towards microservices or containerization of applications it becomes the defacto standard to have a stack for centralized log processing and storage.
Can we maybe go the next step and use that information proactively instead of just for just finding the cause of problems long after they appeared?

If **we were to consider the log events as a stream of data for things happening in realtime** in your system it would be very interesting to tap into and perform **realtime data analysis** with all sorts of uses like **detecting fraudulent behavior** for example by aggregating different streams of information while the "attack" is happening and block the attacker instead of the "traditional" way of just using log data for forensics to investigate after the issue appeared.


For ex. we could **filter** only events of a certain type, **'group by'** a common key like the userID and count them in a time-window to get the number of actions of that type the user is doing in a certain timeframe.
```java
  failedLoginsStream()
                .window(5, TimeUnit.SECONDS)
                .flatMap(window ->
                          window
                            .groupBy(propertyStringValue("remoteIP"))
                            .flatMap(grouped -> grouped
                                .count()
                                .map(failedLoginsCount -> {
                                    final String remoteIp = grouped.getKey();
                                    return new Pair<>(remoteIp, failedLoginsCount);
                                }))
                )
                .filter(pair -> pair.get > 10)
                .forEach(system.out:println);
```

We could trigger queries in other systems and treat those responses as streams to which we can subscribe and apply **a multitude of common stream operators** that reactive streams frameworks provide.

And the good thing is that since you're working with already existing log outputs you can build this separate and extend it without touching or polluting the business logic of the application.


### Learning a new programming paradigm       
This could be a good excuse to get into the world of **Reactive stream programming**.
Reactive programming is about **non-blocking**, **event-driven** applications that scale even on a small number of threads with back-pressure(feedback mechanism to ensure producers do not overwhelm consumers).

The biggest new thing **Spring5 will bring will be [Reactive support](https://spring.io/blog/2016/04/19/understanding-reactive-types)**. A new module [spring-web-reactive](https://github.com/spring-projects/spring-framework/tree/master/spring-web-reactive) a framework similar with spring-web-mvc that enables async response(non blocking) REST services and a reactive web-client and  that will probably work great for microservice architectures. 
The reactive-streams concepts is not just Spring specific, but instead there is a common specification [reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm) agreed upon by the major reactive frameworks(so while there might not be exact name matches the concepts will make it easy to switch frameworks). 
Historically Rx.NET introduced the reactive-streams model, and Netflix ported it to java with RxJava. Then the concept has been implemented into other languages as well, under the [Reactive EXtensions](http://reactivex.io/) umbrella.   
Then since other companies were going kinda in the same direction the [reactive-streams](http://www.reactive-streams.org/) specification took of. Now **RxJava** since it was kind of the pioneer needs to do a bigger refactor(in version **[2.x](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0)** to better match the specification, while **Spring reactor** being newer could start fresh with directly implementing the spec.
You care read more about how they [relate](https://spring.io/blog/2016/06/07/notes-on-reactive-programming-part-i-the-reactive-landscape). 

Also reactive streams will come to Java 9 as Doug Lea wants to include the reactive-streams under container object [java.util.concurrent.Flow](http://gee.cs.oswego.edu/dl/jsr166/dist/docs/java/util/concurrent/Flow.html) 
 

#### Benefits from a performance perspective       
Also another buzzword right now is the **microservices architecture** where you need to be able to make requests to multiple other services. Ideally you'd want to do it **without blocking** waiting for the whole response before making the request to the next service. Think that instead of waiting for a whole possibly huge List<Objects> to be returned by a service, it might be the case to start a query in another system as soon as the first element is available.   

![Never block](/assets/rxjava/dogs.jpg)

Treating that remote call response as a Stream to which we subscribe for an action when the response arrives, instead of blocking the thread waiting for the response means we can use **less threads overall** which in turn means **less resources wasted(cpu for context switching between threads and memory for each thread stack)**.
So by using reactive programming we should be **able to handle a larger amount of log events with commodity hardware**.


An example: If we're a service like GMail and we need to display the user's emails. However emails in turn might have many people in CC. It would be nice to display a photo of those that the user has in his contacts - which means a REST call in the ContactsService

We'd normally have something like
```java
Future<List<Mail>> emailsFuture = mailstoreService.getUnreadEmails();
List<Mail> emails = emailsFuture.get(); //blocking the current thread
//waiting possibly for a long time to get the whole list
//we cannot start next processing as soon as the first email is found??

Future<List<Contacts>> contacts = getContactsForEmails(emails);
for(Mail mail : emails) {
  streamRenderEmails(mail, contacts); //push emails to the client
}
```
Partially the problem has been improved with the reactive support from Java8 with **CompletableFuture**(which with it's thenCompose, thenCombine, thenAccept and other 50 something methods it's not making it easy to remember what each method does, which in turn I think doesn't help readability).

```
CompletableFuture<List<Mail>> emailsFuture = mailstoreService.getUnreadEmails();

CompletableFuture<List<Contact>> emailsFuture
  .thenCompose(emails -> getContactsForEmails(emails)) //we still need to wait for the List<Mail> to 
  .thenAccept(emailsContactsPair -> streamRenderEmails(emailsContactsPair.getKey(), emailsContactsPair.getValue()))
```

we could switch to an Iterator data type instead of List still there are no methods to tell it to do something when a new value arrives. SQL does this with returning the ResultSet(on which you can do rs.next()) instead of getting the whole data in memory.  
```
public interface Iterator<E> {
    /**
     * Returns {@code true} if the iteration has more elements.
     */
    boolean hasNext();

    /**
     * Returns the next element in the iteration.
     */
    E next();
}
```

But we need to constantly ask "do you have a new value"?
```
Iterable<Mail> emails = mailstoreService.getUnreadEmails();
Iterator<Mail> emailsIt = emails.iterator();

while(emailsIt.hasNext()) {
  Mail mail = emailsIt.next(); //nonblockin but we still need to constantly waste cpu asking
  if(mail != null) {           //for new values
      ....
  }
}

```
What we'd need instead would be like a **reactive Iterator**, a datatype to which you could subscribe for an action to be executed once there is a new value ready and this is where reactive stream programming begins.

#### So what is a Stream?
![Everything is a stream](https://camo.githubusercontent.com/e581baffb3db3e4f749350326af32de8d5ba4363/687474703a2f2f692e696d6775722e636f6d2f4149696d5138432e6a7067)

A Stream is simply **a sequence of events ordered in time** (eventX was emitted after eventY, **there are no concurrent events**).  

A Stream is modeled so it can emit **0..N events** and **either one of two terminal operations**:
  - **completed** event through which it signals subscribers that it finished emitting data        
  - **error** event for signaling it finished exceptionally

We can describe that visually with the use of 'marble diagrams'.
![Marble diagram for Observable](/assets/rxjava/RxJava1.svg)

So everything can be thought as being a Stream, not just log events. Even a single value can be expressed as a Stream by emitting the value followed by an **completed** event.
An infinite stream is one that only emits events but not any of the terminal events(completed | error).

RxJava defines the **Observable<T>** **data type** to model the Stream of events of type &lt;T&gt;. Spring Reactor's equivalent is [Flux<T>](https://projectreactor.io/core/docs/api/reactor/core/publisher/Flux.html) 

   - Observable&lt;Double&gt; to represent a stream of temperatures taken at various intervals.  

   - Observable&lt;CartItem&gt; to represent a stream of products bought in our web store.

   - Observable&lt;User&gt; to represent a single User returned by a DB query
```
    public Observable<User> findByUserId(String userId) {...}
    //or Single for being more explicit 
    public Single<User> findByUserId(String userId) {...}
```

But **Observable&lt;T&gt;** is just a datatype and same as the Publish/Subscriber pattern, we need a Subscriber to process the 3 types of events

```java
        Observable<CartItem> cartItemsStream = ...;

        Subscriber<CartItem> subscriber = new Subscriber<CartItem>() {
            @Override
            public void onNext(CartItem cartItem) {
                System.out.println("Cart Item added " + cartItem);
            }

            @Override
            public void onCompleted() {
            }

            @Override
            public void onError(Throwable e) {
                e.printStackTrace();
            }
        };

        cartItemsStream.subscribe(subscriber);
```


#### Reactive operators
But that is just the Stream part, and until now we've not been doing something more special than the classic Observer pattern.  
The **Reactive** part means we can **define some Function**(the **operators**) that will be executed when the stream emits an event.
That means another stream will be created(streams are immutable) to which we can subscribe another operator and so on.

```java
Observable<CartItem> filteredCartStream = cartStream.filter(new Func1<CartItem, Boolean>() {
            @Override
            public Boolean call(CartItem cartItem) {
                return cartItem.isLaptop();
            }
        });

Observable<Long> laptopCartItemsPriceStream = filteredCartStream.map(new Func1<CartItem, Long>() {
            @Override
            public Long call(CartItem cartItem) {
                try {
                    return priceService.getPrice(cartItem.getId());
                } catch(PriceServiceException e) {
                    thrown new RuntimeException(e);
                }
            }
        });
```

Since the operator methods of the Observable class(filter, map, groupBy,...) return Observable, it means **we can chain the operators together** and combined with lambda syntax we can write something pretty
```
Observable<BigDecimal> priceStream = cartStream
                        .filter((cartItem) -> cartItem.isLaptop()).
                        .map((laptop) -> {
                             try {
                                  return priceService.getPrice(cartItem.getId());
                            } catch(PriceServiceException e) {
                                 thrown new RuntimeException(e);
                            }
                        });
```


The thing to notice above is that **when creating _priceStream_ nothing is happening** in the sense that **priceService.getPrice()** is not getting invoked until there is an item flowing through the operators chain.
That means we created through the rx-operators sort of a blueprint of how the data will be manipulated once it starts flowing downstream(a subscriber registers).

When asked to explain **reactive programming** people jokingly give as an example an Excel sheet where you write the formulas for columns and as soon as you update a cell the formula is triggered which updates another cell which in turn triggers another formula and so on. Just like that the rx-operators don't do anything by themselves they are just formulas for data manipulation and each gets a chance to do it's thing before passing it down the chain.

To help understand better how events travel along the operators chain I found helpful the analogy of the house movers proposed by [Thomas Nield](https://tomstechnicalblog.blogspot.ro/2015/10/understanding-observables.html) where each house mover is an operator passing along house objects.

He gives as example the following code:
```java
Observable<Item> mover1 = Observable.create(s -> {
   while (house.hasItems()) {
    s.onNext(house.getItem());
   }
   s.onCompleted();
});

Observable<Item> mover2 = mover1.map(item -> putInBox(item));

Subscription mover3 = mover2.subscribe(box -> putInTruck(box),
   () -> closeTruck()); //this is what runs for onCompleted
```
![Movers](https://1.bp.blogspot.com/-1RuGVz4-U9Q/VjT0AsfiiUI/AAAAAAAAAKQ/xWQaOwNtS7o/s640/animation_2.gif)


"**Mover 1** on the far left is the **source Observable**. He creates the emissions by picking items out of the house. 
He then calls onNext() on **Mover 2, who does a map() operation**. When his onNext() is called he takes that Item and puts it in a Box. Then he calls onNext() on **Mover 3, the final Subscriber** who loads the box on the truck."


The magic or RxJava is the large set of operators available and your job on how to combining them together to manipulate the flow of data.
![](https://github.com/Netflix/RxJava/wiki/images/rx-operators/Composition.1.png)

The **many Stream operators** help establish **a common vocabulary of manipulations when dealing with streams** that can have implementations in popular languages(RxJava, RxJS, Rx.NET, etc) of the ReactiveX framework.
These concepts should be familiar even when using different reactive streams frameworks like [Spring Reactor]()(with the hope of an agreement for a common base of [operators](https://github.com/reactor/reactive-streams-commons)).

Until now we only saw simple operators like **filter**:

![Filter](http://reactivex.io/documentation/operators/images/filter.png)

Which means but it only pushes downstream elements which pass the filtering condition(one mover drops everything with a value < 100$ instead of passing it to the next mover)

There are however operators which can split a stream into many streams - Observable&lt;Observable&lt;T&gt;&gt;(Stream of Streams) - operators like  **groupBy**

![GroupBy](https://blogs.endjin.com/wp-content/uploads/2014/04/event-stream-with-groupby1.png)

```java
        Observable<Integer> values = Observable.just(1,4,5,7,8,9,10);
        Observable<GroupedObservable<String, Integer>> oddEvenStream = values.groupBy((number) -> number % 2 == 0 ? "odd":"even");
        Observable<Integer> remergedStream = Observable.concat(oddEvenStream);
        remergedStream.subscribe(number -> System.out.print(number +" "));

//Outputs
//1 5 7 9 4 8 10 
```
and the rather simple operator **concat** which merges back the "odd" and "even" stream into a single one to which we subscribe.

![Concat](http://reactivex.io/documentation/operators/images/concat.png)

as we can see the **concat** operator waits for a stream to complete before appending another one and so on, creating back a single stream. Thus the odd numbers were displayed first.


Also we have the way to merge back together multiple streams like **zip** operator
![Zip Operator](/content/images/2016/08/zip.png) 


Zip is not named that way because it's acting like an archiving program but rather from the way a **zipper** works to combine events from two streams.  

![Zipper](https://upload.wikimedia.org/wikipedia/commons/f/f0/Zipper_animated.gif) 

It's taking one event from a stream and pairs it with another from the other stream as soon as one is ready, and applies a merging operator before sending it downstream.
PS: It also works with more than just two streams.
 
So even if one stream is emitting faster, the downstream listener will only see the combined event as soon as there is a matching event being emitted on the slower stream.
It's actually very useful as a way to "wait" for the response of multiple remote calls which return streams.
  
The **combineLatest** on the other hand it's not waiting for a pair emission to appear but instead uses the last emission from the slower stream before applying the merge function and sending it downstream.
![Combine latest](https://camo.githubusercontent.com/2a1a467dc61743d40f92fd6df1038f4e2a3ded7c/687474703a2f2f692e696d6775722e636f6d2f3670316931447a2e706e67)

#### Moving to a Push based mindset
Let's see some examples of how you can actually create Observables. The most verbose variant but which let us understands more:

```
log.info("Before create Observable");
Observable<Integer> someIntStream = Observable
    .create(new Observable.OnSubscribe<Integer>() {
                @Override
                public void call(Subscriber<? super Integer> subscriber) {
                    log("Create");

                    subscriber.onNext(3);
                    subscriber.onNext(4);
                    subscriber.onNext(5);
                    subscriber.onCompleted();

                    log("Completed");
                }
    });   
log.info("After create Observable");

log.info("Subscribing 1st");
someIntStream.subscribe((val) -> log.info("received " + val)); //we don't have to implement
//the other methods(for onError and onComplete) if we don't want to do something specific

log.info("Subscribing 2nd");
someIntStream.subscribe((val) -> log.info("received " + val));    
```

**Events are pushed onto the subscriber as soon as it subscribes**. It's not doing this on construction, here we just passed it a **new OnSubscribe** object which represent what to do when someone subscribes.   
Until we subscribe to the Observable there is no output, there is nothing happening - the data is not flowing-. 
When someone subscribes, the call() method is invoked and 3 events are pushed downstream followed by the signal that the stream completed.


Above we subscribed twice, the code inside call(...) method will be invoke twice also. So it's effectively re-pushing the same values as soon as someone else subscribes and the following output will be produced:
```
mainThread: Before create Observable
mainThread: After create Observable
mainThread: Subscribing 1st
mainThread: Create
mainThread: received 3
mainThread: received 4
mainThread: received 5
mainThread: Completed
mainThread: Subscribing 2nd
mainThread: Create
mainThread: received 3
mainThread: received 4
mainThread: received 5
mainThread: Completed
```

Important thing to notice is that rx operators don't necessarily mean multithreading. RxJava doesn't inject any concurrency by default between the Observable and the Subscriber. This is why all the calls are happening on the 'main' thread. 


This kind of Observable that is starting even emission when someone subscribes is what we call **cold observables**. The other type is **hot observables**, they emit events even when nobody is subscribed.

   - **Cold Observables**
        Only begins emitting the events when someone subscribes - they start the work(ex: makes a DB query) when subscribed to-. Each subscriber gets the same events.
        Sort of **like a CD where the same songs are played** to whomever puts the cd into the player to listen.

   - **Hot Observables**
        Events are emitted even when there are no subscribers.
        **Like a radio stations where it plays the song in the air even when nobody is tuned in**. Just like when you tune in later on a station, you miss previous events. Hot observables model events to which you don't have control over when they emit. Like when the log events are produced.

**Subject** s are special kind of Observable that is also an Observer(like Subscriber - which means we can push events(call onNext()) to them-) and make implementing hot Observables easier. There exists more implementations like **ReplaySubject** that keeps the emitted events in a buffer and replay them on subscription(you can ofcourse specify the size of the buffer to prevent OutOfMemory), while **PublishSubject** only pass on events that happened after subscription.



And of course there are more static helper methods for creating Observables from other sources
```
Observable.just("This", "is", "something")
Observable.from(Iterable<T> collection)
Observable.from(Future<T> future) - emits the value when the future completes
```

<br/>

### Adding a push based data emitter to our ELK stack - RabbitMQ
In a traditional [ELK](https://balamaci.ro/java-app-monitoring-with-elk-logstash/) stack we use ElasticSearch to query the log events data so a more **'pull based'** system.
Could we have instead a **push based** where we're notified **"immediately"** when another log event appears to further reduce the reaction time from when the event is produced to when we begin processing it.
One of the many possible solutions would be **RabbitMQ** as being a battle tested solution with a very good reputation for performance for handling a huge amount of messages. Besides that Logstash already supplies a plugin for RabbitMQ(there is also one for FluentD) so we can easily integrate it in our existing ELK stack and write the log data both to ElasticSearch and RabbitMQ.
You may remember from that Logstash can act as a controller and choose how to process and where to send/store the log events. That means we can decide to filter log events we want to process or to send them to different RabbitMQ queues.


There is even the option to send data directly to RabbitMQ through a Logback [Appender](http://docs.spring.io/spring-amqp/api/org/springframework/amqp/rabbit/logback/AmqpAppender.html) should you want to bypass Logstash. 
Sidenote: While named AmqpAppender, it's rather specific to RabbitMQ AMQP implementation(AMQP protocol version 0-9-1, 0-9). ActiveMQ for ex.(while also supplying an AMQP connector) seems to implement AMQP protocol 1.0, while the spring-amqp library works with the protocol versions 0-9-1, 0-9 which are totally different at the wire level than 1.0) so you'll encounter a nice exception like
'org.apache.activemq.transport.amqp.AmqpProtocolException: Connection from client using unsupported AMQP attempted'

However our solution was to use [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder) to send JSON formatted log events to Logstash. We'll now redirect the logstash output to a RabbitMQ exchange.  



We'll use **docker-compose** to start up a logstash-rabbitmq cluster. You can clone the [repo](https://github.com/balamaci/blog-elk-docker).  

```
docker-compose -f docker-compose-rabbitmq.yml up
```
and you can use **./event-generate.sh** to generate some random events that get pushed to logstash.

It's using the logstash configuration [file](https://github.com/balamaci/blog-elk-docker/blob/rabbitmq/logstash/config-rabbitmq/logstash.conf) to specify where we want to send the data. We use the [rabbitmq-output-plugin](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-rabbitmq.html) as reference:
```
output {
    rabbitmq {
        exchange => logstash
        exchange_type => direct
        host => rabbitmq
        key => my_app
    }
}
```
RabbitMQ is not a traditional JMS server, instead it uses the **AMQP protocol** which has quite a different concept for queues. 

![AMQP](http://blog.springsource.com/wp-content/uploads/2010/06/rabbit-basics.png)

A publisher sends messages to a named **exchange** and a consumer pulls messages from a queue. The message has a standard header 'routing-key' which is used in a process called **binding** to associate an exchange message to a queue. The queues can filter which messages they receive via the binding key and you can use wildcards in the binding like 'logstash.*' 

For an indepth explanation for AMQP you can read [here](https://spring.io/blog/2010/06/14/understanding-amqp-the-protocol-used-by-rabbitmq/) and [here](https://www.cloudamqp.com/blog/2015-09-03-part4-rabbitmq-for-beginners-exchanges-routing-keys-bindings.html).

So we [configured](https://github.com/balamaci/rxjava-rabbitmq/blob/master/src/main/java/com/balamaci/rx/configuration/AmqpSourceConfiguration.java) a Spring connection to RabbitMq
```
    @Bean
    ConnectionFactory connectionFactory() {
        return new CachingConnectionFactory(host, port);
    }

    @Bean
    RabbitAdmin rabbitAdmin() {
        RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory());
        rabbitAdmin.declareQueue(queue());
        rabbitAdmin.declareBinding(bindQueueFromExchange(queue(), exchange()));
        return rabbitAdmin;
    }

    @Bean
    SimpleMessageListenerContainer container(ConnectionFactory connectionFactory, MessageListenerAdapter listenerAdapter) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames(queueName);
        container.setMessageListener(listenerAdapter);

        return container;
    }

    @Bean
    Queue queue() {
        return new Queue(queueName, false);
    }

    DirectExchange exchange() {
        return new DirectExchange("logstash");
    }

    private Binding bindQueueFromExchange(Queue queue, DirectExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with("my_app");
    }

    @Bean
    MessageListenerAdapter listenerAdapter(Receiver receiver) {
        MessageListenerAdapter messageListenerAdapter = new MessageListenerAdapter(receiver,
                new MessageConverter() {
            public Message toMessage(Object o, MessageProperties messageProperties)
                    throws MessageConversionException {
                throw new RuntimeException("Unsupported");
            }

            public String fromMessage(Message message) throws MessageConversionException {
                try {
                    return new String(message.getBody(), "UTF-8");
                } catch (UnsupportedEncodingException e) {
                    throw new RuntimeException("UnsupportedEncodingException");
                }
            }
        });
        messageListenerAdapter.setDefaultListenerMethod("receive"); //the method in our Receiver class
        return messageListenerAdapter;
    }


    @Bean
    Receiver receiver() {
        return new Receiver();
    }
```
We defined a queue and bind it to the 'logstash' exchange to receive messages with the 'my_app' routing key. 
The **MessageListenerAdapter** above defines that the 'receive' method should be called on Receiver bean every time a new message is received from the queue. 

Since we're expecting a continuous stream of log events we don't have control over, we can think of using a hot observable that pushes events to all subscribers after they subscribed so we can use **PublishSubject** for the job.     
```
public class Receiver {
    private PublishSubject<JsonObject> publishSubject = PublishSubject.create();

    public Receiver() {
    }

    /**
     * Method invoked by Spring whenever a new message arrives
     * @param message amqp message
     */
    public void receive(Object message) {
        log.info("Received remote message {}", message);
        JsonElement remoteJsonElement = gson.fromJson ((String) message, JsonElement.class);
        JsonObject jsonObj = remoteJsonElement.getAsJsonObject();

        publishSubject.onNext(jsonObj);
    }

    public PublishSubject<JsonObject> getPublishSubject() {
        return publishSubject;
    }
}

```

We need to be aware that event the **SimpleMessageListenerContainer** supports having more than one thread that consumes from the queue(and emits the events downstream). However the Observable contract says we cannot emit events concurrently(calls to onNext,onComplete,onError must be serialized):     
```
// DO NOT DO THIS
Observable.create(s -> {
                    // Thread A
                    new Thread(() -> {
                        s.onNext("one");
                        s.onNext("two");
                    }).start();
                    
                    // Thread B
                    new Thread(() -> {
                        s.onNext("three");
                        s.onNext("four");
                    }).start();
                });
// DO NOT DO THIS

//DO THIS
Observable<String> obs1 = Observable.create(s -> {
                    // Thread A
                    new Thread(() -> {
                        s.onNext("one");
                        s.onNext("two");
                    }).start();
                  });

Observable<String> obs2 = Observable.create(s -> {
                    // Thread B
                    new Thread(() -> {
                        s.onNext("three");
                        s.onNext("four");
                    }).start();
                });

Observable<String> c = Observable.merge(obs1, obs2);
```


We could go around this problem by calling Observable.serialize() or Subject.toSerialized() but since we just go with the default of 1 Thread in the ListenerContainer, there is no need to do so. Still you need to be aware of this if you plan to use Subjects as an event bus pushing events onto from multiple threads. Read a more [indepth explanation](https://artemzin.com/blog/rxjava-thread-safety-of-operators-and-subjects/).



For now, you can checkout out the code from the [repo](https://github.com/balamaci/rxjava-rabbitmq) as we continue this long post in [Part II](https://balamaci.ro/reactive-log-stream-processing-with-rxjava-part-2/) or go to the [Rx Playground](https://github.com/balamaci/rxjava-playground) for some **more examples**.
