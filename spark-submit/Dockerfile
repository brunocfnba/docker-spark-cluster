#Spark Submit Shell
FROM ubuntu:16.04

RUN apt-get update && apt-get install -y \
 wget \
 default-jdk \
 python2.7 \
 scala && cd /home && mkdir spark && cd spark && \
 wget https://archive.apache.org/dist/spark/spark-2.1.1/spark-2.1.1-bin-hadoop2.6.tgz && \
 tar -xvf spark-2.1.1-bin-hadoop2.6.tgz

ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-amd64
ENV SPARK_HOME /home/spark/spark-2.1.1-bin-hadoop2.6
ENV PYSPARK_PYTHON python2.7
ENV PATH $PATH:$SPARK_HOME/bin
