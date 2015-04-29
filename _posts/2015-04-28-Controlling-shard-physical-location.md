---
layout: post
title: "Controlling physical shard allocation in ElasticSearch cluster"
abstract: "You might have heterogenous hardware with some extra powerful pieces, or you might have some network limitations that does not allow you to alocate shards automatically. Whatever reason, maybe you just want to control where shards are, this is one way I know about how to accomplish this."
tags: ElasticSearch
---

I faced a situation in which I ended up with ElasticSearch cluster that was geographically separated in two locations.
Unfortunately, the link between them was for some "startup" time bit under-calibrated. Because of that, automatic shard
allocation was not an option for new indexes There was a risk that shards could be allocated to nodes that are
physically located behind the _narrow link_ and other services could start being influenced.

I am quite sure that such situation was not taken in consideration when ElasticSearch was designed, but different one
was. And solution for it fitted quite well for my problem.

### Solution

The short answer is **routing**. But not that routing that is based on document data, but the one that is based on data
associated with nodes (I don't know if that has some special name, so if you know, let me know in comment).

#### Part 1

As I said above, I had 2 parts of cluster connected by _narrow link_. So, I marked all nodes on respective sides of the
link with distinct labels like this.

~~~
Node1 (node.node_label=alpha)                              Node4 (node.node_label=omega)
Node2 (node.node_label=alpha) ------- narrow link -------- Node5 (node.node_label=omega)
Node3 (node.node_label=alpha)                              Node6 (node.node_label=omega)
~~~

This labelling is done by passing it from command line (see example script bellow) or more durably using
`elasticsearch.yml` (but you need to have separate ones if you play on localhost).

<script src="https://gist.github.com/martinhynar/d64609639a4b2e858a1f.js?file=start-cluster.sh"></script>

Now, it is at hand that you want to say somehow _please, place shards of my index to alpha nodes only_. And that is
exactly what ElasticSearch allows you to do.

#### Part 2

In order to inform ElasticSearch where to place shards, you need to create index with some extra settings. In this
example case, index that will be located on _alpha_ nodes is created like this:

<script src="https://gist.github.com/martinhynar/d64609639a4b2e858a1f.js?file=index-to-alpha.sh"></script>

The important parameter here is `"index.routing.allocation.include.node_label" : "alpha"`. Besides that,
`"number_of_shards" : 6` is greater than number of _alpha_ nodes. Use script `shard-allocation.sh` to check what are the
node ID's and where individual shards were placed.
Surprisingly, all shards will be created on nodes with `node.label=node_alpha`. It is, if there are more shards than
nodes with desired label available, they are still placed on those nodes.

<script src="https://gist.github.com/martinhynar/d64609639a4b2e858a1f.js?file=shard-allocation.sh"></script>

### Original motivation for node labelling

AFAIK, node labelling was originally aimed to distinguish stronger and weaker nodes. The goal was simple, usually you
hardware with various parameters, some nodes more powerful, more memory, more cpu cores and on the other side weaker
nodes. For one of the typical usage, log collection, you are bombing one index (index with logs for current day) with
tons of data, and day after you are looking at it with much more relaxed approach (query something here, query something
there and maybe do not query it at all if there was no alert yesterday). Naturally, you want to have index that is just
receiving log events spread on the nodes that could easily handle that load and node labeling is just the approach to
accomplish that.

#### Try it yourself

There is prepared [vagrant recipe](https://gist.github.com/martinhynar/d64609639a4b2e858a1f) for testing this behavior.
Set it up and go to `/srv/salt`. Prepared script `start-cluster.sh` runs 4 ElasticSearch nodes, all connected into
cluster. From these 4 nodes, 2 are labeled _alpha_ and 2 are labeled _omega_. Using script `index-to-alpha.sh`, new
index named `test` is created with setting to use `node_label` for routing. It also creates some documents in index to
have something in. Last script `shard-allocation.sh` gets ID's of individual nodes and also information about shards
(and their allocation). Check that shards of `test` index are on _alpha_ nodes as expected.

### Notice

If you are going to use node labelling, don't forget one thing. Normally, without node labelling, ElasticSearch cluster
will select node for shard on its own (or based on some document field, if you wanted), but with node labelling it will
be limited in cluster reallocation when some nodes go away or cluster becomes unbalanced. Keep that in mind to avoid
overwhelming some node with too much work.