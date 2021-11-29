# Install Hadoop through VM image (Windows)

https://www.simplilearn.com/tutorials/big-data-tutorial/cloudera-quickstart-vm?source=sl_frs_nav_playlist_video_clicked

1.Install VM 

2.Download image https://www.cloudera.com/downloads/quickstart_vms/5-13/config.html

``
Dropdown Title: Select a Platform
Dropdown Type: downloadType
Option Title: Virtual Box
Option Value: vb
Download Location: https://downloads.cloudera.com/demo_vm/virtualbox/cloudera-quickstart-vm-5.13.0-0-virtualbox.zip
Download Type: asset
Option Title: VMWare
Option Value: vmw
Download Location: https://downloads.cloudera.com/demo_vm/vmware/cloudera-quickstart-vm-5.13.0-0-vmware.zip
Download Type: asset
Option Title: KVM
Option Value: kvm
Download Location: https://downloads.cloudera.com/demo_vm/kvm/cloudera-quickstart-vm-5.13.0-0-kvm.zip
Download Type: asset
Option Title: Docker Image
Option Value: docker
Download Location: https://downloads.cloudera.com/demo_vm/docker/cloudera-quickstart-vm-5.13.0-0-beta-docker.tar.gz
Download Type: asset
``

3.By default the VM uses a NAT virtual
  network adapter which means requests can originate in the VM and get
  forwarded by your host to their destination, but you can't get in from the
  outside. There are several other adapter types that will allow you to do
  what you want, but I prefer to use a 'bridged' adapter, which means the VM
  will request an IP address from the same router your host machine is using
  and will look like a peer of your host on the network.
  
  Once this is done, you'll need to reboot the VM and get the new IP address.
  If you run 'cat /etc/hosts' in the VM it should show you what IP address it
  associated the name 'quickstart.cloudera' with for itself. Make sure you
  can ping this IP address from your host: sometimes (especially with
  VirtualBox) you actually need to reboot twice because on the first boot
  after the networking change, the VM won't get an IP address early enough in
  the boot, so it has no choice but to associate 'quickstart.cloudera' with
  127.0.0.1. Once you can ping the IP address from your host, the only other
  thing you need to do is make sure Windows knows that 'quickstart.cloudera'
  corresponds to this IP address.
  
4.In your VM go to spark-2.0.2-bin-hadoop2.7/conf directory and create spark-env.sh file using below command.

cp spark-env.sh.template spark-env.sh

Open spark-env.sh file in vi editor and add below line.

SPARK_MASTER_HOST=192.168.1.132

Stop and start Spark using stop-all.sh and start-all.sh. Now in your program you can set the master like below.

``
val spark = SparkSession.builder()
  .appName("SparkSample")
  .master("spark://192.168.1.132:7077")
  .getOrCreate()
``

![Alt text](screens/Cloudera.jpg?raw=true "Optional Title")

# Guides

####1. Start Kafka, Mariadb, Redis with docker compose and setup Prerequisites

``
docker-compose -f spark-docker.yml up -d
``

Run SetupPrerequisites

####2. Stock aggregation scenario:
   
   ![Alt text](screens/Stock-aggr-scenario.jpg?raw=true "Optional Title")
   
####3. Stock aggregation requirements:
   
   ![Alt text](screens/Stock-aggr-requirements.jpg?raw=true "Optional Title")
   
####4. Batch processing pipeline

How does the Batch Data Engineering Use Case Design look like? 
Let's consider three warehouse locations, namely London, New York and Los Angeles. 
Each location has a MariaDB database called warehouse stock. There is also a central data center 
to which the data needs to be uploaded. In order to store the uploaded data in the 
central data center, we need a file system that is scalable and provides concurrent taxes. 
We want to create folders for each warehouse in this file system and provide permissions 
to the local warehouses to upload their respective files daily. We will choose a distributed file 
system like HDFS artistry that provides concurrent taxes, scaling and security. 
Files will also be created in parquet format that allows for parallel reads by Spark. 
Now, in each local data center, we will run a stock upload job built with Apache Spark. 
This job will run on a daily basis on a local Spark cluster. 
The job will read daily stock information from the warehouse stock database and upload them 
into the respective warehouse folder on the distributed file system. 
The file format will be parquet. 
As Stock Aggregator Job runs on a Spark cluster in the central data center, 
it will start after data uploads are expected to be complete every day. 
The Stock Aggregated Job will read all the parquet files across locations, 
consolidate across warehouses and store the results in a global stock database on MariaDB. 
The bad jobs will run every day and consolidate data for further reporting.

