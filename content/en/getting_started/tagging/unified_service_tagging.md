---
title: Unified Service Tagging
kind: documentation
aliases:
- /getting_started/tagging/unified_service_tagging
further_reading:
- link: "/getting_started/tagging/using_tags"
  tag: "Documentation"
  text: "Learn how to use tags in the Datadog app"
- link: "/tracing/version_tracking"
  tag: "Documentation"
  text: "Use Version tags within Datadog APM to monitor deployments"
- link: "https://www.datadoghq.com/blog/autodiscovery-docker-monitoring/"
  tag: "Blog"
  text: "Learn more about Autodiscovery"
---

## Overview
Unified service tagging ties Datadog telemetry together through the use of three [reserved tags][1]: `env`, `service`, and `version`.

With these three tags you can:

- Identify deployment impact with trace and container metrics filtered by version
- Navigate seamlessly across traces, metrics, and logs with consistent tags
- View service data based on environment or version in a unified fashion within the Datadog site

{{< img src="tagging/unified_service_tagging/overview.mp4" alt="Unified Service Tagging" video=true >}}

**Note**: The official service of a log defaults to the container short-image if no Autodiscovery logs configuration is present. To override the official service of a log, add Autodiscovery [Docker labels/pod annotations][2]. For example: `"com.datadoghq.ad.logs"='[{"service": "service-name"}]'`

### Requirements

- Unified service tagging requires setup of a [Datadog Agent][3] that is 6.19.x/7.19.x or higher.

- Unified service tagging requires a tracer version that supports new configurations of the [reserved tags][1]. More information can be found per language in the [setup instructions][4].


| Language         | Minimum Tracer Version |
|--------------|------------|
| .NET    |  1.17.0+       |
| C++    |  1.1.4+       |
| Go         |  1.24.0+       |
| Java   |  0.50.0+      |
| Node    |  0.20.3+       |
| PHP  |  0.47.0+      |
| Python  |  0.38.0+      |
| Ruby  |  0.34.0+      |

- Unified service tagging requires knowledge of configuring tags. If you are unsure of how to configure tags, read the [Getting Started with Tagging][1] and [Assigning Tags][5] documentation before proceeding to configuration.

## Configuration

To begin configuration of unified service tagging, choose your environment:

