---
title: Logging operator
weight: 400
---

Welcome to the Logging operator documentation!

## Overview

The Logging operator solves your logging-related problems in Kubernetes environments by automating the deployment and configuration of a Kubernetes logging pipeline.

1. The operator deploys and configures a log collector (currently a Fluent Bit DaemonSet) on every node to collect container and application logs from the node file system.
1. Fluent Bit queries the Kubernetes API and enriches the logs with metadata about the pods, and transfers both the logs and the metadata to a log forwarder instance.
1. The log forwarder instance receives, filters, and transforms the incoming the logs, and transfers them to one or more destination outputs. The Logging operator supports Fluentd and syslog-ng as log forwarders.

Your logs are always transferred on authenticated and encrypted channels.

This operator helps you bundle logging information with your applications: you can describe the behavior of your application in its charts, the Logging operator does the rest.

<p align="center"><img src="img/logging_operator_flow.png" ></p>

## Feature highlights

- Namespace isolation
- Native Kubernetes label selectors
- Secure communication (TLS)
- Configuration validation
- Multiple flow support (multiply logs for different transformations)
- Multiple [output]({{< relref "/docs/configuration/plugins/outputs/_index.md" >}}) support (store the same logs in multiple storage: S3, GCS, ES, Loki and more...)
- Multiple logging system support (multiple Fluentd, Fluent Bit deployment on the same cluster)
- Support for both syslog-ng and Fluentd as the central log routing component

## Architecture

{{< include-headless "component-overview.md" >}}

For the detailed CRD documentation, see [List of CRDs]({{< relref "/docs/configuration/crds/_index.md" >}}).

![Logging operator architecture](/docs/img/logging-operator-v2-architecture.png)

## Quickstart

See our [Quickstart guides]({{< relref "/docs/quickstarts/_index.md" >}}).

## Support

If you encounter problems while using the Logging operator the documentation does not address, [open an issue](https://github.com/kube-logging/logging-operator/issues) or talk to us on [Discord](https://discord.gg/9ACY4RDsYN) or [Slack](https://emergingtechcommunity.slack.com/).
