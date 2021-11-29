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

