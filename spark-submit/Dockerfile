#Spark Submit Shell
FROM ubuntu

RUN apt-get update && apt-get install -y \
 wget \
 default-jdk \
 python2.7 \
 scala && cd /home && mkdir spark && cd spark && \
 wget http://ftp.unicamp.br/pub/apache/spark/spark-1.6.2/spark-1.6.2-bin-hadoop2.6.tgz && \
 tar -xvf spark-1.6.2-bin-hadoop2.6.tgz

ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-amd64
ENV SPARK_HOME /home/spark/spark-1.6.2-bin-hadoop2.6
ENV PYSPARK_PYTHON python2.7
ENV PATH $PATH:$SPARK_HOME/bin
