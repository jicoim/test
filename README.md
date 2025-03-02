# Setting Up a 2-Instance Hadoop Cluster & Running Top-K Analysis


## 1. Cluster Setup

We will set up 2 instances for this configuration:
- `node-master`
- `node-worker1`

### Hostnames & Hosts File Setup

First, get the private IP address of each server:

```bash
hostname -I
```

Then, on **node-master**:

```bash
# Set hostname
echo "node-master" | sudo tee /etc/hostname

# Edit hosts file
sudo nano /etc/hosts

# Add these lines:
<Private IP of node-master> node-master
<Private IP of node-worker1> node-worker1
```

On **node-worker1**:

```bash
# Set hostname
echo "node-worker1" | sudo tee /etc/hostname

# Edit hosts file
sudo nano /etc/hosts

# Add these lines:
<Private IP of node-master> node-master
<Private IP of node-worker1> node-worker1
```

### Firewall Rules

Configure firewall to allow necessary connections:

```bash
# Allow laptop to access Hadoop UIs on node-master
sudo ufw allow from <YOUR_LAPTOP_PUBLIC_IP>

# Allow internal cluster traffic (replace 10.3.34 with your subnet)
sudo ufw allow from 10.3.34.0/24
```

### HDFS Formatting & Starting Services

On **node-master**:

```bash
# Format HDFS
hdfs namenode -format

# Start HDFS and YARN services
start-dfs.sh
start-yarn.sh
```

### Verify the Setup

Run the `jps` command on both nodes:

On **node-master** you should see: NameNode, ResourceManager, SecondaryNameNode
On **node-worker1** you should see: DataNode, NodeManager

You can also verify through web interfaces:
- HDFS UI: http://<node-master-ip>:9870
- YARN UI: http://<node-master-ip>:8088

## 2. Uploading Data to HDFS

First, copy your log file to the master node:

```bash
# From your local machine
scp sample.log exouser@<node-master-public-ip>:~/lab2/
```

Then, create HDFS directories and upload the file:

```bash
# Create your HDFS home directory (if needed)
hadoop fs -mkdir -p /user/exouser

# Upload the log file
hadoop fs -put sample.log /user/exouser/sample.log

# Verify the file was uploaded
hadoop fs -ls /user/exouser
```

## 3. MapReduce Job A: Summarize (Hour, IP)

This job counts visits per (hour, IP) combination from the log data.

### Prepare the Scripts

Ensure you have the `mapper_stat2.py` and `reducer_stat2.py` files.


### Run the Hadoop Streaming Job

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.4.0.jar \
-input /user/exouser/sample.log \
-output /user/exouser/stat2-summarized \
-mapper mapper_stat2.py \
-reducer reducer_stat2.py \
-file mapper_stat2.py \
-file reducer_stat2.py
```

### View the Output

```bash
hadoop fs -cat /user/exouser/stat2-summarized/part-00000
```

Sample output:
```
[03:00] 104.194.24.33    1
[03:00] 157.55.39.245    3
[03:00] 17.58.102.43     3
...
```

## 4. MapReduce Job B: Find Top K IPs Per Hour

This job processes the summarized data to find the top K IPs for each hour.

### Prepare the Scripts

Ensure you have the `topK_mapper.py` and `topK_reducer.py` files.


### Run the Hadoop Streaming Job

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.4.0.jar \
-input /user/exouser/stat2-summarized \
-output /user/exouser/topK-output \
-mapper topK_mapper.py \
-reducer "topK_reducer.py 10" \
-file topK_mapper.py \
-file topK_reducer.py
```

Note: The `10` argument to the reducer specifies that we want the top 10 IPs.

### View the Output

```bash
hadoop fs -cat /user/exouser/topK-output/part-00000
```

Sample output:
```
[03:00] 66.111.54.249    38
[03:00] 5.211.97.39      36
[03:00] 66.249.66.194    31
[03:00] 31.56.96.51      22
[03:00] 5.209.200.218    21
```

## 5. MapReduce Part 2: Filter Records by Time Period

This job filters the summarized data by time period and finds the top K IPs across that period.

### Prepare the Scripts

Ensure you have the `time_filter_mapper.py` and `time_filter_reducer.py` files (not shown here).


### Run the Hadoop Streaming Job

For example, to filter hours from 0 to 5 (i.e., 00:00 to 05:00) and output the top 5 IPs:

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.4.0.jar \
-input /user/exouser/stat2-summarized \
-output /user/exouser/timeFilter-output \
-mapper "time_filter_mapper.py 0 5" \
-reducer "time_filter_reducer.py 5" \
-file time_filter_mapper.py \
-file time_filter_reducer.py
```

### View the Output

```bash
hadoop fs -cat /user/exouser/timeFilter-output/part-00000
```

