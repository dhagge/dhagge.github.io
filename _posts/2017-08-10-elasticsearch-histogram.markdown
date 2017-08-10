---
layout: blog
img: blogs/es_histogram_main.png
category: Blog
title: Histograms with Elasticsearch
tags: 
  - elasticsearch
  - elastic.co
  - big data
  - java
description: |
---

Elasticsearch has been [gaining popularity very quickly](http://db-engines.com/en/ranking_trend/system/Elasticsearch) over the past few years, and for good reason. It features fast and efficient text search capability backed by [Lucene](https://lucene.apache.org/), but it also has [excellent real-time analytics](http://www.etondigital.com/elasticsearch-real-time-search-analytics/). I've personally used elasticsearch extensively and generally found it to be better for search-based analytics than other options such as [hadoop](http://hadoop.apache.org/).

Once of the analytics use-cases I've found elasticsearch to excell at is real-time histograms, especially when they need to be real-time [ad-hoc](http://www.learn.geekinterview.com/data-warehouse/dw-basics/what-is-an-ad-hoc-query.html) queries. In this arcticle we'll work through an example of how to build a histogram query with elastcisearch. 

One thing to keep in mind is that elasticsearch works at [scale](https://www.techopedia.com/definition/28987/large-scale-data-analysis)! Everything you see here could also be done in [SQL](https://en.wikipedia.org/wiki/SQL) but there's a point at which relational-databases [don't handle the data volume well](https://www.sitepoint.com/sql-vs-nosql-differences/) and a [NoSQL](https://en.wikipedia.org/wiki/NoSQL) is the only realistic way to query very large datasets. 

## Histograms

Firstly, what is a [histogram](https://en.wikipedia.org/wiki/Histogram)? Well, it's a graphical representation of a time-sliced series of data. Let's say, for example, that you have a dataset of purchases which contains the fields `purchase`, `date` and `price`. A histogram would be a graph of that dataset over time, for example, the average price of purchases for each month of 2016:

<div style="text-align: center;">
  <img src="/img/blogs/avg_sales_x_month_histogram.png" alt="Average Sales By Month Histogram" style="height: 400px;"/>
</div>

## Elasticsearch Histogram

Now let's build our own histogram using elasticsearch. 

### Install and start elasticsearch

First let's [install elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html). 

Once it's installed we can start the instance:

```bash
cd elasticsearch-5.0.2/bin/
./elasticsearch
```

To verify that the instances started correctly goto `http://localhost:9200?pretty` and you should see a response in the browser that looks something like:

```json
{
    "name": "Piper",
    "cluster_name": "elasticsearch",
    "cluster_uuid": "_VGr2MAHQPK7tVxPlCPmDw",
    "version": {
        "number": "2.4.1",
        "build_hash": "c67dc32e24162035d18d6fe1e952c4cbcbe79d16",
        "build_timestamp": "2016-09-27T18:57:55Z",
        "build_snapshot": false,
        "lucene_version": "5.5.2"
    },
    "tagline": "You Know, for Search"
}
```

### Index a dataset 

Now let's [index](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html) a dataset into the elasticsearch instance. Indexing is just the elasticsearch terminology for adding a record.

We'll create a dataset for a cactus distributer in the south-western United States. I've provided two examples below, a command line-based [curl example](#index-curl) and a [Java example](#index-java). You can follow either one since they both do exactly the same thing.

<a name="index-curl"></a>
#### Curl

First let's create an elasticsearch [mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html) for our index. A mapping is like a schema for the documents we are going to index into elasticsearch.

```bash
curl -XPUT http://localhost:9200/cactus -d '{
  "mappings" : {
    "sales" : {
      "properties" : {
        "price" : { "type" : "double" },
        "date" : { "type" : "date" },
        "state" : { "type" : "keyword" },
        "category" : { "type" : "keyword" }
      }
    }
  }
}'
```

This mapping for the `sales` [type](https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html#_type) contains four fields: `price`, `date`, `state` and `category` and specifies [field datatypes](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html) for each one. The [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html) type is critical here since it will allow us to do [terms aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html) on those fields as we'll see later on when we execute our queries.

Now let's index a sample dataset into elasticsearch:
```bash
# pull down the sample bulk index data from github
curl https://raw.githubusercontent.com/dhagge/sitepoint-elasticsearch-aggregation/master/resources/data.index > data.index

# index the data into ES via bulk index request
curl -XPOST http://localhost:9200/_bulk --data-binary "@data.index"; echo
```

Now let's verify that the data actually did index correctly. In a brower go to `http://localhost:9200/cactus/_search?pretty` and you should see results like:
```json
{
    "took": 95,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2000,
        "max_score": 1,
        "hits": [
            {
                "_index": "cactus",
                "_type": "sales",
                "_id": "AVjHuSFdTUnQhGv7Lwlk",
                "_score": 1,
                "_source": {
                    "price": 1916.55,
                    "date": "2016-12-01",
                    "state": "AZ",
                    "category": "Cholla"
                }
            },
        ... etc ...
```

Note that anytime you want to start from scratch you can always drop the `cactus` index by doing:
```bash
curl -XDELETE http://localhost:9200/cactus
```

<a name="index-java"></a>
#### Java

Full source code for the Java population code can be found [here in github](https://github.com/dhagge/sitepoint-elasticsearch-aggregation).

First let's create a method to create our test dataset:
```java
static final List<String> STATES =
        Arrays.asList("AZ", "CA", "CO", "NM", "NV", "TX", "UT");
static final List<String> CATEGORIES = Arrays.asList(
        "Cholla", "Barrel", "Hedgehog", "Prickly Pear", "Saguaro");

static final DecimalFormat df2 = new DecimalFormat(".##");
static final Random random = new Random();
/**
 * Create a random dataset to index into elasticsearch.
 * @param bulkRequest The bulk request builder to add the dataset to
 * @return The document to index
 * @throws IOException
 */
public static void createDataset(Client client,
                                 BulkRequestBuilder bulkRequest)
        throws IOException {
    // put 1000 random records into the dataset
    //  - a random price between 1 and 10,000
    //  - a random date within the past year
    //  - a random state in the southwest
    //  - a random category of cactus
    for (int i=0; i<1000; i++) {
        XContentBuilder data = jsonBuilder()
            .startObject()
            .field("price", df2.format(random.nextDouble() * (10000.0 - 1.0) + 1.0))
            .field("date", LocalDate.now().minusDays(random.nextInt(365)))
            .field("state", STATES.get(random.nextInt(STATES.size())))
            .field("category", CATEGORIES.get(random.nextInt(CATEGORIES.size())))
            .endObject();
        IndexRequestBuilder builder =
                client.prepareIndex("cactus", "sales").setSource(data);
        bulkRequest.add(builder);
    }
}
```

This creates 1000 records with random price and date as well as chooses a random state in the south-west United States and a random category of cactus. Each record is added to our bulk index request via the call to `client.prepareIndex("cactus", "sales").setSource(data)`.

Now let's write some code that creates the [mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html) for our index:
```java
public static XContentBuilder createMapping() throws IOException {
    return jsonBuilder()
        .startObject()
            .startObject("sales")
                .startObject("properties")
                    .startObject("price")
                        .field("type", "double")
                    .endObject()
                    .startObject("date")
                        .field("type", "date")
                    .endObject()
                    .startObject("state")
                        .field("type", "keyword")
                    .endObject()
                    .startObject("category")
                        .field("type", "keyword")
                    .endObject()
                .endObject()
            .endObject()
        .endObject();
}
```

Now let's actually create logic that calls our createDataset(..) method and indexes it into elasticsearch. A [bulk index](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) is an index which takes multiple records in a single call.

```Java
TransportClient client = createClient();

try {

    // create the mapping for our index
    client.admin().indices().prepareCreate("cactus")
            .addMapping("sales", createMapping())
            .execute().actionGet();

    // create a bulk request (which batches multiple requests into a single call)
    BulkRequestBuilder bulkRequest = client.prepareBulk();
    createDataset(client, bulkRequest);

    // now call elasticsearch to index the documents
    BulkResponse bulkResponse = bulkRequest.get();
    if (bulkResponse.hasFailures()) {
        System.out.println("Oops, apparently cactus datasets are hard to create...");
        System.out.println(bulkResponse.buildFailureMessage());
    } else {
        System.out.println("Yay, we have lots of cactus sales!");
    }
} finally {
    client.close();
}
}
``` 
We create a `TransportClient` which connects to the elasticsearch [admin port](https://www.elastic.co/guide/en/elasticsearch/guide/current/_talking_to_elasticsearch.html). Then we create the index via the call to `prepareCreate(...)` and supply our mapping. Finally we add the dataset to the `BulkRequest` and make the request via the call to `bulkRequest.get()`.

Now let's verify that the data actually did index correctly. In a brower go to `http://localhost:9200/cactus/_search?pretty` and you should see the data we just indexed (note that by default an ES query only returns 10 records).

### Histogram queries

To obtain data results that can be graphed as a histogram we use [search aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html). This is a search query which returns [buckets](https://www.elastic.co/guide/en/elasticsearch/guide/master/_buckets.html) of data that we can use to display a graphical representation of the data.

First let's query elasticsearch for a histogram of total sales by month.

#### Curl

```bash
curl -XPOST http://localhost:9200/cactus/_search?pretty -d '{
  "aggregations": {
    "salesByDate": {
      "date_histogram": {
        "interval": "month",
        "field": "date"
      },
      "aggregations": {
        "totalSales": {
          "sum": {
            "field": "price"
          }
        }
      }
    }
  },
  "size": 0
}'
```

We're sending an aggregation request with a `date_histogram` aggregation named `salesByDate` which specifies an `interval` or `month` on the field `date`. We're also including a `totalSales` sub-aggregation which is a `sum` on the field `price`.

This will return us an aggregation response with each `salesByDate` bucket containing a single `totalSales` bucket which in turn contains the sum of all prices for data that belong in that month.

When you run the curl command you should see a response something like:

```bash
{
  "took" : 11,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1000,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "salesByDate" : {
      "buckets" : [
        {
          "key_as_string" : "2015-12-01T00:00:00.000Z",
          "key" : 1448928000000,
          "doc_count" : 75,
          "totalSales" : {
            "value" : 402499.5707550049
          }
        },
        {
          "key_as_string" : "2016-01-01T00:00:00.000Z",
          "key" : 1451606400000,
          "doc_count" : 82,
          "totalSales" : {
            "value" : 424961.34213256836
          }
        },
        ...etc...
      ]
    }
  }
}
```

As you can see each `salesByDate` bucket contains a `key` which is timestamp of the month of data the bucket represents, and a `totalSales` object with the `value` field representing the total sum of all sales for that month.

#### Java

To perform the exact same aggregation query in Java we would do the following:

```java
    DecimalFormat PRICE_FORMATTER = new DecimalFormat(".##");
    DateTimeFormatter DATE_FORMATTER = DateTimeFormat.forPattern("yyyy/MM/dd");

    TransportClient client = ...;

    AggregationBuilder agg =
            AggregationBuilders.dateHistogram("salesByDate").field("date")
            .dateHistogramInterval(DateHistogramInterval.MONTH)
            .subAggregation(AggregationBuilders.sum("totalSales").field("price"));

    SearchResponse sr = client.prepareSearch("cactus")
            .addAggregation(agg)
            .execute().actionGet();

    MultiBucketsAggregation aggResp = sr.getAggregations().get("salesByDate");

    for (MultiBucketsAggregation.Bucket bucket : aggResp.getBuckets()) {
        DateTime key = (DateTime)bucket.getKey();
        InternalSum price = bucket.getAggregations().get("totalSales");
        System.out.println("Date: " + DATE_FORMATTER.print(key) +
                ", Total Sales: " + PRICE_FORMATTER.format(price.getValue()));
    }
```

We're creating the same search aggregation as the [curl example](#TBD): a `date_histogram` aggregation named `salesByDate` which specifies an `interval` or `month` on the field `date`.

If you run this code you'll see the total sales per month output, something similar to:

```
Date: 2015/12/01, Total Sales: 402499.57
Date: 2016/01/01, Total Sales: 424961.34
Date: 2016/02/01, Total Sales: 507919.05
Date: 2016/03/01, Total Sales: 392810.21
Date: 2016/04/01, Total Sales: 380018.01
...etc...
```

### Other histograms

There are, of course lots of other histograms that can be imagined. For example let's say we wanted to actually obtain average sales per month by state:

```bash
curl -XPOST http://localhost:9200/cactus/_search?pretty -d '{
  "aggregations": {
    "salesByDate": {
      "date_histogram": {
        "interval": "month",
        "field": "date"
      },
      "aggregations": {
        "state": {
          "terms": { 
            "field": "state" 
          },
          "aggregations": {
            "totalSales": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}'
```

Or another example of average price by category per day:

```
curl -XPOST http://localhost:9200/cactus/_search?pretty -d '{
  "aggregations": {
    "salesByDate": {
      "date_histogram": {
        "interval": "day",
        "field": "date"
      },
      "aggregations": {
        "state": {
          "terms": { 
            "field": "category" 
          },
          "aggregations": {
            "totalSales": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}'
```

Elasticsearch features rich search capabilities around histograms, which can be explored [here](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/search-aggregations.html).

## Summary

We've seen how we can create an elasticsearch [index](https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html#_index), put a dataset into the index and then run various [aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/search-aggregations.html) on that dataset to obtain raw data which represents a histogram.

Histograms are only a part of what elasticsearch can do and I strongly encourage you to [explore the full documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html) to get a feel for it's full capabilities.
