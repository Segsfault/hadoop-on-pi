# hadoop-on-pi
This repo contains ansible scripts and configurations to setup a cluster of raspberry 
pis to run hadoop and hive.  Why?  As a way to learn and teach about how big data tech works under the hood.
For production workloads, there are better platforms to choose from.

Why this repo? There are plenty of guides that walk through installing Hadoop on a cluster of pi's, but none of
them automate the process.  When standing up my own version, I found that changes in software or OS versions 
required changes in how hadoop was configured.  In other words, those guides are outdated.  Using ansible to handle
configuration management significantly cut down the iteration time.  As a benefit, the complete install record is
captured, which means when this guide becomes outdated, it should be easier to pick it up and modify things to get your
cluster stood up with whatever the latest versions of the software are.

#### Prerequisites
This guide assumes you have at least 2 pis available, that they are accessible via 
SSH on your local network, and they have static IPs assigned to them.  Each pi should also have
a unique hostname.
This repo was
developed using the November 2018 [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/) 
version of Linux.

Apologies in advance to Windows users. This guide was written assuming you have a unix based operating system 
as your development environment (Macs work fine).

### Getting Started: 
Install [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).  You should know
the IPs of your nodes.  Pick one to act as a master node. The rest will be worker nodes.  Ansible needs a way to
differentiate between them, so you'll want to make your first action be setting up a config file. Since I have just
this one application for now, I'm setting the global config in `/etc/ansible/hosts` as follows:

    [master]
    10.10.1.100

    [workers]
    10.10.1.101
    10.10.1.102
    10.10.1.103
    10.10.1.104
    
Replace with your IP addresses.
Strongly recommend ensuring you can ssh into each of your nodes without a password. See the 
[Ansible docs](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html) for more information.
In the ansible commands below, you may need to use the `-u` option to select the remote user (`pi` by default).

### Preparing the cluster nodes
Modify the `00-prepare-nodes.yml` file to reflect the IPs and hostnames in your cluster.  This script will ensure
that the hostnames are known by all nodes.  It also removes a secondary loopback entry that the OS includes by
default.

This single line was the single largest time sink for me when standing up the cluster. Its presence prevented worker
nodes from talking to the master namenode in the hadoop deployment.

Run the script using `ansible-playbook 00-prepare-nodes.yml`

### Create and configure an account dedicated to using hadoop
The scripts in this repo are currently hard coded to create and use the `hduser` account for running anything related
to hadoop.  Ansible requires that the password we use is presented to it in hashed format.  There are a few ways to
convert a password to a hash, but you can leverage one of your pis to run the following:

    sudo apt-get install -y whois
    mkpasswd --method=sha-512
    
When prompted, enter a password.  The command will return the hash as a string that you can cut and paste into
the `hadoop_user_password` variable in the `01-create-hadoop-account.yml` script.

Finally, run the script using `ansible-playbook 01-create-hadoop-account.yml`

### Install Java
While there is a version you can install via `apt-get`, I decided to go straight to 
[Oracle](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) based on reading some
of the other guides.  Not clear to me which is better, but I had success with the download from Oracle.  Since
Rasbian is a 32bit OS (as Pis have 32bit hardware), I selected the jdk-8u191-linux-arm32-vfp-hflt.tar.gz version.

This should be placed within the `installers` directory of the repo, since Ansible is going to look here for it.

Running `ansible-playbook 02-install-java-yml` will take care of moving the tarball to each node, unpacking it, and
configuring the bashrc for the hduser so that the appropriate environment variables are set.

