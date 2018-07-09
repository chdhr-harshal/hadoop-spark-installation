# Apache Hadoop and Apache Spark installation and configuration to use with Jupyter notebook on Macbook Pro running OSX High Sierra

As a new user of Apache Spark at work, I wished to install it on my local machine and soon found out that there are plenty of tutorials out there. Some of the issues that I encountered were that most of the tutorials were for older versions. While the overall process has not changed much, there are some minor changes which can result in rather arcane bunch of errors if not correctly accounted for. Secondly, there are "shortcuts" to install pySpark which can get you up and running in a couple of minutes but are not ideal installation procedures. Just as before, these shortcuts led me to waste hours of time debugging bunch of errors, and the solutions suggested on Stack Overflow may work for you, or not, based upon your underlying installation and configuration. So, I present a detailed guide of how I got Hadoop and Spark working on my local machine.

## Installation of Homebrew and Cask

Homebrew is a free and open-source software package management system that simplifies the installation of software on OSX.
~~~~
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
$ brew install caskroom/cask/brew-cask
~~~~

## Installation of Java and Scala

We can install both Java and Scala using homebrew.
~~~~
$ brew update
$ brew cask install java
$ brew cask install scala
~~~~

Alternatively, you can also install JDK manually from this <a href="http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html">link</a>.

### Set JAVA_HOME and SCALA_HOME environment variables

First, we need to set the environment variables in your ~/.bash_profile or ~/.bashrc (or .zprofile/.zshrc if you use Z shell instead of Bash). Find the directory where your Java is installed with this command.
~~~~
$ /usr/libexec/java_home
/Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home

$ which scala
/usr/local/bin/scala
~~~~

Now we add these to variables at the bottom of your ~/.bash_profile and then source your ~/.bash_profile
~~~~
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home"
export SCALA_HOME="/usr/local/bin/scala"
~~~~

## Configure SSH

Hadoop requires SSH to be allowed for all users (it is disabled by default in OSX High Sierra). To enable it, go to `System Preferences -> Sharing`, change `Allow access for: All Users`

If you need to, generate SSH keys.
~~~~
$ ssh-keygen -t rsa -P “”
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
~~~~

Finally, you can test the connection to localhost.
~~~~
ssh localhost
~~~~
Assuming SSH is configured properly, now we proceed to next step of installing Hadoop. If you are only interested in installation of Apache Spark without HDFS, you can jump forward directly to the Installation of Apache Spark section of this guide.

## Installation of Apache Hadoop 

Just like Java and Scala before, we use homebrew to install Hadoop
~~~~
$ brew install hadoop
~~~~

This will install hadoop under `/usr/local/Cellar/hadoop`. We would like to set up the environment variables ~/.bash_profile to reflect this location. (Note, check the version of hadoop and replace it with appropriate one you installed.)
~~~~
export HADOOP_HOME="/usr/local/Cellar/hadoop/3.1.0/libexec"
export HADOOP_CONF_DIR="/usr/local/Cellar/hadoop/3.1.0/libexec"
~~~~
Next, we need to make changes to the configuration of Hadoop in the HADOOP_CONF_DIR inorder to set it up in the pseudo-distributed mode (simulates a cluster on your local machine by running different instances of JVM).

### 1. Changes to `hadoop-env.sh`

Open `/usr/local/Cellar/hadoop/3.1.0/libexec/etc/hadoop/hadoop-env.sh`.

Replace the line
~~~~
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"
~~~~
with 
~~~~
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true -Djava.security.krb5.realm= -Djava.security.krb5.kdc="
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home"
~~~~

### 2. Changes to `core-site.xml`

Open `/usr/local/Cellar/hadoop/3.1.0/libexec/etc/hadoop/core-site.xml`.

Insert this inside the `<configuration> ... </configuration>` tags.
~~~~
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/Cellar/hadoop/hdfs/tmp</value>
        <description>A base for other temporary directories.</description>
    </property>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://localhost:8020</value>
    </property>
</configuration>
~~~~

### 3. Changes to `hdfs-site.xml`

Open `/usr/local/Cellar/hadoop/3.1.0/libexec/etc/hadoop/hdfs-site.xml`.

Replace the default value '3' with '1'.
~~~~
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
~~~~

### 4. Changes to `mapred-site.xml`

Open `/usr/local/Cellar/hadoop/3.1.0/libexec/etc/hadoop/mapred-site.xml`.

Insert this inside the `<configuration> ... </configuration>` tags.
~~~~
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
~~~~
This uses YARN as our cluster manager.

### 5. Changes to `yarn-site.xml`

Open `/usr/local/Cellar/hadoop/3.1.0/libexec/etc/hadoop/yarn-site.xml`.

Insert this inside the `<configuration> ... </configuration>` tags.
~~~~
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
~~~~

### Format HDFS

First, we format the hdfs with these two commands.
~~~~
$ cd ${HADOOP_HOME}
$ bin/hdfs namenode -format
~~~~

### Fire up Hadoop and Yarn services.
~~~~
$ ${HADOOP_HOME}/sbin/start-dfs.sh
$ ${HADOOP_HOME}/sbin/start-yarn.sh
~~~~

You can use `jps` command to check that the services are working
~~~~
$ /usr/local/Cellar/hadoop/3.1.0/libexec/sbin » jps                          harshal@Harshals-MBP
59616 ResourceManager
92484 DataNode
92388 NameNode
96520 Jps
92619 SecondaryNameNode
59706 NodeManager
~~~~

