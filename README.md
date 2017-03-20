# Spark Cluster using Docker

This readme will guide you through the creation and setup of a 3 node spark cluster using Docker containers, share the same data volume to use as the script source, how to run a script using spark-submit and how to create a container to schedule spark jobs.

### Install docker

Install the appropriate docker version for your operating system. Use the following link to do so.
* [https://www.docker.com/products/docker](https://www.docker.com/products/docker)

### Build the images

In case you don't want to build your own images get it from my docker hub repository.

Create a directory where you'll copy this repo (or create your own following the same structure as here).

Go to the spark-datastore directory where there's the datastore's dockerfile container.

### Build the datastore image
```
docker build -t brunocf/spark-datastore
```
Do the previous step for all the four directories (dockerfiles).

Run the following command to check the images have been created:
```
docker images
```

### Creating and starting the containers

Once you have all the images created it's time to start them up.

##### Create the datastore container

We first create the datastore container so all the other container can use the datastore container's data volume.

```
docker create -v /data --name spark-datastore brunocf/spark-datastore
```
####  Create the spark master container
```
docker run -d -p 8080:8080 -p 7077:7077 --volumes-from spark-datastore --name master brunocf/spark-master
```

#### Create the spark slaves (workers) container

You can create as many workers containers as you want to.
```
docker run -d --link master:master --volumes-from spark-datastore brunocf/spark-slave
```
> The link option allows the container automatically connect to the other (master in this case) by being added to the same network.

At this time you should have your spark cluster running on docker and waiting to run spark code.
To check the containers are running simply run `docker ps` or `docker ps -a` to view even the datastore created container.

### Running a spark code using spark-submit

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

#### Run the spark-submit container
```
docker run --rm -it --link master:master --volumes-from spark-datastore brunocf/spark-submit spark-submit --master spark://172.17.0.2:7077 /data/script.py
```

### Schedule a Spark job

In order to schedule a spark job, we'll still use the same shared data volume to store the spark python script, the crontab file where we first add all the cron schedule and the script called by the cron where we set the environment variables since cron jobs does not have all the environment preset when run.

The spark-cron job runs a simple script that copies all the crontab file scheduled jobs to the /etc/cron.d directory where the cron service actually gets the schedule to run and start the cron service in foreground mode so the container does not exit.
* Ensure this script is added to your shared data volume (in this case /data).

#### start.sh script
```
cp /data/crontab /etc/cron.d/spark-cron
cron -f
```

Now let's create the main_spark.sh script that should be called by the cron service.
Here all the environment variables required to run spark-submit are set and the spark job called.

#### main_spark.sh script
```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
export SPARK_HOME=/home/spark/spark-1.6.2-bin-hadoop2.6
export PYSPARK_PYTHON=python2.7
export PATH=$PATH:$SPARK_HOME/bin
spark-submit --master spark://172.17.0.2:7077 /data/script.py
```

#### Add the schedule entry to the crontab file

Add the script call entry to the crontab file specifying the time you want it to run. In this example the script is scheduled to run everyday at 23:50.
```
50 23 * * * root sh /data/main_spark.sh
```

#### Run the spark-cron docker container
```
docker run  -d --link master:master --volumes-from spark-datastore brunocf/spark-cron
```


# Set up the Spark Cluster in Docker using Bluemix

Bluemix offers Docker containers so you don't have to use your own infrastructure
I'll walk you through the container creation within Bluemix which slightly differ from the normal one we did previoously.

### Create Bluemix account and install the required software in your machine to handle docker container

First go to [www.bluemix.net](http://www.bluemix.net) and follow the steps presented there to create your account.

Now access the [Install IBM Plug-in to work with Docker](https://console.ng.bluemix.net/docs/containers/container_cli_cfic_install.html) link and follow all the instructions in order to have your local environment ready.

### Create the directories and Dockerfiles

Create the same directories for each one of the images as mentioned above and save each Dockerfile within their respective directory (make sure the Dockerfile file name starts with capital D and has no extensions).

### Build the images in Bluemix

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

### Create Bluemix shared volume

In Bluemix you create volumes separetaly and then attach them to the container during its creation.

For this example, I'll create one volume called "data" and then use it shared among all my containers, this is where scripts and crontab files are stored.

Create the volume using the command line (you can also create it using the Bluemix graphic interface)
```
cf ic volume create data
```
To check if it was created use `cf ic volume list`.

### Create and Start the containers

Now it's time to actually start all your container. You can start as many spark-slave nodes as you want too.

If you need detailes on how to use each one of the containers read the previous sections where all the details are described.

#### Starting spark-master
```
cf ic run -d -p 8080:8080 -p 7077:7077 --volume data:/data --name master brunocf_containers/spark-master
```
#### Requesting and binding public IP to the container
Since we may need to check the Spark graphic interface to check nodes and running jobs, you should add a public IP to the container so it's accessible via web browser.

To do so, first list your running containers (by the time there should be only one running container).
Command to list running containers `cf ic ps`

Get the "Container ID" for your spark-master running container, you'll need that to bind the public IP.

First request an IP address (if you are runnning trial, you have 2 free public IPs. And then bind it to your container using the container ID retrieved previously.
```
cf ic ip request

cf ic ip bind <IP adress> <container ID>
```

#### Create spark-slave container
```
cf ic run -d --link master:master --volume data:/data registry.ng.bluemix.net/brunocf_containers/spark-slave
```

#### Create spark-submit container
```
cf ic run --link master:master --volume data:/data registry.ng.bluemix.net/brunocf_containers/spark-submit sh -c "spark-submit --master spark://master:7077 /data/script.py"
```

### Create spark-cron container
```
cf ic run -d --link master:master --volume data:/data registry.ng.bluemix.net/brunocf_containers/spark-cron
```

### Useful Tips

* In Bluemix, if you need access a running container you could do this with the following comand: `cf ic exec -it <container ID> bash`. You'll be redirected to the bash command line within the container so you can create and manage data within your volume and change the container settings.
* All the data added to your volume is persisted even if you stop or remove the container
* If you need to change the container timezone in order to run cron jobs accordingly simply copy the timezone file located on `/usr/share/zoneinfo/<country>/<other options>` to `/etc/localtime`.
Follow a Dockerfile example using the Brazil East timezone:
```
#Spark Cron
FROM ubuntu

RUN apt-get update && apt-get install -y \
 wget \
 default-jdk \
 python2.7 \
 vim \
 cron \
 scala && cd /home && mkdir spark && cd spark && \
 wget http://ftp.unicamp.br/pub/apache/spark/spark-1.6.2/spark-1.6.2-bin-hadoop2.6.tgz && \
 tar -xvf spark-1.6.2-bin-hadoop2.6.tgz && service cron stop && cp /usr/share/zoneinfo/Brazil/East /etc/localtime

RUN service cron start


ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-amd64
ENV SPARK_HOME /home/spark/spark-1.6.2-bin-hadoop2.6
ENV PYSPARK_PYTHON python2.7
ENV PATH $PATH:$SPARK_HOME/bin

ENTRYPOINT /data/start.sh
```

* For any other Docker on Bluemix questions and setup guides refer to [https://console.ng.bluemix.net/docs/containers/container_creating_ov.html](https://console.ng.bluemix.net/docs/containers/container_creating_ov.html)
