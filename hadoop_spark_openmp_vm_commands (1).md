# Linux VM Commands: Hadoop, Spark, OpenMP (+ MPI)

Ubuntu VM / WSL. Run blocks top to bottom. Every block is copy-pasteable.

**Rules**
- Run everything as your **normal user** (not root). `sudo` is already included where needed.
- Hadoop and Spark share **JDK 11** and **Spark 3.5.x** (Spark 4.x needs Java 17+, so avoid it).
- Run `hdfs namenode -format` **only once**.
- `HOME` = `~` = `/home/<your-user>`. Input files live in `~/input`, Java code in `~/wordcount`.
- ARM VM? Replace `amd64` with `arm64` in every `JAVA_HOME`.

## Index and run order

| Part | Experiment | Run order |
|---|---|---|
| 0 | Common prerequisites | 1st |
| A | Hadoop 3.3.6 + MapReduce WordCount (Exp 8) | 2nd |
| B | Spark 3.5.3 WordCount, local + HDFS (Exp 10) | 3rd (HDFS part needs A running) |
| C | OpenMP + MPI (Exp 9) | any time after 0 |
| D | Handy checks and port cleanup | anytime |

Every part has: **Aim, commands, expected output, troubleshooting commands, and a fix table**.

---

## 0. Common prerequisites

```bash
sudo apt update
sudo apt install -y curl wget git unzip build-essential nano openjdk-11-jdk openssh-server
java -version
javac -version
```

Confirm the JDK path (must match `JAVA_HOME` below):
```bash
ls /usr/lib/jvm/
readlink -f $(which java)
```

**Troubleshooting (Part 0)**

| Problem | Command / Fix |
|---|---|
| `E: Unable to locate package` | `sudo apt update` then retry |
| `Could not get lock /var/lib/dpkg/lock` | Wait 1-2 min (auto-update running), then retry the `apt` command |
| `java: command not found` | `sudo apt install -y openjdk-11-jdk` |
| Two Java versions installed | `sudo update-alternatives --config java` and pick the java-11 entry |
| `wget` fails (no internet) | `ping -c 2 archive.apache.org`, check VM network is NAT or Bridged |
| `sudo: unable to resolve host` | Harmless; or `sudo sh -c 'echo "127.0.0.1 $(hostname)" >> /etc/hosts'` |

---

# PART A: HADOOP 3.3.6 (single-node, pseudo-distributed)

**Aim:** Install Hadoop 3.x and run a MapReduce word count job.

## A1. Passwordless SSH to localhost

```bash
sudo service ssh start
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh -o StrictHostKeyChecking=no localhost
exit
```
- If `ssh-keygen` says the file exists, type `n` (key already there) and continue.
- Test: `ssh localhost` must log in **without a password**, then `exit`.

## A2. Environment variables (run once only)

```bash
cat >> ~/.bashrc <<'EOF'
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
EOF
source ~/.bashrc
echo $JAVA_HOME
echo $HADOOP_HOME
```

## A3. Download and install Hadoop

```bash
cd ~/Downloads 2>/dev/null || mkdir -p ~/Downloads && cd ~/Downloads
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz
sudo mv hadoop-3.3.6 /usr/local/hadoop
sudo chown -R "$USER":"$USER" /usr/local/hadoop
hadoop version
```

## A4. Configuration files

**hadoop-env.sh**
```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```

**core-site.xml**
```bash
cat > $HADOOP_HOME/etc/hadoop/core-site.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
</configuration>
EOF
```

**hdfs-site.xml**
```bash
cat > $HADOOP_HOME/etc/hadoop/hdfs-site.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/usr/local/hadoop/hadoop_data/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/usr/local/hadoop/hadoop_data/hdfs/datanode</value>
  </property>
</configuration>
EOF
```

**mapred-site.xml** (in Hadoop 3.x this file already exists; there is no `.template`, so overwrite it directly)
```bash
cat > $HADOOP_HOME/etc/hadoop/mapred-site.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.application.classpath</name>
    <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
  </property>
</configuration>
EOF
```