### Create directory in HDFS for MapReduce
~~~~
$ ${HADOOP_HOME}/bin/hdfs dfs -mkdir -p /user/harshal/test_dir
~~~~

Now that we have created the directory, let us briefly learn how to transfer data between local and hdfs.

#### Copy to HDFS
~~~~
$ ${HADOOP_HOME}/bin/hdfs dfs -put ~/Projects/F1/data.csv /user/harshal/test_dir/.
~~~~
copies `data.csv` from my local directory to the HDFS while,
~~~~
$ ${HADOOP_HOME}/bin/hdfs dfs -get /user/harshal/test_dir/data.csv ~/Projects/F1/.
~~~~
copies it from HDFS to local directory.

### Add to PATH

Add this line to the bottom of your ~/.bash_profile to access these hdfs services quickly.
~~~~
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
~~~~
Now, you can start or stop hdfs/ yarn directly or put get files from HDFS to local machine easily. (Don't forget to source your ~/.bash_profile)
~~~~
$ start-dfs.sh
$ stop-dfs.sh

$ start-yarn.sh
$ stop-yarn.sh

$ hdfs dfs -get </hdfs/path> </local/path>
$ hdfs dfs -put </local/path> </hdfs/path>
$ hdfs dfs -ls </hdfs/path>
~~~~

### Web interface

At this point you should be able to visit <a href="http://localhost:9870">http://localhost:9870</a> to checkout the Hadoop web-interface. You can also access the node information at <a href="http://localhost:8088">http://localhost:8088</a>.

## Installation of Apache Spark

Download the latest Spark 2.3.1 version from <a href="https://spark.apache.org/downloads.html">https://spark.apache.org/downloads.html</a>. Untar the tarball and copy it to the preferred location. I use `/usr/local/bin/spark`.
~~~~
$ tar -xvf spark-2.3.1-bin-hadoop2.7.tgz
$ mv spark-2.3.1-bin-hadoop2.7 /usr/local/bin/spark
~~~~
Set the environment variable SPARK_HOME in the ~/.bash_profile.
~~~~
export SPARK_HOME="/usr/local/bin/spark"
~~~~
Finally, add this line to the bottom of your ~/.bash_profile to use Spark and Scala functionalities quickly.
~~~~
export PATH="$JAVA_HOME/bin:$SCALA_HOME/bin:$SPARK_HOME:$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH"
~~~~
At this point, you should be able to use pyspark-shell directly using the command, and then access the web interface of Apache Spark at <a href="http://localhost:4040/">http://localhost:4040/</a>.
~~~~
$ pyspark
~~~~

## Using PySpark with ipython and Jupyter notebook

Last step required to provide a clean access to PySpark only when required is to create appropriate profiles for ipython and kernels for Jupyter notebook, so that we create a Spark instance only when required.

### ipython

First, create iPython profile as follows
~~~~
$ ipython profile create pyspark
~~~~
Next, go to `~/.ipython/profile_pyspark/startup/` and create a file `00-pyspark-setup.py` in the directory containing,
~~~~
import os
import sys
spark_home = os.environ.get('SPARK_HOME', None)
if not spark_home:
    raise ValueError("SPARK_HOME environment variable is not set.")
sys.path.insert(0, os.path.join(spark_home, 'python'))
sys.path.insert(0, os.path.join(spark_home, 'python/lib/py4j-0.10.4.src.zip'))
exec(open(os.path.join(spark_home, 'python/pyspark/shell.py')).read())
~~~~

Now, you can fire up iPython with Spark instance only when required using this new profile as follows. The spark context can be accessed using `sc` variable. The default profile of ipython would not unnecessarily try to load PySpark. 

~~~
ipython --profile=pyspark
~~~

### Jupyter notebook

As far as I am aware, we cannot create separate profile for Jupyter notebooks, however, we can use different kernels to achieve similar functionality. 

First, go to `~/.ipython/kernels/pyspark/` (create the directory if not present) and create a file `kernel.json` containing,
~~~~
{
    "display_name": "pySpark (Spark 2.3.1)",
    "language": "python",
    "argv": [
        "/anaconda2/bin/python",
        "-m",
        "ipykernel",
        "-f",
        "{connection_file}"
    ],
    "env": {
        "CAPTURE_STANDARD_OUT": "true",
        "CAPTURE_STANDARD_ERR": "true",
        "SEND_EMPTY_OUTPUT": "false",
        "SPARK_HOME": "/usr/local/bin/spark",
        "PYTHONPATH": "/usr/local/bin/spark/python/lib/py4j-0.10.7.src.zip",
        "PYTHONSTARTUP": "/usr/local/bin/spark/python/pyspark/python/pyspark/shell.py",
        "PYSPARK_SUBMIT_ARGS": "--master local[*] --num-executors 7 --executor-memory 2g --     packages com.databricks:spark-csv_2.10:1.4.0 pyspark-shell"
    }
}
~~~~

Note, you can change the first argument of `argv` inorder to use a different python (I use anaconda). You can also change the number of executors and the memory corresponding to each one. If you have followed my guide, rest of the variables should be the same. You can include additional packages that you like to loaded by default with your spark context. 

Now, whenever you create a new jupyter notebook, you can use the `pySpark (Spark 2.3.1)` kernel instead of your default python kernel. The corresponding notebook should contain the spark context under the variable `sc`.

## Important Note

If you installed Apache Hadoop and Apache Spark using this guide, your Spark will try to access hdfs by default. However, if you wish to use local disk instead of HDFS, just comment out the HADOOP_HOME and HADOOP_CONF_DIR environment variable in your ~/.bash_profile.

## Wish you a Sparkling day!
