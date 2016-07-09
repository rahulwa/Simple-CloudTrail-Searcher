# **CloudTrail-Searcher-Setup**
Many administrators are facing an auditing problem because they can't search events older than 1 week in AWS CloudTrail console. Good new is that all logs are available in S3 bucket. But problem is **How to in it?** I have an simple solution to this problem, [Elasticsearch](https://www.elastic.co/products/elasticsearch) can be used here as search engine to the job.

## Why Elasticsearch?
- Elasticsearch consumes json document as an input and CloudTrail logs saved in S3 are in json format. So **no conversions**.
- It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents.

## Setup Overview
- We are going to use Docker containers to setup and running Elasticsearch because of  universality. 
- Then we are going to download Cloudtrail logs from S3 to local machine.
- Then will unzip and feed it to Elasticsearch.

