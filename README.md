# Spark Cluster using Docker

This readme will guide you through the creation and setup of a 3 node spark cluster using Docker containers, share the same data volume to use as the script source and how to run a script using spark-submit.

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
conf = SparkConf().setAppName("myTestApp")
sc = SparkContext(conf=conf)

words = sc.textFile("/data/test.txt")
words.saveAsTextFile("/data/test2.txt")
```

####Run the spark-submit container
```
docker run --rm -it --link master:master --volumes-from spark-datastore brunocf/spark-submit spark-submit --master spark://172.17.0.2:7077 /data/script.py
```