- [Containerized](#containerized-environment)
- [Non-Containerized](#non-containerized-environment)

### Containerized environment

In containerized environments, `env`, `service`, and `version` are set through the service's environment variables or labels (for example, Kubernetes deployment and pod labels, Docker container labels). The Datadog Agent detects this tagging configuration and applies it to the data it collects from containers.

To setup unified service tagging in a containerized environment:

1. Enable [Autodiscovery][6]. This allows the Datadog Agent to automatically identify services running on a specific container and gathers data from those services to map environment variables to the `env`, `service,` and `version` tags.

2. If you are using [Docker][2], make sure the Agent can access your container's [Docker socket][7]. This allows the Agent detect the environment variables and map them to the standard tags.

4. Configure your environment based on either full configuration or partial configuration detailed below.

#### Configuration

{{< tabs >}}
{{% tab "Kubernetes" %}}

##### Full configuration

To get the full range of unified service tagging when using Kubernetes, add environment variables to both the deployment object level and the pod template spec level:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    tags.datadoghq.com/env: "<ENV>"
    tags.datadoghq.com/service: "<SERVICE>"
    tags.datadoghq.com/version: "<VERSION>"
...
template:
  metadata:
    labels:
      tags.datadoghq.com/env: "<ENV>"
      tags.datadoghq.com/service: "<SERVICE>"
      tags.datadoghq.com/version: "<VERSION>"
  containers:
  -  ...
     env:
          - name: DD_ENV
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['tags.datadoghq.com/env']
          - name: DD_SERVICE
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['tags.datadoghq.com/service']
          - name: DD_VERSION
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['tags.datadoghq.com/version']
```

##### Partial configuration

###### Pod-level metrics

To configure pod-level metrics, add the following standard labels (`tags.datadoghq.com`) to the pod spec of a Deployment, StatefulSet, or Job:

```yaml
template:
  metadata:
    labels:
      tags.datadoghq.com/env: "<ENV>"
      tags.datadoghq.com/service: "<SERVICE>"
      tags.datadoghq.com/version: "<VERSION>"
```
These labels cover pod-level Kubernetes CPU, memory, network, and disk metrics, and can be used for injecting `DD_ENV`, `DD_SERVICE`, and `DD_VERSION` into your service's container through [Kubernetes's downward API][1].

If you have multiple containers per pod, you can specify standard labels by container:

```yaml
tags.datadoghq.com/<container-name>.env
tags.datadoghq.com/<container-name>.service
tags.datadoghq.com/<container-name>.version
```

###### State metrics

To configure [Kubernetes State Metrics][2]:

1. Set `join_standard_tags` to `true` in your [configuration file][3].

2. Add the same standard labels to the collection of labels for the parent resource, for example: `Deployment`.

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      tags.datadoghq.com/env: "<ENV>"
      tags.datadoghq.com/service: "<SERVICE>"
      tags.datadoghq.com/version: "<VERSION>"
  spec:
    template:
      metadata:
        labels:
          tags.datadoghq.com/env: "<ENV>"
          tags.datadoghq.com/service: "<SERVICE>"
          tags.datadoghq.com/version: "<VERSION>"
  ```

###### APM tracer and StatsD client

To configure [APM tracer][4] and [StatsD client][5] environment variables, use the [Kubernetes's downward API][1] in the format below:

```yaml
containers:
-  ...
    env:
        - name: DD_ENV
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['tags.datadoghq.com/env']
        - name: DD_SERVICE
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['tags.datadoghq.com/service']
        - name: DD_VERSION
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['tags.datadoghq.com/version']
```

[1]: https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api
[2]: /agent/kubernetes/data_collected/#kube-state-metrics
[3]: https://github.com/DataDog/integrations-core/blob/master/kubernetes_state/datadog_checks/kubernetes_state/data/conf.yaml.example#L70
[4]: /tracing/send_traces/
[5]: /integrations/statsd/
{{% /tab %}}

{{% tab "Docker" %}}
##### Full configuration

Set the `DD_ENV`, `DD_SERVICE`, and `DD_VERSION` environment variables and corresponding Docker labels for your container to your get the full range of unified service tagging.

The values for `service` and `version` can be provided in the Dockerfile:

```yaml
ENV DD_SERVICE <SERVICE>
ENV DD_VERSION <VERSION>

LABEL com.datadoghq.tags.service="<SERVICE>"
LABEL com.datadoghq.tags.version="<VERSION>"
```

Since `env` is likely determined at deploy time, you can inject the environment variable and label later:

```shell
docker run -e DD_ENV=<ENV> -l com.datadoghq.tags.env=<ENV> ...
```

You may also prefer to set everything at deploy time:

```shell
docker run -e DD_ENV="<ENV>" \
           -e DD_SERVICE="<SERVICE>" \
           -e DD_VERSION="<VERSION>" \
           -l com.datadoghq.tags.env="<ENV>" \
           -l com.datadoghq.tags.service="<SERVICE>" \
           -l com.datadoghq.tags.version="<VERSION>" \
           ...
```

##### Partial configuration

If your service has no need for the Datadog environment variables (for example, third party software like Redis, PostgreSQL, NGINX, and applications not traced by APM) you can just use the Docker labels:

```yaml
com.datadoghq.tags.env
com.datadoghq.tags.service
com.datadoghq.tags.version
```

As explained in the full configuration, these labels can be set in a Dockerfile or as arguments for launching the container.

{{% /tab %}}

{{% tab "ECS" %}}
##### Full configuration

Set the `DD_ENV`, `DD_SERVICE`, and `DD_VERSION` environment variables and corresponding Docker labels in the runtime environment of each service's container to get the full range of unified service tagging. For instance, you can set all of this configuration in one place through your ECS task definition:

```
"environment": [
  {
    "name": "DD_ENV",
    "value": "<ENV>"
  },
  {
    "name": "DD_SERVICE",
    "value": "<SERVICE>"
  },
  {
    "name": "DD_VERSION",
    "value": "<VERSION>"
  }

"dockerLabels": {
  "com.datadoghq.tags.env": "<ENV>",
  "com.datadoghq.tags.service": "<SERVICE>",
  "com.datadoghq.tags.version": "<VERSION>"
  }
]
```

##### Partial configuration

If your service has no need for the Datadog environment variables (for example, third party software like Redis, PostgreSQL, NGINX, and applications not traced by APM) you can just use the Docker labels in your ECS task definition:

```
"dockerLabels": {
  "com.datadoghq.tags.env": "<ENV>",
  "com.datadoghq.tags.service": "<SERVICE>",
  "com.datadoghq.tags.version": "<VERSION>"
}
```

{{% /tab %}}
{{< /tabs >}}

### Non-containerized environment

Depending on how you build and deploy your services' binaries or executables, you may have several options available for setting environment variables. Since you may run one or more services per host, it is recommended that these environment variables be scoped to a single process.

To form a single point of configuration for all telemetry emitted directly from your service's runtime for [traces][8], [logs][9], and [StatsD metrics][10], you can either:

1. Export the environment variables in the command for your executable:

    `DD_ENV=<env> DD_SERVICE=<service> DD_VERSION=<version> /bin/my-service`

2. Or use [Chef][11], [Ansible][12], or another orchestration tool to populate a service's systemd or initd configuration file with the `DD` environment variables. That way when the service process is started it has access to those variables.

{{< tabs >}}
{{% tab "Traces" %}}

When configuring your traces for unified service tagging:

1. Configure the [APM Tracer][1] with `DD_ENV` to keep the definition of `env` closer to the application that is generating the traces. This method allows the `env` tag to be sourced automatically from a tag in the span metadata.

2. Configure spans with `DD_VERSION` to add version to all spans that fall under the service that belongs to the tracer (generally `DD_SERVICE`). This means that if your service creates spans with the name of an external service, those spans do not receive `version` as a tag.

    As long as version is present in spans, it is added to trace metrics generated from those spans. The version can be added manually in-code or automatically by the APM Tracer. When configured, these are used by the APM and [DogStatsD clients][2] to tag trace data and StatsD metrics with `env`, `service`, and `version`. If enabled, the APM tracer also injects the values of these variables into your logs.

    **Note**: There can only be **one service per span**. Trace metrics generally have a single service as well. However, if you have a different service defined in your hosts' tags, that configured service tag shows up on all trace metrics emitted from that host.

[1]: /tracing/setup/
[2]: /developers/dogstatsd/
{{% /tab %}}

{{% tab "Logs" %}}

If you're using [connected logs and traces][1], enable automatic logs injection if supported for your APM Tracer. Then, the APM Tracer automatically injects `env`, `service`, and `version` into your logs, therefore eliminating manual configuration for those fields elsewhere.

**Note**: The PHP Tracer does not support configuration of unified service tagging for logs.

[1]: /tracing/connect_logs_and_traces/
{{% /tab %}}

{{% tab "Custom Metrics" %}}

Tags are added in an append-only fashion for [custom statsd metrics][1]. For example, if you have two different values for `env`, the metrics are tagged with both environments. There is no order in which one tag overrides another of the same name.

If your service has access to `DD_ENV`, `DD_SERVICE`, and `DD_VERSION`, then the DogStatsD client automatically adds the corresponding tags to your custom metrics.

**Note**: The Datadog DogStatsD clients for .NET and PHP do not yet support this functionality.

[1]: /metrics/
{{% /tab %}}

{{% tab "System Metrics" %}}

`env` and `service` can also be added to your infrastructure metrics.

The tagging configuration for service metrics lives closer to the Agent in non-containerized contexts.
Given that this configuration does not change for each invocation of a service's process, adding `version`
to the configuration is not recommended.

##### Single service per host

Set the following configuration in the Agent's [main configuration file][1]:

```yaml
env: <ENV>
tags:
    - service:<SERVICE>
```

This setup guarantees consistent tagging of `env` and `service` for all data emitted by the Agent.

##### Multiple services per host

Set the following configuration in the Agent's [main configuration file][1]:

```yaml
env: <ENV>
```

To get unique `service` tags on CPU, memory, and disk I/O metrics at the process level, you can configure a [process check][2]:

```yaml
init_config:
instances:
    - name: web-app
      search_string: ["/bin/web-app"]
      exact_match: false
      service: web-app
    - name: nginx
      search_string: ["nginx"]
      exact_match: false
      service: nginx-web-app
```

**Note**: If you already have a `service` tag set globally in your Agent's main configuration file, the process metrics are tagged with two services. Since this can cause confusion with interpreting the metrics, it is recommended to configure the `service` tag only in the configuration of the process check.

[1]: /agent/guide/agent-configuration-files
[2]: /integrations/process
{{% /tab %}}
{{< /tabs >}}

### Serverless environment

#### AWS Lambda functions

See [how to connect your Lambda telemetry using tags][13].
## Further Reading

{{< partial name="whats-next/whats-next.html" >}}


[1]: /getting_started/tagging/
[2]: /agent/docker/integrations/?tab=docker
[3]: /getting_started/agent
[4]: /tracing/setup
[5]: /getting_started/tagging/assigning_tags?tab=noncontainerizedenvironments
[6]: /getting_started/agent/autodiscovery
[7]: /agent/docker/?tab=standard#optional-collection-agents
[8]: /getting_started/tracing/
[9]: /getting_started/logs/
[10]: /integrations/statsd/
[11]: https://www.chef.io/
[12]: https://www.ansible.com/
[13]: /serverless/configuration/#connect-telemetry-using-tags
