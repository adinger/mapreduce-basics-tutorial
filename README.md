# MapReduce Overview

This article illustrates the basic operations in the MapReduce programming paradigm. Modern big data frameworks like Spark, Flink, Pig, and Hive, and most functional programming languages, all provide interfaces for these common operations in some form or another, so understanding them will help you learn these technologies more easily. The operations that comprise the MapReduce paradigm are map, reduce, partition, group, sort, and combine. 

MapReduce is derived from functional programming concepts. Each of these operations are higher order functions, which in layman's terms means their behavior can be customized by passing them a function you define. I will explain all of these operations using an example of counting different types of fruit in a basket.

### 1. Map
This does exactly what the name implies. It outputs a map, or a dictionary. Each map() call takes in one item, and outputs zero or one items. If we want to count how many of each type of fruit are in a basket, map would output a tuple of (fruit_type, count)

```
map((grape, green, 8))   -> (grape, 8)
map((grape, green, 6))   -> (grape, 6)
map((apple, red, 4))     -> (apple, 4)
map((banana, yellow, 3)) -> (banana, 3)
map((apple, red, 2))     -> (apple, 2)
```

### 2. Sort
Sorting happens in what is called the "shuffling" phase of a MapReduce job, defined as the step where data moves between nodes. This does a global sort of all keys. The default sorting logic can be overwritten. You can use a sorter by implementing a compare() method, which needs to take two keys as inputs, and outputs one of three things: `1` (key 1 > key 2), `-1` (key 1 < key 2), or `0` (key 1 = key 2).  Say we want the output to be sorted by ascending alphabetical order of the fruit's name, then the output of the sorter would be the following.

```
sort((grape, 8),
     (grape, 6),
     (apple, 4),
     (banana, 3),
     (apple, 2))

     ->

     (apple, 4),
     (apple, 2),
     (banana, 3),
     (grape, 8),
     (grape, 6)

```

### 3. Partition
After sorting, the framework needs to decide where to send each item. This is called partitioning, and it also happens in the "shuffling" phase. The partitioner tells the job which reducer, and effectively which node in a cluster, to send each item to. In a MapReduce application, each reducer is identified by computing a hash of the item. This behavior can be overwritten to use custom hashing logic. Just remember, each partition is associated with a reducer.

```
partition((apple, 4))   -> reducer_1
partition((apple, 2)    -> reducer_1
partition((banana, 3))  -> reducer_2
partition((grape, 8))   -> reducer_2
partition((grape, 6))   -> reducer_2
```

### 4. Group
Grouping is the last operation in the shuffling phase. Note how a partitioner only guarantees that a reducer will receive all the keys for a fruit, but multiple types of fruit can be sent to the same reducer; `reducer_2` receives both the banana's and grape's keys. We can use a group operator to ensure that the keys for banana are processed in a different group than the keys for grape. Within a reducer, multiple reduce() calls are made (in a for loop), and each reduce() call processes one group. To tell each item which reduce() call it should go to, we define what is called a GroupComparator in Java MapReduce. It takes two items, and outputs true if they are equal, and false if they are not. You can customize the equality logic. If the two items are equal, they are sent to the same reduce() call. If they are not equal, they will be sent to different reduce() calls. In our example, we compare equality of the fruit type.

```
group((apple, 4))    -> reducer_1, group_a
group((apple, 2))    -> reducer_1, group_a
group((banana, 3))   -> reducer_2, group_b
group((grape, 8))    -> reducer_2, group_g
group((grape, 6))    -> reducer_2, group_g
```

### 5. Reduce
A Reducer can make multiple reduce() calls (in a for loop, essentially), and reduces them into a lesser number of outputs. N inputs, N or less outputs. Remember, a Reducer can receive multiple groups of items, and one reduce() call inside a Reducer receives only one group. In a Java MapReduce application, each Reducer runs in its own JVM on the node to which it's assigned. The reducing logic is implemented within a reduce() call. For our fruit counting example, we want each reduce() call to sum up the counts of the fruit it's assigned.

```
reducer_1:
reduce((apple, 4), (apple, 2))   -> (apple, 6)

reducer_2:
reduce((banana, 3))              -> (banana, 3)
reduce((grape, 8), (grape, 6))   -> (grape, 14)
```

Each reducer outputs one file, so the output of the whole MapReduce application will be two files:

reducer_1.txt:
```
(apple, 6)
```

reducer_2.txt:
```
(banana, 3)
(grape, 14)
```

### 6. Bonus: Combine
A Combiner is a special kind of reducer that operates on the map side. The purpose is to reduce I/O across the network because it locally reduces the number of items on the node running the mapper before sending them to the reducers. Unlike a reducer, the input key schema and output key schema must be the same. 

For example, the reducer can output just the counts:

```
reducer_1:
reduce((apple, 4), (apple, 2))   -> (6)

reducer_2:
reduce((banana, 3))              -> (3)
reduce((grape, 8), (grape, 6))   -> (14)
```

However, the combiner's input and output must both have the format (fruit, count):

```
reducer_1:
reduce((apple, 4), (apple, 2))   -> (apple, 6)

reducer_2:
reduce((banana, 3))              -> (banana, 3)
reduce((grape, 8), (grape, 6))   -> (grape, 14)
```

The reason for the input and output formats being the same is multiple combine operations will run in a chain, so the output of one combiner can be the input of the subsequent combiner.

### Stay Tuned
The next post will demonstrate how to implement the fruit-counting example in Java MapReduce code.