![Alt text](screens/Stock-aggr-design.jpg?raw=true "Optional Title")

##### 1) Generate sample data
Before we build the jobs for stock processing, we need to create some sample data in the source 
database warehouse stock. We will do so using the **_RawStockGenerator.java_** class. 
The generator generates data for three warehouses, namely New York, Los Angeles and London. 
In real life, these will be three separate database instances. But for this example, 
we will create all the data in the same database. 
The generateData method is called once for each of these warehouses. This method uses a simulated list 
of items and their values. It also uses a random number generator to generate stock counts. 
It generates stock information for three days, starting from the 1st of June. 
We first connect to the MariaDB instance of warehouse stock and create a prepared statement for 
these inserts. We then iterate for three days. For each item in the list of items, we pick random 
values for the opening stock, receives and issues. Then we insert them into the database.

##### 2) Uploading stock to a central store
 If you're using Windows, make sure to download the hadoop binaries into a local directory and set 
up the hadoop.home.dir system property to point to that directory. The upload stock method is 
used to upload stock for a given warehouse under time period. Multiple copies of the job are 
expected to run one each for each of the local data centers. We will however run all of them together 
in this example for demo purposes. In the job, we first create a SQL query to find the minimum 
and maximum values of the ID column for the given date range and warehouse ID. The ID is an 
auto-generated column populated in the source database when new records are inserted. We will 
use this column to partition database reads across multiple spark executors. The minimum and 
maximum values are captured and stored in the min bounds and max bounds variables. Next, we 
create a spark session. We use the local of two to run the embedded version, with two task notes. 
The default parallelism is also set to two. While running in a cluster this can be fine tuned based on 
scale requirements. Now we set up the query to get stock information for a given date range and 
warehouse ID. We execute this query and read the results into our datasets. While reading, we 
provide various database connection parameters like URL, DB table, user, and password. In order to 
use partitioning, we convert the query into a temporary table and provide that to a DB table option. 
In order to enable parallelism we need to provide the partition column and the upper and lower 
bounds. The numb partition is usually equal to the number of available executors. This will enable 
reading and uploading of data in parallel across multiple executors and enables scaling for the job. 
We are not going to use S3 or HDFS as the central file store for this example. Instead, we are going 
to use the local file store created under the raw data directory under the project folder. Please delete 
this raw folder if it already exists. Otherwise it may generate an error. For using S3 or HDFS, simply 
provide the corresponding URL in the path provider. The file will be created in parquet format. We 
will also partition this data by stock date and partition ID. This will help in querying data later if 
filters need to be applied based on these attributes. Finally, we print the total number of records in 
the stock data frame. Let's execute this code and review the results. We now see the raw data folder 
created. On exploring the folder, we see folders created based on stock date and by warehouse ID. 
Data here is stored in parquet format. We also see the total number of records printed along with its 
contents.

![Alt text](screens/Uploading-stock.jpg?raw=true "Optional Title")

##### 3) Aggregating stock across warehouses
 We read the source directory in Parquet format and create a data set. We then print the schema for 
the data set to make sure that the data has been correctly read. Next, we create a view on the data 
set. Creating a view enables us to directly run SQL statements against the table. It makes it easier to 
do computations and queries this way. We run a Count Star query and display the number of 
records in this table for checking purposes. Next, we run a Group By query for summarizing stock 
across warehouses. It groups data by stock date and item name. It adds up the opening stock, 
receipts, and issues. It also computes the closing stock and the value of that stock that is also a 
stored in the stock summary data set. Now we will write these results to the item stock table in the 
global stock database. Spark allows out-of-the-box capabilities to open data to a JDBC table. Do 
note that we are using MySQL in the URL instead of MariaDB, since there could be potential issues 
by using MariaDB in the URL. The functionality is the same whether the target database is MySQL 
or MariaDB. Let's run this code and review the results. The job has been successfully executed. It 
has printed the schema, the total number of records available and the stock summary that has been 
posted to the database.

![Alt text](screens/Stock-aggregator.jpg?raw=true "Optional Title")

 We can also review the counts of the global stock database that has been created using the 
