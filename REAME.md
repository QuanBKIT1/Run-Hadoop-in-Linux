# How to run Hadoop in Linux

## Table of Content

* [Prerequisites](#prerequisites)
* [Setting up a Pseudo Distributed](#setting-up-a-pseudo-distributed)
* [Execution](#execution)
* [Run a MapReduce job](#run-a-mapreduce-job)

# Prerequisites
- Linux environment: Unbuntu 18.04 LTS 
- Java JDK 8
- ssh and pdsh
- Hadoop distribution
s
References: https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html


# Setting up a Pseudo-Distributed in Yarn Mode

Goal: Start successful daemon:
- ResourceMananger
- NodeManager
- Namenode
- SecondaryNamenode
- Datanode

Install Java JDK 8 ,ssh and pdsh 
---
```shell
  $ sudo apt-get install openjdk-8-jdk
  $ sudo apt-get install ssh
  $ sudo apt-get install pdsh
```

Setup passphraseless ssh
---
```shell
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
```
Note: 
- `pdsh` uses `rsh` by default, not `ssh`. So, add a line to the end of file `~/.bashrc`:
```shell
export PDSH_RCMD_TYPE=ssh
```

Edit environment variables 
---


1. Set variable in `~/.bashrc`: 

```shell
export JAVA_HOME=/usr/java/latest
export HADOOP_HOME=/path/to/hadoop
export PATH=${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${PATH}
```

2. Set variable in `${HADOOP_HOME}/etc/hadoop/hadoop-env.sh`:

Uncomment `JAVA_HOME` and set:
```shell
export JAVA_HOME=/usr/java/latest
```

Note:
- `/usr/java/latest` can be `usr/liv/jvm/java-1.8.0-openjdk-amd64`

- All terminal command is executed in directory `$HADOOP_HOME` 

Configuration
---
Edit files:
- `${HADOOP_HOME}/etc/hadoop/core-site.xml`

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>

    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/username/tmp</value>
    </property>

</configuration>
```

**Note: `hadoop.tmp.dir` value is where Hadoop namenode, datanode and namenode secondary store its data, by default is `/tmp/` dir, it is deleted after restart**
- `${HADOOP_HOME}/etc/hadoop/hdfs-site.xml`
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
- `${HADOOP_HOME}/etc/hadoop/mapreduce-site.xml`
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```
- `${HADOOP_HOME}/etc/hadoop/yarn-site.xml`
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

## Execution
1. Format the filesystem for first time:
```shell
  $ hdfs namenode -format
```
2. Start ResourceManager daemon and NodeManager daemon:

```shell
 $ start-yarn.sh
```

3. Start NameNode daemon and DataNode daemon:
```shell
 $ start-dfs.sh
```

**Note: You can use command `start-all.sh` to start all daemons**

Check for working:
```shell
jps
```

# Run a MapReduce job
Create  WordCount Job to sumbit
---
Create a WordCount.java file:
```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

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
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

To compile WordCount.java file above, we need set classpath to hadoop classpath, by following step:
1. Get all hadoop classpath:
```shell
hadoop classpath
``` 
2. Copy all hadoop classpath, edit `~/.bashrc` file:
```shell
export CLASSPATH='content of hadoop classpath' 
```
3. Apply the change: 
```shell
source ~/.bashrc
```
Complile WordCount.java:
```shell
javac WordCount.java
```
After compile, it generates 3 classes file: `WordCount.class`, `WordCount$TokenizerMapper.class`, `WordCount$IntSumReducer.class`.

Create a jar file from these classes:
```shell
jar cf wc.jar WordCount*.class
```

**Note: Các command được thực hiện trên working directory: $HADOOP_HOME**
- Create HDFS directories:
```
$ bin/hdfs dfs -mkdir -p /user/
```

- Copy files or directories from your local file system to Hadoop's HDFS
```
bin/hdfs dfs -put from_local to_hdfs 
```

Prepare data in HDFS 
---
Create txt file in local and add some content: 
```shell
touch wc_data.txt

nano wc_data.txt

# Add some content and close file
```
Upload data from local to HDFS:
```
hadoop fs -mkdir -p /user/username/

hadoop fs -put wc_data.txt /user/username/wc_data.txt
```

Run a MapReduce job locally
---

```shell
hadoop jar wc.jar WordCount /user/username/wc_data.txt /user/username/output
```

Run a MapReduce job on YARN
---

```shell
yarn jar wc.jar WordCount /user/username/wc_data.txt /user/username/output
```

When you’re done, stop the daemons with:
```shell
stop-yarn.sh

stop-dfs.sh
```

# Web interface
You can see the namenode, datanode and mapreduce job status in web interface 
- ResourceManager - http://localhost:8088/
- NameNode - http://localhost:9870/
