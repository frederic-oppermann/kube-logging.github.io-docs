---
title: Transport all logs into Amazon S3  with Logging operator
linktitle: Amazon S3 with Fluentd
weight: 200
---

<p align="center"><img src="../../img/s3_logo.png" width="340"></p>

This guide describes how to collect all the container logs in Kubernetes using the Logging operator, and how to send them to Amazon S3.

{{< include-headless "quickstart-figure-intro.md" >}}

<p align="center"><img src="../../img/s3_flow.png" width="900"></p>

## Deploy the Logging operator

Install the Logging operator.

### Deploy the Logging operator with Helm {#helm}

{{< include-headless "deploy-helm-intro.md" >}}

1. Add the chart repository of the Logging operator using the following commands:

    ```bash
    helm repo add kube-logging https://kube-logging.dev/helm-charts
    helm repo update
    ```

1. Install the Logging operator into the *logging* namespace:

    ```bash
    helm upgrade --install --wait --create-namespace --namespace logging logging-operator kube-logging/logging-operator
    ```

1. [Validate your deployment](#validate).

## Configure the Logging operator

1. Create AWS `secret`

    > If you have your `$AWS_ACCESS_KEY_ID` and `$AWS_SECRET_ACCESS_KEY` set you can use the following snippet.

    ```bash
    kubectl -n logging create secret generic logging-s3 --from-literal "awsAccessKeyId=$AWS_ACCESS_KEY_ID" --from-literal "awsSecretAccessKey=$AWS_SECRET_ACCESS_KEY"
    ```

    Or set up the secret manually.

    ```bash
        kubectl -n logging apply -f - <<"EOF"
        apiVersion: v1
        kind: Secret
        metadata:
          name: logging-s3
        type: Opaque
        data:
          awsAccessKeyId: <base64encoded>
          awsSecretAccessKey: <base64encoded>
        EOF
    ```

1. Create the `logging` resource.

     ```bash
     kubectl -n logging apply -f - <<"EOF"
     apiVersion: logging.banzaicloud.io/v1beta1
     kind: Logging
     metadata:
       name: default-logging-simple
     spec:
       fluentd: {}
       fluentbit: {}
       controlNamespace: logging
     EOF
     ```

     > Note: You can use the `ClusterOutput` and `ClusterFlow` resources only in the `controlNamespace`.

1. Create an S3 `output` definition.

     ```bash
    kubectl -n logging apply -f - <<"EOF"
    apiVersion: logging.banzaicloud.io/v1beta1
    kind: Output
    metadata:
      name: s3-output
      namespace: logging
    spec:
      s3:
        aws_key_id:
          valueFrom:
            secretKeyRef:
              name: logging-s3
              key: awsAccessKeyId
        aws_sec_key:
          valueFrom:
            secretKeyRef:
              name: logging-s3
              key: awsSecretAccessKey
        s3_bucket: logging-amazon-s3
        s3_region: eu-central-1
        path: logs/${tag}/%Y/%m/%d/
        buffer:
          timekey: 10m
          timekey_wait: 30s
          timekey_use_utc: true
    EOF
     ```

     > Note: In production environment, use a longer `timekey` interval to avoid generating too many objects.

1. Create a `flow` resource. (Mind the label selector in the `match` that selects a set of pods that we will install in the next step)

     ```bash
     kubectl -n logging apply -f - <<"EOF"
     apiVersion: logging.banzaicloud.io/v1beta1
     kind: Flow
     metadata:
       name: s3-flow
     spec:
       filters:
         - tag_normaliser: {}
       match:
         - select:
             labels:
               app.kubernetes.io/name: log-generator
       localOutputRefs:
         - s3-output
     EOF
     ```

1. Install log-generator to produce logs with the label `app.kubernetes.io/name: log-generator`

     ```bash
     helm upgrade --install --wait --create-namespace --namespace logging log-generator kube-logging/log-generator
     ```

1. [Validate your deployment](#validate).

## Validate the deployment {#validate}

Check fluentd logs (errors with AWS credentials should be visible here):
```bash
kubectl exec -ti -n logging default-logging-simple-fluentd-0 -- tail -f /fluentd/log/out
```

Check the output. The logs will be available in the bucket on a `path` like:
```bash
/logs/default.default-logging-simple-fluentbit-lsdp5.fluent-bit/2019/09/11/201909111432_0.gz
```

{{< include-headless "note-troubleshooting.md" >}}
