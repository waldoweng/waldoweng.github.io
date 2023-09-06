---
title: "Catla Development Log (2023/09/03) - Thoughts"
date: 2023-09-03T21:06:19+08:00
draft: false
---

Long have I wanted to create a Time Series Database of my own, and from scratch.

After working with *VictoriaMetrics* and *Prometheus* for almost 3 year, I got the feeling that although both systems (namely *Prometheus* and *VictoriaMetrics*) works pretty effective in production, there is still room for refinement for both systems.

*Prometheus*, as a CNCF project, adopted by kubernetes monitoring, and with a very healthy community support such as OpenMetrics and OpenTelemetry and many exporters enables users to monitor almost everything they wants, however, it provides no scalability as *Prometheus* can only run as a single instance, not as a cluster of instance.  *Prometheus* did provides a mechanism call *remote write / remote read* for third-party storage to integrate with *Prometheus*. In this scheme, data was scrape by *Prometheus* as the single instance setup and send to the third-party storage which, hopefully, can scale horizontally, and then pull back to *Prometheus*'s query engine in case of query. And here is where *VictoriaMetrics* comes into play.

*VictoriaMetrics* is a challenger for *Prometheus* from my point of view. *VictoriaMetrics* is rather radical by extending a new query language call *MetricsQL* from *PromQL* but not compatible with *PromQL*, and from a user's view, both the resource *VictoriaMetrics* require and the respond time for queries are rather low compare to *Thanos*, which base on *Prometheus*. But in production, I do observe sometimes instability, and, as a rapidly developing open source project,  *VictoriaMetrics* seems knows not quit well about the working situation in the production environment. As a multi-tenant system, the requirement for stability more dire, because the failure domain is expended as system failure or even performance degradation in a short period of time can effect multiple group of users, Furthermore, a multi-tenant system should also try its best to isolate each tenant by introducing *Quota* and *Qos mechanism* for both data *Ingestion* and *Query*, and support a ease migration for data of one tenant from one cluster to another cluster.

These are what *Catla* will try to accomplish. *Catla* will be my ***personal*** side project, inspired by *Prometheus* and *VictoriaMetrics*, and will be more radical than *VictoriaMetrics*. The purpose is to explore more possibilities of different approach at both architectural level and implementation level, and broaden my perspective in Time Series Database domain beside *Prometheus* and *VictoriaMetrics*. Hopefully this will be a long journey, of exploration and idea, I'm just so excited about it.


#### Architectural Design

*VictoriaMetrics* has two versions: single instance version and cluster version, and they're build from different branch of one source code repo. I think this way increase the development and maintenance effort need. So for *Catla*, I shall support single instance setup and cluster setup by using different command, invoke the executable with no subcommand setup a fully functional single instance, when added with other components which setup by invoking the executable with different subcommands, *Catla* shall support a cluster configuration. while in cluster setup, each instance will still keep their API accessible, in this way I hope less effort will be needed to for a user to troubleshoot issues with one data partition.

*Catla* will have the following subcommands:

1. ***catla***: with no subcommands setup a fully functional single instance.
2. ***catla-ingest***: setup a proxy to accept data and than divide the data stream to each storage group.
3. ***catla-query***: setup a query engine to handle queries issues from the user.
4. ***catla-store***: setup a storage to time series data, and support horizontal scalability.
5. ***catla-index***: setup a indexer to maintain some global index, and support horizontal scalability.
6. ***catla-meta***: set up a coordinator to the whole cluster.

There should also be some standalone executables:

1. ***catla-scrape***: scraper.
2. ***catla-rule***: executes alert rules, recording rule feature will not be implement.

Quite similar with *VictoriaMetrics*, *Catla* only:

1. split ***vmstorage*** into ***catla-store*** and ***catla-index***, for new time series ingestion have long haunt me for performance degradation. Decoupling index and data allows users to scale the system for different churn rate, data point ingestion rate, and inverted index traves rate.
2. ***catla-meta*** is added to maintain the global cluster configuration, as a metadata server.

For Qos, *Catla* shall use a variation of *MClock* algorithm to provide isolation of resource.

For *recording rules*, *Catla* has no plan to implement such feature. In my opinion, a *Statd*-like mechanism should serve well in reducing the number of series involved in one query, and hence reduce the time require for ***catla-query*** to process it, and this mechanism should be implement inside the scraper.

#### Interface Design

*Catla* should also support the Prometheus *remote write protocol* like ***vminsert*** did, but since I always wonder how *Prometheus* came up with the notorious *staleness marker* mechanism, I think I can just have data send to ***catla-ingest*** to have a life span, which is the scrape interval. In this way *Catla* should not need the incomprehensible *staleness marker* mechanism.

```protobuf
message WriteRequest {
  repeated TimeSeries timeseries = 1;
  // Cortex uses this field to determine the source of the write request.
  // We reserve it to avoid any compatibility issues.
  reserved  2;

  // Prometheus uses this field to send metadata, but this is
  // omitted from v1 of the spec as it is experimental.
  reserved  3;
}

message TimeSeries {
  repeated Label labels   = 1;
  repeated Sample samples = 2;
}

message Label {
  string name  = 1;
  string value = 2;
}

message Sample {
  double value    = 1;
  int64 timestamp = 2;
  
  // added by catla: life span in milliseconds.
  // which should equal to the scrape interval.
  optional int64 ttl = 3;
}
```
the above specification is base on: https://prometheus.io/docs/concepts/remote_write_spec/

*Catla* should support the following *HTTP APIs*:

1. ***/api/v1/query*** : the same as *Prometheus* and *VictoriaMetrics*.
2. ***/api/v1/query_range*** : the same as *Prometheus* and *VictoriaMetrics*.
3. ***/api/v1/labels*** : the same as *Prometheus* and *VictoriaMetrics*.
4. ***/api/v1/label_values*** : the same as *Prometheus* and *VictoriaMetrics*.
5. ***/api/v1/series*** : the same as *Prometheus* and *VictoriaMetrics*.
6. ***/api/v1/tsdb_status*** : the same as *Prometheus* and *VictoriaMetrics*.
7. ***/api/v1/metric_status*** : the status of a single metrics, should include cardinality information, for troubleshooting.
8. ***/api/v1/status*** : the status of a single instance.


#### Other Design Decisions

*Catla* should be built with ***Rust***. In my opinion, fighting *Garbage Collector* is simply wasting time when building data-intensive application such as databases. I would rather rely on *RTTI*-like technique.

*Catla* should also be built with observability in mind. *Catla* will also expose Prometheus metrics with the */metrics* HTTP endpoint and integrated with tracing systems such as *Jaeger*.