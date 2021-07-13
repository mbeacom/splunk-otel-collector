# AWS Fargate Deployment
Familiarity with AWS Fargate (Fargate) is assumed. Consult the 
[User Guide for AWS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html) for further reading.

The
[Splunk OpenTelemetry Collector](https://github.com/signalfx/splunk-otel-collector)
(splunk-collector or collector) can be deployed to an ECS Task as a standalone container 
or as sidecar container alongside the application containers that you want to monitor.

Find the Docker image repository for the collector
[here](https://quay.io/repository/signalfx/splunk-otel-collector?tab=tags).
The latest tag is `quay.io/signalfx/splunk-otel-collector:latest`.

## Standalone Container
As a standalone container, you add the container definition for the collector to a
task that is separate from the task containing the application containers that you 
want to monitor. You configure the collector to use extension
[Amazon Elastic Container Service Observer (ecsobserver)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/observer/ecsobserver#amazon-elastic-container-service-observer).
to discover targets in running tasks, filtered by service names, task definitions
and container labels. The ecsobserver is currently limited to discovering
prometheus targets. Also, the task role must include the read-only permissions below. 
The permissions can be added to a customer-managed policy that is attached to the
task role.

```text
ecs:List*
ecs:Describe*
```

Below is an example of a collector configuration that configures extension
`ecs_observer` (ecsobserver) to find prometheus targets in cluster
`splunk-fargate-cluster` in region `us-west-2` with container label
`NGINX_PROMETHEUS_EXPORTER_PORT`. `ecs_observer` then writes the results
to file `/etc/ecs_sd_targets.yaml` which receiver `prometheus` is configured 
to read.

```yaml
extensions:
  ecs_observer:
    refresh_interval: 10s
    cluster_name: 'splunk-fargate-cluster'
    cluster_region: 'us-west-2'
    result_file: '/etc/ecs_sd_targets.yaml'
    docker_labels:
      - port_label: 'NGINX_PROMETHEUS_EXPORTER_PORT'
receivers:
  signalfx:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'splunk-fargate-standalone-nginx'
          scrape_interval: 10s
          file_sd_configs:
            - files:
                - '/etc/ecs_sd_targets.yaml'
processors:
  batch:
exporters:
  signalfx:
    access_token: ${SPLUNK_ACCESS_TOKEN}
    realm: ${SPLUNK_REALM}
service:
  extensions: [ecs_observer]
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: [batch]
      exporters: [signalfx]
```

The configuration YAML can be supplied to the collector in a file whose path
is the value for flag `--config` or environment variable `SPLUNK_CONFIG`. The YAML
can also be specified directly at the commandline using environment variable 
`SPLUNK_CONFIG_YAML`. See the custom configuration section 
[here](https://github.com/signalfx/splunk-otel-collector/blob/main/docs/getting-started/linux-manual.md#custom-configuration)
for details. 

In Fargate, access to the filesystem is not readily available so using
`SPLUNK_CONFIG_YAML` is the most convenient option. You can simply create a parameter
in the AWS Systems Manager (Amazon SSM) Parameter Store and have the configuration YAML as its 
value. Then in the container definition for the collector, you specify 
environment variable `SPLUNK_CONFIG_YAML` to get its value from the parameter.
Also, grant the collector access permissions to Amazon SSM by attaching policy
`AmazonSSMReadOnlyAccess` to the task role.

Furthermore, note that the values for `access_token` and `realm` are provided via 
environment variables `SPLUNK_ACCESS_TOKEN` and `SPLUNK_REALM` so these environment
variables must be specified in the container definition.

## Sidecar Container
For the sidecar container, the collector uses the
[Amazon ECS Resource Detector Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourcedetectionprocessor/README.md)
to query the
[Amazon ECS task metadata endpoint](https://docs.aws.amazon.com/AmazonECS/latest/userguide/task-metadata-endpoint-fargate.html)
for targets in the current task in which the collector container is running.