**yarn-site.xml**
```bash
cat > $HADOOP_HOME/etc/hadoop/yarn-site.xml <<'EOF'
<?xml version="1.0"?>
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
EOF
```

## A5. Create HDFS directories, format, start

```bash
mkdir -p /usr/local/hadoop/hadoop_data/hdfs/namenode
mkdir -p /usr/local/hadoop/hadoop_data/hdfs/datanode
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```
Expected in `jps`: `NameNode`, `DataNode`, `SecondaryNameNode`, `ResourceManager`, `NodeManager`, `Jps`.

Web UIs: HDFS `http://localhost:9870` | YARN `http://localhost:8088`

## A6. Create input file and upload to HDFS

```bash
mkdir -p ~/input
printf 'hello hadoop\nhello cloud computing\ncloud lab hadoop\n' > ~/input/file.txt
cat ~/input/file.txt
hdfs dfs -mkdir -p /input
hdfs dfs -put ~/input/file.txt /input/
hdfs dfs -ls /input
```
Re-upload if the file changed:
```bash
hdfs dfs -rm -f /input/file.txt
hdfs dfs -put ~/input/file.txt /input/
```

## A7. WordCount program

```bash
mkdir -p ~/wordcount
cd ~/wordcount
cat > WordCount.java <<'EOF'
import java.io.IOException;
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
    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            String[] words = value.toString().trim().split("\\s+");
            for (String w : words) {
                if (w.isEmpty()) continue;
                word.set(w);
                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public void reduce(Text key, Iterable<IntWritable> values, Context context)
                throws IOException, InterruptedException {
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
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
EOF
```

## A8. Compile, run, view output

```bash
cd ~/wordcount
javac -classpath $(hadoop classpath) WordCount.java
jar cf wc.jar WordCount*.class
hadoop jar wc.jar WordCount /input/file.txt /output/wc
hdfs dfs -ls /output/wc
hdfs dfs -cat /output/wc/part-r-00000
```
Expected output for the sample `file.txt` in A6:
```text
cloud	2
computing	1
hadoop	2
hello	2
lab	1
```
Output folder must **not** exist before a rerun:
```bash
hdfs dfs -rm -r /output/wc
```

## A9. Stop Hadoop

```bash
stop-yarn.sh
stop-dfs.sh
jps
```
Start again later (**no** re-format):
```bash
sudo service ssh start
start-dfs.sh
start-yarn.sh
jps
```

## A10. Hadoop troubleshooting

### Step 1: health-check commands (run first)
```bash
jps
hdfs dfsadmin -report
hdfs dfs -ls /
yarn node -list
yarn application -list -appStates ALL
ss -tulpn
```
`ss -tulpn` should show ports `9000`, `9870`, `8088` while Hadoop runs.

### Step 2: read the logs
```bash
ls $HADOOP_HOME/logs
tail -n 50 $HADOOP_HOME/logs/hadoop-*-namenode-*.log
tail -n 50 $HADOOP_HOME/logs/hadoop-*-datanode-*.log
tail -n 50 $HADOOP_HOME/logs/yarn-*-resourcemanager-*.log
tail -n 50 $HADOOP_HOME/logs/yarn-*-nodemanager-*.log
```
Job logs: `yarn logs -applicationId <application_id>` (get the ID from `yarn application -list -appStates ALL`).

### Step 3: fix by symptom

