# CloudTrail-Searcher-Setup
Many administrators are facing an auditing problem because they can't search events older than 1 week in AWS CloudTrail console. Good new is that all logs are available in S3 bucket. But problem is **How to search in it?** I have an simple solution to this problem, [Elasticsearch](https://www.elastic.co/products/elasticsearch) can be used here as search engine to the job.

## Why Elasticsearch?
- Elasticsearch consumes json document as an input and CloudTrail logs saved in S3 are in json format. So **no conversions**.
- It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents.

## Setup Overview
- We are going to use Docker containers to setup and running Elasticsearch because of  universality. 
- Then we are going to download Cloudtrail logs from S3 to local machine.
- Then will unzip and feed it to Elasticsearch.

## Installing prerequisite : Docker and AWS cli
- Install Docker on your OS using [Docker official guide](https://docs.docker.com/engine/installation/).
- Install AWS cli following [AWS official guide](http://docs.aws.amazon.com/cli/latest/userguide/installing.html).
- And start both programs.

## Making Elasticsearch container 
As we want to use [head plugin](https://github.com/mobz/elasticsearch-head) (that will provides web gui), we have to include it in Elasticsearch docker image. Here we are making a [Dockerfile](https://docs.docker.com/engine/reference/builder/) to build a custom image [from official Elasticsearch docker image](https://hub.docker.com/_/elasticsearch/) and then installing head plugin in that image.
```sh
# mkdir ~/es-head
# cd ~/es-head
# cat > Dockerfile <<EOF
>FROM elasticsearch:2.3
>
>RUN /usr/share/elasticsearch/bin/plugin install mobz/elasticsearch-head
>EOF
# 
```
Now build a Docker image named “elasticsearch-head” from above Dockerfile(make sure that this directory should not contain anything other than Dockerfile).
```sh
# ls 
Dockerfile
# docker build -t elasticsearch-head:1 .
```
Below command is use to create a [data only container](https://docs.docker.com/engine/tutorials/dockervolumes/#/creating-and-mounting-a-data-volume-container) from above custom image that will print "DATA only ES" and then exits. We will use this container indirectly to store data of our actual service to /es_data/DATA on host.
```sh
# mkdir -p /es_data/DATA
# docker run --name es-cloudtrail_DATA -v "/es_data/DATA":/usr/share/elasticsearch/data elasticsearch:2.3 echo "DATA only ES"
DATA only ES
#
```
From [Docker run](https://docs.docker.com/v1.8/reference/commandline/run/), discription of above command.
```
Usage: docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
--name=""                     Assign a name to the container
-v, --volume=[]               Bind mount a volume
```
Finally we are actually launching Elasticsearch container that will be used for searching CloudTrail logs.
```sh 
# docker run -d --volumes-from es-cloudtrail_DATA -p 9200:9200 -p 9300:9300 --name es-cloudtrail  elasticsearch-head:1 -Des.node.name="es-cloudtrail"
```
```
Usage: docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
-d, --detach=false            Run container in background and print container ID
--name=""                     Assign a name to the container
--volumes-from=[]             Mount volumes from the specified container(s)
-p, --publish=[]              Publish a container's port(s) to the host
```
## Downloading CloudTrail logs from S3
- Create an IAM user with read only access to s3.
- Configure AWS cli using [document](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) for above created IAM user.
```sh
# aws configure --profile cloudtrail
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: ENTER
```
- Now we are downloading all CloudTail logs from S3 using below command, make sure that enough space is available.
```sh
# mkdir /es_data/traillogs
# aws s3 --profile cloudtrail cp s3://s3-traillogs-bucket-name/AWSLogs /es_data/traillogs/ --recursive
```
> Inplace of "s3-traillogs-bucket-name", use your actual s3 bucket where traillogs are saved.
## Sending logs to Elasticsearch
- Unzip all downloaded cloudtrail logs.
```sh
# gunzip -rv /es_data/traillogs
```
We can use many things to send logs to Elasticsearch but simplest solution is to use [HTTP PUT/POST](https://www.elastic.co/guide/en/elasticsearch/guide/current/index-doc.html) request as we have documents already in json format.

So now we are going to make a simple script that will do this jobs.
- First we will make a list of all traillogs file names with their full path. This is as simle is
```sh
# find /es_data/traillogs -name '*.json' -type f > /es_data/logs.list
```
- Now we will prefix a HTTP POST request for elasticsearch to  each line of above created logs.list file. This will make a POST request for each traillogs to elasticsearch.
```sh
# sed 's/^/curl -XPOST http://localhost:9200/AWS-account-name/old -d @/' /es_data/logs.list > /es_data/logs.sh
```
> here we are making a HTTP POST request to Elasticsearch on port 9200 with [Index and type](https://www.elastic.co/guide/en/elasticsearch/guide/current/index-doc.html#_autogenerating_ids) as "AWS-account-name"(your AWS account name to identify easily) and "old"(useful in future).

- Now make it executable and run.
```sh
# chmod u+x /es_data/logs.sh
# nohup bash /es_data/logs.sh &
```
- For space management, gzip again all downloaded traillogs.
```sh
# gzip -rv /es_data/traillogs
```
**Now setup is complete and we are ready to search.**
## Searching
Now elasticsearch can be accessed by [below URL](http://localhost:9200/_plugin/head).
```
http://localhost:9200/_plugin/head
```
Follow [Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/search-in-depth.html) and [head plugin](https://mobz.github.io/elasticsearch-head/) for more on searching.
## Future
