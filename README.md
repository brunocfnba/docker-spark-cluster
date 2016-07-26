# Spark Cluster using Docker

This readme will guide you through the creation and setup of a 3 node spark cluster using Docker containers, share the same data volume to use as the script source, how to run a script using spark-submit and how to create a container to schedule spark jobs.

###Install docker

Install the appropriate docker version for your operating system. Use the following link to do so.
* [https://www.docker.com/products/docker](https://www.docker.com/products/docker)

###Build the images

In case you don't want to build your own images get it from my docker hub repository.

Create a directory where you'll copy this repo (or create your own following the same structure as here).

Go to the spark-datastore directory where there's the datastore's dockerfile container.

###Build the datastore image
```
docker build -t brunocf/spark-datastore
```
Do the previous step for all the four directories (dockerfiles).

Run the following command to check the images have been created:
```
docker images
```

###Creating and starting the containers

Once you have all the images created it's time to start them up.

#####Create the datastore container

We first create the datastore container so all the other container can use the datastore container's data volume.

```
docker create -v /data --name spark-datastore brunocf/spark-datastore
```
####Create the spark master container
````
docker run -d -p 8080:8080 -p 7077:7077 --volumes-from spark-datastore --name master brunocf/spark-master
```

####Create the spark slaves (workers) container

You can create as many workers containers as you want to.
```
docker run -d --link master:master --volumes-from spark-datastore brunocf/spark-slave
```
> The link option allows the container automatically connect to the other (master in this case) by being added to the same network.

At this time you should have your spark cluster running on docker and waiting to run spark code.
To check the containers are running simply run `docker ps` or `docker ps -a` to view even the datastore created container.

###Running a spark code using spark-submit

Another container is created to work as the driver and call the spark cluster. The container only runs while the spark job is running, as soon as it finishes the container is deleted.

The python code I created, a simple one, should be moved to the shared volume created by the datastore container.
Since we did not specify a host volume (when we manually define where in the host machine the container is mapped) docker creates it in the default volume location located on `/var/lib/volumes/<container hash>/_data`

Follow the simple python code in case you don't want to create your own.
This script simply read data from a file that you should create in the same directory and add some lines to it and generates a new directory with its copy.

```
from pyspark import SparkContext, SparkConf

conf = SparkConf().setAppName("myTestApp")
sc = SparkContext(conf=conf)

words = sc.textFile("/data/test.txt")
words.saveAsTextFile("/data/test2.txt")
```

####Run the spark-submit container
```
docker run --rm -it --link master:master --volumes-from spark-datastore brunocf/spark-submit spark-submit --master spark://172.17.0.2:7077 /data/script.py
```

###Schedule a Spark job

In order to schedule a spark job, we'll still use the same shared data volume to store the spark python script, the crontab file where we first add all the cron schedule and the script called by the cron where we set the environment variables since cron jobs does not have all the environment preset when run.

The spark-cron job runs a simple script that copies all the crontab file scheduled jobs to the /etc/cron.d directory where the cron service actually gets the schedule to run and start the cron service in foreground mode so the container does not exit.
* Ensure this script is added to your shared data volume (in this case /data).

####start.sh script
```
cp /data/crontab /etc/cron.d/spark-cron
cron -f
```

Now let's create the main_spark.sh script that should be called by the cron service.
Here all the environment variables required to run spark-submit are set and the spark job called.

####main_spark.sh script
```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
export SPARK_HOME=/home/spark/spark-1.6.2-bin-hadoop2.6
export PYSPARK_PYTHON=python2.7
export PATH=$PATH:$SPARK_HOME/bin
spark-submit --master spark://172.17.0.2:7077 /data/script.py
```

####Add the schedule entry to the crontab file

Add the script call entry to the crontab file specifying the time you want it to run. In this example the script is scheduled to run everyday at 23:50.
```
50 23 * * * root sh /data/main_spark.sh
```

####Run the spark-cron docker container
```
docker run  -d --link master:master --volumes-from spark-datastore brunocf/spark-cron
```

#Set up the Spark Cluster in Docker using Bluemix

Bluemix offers Docker containers so you don't have to use your own infrastructure
I'll walk you through the container creation within Bluemix which slightly differ from the normal one we did previoously.

###Create Bluemix account and install the required software in your machine to handle docker container

First go to [www.bluemix.net](http://www.bluemix.net) and follow the steps presented there to create your account.

Now access the [Install IBM Plug-in to work with Docker](https://console.ng.bluemix.net/docs/containers/container_cli_cfic.html) link and follow all the instructions in order to have your local environment ready.

###Create the directories and Dockerfiles

Create the same directories for each one of the images as mentioned above and save each Dockerfile within their respective directory (make sure the Dockerfile file name starts with capital D and has no extensions).

###Build the images in Bluemix

Go to the spark-master directory and build its image directly in Bluemix. From now on all the docker commands will be "changed" to `cf ic` command that performs the same actions as docker command but in the Bluemix environment.

Build the image (remember to use your our bluemix registry you created at the beginning intead of brunocf_containers)
```
cf ic build -t brunocf_containers/spark-datastore .
```
This will start the building and pushing process and when it finishes your should be able to view the image in Bluemix or via console (within your container images area) or by command line as follows

```
cf ic images
```

Repeat the same steps for all the Dockerfiles so you have all the images created in Bluemix (you don't have to create the spark-datastore image since in Bluemix we create our volume in a different way)