| Problem | Command / Fix |
|---|---|
| `hadoop: command not found` | `source ~/.bashrc` then `echo $HADOOP_HOME` and `ls /usr/local/hadoop/bin/hadoop` |
| `JAVA_HOME is not set` on start | `grep JAVA_HOME $HADOOP_HOME/etc/hadoop/hadoop-env.sh`; if missing, re-run the echo line in A4 |
| `Connection refused` on port 22 | `sudo service ssh start` then `sudo service ssh status` |
| SSH asks for a password | `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` then `ssh localhost` |
| `Permission denied` inside `/usr/local/hadoop` | `sudo chown -R "$USER":"$USER" /usr/local/hadoop` |
| DataNode missing in `jps`, or log says `Incompatible clusterIDs` | Run the **full reset** block below |
| NameNode missing after VM reboot | `start-dfs.sh`; if it still fails and the namenode folder is empty, run the full reset block |
| `Name node is in safe mode` | `hdfs dfsadmin -safemode leave` |
| `put: File exists` | `hdfs dfs -put -f ~/input/file.txt /input/` |
| `Input path does not exist` | `hdfs dfs -ls /input`; re-upload the file (A6) |
| `Output directory ... already exists` | `hdfs dfs -rm -r /output/wc` |
| `Could not find or load main class ...MRAppMaster` | `grep -c HADOOP_MAPRED_HOME $HADOOP_HOME/etc/hadoop/mapred-site.xml` must print `4`; if not, redo the mapred-site step in A4 |
| Job stuck at `ACCEPTED` | `yarn node -list` (NodeManager must be RUNNING); check `jps`; give the VM at least 4 GB RAM |
| `javac: package org.apache.hadoop... does not exist` | `hadoop classpath` must print jar paths; if not, fix PATH (A2) |
| `ClassNotFoundException: WordCount` | Rebuild inside `~/wordcount`: `jar cf wc.jar WordCount*.class` |
| Web UI on 9870 / 8088 does not open | `jps`, then `ss -tulpn`; from a host browser use the VM IP from `hostname -I` |
| Running as root: `HDFS_NAMENODE_USER not defined` | Use a normal user (recommended) |

### Full reset (only if the above fails; **deletes all HDFS data**)
```bash
stop-yarn.sh
stop-dfs.sh
rm -rf /usr/local/hadoop/hadoop_data/hdfs/namenode/* /usr/local/hadoop/hadoop_data/hdfs/datanode/*
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
hdfs dfs -mkdir -p /input
hdfs dfs -put ~/input/file.txt /input/
```

---

# PART B: APACHE SPARK 3.5.x (local mode + optional HDFS)

**Aim:** Install Spark and run a word count using RDDs.

Uses the same JDK 11 already set for Hadoop.

## B1. Download and install Spark

```bash
cd ~/Downloads 2>/dev/null || mkdir -p ~/Downloads && cd ~/Downloads
wget https://archive.apache.org/dist/spark/spark-3.5.3/spark-3.5.3-bin-hadoop3.tgz
tar -xzf spark-3.5.3-bin-hadoop3.tgz
sudo mv spark-3.5.3-bin-hadoop3 /usr/local/spark
sudo chown -R "$USER":"$USER" /usr/local/spark
```

## B2. Environment variables (run once only)

```bash
cat >> ~/.bashrc <<'EOF'
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin
EOF
source ~/.bashrc
echo $SPARK_HOME
```

## B3. Verify

```bash
spark-shell --version
```

## B4. Create input file

```bash
mkdir -p ~/sparklab
cd ~/sparklab
printf 'cloud computing cloud lab\nspark processes data\n' > input.txt
cat input.txt
```

## B5. Word count (local files)

Start the shell **from `~/sparklab`**:
```bash
cd ~/sparklab
spark-shell
```
At the `scala>` prompt (**not** Bash), paste:
```scala
val inputFile = sc.textFile("input.txt")
val counts = inputFile.flatMap(line => line.split("\\s+")).filter(_.nonEmpty).map(word => (word, 1)).reduceByKey(_ + _)
counts.collect().foreach(println)
counts.saveAsTextFile("output")
:quit
```
Spark UI while the shell is open: `http://localhost:4040`

Expected `collect()` output (order may differ):
```text
(cloud,2)
(computing,1)
(lab,1)
(spark,1)
(processes,1)
(data,1)
```
View output (back in Bash):
```bash
cd ~/sparklab
ls output
cat output/part-*
```
`part-*` is needed because the result can be split over several part files.

Rerun (Spark refuses to overwrite):
```bash
rm -r ~/sparklab/output
```

## B6. Optional: Spark on HDFS (Hadoop must be running; `jps` shows all 5 daemons)

