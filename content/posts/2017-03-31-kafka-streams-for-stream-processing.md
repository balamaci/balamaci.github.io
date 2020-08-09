+++
author = "serban.balamaci"
date = 2017-03-31T13:46:34Z
description = "Using Kafka-Stream for stream processing"
draft = false
slug = "kafka-streams-for-stream-processing"
title = "Kafka Streams for Stream processing"

+++


### A few words about how Kafka works.
At it's base, Kafka has the distributed log concept. By log we understand an immutable(append only) data structure. 
![Kafka Log](https://kafka.apache.org/0102/images/log_consumer.png)


So a **Producer**(or more) **append** entries(**never overwrite or delete existing data**) at the end of the data structure, while any number of **Consumers**  can read from it at their own pace by each keeping track of the offset from where they last  read and advance to the next records and so on.

A **Topic is like a category** "/orders", "/logins", a feed name to which **Producers** can write to and **Consumers** to read from.

But **would Kafka be so fast if multiple users would have to synchronize to append after each other to the same Topic?**
Well sequential writes to the filesystem are fast, but a very big performance boost comes from the fact that **Topics can be split into multiple Partitions which can reside on different machines**. 
So multiple **Producers can write to different Partitions** of the same Topic. 

**Partitions can be distributed on different machines in a cluster** in order to achieve high performance with horizontal scalability. 

![Topic with partitions](https://sookocheff.com/post/kafka/kafka-in-a-nutshell/log-anatomy.png)

Notice how in the image above Partition 1 seems to have fewer entries than the others. 
That's because multiple producers can write to different partitions(and maybe different machines) to achieve high throughput. 
**Order is maintained only in a single partition**, producers write at their own pace so **order of events cannot be guaranteed across partitions**(even of the same topic). Luckily there is a way to implement **custom partitioning Strategy** so the messages goes to the same partition based on the data. 
This means we can for example have all the events of a certain 'userId' to be sent to the same partition. Also a 'Round Robin' partition strategy can be chosen so that the data is evenly distributed across partitions.

On the other hand multiple producers can write to the same topic partition, but there are no guarantees that messages will not be intermixed.**There is no locking concept** like a producer to be blocked waiting for the other producer to finish writing a batch of messages and messages sent to a topic partition will be appended to the commit log in the order they are sent.     

#### Kafka consistency and failover
Each node in the cluster is called a **Kafka Broker**.

**Each partition can be replicated across multiple Kafka broker nodes to tolerate node failures.**
One of a partition's replicas is chosen as leader, and **the leader handles all reads and writes of messages in that partition**. 
Writes are serialized by the leader and synchronously replicated to a configurable number of replicas(the number of replicas can be set on a topic-by-topic basis). 
On leader failure, one of the in-sync replicas is chosen as the new leader.

![Partition leader](https://sookocheff.com/post/kafka/kafka-in-a-nutshell/producing-to-partitions.png)


Another partition can be leader on another broker. 
![Another partition is leader](https://sookocheff.com/post/kafka/kafka-in-a-nutshell/producing-to-second-partition.png)


More about failover and replication [here](https://kafka.apache.org/documentation.html#replication) or [here](https://www.confluent.io/blog/hands-free-kafka-replication-a-lesson-in-operational-simplicity/)

A message is considered "committed" when all in sync replicas for that partition have applied it to their log. 
**Only committed messages are ever given out to the consumer.**


### Setting up a Kafka
We can use a [Kafka docker image](https://wurstmeister.github.io/kafka-docker/). It already has a detailed running guide, so no need to discuss it again.

So **when creating a Topic we need to specify in how many partitions we want to split it and how many replicas we want**.

```
$KAFKA_HOME/bin/kafka-topics.sh --create --topic userLogins --partitions 4 --zookeeper zkEndpoint --replication-factor 2
```

Zookeper is a key part of the cluster alonside the Kafka brokers it's role being that of leader election for partitions when one of the brokers is failing and keeps track of which brokers are alive.

### Writing application log events to Kafka
To write data into Kafka it's pretty simple using the Java Kafka  client api. A record / message consists of a **Key** and **Value**.
**Keys** play a role into assigning the topic partition(the default Kafka Producer hashes the key and sends the record always to the same partition for the same hash). The **Key** can be null, in this case a round-robin algorithm is used.

```
  Properties props = new Properties();
  props.put("bootstrap.servers", "localhost:9092");
  props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, LongSerializer.class.getName());
  props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

  Producer<String, String> producer = new KafkaProducer<>(props);
  for(int i = 0; i < 100; i++) {
      Future<RecordMetadata> responseFuture = producer.send(new ProducerRecord<String, String>("my-topic", "KEY-" + Integer.toString(i), "VALUE-" + Integer.toString(i)));
  }
```
the writing is async as the _send()_ method returns a Future. We can do _future.get()_ to turn it into a blocking one where we wait for the response.

There is another overloaded _send()_ method which takes a callback to be invoked if the message was written to Kafka or if an exception was encountered:
```
producer.send(new ProducerRecord<String, String>("my-topic", "KEY-" + Integer.toString(i), "VALUE-" + Integer.toString(i)), new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                long partition = metadata.partition(); //the partition where the message was written
            }
        });
```
The response object **RecordMetadata** has the 'partition' where the record was written and the 'offset' of the message.

For high throughput, the Kafka Producer allows to wait and buffer for multiple messages and send them as a batch with fewer network requests.
```
//batch.size in bytes of h, set it to 0 to disable batching altogether
props.put("batch.size", 16384);

//how much to wait for other records before flushing the batch for network sending
props.put("linger.ms", 5);

//buffer.memory - the amount of memory
//important as if network sending cannot keep up(or if the browkers are down), it will block
props.put("buffer.memory", 33554432);

//we can control how much time it will block before sending throws a BufferExhaustedException
props.put("max.block.ms", 10);

try {
    Future<RecordMetadata> responseFuture = producer.send(new ProducerRecord<String, String>("my-topic", ..));
} catch (BufferExhaustedException e) { ... }
```

But following on the steps of the blog other entries on  application logs processing we're going to use simple json log entries submitted by a **Logback Appender** directly into Kafka.
Notice how with &lt;producerConfig&gt; we can pass the producer config we explained above.
```
<appender name="STASH" class="com.github.danielwegener.logback.kafka.KafkaAppender">

    <encoder class="com.github.danielwegener.logback.kafka.encoding.PatternLayoutKafkaMessageEncoder">
       <layout class="net.logstash.logback.layout.LoggingEventCompositeJsonLayout">
          <providers>
             <mdc/>
             <context/>
             <version/>

             <logLevel/>
             <loggerName/>

             <pattern>
                <pattern>
                  {
                    "appName": "testdata",
                    "appVersion": "1.0"
                  }
                 </pattern>
             </pattern>
             <threadName/>

             <message/>

             <logstashMarkers/>
             <arguments/>

             <stackTrace/>
          </providers>
       </layout>
    </encoder>

    <!--The Kafka topic to write to -->
    <topic>logs</topic>


    <!-- delivery strategy that uses the producer.send() method with callback -->
    <deliveryStrategy class="com.github.danielwegener.logback.kafka.delivery.AsynchronousDeliveryStrategy" />
    <!-- synchronous strategy does a Future<>.get() after the .poll()-->

    <!-- each <producerConfig> translates to regular kafka-client config (format: key=value) -->
    <!-- producer configs are documented here: https://kafka.apache.org/documentation.html#newproducerconfigs -->
    <!-- bootstrap.servers is the only mandatory producerConfig -->
    <producerConfig>bootstrap.servers=${KAFKA_BOOTSTRAP_SERVERS}</producerConfig>


    <!-- RoundRobinKeyingStrategy just returns null for the record KEY, thus RoundRobin-->
    <keyingStrategy class="com.github.danielwegener.logback.kafka.keying.RoundRobinKeyingStrategy" />

</appender>
```
We can control explicitly that related events go to the same partition by providing consistent keys for the log events.
Remember that the [Kafka default partitioner](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java) uses hash of the keys to choose the partitions, or a round-robin strategy if the key is null.

For example having the Thread's name hashcode used as the key is already provided by the logback-kafka library. This means that log events produced :
```java
public class ThreadNameKeyingStrategy implements KeyingStrategy<ILoggingEvent> {

    @Override
    public byte[] createKey(ILoggingEvent e) {
        return ByteBuffer.allocate(4).putInt(e.getThreadName().hashCode()).array();
    }
}
```

A custom Partitioner can be set in the Producer properties if for some reason you don't want to base on the keys:
```
public class KafkaCustomPartionStrategy implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        int partitions = cluster.availablePartitionsForTopic("logs").size();
        String json = new String(valueBytes, StandardCharsets.UTF_8);
        ...
        return partitionNumber;
    }
}
//and set it on the Producer
<producerConfig>partitioner.class=ro.fortsoft.kafka.partitioner.KafkaCustomPartionStrategy</producerConfig>
```

### Consumer groups
Consuming Kafka messages is more interesting as we can start multiple instances of consumers. Now the problem arise **how the topic partitions are to be distributed** so multiple consumers can work in parallel and collaborate to consume messages, scale out or fail over.
This is solved in Kafka by marking the consumers with a common **group identifier**(a string)  that they belong to the same **Consumer Group** while another **group identifier** would mean another Consumer Group who want to process the messages in the same topic for some other usecase.
For each consumer group, one of the brokers is selected as the **group coordinator - with the job to assign partitions when new members arrive, or reassign partitions when old members depart or topic metadata changes-**.
![Consumer group](https://cdn2.hubspot.net/hubfs/540072/New_Consumer_figure_1.png)

It's irrelevant if by Consumers we mean multiple separate instances(processes) of our consumer app residing on the same machine or on different machines. We can have multiple consumers even in the same process by spawning threads(each consumer in it's own thread and each assigned a different partition).

Kafka brokers keep tracks of the **offset**(position) of the consumed messages in a topic **partition** for each Consumer Group. **The messages in each partition log are then read sequentially**.
As the consumer makes progress, it **commits the offsets** of messages it has successfully processed.
The **commit offset** message from the consumer can come later after a larger batch of messages is processed. This helps with performance.

![Consumer offset](https://cdn2.hubspot.net/hubfs/540072/New_Consumer_Figure_2.png)


Should the consumer fails before being able to send the **commit offset XXX** to the Kafka broker, on restart(or reallocation ), **the new Consumer instance will continue from the last committed offset that the broker has**, meaning it will reprocess some of the messages(this is 'at least once' behavior).

For ex. in the image above the Consumer has received messages up to the current position(offset 6). Should it suddenly crash, then the group member taking over the partition would begin consumption from offset 1. In that case, it would have to reprocess the messages up to the crashed consumer’s position of 6.
The **"Log end offset"** is the offset of the last message written to the log and where Producers will append next.
The **"High watermark"** is the offset of the last message that was successfully copied to all of the log’s replicas. From the perspective of the consumer, it can only read up to the high watermark. This prevents the consumer from reading unreplicated data which could later be lost.

For this to work though, it means that **we can't have more that a single consumer(from the same group) allocated on the same partition**.
So we have the following behaviors when starting up Consumers in relation to number of Partitions:

- **If the number of Consumer Group instances is more than the number of partitions, the excess nodes remain idle**. This might be desirable to handle failover.

- **If there are more partitions than Consumer Group instances, then some nodes will be reading from more than one partition.** Like in Fig. 1 above.

### Consumer code
The essence of the consumer is that after subscribing to a topic, we start an event loop to get a partition assignment and begin fetching data.
```
try {
  while (running) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records)
        log.info("Partition={}, Offset={}, Val={}", record.partition(), record.offset(), record.value());
  }
} finally {
  consumer.close();
}
```
 - The **poll()** method returns fetched records based on the current offset for the partition. It is a blocking method waiting for the specified time if there are no records available. If there are records available, the method returns immediately.
 - We can control the maximum records returned by the **poll()** with a Consumer property.
 - When the group is first created, **the read position will be set according to the reset policy** (**"earliest"** or **"latest"** offset for each partition)
 - The consumer api is designed to be run in it's own thread(we shouldn't call poll() from multiple threads in hope of a performance gain).

Let's see a full scenario where we start multiple Consumers of the same group and see how the partitions are split and allocated by the partition coordinator and how a standby consumer will take over. We'll also chose to commit the offsets ourselves.

It might not be evident from above ex., but **we can consume with the same Consumer from multiple Topics**.

```java
public class SimpleConsumer implements Runnable {
    private final KafkaConsumer<String, String> consumer;
    private final List<String> topics;
    private final int id;

    private static final Logger log = LoggerFactory.getLogger(SimpleConsumer.class);

    public SimpleConsumer(int id, List<String> topics, Properties consumerProps) {
        this.id = id;
        this.topics = topics;
        this.consumer = new KafkaConsumer<>(consumerProps);
    }

    @Override
    public void run() {
        try {
            consumer.subscribe(topics);
            log.info("Consumer {} subscribed ...", id);

            while (true) {
                log.info("Consumer {} polling ...", id);
                ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
                log.info("Received {} records", records.count());

                for (TopicPartition topicPartition : records.partitions()) {
                    List<ConsumerRecord<String, String>> topicRecords = records.records(topicPartition);

                    for (ConsumerRecord<String, String> record : topicRecords) {
                        log.info("ConsumerId:{}-Topic:{} => Partition={}, Offset={}, EventTime:[{}] Val={}", id,
                             topicPartition.topic(), record.partition(), record.offset(), record.timestamp(),
                             record.value());
                    }

                    long lastPartitionOffset = topicRecords.get(topicRecords.size() - 1).offset();
                    consumer.commitSync(Collections.singletonMap(topicPartition,
                            new OffsetAndMetadata(lastPartitionOffset + 1)));
                }
            }
        } catch (WakeupException ignored) {
            // ignore for shutdown
        } catch (Exception e) {
            log.error("Consumer encountered error", e);
        } finally {
            consumer.close();
        }
    }

    public void shutdown() {
        //trigger a
        consumer.wakeup();
    }
}
```
We can wait indefinitely in the **poll(Long.MAX_VALUE)** method, but we can break by calling from another thread **consumer.wakeup()** which **throws WakeupException** if thread was blocked on polling.

To commit the offset, we don't do it for each message, but instead we commit the whole batch by committing the offset of the last message for each topic we received records.(OffsetAndMetadata - metadata is just a string with which you can enhance the commit)

Since consumers are single threaded we'll need to create a pool of threads for each one. Let's see how if we use a single topic 'logs' with 2 partitions, how are the partitions assigned:

```java
public static void main(String[] args) {
    int numConsumers = 3;

    List<String> topics = Arrays.asList("logs");
    ExecutorService executor = Executors.newFixedThreadPool(numConsumers);

    final List<SimpleConsumer> consumers = new ArrayList<>();
    for (int i = 1; i <= numConsumers; i++) {
         SimpleConsumer consumer = new SimpleConsumer(i, topics, properties(CONSUMER_GROUP_ID));
         consumers.add(consumer);
         executor.submit(consumer);
    }

    //Register shutdown hook that will signal consumer threads to break from blocking poll()
    registerShutdownHook(executor, consumers);
}

private static Properties properties(String groupId) {
    Properties props = new Properties();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "host1:9092,host2:9092");
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "simple-logs-processing");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "5");

    return props;
}
```
 - ConsumerConfig.GROUP\_ID\_CONFIG sets the group id - "simple-logs-processing". This is how Kafka knows to assign the partitions according to Consumers that have identified themselves as part of the same group. It's also how Kafka knows what was the last commit offset for this consumer group. Change the group id and Kafka will tell the consumer to start over with reading records from the beginning or the end according to the AUTO\_OFFSET\_RESET\_CONFIG policy bellow.
 - ConsumerConfig.AUTO\_OFFSET\_RESET\_CONFIG - ("earliest" or "latest") initially when the consumer group connects for the first time, there is no commit offset so using this we say whether we want to start consuming from the beginning or from the end and listen for new appended records.
 - ConsumerConfig.MAX\_POLL\_RECORDS\_CONFIG - How many records a call to poll() will return in a single call.
 - ConsumerConfig.ENABLE\_AUTO\_COMMIT\_CONFIG "false". By setting it to "true" instead, we could have chosen to let the consumer commit the records periodically with the value set by ConsumerConfig.AUTO\_COMMIT\_INTERVAL\_MS\_CONFIG

#### 2 partitions, 1 consumer
The Kafka broker group coordinator will assign both the 2 partitions("logs-0" and "logs-1") to the single consumer which will receive one batch of records from "logs-0" and another batch from "logs-1".

#### 2 partitions, 3 consumers.
The Kafka broker group coordinator will assign the 2 partitions of "logs" topic("logs-0" and "logs-1") to 2 consumers, while 3rd consumer is blocked waiting on poll() as a standby consumer.
In [SimpleConsumer](https://github.com/balamaci/blog-kafka-streams/blob/master/src/main/java/com/balamaci/kafka/consumer/SimpleConsumer.java) we simulated that Consumer1 crashed with uncommitted records. The Kafka brokers detect the disconnect and reassign the partition to the 3rd consumer which continues processing from the last committed offset.
Sidenote: Kafka group coordinator listen for a hearbeat from the Consumers that is sent when **.poll()** method is called, but if you busy processing the records received from previous poll, it might take longer. The parameter to set this is ConsumerConfig.SESSION\_TIMEOUT\_MS\_CONFIG(default 30 secs) but setting the value too high will take more time for Kafka to declare the consumer dead and initiate a rebalance.

#### Start consuming from a certain offset
It's also possible to go back to previous records and a certain offset by using the **seek()** method of the consumer.

We can intervene right when the partition is assigned by the Kafka partition owner to the Consumer. This way we also show another useful feature - a Listener for when a partition is assigned to the Consumer-.
```
private class CustomOffsetsRebalanceListener implements ConsumerRebalanceListener {

    private long offset;

    public CustomOffsetsRebalanceListener(long offset) {
         this.offset = offset;
    }

    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        for(TopicPartition partition: partitions) {
             log.info("*** Partitions revoked {}", partition);
        }
    }

    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        for(TopicPartition partition: partitions) {
              log.info("*** Partitions assigned {}", partition);

              consumer.seek(partition, offset);
        }
    }
}

....
//and register the listener
consumer.subscribe(topics, new CustomOffsetsRebalanceListener(startOffset));

//now the poll will return
ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
...
```

The method **consumer.seekToBeginning()** and **consumer.seekToEnd()**.
Kafka log have a retention period(which can be set differently per topic) after which the records are purged - this is why seekToBeginning() doesn't necesarely mean offset 0-.

We can implement **a reset tool** by identifying with the same group.id as your application, using consumer.seek(XXX) and then commit the offset for a certain topic or partition.


### Kafka Streams

#### Why can't we just use RxJava or Spring Reactor?
Before getting into Kafka Streams I was already a fan of RxJava and Spring Reactor which are great **reactive streams processing** frameworks. So why do we need Kafka Streams(or the other big stream processing frameworks like Samza)?
We surely can use RxJava / Reactor to process a Kafka partition as a stream of records. But dealing with large amounts of data in streams means we also probably need a storage system(preferably offheap -to not be impacted by GC-) to store partial computation  results and have those results persisted in a way that if one Processor dies another can takeover and continue from the partial result.
So for fault tolerance there is this need of combining **an operator** with some **state** that keeps what the Processor has parsed till a certain point, **state** that can be persisted and replicated to the failover processor.

RxJava / Reactor don't have any out-of-the-box solutions for this, they instead have a massive amount of operators like the common **filter**, **map**, **reduce**, **groupBy** and deal with a reactive way of processing the data **in the same java process**(scheduled on multi threads, but in the end inside a single java process).

On the other hand Kafka Streams knows that it can rely on Kafka brokers so it can use it to redirect the output of Processors(operators) to new "intermediate" Topics from where they can be picked up by a Processor maybe deployed on another machine, a feature we already saw when we talked about the **Consumer group** and the **group coordinator** inside Kafka broker itself.

Sidenote: [reactor-kafka](https://github.com/reactor/reactor-kafka) is a library which handles the Producer/Consumer scenario in a reactive way, it doesn't do "anything else" like store local state derived from stream processing.



#### Other streaming frameworks
There are many streaming frameworks that can work with events from Kafka. However most work and rely on YARN(so it requires a Hadoop cluster) or Mesos to assign the jobs, keep state and restart the failed processors.
![](https://www.madewithtea.com/images/streaming.png)

Kafka Streams instead is just a library(like rxjava / reactor) that you add to your project, it doesn't need a special runtime other than packing it inside a .jar and start it like a normal java process and distribute and start it on as many machines as needed.
It's **masterless** - it doesn't rely on a central node to detect failure of the processors and reelect a new master, etc.-, rather it uses Kafka's build in coordination mechanism to assign partitions to available processors to provide scalability, fault-tolerance and failover.
Looks like a pretty sweet deal on what it can do without being tied to a particular deploy architecture and cluster resources.

A more in-depth comparison with Flink [here](https://www.confluent.io/blog/apache-flink-apache-kafka-streams-comparison-guideline-users/).

### High level DSL and low-level API
Kafka Streams has two APIs we can use:

The **High level DSL**(the KStream API) allows us to write stream processing operators similar to what we are familiar with from Java Streams, Reactor, etc.

```java
KStream<Long, String> productViews = builder.stream("ProductViewsTopic");

productViews.groupByKey()
         .count("ProductViewCounts")
         .filter((Long pageId, Long viewCount) -> viewCount == 1000)
         .process(() -> new PopularProductEmailAlert("alerts@yourcompany.com"));

KafkaStreams streams = new KafkaStreams(builder, streamsConfiguration);
```

However this DSL is built on top of the **'low level' Processor API** which we can also use, although I think makes it clear how Kafka Streams is actually working, we probably use the DSL more frequently.

#### Low level processing API
We need to define a **processing Topology** - a graph of processors that will describe the processing "pipeline"-.

Implementing a Processor looks like :

```java
public class MyProductCountProcessor extends AbstractProcessor<Long, String> {
    private ProcessorContext context;

    private KeyValueStore<Long, Long> kvStore;

    public void init(ProcessorContext context) {
        // keep the processor context locally because we need it to forward results downstream and for commit
        this.context = context;

        // call this processor's punctuate() method every 1000 milliseconds.
        // as a way to do something periodically
        this.context.schedule(1000);

        // retrieve the key-value store named "Counts"
        this.kvStore = (KeyValueStore<Long, Long>) context.getStateStore("Counts");
    }

    @Override
    public void process(Long productId, String browser) {
        Long oldValue = this.kvStore.get(word);

        if (oldValue == null) {
             this.kvStore.put(word, 1L);
        } else {
             this.kvStore.put(word, oldValue + 1L);
        }
    }

    //This method will be called as a way to do something periodically
    @Override
    public void punctuate(long timestamp) {
        KeyValueIterator<Long, Long> iter = this.kvStore.all();

        while (iter.hasNext()) {
            KeyValue<String, Long> entry = iter.next();

            //forward the calculated values downstream
            context.forward(entry.key, entry.value.toString());
        }

        iter.close();
        // commit the current processing progress
        context.commit();
    }
}
```
Considering for now the kvStore as a generic key-value store I think the code is pretty self explanatory. The **punctuate** method will be executed periodically - with the frequency we set in **init()** call to context.schedule(1000) -. We send the data to the next Processor with **context.forward(entry.key, entry.value.toString())**.

A Topology can be described by adding processing nodes by linking a **Processor** to one or more parent **Processor(s)**, or **Source**(the topics). So you can imagine the Topology like a graph([DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)). 

```java
TopologyBuilder topologyBuilder = new TopologyBuilder();

StateStoreSupplier countStore = Stores.create("Counts")
         .withKeys(Serdes.Long())
         .withValues(Serdes.Long())
         .persistent()
         .build();

topologyBuilder.addSource("SOURCE", "logs"); //specify the source topic(s)

//addProcessor method takes (processor name, processor instance supplier, parent processor(s) name)
topologyBuilder.addProcessor("PRODUCT_COUNTER", MyProductCountProcessor::new, "SOURCE")

//we link one or more state store to the Processor
topologyBuilder.addStateStore(countStore, "PRODUCT_COUNTER");

topologyBuilder.addProcessor("PRINT", PrintProcessor::new, "PRODUCT_COUNTER");

//write to another Kafka topic "counter_topic"
topologyBuilder.addSink("SINK_COUNTER", "counter_topic",
                Serdes.Long().serializer(), Serdes.Long().serializer(), "PRODUCT_COUNTER");

KafkaStreams streams = new KafkaStreams(topologyBuilder, streamsConfiguration);
```
**Sink** is just a destination Kafka topic to where the result of the parent processor can be written. 

while streamsConfiguration is a Properties through you can set different configuration props:

```
final Properties streamsConfiguration = new Properties();

streamsConfiguration.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-example");
// Where to find Kafka broker(s).
streamsConfiguration.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,
        config.getString("kafka.bootstrap.servers"));

// Specify default (de)serializers for record keys and for record values.
streamsConfiguration.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG, Serdes.Long().getClass().getName());
streamsConfiguration.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
```
Code [here](https://github.com/balamaci/blog-kafka-streams/blob/master/src/main/java/com/balamaci/kafka/streams/StartLowLevelProcessor.java)

#### How it works
The **KafkaStreams** object defined above holds the Topology and runs one or more **StreamThread** instances(configurable by StreamsConfig.NUM\_STREAM\_THREADS\_CONFIG param).
Each **StreamThread** has an embedded Kafka **Consumer** and **Producer** that handles reading writing to Kafka.
In addition to specifying source topic(s), you also need to provide **Serdes** for the keys and values. A Serdes instance contains the **ser**ializer and **des**erializer needed to convert objects from and to byte arrays.

Since Kafka Streams through **StreamThread** just use normal **Consumer** with a group, Kafka will assign a topic partition to each running **StreamThread**.
We can also start different processes and by identifying with the same APPLICATION_ID will get a partition assigned or just have it on standby for failover purpose.

We introduced in the topology the **state store** 'Counts' to which the Processor has access.
As we said, it helps with fault tolerance of restarting but being local to the node it also has great performance and can be used to implement aggregations and even joins as we'll see further.

PS: Doing **streams.toString()** will print the topology and associated stores. Just that it matters when you call it as it takes some time for the partitions to be assigned to the new  consumers and it's dynamic in a sense that they can be reassigned if one goes down.

#### Fault-tolerant stream processing, local state stores
The persistent local state-store by default is implemented using [RocksDB](https://github.com/facebook/rocksdb). **RocksDB** is a fast and low-latency key-value storage that persists to disk(optimized for fast disks like SSDs) written in C++. It was opensourced by Facebook, based upon LevelDB(created by Google, but which doesn't seem to be maintained). It is "embedded" in the sense that although it's written in C, the Kafka Streams pom.xml has a dependency on the **rocksdbjni** jar  library which contains inside **.so** and **.dll**(.so for Linux, .dll for Windows) files with the compiled RocksDB and at start time we can configure the directory location where RocksDB will store its data files for each Kafka Streams instance.
But if the state store are files local to each process / node, how are they fault tolerant, how does a process started on another node communicate with our node to receive the current state?
This works because Kafka Streams library creates **for each state store a replicated changelog Kafka topic** in which it tracks any state updates that it did locally. **changelog topics** are topics where if we update the information for a certain key, only the last key value is kept. Although it still works like a normal topic with append only records, a Kafka **log compaction** process can run and purge "outdated" values for keys as to keep storage space and time to replay changes to a minimum.
So if we put together the state stores backed changelog topics with the committed offset of the records we processed, since Kafka topics can be consumed an arbitrary number of times, every time a processor node fails, a new processor can take over by restoring its local state database by replaying mutation logs from the changelog topic and resume processing from the committed offset.
There is a command **streams.cleanUp()**. This deletes any local state files and recreates the localstate from the backed changelog topic from Kafka.

It's worth considering that we don’t need an external distributed storage to have a state store for a Processor. Kafka and Kafka Streams are the only things we need.


#### Querying the state store
It's also possible to query the state stores so you can expose it maybe as a REST service.
```java
ReadOnlyKeyValueStore<Long, Long> countsStore = streams
                .store("Count",  QueryableStoreTypes.keyValueStore());
```
Still you need to realize that if a topic is split into more partitions that the local state store will only contain the values each instance of Kafka Streams is working on, not like a distributed global storage that all the processing nodes write to.
There are multiple partitions of the changelog topic that backs up the local state store and the number of partitions changelog topic is split into the number of partitions in the source topic, so we'll have "group-id-Count-changelog-0", "group-id-Count-changelog-1", etc.


#### Available operations
Getting back to the **high level** API, besides the usual **map**, **filter** which are stateless, there are stateful operators like **aggregations** that rely on state stores to keep previous process state, for example doing a **count** can be implemented as an aggregation which will rely on the local store to which we add 1 and store the new value. It's interesting that the result of an aggregation is a **KTable** which can be used to directly JOINed with another table by matching on their keys.

```java
KTable<String, Integer> browserActionsCountTable =
   browserActionsStream
    .map((key, jsonValue) -> {
       String browserHash = jsonValue.propertyStringValue("browserHash");

       return new KeyValue<String, Integer>(browserHash, 1);
    })
    .groupByKey(Serdes.String(), Serdes.Integer())
    .aggregate(() -> 0, (key, val, agg) -> (agg + 1), Serdes.Integer(), "browserActionsCountStore");
    //there is already a .count("storeName") method that does the same as above aggregation
```
Let's see how we can join them with the 

For ex. we had the number of actions(in a real-case scenario we'd probably want to do the count in a time-window) from a certain user's browser and we plan to ask the user to solve a captcha if we didn't ask him already.

To do this we first need to generate a table of browsers that passed the captcha and join the browserActionsTable with it.
We'll do a **browserActionsTable.leftJoin(captchaVerifiedTable, (valA, valB) -> )** to also return results for which we don't have any matching values in the table with the browsers that solved captcha challenge. These results that don't have a matching key will have a null value.

```java
//We construct the table of browser fingerprints for which the user successfully verified 
KTable<String, Integer> captchaVerifiedTable = kStream.mapValues(new JsonMapper())
     .filter((key, jsonValue) -> jsonValue.propertyStringValue("logger_name")
             .contains("BrowserCaptchaVerified"))
     .map((key, jsonValue) -> {
           String browserHash = jsonValue.propertyStringValue("browserHash");

           return new KeyValue<String, Integer>(browserHash, 1);
     })
    .groupByKey(Serdes.String(), Serdes.Integer())
    .reduce((val, val2) -> val, "captchaStore"); //we don't care about value,
    //we just want the browser fingerprint as the key in the KTable


KTable<String, Integer> browserActionsCountTable =
   browserActionsStream
    .map((key, jsonValue) -> {
       String browserHash = jsonValue.propertyStringValue("browserHash");

       return new KeyValue<String, Integer>(browserHash, 1);
    })
    .groupByKey(Serdes.String(), Serdes.Integer())
    .count("browserActionsCountStore");


//resulted table after the join
KTable<String, Long> captchaSolvedBrowserCountsTableJoined =
       browserActionsCountTable.leftJoin(captchaVerifiedTable, (actionsCount, captchaVal) -> {
          if(captchaVal == null) { //means we didn't have a matching value
             return actionsCount;  //in the solvedCaptcha table
          }

          //if the value was non-null means we have a validated browser
          //and we can return a small value
          return 0L;
       });
```

![Stream table duality](http://docs.confluent.io/3.0.0/_images/streams-table-duality-02.jpg)

**we can turn a KTable back into a Stream**. **Every new entry in the KTable or update means a new record forwarded downstream**.
So whenever a count is updated, or a new captcha validated browser is inserted, we get a chance to do something with it.
```java
captchaSolvedBrowserCountsTableJoined
    .toStream()
    .filter((browser, count) -> count >= 5)
    .foreach((k, v) -> log.info("Asking captcha for {}", k));
```

Still we need to be aware that each stream member node has just the data in the KTable for its assigned partitions(because the state store is also split into partitions as many as the source).
To help, a **GlobalKTable** construct was recently added by the Kafka streams team which has it's values replicated across processors, by having each stream member node consume on a separate thread the full changelog topic that backs the GlobalKTable.
Joining with a GlobalKTable gives you the extra option of providing the function that will produce the key lookup on which the join to the GlobalTable is performed.

```java
GlobalKTable<String, Integer> captchaTable = builder.globalTable(Serdes.String(), Serdes.Integer(),
                "captchaStream",
                "captchaStreamStore");

browserActionsCountTable
                .toStream()
                .leftJoin(captchaTable,
                        //we need to provide the function that builds the join key,
                        //in our case we just use return the same key which
                        (streamKey, streamValue) -> streamKey,
                        (actionsCount, captchaVal) -> {
                            if(captchaVal == null) {
                                return actionsCount;
                            }
                            return 0L;
                        })
                .filter((browser, count) -> count >= 5)
                .foreach((k, v) -> log.info("Asking captcha for {}", k));
```

#### Event time and Windowing
In a Kafka record besides the key-value data it's stored also a **timestamp** which if not specifically set when the record is sent to Kafka. It can be set to a specific value, or it will be the append time to Kafka log.
Interesting is that Kafka Streams use this record timestamp to provide us with time based calculations based upon the application time like 'number of events / 5 sec'.
```java
KTable<Windowed<String>, Long> browserActionsCountTable =
   browserActionsStream
    .map((key, jsonValue) -> {
       String browserHash = jsonValue.propertyStringValue("browserHash");

       return new KeyValue<String, Integer>(browserHash, 1);
    })
    .groupByKey(Serdes.String(), Serdes.Integer())
    .count(TimeWindows.of(5 * 1000L).advanceBy(2 * 1000L), "counterStore");

browserActions
   .toStream()
   .foreach((windowKey, count) -> {
       long windowStart = windowKey.window().start();
       long windowEnd = windowKey.window().end();

       log.info("Window [{}-{}] Browser:{} count={}", millisFormat(windowStart),
                          millisFormat(windowEnd), windowKey.key(), count);
   });


#Processing time                   #Kafka append time
15:50:23 [StreamThread-1] - Window [14:11:52-14:11:57] Browser:XXXYXXXY0 count=1
15:50:23 [StreamThread-1] - Window [14:11:54-14:11:59] Browser:XXXYXXXY0 count=1
15:50:23 [StreamThread-1] - Window [14:11:52-14:11:57] Browser:XXXYXXXY4 count=1
15:50:23 [StreamThread-1] - Window [14:11:54-14:11:59] Browser:XXXYXXXY4 count=1
15:50:23 [StreamThread-1] - Window [14:11:52-14:11:57] Browser:XXXYXXXY2 count=1
15:50:23 [StreamThread-1] - Window [14:11:54-14:11:59] Browser:XXXYXXXY2 count=1
15:50:23 [StreamThread-1] - Window [14:11:52-14:11:57] Browser:XXXYXXXY3 count=2
15:50:23 [StreamThread-1] - Window [14:11:54-14:11:59] Browser:XXXYXXXY3 count=6
15:50:23 [StreamThread-1] - Window [14:11:56-14:12:01] Browser:XXXYXXXY3 count=6
15:50:23 [StreamThread-1] - Window [14:11:58-14:12:03] Browser:XXXYXXXY3 count=3
15:50:23 [StreamThread-1] - Window [14:11:56-14:12:01] Browser:XXXYXXXY4 count=1
...
```
The code above will generate Windows(startTime:endTime) of 5 seconds length in which the count is calculated, every 2 seconds. So these are overlapping windows of 5 secs calculated every 2 seconds.

Having the timestamp with the record is useful in windowing operations, because if you have already existing data trying to look for suspicious calls / 5 sec, Kafka Streams being fast will maybe be able to crunch through some days of log data in 5 sec and give misleading results.
Should you want to just ignore the record timestamp and just use the processing time, it's also possible:

```java
streamsConfiguration.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG, WallclockTimestampExtractor.class);
```
or even implement your own TimestampExtractor.

#### Intermediate topics and Backpressure
One of the big concepts with reactive stream processing frameworks is **backpressure**, the possibility of the consumers and processors downstream to signal upstream to the producer that it cannot process anymore data and that it should slow the amount of data it sends downstream. This is done in RxJava / Spring Reactor by the subscriber requesting x records from upstream on subscribing and requesting another batch of x or y after it manages to fully or partially process what it received.

Kafka-Streams being **pull based** (with the poll() method we saw from the Consumer implementation being invoked after the received records are processed) should not have such a problem. Once a record is retrieved from Kafka it will be **forward**ed to all the downstream operators before requesting more. Still, there might be different processing "speeds" in different stages in processing topology like for example having to lookup some ids in a slower DB might mean. We can isolate the problem by writing the results back to Kafka in another Topic(which we can configure with a higher number of partitions) so they can be assigned to other instances of KafkaStreams.


```
...
stream1.to("my-topic");
stream2 = builder.stream("my-topic");
```

This leads to interesting situations where parts of the Topology will run only on some instances of your application, while "on standby" on other instances which run other operators of the Topology sent through the intermediate topic.



#### Conclusion
I think Kafka Streams has a strong position because it provides a low barrier entry by being just a Java library and not having to depend on a cluster management solution, however it does come with a cost of having to understand some of its limitations like the local state store only having so it does force you to think about Key-ing strategies so that related entities end up on the same processor.
If we need to slice & dice and work with some clear key-value joins between data, it can do the job very well. Should we want something like an SQL dialect for our table joins, we should go to the more complex(and complicated) solutions like Spark or Flink.

### Sources
https://github.com/balamaci/blog-kafka-streams
https://github.com/confluentinc/examples/tree/3.1.x/kafka-streams/src/main/java/io/confluent/examples/streams

### Reference
https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/ 

http://www.javaworld.com/article/3066873/big-data/big-data-messaging-with-kafka-part-2.html

http://docs.confluent.io/3.1.2/clients/consumer.html#java-client

http://docs.confluent.io/3.0.0/streams/index.html 
