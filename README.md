# pyspark-note

# Concept

The records/items/elemets are stored in RDD(s).

Each RDD composists of `Partitions`; each `Partition` contains equal number of `items`/`elements`.

<img src="img\rddDataset-partitions-items.png" height="200"/>

# Basic operation

RDD Programming Guide

==> https://spark.apache.org/docs/latest/rdd-programming-guide.html

## Transformations

### `.map()` v.s. `.mapPartitions()` v.s. `.mapPartitionsWithIndex()`

Spark的map，mapPartitions，mapPartitionsWithIndex詳解

==> https://blog.csdn.net/QQ1131221088/article/details/104051087

---

#### 1. `.map()`

Return a new distributed dataset formed by passing each element of the source through a function func.

```python
rdd_2 = sc.parallelize(range(10), 4)

new_rdd_2 = rdd_2.map(lambda x: str(x))
print('> .map() =\n', new_rdd_2.glom().collect())
print()

#################################################
from pyspark import TaskContext

result = rdd_2.map(lambda x :x+TaskContext().partitionId())
print('> Original RDD, rdd_2.glom().collect() =\n', rdd_2.glom().collect())
print()
print('> .map() with TaskContext().partitionId() =\n', result.glom().collect())
```

Output:

```shell
> .map() =
 [['0', '1'], ['2', '3', '4'], ['5', '6'], ['7', '8', '9']]

> Original RDD, rdd_2.glom().collect() =
 [[0, 1], [2, 3, 4], [5, 6], [7, 8, 9]]

> .map() with TaskContext().partitionId() =
 [[0, 1], [3, 4, 5], [7, 8], [10, 11, 12]]
```

#### 2. `.mapPartitions()`

Similar to `.map()`, but runs separately on each partition (block) of the RDD, so func must be of type `Iterator<T> => Iterator<U>` when running on an RDD of type T.

```python 
rdd_2 = sc.parallelize([1,2,3,4,5,'a','b','c','d','e'], 4)

def func(itemsInPartition):
    # apply this `func` to each partition (=the whole partition) of the RDD
    yield str(itemsInPartition)
    
rdd_func = rdd_2.mapPartitions(func)
print('rdd_2.mapPartitions(func) =\n', rdd_func.glom().collect())
print()
    
    
def func_2(itemsInPartition):
    # you loop through each item in each partition of the RDD
    # = just apply this `func_2` to each item in each partition
    for item in itemsInPartition: 
        yield str(item)

rdd_func_2 = rdd_2.mapPartitions(func_2)
print('rdd_2.mapPartitions(func_2) =\n', rdd_func_2.glom().collect())
```

Output:

```shell
rdd_2.mapPartitions(func) =
 [['<itertools.chain object at 0x7ff8094580d0>'], ['<itertools.chain object at 0x7ff8094580d0>'], ['<itertools.chain object at 0x7ff8094580d0>'], ['<itertools.chain object at 0x7ff8094580d0>']]

rdd_2.mapPartitions(func_2) =
 [['1', '2'], ['3', '4'], ['5', 'a'], ['b', 'c', 'd', 'e']]
```

#### 3. `.mapPartitionsWithIndex()`

Similar to `mapPartitions`, but also provides func with an integer value representing the index of the partition, so func must be of type `(Int, Iterator<T>) => Iterator<U>` when running on an RDD of type T.

---

By using mapParitionsWithIndex you could output new elements which have their partition in it, then when you reduce you will know which partition you are handling the elements from. 

==> https://stackoverflow.com/questions/31281225/find-out-the-partition-no-id

