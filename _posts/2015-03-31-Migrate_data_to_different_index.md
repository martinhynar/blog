---
layout: post
title: "ElasticSearch - Migrate data different index"
abstract: "Because I want to use index with different setting that cannot be changed on existing one, typically shard count"
tags: clojure cli
---

You can read in ElasticSearch guide that there is no silver bullet on defining number of shards for your index. Also, for number of shards,
you have no other option than define it for your index once. This is just matter of fact because other functions (e.g. routing simply counts
with that). What you can do, and what is also recomended is to do a homework on what your data will look like, how big they gonna be and what
response you can expect. This is, go ahead and understand your data first.

### Move data from index to another one with different settings

{% highlight clojure linenos %}
~~~
(ns leiningen.inject-parameters)

(defn inject-parameters [project & args]
  (let [resource-file (format "%s/resources/injected.edn" (:root project))
        parameters (select-keys project [:version :name])]
    (spit resource-file parameters :append false)
    )
  )
~~~
{% endhighlight %}

