---
title: StreamNode
note: Auto generated by tickdoc

menu:
  kapacitor_010:
    name: Stream
    identifier: stream_node
    weight: 160
    parent: tick
---

A [StreamNode](/kapacitor/v0.10/tick/stream_node/) represents the source of data being 
streamed to Kapacitor via any of its inputs. 
The stream node allows you to select which portion of the stream 
you want to process. 
The `stream` variable in stream tasks is an instance of 
a [StreamNode.](/kapacitor/v0.10/tick/stream_node/) 

Example: 


```javascript
    stream
        .from()
           .database('mydb')
           .retentionPolicy('myrp')
           .measurement('mymeasurement')
           .where(lambda: "host" =~ /logger\d+/)
        .window()
        ...
```

The above example selects only data points from the database `mydb` 
and retention policy `myrp` and measurement `mymeasurement` where 
the tag `host` matches the regex `logger\d+` 


Properties
----------

Property methods modify state on the calling node. They do not add another node to the pipeline, and always return a reference to the calling node.

### Database

The database name. 
If empty any database will be used. 


```javascript
node.database(value string)
```


### Measurement

The measurement name 
If empty any measurement will be used. 


```javascript
node.measurement(value string)
```


### RetentionPolicy

The retention policy name 
If empty any retention policy will be used. 


```javascript
node.retentionPolicy(value string)
```


### Truncate

Optional duration for truncating timestamps. 
Helpful to ensure data points land on specfic boundaries 
Example: 


```javascript
    stream
       .from().measurement('mydata')
           .truncate(1s)
```

All incoming data will be truncated to 1 second resolution. 


```javascript
node.truncate(value time.Duration)
```


### Where

