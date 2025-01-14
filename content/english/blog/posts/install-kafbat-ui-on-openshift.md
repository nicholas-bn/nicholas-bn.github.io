---
title: "Install Kafbat UI on Openshift"
meta_title: ""
description: "this is meta description"
date: 2025-01-14T06:00:00Z
image: "/images/post-3/kafbat-ui-logo.png"
categories: ["Kafka", "Openshift", "Kafbat"]
author: "Nicholas"
tags: ["Kafka", "Kafbat UI", "Openshift", "Kafka UI"]
draft: false
summary: "In this article I'll show how to do a minimal Kafbat UI (also known as Kafka UI) install on an Openshift cluster."
---

## Introduction

This article will provide a deployment, service and route to run easily and rapidly a [Kafbat UI](https://github.com/kafbat/kafka-ui) on your Openshift cluster.

**Kafbat UI is a free, open-source web UI to monitor and manage Apache Kafka clusters.**

Kafbat UI is a simple tool that makes your data flows observable, helps find and troubleshoot issues faster and deliver optimal performance. Its lightweight dashboard makes it easy to track key metrics of your Kafka clusters - Brokers, Topics, Partitions, Production, and Consumption.

<hr>

## Step 0 - Prerequisites

* Have an Openshift cluster with enough priviledges
* Have access to the `kafbat/kafka-ui` image from your Openshift

<hr>

## Step 1 - Create a deployment 


{{< notice "info" >}}
Please note the `DYNAMIC_CONFIG_ENABLED`. If it's set to false, you won't be able to modify your config, that means you won't be able to add clusters and modify them.
{{< /notice >}}

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: readkafka
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readkafka
  template:
    metadata:
      labels:
        app: readkafka
    spec:
      containers:
        - name: container
          image: kafbat/kafka-ui
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: DYNAMIC_CONFIG_ENABLED
              value: 'true'
```

<hr>

## Step 2 - Create a service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: readkafka-serv
  namespace: default
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  selector:
    app: readkafka
```

<hr>

## Step 3 - Create a route

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: readkafka
  namespace: default
spec:
  to:
    kind: Service
    name: readkafka-serv
    weight: 100
  port:
    targetPort: 8080
```

<hr>

## Step 4 - Use Kafbat UI

Now you can add cluster to manage. You can add/remove topics, check messages, delete them, etc. 

This tool is quite useful when testing an application/product that deals with a Kafka instance.

{{< image src="images/post-3/overview.gif" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}