```bash
cd ~/sparklab
hdfs dfs -mkdir -p /input
hdfs dfs -put -f input.txt /input/input.txt
hdfs dfs -ls /input
spark-shell
```
At the `scala>` prompt:
```scala
val hdfsInput = sc.textFile("hdfs://localhost:9000/input/input.txt")
val hdfsCounts = hdfsInput.flatMap(line => line.split("\\s+")).filter(_.nonEmpty).map(word => (word, 1)).reduceByKey(_ + _)
hdfsCounts.collect().foreach(println)
hdfsCounts.saveAsTextFile("hdfs://localhost:9000/output/spark-wordcount")
:quit
```
View in Bash:
```bash
hdfs dfs -ls /output/spark-wordcount
hdfs dfs -cat /output/spark-wordcount/part-*
```
Remove before rerun:
```bash
hdfs dfs -rm -r /output/spark-wordcount
```

## B7. Spark troubleshooting

### Diagnose commands
```bash
echo $SPARK_HOME
echo $JAVA_HOME
java -version
ls $SPARK_HOME/bin
spark-shell --version
ls ~/sparklab
ss -tulpn
```
Inside `spark-shell` (`scala>`):
```scala
sc.master
counts.getNumPartitions
```

### Fix by symptom

| Problem | Command / Fix |
|---|---|
| `spark-shell: command not found` | `export SPARK_HOME=/usr/local/spark` then `export PATH=$PATH:$SPARK_HOME/bin` (or `source ~/.bashrc`) |
| `Input path does not exist: file:/.../input.txt` | `cd ~/sparklab` and start `spark-shell` there, or use the full path `sc.textFile("/home/<user>/sparklab/input.txt")` |
| `FileAlreadyExistsException: output` | `rm -r ~/sparklab/output` (HDFS: `hdfs dfs -rm -r /output/spark-wordcount`) |
| Output shows only some words | You read one file; use `cat output/part-*` |
| Java version / `UnsupportedClassVersionError` | Use Spark 3.5.x with JDK 11; Spark 4.x needs `JAVA_HOME` on JDK 17 |
| `Connection refused: localhost:9000` (HDFS mode) | Hadoop is down: `start-dfs.sh` then `jps` |
| `Port 4040 is in use` | Harmless; Spark uses 4041. Or close the old `spark-shell` |
| Warning: hostname resolves to loopback address | Harmless; or `export SPARK_LOCAL_IP=127.0.0.1` |
| Warning: unable to load native-hadoop library | Harmless, ignore |
| Too many INFO log lines | `cp $SPARK_HOME/conf/log4j2.properties.template $SPARK_HOME/conf/log4j2.properties` then `sed -i 's/rootLogger.level = info/rootLogger.level = warn/' $SPARK_HOME/conf/log4j2.properties` |
| `wget` gives 404 | Browse `https://archive.apache.org/dist/spark/` and change the version in B1 |
| Shell is slow / low memory | `spark-shell --master local[2] --driver-memory 1g` |

---

# PART C: OpenMP (and MPI, same experiment)

**Aim:** Use OpenMP for thread-based parallelism and MPI for process-based parallelism.

## C1. Install and verify

```bash
sudo apt update
sudo apt install -y build-essential openmpi-bin libopenmpi-dev
gcc --version
mpirun --version
nproc
```

## C2. Working folder

```bash
mkdir -p ~/parallel
cd ~/parallel
```

## C3. OpenMP Hello

```bash
cat > omp_hello.c <<'EOF'
#include <omp.h>
#include <stdio.h>

int main() {
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        printf("Hello from thread %d\n", tid);
    }
    return 0;
}
EOF
gcc -fopenmp omp_hello.c -o omp_hello
./omp_hello
```
Set the thread count:
```bash
export OMP_NUM_THREADS=4
./omp_hello
```

## C4. OpenMP reduction

```bash
cat > omp_reduction.c <<'EOF'
#include <stdio.h>
#include <omp.h>

int main() {
    int sum = 0;
    int arr[10] = {1,2,3,4,5,6,7,8,9,10};

    #pragma omp parallel for reduction(+:sum)
    for (int i = 0; i < 10; i++) {
        sum += arr[i];
    }

    printf("Sum = %d\n", sum);
    return 0;
}
EOF
gcc -fopenmp omp_reduction.c -o omp_reduction
./omp_reduction
```
Expected: `Sum = 55`