GlobalStockDBBrowser.java class. Let's run the browser and review the counts in the table. We can 
see the number of records by day. You can also modify the query to further explore the database 
contents. Alternatively, you can use a MySQL client to connect to this database and explore the 
data. This completes our batch data engineering pipeline, where we acquired data, transported it to 
a central location, consolidated, and then persisted the results in a destination database.

![Alt text](screens/DB-browser.jpg?raw=true "Optional Title")


####5. Real-time processing pipeline

![Alt text](screens/Web-analit-scenario.jpg?raw=true "Optional Title")

 The business scenario for the real time example is website analytics. An enterprise runs an 
e-commerce website, selling multiple items like amazon.com. This website is used across multiple 
countries. The enterprise wants to track visits to the website by its users. It wants to know about 
the visit date, country, and duration for which the user stayed in the website. It also wants to track 
the last action before the user exited out of the website. This last action may be in the catalog page, 
the FAQ page, the shopping cart, or the order fulfillment page. To enable this real time analytics, 
we need to build a pipeline that will receive a stream of website visit records. For each user who 
visits this website, the stream will have one record. The record will contain the visit date, the last 
action performed by the user, the country from where the browser session was initiated and the 
duration of the visit. Using this information, we need to compute the following in real time. We need 
to compute five second summaries by last action performed and write it to a database. We need to 
keep track of the number of zips by users by each country. We also need to identify records where 
the last action is shopping cart. Then this needs to be published into an abandoned shopping cart 
queue for further downstream processing.

![Alt text](screens/Web-analit-requirements.jpg?raw=true "Optional Title")

 We have an Ecommerce application that is running in the cloud data center. The application creates 
user visit records when the user exits the application and publishes them to a Kafka queue called 
spark.streaming.website.visits. It's possible that the application is located in multiple data centers 
across the globe. Even in such cases, the data is streamed into a single central Kafka queue. An 
Apache spark job called Website Analytics runs and consumes the visit records in real time from the 
Kafka queue. On the data that is received, it will execute multiple actions. First it computes five 
second summaries and inserts them into a MariaDB database called website stats. A table called visit 
stats is used to capture that information. Next, it maintains a running counter of the total duration 
by country using a Redis sorted stat. Finally it filters those visits, which ended in the shopping cart 
and publishes them to the spark.streaming.cards.abandoned topic. This solution is natively scalable. 
Kafka provides scalable real-time streaming of data. Spark jobs can scale across multiple executor's 
based on the incoming data load. Redis is a distributor real-time database, that again can scale 
horizontally. Each executer can open individual connectors to MariaDB and insert summary records 
in Bedale. Let's now proceed to implement and execute the solution.

![Alt text](screens/Web-analit-design.jpg?raw=true "Optional Title")

##### 1) Generating a visits data stream

To execute the website visits by plane. We first need a streaming data source. Let's create an 
example data stream. The visits data generator is available in the 
website, visitsdatagenerator.javafile. We have already created the required Maria DB table and CAFCA 
topics for this chapter. When we ran the setup prerequisites script earlier. In this Java 
file, we simulate a data stream and publish it into a CAFCA topic. It implements the runnable 
interface and will be called from the main code to trigger simulation in real time. In the run 
muttered, we wait for five seconds to allow for the calling program to be set up. We then defined 
various properties for the CAFCA instance, running in the Docker container we started earlier in the 
course. We define a sample list of countries and the list of last actions. We also create a random 
number generator. We then proceed to create a hundred sample visit records. For each record, we 
pick a random country and last action from the predefined list. We also set the time to the current 
time. We set the duration to a random value. We form adjacent record for this visit with all the 
simulated attributes. Then, we publish the data to the CAFCA topic, spark.streaming.website.visits. 
We then print the data published to the console. We will now proceed to create the main website 
analytics spark job

##### 2) Building a website analytics job

