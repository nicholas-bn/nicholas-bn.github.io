---
title: "Quick install of a Kafka instance on an IBM Cloud VSI"
meta_title: ""
description: "this is meta description"
date: 2025-01-06T05:00:00Z
image: "/images/post-2/logo-kafka-install.png"
categories: ["Kafka", "IBM Cloud"]
author: "Nicholas"
tags: ["Kafka", "VSI", "IBM Cloud"]
draft: false
summary: "In this article I'll show how I installed quickly a Kafka instance I used to test my ClusterLogForwarder config in Openshift."
---

## Introduction

I had to reproduce and modify a [ClusterLogForwarder](https://docs.openshift.com/container-platform/4.14/observability/logging/log_collection_forwarding/configuring-log-forwarding.html) configuration to send logs to a third party Kafka. 

As I wanted to test it on multiple different topics, modify the config rapidly if necessary and be able to delete/create as I wanted, I chose to install Kafka in a VSI, here's how I did it.

This example is applied to IBM Cloud, but it could be done on every public cloud provider.

<hr>

## Step 0 - Prerequisites

1. Create a [VPC](https://cloud.ibm.com/docs/vpc?topic=vpc-about-vpc) on your account
2. Create a [Public Gateway](https://cloud.ibm.com/docs/vpc?topic=vpc-about-public-gateways) on the subnet where you'll provision a VSI

<hr>

## Step 1 - Provision a Virtual Server Instance 

{{< notice "info" >}}
I'm doing those steps for a VSI on a VPC (Virtual Private Cloud) but you could do the same on a [Virtual Server for Classic](https://cloud.ibm.com/docs/virtual-servers?topic=virtual-servers-getting-started-tutorial)
{{< /notice >}}

1. Create a [Virtual Server Instance](https://cloud.ibm.com/docs/vpc?topic=vpc-about-advanced-virtual-servers). In our case we used an Ubuntu 24 image.
2. Create and assign a [Floating IP](https://cloud.ibm.com/docs/vpc?topic=vpc-fip-about) 
{{< image src="images/post-2/floating-ip/step-1.png" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}
{{< image src="images/post-2/floating-ip/step-2.png" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}
{{< image src="images/post-2/floating-ip/step-3.png" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}
3. Update your [Access Control List (ACL)](https://cloud.ibm.com/docs/vpc?topic=vpc-using-acls) and [Security Groups (SG)](https://cloud.ibm.com/docs/vpc?topic=vpc-using-security-groups) so you can SSH on the VSI and access the ports required for Kafka: `22`, `9092` and `9093`.

<hr>

## Step 2 - Install and launch Kafka

{{< notice "note" >}}
Source : https://kafka.apache.org/quickstart under the `Using downloaded files` part
{{< /notice >}}
1. SSH on your VSI with the SSH key you've setup while provisionning it.
2. Download Kafka TAR, un-TAR it and go in it :
```shell
$ wget https://dlcdn.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
$ tar -xzf kafka_2.13-3.9.0.tgz
$ cd kafka_2.13-3.9.0
```
3. Modify the configuration file to expose the VSI IP to the listeners : `config/kraft/reconfig-server.properties`
```cfg
advertised.listeners=PLAINTEXT://<VSI_IP>:9092,CONTROLLER://<VSI_IP>:9093
```
4. You can modify other configs, for example we changed the default retention time : 
```cfg
log.retention.hours=12
```
5. Create a cluster UUID, format the log directories and start your instance :
```shell
$ KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
$ bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/kraft/reconfig-server.properties
$ bin/kafka-server-start.sh config/kraft/reconfig-server.properties
```
6. Create your topic :
```shell
$ bin/kafka-topics.sh --create --topic your-topic --bootstrap-server localhost:9092
```

<hr>

## Step 3 - Check you can reach your kafka from an external source

You have multiple possibilities, you can directly hit it with a Kafka consumer library in your app. 
For standalone tests, you can use the Kafka TAR scripts :
```shell
$ bin/kafka-console-consumer.sh --topic your-topic --from-beginning --bootstrap-server '<VSI_IP>:9092'
```

Or you can use tools like [Kafbat UI](https://github.com/kafbat/kafka-ui) (ex Kafka UI) deployed locally or on your Openshift like we did in this [article]({{< ref "/blog/posts/install-kafbat-ui-on-openshift.md" >}}). 
{{< image src="images/post-2/kafbat-ui/kafbat-ui.jpg" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}
