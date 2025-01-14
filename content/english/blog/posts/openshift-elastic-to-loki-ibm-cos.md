---
title: "Migration of Openshift's ClusterLogging operator from Elastic to Loki using an IBM Cloud Object Storage"
meta_title: ""
description: "this is meta description"
date: 2024-12-27T05:00:00Z
image: "/images/post-1/logo_loki.png"
categories: ["Openshift", "Logging", "IBM Cloud"]
author: "Nicholas"
tags: ["Openshift", "Elastic", "Lokistack", "Logging", "IBM Cloud"]
draft: false
summary: "In this article I'll explain step by step on: how to use IBM Cloud Object S3 API for Loki's ingester and compactor, how to migrate from Elastic to Loki, cleanup old ressources and check it's working fine."
---

## Introduction

I had to do a migration of an Openshift cluster Logging operator from EFK (Elastic, Fluentd and Kibana) stack that are now tagged as deprecated to Lokistack and Vector.

A lot of articles/blogs are available for the Fluentd to Vector so I skipped it in this tutorial.

In our specific case we had to make Loki work with a IBM Cloud Cloud Object Storage. As I didn't find any info on this part I decided to write this post.

Hope it helps !

<hr>

## Step 0 - Prerequisites

* User who will apply this procedure needs to be an administrator
* Have `yq` utilitary (used in the backup step, but you can clean up the yml manually instead) (optionnal)
* Have `aws` CLI if you want to precheck the S3 API of your Object Storage using the HMAC Credential (optionnal)

<hr>

## Step 1 - Create the COS, HMAC credentials and test them