```python
rdd_3 = sc.parallelize(range(10), 4)

# mapPartitionsWithIndex
def func(partitionIndex, itemsInPartition): 
    # apply this `func` to each partition (=the whole partition) of the RDD
    yield (partitionIndex, sum(itemsInPartition))
new_rdd_3 = rdd_3.mapPartitionsWithIndex(func)

# glom() flattens elements on the same partition
print('> rdd_3.glom().collect() =', rdd_3.glom().collect())
print('> new_rdd_3.glom().collect() =', new_rdd_3.glom().collect())


################################################################################

def func_2(partitionIndex, itemsInPartition):
    # you loop through each item in each partition of the RDD
    # = just apply this `func_2` to each item in each partition
    for item in itemsInPartition: 
        yield str(item+partitionIndex)
new_2_rdd_3 = rdd_3.mapPartitionsWithIndex(func_2)

# glom() flattens elements on the same partition
print()
print('>> new_2_rdd_3.glom().collect() =', new_2_rdd_3.glom().collect())
```

Output:

```shell
> rdd_3.glom().collect() = [[0, 1], [2, 3, 4], [5, 6], [7, 8, 9]]
> new_rdd_3.glom().collect() = [[(0, 1)], [(1, 9)], [(2, 11)], [(3, 24)]]

>> new_2_rdd_3.glom().collect() = [['0', '1'], ['3', '4', '5'], ['7', '8'], ['10', '11', '12']]
```

---

### `.map()` v.s. `.flatmap()`

```python
print(sc.version, '\n')

py_list = [str(x) for x in range(5)]
rdd = sc.parallelize(py_list)

# map
new_rdd = rdd.map(lambda item: item+'xx')
print('.map() =\n', new_rdd.collect())
print()

# flatmap
# same as .map(), but flatten the results before returns
# i.e. remove all `list of list`/`nested list`
new_rdd = rdd.flatMap(lambda item: item+'xx')
print('.flatmap() =\n', new_rdd.collect())
```

Output:

```shell
3.1.2

.map() =
['0xx', '1xx', '2xx', '3xx', '4xx']

.flatmap() =
['0', 'x', 'x', '1', 'x', 'x', '2', 'x', 'x', '3', 'x', 'x', '4', 'x', 'x']
```

### `.foreach()` v.s. `.map()`

