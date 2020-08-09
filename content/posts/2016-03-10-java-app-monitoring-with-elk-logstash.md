+++
author = "Serban Balamaci"
categories = ["elk"]
date = 2016-03-10T19:53:21Z
description = ""
draft = false
slug = "java-app-monitoring-with-elk-logstash"
tags = ["elk"]
title = "Java app monitoring with ELK - Part I - Logstash and Logback"

+++


**Logs for developers are undeniably the most important source of information** available to track down problems and understand what is happening with your applications.

It makes sense to have a good tool in our toolbox that will enable us to get better insight of this data.

### There is gold in them logs!
People mainly start using the ELK stack as a way to be able to store and search distributed logs through a web interface. This is reason enough to add the ELK stack to your projects, but you might not have known **there is also an analytics side to the ELK stack that can be used for processing the data in your logs and this is very useful for monitoring or investigations**. Take for example:

  - **We could count the numbers of errors of a certain type but grouped by unique users**. Is it the same user that tries and gets the same error over and over or it's a more general error that other users also encounter? 
  - Count the number of events of a certain type to determine the numbers of user that were affected by an issue or if the application is exploited by a spammer. 
  - Measure the impact after you made a change in your system. 

Basically the examples of how useful ELK stack can be are limitless.

The good part is this can actually be obtained without any or minimal change in your business code.

We first discuss what the ELK stack is about and what exactly each of it's components does. We're taking advantages of using available Docker images to get around doing the sysadmin work of and rather focus on how we can use it.