### Installing hadoop
I used the latest version (3.1.1) available from the [Hadoop Download page](https://hadoop.apache.org/releases.html).
As with the java tarball, place this in the `installers` directory.

Before installing, we need to set some parameters for the hadoop configuration. You'll find these in the `hadoop_config`
directory.

In `core-site.xml`, look for the line with `<value>hdfs://master:9000</value>` and replace `master` with the hostname
of your master node.

In `hadoop-env.sh`, add a line with the following content: `export JAVA_HOME=/opt/jdk1.8.0_191/jre`

In `hdfs-site.xml`, look for the line `<value>3</value>`. That number should be no larger than the number of Pis in
your cluster.

In `mapred-site.xml`, there are three lines that reference `/opt/hadoop-3.1.1`. If you used a different version or
install directory, modify accordingly.

In `workers` change the lines to reflect the hostnames of you master and worker nodes.

In `yarn-site.xml`, replace the value of the hostname for the `yarn.resourcemanager.hostname` property with the
hostname of your master.

Execute `ansible-playbook 03-install-hadoop.yml`

### Installing Hive
Head over to the [Hive Download Site](https://hive.apache.org/downloads.html) and fetch the latest binary release.
In my case, I used 3.1.1.  Plop this into the `installers` directory.

Run `ansible-playbook 04-install-hive.yml`

## First time setup and testing
Login as the `hduser` on the master node.
* Format the namenode using `hdfs namenode -format`
* Start up the hadoop file system using `start-dfs.sh`
* Run a report on the system: `hdfs dfsadmin -report`

You should see output like this:

    2019-01-02 13:08:34,681 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    Configured Capacity: 314999418880 (293.37 GB)
    Present Capacity: 289826250752 (269.92 GB)
    DFS Remaining: 289337090048 (269.47 GB)
    DFS Used: 489160704 (466.50 MB)
    DFS Used%: 0.17%
    Replicated Blocks:
        Under replicated blocks: 0
        Blocks with corrupt replicas: 0
        Missing blocks: 0
        Missing blocks (with replication factor 1): 0
        Pending deletion blocks: 0
    Erasure Coded Block Groups:
        Low redundancy block groups: 0
        Block groups with corrupt internal blocks: 0
        Missing block groups: 0
        Pending deletion blocks: 0
    
    -------------------------------------------------
    Live datanodes (5):
    
Followed by summary information for each of the nodes in the cluster.

* Start the resource manager with `start-yarn.sh`

* Prepare hive by running the following commands

      hadoop fs -mkdir /tmp
      hadoop fs -mkdir -p /user/hive/warehouse
      hadoop fs -chmod g+w   /tmp
      hadoop fs -chmod g+w   /user/hive/warehouse
      schematool -initSchema -dbType derby
      
      
## Try it out!
We'll try some data processing with first hadoop and then hive.

### Hadoop wordcount example

Head over to [Project Gutenberg](https://www.gutenberg.org/) and download some ebooks in text format.  About 15 or 20 
should do. Combine them together to build a large text file called `books.txt`. Copy that over to your master node.

On the master, move the file to HDFS using `hdfs dfs -copyFromLocal books.txt /books.txt`

Now run the wordcount demo on it using `hadoop jar /opt/hadoop-3.1.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.1.jar wordcount /books.txt /books-result`

That will likely take around 3-4 minutes to complete. When done, check out the results using `hdfs dfs -cat /books-result/part-r* | head -n 20`

### Hive query example.
We'll use a simple table from the US department of Social Security that collects First names registered each year
for children born in the US.  Grab the 
[State Specific](https://www.ssa.gov/OACT/babynames/limits.html) data set, unpack it, and then combine all the state
by state data together into a single file called `usa_names.csv`. Get this into HDFS.

Run `hive` on the master.  When you get the CLI prompt, enter the following commands:

    CREATE DATABASE IF NOT EXISTS sandbox;
    USE sandbox;
    CREATE TABLE usa_names (state STRING, gender STRING, birth_year INT, name STRING, frequency INT) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS textfile;
    LOAD DATA INPATH '/usa_names.csv' OVERWRITE INTO table usa_names;
    SELECT birth_year,sum(frequency) FROM usa_names GROUP BY birth_year ORDER BY birth_year;
    
The query will return the total number of registrations in the per year.