Let's start building the streaming website analytics job now. the code for this is available in the 
streaming website, analytics store, Java class. In this job, we first set up the Hadoop home data tree 
property if it is a windows based system. Next, we need to set up a Rediswriter client. The client is 
available in the Rediswriter .java class. This is a simple client that connects to the writers running in 
the Docker container we set up earlier in the course. It manages a socket set called country stats. 
The setup method clears the key if it already exists. The open method opens a writers connection. 
The process method increments the counter for the exalted set with a specific country as the key and 
duration as the value. We also have a Maria DB manager that manages connections to a Maria DB 
database. The Maria DB writer is used to write records to the database through the Maria DB 
manager. Now back to the main job. We initiate the website visits data generator in a separate 
thread and start the thread. This will start publishing, visit data into the Kafka topic, which will 
immediately be consumed by the job. In the job, we define a schema for the visit record received 
through Kafka. Then, we initiate a spark session with local as the master. This runs an embedded 
spark instance. We then proceed to subscribe and receive visit data from the Kafka topic. To extract 
relevant attributes from the Kafka record, we use the schema defined earlier and cast the values to 
specific columns. We print the scheme I received as well as the data received from the spark data 
frame to the console. The first analysis we perform is to filter for records where the last action is 
shopping cart. We publish these records to a Kafka topic called spark.streaming.carts.abandoned. 
Here, we first format the record into a CSV and then publish to Kafka. Next, we updated this to 
maintain a running counter by country. The Rediswriter class is used for this purpose. Finally, we 
will compute five second summaries. For this, we first create a watermark with an even time using 
the timestamp attribute that is part of the visit record. Then, we group by timestamp and last action 
and find the sum of the duration. To write this data to Maria DB, we will use the Maria DB writer 
instance. There will be one instance created for each of the executors. These three tasks will run 
continuously in parallel. As new records are received from Kafka. Finally, we will start a 
CountDownLatch to keep the program running.

The real time pipeline writes to three different destinations, namely, Kafka, MariaDB and Redis. We 
need to monitor these destinations to make sure that they are updated while the pipeline is running. 
For this purpose, we have some utility browser classes available. Let's start with them. We start with 
the ShoppingCartTopicBrowser.java. This class listens on abandoned carts topic and prints the 
records as they are received.
 
 ![Alt text](screens/Country-stats-browser.jpg?raw=true "Optional Title")
 
 Let's execute this topic browser. It will wait until new messages are 
received. 

![Alt text](screens/Visit-stats-DBbrowser.jpg?raw=true "Optional Title")

Next, we go to the CountryStatsBrowser.java. This reads the sorted set from Redis and 
prints the duration every five seconds. Let's start this browser too. We then move on to the 
VisitStatsDBBrowser.java. 

![Alt text](screens/Streaming-analyt.jpg?raw=true "Optional Title")

This class reads the visit stats table every five seconds and prints the total 
count and duration. Let's start this browser too. Finally, let's go back to the main program, 
StreamingWebsiteAnalytics.Java. Let's run this code. We see the Kafka data generator generating 
sample records into Kafka topics. The records are being received and processed. Now we will look at 
the country stats browser. 

![Alt text](screens/Country-stats-browser5.jpg?raw=true "Optional Title")


We see the counts being updated every five seconds. We move on to the 
visit stats browser. We see that the number of visits stats records are increasing also every five 
seconds. Note that there is a watermark set, so it will take some time before we start seeing records 
here. 

![Alt text](screens/Visit-stats-DBbrowser5.jpg?raw=true "Optional Title")

Finally, we look at the shopping cart topic browser. We see that the records for shopping carts 
are now available in this topic. One key error to watch out is for checkpoint location rights. If there 
are failures right into checkpoint locations, then the pipeline may not work. Try changing the 
checkpoint location to new values if you face this error. This completes our example for Real time 
Data Engineering with Apache Spark.

![Alt text](screens/Streaming-analyt5.jpg?raw=true "Optional Title")

####6. Spark best practices

##### 1) Batch vs. real-time options

Data engineers and architects have a tendency to build all pipelines as real-time pipelines whenever 
possible. The key justification is that it is super fast, would generate the required insights instantly, 
and enable business actions. It is also considered cool in the data engineering world, but before 
jumping into building real-time pipelines, we need to understand the complexities involved. 
Real-time pipelines deal with unbounded datasets. This makes it difficult to size compute resources 
like memory and clusters. When doing time-based aggregations, we need to deal with windowing 
and watermarks. Irrespective of how delayed the watermarks are, we do end up with missed events 
and incorrect aggregations. Real-time state management is another challenge to ensure that state is 
properly maintained across the Spark cluster. Reprocessing of events is complex when issues 
happen. It is not a simple redo of the job, but a number of items need to be reset like checkpoints 
and state. So, when building a pipeline, it's important to analyze the requirements thoroughly before 
deciding if a real-time pipeline is appropriate for the use case. Go for real-time pipelines when 
responses, analytics and actions are needed with latency of few seconds. Do make sure that this is a 
mandatory requirement, not a nice one to have. If this pipeline needs to process a continuous 
stream of real-time events, then real-time pipelines are a good choice. The processing should also 
involve minimal and mandatory aggregation and analytics. Adding too many aggregation steps will 
introduce complexity with watermarks and state management. Make sure that the compute resources 
are sufficiently available for real time pipelines. Choose real-time pipelines only when the use case 
demands it. You can also build hybrid pipelines where part of the pipeline is real-time and the rest is 
batch.

