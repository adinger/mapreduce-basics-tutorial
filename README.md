# MapReduce as a Foundation for Big Data Frameworks

This post illustrates the basic operations in a MapReduce job. Modern big data frameworks like Spark, Flink, Pig, Hive, and even Java 8 and Python all provide interfaces for these operations, so it is highly beneficial have an understanding of them in order to work with these tools. The operations that comprise the MapReduce paradigm are map, reduce, partition, group, sort, and combine. I will explain each of them using an example of counting different types of fruit in a basket as an example.

### Map
This does exactly what the name implies. It outputs a map. Each map() call takes in one input, and outputs zero or one outputs. If we want to count how many of each type of fruit are in a basket, map would output a tuple of (fruit_type, count)

```
map((grape, green, 8))   -> (grape, 8)
map((grape, green, 6))   -> (grape, 6)
map((apple, red, 4))     -> (apple, 4)
map((banana, yellow, 3)) -> (banana, 3)
map((apple, red, 2))     -> (apple, 2)
```

### Sort
Sorting happens in what is called the "shuffling" phase of a MapReduce job, defined as the step where data moves between nodes. This does a global sort of all keys. The default sorting logic can be overwritten. You can use a sorter by implementing a compare() method, which needs to take two keys as inputs, and outputs one of three things: 1 (key 1 > key 2), -1 (key 1 < key 2), or 0 (key 1 = key 2).  Say we want the output to be sorted by ascending alphabetical order of the fruit's name, then the output of the sorter would be the following.

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

### Partition
After sorting, the framework needs to decide where to send each item. This is called partitioning, and it also happens in the "shuffling" phase. The partitioner tells the job which reducer, and effectively which node in a Hadoop cluster, to send each item to. In a MapReduce application, each reducer is identified by computing a hash of the item. This behavior can be overwritten to use custom hashing logic. Just remember, each partition is associated with a reducer.

```
partition((apple, 4))   -> reducer_1
partition((apple, 2)    -> reducer_1
partition((banana, 3))  -> reducer_2
partition((grape, 8))   -> reducer_2
partition((grape, 6))   -> reducer_2
```

### Group
Grouping is the last operation in the shuffling phase. Note how a partitioner only guarantees that a reducer will receive all the keys for a fruit, but multiple types of fruit can be sent to the same reducer; `reducer_2` receives both the banana's and grape's keys. We can use a group operator to ensure that the keys for banana are processed in a different group than the keys for grape. Within a reducer, multiple reduce() calls are made (in a for loop), and each reduce() call processes one group. To tell each item which reduce() call it should go to, we define what is called a GroupComparator in Java MapReduce. It takes two items, and outputs true if they are equal, and false if they are not. You can customize the equality logic. If the two items are equal, they are sent to the same reduce() call. If they are not equal, they will be sent to different reduce() calls. In our example, we compare equality of the fruit type.

```
group((apple, 4))    -> reducer_1, group_a
group((apple, 2))    -> reducer_1, group_a
group((banana, 3))   -> reducer_2, group_b
group((grape, 8))    -> reducer_2, group_g
group((grape, 6))    -> reducer_2, group_g
```

### Reduce
A Reducer can make multiple reduce() calls (in a for loop, essentially), and reduces them into a lesser number of outputs. N inputs, N or less outputs. Remember, a Reducer can receive multiple groups of items, and one reduce() call within a Reducer receives only one group. The actual reducing computation is implemented within a reduce() call. For our fruit counting example, we want each reduce() call to sum up the counts of the fruit it's assigned.

```
reducer_1:
reduce((apple, 4), (apple, 2))   -> (apple, 6)

reducer_2:
reduce((banana, 3))              -> (banana, 3)
reduce((grape, 8), (grape, 6))   -> (grape, 14)
```

Each reducer outputs one file, so we would have two files, with the following contents.

File 1:
```
(apple, 6)
```

File 2:
```
(banana, 3)
(grape, 14)
```