We will then generate and feed random log entries of failed login attempts to the ELK stack and using [ElastAlert](https://github.com/Yelp/elastalert) to get notifications when we detect too many login attempts.


### ELK stack
![ELK](/assets/2016/elkstack.png)
The ELK stack consist of 
  - **L**ogstash which serves as a data ingestion engine and can receive(through different plugins) data from lots of custom data sources, manipulate it and sending out to either storage in ElasticSearch or to any other system.    
  - **E**lasticSearch is where the data gets stored and queried. ES provides full text search technology built on top of Lucene and also analytics.
It is distributed by default and easy to scale. 
  - **K**ibana is a visualisation platform for running searches in ElasticSearch. It also allows to run analytics and charting the data and organize the charts into dashboards.

The ELK stack is pretty famous by now, and you may have encountered lots of variations to it. Don't let that discourage you, once you understand what each component is supposed to do you can find the alternatives that best suit you setup. For ex. replacing Logstash in favor of FluentD and you now have [EFK stack](https://geowarin.github.io/spring-boot-logs-in-elastic-search.html)
    

### Logstash
Logstash can play the part of a central controller and router for log data. 
It can collect logs from a variety of sources (using various **[input plugins](https://www.elastic.co/guide/en/logstash/current/input-plugins.html)**), process the data into a common format by using **filters** and stream that data to a variety of endpoints (using **output plugins**). These make up the [Logstash Processing Pipeline](https://www.elastic.co/guide/en/logstash/current/pipeline.html).

![LogStash](https://www.elastic.co/guide/en/logstash/current/static/images/logstash.png)

Logstash allows the configuration of each of the parts of the pipeline **input** - **filter** - **output** by writing the **logstash.conf** file.
 
#### input
Logstash can receive the data through external plugins from a multitude of sources, some common like 'file', 'tcp/udp' but also some more special like Kafka topics or ZeroMQ. This is an example config for the "input" phase where we configure the port where we are listening on and also that we expect to receive the log lines already in a json key:value format

```
input {
  tcp {
   port => 5000
   codec => "json"
  }
}
```


#### filter
Filters are the real processors of log lines. They can parse log entries taking a meaningless stream of text and turn it into structured entries with separate fields. For ex. say we have the following log entry :
```
"55.3.244.1 GET /index.html 15824 0.043"
```
with the logstash filter  
```
filter {
  mutate {
    remove_field => [ "@version" ]
  }
  
  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{IP:clientip} %{WORD:method} %{URIPATHPARAM:request}
                 %{NUMBER:bytes} %{NUMBER:duration}" }
    }
    geoip {
      source => "clientip"
      target => "geoip"
      database => "/etc/logstash/GeoLiteCity.dat"
      add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
      convert => [ "[geoip][coordinates]", "float"]
    }
  }
}
```
Through **[grok patterns](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)** -a Logstash proprietary "regex like" concept- we extracted separate fields from the initial "meaningless" text line.

The syntax for a grok pattern is %{SYNTAX:SEMANTIC}

  - SYNTAX is the name of the pattern
      - **IP** will match an IP like pattern like "192.168.1.10"
      - **NUMBER** will match a number like "1234", etc.
  - SEMANTIC 
      Is the identifier we give the matched piece of text.

so applying the above GROK we'll get the fields 

 - clientip: 55.3.244.1
 - method: GET
 - request: /index.html
 - bytes: 15824
 - duration: 0.043

through the **geoip** plugin there is also the possibility to lookup the previous extracted **clientip** field to add extra 'geoip' fields. 

There are lots of **grok** patterns already available in logstash and you can also build your own and also a multitude of [filter plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html).  
While certainly a powerful mechanism, I'm not going to insist on it, because we plan on sending json logs (so {key:value} already provided) into Logstash.
    

#### output
Specifies where Logstash will be delivering the processed data. Again there are more than one output options available through the use of different plugins. We'll be using the 'elasticsearch' plugin, but there is also the possibility to configure the output to plain stdout (great for local debug) or even HadoopFS hdfs, or throw it back into a Kafka queue for some realtime analytics. Most likely scenario you'd want to separate to which index in ElasticSearch you want to store the logs based upon which services are sending the logs. 

```
output {
    if ([environment] == "qa" {
        stdout {}
    } else {
        elasticsearch {
            host => "elasticsearch_host"
            index => "elk-%{+YYYY.MM.dd}"
            document_type => "log"
        }
    }
}
```
For a more detailed look on what you can configure you can read [logstash-output-elasticsearch](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html)


### Pushing the log data to Logstash
There are more than one way to get the log data into Logstash. 

 - One way to go about this is to **use an external log shipper**, some external agent that would periodically read and forward the data to logstash and knows about the rolling file concept of the logs. One such option is to do it with [Beats](https://github.com/elastic/beats).
    
 - People using Docker containers to run their apps suggest  leveraging the feature of docker 'log drivers' ( [FluentD Docker Logging](http://www.fluentd.org/guides/recipes/docker-logging)) to capture the stdout output - [EFK stack](https://geowarin.github.io/spring-boot-logs-in-elastic-search.html). And also you might want to take a look at [Logspout](https://github.com/gliderlabs/logspout)
 - **Ship logs directly from your application using a [custom Logback Appender](https://github.com/logstash/logstash-logback-encoder)**. Since I'm already using **[Logback](http://logback.qos.ch/)** as the logging implementation of the popular SLF4J logging interface, my chosen solution is to use the  [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder) open source project which provides a custom [Logback Appender](https://www.loggly.com/ultimate-guide/java-logging-basics/) that we can use to **send our log events directly as JSON to Logstash**.
This makes it **easy to integrate in your existing java projects by just changing the logging configuration**(and importing the  ofc).


### Using [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder)

Now before jumping on this solution for you Java app, you might have questions and worries like "what happens when you want to log an event but the Logstash server is down?", "Do the log calls suddenly block waiting for the connection to be obtained?", "Does it have a performance impact?". The answer is NO, because **the data is being sent to Logstash asynchronously**. 
The [LogstashTcpSocketAppender](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/appender/AbstractLogstashTcpSocketAppender.java) actually writes into a [LMAX RingBuffer](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/appender/AsyncDisruptorAppender.java) and that RingBuffer is consumed and the data pushed to Logstash on a separate thread. So we'll not see any blocking of the application code even if the Logstash server is unavailable. 

Also you need to understand that **the events will be dropped in case the events are produced faster than Logstash can consume them**(or if Logstash is not available) so there is no danger of running out of memory with the increase of unconsumed log events. Further customization and tweaking of the RingBuffer is specified in the logstash-logback-encoder [documentation](https://github.com/logstash/logstash-logback-encoder).

Let's see how a _logback.xml_ configuration would look like using this custom Logstash Appender:
```
    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>logstash_host:5000</destination>

        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <mdc/> <!-- MDC variables on the Thread will be written as JSON fields--> 
                <context/> <!--Outputs entries from logback's context -->                               
                <version/> <!-- Logstash json format version, the @version field in the output-->
                <logLevel/>
                <loggerName/>

                <pattern>
                 <pattern> <!-- we can add some custom fields to be sent with all the log entries.                  
                    {      <!--make filtering easier in Logstash-->
                    "appName": "elk-testdata",<!--or searching with Kibana-->
                    "appVersion": "1.0"
                    }
                 </pattern>
                </pattern>

                <threadName/>
                <message/>

                <logstashMarkers/> <!-- Useful so we can add extra information for specific log lines as Markers--> 
                <arguments/> <!--or through StructuredArguments-->

                <stackTrace/>
            </providers>
        </encoder>
    </appender>

    <!-- Our logger writes to file, console and sends the data to Logstash -->
    <logger name="ro.fortsoft.elk.testdata" level="INFO" additivity="false">
        <appender-ref ref="FILE"/>
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="STASH"/>
    </logger>    
```

so the line
```
log.info("FAILED login for user='{}'", username);
```

pushes the following json to Logstash(let's consider that 'sid', 'userId' and 'remoteIP' were [MDC variables](http://logback.qos.ch/manual/mdc.html)):
```
{
"sid" => "1t1z90su54ca3xnzp71u2wdwa",
"userId" => "80503",
"remoteIP" => "192.168.1.10",
"HOSTNAME" => "DX-56WKT93",
"@version" => 1,
"appName" => "elk-testdata",
"appVersion" => "1.0",
"logger_name" => "ro.fortsoft.elk.testdata.generator.LoginEvent",
"level" => "INFO",
"thread_name" => "frontend-web-6",
"message" => "FAILED login for user='oneX5@gmail.com'",
"@timestamp" => "2015-07-23T10:22:16.757Z",
"host" => "127.0.0.1",
"type" => "syslog"
}
``` 

#### MDC variables automatically added to Logstash
MDC variables are perfect for scenarios like web requests which are processed by a single thread. 
Through a Servlet Filter you might read some data from the web requests or from the web session and then pass the control to the Servlet that does the processing(while clearing the value ). The simplest example would be the 'userId' for an authenticated user to always appear in the logs and not have to 'manually' add it for each log statement. 
 
Or another ex. could be to implement a request tracking system across a series of microservices. By passing along some kind of correlationId you could see which requests in other systems were triggered by user actions and why they failed.
One simple way to do this could be by having a Servlet filter that reads a value say 'GURID' from the Requests Headers and place it in an MDC variable. When making REST requests to other services read the 'GURID' variable from the MDC context and pass it along in the request headers to other service and so on.
 
#### Sending custom fields for a logging statement
There might be cases when we might want to explicitly supply a separate field for certain logging statements. 

```
log.info("FAILED login for user='{}' from IP={}", username, ipArgument);
```

They are useful because in our quest for doing log analytics we might want to do some grouping by a certain field.  
Like in the example above we might want to see how many "FAILED Login" entries there are, but from the same IP -perhaps we want to temporarily ban that IP-.

There are two ways of adding custom fields to be sent to Logstash:
    
 - SLF4J **Markers**, from when you don't want the field value to appear in the log message(that gets written to console or file)         
   You might not have used it but there are already available methods(for each log level) in standard SLF4J that allow a Marker parameter:
```
public void debug(Marker marker, String format, Object arg);
public void info(Marker marker, String format, Object arg);
...
```

so
```
 Marker ipArgument = Markers.append("remoteIP", remoteIP);
 log.info(ipArgument, "FAILED login for user='{}'", username);
```
would produce 
```
{ ....
 "remoteIP":"192.168.1.178",
 "message":"FAILED login for user='hacker'"
...
}
```

   - **StructuredArguments** is a class provided by logstash-logback-encoder. You can use it when you want it both sent in the json and also key and/or value inside the log message.

So we could do the same as above
```
  StructuredArgument ipArgument = StructuredArguments.value("remoteIp", remoteIP);
  log.info("FAILED login for user='{}' IP={}", username, ipArgument);
```
```
{ ....
 "remoteIP":"192.168.1.178",
 "message":"FAILED login for user='hacker' IP=192.168.1.178"
...
}
```

PS: As seen above the 'remoteIP' field could have also been extracted by Logstash by writing some grok pattern on the log message.     

Conclusion:
I hope to have opened your appetite for log shipping and doing analytics on your log data by trying the ELK stack.
You can look at the [github repo](https://github.com/balamaci/blog-elk-docker) to get a look at the code to start your own ELK stack on [docker](https://balamaci.ro/docker-series-intro-to-docker/) images.

Stay tuned for [Part II](https://balamaci.ro/java-app-monitoring-with-elk-elastic-search/) in which I plan to discuss in more detail ElasticSearch.
