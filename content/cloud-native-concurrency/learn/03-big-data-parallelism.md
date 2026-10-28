# Parallelism in Big Data & Stream Processing

When datasets scale into terabytes or petabytes, single-machine concurrency primitives (such as thread pools or executors) saturate CPU cores and memory limits. Analyzing massive volumes of data requires **parallel processing distributed across computer clusters**. Java provides the core foundation and native APIs for the industry's leading big data engines.

---

## Concurrency vs. Parallelism at Scale

Understanding the distinction is vital when scaling computing architectures:

- **Concurrency:** Handling multiple tasks by interleaving execution on overlapping timelines (e.g., event loops, non-blocking I/O).
- **Parallelism:** Executing multiple computational tasks literally simultaneously across separate physical hardware cores or distributed nodes.

$$\text{Speedup} \le \frac{1}{(1 - P) + \frac{P}{N}}$$

According to Amdahl's Law, the maximum speedup achievable through parallel processing depends strictly on the parallelizable fraction ($P$) of the workload across $N$ worker nodes.

---

## 1. The MapReduce Paradigm (Apache Hadoop)

Apache Hadoop revolutionized big data by pairing the Hadoop Distributed File System (HDFS) with the **MapReduce** parallel programming model. Developers author data transformations using native Java classes:

```java
// Word Count Mapper in Hadoop Java API
public class TokenizerMapper 
    extends Mapper<Object, Text, Text, IntWritable> {

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    @Override
    public void map(Object key, Text value, Context context) 
            throws IOException, InterruptedException {
        StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, one);
        }
    }
}
```

### The MapReduce Execution Pipeline

```text
[ Input Splits ] -> [ Map Phase ] -> [ Shuffle & Sort ] -> [ Reduce Phase ] -> [ Output ]
 (HDFS Blocks)      (Distributed)      (Network I/O)        (Distributed)       (HDFS)
```

1. **Input Splitting:** Data is partitioned into chunks and distributed across worker nodes.
2. **Map Phase:** Each worker runs map functions in parallel, generating intermediate key-value pairs.
3. **Shuffle & Sort:** The cluster routes all intermediate values sharing the same key to the same reducer node.
4. **Reduce Phase:** Reducer workers aggregate, summarize, and write outputs back to HDFS.

---

## 2. In-Memory Parallelism with Apache Spark

While MapReduce persists intermediate state to physical disk between stages, **Apache Spark** performs distributed computations entirely in-memory using **Resilient Distributed Datasets (RDDs)** and Directed Acyclic Graph (DAG) execution.

Although Spark's runtime is implemented in Scala, it exposes first-class Java APIs:

```java
// Apache Spark parallel processing in Java
SparkConf conf = new SparkConf()
    .setAppName("LogAnalyzer")
    .setMaster("spark://master:7077");

try (JavaSparkContext sc = new JavaSparkContext(conf)) {
    JavaRDD<String> lines = sc.textFile("hdfs://logs/access.log");

    long errorCount = lines
        .filter(line -> line.contains("ERROR 500"))
        .map(String::trim)
        .count();

    System.out.println("Total 500 errors: " + errorCount);
}
```

Spark analyzes lineage graphs to automatically recompute partitions if a node fails mid-stream, delivering fault tolerance without the disk I/O penalties of older batch frameworks.

---

## 3. Real-Time Stream Processing

Batch processing models analyze static, bounded data at rest. Modern cloud architectures frequently require continuous analysis of unbounded event streams as data arrives in real time.

```text
[ Sensor Stream ] -> [ Apache Kafka ] -> [ Apache Flink ] -> [ Real-Time Dashboard ]
                                          (Sub-second CEP)
```

### Java Stream API vs Distributed Streaming
- **Java Streams (`java.util.stream`):** Built into JDK 8+. Uses ForkJoinPool (`parallelStream()`) to parallelize operations across local CPU cores on a single JVM. Unsuitable for cluster-scale data.
- **Apache Flink:** True event-driven stream processing engine with native stateful stream operators, exactly-once processing semantics, and low-latency sliding windows.
- **Apache Storm:** Micro-topology engine providing sub-second record-by-record processing primitives (`Spouts` and `Bolts`).

```java
// Streaming word count in Apache Flink (Java API)
StreamExecutionEnvironment env = 
    StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.socketTextStream("localhost", 9999);

DataStream<Tuple2<String, Integer>> counts = text
    .flatMap(new Tokenizer())
    .keyBy(value -> value.f0)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .sum(1);

counts.print();
env.execute("Flink Streaming Window WordCount");
```

---

## Processing Frameworks Comparison

| Feature | Apache Hadoop MapReduce | Apache Spark | Apache Flink |
| :--- | :--- | :--- | :--- |
| **Model** | Batch (disk-bound) | Micro-batch / In-memory | Native real-time streaming |
| **Latency** | Minutes to hours | Seconds to minutes | Milliseconds (event-time) |
| **State Storage** | HDFS Disk | Memory (RDDs/DataFrames) | RocksDB / In-Memory state |
| **Java Support** | First-class native Java | Java RDD & Dataset APIs | Native Java DataStream API |
| **Best Used For** | Archival batch transformations | Interactive analytics & ML | Complex Event Processing (CEP) |
