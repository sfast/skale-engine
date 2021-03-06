# Skale Reference

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Overview](#overview)
- [Core concepts](#core-concepts)
- [Working with datasets](#working-with-datasets)
  - [Sources](#sources)
  - [Transformations](#transformations)
  - [Actions](#actions)
- [Skale module](#skale-module)
  - [skale.context([config])](#skalecontextconfig)
      - [sc.env](#scenv)
      - [sc.end()](#scend)
      - [sc.lineStream(input_stream)](#sclinestreaminput_stream)
      - [sc.objectStream(input_stream)](#scobjectstreaminput_stream)
      - [sc.parallelize(array)](#scparallelizearray)
      - [sc.range(start[, end[, step]])](#scrangestart-end-step)
      - [sc.source(size, callback[, args])](#scsourcesize-callback-args)
      - [sc.textFile(path[, options])](#sctextfilepath-options)
  - [Dataset methods](#dataset-methods)
      - [ds.aggregate(reducer, combiner, init[, obj][, done])](#dsaggregatereducer-combiner-init-obj-done)
      - [ds.aggregateByKey(reducer, combiner, init,[ obj])](#dsaggregatebykeyreducer-combiner-init-obj)
      - [ds.cartesian(other)](#dscartesianother)
      - [ds.coGroup(other)](#dscogroupother)
      - [ds.collect([done])](#dscollectdone)
      - [ds.count([done])](#dscountdone)
      - [ds.countByKey([done])](#dscountbykeydone)
      - [ds.countByValue([done])](#dscountbyvaluedone)
      - [ds.distinct()](#dsdistinct)
      - [ds.filter(filter[, obj])](#dsfilterfilter-obj)
      - [ds.first([done])](#dsfirstdone)
      - [ds.flatMap(flatMapper[, obj])](#dsflatmapflatmapper-obj)
      - [ds.flatMapValues(flatMapper[, obj])](#dsflatmapvaluesflatmapper-obj)
      - [ds.forEach(callback[, obj][, done])](#dsforeachcallback-obj-done)
      - [ds.groupByKey()](#dsgroupbykey)
      - [ds.intersection(other)](#dsintersectionother)
      - [ds.join(other)](#dsjoinother)
      - [ds.keys()](#dskeys)
      - [ds.leftOuterJoin(other)](#dsleftouterjoinother)
      - [ds.lookup(k[, done])](#dslookupk-done)
      - [ds.map(mapper[, obj])](#dsmapmapper-obj)
      - [ds.mapValues(mapper[, obj])](#dsmapvaluesmapper-obj)
      - [ds.partitionBy(partitioner)](#dspartitionbypartitioner)
      - [ds.persist()](#dspersist)
      - [ds.reduce(reducer, init[, obj][, done])](#dsreducereducer-init-obj-done)
      - [ds.reduceByKey(reducer, init[, obj])](#dsreducebykeyreducer-init-obj)
      - [ds.rightOuterJoin(other)](#dsrightouterjoinother)
      - [ds.sample(withReplacement, frac, seed)](#dssamplewithreplacement-frac-seed)
      - [ds.save(url[, options][, done])](#dssaveurl-options-done)
      - [ds.sortBy(keyfunc[, ascending])](#dssortbykeyfunc-ascending)
      - [ds.sortByKey(ascending)](#dssortbykeyascending)
      - [ds.stream([opt])](#dsstreamopt)
      - [ds.subtract(other)](#dssubtractother)
      - [ds.take(num[, done])](#dstakenum-done)
      - [ds.top(num[, done])](#dstopnum-done)
      - [ds.union(other)](#dsunionother)
      - [ds.values()](#dsvalues)
  - [Partitioners](#partitioners)
      - [HashPartitioner(numPartitions)](#hashpartitionernumpartitions)
      - [RangePartitioner(numPartitions, keyfunc, dataset)](#rangepartitionernumpartitions-keyfunc-dataset)
  - [Environment variables](#environment-variables)
- [Machine Learning Module](#machine-learning-module)
  - [ml.binaryClassifiationMetrics(measures, options[, done])](#mlbinaryclassificationmetricsmeasures-options-done)
  - [ml.SGDClassifier(options)](#mlsgdclassifieroptions)
      - [sgdClassifier.fit(trainingSet, iterations[, done])](#sgdclassifierfittrainingset-iterations-done)
      - [sgdClassifier.predict(sample)](#sgdclassifierpredictsample)
  - [ml.StandardScaler()](#mlstandardscaler)
      - [standardScaler.fit(dataset[, done])](#standardscalerfitdataset-done)
      - [standardScaler.transform(features)](#standardscalertransformfeatures)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

Skale is a fast and general purpose distributed data processing
system. It provides a high-level API in Javascript and an optimized
parallel execution engine.

A Skale application consists of a *master* program that runs the
user code and executes various *parallel operations* on a cluster
of *workers*.

The main abstraction Skale provides is a *dataset* which is similar
to a Javascript *array*, but partitioned accross the workers that
can be operated in parallel.

There are several ways to create a dataset: *parallelizing* an existing
array in the master program, or referencing a dataset in a distributed
storage system (such as HDFS), or *streaming* the content of any
source that can be processed through Node.js *Streams*. We call
*source* a function which initializes a dataset.

Datasets support two kinds of operations: *transformations*, which create
a new dataset from an existing one, and *actions*, which
return a value to the *master* program after running a computation
on the dataset.

For example, `map` is a transformation that applies a function to
each element of a dataset, returning a new dataset. On the other
hand, `reduce` is an action that aggregates all elements of a dataset
using some function, and returns the final result to the master.

*Sources* and *transformations* in Skale are *lazy*. They do not
start right away, but are triggered by *actions*, thus allowing
efficient pipelined execution and optimized data transfers.

A first example:

```javascript
var sc = require('skale-engine').context();		// create a new context
sc.parallelize([1, 2, 3, 4]).				// source
   map(function (x) {return x+1}).			// transform
   reduce(function (a, b) {return a+b}, 0).	// action
   then(console.log);						// process result: 14
```

## Core concepts

As stated above, a program can be considered as a workflow of steps,
each step consisting of a transformation which inputs from one or
more datasets (parents), and outputs to a new dataset (child).

### Partitioning

Datasets are divided into several partitions, so each partition can
be assigned to a separate worker, and processing can occur concurently
in a distributed and parallel system.

The consequence of this partitioning is that two types of transformations
exist:

- *Narrow* transformations, where each partition of the parent dataset
  is used by at most one partition of the child dataset. This is the
  case for example for `map()` or `filter()`, where each dataset entry
  is processed independently from each other.
  Partitions are decoupled, no synchronization
  between workers is required, and narrow transformations can be
  pipelined on each worker.

- *Wide* transformations, where multiple child partitions may depend
  on one parent partition. This is the case for example for `sortBy()`
  or `groupByKey()`. Data need to be exchanged between workers or
  *shuffled*, in order to complete the transformation. This introduces
  synchronization points which prevent pipelining.

### Pipeline stages and shuffles

Internally, each wide transformation consists of a pre-shuffle and
a post-shuffle part. All sequences of steps from source to pre-shuffle,
or from post-shuffle to next pre-shuffle or action, are thus only
narrow transformations, or pipelined stages (the most efficient
pattern).  A skale program is therefore simply a sequence of stages
and shuffles, shuffles being global serialization points.

It's important to grab this concept as it sets the limit to the
level of parallelism which can be achieved by a given code.

The synoptic table of [transformations](#transformations) indicates
for each transformation if it is narrow or wide (shuffle).

## Working with datasets

### Sources

After having initialized a cluster context using
[skale.context()](#skale-context), one can create a dataset
using the following sources:

| Source Name                                         | Description                                            |
| ------------------------------------------------------- | ------------------------------------------------------ |
|[lineStream(stream)](#sclinestreaminput_stream)          | Create a dataset from a text stream                    |
|[objectStream(stream)](#scobjectstreaminput_stream)      | Create a dataset from an object stream                 |
|[parallelize(array)](#scparallelizearray)                | Create a dataset from an array                         |
|[range(start,end,step)](#scrangestart-end-step)          | Create a dataset containing integers from start to end |
|[source(size,callback,args)](#scsourcesize-callback-args)| Create a dataset from a custom source function         |
|[textFile(path[, options])](#sctextfilepath-options)     | Create a dataset from text file                        |

### Transformations

Transformations operate on a dataset and return a new dataset. Note that some
transformation operate only on datasets where each element is in the form
of 2 elements array of key and value (`[k,v]` dataset):

	[[Ki,Vi], ..., [Kj, Vj]]

A special transformation `persist()` enables one to *persist* a dataset
in memory, allowing efficient reuse accross parallel operations.

|Transformation Name              | Description                                   | In    | Out   |Shuffle|
| -----------------               |-----------------------------------------------|-------|-------|-------|
|[aggregateByKey(func, func, init)](#dsaggregatebykeyreducer-combiner-init-obj)| reduce and combine by key using functions| [k,v]| [k,v]|yes|
|[cartesian(other)](#dscartesianother) | Perform a cartesian product with the other dataset | v w | [v,w]|yes|
|[coGroup(other)](#dscogroupother) | Group data from both datasets sharing the same key | [k,v] [k,w] |[k,[[v],[w]]]|yes|
|[distinct()](#dsdistinct)    | Return a dataset where duplicates are removed | v | w|yes|
|[filter(func)](#dsfilterfilter-obj)| Return a dataset of elements on which function returns true | v | w|no|
|[flatMap(func)](#dsflatmapflatmapper-obj)| Pass the dataset elements to a function which returns a sequence | v | w|no|
|[flatMapValues(func)](#dsflatmapflatvaluesmapper-obj)| Pass the dataset [k,v] elements to a function without changing the keys | [k,v] | [k,w]|no|
|[groupByKey()](#dsgroupbykey)| Group values with the same key | [k,v] | [k,[v]]|yes|
|[intersection(other)](#dsintersectionother) | Return a dataset containing only elements found in both datasets | v w | v|yes|
|[join(other)](#dsjoinother)       | Perform an inner join between 2 datasets | [k,v] | [k,[v,w]]|yes|
|[leftOuterJoin(other)](#dsleftouterjoinother) | Join 2 datasets where the key must be present in the other | [k,v] | [k,[v,w]]|yes|
|[rightOuterJoin(other)](#dsrightouterjoinother) | Join 2 datasets where the key must be present in the first | [k,v] | [k,[v,w]]|yes|
|[keys()](#dskeys)            | Return a dataset of just the keys | [k,v] | k|no|
|[map(func)](#dsmapmapper-obj) | Return a dataset where elements are passed through a function | v | w|no|
|[mapValues(func)](#dsmapvaluesmapper-obj)| Map a function to the value field of key-value dataset | [k,v] | [k,w]|no|
|[reduceByKey(func, init)](#dsreducebykeyreducer-init-obj)| Combine values with the same key | [k,v] | [k,w]|yes|
|[partitionBy(partitioner)](#dspartitionbypartitioner)| Partition using the partitioner | v | v|yes|
|[persist()](#dspersist)      | Idempotent. Keep content of dataset in cache for further reuse. | v | v|no|
|[sample(rep, frac, seed)](#dssamplewithreplacement-frac-seed) | Sample a dataset, with or without replacement | v | w|no|
|[sortBy(func)](#dssortbykeyfunc-ascending) | Sort a dataset | v | v|yes|
|[sortByKey()](#dssortbykeyascending) | Sort a [k,v] dataset | [k,v] | [k,v]|yes|
|[subtract(other)](#dssubtractother) | Remove the content of one dataset | v w | v|yes|
|[union(other)](#dsunionother)     | Return a dataset containing elements from both datasets | v | v w|no|
|[values()](#dsvalues)        | Return a dataset of just the values | [k,v] | v|no|

### Actions

Actions operate on a dataset and send back results to the *master*. Results
are always produced asynchronously and send to an optional callback function,
alternatively through a returned [ES6 promise].

|Action Name | Description | out|
|------------------             |----------------------------------------------|--------------|
|[aggregate(func, func, init)](#dsaggregatereducer-combiner-init-obj-done)| Similar to reduce() but may return a different type| value |
|[collect()](#dscollectdone)         | Return the content of dataset | array of elements|
|[count()](#dscountdone)             | Return the number of elements from dataset | number|
|[countByKey()](#dscountbykeydone)     | Return the number of occurrences for each key in a `[k,v]` dataset | array of [k,number]|
|[countByValue()](#dscountbyvaluedone) | Return the number of occurrences of elements from dataset | array of [v,number]|
|[first()](#dsfirstdone)               | Return the first element in dataset | value |
|[forEach(func)](#dsforeachcallback-obj-done)| Apply the provided function to each element of the dataset | empty |
|[lookup(k)](#dslookupk-done)          | Return the list of values `v` for key `k` in a `[k,v]` dataset | array of v|
|[reduce(func, init)](#dsreducereducer-init-obj-done)| Aggregates dataset elements using a function into one value | value|
|[save(url)](#dssaveurl-options-done)       | Save the content of a dataset to an url | empty |
|[stream()](#dsstream-opt)            | Stream out a dataset | stream |
|[take(num)](#dstakenum-done)         | Return the first `num` elements of dataset | array of value|
|[top(num)](#dstopnum-done)           | Return the top `num` elements of dataset | array of value|

## Skale module

The Skale module is the main entry point for Skale functionality.
To use it, one must `require('skale-engine')`.

### skale.context([config])

Creates and returns a new context which represents the connection
to the Skale cluster, and which can be used to create datasets on that
cluster. Config is an *Object* which defines the cluster server,
with the following defaults:

```javascript
{
  host: 'localhost',	// Cluster server host, settable also by SKALE_HOST env
  port: '12346'			// Cluster server port, settable also by SKALE_PORT env
}
```

Example:

```javascript
var skale = require('skale-engine');
var sc = skale.context();
```

#### sc.env

The `sc.env` property returns an object containing user environment variables
to be set in workers.

To set and propagate an environment variable to all workers, assign `sc.env` object
prior to invoking an action.

Example:

```javascript
sc.env.MY_VAR = 'my_value';
```

#### sc.end()

Closes the connection to the cluster.

#### sc.lineStream(input_stream)

Returns a new dataset of lines of text read from input_stream
*Object*, which is a [readable stream] where dataset content is
read from.

The following example computes the size of a file using streams:

```javascript
var stream = fs.createReadStream('data.txt', 'utf8');
sc.lineStream(stream).
   map(s => s.length).
   reduce((a, b) => a + b, 0).
   then(console.log);
```

#### sc.objectStream(input_stream)

Returns a new dataset of Javascript *Objects* read from input_stream
*Object*, which is a [readable stream] where dataset content is
read from.

The following example counts the number of objects returned in an
object stream using the mongodb native Javascript driver:

```javascript
var cursor = db.collection('clients').find();
sc.objectStream(cursor).count().then(console.log);
```

#### sc.parallelize(array)

Returns a new dataset containing elements from the *Array* array.

Example:

```javascript
var a = sc.parallelize(['Hello', 'World']);
```

#### sc.range(start[, end[, step]])

Returns a new dataset of integers from *start* to *end* (exclusive)
increased by *step* (default 1) every element. If called with a
single argument, the argument is interpreted as *end*, and *start*
is set to 0.

```javascript
sc.range(5).collect().then(console.log)
// [ 0, 1, 2, 3, 4 ]
sc.range(2, 4).collect().then(console.log)
// [ 2, 3 ]
sc.range(10, -5, -3).collect().then(console.log)
// [ 10, 7, 4, 1, -2 ]
```

#### sc.source(size, callback[, args])

Returns a new dataset of *size* elements, where each element is
generated by a custom function *callback* executed on workers.

- *size*: an integer *Number* of elements in the dataset
- *callback*: a function of the form `function(index, args[, wc])`
  which returns the next element and with:
    - *index*: the index of the element in the dataset, comprised
      between `0` and `size - 1`
    - *args*: the same parameter *args* passed to source
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies
- *args*: user custom parameter, passed to *callback*

```javascript
function randArray(index, len) {
  var arr = [];
  for (var i = 0; i < len; i++)
    arr.push(Math.floor(Math.random() * 100));
  return arr;
}

sc.source(3, randArray, 2).collect().then(console.log);
// [ [ 31, 85 ], [ 93, 21 ], [ 99, 58 ] ]
```

#### sc.textFile(path[, options])

Returns a new dataset of lines in file specified by path *String*.

- *path*: a *String* of the general form `protocol://host/path` or `/path`,
  where protocol can be one of:
  - *file*: if path is on local filesystem
  - *s3*: if path relates to a repository on AWS [S3] storage system
  - *wasb*: if path relates to a repository on Azure blob storage system
- *options*: an *Object* with the following fields:
  - *maxFiles*: a *Number* of maximum files to process if the path refers
    to a directory.
  - *parquet*: a *Boolean* to indicate that all files are in the
    [parquet] format. Default value is *false*.

if *path* ends by a '/' (directory separator), then the dataset
will be composed of all the files in the directory. Sub-directories
are not supported. Wildcard characters such as `*`, `?`, etc, as
in the Unix Shell globbing patterns are supported.

If a file name ends by '.gz', then its content will be automatically
uncompressed using GZIP.

If a file name ends by '.parquet', it will automatically be processed
as a [parquet].

Note: If using a path on the local filesystem, the file must also
be accessible at the same path on worker nodes. Either copy the
file to all workers or use a network-mounted shared file system.

For example, the following program prints the length of a text file:

```javascript
var lines = sc.textFile('data.txt');
lines.map(s => s.length).reduce((a, b) => a + b, 0).then(console.log);
```

### Dataset methods

Dataset objects, as created initially by above skale context source
functions, have the following methods, allowing either to instantiate
a new dataset through a transformation, or to return results to the
master program.

#### ds.aggregate(reducer, combiner, init[, obj][, done])

This [action] computes the aggregated value of the elements
of the dataset using two functions *reducer()* and *combiner()*,
allowing to use an arbitrary accumulator type, different from element
type (as opposed to `reduce()` which imposes the same type for
accumulator and element).
The result is passed to the *done()* callback if provided, otherwise an
[ES6 promise] is returned.

- *reducer*: a function of the form `function(acc, val[, obj[, wc]])`,
  which returns the next value of the accumulator (which must be
  of the same type as *acc*) and with:
     - *acc*: the value of the accumulator, initially set to *init*
     - *val*: the value of the next element of the dataset on which
       `aggregate()` operates
     - *obj*: the same parameter *obj* passed to `aggregate()`
     - *wc*: the worker context, a persistent object local to each
       worker, where user can store and access worker local dependencies.
- *combiner*: a function of the form `function(acc1, acc2[, obj])`,
  which returns the merged value of accumulators and with:
     - *acc1*: the value of an accumulator, computed locally on a worker
     - *acc2*: the value of an other accumulator, issued by another worker
     - *obj*: the same parameter *obj* passed to `aggregate()`
- *init*: the initial value of the accumulators that are used by
  *reducer()* and *combiner()*. It should be the identity element
  of the operation (a neutral zero value, i.e. applying it through the
  function should not change result).
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset.
- *done*: a callback of the form `function(error, result)` which is
  called at completion. If *undefined*, `aggregate()` returns an
  [ES6 promise].

The following example computes the average of a dataset, avoiding a `map()`:

```javascript
sc.parallelize([3, 5, 2, 7, 4, 8]).
   aggregate((a, v) => [a[0] + v, a[1] + 1],
	(a1, a2) => [a1[0] + a2[0], a1[1] + a2[1]], [0, 0]).
   then(function(data) {
	console.log(data[0] / data[1]);
  })
// 4.8333
```

#### ds.aggregateByKey(reducer, combiner, init,[ obj])

When called on a dataset of type `[k,v]`, returns a dataset of type
`[k,v]` where `v` is the aggregated value of all elements of same
key `k`. The aggregation is performed using two functions *reducer()*
and *combiner()* allowing to use an arbitrary accumulator type,
different from element type.

- *reducer*: a function of the form `function(acc, val[, obj[, wc]])`,
  which returns the next value of the accumulator (which must be
  of the same type as *acc*) and with:
     - *acc*: the value of the accumulator, initially set to *init*
     - *val*: the value `v` of the next `[k,v]` element of the dataset
	   on which `aggregateByKey()` operates
     - *obj*: the same parameter *obj* passed to `aggregateByKey()`
     - *wc*: the worker context, a persistent object local to each
       worker, where user can store and access worker local dependencies.
- *combiner*: a function of the form `function(acc1, acc2[, obj])`,
  which returns the merged value of accumulators and with:
     - *acc1*: the value of an accumulator, computed locally on a worker
     - *acc2*: the value of an other accumulator, issued by another worker
     - *obj*: the same parameter *obj* passed to `aggregate()`
- *init*: the initial value of the accumulators that are used by
  *reducer()* and *combiner()*. It should be the identity element
  of the operation (a neutral zero value, i.e. applying it through the
  function should not change result).
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset.

Example:

```javascript
sc.parallelize([['hello', 1], ['hello', 1], ['world', 1]]).
   aggregateByKey((a, b) => a + b, (a, b) => a + b, 0).
   collect().then(console.log);
// [ [ 'hello', 2 ], [ 'world', 1 ] ]
```

#### ds.cartesian(other)

Returns a dataset wich contains all possible pairs `[a, b]` where `a`
is in the source dataset and `b` is in the *other* dataset.

Example:

```javascript
var ds1 = sc.parallelize([1, 2]);
var ds2 = sc.parallelize(['a', 'b', 'c']);
ds1.cartesian(ds2).collect().then(console.log);
// [ [ 1, 'a' ], [ 1, 'b' ], [ 1, 'c' ],
//   [ 2, 'a' ], [ 2, 'b' ], [ 2, 'c' ] ]
```

#### ds.coGroup(other)

When called on dataset of type `[k,v]` and `[k,w]`, returns a dataset of type
`[k, [[v], [w]]]`, where data of both datasets share the same key.

Example:

```javascript
var ds1 = sc.parallelize([[10, 1], [20, 2]]);
var ds2 = sc.parallelize([[10, 'world'], [30, 3]]);
ds1.coGroup(ds2).collect().then(console.log);
// [ [ 10, [ [ 1 ], [ 'world' ] ] ],
//   [ 20, [ [ 2 ], [] ] ],
//   [ 30, [ [], [ 3 ] ] ] ]
```

#### ds.collect([done])

This [action] returns the content of the dataset in form of an array.
The result is passed to the *done()* callback if provided, otherwise an
[ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([1, 2, 3, 4]).
   collect(function (err, res) {
     console.log(res);
   });
// [ 1, 2, 3, 4 ]
```

#### ds.count([done])

This [action] computes the number of elements in the dataset. The
result is passed to the *done()* callback if provided, otherwise
an [ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([10, 20, 30, 40]).count().then(console.log);
// 4
```

#### ds.countByKey([done])

When called on a dataset of type `[k,v]`, this [action] computes
the number of occurrences of elements for each key in a dataset of
type `[k,v]`. It produces an array of elements of type `[k,w]` where
`w` is the result count.  The result is passed to the *done()*
callback if provided, otherwise an [ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([[10, 1], [20, 2], [10, 4]]).
   countByKey().then(console.log);
// [ [ 10, 2 ], [ 20, 1 ] ]
```

#### ds.countByValue([done])

This [action] computes the number of occurences of each element in
dataset and returns an array of elements of type `[v,n]` where `v`
is the element and `n` its number of occurrences.  The result is
passed to the *done()* callback if provided, otherwise an [ES6
promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([ 1, 2, 3, 1, 3, 2, 5 ]).
   countByValue().then(console.log);
// [ [ 1, 2 ], [ 2, 2 ], [ 3, 2 ], [ 5, 1 ] ]
```

#### ds.distinct()

Returns a dataset where duplicates are removed.

Example:

```javascript
sc.parallelize([ 1, 2, 3, 1, 4, 3, 5 ]).
   distinct().
   collect().then(console.log);
// [ 1, 2, 3, 4, 5 ]
```

#### ds.filter(filter[, obj])

- *filter*: a function of the form `callback(element[, obj[, wc]])`,
  returning a *Boolean* and where:
    - *element*: the next element of the dataset on which `filter()` operates
    - *obj*: the same parameter *obj* passed to `filter()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

Applies the provided filter function to each element of the source
dataset and returns a new dataset containing the elements that passed the
test.

Example:

```javascript
function filter(data, obj) { return data % obj.modulo; }

sc.parallelize([1, 2, 3, 4]).
   filter(filter, {modulo: 2}).
   collect().then(console.log);
// [ 1, 3 ]
```

#### ds.first([done])

This [action] computes the first element in this dataset.
The result is passed to the *done()* callback if provided, otherwise an
[ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

```javascript
sc.parallelize([1, 2, 3]).first().then(console.log);
// 1
```

#### ds.flatMap(flatMapper[, obj])

Applies the provided mapper function to each element of the source
dataset and returns a new dataset.

- *flatMapper*: a function of the form `callback(element[, obj[, wc]])`,
  returning an *Array* and where:
    - *element*: the next element of the dataset on which `flatMap()` operates
    - *obj*: the same parameter *obj* passed to `flatMap()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

Example:

```javascript
sc.range(5).flatMap(a => [a, a]).collect().then(console.log);
// [ 0, 0, 1, 1, 2, 2, 3, 3, 4, 4 ]
```

#### ds.flatMapValues(flatMapper[, obj])

Applies the provided flatMapper function to the value of each [key,
value] element of the source dataset and return a new dataset containing
elements defined as [key, mapper(value)], keeping the key unchanged
for each source element.

- *flatMapper*: a function of the form `callback(element[, obj[, wc]])`,
  returning an *Array* and where:
    - *element*: the value v of the next [k,v] element of the dataset on
      which `flatMapValues()` operates
    - *obj*: the same parameter *obj* passed to `flatMapValues()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

Example:

```javascript
function valueFlatMapper(data, obj) {
	var tmp = [];
	for (var i = 0; i < obj.N; i++) tmp.push(data * obj.fact);
	return tmp;
}

sc.parallelize([['hello', 1], ['world', 2]]).
   flatMapValues(valueFlatMapper, {N: 2, fact: 2}).
   collect().then(console.log);
// [ [ 'hello', 2 ], [ 'hello', 2 ], [ 'world', 4 ], [ 'world', 4 ] ]
```

#### ds.forEach(callback[, obj][, done])

This [action] applies a *callback* function on each element of the dataset.
If provided, the *done()* callback is invoked at completion, otherwise an
[ES6 promise] is returned.

- *callback*: a function of the form `function(val[, obj[, wc]])`,
  which returns *null* and with:
     - *val*: the value of the next element of the dataset on which
       `forEach()` operates
     - *obj*: the same parameter *obj* passed to `forEach()`
     - *wc*: the worker context, a persistent object local to each
       worker, where user can store and access worker local dependencies.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset
- *done*: a callback of the form `function(error, result)` which is
  called at completion.


In the following example, the `console.log()` callback provided
to `forEach()` is executed on workers and may be not visible:

```javascript
sc.parallelize([1, 2, 3, 4]).
   forEach(console.log).then(console.log('finished'));
```

#### ds.groupByKey()

When called on a dataset of type `[k,v]`, returns a dataset of type `[k, [v]]`
where values with the same key are grouped.

Example:

```javascript
sc.parallelize([[10, 1], [20, 2], [10, 4]]).
   groupByKey().collect().then(console.log);
// [ [ 10, [ 1, 4 ] ], [ 20, [ 2 ] ] ]
```

#### ds.intersection(other)

Returns a dataset containing only elements found in source dataset and *other*
dataset.

Example:

```javascript
var ds1 = sc.parallelize([1, 2, 3, 4, 5]);
var ds2 = sc.parallelize([3, 4, 5, 6, 7]);
ds1.intersection(ds2).collect().then(console.log); // [ 3, 4, 5 ]
```

#### ds.join(other)

When called on source dataset of type `[k,v]` and *other* dataset of type
`[k,w]`, returns a dataset of type `[k, [v, w]]` pairs with all pairs
of elements for each key.

Example:

```javascript
var ds1 = sc.parallelize([[10, 1], [20, 2]]);
var ds2 = sc.parallelize([[10, 'world'], [30, 3]]);
ds1.join(ds2).collect().then(console.log);
// [ [ 10, [ 1, 'world' ] ] ]
```

#### ds.keys()

When called on source dataset of type `[k,v]`, returns a dataset with just
the elements `k`.

Example:

```javascript
sc.parallelize([[10, 'world'], [30, 3]]).
   keys.collect().then(console.log);
// [ 10, 30 ]
```

#### ds.leftOuterJoin(other)

When called on source dataset of type `[k,v]` and *other* dataset of type
`[k,w]`, returns a dataset of type `[k, [v, w]]` pairs where the key
must be present in the *other* dataset.

Example:

```javascript
var ds1 = sc.parallelize([[10, 1], [20, 2]]);
var ds2 = sc.parallelize([[10, 'world'], [30, 3]]);
ds1.leftOuterJoin(ds2).collect().then(console.log);
// [ [ 10, [ 1, 'world' ] ], [ 20, [ 2, null ] ] ]
```

#### ds.lookup(k[, done])

When called on source dataset of type `[k,v]`, returns an array
of values `v` for key `k`.
The result is passed to the *done()* callback if provided, otherwise an
[ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([[10, 'world'], [20, 2], [10, 1], [30, 3]]).
   lookup(10).then(console.log);
// [ world, 1 ]
```

#### ds.map(mapper[, obj])

Applies the provided mapper function to each element of the source
dataset and returns a new dataset.

- *mapper*: a function of the form `callback(element[, obj[, wc]])`,
  returning an element and where:
    - *element*: the next element of the dataset on which `map()` operates
    - *obj*: the same parameter *obj* passed to `map()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

Example:

```javascript
sc.parallelize([1, 2, 3, 4]).
   map((data, obj) => data * obj.scaling, {scaling: 1.2}).
   collect().then(console.log);
// [ 1.2, 2.4, 3.6, 4.8 ]
```

#### ds.mapValues(mapper[, obj])

- *mapper*: a function of the form `callback(element[, obj[, wc]])`,
  returning an element and where:
    - *element*: the value v of the next [k,v] element of the dataset on
      which `mapValues()` operates
    - *obj*: the same parameter *obj* passed to `mapValues()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

Applies the provided mapper function to the value of each `[k,v]`
element of the source dataset and return a new dataset containing elements
defined as `[k, mapper(v)]`, keeping the key unchanged for each
source element.

Example:

```javascript
sc.parallelize([['hello', 1], ['world', 2]]).
   mapValues((a, obj) => a*obj.fact, {fact: 2}).
   collect().then(console.log);
// [ ['hello', 2], ['world', 4] ]
```

#### ds.partitionBy(partitioner)

Returns a dataset partitioned using the specified partitioner. The
purpose of this transformation is not to change the dataset content,
but to increase processing speed by ensuring that the elements
accessed by further transfomations reside in the same partition.

Example:

```javascript
var skale = require('skale-engine');
var sc = skale.context();

sc.parallelize([['hello', 1], ['world', 1], ['hello', 2], ['world', 2], ['cedric', 3]])
  .partitionBy(new skale.HashPartitioner(3))
  .collect.then(console.log)
// [ ['world', 1], ['world', 2], ['hello', 1], ['hello', 2], ['cedric', 3] ]
```

#### ds.persist()

Returns the dataset, and persists the dataset content on disk (and
in memory if available) in order to directly reuse content in further
tasks.

Example:

```javascript
var dataset = sc.range(100).map(a => a * a);

// First action: compute dataset
dataset.collect().then(console.log)

// Second action: reuse dataset, avoid map transform
dataset.collect().then(console.log)
```

#### ds.reduce(reducer, init[, obj][, done])

This [action] returns the aggregated value of the elements
of the dataset using a *reducer()* function.
The result is passed to the *done()* callback if provided, otherwise an
[ES6 promise] is returned.

- *reducer*: a function of the form `function(acc, val[, obj[, wc]])`,
  which returns the next value of the accumulator (which must be
  of the same type as *acc* and *val*) and with:
     - *acc*: the value of the accumulator, initially set to *init*
     - *val*: the value of the next element of the dataset on which
       `reduce()` operates
     - *obj*: the same parameter *obj* passed to `reduce()`
     - *wc*: the worker context, a persistent object local to each
       worker, where user can store and access worker local dependencies.
- *init*: the initial value of the accumulators that are used by
  *reducer()*. It should be the identity element of the operation
  (i.e. applying it through the function should not change result).
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset
- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.parallelize([1, 2, 4, 8]).
   reduce((a, b) => a + b, 0).
   then(console.log);
// 15
```

#### ds.reduceByKey(reducer, init[, obj])

- *reducer*: a function of the form `callback(acc,val[, obj[, wc]])`,
  returning the next value of the accumulator (which must be of the
  same type as *acc* and *val*) and where:
    - *acc*: the value of the accumulator, initially set to *init*
    - *val*: the value `v` of the next `[k,v]` element of the dataset on
      which `reduceByKey()` operates
    - *obj*: the same parameter *obj* passed to `reduceByKey()`
    - *wc*: the worker context, a persistent object local to each
      worker, where user can store and access worker local dependencies.
- *init*: the initial value of accumulator for each key. Will be
  passed to *reducer*.
- *obj*: user provided data. Data will be passed to carrying
  serializable data from master to workers, obj is shared amongst
  mapper executions over each element of the dataset

When called on a dataset of type `[k,v]`, returns a dataset of type `[k,v]`
where the values of each key are aggregated using the *reducer*
function and the *init* initial value.

Example:

```javascript
sc.parallelize([[10, 1], [10, 2], [10, 4]]).
   reduceByKey((a,b) => a+b, 0).
   collect().then(console.log);
// [ [10, 7] ]
```

#### ds.rightOuterJoin(other)

When called on source dataset of type `[k,v]` and *other* dataset of type
`[k,w]`, returns a dataset of type `[k, [v, w]]` pairs where the key
must be present in the *source* dataset.

Example:

```javascript
var ds1 = sc.parallelize([[10, 1], [20, 2]]);
var ds2 = sc.parallelize([[10, 'world'], [30, 3]]);
ds1.rightOuterJoin(ds2).collect().then(console.log);
// [ [ 10, [ 1, 'world' ] ], [ 30, [ null, 2 ] ] ]
```

#### ds.sample(withReplacement, frac, seed)

- *withReplacement*: *Boolean* value, *true* if data must be sampled
  with replacement
- *frac*: *Number* value of the fraction of source dataset to return
- *seed*: *Number* value of pseudo-random seed

Returns a dataset by sampling a fraction *frac* of source dataset, with or
without replacement, using a given random generator *seed*.

Example:

```javascript
sc.parallelize([1, 2, 3, 4, 5, 6, 7, 8]).
   sample(true, 0.5, 0).
   collect().then(console.log);
// [ 1, 1, 3, 4, 4, 5, 7 ]
```

#### ds.save(url[, options][, done])

This [action] saves the content of the dataset to the destination URL. The
destination is a flat directory which will contain as many files as partitions
in the dataset. Files are named from partition numbers, starting at 0.
The file format is a stream of JSON strings (one per dataset
element) separated by newlines.

- *url*: a *String* of the general form `protocol://host/path` or `/path`. See
  below for supported protocols
- *options*: an *Object* with the following fields:
	- *gzip*: *Boolean* (default false) to enable gzip compression. If compression
	  is enabled, files are suffixed with `.gz`
- *done*: an optional callback function of the form `function(error, result)`
  called at completion. If not provided, an [ES6 promise] is returned.

##### File protocol

The URL form is `file://path` or simply `path` where *path* is an absolute
pathname in the master host local file system.

Example:

```javascript
sc.range(300).save('/tmp/results/').then(sc.end());
// will produce /tmp/results/0, /tmp/results/1
```

##### AWS S3 protocol

The URL form is `s3://bucket/key`. AWS credentials must be provided by environment
variables i.e `AWS_SECRET_ACCESS_KEY`, `AWS_ACCESS_KEY_ID`.

Example:

```javascript
sc.range(300).save('s3://myproject/mydataset', {gzip: true}).then(sc.end());
// will produce https://myproject.s3.amazonaws.com/mydataset/0.gz
```

##### Azure blob storage protocol

The URL form is `wasb://container@user.blob.core.windows.net/blob_name`. Azure
credentials must be provided by environment variables i.e
`AZURE_STORAGE_ACCOUNT` and `AZURE_STORAGE_ACCESS_KEY`.

#### ds.sortBy(keyfunc[, ascending])

Returns a dataset sorted by the given *keyfunc*.

- *keyfunc*: a function of the form `function(element)` which returns
  a value used for comparison in the sort function and where `element`
  is the next element of the dataset on which `sortBy()` operates
- *ascending*: a boolean to set the sort direction. Default: true

Example:

```javascript
sc.parallelize([4, 6, 10, 5, 1, 2, 9, 7, 3, 0])
  .sortBy(a => a)
  .collect().then(console.log)
// [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

#### ds.sortByKey(ascending)

When called on a dataset of type `[k,v]`, returns a dataset of type `[k,v]`
sorted on `k`. The optional parameter *ascending* is a boolean which sets
the sort direction, true by default.

Example:

```javascript
sc.parallelize([['world', 2], ['cedric', 3], ['hello', 1]])
  .sortByKey()
  .collect().then(console.log)
// [['cedric', 3], ['hello', 1], ['world', 2]]
```

#### ds.stream([opt])

This [action] returns a [readable stream] of dataset content. The order
of data and partitions is maintained.

- *opt*: an object with the following fields:
  - *end*: *Boolean*, when true, call `sc.end()` on stream `end` event. Default value: false.
  - *gzip*: *Boolean*, when true, enable gzip compression. Default value: false.

Example:

```javascript
var s = sc.range(4).stream();
s.pipe(process.stdout);
// 0
// 1
// 2
// 3
```

#### ds.subtract(other)

Returns a dataset containing only elements of source dataset which
are not in *other* dataset.

Example:

```javascript
var ds1 = sc.parallelize([1, 2, 3, 4, 5]);
var ds2 = sc.parallelize([3, 4, 5, 6, 7]);
ds1.subtract(ds2).collect().then(console.log);
// [ 1, 2 ]
```

#### ds.take(num[, done])

This [action] returns an array of the `num` first elements of the
source dataset.  The result is passed to the *done()* callback if
provided, otherwise an [ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.range(5).take(2).then(console.log);
// [1, 2]
```

#### ds.top(num[, done])

This [action] returns an array of the `num` top elements of the
source dataset.  The result is passed to the *done()* callback if
provided, otherwise an [ES6 promise] is returned.

- *done*: a callback of the form `function(error, result)` which is
  called at completion.

Example:

```javascript
sc.range(5).top(2).then(console.log);
// [3, 4]
```

#### ds.union(other)

Returns a dataset that contains the union of the elements in the source
dataset and the *other* dataset.

Example:

```javascript
var ds1 = sc.parallelize([1, 2, 3, 4, 5]);
var ds2 = sc.parallelize([3, 4, 5, 6, 7]);
ds1.union(ds2).collect().then(console.log);
// [ 1, 2, 3, 4, 5, 3, 4, 5, 6, 7 ]
```

#### ds.values()

When called on source dataset of type `[k,v]`, returns a dataset with just
the elements `v`.

Example:

```javascript
sc.parallelize([[10, 'world'], [30, 3]]).
   keys.collect().then(console.log);
// [ 'world', 3 ]
```

### Partitioners

A partitioner is an object passed to
[ds.partitionBy(partitioner)](#dspartitionbypartitioner) which
places data in partitions according to a strategy, for example hash
partitioning, where data having the same key are placed in the same
partition, or range partitioning, where data in the same range are
in the same partition. This is useful to accelerate processing, as
it limits data transfers between workers during jobs.

A partition object must provide the following properties:

- *numPartitions*: a *Number* of partitions for the dataset
- *getPartitionIndex*: a *Function* of type `function(element)`
  which returns the partition index (comprised between 0 and
  *numPartitions*) for the `element` of the dataset on which
  `partitionBy()` operates.

#### HashPartitioner(numPartitions)

Returns a partitioner object which implements hash based partitioning
using a hash checksum of each element as a string.

- *numPartitions*: *Number* of partitions for this dataset

Example:

```javascript
var hp = new skale.HashPartitioner(3)
var dataset = sc.range(10).partitionBy(hp)
```

#### RangePartitioner(numPartitions, keyfunc, dataset)

Returns a partitioner object which first defines ranges by sampling
the dataset and then places elements by comparing them with ranges.

- *numPartitions*: *Number* of partitions for this dataset
- *keyfunc*: a function of the form `function(element)` which returns
  a value used for comparison in the sort function and where `element`
  is the next element of the dataset on which `partitionBy()` operates
- *dataset*: the dataset object on which `partitionBy()` operates

Example:

```javascript
var dataset = sc.range(100)
var rp = new skale.RangePartitioner(3, a => a, dataset)
var dataset = sc.range(10).partitionBy(rp)
```

### Environment variables

- `SKALE_HOST`: The hostname of the skale-server process in distributed mode. If unset, the master runs in standalone mode.
- `SKALE_PORT`: The port of the skale-server process in distributed mode. Default value: "12346"
- `SKALE_KEY`: An authentication token which may be required by the skale-server process
- `SKALE_DEBUG`: set the debug trace level to the following values:
  - `0`: or unset: no traces
  - `1`: debug traces from master side
  - `2`: above traces plus worker traces
  - `3`: above traces plus network protocol traces (if running in distributed mode)

## Machine Learning module

The Machine Learning (ML) module provides scalable functions for
supervised (classification, regression) and unsupervised (clustering)
statistical learning on top of skale datasets and distributed
map-reduce engine.

The module can be loaded using:

```javascript
var ml = require('skale-engine/ml')
```

### ml.binaryClassificationMetrics(measures, options[, done])

### ml.SGDClassifier(options)

Creates a regularized linear classifier model fitted via [stochastic
gradient descent] learning. SGD is sensitive to the scaling
of the features. For best results, the data should have zero mean and
unit variance, which can be achieved with [ml.StandardScaler].

The model it fits can be controlled with the *loss* option; by default,
it fits a linear [support vector machine] (SVM). A regularization term
can be added to the loss, by default the squared euclidean norm L2.

- *options*: an *Object* with the following fields:
  - *fitIntercept*: *Boolean* indicating whether to include an intercept. Default: *true*
  - *loss*: *String* specifying the [loss function] to be used. Possible values are:
      - `hinge`: (default), gives a linear SVM
      - `log`: gives logistic loss, a probabilistic classifier
      - `square`: gives square loss fit
  - *penalty*: *String*  specifying the [regularization] term. Possible values are:
      - `l2`: (default) squared euclidean norm L2, standard regularizer for linear SVM models
      - `l1`: absolute norm L1, might bring sparsity to the model, not achievable with `l2`
      - `none`: zero penalty
  - *proba*: *Boolean* (default *false*). If *true* predict returns a probability rather than a raw number. Only applicable when logisitic loss is selected.
  - *regParam*: *Number*  >= 0, defaults to 0.001, defines the trade-off between the
    two goals of minimizing the loss (i.e. training error) and minimizing model complexity
    (i.e. to avoid overfitting)
  - *stepSize*: *Number* >= 0, defaults to 1, defines the initial step size of the gradient
    descent

Example:

```javascript
var trainingSet = sc.parallelize([
 [1, [0.5, -0.7]],
 [-1 [-0.5, 0.7]]
]);
var model = new ml.SGDClassifier()
await model.fit(trainingSet, 2)
model.weights
// [ 0.8531998372026804, -1.1944797720837526 ]
model.predict([2, -2])
// 0.9836229103782058
```

#### sgdClassifier.fit(trainingSet, iterations[, done])

This [action] updates *sgdClassifier* model by fitting it to the
input dataset *trainingSet*.

- *trainingSet*: a dataset where entries are in the following format:
  `[label, [feature0, feature1, ...]]` with *label* being either 1 or -1,
  and *featureN* being a float number, preferentially with a zero mean and
  unit variance (in range [-1, 1]). Sparse vectors with undefined features
  are supported.

#### sgdClassifier.predict(sample)

### ml.StandardScaler()

#### standardScaler.fit(dataset, [done])

#### standardScaler.transform(features)

[readable stream]: https://nodejs.org/api/stream.html#stream_class_stream_readable
[ES6 promise]: https://promisesaplus.com
[action]: #actions
[loss function]: https://en.wikipedia.org/wiki/Loss_functions_for_classification
[logistic regression]: https://en.wikipedia.org/wiki/Logistic_regression
[ml.StandardScaler]: #mlstandardscaler
[parquet]: https://parquet.apache.org
[regularization]: https://en.wikipedia.org/wiki/Regularization_(mathematics)
[stochastic gradient descent]: https://en.wikipedia.org/wiki/Stochastic_gradient_descent
[support vector machine]: https://en.wikipedia.org/wiki/Support_vector_machine
