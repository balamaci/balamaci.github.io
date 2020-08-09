+++
author = "Serban Balamaci"
date = 2016-03-21T13:10:26Z
description = "Java application monitoring with ELK stack"
draft = false
slug = "java-app-monitoring-with-elk-elastic-search"
title = "Java app monitoring with ELK - Part II- ElasticSearch"

+++


### ElasticSearch
ElasticSearch is probably the key element in our ELK stack, it acts the part of o a database, where we store the log entries and send our queries for retrieving the logs that match our searches.

Compared to other storage solutions **ElasticSearch** can be described as a **distributed, high availability, JSON document oriented storage solution targeted at fulltext searching**.

Let's try to explain some of the buzzwords:
ElasticSearch was designed from the start with the purpose of storing and searching text documents and is able to **index huge amounts of text data** and make it **searchable** with a lots of query options and even . 
It's **distributed** as it's relatively easy to start a  cluster of nodes which allows storage replication and the possibility to run queries on multiple nodes in parallel and then aggregate their response for faster response time and load distribution. 
**High availability** in the sense that failed cluster nodes are detected and data is rebalanced and reorganized to makeup for the lost nodes and still be able to index  documents and respond to queries.

The basic unit of reference in ES is the "Document" entity. All Documents are represented in JSON format. ElasticSearch exposes a REST API and almost **any action can be performed using a simple RESTful API using JSON over HTTP**.

Let's first have a business example:
One very common use(which speaks for ES versatility) is in online stores where the "Document" concept could be associated to an item sold on the site. Say we have a "Toshiba Laptop Satellite C55-C-137" we sell on our site among other laptops. We could map each laptop item with it's specifications into a "Document" to be stored inside ES: 


```
$ curl -XPUT 'http://es_host:port/products/laptop/1' -d '{ 
    "name":"Toshiba Laptop Satellite C55-C-137"
    "manufacturer": "Toshiba",
    "processor": "i3-4005U",
    "processor frequency": 1.7,
    "memory": 4,
    "description": "The best performance, this laptop is a top seller in our line, you can..",
    "price":300
 }'
```

We could then get a list of Toshiba or Dell laptops that are withing a price range by **combine** a **"term query"** matching the _manufacturer_ field with "Toshiba" or "Dell"  with a **"range query"** on the _price_ field basically mimicking a SQL query.

```
$ curl -XGET 'http://es_host:port/products/laptop/_search' -d '
  "filter" : {
     "bool" : {
        "should" : [ //should - acts as OR between the terms
             { "term" : {"manufacturer" : "toshiba"}}, 
             { "term" : {"manufacturer" : "dell"}} 
        ],
        "must" : {
             { "range": { "price": { "gte": 100, "lte":400 }}}
        }
    }
}'
```


But since we plan to use ElasticSearch for logs storage and searching, the Document entity will be a log event expressed as  JSON like:
```
{
"userId": "80503",
"remoteIP": "192.168.1.10",
"HOSTNAME": "DX-56WKT93",
"@version": 1,
"appName": "elk-testdata",
"appVersion": "1.0",
"logger_name":"ro.fortsoft.elk.testdata.generator.LoginEvent",
"level": "INFO",
"thread_name": "frontend-web-6",
"message": "FAILED login for user='oneX5@gmail.com'",
"@timestamp": "2015-07-23T10:22:16.757Z",
"host": "127.0.0.1",
"type": "syslog"
}
```

<br/>
#### ElasticSearch Index and Shards notions explained
At the base of ES there is the powerful java **Lucene text processing library**. Lucene handles the breaking down of a text into a series of **terms**(you might think of them for now as "words"). 
The terms are stored in a data structure called an **Inverted Index**, basically a Map in which the keys are the terms, and value is the list of documents which contain that term(therefore the 'inverted' part).
<div class="image-div">
![Inverted Index](/assets/2016/iindex.png)
</div>

Lucene existed long before ES and can be used on it's own for text processing documents. The **Index** concept in Lucene is different from the **Index** in ElasticSearch. 