![Alt text](screens/Real-time.jpg?raw=true "Optional Title")

##### 2) Scaling extraction and loading operations

When scaling a data engineering pipeline, all stages in the pipeline need to scale in order for the 
entire pipeline to scale. Extracting data and loading process data into destinations are time 
consuming as they usually deal with this discreets and rights. How do we scale these steps when 
building pipelines with Apache Spark? Let's start with data extraction. 

![Alt text](screens/Scale-extr.jpg?raw=true "Optional Title")

Sparks support parallel 
extraction of data from various data sources. For example, Spark can read JDBC records, in parallel, 
across its executors. Similarly, it can divide up Kafka partitions between executors and process them 
in parallel. With the data source being used, analyze the out of the box options provided by Spark for 
that data source. Explore these options for parallel reads of data while maintaining data consistency. 
If possible, choose a source technology and build the source schema in such a way to suit parallel 
operations. This option is only available if the source systems are also being designed and built at 
the same time as the data engineering pipelines. Spark support predicate push downs. This pushes 
don't filtering operations to the data sources so unwanted records are not read into memory. 
Leverage them when possible. When doing parallel extractions, build defenses against missed 
records and duplicate records to ensure accurate results. What about loading data after processing 
into data sinks? Each Spark executor can write independently and in parallel to a data sink for most 
out of the box data sinks. Analyze how Spark works with a specific data sink technology like RDBMS 
or a Kafka queue. Understand the impact of record additions and updates on transactions, locking, 
and serialization. You may have to limit the number of parallel connections to the sink based on 
supported capacity and parallelism. Repartitioning RDDs is an effective way to control parallelism. 
Finally, build defenses against missed updates and duplicate updates.

![Alt text](screens/Scale-load.jpg?raw=true "Optional Title")

##### 3) Scaling processing operations

 Apache Spark is built for massive distributed operations, but care should be taken during pipeline 
design to our choke points and shuffles between executors. Filter any unwanted data as early as 
possible in the pipeline. This reduces the amount of data kept in memory and move around 
executors. Minimize reduce operations and associated shuffling early in the pipeline. If needed, 
move down to the end of the pipeline as much as possible. Repartition data as needed to reduce the 
number of partitions. Certain operations have the tendency to create too many RDDs, so keep track 
of this. Cache results in the pipeline when appropriate to reduce reprocessing of data. Avoid actions 
that send back data to the driver program until the end. This minimizes shuffles between executors.

![Alt text](screens/Scale-proc.jpg?raw=true "Optional Title")

##### 4) Building resiliency

When we build scalable data pipelines, we also need to build resilience into them to make sure that 
the pipelines can support critical operations and deliver reliable outcomes. Apache Spark provides 
various resilience capabilities out of the box against failures. It handles failures at a task level, stage 
level and executor level. That is an important reason to use Apache Spark. However, some additional 
considerations are required to make the solution resilient. Monitor Spark jobs and Spark clusters for 
unhandled failures. Create alerting systems by which required personnel are informed and problems 
can be fixed quickly. Use checkpoints for streaming jobs with persistent data stores. Even if Spark 
restarts, it can resume from where it left off. Build resilience in associated input stores and output 
sinks. After all, it's not enough for Spark jobs to be resilient, but the entire solution should be. 
Choose data stores that provide redundancy and persistence of data. The compute infrastructure 
needs resilience, too. Deploy Spark on cluster setups, so failure of one node does not bring down 
the entire pipeline.

![Alt text](screens/Build-resil.jpg?raw=true "Optional Title")

####7. Hybrid pipeline

##### 1) Project requirements