Filter the current stream using the given expression. 
This expression is a Kapacitor expression. Kapacitor 
expressions are a superset of InfluxQL WHERE expressions. 
See the [expression](https://docs.influxdata.com/kapacitor/v0.10/tick/expr/) docs for more information. 

Multiple calls to the Where method will `AND` together each expression. 

Example: 


```javascript
    stream
       .from()
          .where(lambda: condition1)
          .where(lambda: condition2)
```

The above is equivalent to this 
Example: 


```javascript
    stream
       .from()
          .where(lambda: condition1 AND condition2)
```


NOTE: Becareful to always use `.from` if you want multiple different streams. 

Example: 


```javascript
  var data = stream.from().measurement('cpu')
  var total = data.where(lambda: "cpu" == 'cpu-total')
  var others = data.where(lambda: "cpu" != 'cpu-total')
```

The example above is equivalent to the example below, 
which is obviously not what was intended. 

Example: 


```javascript
  var data = stream
              .from()
                  .measurement('cpu')
                  .where(lambda: "cpu" == 'cpu-total' AND "cpu" != 'cpu-total')
  var total = data
  var others = total
```

The example below will create two different streams each selecting 
a different subset of the original stream. 

Example: 


```javascript
  var data = stream.from().measurement('cpu')
  var total = stream.from().measurement('cpu').where(lambda: "cpu" == 'cpu-total')
  var others = stream.from().measurement('cpu').where(lambda: "cpu" != 'cpu-total')
```


If empty then all data points are considered to match. 


```javascript
node.where(expression tick.Node)
```


Chaining Methods
----------------

Chaining methods create a new node in the pipeline as a child of the calling node. They do not modify the calling node.

### Alert

Create an alert node, which can trigger alerts. 


```javascript
node.alert()
```

Returns: [AlertNode](/kapacitor/v0.10/tick/alert_node/)


### Deadman

Helper function for creating an alert on low throughput, aka deadman&#39;s switch. 

- Threshold -- trigger alert if throughput drops below threshold in points/interval. 
- Interval -- how often to check the throughput. 
- Expressions -- optional list of expressions to also evaluate. Useful for time of day alerting. 

Example: 


```javascript
    var data = stream.from()...
    // Trigger critical alert if the throughput drops below 100 points per 10s and checked every 10s.
    data.deadman(100.0, 10s)
    //Do normal processing of data
    data....
```

The above is equivalent to this 
Example: 


```javascript
    var data = stream.from()...
    // Trigger critical alert if the throughput drops below 100 points per 10s and checked every 10s.
    data.stats(10s)
          .derivative('collected')
              .unit(10s)
              .nonNegative()
          .alert()
              .id('node \'stream0\' in task \'{{ .TaskName }}\'')
              .message('{{ .ID }} is {{ if eq .Level "OK" }}alive{{ else }}dead{{ end }}: {{ index .Fields "collected" | printf "%0.3f" }} points/10s.')
              .crit(lamdba: "collected" <= 100.0)
    //Do normal processing of data
    data....
```

The `id` and `message` alert properties can be configured globally via the &#39;deadman&#39; configuration section. 

Since the [AlertNode](/kapacitor/v0.10/tick/alert_node/) is the last piece it can be further modified as normal. 
Example: 


```javascript
    var data = stream.from()...
    // Trigger critical alert if the throughput drops below 100 points per 1s and checked every 10s.
    data.deadman(100.0, 10s).slack().channel('#dead_tasks')
    //Do normal processing of data
    data....
```

You can specify additional lambda expressions to further constrain when the deadman&#39;s switch is triggered. 
Example: 


```javascript
    var data = stream.from()...
    // Trigger critical alert if the throughput drops below 100 points per 10s and checked every 10s.
    // Only trigger the alert if the time of day is between 8am-5pm.
    data.deadman(100.0, 10s, lambda: hour("time") >= 8 AND hour("time") <= 17)
    //Do normal processing of data
    data....
```



```javascript
node.deadman(threshold float64, interval time.Duration, expr ...tick.Node)
```

Returns: [AlertNode](/kapacitor/v0.10/tick/alert_node/)


### Derivative

Create a new node that computes the derivative of adjacent points. 


```javascript
node.derivative(field string)
```

Returns: [DerivativeNode](/kapacitor/v0.10/tick/derivative_node/)


### Eval

Create an eval node that will evaluate the given transformation function to each data point. 
A list of expressions may be provided and will be evaluated in the order they are given 
and results of previous expressions are made available to later expressions. 


```javascript
node.eval(expressions ...tick.Node)
```

Returns: [EvalNode](/kapacitor/v0.10/tick/eval_node/)


### From

Creates a new stream node that can be further 
filtered using the Database, RetentionPolicy, Measurement and Where properties. 
From can be called multiple times to create multiple 
independent forks of the data stream. 

Example: 


```javascript
    // Select the 'cpu' measurement from just the database 'mydb'
    // and retention policy 'myrp'.
    var cpu = stream.from()
                       .database('mydb')
                       .retentionPolicy('myrp')
                       .measurement('cpu')
    // Select the 'load' measurement from any database and retention policy.
    var load = stream.from()
                        .measurement('load')
    // Join cpu and load streams and do further processing.
    cpu.join(load)
            .as('cpu', 'load')
        ...
```



```javascript
node.from()
```

Returns: [StreamNode](/kapacitor/v0.10/tick/stream_node/)


### GroupBy

Group the data by a set of tags. 

Can pass literal * to group by all dimensions. 
Example: 


```javascript
    .groupBy(*)
```



```javascript
node.groupBy(tag ...interface{})
```

Returns: [StreamNode](/kapacitor/v0.10/tick/stream_node/)


### HttpOut

Create an http output node that caches the most recent data it has received. 
The cached data is available at the given endpoint. 
The endpoint is the relative path from the API endpoint of the running task. 
For example if the task endpoint is at &#34;/api/v1/task/&lt;task_name&gt;&#34; and endpoint is 
&#34;top10&#34;, then the data can be requested from &#34;/api/v1/task/&lt;task_name&gt;/top10&#34;. 


```javascript
node.httpOut(endpoint string)
```

Returns: [HTTPOutNode](/kapacitor/v0.10/tick/http_out_node/)


### InfluxDBOut

Create an influxdb output node that will store the incoming data into InfluxDB. 


```javascript
node.influxDBOut()
```

Returns: [InfluxDBOutNode](/kapacitor/v0.10/tick/influx_d_b_out_node/)


### Join

Join this node with other nodes. The data is joined on timestamp. 


```javascript
node.join(others ...Node)
```

Returns: [JoinNode](/kapacitor/v0.10/tick/join_node/)


### MapReduce

Perform a map-reduce operation on the data. 
The built-in functions under `influxql` provide the 
selection,aggregation, and transformation functions 
from the InfluxQL language. 

MapReduce may be applied to either a batch or a stream edge. 
In the case of a batch each batch is passed to the mapper independently. 
In the case of a stream all incoming data points that have 
the exact same time are combined into a batch and sent to the mapper. 


```javascript
node.mapReduce(mr MapReduceInfo)
```

Returns: [ReduceNode](/kapacitor/v0.10/tick/reduce_node/)


### Sample

Create a new node that samples the incoming points or batches. 

One point will be emitted every count or duration specified. 


```javascript
node.sample(rate interface{})
```

Returns: [SampleNode](/kapacitor/v0.10/tick/sample_node/)


### Stats

Create a new stream of data that contains the internal statistics of the node. 
The interval represents how often to emit the statistics based on real time. 
This means the interval time is independent of the times of the data points the source node is receiving. 


```javascript
node.stats(interval time.Duration)
```

Returns: [StatsNode](/kapacitor/v0.10/tick/stats_node/)


### Union

Perform the union of this node and all other given nodes. 


```javascript
node.union(node ...Node)
```

Returns: [UnionNode](/kapacitor/v0.10/tick/union_node/)


### Window

Create a new node that windows the stream by time. 

NOTE: Window can only be applied to stream edges. 


```javascript
node.window()
```

Returns: [WindowNode](/kapacitor/v0.10/tick/window_node/)

