# Prometheus ObjectServer Exporter [![Build Status](https://travis-ci.org/free/sql_exporter.svg)](https://travis-ci.org/free/sql_exporter) [![Go Report Card](https://goreportcard.com/badge/github.com/free/sql_exporter)](https://goreportcard.com/report/github.com/free/sql_exporter) [![GoDoc](https://godoc.org/github.com/free/sql_exporter?status.svg)](https://godoc.org/github.com/free/sql_exporter) [![Docker Pulls](https://img.shields.io/docker/pulls/githubfree/sql_exporter.svg?maxAge=604800)](https://hub.docker.com/r/githubfree/sql_exporter)

Object exporter for [Prometheus](https://prometheus.io).

## Overview

ObjServ Exporter is a configuration driven exporter that exposes metrics gathered from Netcool's ObjectServer, for use by the Prometheus
monitoring system.

The collected metrics and the queries that produce them are entirely configuration defined. SQL queries are grouped into
collectors -- logical groups of queries, e.g. *query stats* or *I/O stats*, mapped to the metrics they populate.
Collectors may be DBMS-specific (e.g. *MySQL InnoDB stats*) or custom, deployment specific (e.g. *pricing data
freshness*). This means you can quickly and easily set up custom collectors to measure data quality, whatever that might
mean in your specific case.

Per the Prometheus philosophy, scrapes are synchronous (metrics are collected on every `/metrics` poll) but, in order to
keep load at reasonable levels, minimum collection intervals may optionally be set per collector, producing cached
metrics when queried more frequently than the configured interval.

### Install dependencines

[FreeTDS](http://www.freetds.org/) libraries must be installed on the system.

Mac
```shell
brew install freetds
```
Ubuntu, Debian...
```shell
sudo apt-get install freetds-dev
```

## Usage

Get Prometheus Objserv_Exporter, either as a [packaged release](bitbucket)
build it yourself:

```
$ go install github.com/free/objserv_exporter/cmd/objserv_exporter
```

then run it from the command line:

```
$ sql_exporter
```

Use the `-help` flag to get help information.

```
$ ./objserv_exporter -help
Usage of ./objserv_exporter:
  -config.file string
      ObjectServer Exporter configuration file name. (default "objserv_exporter.yml")
  -web.listen-address string
      Address to listen on for web interface and telemetry. (default ":9399")
  -web.metrics-path string
      Path under which to expose metrics. (default "/metrics")
  [...]
```

## Configuration

ObjectServer Exporter is deployed alongside the OS server it collects metrics from. If both the exporter and the DB
server are on the same host, they will share the same failure domain: they will usually be either both up and running
or both down. When the database is unreachable, `/metrics` responds with HTTP code 500 Internal Server Error, causing
Prometheus to record `up=0` for that scrape. Only metrics defined by collectors are exported on the `/metrics` endpoint.
ObjectServer Exporter process metrics are exported at `/objserv_exporter_metrics`.

The configuration examples listed here only cover the core elements. For a comprehensive and comprehensively documented
configuration file check out 
[`documentation/sql_exporter.yml`](https://github.com/free/sql_exporter/tree/master/documentation/sql_exporter.yml).
You will find ready to use "standard" DBMS-specific collector definitions in the
[`examples`](https://github.com/free/sql_exporter/tree/master/examples) directory. You may contribute your own collector
definitions and metric additions if you think they could be more widely useful, even if they are merely different takes
on already covered DBMSs.

**`./objserv_exporter.yml`**

```yaml
# Global settings and defaults.
global:
  # Subtracted from Prometheus' scrape_timeout to give us some headroom and prevent Prometheus from
  # timing out first.
  scrape_timeout_offset: 500ms
  # Minimum interval between collector runs: by default (0s) collectors are executed on every scrape.
  min_interval: 0s
  # Maximum number of open connections to any one target. Metric queries will run concurrently on
  # multiple connections.
  max_connections: 3
  # Maximum number of idle connections to any one target.
  max_idle_connections: 3

# The target to monitor and the list of collectors to execute on it.
target:
  # Data source name always has a URI schema that matches the driver name. In some cases (e.g. MySQL)
  # the schema gets dropped or replaced to match the driver expected DSN format.
  data_source_name: 'mssql://Server=TBSM;user=netcool_reader;password=xxxxx;compatibilty=openserver;'

  # Collectors (referenced by name) to execute on the target.
  collectors: [objectserver_alerts]

# Collector definition files.
collector_files: 
  - "*.collector.yml"
```

### Collectors

Collectors may be defined inline, in the exporter configuration file, under `collectors`, or they may be defined in
separate files and referenced in the exporter configuration by name, making them easy to share and reuse.

The collector definition below generates gauge metrics of the form `pricing_update_time{market="US"}`.

**`./objectserver_alerts.collector.yml`**

```yaml
# This collector will be referenced in the exporter configuration as `pricing_data_freshness`.
collector_name: objectserver_alerts

# A Prometheus metric with (optional) additional labels, value and labels populated from one query.
metrics:
  - metric_name: pricing_update_time
    type: gauge
    help: 'Time when prices for a market were last updated.'
    key_labels:
      # Populated from the `market` column of each row.
      - result
    static_labels:
      # Arbitrary key/value pair
      alertname: test_hostxxx
    values: [num, max_severity]
    query: |
      SELECT count(*), Max(Severity)
      FROM status
      WHERE Identifer='xxxxx'
```

### Data Source Names

mssql://Server=DSN_NAME(from freetds.conf);user=user;password=passw;compatibility=openserver

## Why It Exists
Objserv_exporter is a fork of sql_exporter with gofreetds database/sql driver.

SQL Exporter started off as an exporter for Microsoft SQL Server, for which no reliable exporters exist. But what is
the point of a configuration driven SQL exporter, if you're going to use it along with 2 more exporters with wholly
different world views and configurations, because you also have MySQL and PostgreSQL instances to monitor?

A couple of alternative database agnostic exporters are available -- https://github.com/justwatchcom/sql_exporter and
https://github.com/chop-dbhi/prometheus-sql -- but they both do the collection at fixed intervals, independent of
Prometheus scrapes. This is partly a philosophical issue, but practical issues are not all that difficult to imagine:
jitter; duplicate data points; or collected but not scraped data points. The control they provide over which labels get
applied is limited, and the base label set spammy. And finally, configurations are not easily reused without
copy-pasting and editing across jobs and instances.


