---
title: Kamon | OpenTSDB | 
tree_title: Sending Metrics to OpenTSDB
layout: module-documentation
---

Reporting Metrics to StatsD
===========================

[OpenTSDB] is a high performance open source database for handling time series
data. It uses hbase to allow it to store and serve massive amounts of time series data 
without losing granularity.


Installation
------------

Add the `kamon-opentsdb` dependency to your project and ensure that it is in your classpath at runtime, that's it.
Kamon's module loader will detect that the OpenTSDB module is in the classpath and automatically start it.

Follow [OpenTSDB installation instruction](http://opentsdb.net/docs/build/html/installation.html) if you don't already have an instance running.

Configuration
-------------

### Basic ###

You will need to set the `kamon.opentsdb.direct.quorum` configuration to point at your ZooKeeper servers.
Additionally to that, you can configure the metric categories to which this module will subscribe using the `kamon.opentsdb.subscriptions` key.
By default, the following subscriptions are included:

{% code_block typesafeconfig %}
kamon.opentsdb {
  subscriptions {
    histogram       = [ "**" ]
    min-max-counter = [ "**" ]
    gauge           = [ "**" ]
    counter         = [ "**" ]
    trace           = [ "**" ]
    trace-segment   = [ "**" ]
    akka-actor      = [ "**" ]
    akka-dispatcher = [ "**" ]
    akka-router     = [ "**" ]
    system-metric   = [ "**" ]
    http-server     = [ "**" ]
  }
}
{% endcode_block %}

If you are interested in reporting additional entities to OpenTSDB please ensure that you include the categories and name
patterns accordingly.  


### Metric Name ###

Names are generated by concatenating the results of several [rules](#rules) together 

```typesafeconfig
kamon.opentsdb.default.name = [application, category, entity, metric, stat]
```

This design allows you to easily customize the metric name by reordering, adding, and removing [rules](#rules) as you see fit. 
`kamon.opentsdb.name.separator` is inserted between each rule who result is non-empty.  Empty rules are removed from the metric name

### Metric Tags ###
All tags associated with the kamon metric will be passed through to OpenTSDB.
Additional tags may be added by mapping tag names to [rules](#rules).
A tag will be named the exact value listed in the config, if you would like
a different name, create a new rule (See [Extensibility](#extensibility))

```typesafeconfig
kamon.opentsdb.default.tags = [ application, host ]
```
### Stats ###

Many statistics can be calculated for each Kamon metric, and ,by default,
each statistic will be stored as a separate OpenTSDB metric.

```typesafeconfig
kamon.opentsdb.default.stats = [ count, rate, min, max, median, mean, 90, 99 ]
```

Most statistics can only be used with histograms, however assigning them
to a counter is harmless.

__Counter stats__
* __count__: The value of the counter
* __rate__: The value of the counter divided by the tick length in seconds

__Histogram Stats__
* __count__: The number of values stored in the histogram
* __rate__: The number of values stored in the histogram divided by the tick length in seconds
* __mean__: The average of all values stored during the tick
* __max__: The largest value stored during the tick
* __min__: The smallest value stored during the tick
* __median__: The median value (synonym for "50")

Additionally, numeric values in the stat list, will generate percentile statistics in the OpenTSDB database.
* __50__: 50th percentile
* __70.5__: 70.5th percentile
* __90__: 90th percentile

See [Extensibility](#extensibility) to learn how to create your own stats.

### Rules ###

Rules can either be constant values or dynamical generated.  
All the provided rules are dynamically generated except for "application", which is empty by default.
To set the application name, provide set `kamon.opentsdb.rules.application.value`   

* __category__: The entity's category.
* __entity__: The entity's name.
* __metric__: The metric name assigned in the entity recorder.
* __stat__: The name of the statistic
* __host__: The local host name
* __application__ : The name of the application.  Undefined by default.

See [Extensibility](#extensibility) to learn how to create your own rules.

### Idle metrics ###
if `kamon.opentsdb.default.filterIdle` is true, values will not be sent to opentsdb for inactive metrics, 
as opposed to sending 0.

### Timestamps ###
Set`kamon.opentsdb.default.timestamp` to 'seconds' or 'milliseconds'.
 
Advanced configuration
----------------------

Metrics can be configured at the global, category, and individual metric level.

Altering the configuration values under `kamon.opentsdb.default` will impact all metrics.

Category specific configuration can be achieved by creating entries under `kamon.opentsdb.category.<category_name>`
Any values not set on the category level, will be inherited from the defaults.

```typesafeconfig
kamon.opentsdb.category.counter = { name = [ metric ], stats = [count ] }
```

This configuration will change the [name](#metric-name) and [stats](#stats) recorded for counter metrics, but
the [tags](#metric-tags) and *timestamp* from `kamon.opentsdb.default` will be used.

Individual metric can be configured by add entries to `kamon.opentsdb.metrics`. 
Any values not set on the metric level, will be inherited from first the category and then defaults. 

```typesafeconfig
kamon.opentsdb.metrics."my.metric.name" = { stats = [ 90, 95, 99, 99.9, 99.999 ], filterIdle = false }
```

Here the [name](#metric-name), [tags](#metric-tags), and *timestamp* from the defaults will be used, unless
the metric is a "counter", in which case the [name](#metric-name) and [stats](#stats) from above will be used.

### Extensibility ###

You can add static values to your metric names and tags by adding
entries of the format `kamon.opentsdb.rules.<rule-name>.value = "some string"`
These values can be referenced in [name](#metric-name) and [tags](#metric-tags) by using *rule-name*

EX.
```typesafeconfig
kamon.opentsdb.rules.cluster.value = "EC2"
kamon.opentsdb.default.name = [ cluster, application, category, entity, metric, stat]
```

You can create your own dynamic rules by subclassing `kamon.opentsdb.Rule` and adding
an entry of the format `kamon.opentsdb.rules.<name>.generator = "fully.qualified.class.name"`

EX.
```typesafeconfig
kamon.opentsdb.rules.timezone.generator = "leider.ken.application.TimezoneRule"
kamon.opentsdb.tags = { host = host, tz = timezone }
```

You can create you own stats by subclassing `kamon.opentsdb.Stat` and adding
an entry of the format `kamon.opentsdb.stats.<name> = "fully.qualified.class.name"`

EX.
```typesafeconfig
kamon.opentsdb.stats.integral = "leider.ken.application.IntegralStat"
kamon.opentsdb.counter.stats = [ count, rate, integral ] 
```

Visualization and Fun
---------------------

OpenTSDB comes with a web service that provides a REST API and basic visualization. For our internal testing we
choose [Grafana] to create beautiful dashboards.  Check out how your metrics data might look like in Grafana with the screenshots below.

<img class="img-fluid" src="/assets/img/kamon-statsd-grafana.png">

<img class="img-fluid" src="/assets/img/kamon-system-metrics.png">


[StatsD]: https://github.com/etsy/statsd/
[get started]: /documentation/get-started/
[Graphite]: http://graphite.wikidot.com/
[Grafana]: http://grafana.org
[docker image]: https://github.com/kamon-io/docker-grafana-graphite
[Datadog Module]: /documentation/kamon-datadog/0.6.6/overview/
