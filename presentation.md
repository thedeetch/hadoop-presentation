class: center, middle

# Hadoop
## my attempt to kill the "big data" buzzword

---

class: middle

# Who is this guy?

* (Big) Data Developer
* MS SQL Server DBA
* Newly minted MS in CS (go jackets!)
* BS in CS (go chargers!)
* Interests 
  * machine learning
  * computer vision
  * Chicago Cubs
* Spaces > Tabs

---

# What is Hadoop?

![Hadoop logo](http://hadoop.apache.org/images/hadoop-logo.jpg)

A cluster computing framework based on the [Google MapReduce][mapreduce] paper.

**TL;DR:**  Most efficient way of processing large amounts of data is to push compute to the data, using `map()` and `reduce()` on commodity hardware.

---

# Hadoop out of the box

### Hadoop File System (HDFS)
* Immutable distributed file system
* Very similar to S3

### MapReduce
* Pushes `map()` and `reduce()` functions to data
* Other projects provide higher level abstractions

### Yet Another Resource Negotiator (YARN)
* Similar in function to an OS scheduler (preemption, resourcing)

---

# An ecosystem of projects

| Project | Description | Equivalent | 
| ---- | ----------------- | -------------------|
| HBase | No-SQL store | Amazon DynamoDB |
| Hive/Impala/Drill | SQL engine | Amazon Redshift |
| Kafka | Pub/Sub messaging | Amazon Kinesis |
| Solr | Search | Elasticsearch |
| Spark/Storm/Flink | Compute engine | Google Dataflow |
| Flume/Sqoop | Ingestion | |
| Zookeeper | Coordinator | Consul, Etcd |
| Pig | ETL language | |
| Avro/Parquet | Serialization | Protobuf |
| Oozie | Workflows | cron, Luigi |
| Mahout | Machine learning | scikit-learn, Weka |

[Hadoop Ecosystem Table][ecosystem]
---

# HDFS

1. Break file into blocks (typically 64-128 MB)
2. Distribute blocks across nodes (data locality)
3. NameNode keeps metadata ([the small files problem][smallfiles])

![HDFS Architecture][hdfs]

---

# MapReduce

Slightly more than just `map()` and `reduce()`.

![MapReduce WordCount][wordcount]

---

# Mapper

```java
public void map(Object key, Text value, Context context
                ) throws IOException, InterruptedException {
  String line = (caseSensitive) ?
      value.toString() : value.toString().toLowerCase();
  for (String pattern : patternsToSkip) {
    line = line.replaceAll(pattern, "");
  }
  StringTokenizer itr = new StringTokenizer(line);
  while (itr.hasMoreTokens()) {
    word.set(itr.nextToken());
    context.write(word, one);
    Counter counter = context.getCounter(CountersEnum.class.getName(),
        CountersEnum.INPUT_WORDS.toString());
    counter.increment(1);
  }
}
```

---

# Reducer

```java
public void reduce(Text key, Iterable<IntWritable> values,
                   Context context
                   ) throws IOException, InterruptedException {
  int sum = 0;
  for (IntWritable val : values) {
    sum += val.get();
  }
  result.set(sum);
  context.write(key, result);
}
```

Plus another [100 lines of boilerplate code][wordcountcode].

---

# What's the big deal with Spark?

* Faster (100x) than MapReduce (disk vs memory)
* Extension of Scala collections API
* First-class support for Scala, Java, Python, and R
* Includes machine learning, SQL, and graph processing libraries
* Streaming and batch modes have nearly identical syntax
* Doesn't depend on Hadoop (although it is nice)

---

# Let's use Spark instead

```scala
val textFile = sc.textFile("hdfs://...")
val counts = textFile.flatMap(line => line.split(" "))
                 .map(word => (word, 1))
                 .reduceByKey(_ + _)
counts.saveAsTextFile("hdfs://...")
```

---

# Counting words is boring

* [Classify languages of Tweets][tweets]
* [Detect fraud][fraud]
* [Real time log analytics][logs]
* [Build a data lake][lake]
* [Save lives!][nickscore]

---

# The problem

## Sepsis is really really bad

* 64% mortality rate
* Costs $16.7 billion per year
* Single most expensive condition in U.S. hospitals

Early detection decreases mortality by 15%, **but relies on humans**.

---

# What could be done?

* Eliminate the human (not the ICU patient)
* Real time identification
* Improve existing sepsis prediction models

---

class: middle, center

### Computers + Big Data + Machine Learning + a bunch of Buzzwords = 

# NickScore

---

class: middle, center

## NickScore can identify **82%** of sepsis patients in an ICU **28 hours** before onset in **real time**

---

# It uses current vital signs

from the [MIMIC III critical care database][mimic] with over 10 years of data from 46,000 patients.

|     |     |
| --- | --- |
| Arterial PH | Platelet Count |
| Aespiratory Rate | White Blood Cell Count |
| Aystolic Blood Pressure | Fraction of inspired oxygen |
| Areatinine | Heart Rate |
| Ailirubin | Blood Urea Nitrogen |
| ANR | Arterial PaO2 |
| Arine Output | Glasgow Coma Score |

---

<div style="text-align:center;">
  <img src="http://d3ukwgt0ah4zb1.cloudfront.net/content/scitransmed/7/299/299ra122/F1.large.jpg" alt="NickScore Feature Calculation" style="
    max-height: 600px;" align="middle">
</div>

---

# How was it done?

Data flow diagram

```
# feature extraction          # model fitting        # profit!

   raw data          |       |   train model   |
      |              |       |     |     ^     |
      v              |       |     v     |     |
extract features     |  -->  |  evaluate model | --> final model
      |              |       |                 |
      v              |       |                 |
cache in memory      |       |                 |
```

---

# How was it done?

5 node Hadoop cluster on Google Cloud Dataproc

```bash
gcloud dataproc clusters create cluster-1 \
  --zone us-central1-f --master-machine-type n1-standard-4 \
  --master-boot-disk-size 500 --num-workers 4 \
  --worker-machine-type n1-standard-4 \
  --worker-boot-disk-size 500 --num-worker-local-ssds 1 \
  --project cse8803-project
```

---

# How was it done?

Spark MLlib and Dataframe API

```scala
// build features
val chartEvents = sqlContext.load.csv("chartevents.csv")
// ...
val features = buildFeatures(chartEvents, ...)

// fit model using grid search and cross validation
val lr = new LogisticRegression().setMaxIter(10)
val pipeline = new Pipeline().setStages(Array(lr))
val crossval = new CrossValidator()
  .setEstimator(pipeline)
  .setEvaluator(new BinaryClassificationEvaluator)

val paramGrid = new ParamGridBuilder()
  .addGrid(lr.regParam, Array(0.1, 0.01))
  .build()
crossval.setEstimatorParamMaps(paramGrid)
crossval.setNumFolds(3)

val cvModel = crossval.fit(features)
val bestModel = cvModel.bestModel.asInstanceOf[PipelineModel].stages.last.asInstanceOf[LogisticRegressionModel]
```

---

class: middle, center

# Questions?

---

# Demo!

```scala
val chartEvents = sqlContext.read.parquet("gs://cse8803-project/CHARTEVENTS")
chartEvents.show

// show off caching
chartEvents.persist()
chartEvents.count()
chartEvents.count()

// write some sql
chartEvents.registerTempTable("chart_events")
val chartEventSelected = sqlContext.sql("SELECT subject_id, charttime, itemid, value, valueuom FROM chart_events")
chartEventSelected.show

// use scala collections api
import org.apache.spark.sql.Row
val maxPressure = chartEventSelected.rdd
  .filter{
    case Row(subjectId, chartTime, itemId, value, valueuom) =>
      subjectId == 27577 && itemId == 220060
  }
  .map(_.getString(3).toInt)
  .reduce(_ max _)
```

---

class: middle, center

# Thank you!

[mapreduce]: http://research.google.com/archive/mapreduce.html
[ecosystem]: https://hadoopecosystemtable.github.io/
[hdfs]: https://cvw.cac.cornell.edu/mapreduce/images/hdfs.png
[smallfiles]: http://blog.cloudera.com/blog/2009/02/the-small-files-problem/
[wordcount]: https://www.safaribooksonline.com/library/view/programming-hive/9781449326944/httpatomoreillycomsourceoreillyimages1321235.png
[wordcountcode]: https://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v2.0
[tweets]: https://databricks.gitbooks.io/databricks-spark-reference-applications/content/twitter_classifier/index.html
[fraud]: http://blog.cloudera.com/blog/2015/07/designing-fraud-detection-architecture-that-works-like-your-brain-does/
[logs]: http://blog.cloudera.com/blog/2015/02/how-to-do-real-time-log-analytics-with-apache-kafka-cloudera-search-and-hue/
[lake]: https://en.wikipedia.org/wiki/Data_lake
[nickscore]: https://docs.google.com/presentation/d/1c3YOc5GSmgen2B4CuG0rw_jn67dRZUjpVFIR5Mk5_AQ/edit?usp=sharing
[mimic]: https://mimic.physionet.org/
[features]: http://d3ukwgt0ah4zb1.cloudfront.net/content/scitransmed/7/299/299ra122/F1.large.jpg
