+++
author = "Serban Balamaci"
date = 2016-08-17T19:41:28Z
description = ""
draft = false
slug = "reactive-log-stream-processing-with-rxjava-part-2"
title = "Reactive log stream processing with RxJava - Part II"

+++


In the previous post we saw how we can add a push based solution(RabbitMQ) to our "ELK" stack and how we can connect from Spring to RabbitMQ and have the log events emitted as a reactive stream. 

#### Json file as the source of log events
Since our main target is to play around as easily as possible with RxJava operators, we'll simulate receiving the events from RabbitMQ by reading them from a json file instead.
This helps that we don't need to start the full stack of RabbitMQ+Logstash+Log emitter app(through docker-compose) to test our operator chain setup, and also having the same events being emitted in a deterministic manner helps us track down where we went wrong.


We'll be using Spring Profiles to switch between how the events are been generated so apart from the [AmqpSourceEmitterConfiguration](https://github.com/balamaci/blog-rxjava-rabbitmq/blob/master/src/main/java/com/balamaci/rx/configuration/AmqpSourceEmitterConfiguration.java) we'll have [JsonFileEmitterSourceConfiguration](https://github.com/balamaci/blog-rxjava-rabbitmq/blob/master/src/main/java/com/balamaci/rx/configuration/JsonFileEmitterSourceConfiguration.java) which pushes the json entries onto the **PublishSubject**(to be consistent with how the AMQP MessageListenerAdapter does it).

```java
@Configuration
@Profile("hardcoded-events")
public class JsonFileEmitterSourceConfiguration implements ApplicationListener<ContextRefreshedEvent> {

    @Bean
    public Receiver receiver() {
        return new Receiver();
    }

    @Bean(name = "events")
    public Observable<JsonObject> emitEvents(Receiver receiver) {
        return receiver.getPublishSubject();
    }

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        Receiver receiver = event.getApplicationContext().getBean(Receiver.class);
        startEmitting(receiver);
    }

    private void startEmitting(Receiver receiver) {
        PublishSubject<JsonObject> publishSubject = receiver.getPublishSubject();

        Supplier<Integer> waitForMillis = () -> 200; //force a little fixed delay between the events emitter.
        Executors.newSingleThreadExecutor() //we start a single separate thread to simulate the same conditions like  
            .submit(() -> produceEventsFromJsonFile(publishSubject, waitForMillis));
    }


    private void produceEventsFromJsonFile(Observer<JsonObject> subscriber, Supplier<Integer> waitTimeMillis) {
        JsonArray events = Json.readJsonArrayFromFile("events.json");
        events.forEach(ev -> {
            sleepMillis(waitTimeMillis.get());

            JsonObject jsonObject = (JsonObject) ev;
            log.info("Emitting {}", Json.propertyStringValue("message").call(jsonObject));

            subscriber.onNext(jsonObject);
        });
    }
}
```
so now we have a Spring Bean named "events" which is an Observable<JsonObject> - the main stream of events -.


and we can switch between profiles at start time 
```java
public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.setDisplayName("RxJava");

        context.getEnvironment().setActiveProfiles("hardcoded-events");
        //context.getEnvironment().setActiveProfiles("amqp");

        context.scan("com.balamaci.rx.configuration");
        context.scan("com.balamaci.rx.observable");
        context.refresh();
        context.start();
}
```
Until this point even if events are being emitted(since we're using a hot observable), nobody is yet subscribed so the events are just lost. 
 
 
Let's start by just displaying the failed login events and in the end show how many failed logins there were from each ip.

The events being emitted are just some simulated Login events that we load from [events.json](https://github.com/balamaci/rxjava-rabbitmq/blob/master/events.json) which look like:
```
{
    "appName": "elk-testdata",
    "level": "INFO",
    "logger_name": "ro.fortsoft.elk.testdata.generator.event.LoginEvent",
    "thread_name": "pool-3-thread-4",
    "message": "SUCCESS login for user\u003d\u0027giannaodom@yahoo.com\u0027",
    "remoteIP": "192.168.0.32",
    "userName":"giannaodom@yahoo.com",
    "@timestamp": "2016-06-30T11:33:40.872Z",
    "host": "172.17.0.4"
  },
  {
    "appName": "elk-testdata",
    "level": "ERROR",
    "logger_name": "ro.fortsoft.elk.testdata.generator.event.LoginEvent",
    "thread_name": "pool-3-thread-6",
    "message": "FAILED login for user\u003d\u0027perez@yahoo.com\u0027",
    "remoteIP": "192.168.0.105",
    "userName":"perez@yahoo.com",
    "@timestamp": "2016-06-30T11:33:40.874Z",
    "host": "172.17.0.4",
  }
```



```
@Configuration
public class LoginObservables {

    @Autowired
    @Qualifier("events")
    protected Observable<JsonObject> events;

    @Bean
    Observable<JsonObject> loginEvents() {
        return events
                .filter(checkPropertyFunc("logger_name", val -> val.contains("LoginEvent")));
    }

    @Bean
    Observable<JsonObject> failedLogins() {
        return loginEvents()
            .filter(Json.checkPropertyFunc("message", val -> val.startsWith("FAILED login")));
            .doOnNext((jsonObject) -> log.info("Debug {}", Json.propertyStringValue("messsage")));
    }

    @Bean
    public Subscription displayLogEventsSubscription() {
        failedLogins()
            .subscribe((jsonObject) -> log.info("Got {}", Json.propertyStringValue("messsage")));
    }    
}
```
Remember until we connect the "sink"(the subscriber) events are not traveling down the operators chain -the operator functions are not executed-.
We can prove this by adding **'onNext()'** - which is just a simple way to interpose an action in the chain, great for debugging purposes. 
We registered the subscriber by annotating @Bean the 'displayLogEventsSubscription' that method is invoked automatically by Spring. 
Without the Subscription bean the 'Debug' message is not seen.  

### Too many failed logins from the same ips
Now let's dive right in and try to see the cases where there are too many failed logins from a certain ip.
First we need a time range to measure the failed logins so we get a result of X failed logins / 10 secs.
We can use the **window** operator for this.

Operator **window(long timespan, TimeUnit unit)** splits the events stream into multiple windows of fixed time. It returns a stream of stream(Observable&lt;Observable&lt;T&gt;&gt;) each stream emitting events in it's timerange.
 

```java
Observable<Observable<JsonObject>> windowStream = failedLogins().window(30, TimeUnit.SECONDS);
```
Sidenote: You might wonder why not a single continuous stream as return value? Well, because that the window operator needs to signal **onComplete** after it's reached the specified time value, and the reactive streams specs say that you cannot emit new events after **onComplete**. 

There is also an overloaded version of **window(long timespan, long timeshift, TimeUnit unit)** - it's creating a new window with the specific timespan each 'timeshift' period. So in this case there may be overlaping windows. 

### Flatmap operator

Now the tricky part is that we need to register for events on each stream. This is where the **flatMap** operator comes in. 
The **flatMap** operator is like the swiss army knife, it can be used for all sorts of usecases. 
**1. The fork-join**. Consider the common case where for an event we need to make remote calls to other services. Each of those remote calls might return list of objects(but in the reactive streams world you might want to return Observable<T> instead of List<T>). 
So each event might start other streams and we register the downstream for notifications for each event.

**T --&gt; Observable&lt;T&gt;** 

This looks **similar to a fork-join** operation.

![flatMap](https://camo.githubusercontent.com/1db5afe0637db37bf24a0476deeb4a6af5f846a1/687474703a2f2f692e696d6775722e636f6d2f5a75364f5034442e706e67")
So the **3,2,1** events make remote calls that each return an Observable&lt;Integer&gt; while zs is the internal stream(the flatten operation) that downstreams operators subscribe on. 
We'll see later a concrete example of this(by making a rest call to mock REST services)
Sidenote: The thing to notice from the image is that it has no longer an order guarantee. The events from the merged streams appear as soon as they are emitted and so you might get 'mixed' events between the streams.
PS: Should we want to maintain order we can use **concatMap** like bellow:
![concatMap](https://camo.githubusercontent.com/2a578ecf4d5c6890f82ef27a5eb5330e3a613d4c/687474703a2f2f692e696d6775722e636f6d2f4b386a5147316e2e706e67)



**2.The stream 'walker'** - There is already a stream of streams who emit events, we want the downstream operators to react on events that appears on each stream. This is the usecase we need right now. As a rule of thumb when you have an Observable&lt;Observable&lt;T&gt;&gt; you probably need flatMap next.


```
Observable<Observable<JsonObject>> windowStream = failedLogins().window(30, TimeUnit.SECONDS);

Observable<JsonObject> windowEvents = windowStream.flatMap(jsonWindowEventObservable -> jsonWindowEventObservable); //notice we now have a single stream as return value
```

The statement
```
flatMap(jsonWindowEventObservable -> jsonWindowEventObservable)    
```
looks like it's not doing anything, but we just say that the map function should return our json event unchanged, but we now have the events from all the windows flattened into a single stream.

Since we want to count the number of failed logins per ip, we need the **groupBy** operator. It takes as parameter a function which gives the grouping key, in our case a function which returns the 'remoteIP' field in the json event for failed logins.
The **groupBy** operator returns a stream of streams, each new ip starts a new stream, while already "seen" ips emit events on their stream.

```
Observable<GroupedObservable<String, JsonObject>> groupedEventsPerIp = windowEvents.groupBy(Json.propertyStringValueFunc("remoteIP"));
```

192.168.0.11 -(#)--------(#)----->
192.168.0.68 ---------(#)-------->
192.168.0.88 -----(#)----------->

This is another reason why we first chose to use **window** on the initial stream of log events. 
Since the initial stream of potentially infinite, groupBy splits it into potentially infinite number of streams aka a recipe for OutOfMemory.
  
Now that we have to count the number of events per each ip-substream. We again use flatMap to be able to register a function for each substream perform some calculation and then "flatten" the result back to a single stream. 
```
.flatMap(groupEventForIP -> groupEventForIP
                                .count()
                                .map(failedLoginsCount -> {
                                    final String remoteIp = grouped.getKey();
                                    return new Pair<>(remoteIp, failedLoginsCount);
                                }))

```
The flatMap operator has received as a parameter a _groupEventForIP_ which is a GroupedObservable&lt;String, JsonObject&gt; (the string key being the remoteIP value) - so it's still a stream which means we can go ahead and apply stream operators on this substream-. 
Since we need to count how many failed logins there were, we just apply the **count()** operator which returns a number that we use to pass downstream a Pair&lt;remoteIP, failedLoginsCount&gt;. 

All together this looks like:  
```
    Observable<Pair<String, Integer>> failedLoginsPerWindow(int windowSecs) {
        return failedLogins()
                .window(windowSecs, TimeUnit.SECONDS)
                .flatMap(jsonWindowEventObservable -> jsonWindowEventObservable
                 .groupBy(Json.propertyStringValueFunc("remoteIP"))
                            .flatMap(groupEventForIP -> groupEventForIP
                                .count()
                                .map(failedLoginsCount -> {
                                    final String remoteIp = groupEventForIP.getKey();
                                    return new Pair<>(remoteIp, failedLoginsCount);
                                }))
                );
    }

    @Bean
    public Subscription failedLoginsPerIpSubscription() {
        return failedLoginsPerWindow(10) //10 secs window
                .filter(failedLoginPair -> failedLoginPair.getValue() > minFailedLoginAttempts)
                .subscribe((failedLoginPair) ->log.info("Possible brute force login: {}", failedLoginPair));
    }
```

#### Usecase for flatMap as the mentioned fork-join operator to do extra checks
We dealt with the bad guys, however we've yet to seen how we can deal with a scenario in which we need to run extra checks in remote services for certain events.
Let's invent some usecase.
Say that after login it would make sense to start some credit check for our user like verify he payed at least one bill or that he registered a valid mobile phone with us. Since these checks are usually slow(involving some legacy core business components) and change frequently, we don't want to put them in our frontend app to slow down the load time of pages.
Instead we can do these checks on our log stream processing machines and write the results in some "isAllowedToXXX" field. 
The frontend app would just check these fields before letting the user use services that have money costs. Since the checks logic is external, we can change it frequently without redeploying the frontend app.
The user would be free to use his account to look around and just force him to enter extra data whenever he attempts to actually use those money involving services.

This would look like:
```
public class UserService {
    public Observable<UserScoring> retrieveScoring(String username) {..}
}
```
Now we need another check to see how we can trigger them in parallel and still work with the result from both of them.  
Some sites when you travel to another country and see you login from an IP this country you've never logged from recently ask for some 2Factor authentication(like a sms code they send to your phone).  
This is a good idea since a login from a different country could mean someone else stole your credentials and is accessing your account. 
Now this is normally baked right in the authentication flow and not left to an external log event analysis tool, still it makes a good excuse to imagine it as our second extra check that the IP is from an user's country.  
```    
public class UserService {
    /** Returns  UserCreditScoring.CREDIT_CARD_STORED, MOBILE_PHONE_STORED, NONE;  */
    public Observable<UserCreditScoring> retrieveScoring(String username) { }
    
    /** Returns UserLocationRating.SAFE, SUSPICIOUS */
    public Observable<UserLocationRating> retrieveUserLocationBasedOnIp(String username, String ip) { }
}

```

So for every successful login we want to start these two checks and in the case both of them return something other than SAFE / CREDIT-CARD-STORED we mark the check for the frontend to ask extra info.

But first we need to modify
```
JsonObject(json successful login event) -> Pair<Username, IP>
```

We need a simple **map** operator. Compared to **flatMap**, the **map** operator just transforms T -&gt; X. 
```
Observable<Pair<String, String>> successLoginStream = 
    succesfullLogins()
                .map(jsonObject ->
                        new Pair<>(new Json(jsonObject).propertyStringValue("userName"),
                                new Json(jsonObject).propertyStringValue("remoteIP")))

```
and we got the two checks:
```
Observable<UserCreditScoring> scoringObservable = userService.retrieveScoring(username); 
Observable<UserLocationRating> locationObservable = userService.retrieveUserLocationBasedOnIp(username, ip);
```
We need to wait for the result of both scoring and location checks stream. 
You might remember the **zip** operator that groups together one by one events but since they are different types of events(UserCreditScoring and UserLocationRating) we need a way to merge them together. 

So we'll create a method that returns a Tuple(from [JavasLang](http://www.javaslang.io/) library) of the 3 values **&lt;username, UserCreditScoring, UserLocationRating&gt;**
```
private Observable<Tuple3<String, UserCreditScoring, UserLocationRating>> checkUserScoringAndLocationRating(String username, String ip) {
    Observable<UserCreditScoring> scoringObservable = userService.retrieveScoring(username);
    Observable<UserLocationRating> locationObservable = userService.retrieveUserLocationBasedOnIp(username, ip);

    Observable<Tuple3<String, UserCreditScoring, UserLocationRating>> scoringAndLocationObservable =
                scoringObservable
                        .zipWith(locationObservable, (scoring, location) -> new Tuple3<>(username, scoring, location));

    return scoringAndLocationObservable;
}
```
we passed to the **zip** operator a merging function to say how we want to combine the scoring and location. This function just creates a new Tuple 
```
(scoring, location) -> new Tuple3<>(username, scoring, location)
```
Remember that for **T -&gt; Observable&lt;Y&gt;** we need **flatMap** to do the fork-join logic(to spin off new streams and then rejoining the results)
```
succesfullLogins()
                .map(jsonObject ->
                        new Pair<>(new Json(jsonObject).propertyStringValue("userName"),
                                new Json(jsonObject).propertyStringValue("remoteIP")))
                .flatMap(pair -> checkUserScoringAndLocationRating(pair.getKey(), pair.getValue()))

```


### RxJava Schedulers

I'm gonna add some debugging statements and show the output to understand something else. Btw we also need to subscribe to the observable first
```
    private Observable<Tuple3<String, UserCreditScoring, UserLocationRating>> userScoringAndLocationForSuccessfulLogins() {
        return succesfullLogins()
                .map(jsonObject ->
                        new Pair<>(new Json(jsonObject).propertyStringValue("userName"),
                                new Json(jsonObject).propertyStringValue("remoteIP")))
                .doOnNext(userIpPair -> log.info("After map {}", userIpPair))
                .flatMap(pair ->
                                 checkUserScoringAndLocationRating(pair.getKey(), pair.getValue())
                                 .doOnNext(tuple3 -> log.info("Tupple {}", tuple3)) //-->> the extra debug statement
                        );
    }


```


```
[json-hardcoded-file1] INFO JsonFileEmitterSourceConfiguration - Emitting SUCCESS login for user='mendoza@yahoo.com'
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Tupple (mendoza@yahoo.com, MOBILE_PHONE_STORED, SAFE)
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Subscriber got (mendoza@yahoo.com, MOBILE_PHONE_STORED, SAFE) ****
[json-hardcoded-file1] INFO JsonFileEmitterSourceConfiguration - Emitting FAILED login for user='perez@yahoo.com'
[json-hardcoded-file1] INFO JsonFileEmitterSourceConfiguration - Emitting SUCCESS login for user='powell@yahoo.com'
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Tupple (powell@yahoo.com, MOBILE_PHONE_STORED, SAFE)
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Subscriber got (powell@yahoo.com, MOBILE_PHONE_STORED, SAFE) ****
[json-hardcoded-file1] INFO JsonFileEmitterSourceConfiguration - Emitting SUCCESS login for user='walton@gmail.com'
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Tupple (walton@gmail.com, NONE, SAFE)
[json-hardcoded-file1] INFO SuccessfulLoginObservable - Subscriber got (walton@gmail.com, NONE, SAFE) ****
```
The thing to notice is that all the operations are happening on 'json-hardcoded-file1' which is the same thread on which we read the json events so if we'd be doing some slow processing with the events we'd keep all this thread blocked and slow down receiving of events.
Can we maybe make part of the work to run in parallel?

RxJava provides some high level concepts for concurrent execution, like ExecutorService we're not dealing with the low level constructs like creating the Threads ourselves. Instead we're using **Schedulers** which create Workers who are responsible for scheduling and running code. By default RxJava will not introduce concurrency and will run the operations on the subscription thread.

There are two methods through which **we can introduce Schedulers** into our chain of operations: **subscribeOn** and **observeOn**.
	
   - **subscribeOn** acts upon the creation of events part -> at Observable.create and it will use the changed thread for the downstream operations. 

```java
	Observable<Integer> observable = Observable
                .create(subscriber -> {
                    /** some possible slow network op**/
                    log.info("Started emitting");
                    subscriber.onNext(1);

                    subscriber.onCompleted();
                });
		
	observable
//                .subscribeOn(Schedulers.computation()) //--> changes the thread where create above runs 
                .map(val -> {
                    int newValue = val * 2;
                    log.info("Mapping new val {}", newValue);
                    return newValue;
                })
                .subscribe(val -> log.info("Subscribe received " + val));
```
Short recap: **Observable are lazy**(the code inside the first line it's not executed) until we call **observable.subscribe()** and then the thread which called **subscribe()** will be used to run the code inside **create** along with the slow network op unless ofcourse we specified another scheduler with **subscribeOn** to execute it.

with the subscribeOn line commented out we have:
```
[main] INFO SchedulerTests - Starting
[main] INFO SchedulerTests - Started emitting
[main] INFO SchedulerTests - Mapping new val 2
[main] INFO SchedulerTests - Subscribe received 2
```

while with subscribeOn enabled:

```
[main] INFO SchedulerTests - Starting
[RxComputationScheduler-1] INFO SchedulerTests - Started emitting
[RxComputationScheduler-1] INFO SchedulerTests - Mapping new val 2
[RxComputationScheduler-1] INFO SchedulerTests - Subscribe received 2
```

Since subscribeOn is related to which thread the **Observable.create** method is invoked on - and not with the code inside **.subscribe(val -> { ... })** part(which is the last operation in our chain), it doesn't actually matter where we call **subscribeOn** in our method chain, neither does it matter if we call it twice(just the first one will count). And very important in our case **calling subscribeOn for Subjects is pointless** since hot observables are already pushing events on some thread even without an active subscriber registered.

   - **observeOn** relates to how subsequent operators are being executed.

```
	Observable<Integer> observable = Observable
                .create(subscriber -> {
                    
                    log.info("Started emitting");
                    subscriber.onNext(1);

                    subscriber.onCompleted();
                })
                .observeOn(Schedulers.computation())  //--> change subsequent operations thread
                .map(val -> {
                    int newValue = val * 2;
                    log.info("First mapping new val {}", newValue);
                    return newValue;
                })
                .observeOn(Schedulers.io())          // --> change it again
                .map(val -> {
                    String newValue = "*" + val + "*";
                    log.info("Second mapping new val {}", newValue);
                    return newValue;
                })
                .subscribe(val -> log.info("Subscribe received " + val));

```
we get 
```
12:49:31 [main] INFO SchedulerTests - Started emitting
12:49:31 [RxComputationScheduler-1] INFO SchedulerTests - First mapping new val 2
12:49:31 [RxIoScheduler-2] INFO SchedulerTests - Second mapping new val *2*
12:49:31 [RxIoScheduler-2] INFO SchedulerTests - Subscribe received *2*
```
because the subscription happened on the 'main' thread, while before each operator we changed the Scheduler.

RxJava provides some **out of the box Schedulers**:

   - **Schedulers.newThread()** - The scheduler just starts a new thread every time it is requested via subscribeOn() or observeOn(). Not usually a good choice since threads are not reused, and a new one is always created.
   - **Schedulers.io()** - as the name implies it's supposed to be used for blocking network/disk code. Nothing really magical, it's a threadpool with an unbounded number of threads. Threads are being reused, but since this will be a common pool(just as using the same ForkJoinPool for all calls to Streams.parallel() is a bad idea), it's probably good to be using it sparsely.
   - **Schedulers.computation()** - should be used on operations that require computational power and have no blocking code. It's limiting the pool size with the number of cores so there are no more concurrent threads running than CPU cores and all other requests are queued up until a thread becomes available.
   - **Schedulers.from(Executor executor)** -  provides a Scheduler based on an Executor.

So back to our code, we just need to mention a Scheduler upfront and the subsequent operations will use a worker from it.
```
    @Bean
    Observable<JsonObject> loginEvents() {
        return events
                .observeOn(Schedulers.io())
                .filter(checkPropertyFunc("logger_name", val -> val.contains("LoginEvent")));
    }
```


Next we're going to see how we can call a REST service the reactive way to make good on our promise of less used threads for increased performance of many events handling.