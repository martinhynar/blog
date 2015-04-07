---
layout: post
title: "ElasticSearch - Migrate data to another index"
abstract: "Because I want to use index with different setting that cannot be changed on existing one, typically shard count"
tags: clojure cli
---

You can read in ElasticSearch guide that there is no silver bullet on defining number of shards for your index. Also, for number of shards,
you have no other option than define it for your index once. This is just matter of fact because other functions (e.g. routing simply counts
with that). What you can do, and what is also recomended is to do a homework on what your data will look like, how big they gonna be and what
response you can expect. This is, go ahead and understand your data first.

### Move data from index to another one with different settings

Having index created with

~~~
curl -XPUT 'http://localhost:9200/threeshards' -d '
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3,
            "number_of_replicas" : 0
        }
    }
}'
~~~

Check the settings of the index using `curl 'http://localhost:9200/threeshards/_settings?pretty'`

Now come the part where the data shall be generated. At this moment, you run your application and gather data. For the purpose of the example,
lets just generate some documents artificially. E.g. 100 docs with some timestamp in it, but that does not matter now.

~~~
for i in $(seq 100); do
   curl -s -XPOST http://localhost:9200/threeshards/test/$i?pretty -d "{ \"when\" : \"$(date --rfc-3339=ns)\"}" > /dev/null
done;
~~~

Check that documents are there with `curl 'http://localhost:9200/threeshards/_stats/docs?pretty'`. You should see something like 100 docs.
These are now spread across 3 shards.

Now create index with different settings. In this case, different number of shards.

~~~
curl -XPUT 'http://localhost:9200/twoshards' -d '
{
    "settings" : {
        "index" : {
            "number_of_shards" : 2,
            "number_of_replicas" : 0
        }
    }
}'
~~~

You can again check it with `curl 'http://localhost:9200/twoshards/_settings?pretty'`

In order to migrate data from index `threeshards` to index `twoshards` you need to do this.

1. Run the **scan** search request to get **scroll id**
Check the reponse here: `curl -s 'http://localhost:9200/threeshards/_search?search_type=scan&scroll=1m&pretty' -d '{ "size":  10 }'`
2. Use the **scroll id** in **scroll** search request to get documents
Run `curl -s -XGET  'localhost:9200/_search/scroll?scroll=1m' -d 'SCROLL ID GOES HERE'`
3. Index received documents into new index
4. Repeat steps 2 and 3 until all documents are processed. Also, **scroll** query will always return fresh **scroll id**

Looks simple, but making those steps require some scripting. In pure bash, it is rather complicated as you get JSON responses
and documents sources are buried inside. To make things way easier, the best idea is to use some language that has API for
ElasticSearch. I like Clojure, so I will use Elastisch here. (But any other language will do, maybe even better.)

In the example curl commands above, the `scroll=1m` part indicated to ElasticSearch that the data shall be kept accessible for 1 minute.

Of course, you can do this using ElasticSearch REST API, it is supported, but you'll not enjoy it.

<script src="https://gist.github.com/martinhynar/c2d5a8860f6e6963c974.js"></script>