==> See [Action](#actions) / [`.map()` v.s. `.foreach()`](#`.map()` v.s. `.foreach()`)


## Actions

**Below example shows there are 100 items/elements in this RDD, and this RDD is partitioned into 4 partitions (or items are grouped in 4 partitions).**

### `sc.parallelize()`

Store python list `[0,1,...,99]` as RDD in Spark. This dataset ***is not loaded in memory***. It is merely ***a pointer to the Python `py_list`***.

```python
# Stores python list `[0,1,...,99]` as RDD in Spark
## This dataset is not loaded in memory
## It is merely a pointer to the Python `py_list` 
py_list = range(100)
rdd = sc.parallelize(py_list)
print(rdd)
```

Output:

```shell
PythonRDD[11] at RDD at PythonRDD.scala:53
```

### `.count()`

Returns the number of items in this RDD

```python
#.count()
# Shows the number of items in this RDD
print('rdd.count()=', rdd.count())
```

Output:
```shell
100
```

### `.collect()`

Returns all the items in this RDD as python list

```python
#.collect()
# Returns all the items in this RDD as python list
print('rdd.collect()=', rdd.collect())
```

Output:
```shell
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99]
```

### `.getNumPartitions()`

Returns number of partitions in this RDD

```python
#.getNumPartitions()
# Gets number of partitions in this RDD
print('rdd.getNumPartitions()=', rdd.getNumPartitions())
```

Output:
```shell
4
```

### `.glom().collect()`

Returns the content of each partitions as `nested list` / `list of list`

```python
# Returns the content of each partitions as `nested list` / `list of list`
rdd.glom().collect()
```

Output:

```shell
[
 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24], 
 [25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49], 
 [50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74], 
 [75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99]
]
```

### `.foreach()`

Just executes inside function for each data element in the **RDD**, but return ***NOTHING***. 

#### `.map()` v.s. `.foreach()`

==> [reference](https://stackoverflow.com/questions/354909/is-there-a-difference-between-foreach-and-map)

* The important difference between them is that `map` accumulates all of the results into a collection, whereas `foreach` returns nothing. `map` is usually used when you want to transform a collection of elements with a function, whereas `foreach` simply executes an action for each element.

* In short, `foreach` is for applying an operation on each element of a collection of elements, whereas `map` is for transforming one collection into another.

* `foreach` works with a single collection of elements. This is the input collection.

* `map` works with two collections of elements: the input collection and the output collection.

# RDD - Closure

https://mycupoftea00.medium.com/understanding-closure-in-spark-af6f280eebf9

https://spark.apache.org/docs/latest/rdd-programming-guide.html#understanding-closures-

# spark-install-macos

Run the following in macOS terminal,

## How to start Jupyter Notebook with Spark + GraphFrames
1. Modify the PATH variables,
    ``` shell
    $ nano ~/.bashrc
    ```

2. Add the following lines in `~/.bashrc`, so `spark` and `jupyter notebook` can be launched at the same time.

    ```shell
    # Setting PATH for Spark 3.1.2
    export SPARK_HOME=/usr/local/Cellar/apache-spark/3.1.2/libexec
    export PATH="$SPARK_HOME/bin/:$PATH"
    export PYSPARK_DRIVER_PYTHON="jupyter"
    export PYSPARK_DRIVER_PYTHON_OPTS="notebook"
    ```

    In terminal,
    
    <img src="img\nano bashrc PATH variables.png" style="zoom:40%;" />

3. Update $PATH variable,

    ```shell
    $ source ~/.bashrc
    ```

4. Ensure `graphframes-0.8.1-spark3.0-s_2.12.jar` presents in the folder:

    <img src="img\folder-files.png"  height="350" />

    Start the `pyspark` with `graphframes` in terminal,
    
    ==> Needs 2 flags, `--packages` and `--jars`

    ==> Ensure the `graphframes` package name is same as `graphframes-0.8.1-spark3.0-s_2.12.jar` in folder.
    
    => Deal with error: `java.lang.ClassNotFoundException: org.graphframes.GraphFramePythonAPI`

    => https://blog.csdn.net/qq_42166929/article/details/105983616 
    
    ```shell
    $ pyspark --packages graphframes:graphframes:0.8.1-spark3.0-s_2.12 --jars graphframes-0.8.1-spark3.0-s_2.12.jar
    ```
    
    Terminal output:
    
    <img src="img\start-pyspark-terminal-output.png" height="500" />

    Jupyter Notebook:
    <img src="img\start-pyspark-jupyter-notebook.png" height="200" />



## Test Spark in Jupyter Notebook

Inside Jupyter Notebook:

==> Reference: https://medium.com/@roshinijohri/spark-with-jupyter-notebook-on-macos-2-0-0-and-higher-c61b971b5007

<img src="img\test-spark-jupyter-notebook.png" height="450" />

Cell 1:

```python
print(sc.version)
```

Output 1:

```shell
3.1.2
```

Cell 2:

```python
path_to_the_jar_file = 'graphframes-0.8.1-spark3.0-s_2.12.jar'
sc.addPyFile(path_to_the_jar_file)
```

Cell 3:

```python
from pyspark.sql import SQLContext 
sqlContext = SQLContext(sc) 
v = sqlContext.createDataFrame([("a", ),("b", ),], ["id", ]) 
e = sqlContext.createDataFrame([("a", "b"),], ["src", "dst"]) 

from graphframes import * 
g = GraphFrame(v, e) 
g.inDegrees.show()
```

Output 3:

```shell
+---+--------+
| id|inDegree|
+---+--------+
|  b|       1|
+---+--------+
```

Cell 4:

```python
import pyspark
from pyspark.sql.session import SparkSession
spark = SparkSession.builder.appName("spark test").getOrCreate()
columns = ['id', 'dogs', 'cats']
vals = [
     (1, 2, 0),
     (2, 0, 1)
]
# create DataFrame
df = spark.createDataFrame(vals, columns)
df.show()
```

Output 4:

```shell
+---+----+----+
| id|dogs|cats|
+---+----+----+
|  1|   2|   0|
|  2|   0|   1|
+---+----+----+
```

