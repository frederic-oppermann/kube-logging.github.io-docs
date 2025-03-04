---
title: Store Nginx Access Logs in Grafana Loki with Logging operator
linktitle: Grafana Loki with Fluentd
weight: 500
aliases:
    - /docs/one-eye/logging-operator/quickstarts/loki-nginx/
---

<p align="center"><img src="../../img/nll.png" width="340"></p>

This guide describes how to collect application and container logs in Kubernetes using the Logging operator, and how to send them to Grafana Loki.

{{< include-headless "quickstart-figure-intro.md" >}}

<p align="center"><img src="../../img/nginx-loki.png" width="900"></p>

## Deploy Loki and Grafana

1. Add the chart repositories of Loki and Grafana using the following commands:

    ```bash
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo add loki https://grafana.github.io/loki/charts
    helm repo update
    ```

1. Install Loki into the *logging* namespace:

    ```bash
    helm upgrade --install --create-namespace --namespace logging loki loki/loki
    ```

    > [Grafana Loki Documentation](https://github.com/grafana/loki/tree/master/production/helm)

1. Install Grafana into the *logging* namespace:

   ```bash
    helm upgrade --install --create-namespace --namespace logging grafana grafana/grafana \
    --set "datasources.datasources\\.yaml.apiVersion=1" \
    --set "datasources.datasources\\.yaml.datasources[0].name=Loki" \
    --set "datasources.datasources\\.yaml.datasources[0].type=loki" \
    --set "datasources.datasources\\.yaml.datasources[0].url=http://loki:3100" \
    --set "datasources.datasources\\.yaml.datasources[0].access=proxy"
    ```

## Deploy the Logging operator and a demo application

Install the Logging operator and a demo application to provide sample log messages.

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
1. 
2. Create the `logging` resource.

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

1. Create a Loki `output` definition.

     ```bash
    kubectl -n logging apply -f - <<"EOF"
    apiVersion: logging.banzaicloud.io/v1beta1
    kind: Output
    metadata:
      name: loki-output
    spec:
      loki:
        url: http://loki:3100
        configure_kubernetes_labels: true
        buffer:
          timekey: 1m
          timekey_wait: 30s
          timekey_use_utc: true
    EOF
     ```

     > Note: In production environment, use a longer `timekey` interval to avoid generating too many objects.

1. Create a `flow` resource.

     ```bash
     kubectl -n logging apply -f - <<"EOF"
     apiVersion: logging.banzaicloud.io/v1beta1
     kind: Flow
     metadata:
       name: loki-flow
     spec:
       filters:
         - tag_normaliser: {}
         - parser:
             remove_key_name_field: true
             reserve_data: true
             parse:
               type: nginx
       match:
         - select:
             labels:
               app.kubernetes.io/name: log-generator
       localOutputRefs:
         - loki-output
     EOF
     ```

1. Install log-generator to produce logs with the label `app.kubernetes.io/name: log-generator`

     ```bash
     helm upgrade --install --wait --create-namespace --namespace logging log-generator kube-logging/log-generator
     ```

1. [Validate your deployment](#validate).

## Validate the deployment {#validate}

### Grafana Dashboard

1. Use the following command to retrieve the password of the Grafana `admin` user:

    ```bash
    kubectl get secret --namespace logging grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

1. Enable port forwarding to the Grafana Service.

    ```bash
    kubectl -n logging port-forward svc/grafana 3000:80
    ```

1. Open the Grafana Dashboard: [http://localhost:3000](http://localhost:3000)

1. Use the `admin` username and the password retrieved in Step 1 to log in.

1. Select **Menu > Explore**, select **Data source > Loki**, then select **Log labels > namespace > logging**. A list of logs should appear.

    ![Sample log messages in Loki](../../img/loki1.png)

{{< include-headless "note-troubleshooting.md" >}}
