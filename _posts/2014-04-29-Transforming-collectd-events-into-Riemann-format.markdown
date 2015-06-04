---
layout: post
title: "Transforming collectd events into Riemann.io format"
abstract: "How to split n-value events gathered with collectd into events that could be nicely consumed with Riemann.io."
---
*{{page.abstract}}*

-----

<script src="https://cdnjs.cloudflare.com/ajax/libs/raphael/2.1.4/raphael-min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/underscore.js/1.8.3/underscore-min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

For one of my projects, I needed to collect various statistics about resource utilization in order to detect that something went wrong and be able to reason about what could be the cause of it. Because of other requiements and goals that are not important here, I ended up with this setup

<div class="diagram">
collectd->logstash:
logstash->riemann:
</div>
<script>
$(".diagram").sequenceDiagram({theme: 'hand'});
</script>

    +--Monitored Node------------+
    |  +--------+     +--------+ |          +-------+
    |  |collectd| --> |logstash| ---------> |Riemann|
    |  +--------+     +--------+ |          +-------+
    +----------------------------+


In this setup, [collectd](collectd.org) gathers effectively resource stats and passes them to [LogStash](logstash.net) via its [collectd input](http://logstash.net/docs/1.4.0/inputs/collectd). Further, in Riemann I do some magic to detect whether some resources are at the levels that might affect running services, but that is different topic. So, now it is the task LogStash to prepare an event that [Riemann](riemann.io) expects. This is, event of this format

    { :host where-this-event-happened
      :service a-service-name,
      :state state,
      :description a-description,
      :metric number,
      :tags [list-of-tags-to-mark-event-with-additional-flags],
      :time a-timestamp-when-this-event-happened,
      :ttl how-long-this-event-is-valid }


The problem that appears when pushing events from collectd to LogStash is that the events do not have this format, and to complicate things more, the events could be aggregated, such that one event contains several numeric values in one event. Ofcourse, this is not incorrect (why spend communication resources when it happened at once), but for Riemann, we need to have it separate.

So, consider this collectd event for swap utilization:

~~~ json
{ "@version" : "1",
  "@timestamp" : "2014-04-29T12:22:04.813Z",
  "host" : "example.com",
  "plugin" : "vmem",
  "collectd_type" : "vmpage_io",
  "type_instance" : "swap",
  "in" : 763,
  "out" : 33727 }
~~~

This event has pair of numeric values - `in` and `out` - that I want to separate into pair of Riemann-compliant events:

~~~ json
{ "@version" : "1",
  "@timestamp" : "2014-04-29T12:22:04.813Z",
  "host" : "example.com",
  "type" : ["swap-in","riemann"],
  "state" : "ok",
  "service" : "vmem-vmpage_io-swap-in",
  "metric" : "763" }
    
{ "@version" : "1",
  "@timestamp" : "2014-04-29T12:22:04.813Z",
  "host" : "example.com",
  "type" : ["swap-out","riemann"],
  "state" : "ok",
  "service":"vmem-vmpage_io-swap-out",
  "metric":"33727" }
~~~

In order to be able to do such trasformation, I used this LogStash `filter`

~~~ ruby
filter {
  if [type_instance] == "swap" {
    # First make as many clones as the desired number of output Riemann events
    clone {
      # This will become individual type of each cloned event
      clones => ["swap-in", "swap-out"]
    }
    # Now mutate each clone into specific Riemann event
    mutate {
      # This is deprecated, but at the moment of writting, conditionals does not work on cloned events
      type => "swap-in"
      add_field => {
          # Set the fields Riemann expects based on existing fields
          "state" => "ok"
          "service" => "%{plugin}-%{collectd_type}-%{type_instance}-in"
          "metric" => "%{in}"
          # Mark this event to be used for Riemann
          "type" => "riemann"
      }
      # Remove the fields that are of no further use
      remove_field => ["in", "out", "plugin", "collectd_type", "type_instance"]
    }
    mutate {
      type => "swap-out"
      add_field => {
          "state" => "ok"
          "service" => "%{plugin}-%{collectd_type}-%{type_instance}-out"
          "metric" => "%{out}"
          "type" => "riemann"
      }
      remove_field => ["in", "out", "plugin", "collectd_type", "type_instance"]
    }
  }
}
~~~

As mentioned in the LogStash filter, at the moment of writting, conditionals were defunct for cloned events so I had to use deprecated `type` filtering. As soon as [LOGSTASH-1476](https://logstash.jira.com/browse/LOGSTASH-1476) issue is solved, use conditionals instead.

Now, when I have events for Riemann prepared, I can forward them to Riemann server in `output` section.

~~~ ruby
output {
  if "riemann" in [type] {
    riemann { ... }
  }
}
~~~
