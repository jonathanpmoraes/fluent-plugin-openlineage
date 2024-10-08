# Fluent-plugin-openlineage, a plugin for [Fluentd](https://www.fluentd.org)
[![Gem Version](https://badge.fury.io/rb/fluent-plugin-openlineage.svg)](https://badge.fury.io/rb/fluent-plugin-openlineage)

fluent-plugin-openlineage is a Fluentd plugin that verifies if a JSON matches the OpenLineage schema. 
It is intended to be used together with a [Fluentd Application](https://github.com/fluent/fluentd).

## Requirements

| fluent-plugin-prometheus | fluentd    | ruby   |
|--------------------------|------------|--------|
| 1.x.y                    | >= v1.9.1  | >= 2.4 |
| 1.[0-7].y                | >= v0.14.8 | >= 2.1 |
| 0.x.y                    | >= v0.12.0 | >= 1.9 |

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'fluent-plugin-openlineage'
```

And then execute:

    $ bundle

Or install it yourself using one of the following:

    $ gem install fluent-plugin-openlineage

    $ fluent-gem install fluent-plugin-openlineage

## Usage

fluentd-plugin-openlineage include only one plugin.

- `openlineage` parse plugin

## Why are Fluentd and Openlineage a perfect match? 
### This is part of the OpenLineage Project repository at: https://github.com/OpenLineage/OpenLineage/tree/main/proxy/fluentd

<<<<<<< HEAD
Modern data collectors (Fluentd, Logstash, Vector, etc.) can be extremely useful when designing
production-grade architectures for processing Openlineage events.

They can be used for features such as:
* A server-proxy in front of the Openlineage backend (like Marquez) to handle load spikes and buffer incoming events when the backend is down (e.g., due to a maintenance window).
* The ability to copy the event to multiple backends such as HTTP, Kafka or cloud object storage. Data collectors implement that out-of-the-box.

They have great potential except for a single missing feature: *the ability to parse and validate OpenLineage events at the point of HTTP input*.
This is important as one would like to get a `Bad Request` response immediately when sending invalid OpenLineage events to an endpoint.
Fortunately, this missing feature can be implemented as a plugin.

We decided to implement an OpenLineage parser plugin for Fluentd because:
* Fluentd has a small footprint in terms of resource utilization and does not require that JVM be installed,
* Fluentd plugins can be installed from local files (no need to register in a plugin repository).

As a side effect, the Fluentd integration can be also used as a OpenLineage HTTP validation backend for
development purposes.

## Fluentd features

Some interesting Fluentd features are available according to the [official documentation](https://docs.fluentd.org/):

* [Buffering/retrying parameters](https://docs.fluentd.org/output#buffering-retrying-parameters),
* Useful output plugins:
   * [Output Kafka plugin](https://docs.fluentd.org/output/kafka),
   * [Output S3 plugin](https://docs.fluentd.org/output/s3),
   * [Output copy plugin](https://docs.fluentd.org/output/copy),
   * [Output HTTP plugin](https://docs.fluentd.org/output/http) with options such as [retryable_response_codes](https://docs.fluentd.org/output/http#retryable_response_codes) to specify backend codes that should cause a retry,
* [Buffer configuration](https://docs.fluentd.org/configuration/buffer-section),
* [Embedding Ruby Expressions in config files to contain environment variables](https://docs.fluentd.org/configuration/config-file#embedding-ruby-expressions).

The official Fluentd documentation does not mention guarantees about event ordering. However, retrieving
Openlineage events and buffering in file/memory should be considered a millisecond-long operation,
while any HTTP backend cannot guarantee ordering in such a case. On the other hand, by default
the amount of threads to flush the buffer is set to 1 and configurable ([flush_thread_count](https://docs.fluentd.org/output#flush_thread_count)).

## Quickstart with Docker

Please refer to the [`Dockerfile`](docker/Dockerfile) and [`fluent.conf`](docker/conf/fluent.conf) to see how to build and install the plugin with
the example usage scenario provided in [`docker-compose.yml`](docker/docker-compose.yml). To run the example setup, go to the `docker` directory and execute the following command:

```shell
docker-compose up
```

After all the containers have started, send some HTTP requests:

```shell
curl -X POST \
-d '{"test":"test"}' \
-H 'Content-Type: application/json' \
http://localhost:9880/api/v1/lineage
```
In response, you should see the following message:

`Openlineage validation failed: path "/": "run" is a required property, path "/": "job" is a required property, path "/": "eventTime" is a required property, path "/": "producer" is a required property, path "/": "schemaURL" is a required property`

Next, send some valid requests:

```shell
curl -X POST \
-d "$(cat test-start.json)" \
-H 'Content-Type: application/json' \
http://localhost:9880/api/v1/lineage
```

```shell
curl -X POST \
-d "$(cat test-complete.json)" \
-H 'Content-Type: application/json' \
http://localhost:9880/api/v1/lineage
```

After that you should see entities in Marquez (http://localhost:3000/) in the `my-namespace` namespace.

To clean up, run
```shell
docker-compose down
```

### Configuration

Although Openlineage event is specified according to Json-Schema, its real-life validation may
vary and backends like Marquez may have less strict approach to validating certain types of facets. 
For example, Marquez allows a non-valid `DataQualityMetricsInputDatasetFacet`. 
To give more flexibility, fluentd parser allows following configuration parameters:
```ruby
validate_input_dataset_facets => true/false
validate_output_dataset_facets => true/false
validate_dataset_facets => true/false
validate_run_facets => true/false
validate_job_facets =>  true/false
```
By default, only `validate_run_facets` and `validate_job_facets` are set to `true`/

### Development

To build dependencies:
```shell
bundle install
bundle
```

To run the tests:
```shell
bundle exec rake test
```

#### Installation

The easiest way to install the plugin is to install the main application Fluentd and along with it, these external packages for example in a Dockerfile:
* `rusty_json_schema` installs a JSON validation library for Rust,
* `fluent-plugin-out-http` allows non-bulk HTTP out requests (sending each OpenLineage event in a separate request).
```shell
fluent-gem install rusty_json_schema
fluent-gem install fluent-plugin-out-http
fluent-gem install fluent-plugin-openlineage
```

## Fluentd proxy setup
### Monitoring with Prometheus and some other configs you can try by running a separate fluent.conf file

You may also want to check how Fluentd application itself is doing using Prometheus and for that, you may want to add the plugin: fluent-plugin-prometheus at https://github.com/fluent/fluent-plugin-prometheus and include the following setup in your prometheus.yml file:

```yml
global:
  scrape_interval: 10s # Set the scrape interval to every 10 seconds. Default is every 1 minute.

#### A scrape configuration containing exactly one endpoint to scrape:
#### Here it's Prometheus itself.
scrape_configs:
  - job_name: 'fluentd'
    static_configs:
      - targets: ['localhost:24231']
````

You may also want to include the following additional parameters to your fluent.conf file:

```xml
#### source
<source>
  @type forward
  bind 0.0.0.0
  port 24224
</source>
#### count the number of incoming records per tag
<filter company.*>
  @type prometheus
  <metric>
    name fluentd_input_status_num_records_total
    type counter
    desc The total number of incoming records
    <labels>
      tag ${tag}
      hostname ${hostname}
    </labels>
  </metric>
</filter>
#### count the number of outgoing records per tag
<match company.*>
  @type copy
  <store>
    @type forward
    <server>
      name myserver1
      host 192.168.1.3
      port 24224
      weight 60
    </server>
  </store>
  <store>
    @type prometheus
    <metric>
      name fluentd_output_status_num_records_total
      type counter
      desc The total number of outgoing records
      <labels>
        tag ${tag}
        hostname ${hostname}
      </labels>
    </metric>
  </store>
</match>
#### expose metrics in prometheus format
<source>
  @type prometheus
  bind 0.0.0.0
  port 24231
  metrics_path /metrics
</source>
<source>
  @type prometheus_output_monitor
  interval 10
  <labels>
    hostname ${hostname}
  </labels>
</source>
```
### You can check the docker file for some other examples related to configurations

For any additional information, you can check out Fluentd official documentation on https://docs.fluentd.org/monitoring-fluentd/monitoring-prometheus#example-prometheus-queries# fluentd-openlineage-parser
