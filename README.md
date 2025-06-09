# geeks3-flume
GEEKS3 : Flume

# Lab : Get Log Data with Flume

- [Upload multiple files to HDFS using FLUME](#upload-multiple-files-to-hdfs-using-flume)
- [Stream Log data from Syslog to HDFS](#stream-log-data-from-syslog-to-hdfs)
- [Read data from single file to HDFS](#read-data-from-single-file-to-hdfs)

## Upload multiple files to HDFS using FLUME

#### 1) Review file content and prepare a directory on HDFS
Dataset Explanations: Multiple CSV format files are on a local path. The data in the files is banking transactions.  

Investigate files on local directory
```sh
cd ~/flume_data
head -n 10 flume_data/001.csv
head -n 10 flume_data/002.csv
head -n 10 flume_data/003.csv
```
Create output directory on HDFS
```sh
hadoop fs -mkdir /user/student/flume_output
```
#### 2) Create a FLUME configuration file to spool a local directory
Investigate files on local directory
```sh
cd ~
gedit flume.conf &
```
#### 3) Flume Configuration File (flume.conf)
```cnf
agent1.sources = src1
agent1.channels = ch1
agent1.sinks = sink1

agent1.sources.src1.type = spooldir
agent1.sources.src1.channels = ch1
agent1.sources.src1.spoolDir = /home/student/flume_data
agent1.sources.src1.batchSize = 1000

agent1.channels.ch1.type = memory
agent1.channels.ch1.capacity = 100000000
agent1.channels.ch1.transactionCapacity = 5000

agent1.sinks.sink1.type = hdfs
agent1.sinks.sink1.channel = ch1
agent1.sinks.sink1.hdfs.batchSize = 5000
agent1.sinks.sink1.hdfs.path = /user/student/flume_output
agent1.sinks.sink1.hdfs.filePrefix = log-output
agent1.sinks.sink1.hdfs.rollSize = 5120000
agent1.sinks.sink1.hdfs.rollInterval = 10
agent1.sinks.sink1.hdfs.rollCount = 0
agent1.sinks.sink1.hdfs.fileType = DataStream
agent1.sinks.sink1.hdfs.writeFormat = Text
```
#### 4)	Start a FLUME agent to ingest files into HDFS
```
$ flume-ng agent -n agent1 -f flume.conf -Xms1024m -Xmx2048m
```
NOTE: -Xms and –Xmx is memory configurable for Flume Java Heap

#### 5) View the result on HDFS
View the result on HDFS  
```
[Open new terminal]

hadoop fs -cat flume_output/*
```

## Stream Log data from Syslog to HDFS

#### 1)	Import data from Syslog to HDFS using Flume
Create output directory on HDFS
```sh
hadoop fs -mkdir /user/student/syslog
```

#### 2)	Create a FLUME configuration file to listen syslog port
Investigate files on local directory
```sh
cd ~
nano flume_rsyslog.conf
```

#### 3)	Flume Configuration File (flume_rsyslog.conf)
```cnf
# Name the components on this agent
a1.sources = src1
a1.sinks = sink1
a1.channels = ch1

# I'll be using TCP based Syslog source
a1.sources.src1.type = syslogtcp
# the port that Flume Syslog source will listen on
a1.sources.src1.port = 5140
# the hostname that Flume Syslog source will be running on
a1.sources.src1.host = localhost
a1.sources.src1.channels = ch1

# Describe the sink
a1.sinks.sink1.type = hdfs
a1.sinks.sink1.channel = ch1
a1.sinks.sink1.hdfs.path = /user/student/syslog
a1.sinks.sink1.hdfs.batchSize = 5000
a1.sinks.sink1.hdfs.rollSize = 512000
a1.sinks.sink1.hdfs.fileType = DataStream
a1.sinks.sink1.hdfs.writeFormat = Text

# Use a channel which buffers events in memory
a1.channels.ch1.type = memory
a1.channels.ch1.capacity = 10000000
a1.channels.ch1.transactionCapacity = 5000
```

#### 4)	Start a rsyslog service
Modified the /etc/rsyslog.conf
```sh
sudo nano /etc/rsyslog.d/flume.conf
```
Provides TCP syslog reception
```cnf
# Provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
*.*     @@localhost:5140
```
Start a rsyslog service
```sh
sudo service rsyslog restart
```

#### 5)	Start a FLUME agent to ingest files into HDFS
```sh
flume-ng agent -n a1 -f flume_rsyslog.conf -Xms1024m -Xmx2048m
```
NOTE: -Xms and –Xmx is memory configurable for Flume Java Heap

#### 6)	Test log forward to HDFS
```sh
[Open new terminal]
```
Generate log data
```sh
logger -t test 'testing flume with syslog'
cat /var/log/syslog 
```
View the result on HDFS 
```sh
hadoop fs -cat syslog/*
```

## Read data from single file to HDFS

#### 1)	Read data from log file to flume
Investigate files on local directory
```
cd ~
nano readlog.conf
```

#### 2)	Flume Configuration File (readlog.conf)
```
a1.sources = src1
a1.sinks = sink1
a1.channels = ch1

#Source
a1.sources.src1.type = exec
a1.sources.src1.command = tail -f /var/log/dmesg
a1.sources.src1.channels = ch1

# Describe the sink
a1.sinks.sink1.type = hdfs
a1.sinks.sink1.channel = ch1
a1.sinks.sink1.hdfs.path = /user/student/dmessages
a1.sinks.sink1.hdfs.batchSize = 5000
a1.sinks.sink1.hdfs.rollSize = 512000
a1.sinks.sink1.hdfs.fileType = DataStream
a1.sinks.sink1.hdfs.writeFormat = Text

# Use a channel which buffers events in memory
a1.channels.ch1.type = memory
a1.channels.ch1.capacity = 10000000
a1.channels.ch1.transactionCapacity = 5000
```

#### 3)	Start a FLUME agent to ingest files into HDFS
```
flume-ng agent -n a1 -f readlog.conf -Xms1024m -Xmx2048m
```
NOTE: -Xms and –Xmx is memory configurable for Flume Java Heap

#### 4)	Test log forward to HDFS
```
[Open new terminal]
```
Generate log data
```
sudo su -c 'echo HELLO > /dev/kmsg'
sudo dmesg
sudo su -c "dmesg > /var/log/dmesg"
```
View the result on HDFS 
```
hdfs dfs -cat dmessages/*
```
