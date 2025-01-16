---
title: "How to filter app and infra logs in Openshift's ClusterLogForwarder"
meta_title: ""
description: "this is meta description"
date: 2025-01-16T05:00:00Z
image: "/images/post-4/logo-openshift.png"
categories: ["Openshift"]
author: "Nicholas"
tags: ["Openshift", "Logging"]
draft: false
summary: "This article will explore how we managed to filter application and infrastructure type logs in a Cluster Log Forwarder (Openshift's Cluster Logging operator)."
---

## Introduction

For a specific use case, we wanted to filter `application` and `infrastructure` logs that were transitting through the Cluster Log Forwarder that matched a specific pattern before redirecting it to a specific Kafka topic.

For version 5.8 and before of Cluster Logging, only the `audit` type log can be filtered. 

Starting from Cluster Logging 5.9, you can filter `application` and `infrastructure` logs only if you're using the `Vector` collector.

<hr>

## Prerequisites

1. Update your Cluster Logging to at least 5.9
2. Your collector must be `Vector` (the deprecated `Fluentd` can't handle the filtering of `application` and `infrastructure` logs)

{{< notice "info" >}}
Source on prerequisites : https://access.redhat.com/solutions/6972274
{{< /notice >}}

<hr>

## Context

We already had a forwarding that was in place. We were already redirecting all of our `application` and `infrastructure` logs to a Kafka topic in remote.

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: kafka-main
      kafka:
        brokers:
          - 'tls://<kafka>:9092/'
        topic: oc-main-logs
      type: kafka
  pipelines:
    - name: main-pipeline
      inputRefs:
        - application
        - infrastructure
      outputRefs:
        - default
        - kafka-main
```

As we can see, the pipeline retrieves `application` and `infrastructure` logs and redirects it to the output specified earlier `kafka-main`, and `default`. The `default` is the `logStore` instance specified in your Cluster Logging, either Elasticsearch or Loki.

<hr>

## Filtering

Now, we wanted to add a new pipeline to filter specific logs. Those logs are from specific pods, and also contains a stringified JSON that starts with `{\"header\"` and terminates with `\"}}`.

So we added a new pipeline to do it with a new filter step that checks that if the given log does not match our pods name or the pattern, you drop it. You can filter by logs by [metadata](https://docs.openshift.com/container-platform/4.14/observability/logging/performance_reliability/logging-input-spec-filtering.html) or [content](https://docs.openshift.com/container-platform/4.14/observability/logging/performance_reliability/logging-content-filtering.html). 


```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: kafka-main
      kafka:
        brokers:
          - 'tls://<kafka>:9092/'
        topic: oc-main-logs
      type: kafka
    - name: kafka-filtered
      kafka:
        brokers:
          - 'tls://<kafka>:9092/'
        topic: oc-filtered-logs
      type: kafka
  filters:
    - name: filter_logs
      type: drop 
      drop: 
      - test: 
        - field: .message 
          notMatches: \{\"header\".*?\"}}
        - field: .kubernetes.pod_name
          notMatches: custom-logging-app-.*
  pipelines:
    - name: main-pipeline
      inputRefs:
        - application
        - infrastructure
      outputRefs:
        - default
        - kafka-main
    - name: filtered-pipeline
      inputRefs:
        - application
        - infrastructure
      filterRefs: 
        - filter_logs
      outputRefs:
        - kafka-filtered
```

I can now check assert on my collector pods or on my kafka instance (using [Kafbat UI for example]({{< ref "/blog/posts/install-kafbat-ui-on-openshift.md" >}})) that logs are now filtered and sent.