# How to reproduce

## Start Elastic

[Elastic Dockerfile](http://github.com/docker-library/elasticsearch/blob/master/2.4/Dockerfile)

```
docker run -d elasticsearch
```

## Start Hive/Hadoop

### Hadoop

[Instructions here](https://hadoop.apache.org/docs/current2/hadoop-project-dist/hadoop-common/SingleCluster.html)
[Download Hadoop here](http://apache.xl-mirror.nl/hadoop/common/stable/hadoop-2.7.3.tar.gz)

#### Passwordless ssh
```
1. ssh-keygen -t rsa
2. cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
3. chmod og-wx ~/.ssh/authorized_keys
```
### Hive

[Instructions here](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Installation#AdminManualInstallation-InstallingfromaTarball)
[Download Hive here](http://ftp.tudelft.nl/apache/hive/stable/apache-hive-1.2.1-bin.tar.gz)

## Elastic Hive

### Elastic Hadoop
[Instructions here](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/index.html)
[Download Elastic Hadoop](http://download.elastic.co/hadoop/elasticsearch-hadoop-2.4.0.zip)
[Elastic Hadoop Configurations](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html)

#### Set jar configuration
```
hive> set hive.aux.jars.path;
hive.aux.jars.path=/opt/elasticsearch-hadoop-2.4.0/dist/elasticsearch-hadoop-2.4.0.jar
```

Appears this didn't work. So...

```
cp /opt/elasticsearch-hadoop-2.4.0/dist/elasticsearch-hadoop-2.4.0.jar /opt/hive/lib/
```

#### Create a table

Elastic external table
```
USE elastic;
DROP TABLE IF EXISTS artists;
CREATE EXTERNAL TABLE elastic.artists (
    id      BIGINT,
    name    STRING,
    links   STRUCT<url:STRING, picture:STRING>)
STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES('es.resource' = 'radio/artists', 'es.nodes' = '172.17.0.2');
```

Sample data table
```
CREATE EXTERNAL TABLE source (
    name      STRING,
    url       STRING,
    picture   STRING
)
ROW FORMAT DELIMITED   
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION 'hdfs://localhost:9000/user/rtabora/hive/';
```

```
hdfs dfs -put sample_data.csv /user/rtabora/hive/
```

#### Load the table
```
INSERT OVERWRITE TABLE artists
SELECT NULL, s.name, named_struct('url', s.url, 'picture', s.picture)
FROM source s;
```

```
hive> INSERT OVERWRITE TABLE artists
    > SELECT NULL, s.name, named_struct('url', s.url, 'picture', s.picture)
    > FROM source s;
Query ID = rtabora_20161017234501_7eeb8070-e0d9-4a48-a5c3-65e3c8a6a5a7
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Job running in-process (local Hadoop)
2016-10-17 23:45:02,293 Stage-0 map = 0%,  reduce = 0%
2016-10-17 23:45:03,299 Stage-0 map = 100%,  reduce = 0%
Ended Job = job_local946612083_0002
MapReduce Jobs Launched:
Stage-Stage-0:  HDFS Read: 120 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
Time taken: 2.296 seconds
```

#### Query the table

```
hive> select * from artists;
OK
NULL    name2   {"url":"url2","picture":"picture2"}
NULL    name3   {"url":"url3","picture":"picture3"}
NULL    name1   {"url":"url1","picture":"picture1"}
Time taken: 0.077 seconds, Fetched: 3 row(s)
```