In streaming use case, we created a MariaDB table with five second summaries of the 
last action performed by website users. We are going to use this table as the input, and then build a 
downstream pipeline to analyze significant actions and maintain a running count table of the 
significant actions. Please refer to the SetupPrerequisites.java class in the setup package of the Java 
project if you need to understand the schema of the visit stats stable. Let's review the detail 
requirements for the exercise. The task is to run a job every day and extract last action records with 
a total duration of more than 15 seconds, within a five second window. make sure that 
 there is already data available in count table. Next, stream 
the significant last actions to a real-time Kafka topic. A topic called spark.exercise.lastaction.long has 
already been created for you during the setup process for the course, then analyze the significant 
actions in the Kafka topic in real time, and maintain a running counter of the significant last actions 
by action name. This use case combines a batch processing task and a real-time processing task and 
is an example of a hybrid pipeline.

![Alt text](screens/Requrements.jpg?raw=true "Optional Title")

##### 2) Solution design

What would the hybrid pipeline for the course project look like? We start off with the MariaDB table 
called visit_stats in the website_stats database. We first build a Long Last Action Extraction job. 
Actions with long durations are considered significant actions. This job will filter records from the 
database and extract those with the duration of greater than 15 seconds. Each of these records will 
be published to the Kafka topic, spark.exercise.lastaction.long. We then build a second job Scorecard 
For Last Action job. This job will listen to the Kafka topic in real time. The job will maintain a Redis 
sorted set to track last actions and their counts. Each time a last action is received from the topic, the 
sorted set will be incremented for that specific action. Let's proceed to the next video to review the 
code for this exercise.

![Alt text](screens/Design.jpg?raw=true "Optional Title")


##### 3) Extracting long last actions

Let us examine the example code for the first part of the exercise, extracting long last actions. The 
code for this example is available in the calm.learnin.sparkdataengg chapter six package, The file is 
LongLastActionExtractorJob.java. Please note that the original records were generated using system 
date, when the website visits data generator was run. So use the right date ranges. First, we need to 
extract the records from the MariaDB table. In order to take advantage of parallelism, we first find 
the min and max bounds for the records using the bounds query. Then we define as spark session. 
We execute the query to filter for long actions and store them in the last action data set. We take 
advantage of parallelism by setting the relevant options for bounds and partitions. Next we extracted 
last action column from the dataset and publish it to the Kafka topic: "spark.exercise.lastaction.long" 
Finally, we print the total number of records that were fetched and published to the job. Let's run 
this code and review the results. We see the last action count published as 26. Remember that the 
source data was generated randomly. So this count may be different when you are running it on 
your system. 

![Alt text](screens/Extractor-job.jpg?raw=true "Optional Title")

##### 4) Building a scorecard

Let's build a running ScorecardForLongLastActions published to Kafka in the previous chapter. The 
code for this is available in the ScoreCardForLastActionJob.java. To help 
with managing the latest sorted set, we have the RedisWriter class. This RedisWriter maintains a 
sorted list called Last-action-stats on the latest instance, running on Docker. It has functions to 
maintain the sorted set. Here it increments by one, every time a last action occurs in the datastream. 
Back to the main job ScoreCardForLastAction, we first define a SparkSession for the job. Since we 
only have one value in the topic lastAction, we don't need to define a schema. We read the topic 
directly into the raw lastAction dataset. From there we extract the value as lastAction into the 
lastAction dataset. We print the schema, and we also print the contents to the console. To maintain 
the latest sorted set, we use the .writeStream method and call the RedisWriter for each record. This 
will update and maintain the latest sorted set. We will use the CountDownLatch to keep the job 
running. In order to review the values in the sorted set, we also have a LastActionStatsBrowser. Let's 
first start the browser.

![Alt text](screens/Run-stat-browser.jpg?raw=true "Optional Title")

 Now let's run this ScoreCardForLastActionJob.
 
![Alt text](screens/Run-score-card.jpg?raw=true "Optional Title")

 We also need to run the 
LongLastActionExtractorJob in order to generate new records, as the ScoreCardForLastActionJob only 
listens for new records. 

![Alt text](screens/Run-action-extractor.jpg?raw=true "Optional Title")

Now we see the records being received and processed by the ScoreCardJob. 

![Alt text](screens/Records-received.jpg?raw=true "Optional Title")

We also see the updated values in the LastActionStatsBrowser. 

![Alt text](screens/Updated-values.jpg?raw=true "Optional Title")