Because ES was designed for clustering it needed a method to be able to distribute the data both for backup replication and parallelization and it did this by splitting an ES index into multiple **Lucene index files** - which **are called "Shards"** in ES terminology-. The **Shard is the atomic component of an Index**. ElasticSearch than takes care to distribute the shards on cluster nodes and knows how to query them and aggregate the response. If you want to read  about shard configuration, a good source is [here](http://cpratt.co/how-many-shards-should-elasticsearch-indexes-have/).
<br/>


#### ElasticSearch Document Type
Functionally speaking the **ES index** can be imagined in SQL terms as a **Database**. In an ElasticSearch instance/cluster **you can have as many indexes as you like**.

An ES Index can hold more than one **Document Types** - which in our sql analogy will be the equivalent of **DB Tables**. This is interesting because **we can limit our queries to a specific document type** (laptops, smartphones, tvs -in our online shop ex.).
 **Document Mapping** is the equivalent of DB tables schema definition so **we can specify different field mappings for different Document Types**.


$ curl -XPUT 'http://eshost:port/**index**/**document_type**/id' -d {...}

<br/>
#### Document Mapping

When indexing a JSON document of a specific document type for the first time, ES will try to guess the type of the field by looking at the value. 

"jobStart": "2015-07-23T10:22:16.757Z" is not actually a **string** type and it will be identified as **date** and thus enable us to make a range query between certain dates.

We can specify the field types upfront for an index.

There are two ways to set a Mapping for an index: 
**1. Using the REST endpoint** and PUTing a "mappings" json  like:
```
$ curl -XPUT 'http://eshost:port/products -d '{
  "mappings": {
    "laptops": { 
      "properties": { 
        "name":    { "type": "string"  }, 
	"processor": { "type": "string"  }, 
        "memory": { "type": "integer"  }, 
        "price":  { "type": "integer" }  
      }
    },
    "smartphones": { 
      "properties": { 
        "name":    { "type": "string"  }, 
        "productCode":  {
          "type":   "string", 
          "index":  "not_analyzed"
        },
        "created":  {
          "type":   "date", 
          "format": "strict_date_optional_time||epoch_millis"
        }
      }
    }}}'
```
PS: Getting the mapping for an index
```
$ curl -XGET 'http://es-server:port/products/_mapping?pretty'
```



**2. Using ElasticSearch Templates**

However since we're integrating with logstash which has a default **template** for document mapping. 
Templates define settings and mappings that will be used when a new index is created. They are especially useful if you create daily indexes(like we do) or you have dynamic field names to specify the field mappings.

You can register a template or read the registered templates through the REST API. 

```
$ curl -XPUT http://eshost:port/_template/my_template -d '{
    "template" : "logstash*", --> it's applied whenever new indexes with logstash name are created
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "tweet" : {
            "properties" : {
                "message" : {"type" : "string" },
                "remoteIP" : {"type" : "string", "index": "not_analyzed" }
            }
}}}'
```

However in [Part I](http://balamaci.ro/2016/03/10/java-app-monitoring-with-elk-logstash/) we saw that logstash allows us to configure the elasticsearch plugin options.

```
elasticsearch { 
      host => "elastic_host" 
      port => 9200 
      index => "logstash-%{+YYYY.MM.dd}" 
      template => "/etc/logstash/conf.d/elasticsearch-template.json" 
      template_name => "my-logstash" 
} 
``` 

The name pattern for new indices will make it so that the template you create only applies to that index and not other indices in ElasticSearch. 
Since our logstash index pattern looks like **logstash-YYYY.MM.dd** will have **logstash\*** with the asterisk acting as a wildcard in the template file.
Btw splitting the index like this is useful because we can be more specific(faster) should we want to search for "last 1 hour events" there is no reason to include in the search index files other than the current day(supposing you are passed midnight by 1h).  


Logstash has a [default mapping template](https://github.com/logstash-plugins/logstash-output-elasticsearch/blob/master/lib/logstash/outputs/elasticsearch/elasticsearch-template.json) on which we based our own template and just did minor modifications.

It includes a whole section about **dynamic templates** which **allows to define field mappings based on their name and type**. 

How do we treat new field:
```
      "message_field" : {
          "match" : "message", --> match by field named "message"
          "match_mapping_type" : "string", -->and datatype
          "mapping" : {
            "type" : "string", "index" : "analyzed", "omit_norms" : true
          }
        }
      }, {
        "string_fields" : {
          "match" : "*", --> match all the fields
          "match_mapping_type" : "string", 
          "mapping" : {
            "type" : "string", "index" : "analyzed", "omit_norms" : true,
            "fields" : { 
              "raw" : {"type": "string", "index" : "not_analyzed", "doc_values" : true, "ignore_above" : 256}
            }
          }
        },
        "our_url_field" : {
          "match" : "*_url", --> match field ending with "_url"
          "match_mapping_type" : "string", 
          "mapping" : {
            "type" : "string", "index" : "not_analyzed"
          }
        }
      }
```
but the **properties** section allows us to define mapping for a specific field:
```

```
PS: **Templates** allows also to set the number of **shards** for an index, and also define [custom analyzers](https://www.elastic.co/guide/en/elasticsearch/guide/current/custom-analyzers.html)
```
{
    "template" : "logstash*",
    "settings" : {
        "number_of_shards" : 2,
        "number_of_replicas": 1
    },
```

#### Analyzers

Let's explain the **index** property from above example:
```
...
  "productCode":  {
     "type":   "string", 
     "index":  "not_analyzed"
  }
```
When defining the mapping (schema) for **string** types, we can specify if the field will be **analyzed** or not.

To **extract the terms(words), Lucene applies Analyzers to the text input**. The **Analyzers** are a huge topic in themselves, but the simplest explanation would be to consider the fact that we'd like to index the "logger_name" and "host" fields from the above example log event: 
```
{...
"logger_name": "ro.fortsoft.elk.testdata.generator.LoginEvent",
"remoteIP": "192.168.1.10",
"filePath": "/tmp/uploads/fe45465-r3eff.pdf"
}
```
Now in an "usual" text fragment words are delimited through punctuation like "." and whitespace. 

So if we'd had the [standard analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html)(which drops most punctuation, breaks up text into individual words, and lower cases them) applied to "remoteIP" field we'd have "192","168","1","10" as terms in the inverted index.  
But we don't want this(since one of our goals was to get notifications on too many login attempts coming from the same IP) so we just want the value of the field stored as it is, without being analyzed at all, so we have the entire ip string stored as "192.168.1.10".
And analyzing the "remoteIP" also had the problem that searching for a certain IP it would not have matched.

```
"remoteIP": {
    "type": "string", 
    "index": "not_analyzed"
}
```
To prevent this "weird" behavior for people just starting with the ELK stack, the default logstash mapping template, specified the addition of **.raw** fields for every analyzed field. 
So by default we would get an analyzed **remoteIP** and **remoteIP.raw** which is the original value. Of course this is wasteful and it makes better sense to specify the fields that have no need to be analyzed this is why we pushed our [custom mapping template](https://github.com/balamaci/blog-elk-docker/blob/master/logstash/config/elasticsearch-template.json).

You can take a look at the whole docker setup on [github](https://github.com/balamaci/blog-elk-docker).

### Document Relevance

Text searching is also about document **relevance**. If I indexed all the articles of my blog and then search for "ElasticSearch Java" I'd expect the articles in the ELK series to appear to be at the top. 
But, I might have mentioned "ElasticSearch Java" once or twice in other articles that were actually on totally different topics, so those articles should rank at the bottom of our results. But how can we express this relevancy into something numeric that we can rank?
The answer is through [TF/IDF]() - TermFrequency / Inverse Document Frequency. It basically says that the the more frequent a "rare" term appears in a Document(and by "rare" we understand "low appearance" aka "low frequency" in overall documents), the better chances that the document is **"more relevant"** about the topic. 

So considering the blog indexing ex. and the query for "ElasticSearch Java" because:

   1. ElasticSearch is mentioned many times in the ELK article series.
   2. ElasticSearch is a "rare"(low frequency) word in overall documents compared to the more common "Java" which probably is present with high frequency in all the articles.
would make the ELK series articles highly relevant to the other articles that contain "Java" word.

Besides the TF/IDF calculated value, through the query object you can **boost** the importance of certain fields in how a document ranks for your search. 
Articles having "ElasticSearch" in their "title" probably have more chance of being relevant for "ElasticSearch", so we can "boost" the importance of the "title" field if words match their content.

### Queries vs Filters searches
So we've seen that **Search Queries** have this "score" of relevancy to the search query which is returned as the **_score** field.

our example log event:
```
log.error(ipMarker, "FAILED login for user='{}'", username);
```

```
curl -XGET 'http://eshost:9200/logstash*/_search?size=1&pretty' -d '{
    "query": {
        "match" : {
            "message" : "FAILED"
        }
    }
}'
```

```
{
  "took" : 10,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 79,
    "max_score" : 1.7308875,
    "hits" : [ {
      "_index" : "logstash-2016.03.24",
      "_type" : "logs",
      "_id" : "AVOqAPqLX5dexs9zGl81",
      "_score" : 1.7308875,
      "_source" : {
        "@version" : 1,
        "level" : "INFO",
        "logger_name" : "ro.fortsoft.elk.testdata.generator.LoginEvent",
        "appName" : "elk-testdata",
        "appVersion" : "1.0",
        "thread_name" : "pool-1-thread-10",
        "message" : "FAILED login for user='ryanespinoza@yahoo.com'",
        "remoteIP" : "192.168.0.112",
        "@timestamp" : "2016-03-24T19:01:44.090Z"
      }
    } ]
  }
}
```

Filters are less subtle, they just make a binary choice - **yes** or **no**- either include or not the document in the results.

**Filters make a binary decision: should this document be included in the results list or not?**
Notice the **filter** being used  
```
curl -XGET 'http://eshost:9200/logstash*/_search?size=1&pretty' -d '{
    "filter": {
        "match" : {
            "message" : "FAILED"
        }
    }
}'
```
the maxscore and score are fixed =1 
```
{
  "took" : 52,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 79,
    "max_score" : 1.0,
    "hits" : [ {
      "_index" : "logstash-2016.03.24",
      "_type" : "logs",
      "_id" : "AVOqAOf-X5dexs9zGl8T",
      "_score" : 1.0,
      "_source" : {
        "@version" : 1,
        "level" : "INFO",
        "logger_name" : "ro.fortsoft.elk.testdata.generator.LoginEvent",
        "appName" : "elk-testdata",
        "appVersion" : "1.0",
        "thread_name" : "pool-1-thread-8",
        "message" : "FAILED login for user='carlossavage@yahoo.com'",
        "remoteIP" : "192.168.0.188",
        "@timestamp" : "2016-03-24T19:01:39.253Z"
      }
    } ]
  }
}
```


### Common queries
#### Term queries
Useful for exact matches for a field
```
   "filter" : { //or query
       "term": { "level": "ERROR" } 
   }
```

#### Match queries
If you do not want an exact but rather that the value is contained you need to use **match**
```
   "filter" : { 
       "match": { "message": "FAILED login" } 
   }
```
#### Prefix or wildcard queries
```
   "filter" : { 
       "prefix": { "remoteIP": "192.168" } 
   }
```

#### Combining queries - boolean
Useful to construct something like AND and OR conditions. Say we want to see login events(both failed and successfull) between 2016->01/04/2016 but exclude logins from a certain subnet(start with "192.168.0.x").

The **must** clause serves as an AND. By specifying a **range query** we can filter the required time range.
The **should** clause serves as OR so you can combine . You can even control how , should you have more than one condition 

```
"filter": {
   "bool": {
        "must": {
            "range" : {
                "@timestamp" : {
                    "gte": "2016",
                    "lte": "01/04/2016",
                    "format": "dd/MM/yyyy||yyyy"
                }
            }
        },
        "must_not": { "prefix": { "remoteIP": "192.168.0" }},
        "should": [
                    { "match": { "message": "FAILED login" }},
                    { "match": { "message": "SUCCESS login"}}
        ]
   }
}
```
You can even nest queries 

### Aggregations
So we seen above that basically we can accommodate. But can we have some form of GROUP BY construct?

For example if we'd want to be alerted for when a number of errors reaches a certain threshold. But what if all the errors are for the same user who tries over and over and receives the same error. Not that you should not help that user, but the first thing you'd want to know if it's an isolated case or it's a general error, how many users are affected and maybe take the decision to do a rollback to a previous version.

Another example would be in our **LoginEvent** which we used to simulate users  logins. A high number of failed logins but from a wide range of IPs might be normal, too many failed logins from the same IPs might mean an attack and maybe you should take action to blacklist them. 

To be continued