For context, Loki’s [ingester](https://grafana.com/docs/loki/latest/get-started/components/#ingester) and [compactor](https://grafana.com/docs/loki/latest/get-started/components/#compactor) requires an Object Storage. In our case we had to use an [IBM Cloud Object Storage](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-about-cloud-object-storage), but it offers a S3 API available to use after those next steps: 

1.	Create an IBM Cloud Object Storage in the same region as your ROKS (in our test we used `eu-de`)
2.	Create a bucket in your COS (keep the `bucket name` somewhere as it’ll be used later in step 2) (in our test we used smart tiering)
3.	Create a Service credentials at the COS level.

{{< image src="images/post-1/service-credential.png" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}
{{< image src="images/post-1/create-service-cred.png" caption="" alt="alter-text" height="" width="" position="center" command="fill" option="q100" class="border rounded img-fluid" title="image title"  webp="false" >}}

{{< notice "info" >}}
You can check this documentation if you need help setting up your HMAC credentials: https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-uhc-hmac-credentials-main
{{< /notice >}}

{{< notice "warning" >}}
Do not forget to specify Write role and to include HMAC Credential.
{{< /notice >}}

```json
{
    "apikey": "<API_KEY>",
    "cos_hmac_keys": {
        "access_key_id": "<ACCESS_KEY_ID>",
        "secret_access_key": "<SECRET_ACCESS_KEY>"
    },
    "endpoints": "https://control.cloud-object-storage.cloud.ibm.com/v2/endpoints",
    "iam_apikey_description": "Auto-generated for key ...",
    "iam_apikey_id": "ApiKey-47700220-f7af-4ae9-aeb5-cd51f421e65c",
    "iam_apikey_name": "<API_KEY_NAME>",
    "iam_role_crn": "crn:v1:bluemix:public:iam::::serviceRole:Writer",
    "iam_serviceid_crn": "crn:v1:bluemix:public:iam-identity::...",
    "resource_instance_id": "crn:v1:bluemix:public:cloud-object-storage:global:..."
}
```

4.	Note somewhere the `access_key_id` and `secret_access_key` value that will be used later in the secret creation in step 2.

5.	You’ll also need the `s3 endpoint`, this URL depends on your bucket region. In the service credentials you’ll find a link in the `endpoints` key. Open this link in your browser/curl/etc. Retrieve the URL that matches your region. In our test, we used : `https://s3.eu-de.cloud-object-storage.appdomain.cloud`.

6.	The last parameter is the `region`. For the S3 API consumption you’ll need to find the storage class as specified in this [documentation](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-aws-cli#aws-cli-config)
Depending on your bucket region and type (standard, smart, cold, etc), retrieve the LocationConstraint following this [documentation](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-classes#classes-locationconstraint)
For our test, we used this one : `eu-de-smart`

7. If you want to test your HMAC credential and it’s access to the bucket you can use AWS CLI ([source](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-aws-cli))

```shell
$ aws configure 
AWS Access Key ID []: <access_key_id>
AWS Secret Access Key []: <secret_access_key>
Default region name []: <region>
Default output format [json]: json

$ aws --endpoint-url https://<endpoint>  s3 ls s3://<bucket>
```

<hr>

## Step 2 - Install LokiStack only

1. Install Loki Operator
{{< image src="images/post-1/loki-operator-install.png" caption="" alt="alter-text" height="" width="" position="left" option="q100"  title="image title" class="border rounded img-fluid" webp="false" >}}
2. Create a secret that will contains the Object Storage info. You can modify the secret name (if you do, you’ll need to modify it also in the next step)
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: secret-loki-storage-s3
  namespace: openshift-logging
data:
  access_key_id: <your_ access_key_id>
  access_key_secret: <your_ access_key_secret>
  bucketnames: <your_ bucketnames>
  endpoint: https://<your_ endpoint>
  region: <your_ region>
type: Opaque
```
3. Create a LokiStack instance: 
```yaml
kind: LokiStack
apiVersion: loki.grafana.com/v1
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  managementState: Managed
  size: 1x.extra-small
  storage:
    schemas:
      - effectiveDate: '2022-06-01'
        version: v13
    secret:
      name: secret-loki-storage-s3
      type: s3
  storageClassName: ibmc-vpc-block-general-purpose
  tenants:
    mode: openshift-logging
```

{{< notice "info" >}}
You can modify the size parameter depending on your need (you can update the size after the creation, in our test we passed from 1x.extra-small to 1x. small)
{{< /notice >}}

{{< image src="images/post-1/size-parameter-lokistack.png" caption="" alt="alter-text" height="" width="" position="left" option="q100"  title="image title" class="border rounded img-fluid" webp="false" >}}

<hr>

## Step 3 - Disconnect Elasticsearch and Kibana CRs from ClusterLogging

1. Temporarily set ClusterLogging to State Unmanaged Raw:
```shell
oc -n openshift-logging patch clusterlogging/instance -p '{"spec":{"managementState": "Unmanaged"}}' --type=merge
```
2. Remove ClusterLogging OwnerReferences from Elasticsearch resource:
```shell
oc -n openshift-logging patch elasticsearch/elasticsearch -p '{"metadata":{"ownerReferences": []}}' --type=merge
```
3. Remove ClusterLogging OwnerReferences from Kibana resource:
```shell
oc -n openshift-logging patch kibana/kibana -p '{"metadata":{"ownerReferences": []}}' --type=merge
```
4. Backup Elasticsearch and Kibana resources:
```shell
oc -n openshift-logging get elasticsearch elasticsearch -o yaml | yq -r 'del(.status, .metadata.resourceVersion,.metadata.uid,.metadata.generation,.metadata.creationTimestamp,.metadata.selfLink)'  > /tmp/cr-elasticsearch.yaml
```
```shell
oc -n openshift-logging get kibana kibana -o yaml | yq -r 'del(.status,.metadata.resourceVersion,.metadata.uid,.metadata.generation,.metadata.creationTimestamp,.metadata.selfLink)' > /tmp/cr-kibana.yaml
```

<hr>

## Step 4 - Switch ClusterLogging to LokiStack

1. Switch log storage to LokiStack:
```shell
cat << EOF |oc replace -f -
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"
spec:
  managementState: "Managed"
  logStore:
    type: "lokistack"
    lokistack:
      name: logging-loki
  collection:
    type: vector
EOF
```
2. Enable the console view plugin:
```shell
oc patch consoles.operator.openshift.io cluster  --type=merge --patch '{ "spec": { "plugins": ["logging-view-plugin"] } }'
```

3. You should be able to see the logs directly in the Openshift UI:
{{< image src="images/post-1/observe-logs-menu.png" caption="" alt="alter-text" height="" width="" position="left" option="q100"  title="image title" class="border rounded img-fluid" webp="false" >}}

<hr>

## Step 5 - Delete the ELK stack

When the retention period for the log stored in the Elasticsearch logstore is **expired and no more logs are visible** in the Kibana instance is it possible to remove the old stack to release resources.

1. Delete Elasticsearch and Kibana resources:
```shell
oc -n openshift-logging delete kibana/kibana elasticsearch/elasticsearch
```

2. Delete the PVCs used by the Elasticsearch instances:
```shell
oc delete -n openshift-logging pvc -l logging-cluster=elasticsearch
```

<hr>

## Step 6 - Check Loki’s output on your COS

**After a minimum of 2 hours**, you can check that the `ingester` and `compactor` are working correctly. 
```text
“waiting 2h0m0s for ring to stay stable and previous compactions to finish before starting compactor”
```
You should find some compacted files, e.g:
{{< image src="images/post-1/loki-logs-in-cos.png" caption="" alt="alter-text" height="" width="" position="left" option="q100"  title="image title" class="border rounded img-fluid" webp="false" >}}

