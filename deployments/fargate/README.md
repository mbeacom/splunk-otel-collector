# AWS Fargate Deployment
Familiarity with AWS Fargate (Fargate) is assumed. Consult the 
[User Guide for AWS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html) for further reading.

Unless stated otherwise, it is assumed that the [Splunk OpenTelemetry Collector](https://github.com/signalfx/splunk-otel-collector)
(Collector) is deployed to an ECS Task as **sidecar** container alongside monitored applications.

Applies to Collector release >= v0.30.0. See the Docker image repository
[here](https://quay.io/repository/signalfx/splunk-otel-collector?tab=tags).


## Default Configuration
The default configuration file is located at `/etc/otel/collector/fargate_config.yaml`
in the Collector image. See 
[here](https://github.com/signalfx/splunk-otel-collector/blob/main/cmd/otelcol/config/collector/fargate_config.yaml)
for its contents and 
[here](https://github.com/signalfx/splunk-otel-collector/blob/main/cmd/otelcol/Dockerfile)
for the Dockerfile.

You configure the Collector to use the default configuration file by assigning its path
to environment variable `SPLUNK_CONFIG` in the container definition.
You also need to map the endpoint ports in the default configuration file
(i.e. `13133`, `6060`, `55679`, `14250`, `6832`, `6831`, `14268`, `8888`, `7276`,
`9943`, `9411`, `9080`). 

The default metrics receivers are `hostmetrics` and `smartagent/ecs-metadata`.
Assign a list of metrics you want to exclude environment variable `METRICS_TO_EXCLUDE`.
Assign a list of images whose metrics you want excluded to environment variable
`IMAGES_TO_EXCLUDE`.

## Custom Configuration
In container definition for the Collector, assign the path to your custom configuration
file to environment variable `SPLUNK_CONFIG` and the Collector will pick up the custom
configuration. In Fargate, this means uploading the custom configuration file to a volume
attached to the task.

### ecsobserver
Configure the Collector to use extension
[Amazon Elastic Container Service Observer](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/observer/ecsobserver#amazon-elastic-container-service-observer)
(ecsobserver or ecs_observer) to discover application metrics targets. The `ecs_observer`
can discover targets in running tasks, filtered by service names, task definitions and
container labels. It is currently limited to discovering Prometheus targets and requires
the read-only permissions below. Add the permissions to the task role by adding them to
a customer-managed policy that is attached to the task role.
```text
ecs:List*
ecs:Describe*
```

In the example below, the `ecs_observer` is configured to find Prometheus targets in
cluster `lorem-ipsum-cluster`, region `us-west-2`, where the task arn pattern is 
`^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:[0-9]+$`,
and write the results in file `/etc/ecs_sd_targets.yaml`. Receiver `prometheus` is
configured to read targets from the results file. Note that the `access_token` and `realm`
are read from environment variables `SPLUNK_ACCESS_TOKEN` and `SPLUNK_REALM` so these
environment variables must also be specified in the container definition.

```yaml
extensions:
  ecs_observer:
    refresh_interval: 10s
    cluster_name: 'lorem-ipsum-cluster'
    cluster_region: 'us-west-2'
    result_file: '/etc/ecs_sd_targets.yaml'
    task_definitions:
      - arn_pattern: "^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:[0-9]+$"
        metrics_ports: [9113]
        metrics_path: /metrics
receivers:
  signalfx:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'lorem-ipsum-nginx'
          scrape_interval: 10s
          file_sd_configs:
            - files:
                - '/etc/ecs_sd_targets.yaml'
processors:
  batch:
  resourcedetection:
    detectors: [ecs]
    override: false    
exporters:
  signalfx:
    access_token: ${SPLUNK_ACCESS_TOKEN}
    realm: ${SPLUNK_REALM}
service:
  extensions: [ecs_observer]
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: [batch, resourcedetection]
      exporters: [signalfx]
```
In a sidecar deployment the discovered targets must be within the task in which the Collector
container is running. Note that the task arn pattern in the configuration example above 
restricts the `ecs_observer` to discover targets in running **revision**(s) of task `lorem-ipsum-task`.
This means that the `ecs_observer` will discover targets outside the task in which the Collector is
running when multiple revisions of task `lorem-ipsum-task` are running. One way
to solve this is to use the complete task arn as shown below. This however adds the headache of
updating the configuration to keep pace with task revisions.

```yaml
...
     - arn_pattern: "^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:3$"
...
```

### Direct Configuration
Since access to the filesystem is not readily available in Fargate, it is convenient to
use environment variable `SPLUNK_CONFIG_YAML` which allows you can specify 
the configuration YAML directly at the commandline. You can simply create a parameter
in the AWS Systems Manager (Amazon SSM) Parameter Store and add your custom configuration
YAML as its value. Then in the container definition for the Collector, specify
environment variable `SPLUNK_CONFIG_YAML` and have it get its value from the parameter.
You need to add access permissions to Amazon SSM to the task role, and you can do it by
attaching policy `AmazonSSMReadOnlyAccess` to task role.


### Standalone Container
The `ecs_observer` allows for the Collector to run in a standalone container in a separate
task. You simply add the container definition for the Collector to a task that is separate
from the task containing the monitored application.