## C5. OpenMP data sharing (private, firstprivate, lastprivate, critical)

```bash
cat > omp_data_sharing.c <<'EOF'
#include <stdio.h>
#include <omp.h>

int main(void) {
    int sharedCounter = 0;
    int lastValue = -1;
    int initialValue = 10;

    #pragma omp parallel private(sharedCounter) firstprivate(initialValue)
    {
        int threadId = omp_get_thread_num();
        sharedCounter = threadId;
        initialValue += threadId;
        printf("thread=%d private=%d firstprivate-copy=%d\n",
               threadId, sharedCounter, initialValue);
    }

    #pragma omp parallel for lastprivate(lastValue)
    for (int i = 0; i < 4; i++) {
        lastValue = i;
    }
    printf("lastprivate value after loop = %d (expected 3)\n", lastValue);

    #pragma omp parallel for
    for (int i = 0; i < 1000; i++) {
        #pragma omp critical
        sharedCounter++;
    }
    printf("critical-section counter = %d (expected 1000)\n", sharedCounter);
    return 0;
}
EOF
gcc -fopenmp omp_data_sharing.c -o omp_data_sharing
./omp_data_sharing
```
Thread output order changes on every run (normal).

Expected output:
```text
thread=0 private=0 firstprivate-copy=10
thread=1 private=1 firstprivate-copy=11
...
lastprivate value after loop = 3 (expected 3)
critical-section counter = 1000 (expected 1000)
```

## C6. MPI Hello

```bash
cat > mpi_hello.c <<'EOF'
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    printf("Hello from process %d of %d\n", rank, size);

    MPI_Finalize();
    return 0;
}
EOF
mpicc mpi_hello.c -o mpi_hello
mpirun --oversubscribe -np 4 ./mpi_hello
```
Expected output (order varies):
```text
Hello from process 0 of 4
Hello from process 1 of 4
Hello from process 2 of 4
Hello from process 3 of 4
```
- `--oversubscribe` avoids "not enough slots" when the VM has fewer than 4 cores.
- If you are root, also add `--allow-run-as-root`.

## C7. OpenMP / MPI troubleshooting

### Diagnose commands
```bash
gcc --version
which gcc mpicc mpirun
mpicc --showme
nproc
echo $OMP_NUM_THREADS
ldd ./omp_hello
```
`ldd ./omp_hello` must list `libgomp.so.1` (proves OpenMP was linked).

### Fix by symptom

| Problem | Command / Fix |
|---|---|
| `omp.h: No such file or directory` | `sudo apt install -y build-essential` |
| `undefined reference to omp_get_thread_num` | Compile **and link** with the flag: `gcc -fopenmp file.c -o file` |
| `warning: ignoring '#pragma omp'` | Same cause: `-fopenmp` missing |
| Only one thread prints | `export OMP_NUM_THREADS=4` then `./omp_hello`; check `nproc` |
| `stray '#' in program` or strange characters | Windows line endings: `sed -i 's/\r$//' file.c` |
| `mpi.h: No such file or directory` | `sudo apt install -y libopenmpi-dev`; compile with `mpicc`, not `gcc` |
| `mpicc: command not found` / `mpirun: command not found` | `sudo apt install -y openmpi-bin libopenmpi-dev` |
| `There are not enough slots available` | `mpirun --oversubscribe -np 4 ./mpi_hello` |
| `mpirun has detected an attempt to run as root` | Use a normal user, or add `--allow-run-as-root` |
| `Permission denied: ./mpi_hello` | `chmod +x mpi_hello` |
| Output order differs each run | Normal: threads/processes run concurrently |
| `sharedCounter` result wrong | Keep the `#pragma omp critical` line; without it the update is a race condition |

---

# D. Handy checks

```bash
java -version
javac -version
hadoop version
spark-shell --version
gcc --version
mpirun --version
jps
ss -tulpn | grep -E '9000|9870|8088|4040'
```

Free a busy port:
```bash
lsof -i :9870
kill -9 <PID>
```

**Result line for the record:**
`Thus, the experiment was successfully completed and the output was verified.`
