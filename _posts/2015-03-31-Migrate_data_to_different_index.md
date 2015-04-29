---
layout: post
title: "ElasticSearch - Migrate data to another index"
abstract: "Because I want to use index with different setting that cannot be changed on existing one, typically shard count"
tags: ElasticSearch Clojure
---

You can read in ElasticSearch guide that there is no silver bullet on defining number of shards for your index. Also, for number of shards,
you have no other option than define it for your index once. This is just matter of fact because other functions (e.g. routing simply counts
with that). What you can do, and what is also recommended is to do a homework on what your data will look like, how big they gonna be and what
response you can expect. This is, go ahead and understand your data first.

### Move data from index to another one with different settings

Having index created with

<script src="https://gist.github.com/martinhynar/c2d5a8860f6e6963c974.js?file=threeshards.sh"></script>

Check the settings of the index using `curl 'http://localhost:9200/threeshards/_settings?pretty'`

Now come the part where the data shall be generated. At this moment, you run your application and gather data. For the purpose of the example,
lets just generate some documents artificially. E.g. 100 docs with some timestamp in it, but that does not matter now.

<script src="https://gist.github.com/martinhynar/c2d5a8860f6e6963c974.js?file=feeddata.sh"></script>

Check that documents are there with `curl 'http://localhost:9200/threeshards/_stats/docs?pretty'`. You should see something like 100 docs.
These are now spread across 3 shards.

Now create index with different settings. In this case, different number of shards.

<script src="https://gist.github.com/martinhynar/c2d5a8860f6e6963c974.js?file=twoshards.sh"></script>

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
ElasticSearch. I like Clojure, so I will use Clojure here. (But any other language will do, maybe even better.)

In the example curl commands above, the `scroll=1m` part indicated to ElasticSearch that the data shall be kept accessible for 1 minute.

Of course, you can do this using ElasticSearch REST API, it is supported, but you'll not enjoy it.

<script src="https://gist.github.com/martinhynar/c2d5a8860f6e6963c974.js?file=core.clj"></script>

Note: There's also [project.clj](https://gist.github.com/martinhynar/c2d5a8860f6e6963c974#file-project-clj) with all necessary dependencies needed for this code to work.

Seeing this is probably not what you want to do every time you need to reindex data. Take this code as illustration of how reindexing works.

For serious cases, I recommend to go with [codelibs/reindexing](https://github.com/codelibs/elasticsearch-reindexing) plugin. Using ES plugin gives you tool that is close enough to ElasticSearch to give you reasonable performance.
