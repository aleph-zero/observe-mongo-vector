# observe-mongo-vector

Instructions on setting up ingest of self-managed [MongoDB](https://www.mongodb.com/) metrics and logs into [Observe](https://observeinc.com) (using [Vector](https://vector.dev/) agent)

## Background

Self-managed MongoDB database emits logs (by default to `/var/log/mongodb/`) and exposes a metrics endpoint (by default on `mongodb://localhost:27017`). Vector agent or another forwarder can be used to collect logs and metrics and forward them to Observe. In Observe, schema-on-demand implemented in OPAL is then applied to tranform incoming data into individual datasets, such as Mongo metrics, logs, SQL queries, etc.

## Configure vector agent to forward MongoDB logs and metrics to Observe

Here is a sample configuration of the vector agent to send data to Observe (in `/etc/vector/vector.yaml`):

```yaml
sources:
  mongo_metrics:
    type: mongodb_metrics
    endpoints:
      - mongodb://mongo-db-01:27017
  mongodb_logs:
    type: file
    include:
      - /var/log/mongodb/*.log

transforms:
  logs_transform:
    type: remap
    inputs:
      - mongodb_logs
    source: |-
      .tags.observe_env = "production"
      .tags.observe_host = "mongo-db-01"
      .tags.observe_datatype = "vector_logs"
  mongo_metrics_transform:
    type: remap
    inputs:
      - mongo_metrics
    source: |-
      .tags.observe_env = "production"
      .tags.observe_host = "mongo-db-01"
      .tags.observe_datatype = "vector_mongo_metrics"

sinks:
  observe_metrics:
    type: prometheus_remote_write
    inputs:
      - mongo_metrics_transform
    endpoint: >-
      https://{ CUSTOMER_ID }.collect.observeinc.com/v1/prometheus
    auth:
      strategy: bearer
      token: { METRICS_DATASTREAM_TOKEN }
    healthcheck: false
    request:
      retry_attempts: 5
  observe_logs:
    type: http
    inputs:
      - logs_transform
    encoding:
      codec: json
    uri: >-
      https://{ CUSTOMER_ID }.collect.observeinc.com/v1/http
    auth:
      strategy: bearer
      token: { LOGS_DATASTREAM_TOKEN }
```

**Notes**:

- This is a partial configuration that only collects MongoDB logs and metrics. You are at liberty to add other sources, transforms and sinks as needed.
- For details on MongoDB metrics, check out [Vector MongoDB source documentation](https://vector.dev/docs/reference/configuration/sources/mongodb_metrics/)
- For details on Log collection, check out [Vector File source documentation](https://vector.dev/docs/reference/configuration/sources/file/)

## Troubleshooting

### MongoDB Log Permissions

By default, MongoDB writes into `/var/log/mongodb/` directory, with individual files owned by the user and group `mongodb` and permissions of `600`

```sh
$ ls -lh /var/log/mongodb/*
-rw------- 1 mongodb mongodb 179M Mar 26 16:08 /var/log/mongodb/mongod.log
```

The vector agent is ran by systemd under the unprivileged user and group `vector`. This means, that by default vector daemon doesn't have read access to mongodb log files. There are several ways to potentially address this issue. The quickest (although arguably not the most secure) way of fixing the problem is by reconfiguring the vector service to run as root. You can do that by modifying `User` and `Group` settings in the vector systemd unit in `/lib/systemd/system/vector.service` to `root` instead of `vector` like so:

```ini
[Unit]
Description=Vector
Documentation=https://vector.dev
After=network-online.target
Requires=network-online.target

[Service]
User=root
Group=root
ExecStartPre=/usr/bin/vector validate
ExecStart=/usr/bin/vector
ExecReload=/usr/bin/vector validate
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
AmbientCapabilities=CAP_NET_BIND_SERVICE
EnvironmentFile=-/etc/default/vector
# Since systemd 229, should be in [Unit] but in order to support systemd <229,
# it is also supported to have it here.
StartLimitInterval=10
StartLimitBurst=5
[Install]
WantedBy=multi-user.target
```

Update systemd configuration and restart vector service:

```sh
sudo systemctl daemon-reload
sudo systemctl restart vector.service
```

## Sample OPAL Pipelines

### Metrics

```js
filter OBSERVATION_KIND = "prometheus" and string(EXTRA.observe_datatype) = "vector_mongo_metrics"
make_col
    metric:string(EXTRA.__name__),
    endpoint:string(EXTRA.endpoint),
    engine:string(EXTRA.engine),
    host:string(EXTRA.host),
    tags:string(EXTRA.micros),
    mode:string(EXTRA.mode),
    observe_env:string(EXTRA.observe_env),
    observe_host:string(EXTRA.observe_host),
    state:string(EXTRA.state),
    type:string(EXTRA.type)

make_col
    timestamp:timestamp_ms(int64(FIELDS.timestamp)),
    value:float64(FIELDS.value)

make_col tags:if(not is_null(tags), make_object(le:tags), make_object())

filter not is_null(metric) and not is_null(value) and value < float64("inf")
set_valid_from options(max_time_diff:duration_hr(4)), timestamp

// https://vector.dev/docs/reference/configuration/sources/mongodb_metrics/
make_col metricType:case(
    contains(metric, "asserts_total"), "cumulativeCounter",
    contains(metric, "bson_parse_error_total"), "cumulativeCounter",
    contains(metric, "connections"), "gauge",
    contains(metric, "extra_info_heap_usage_bytes"), "gauge",
    contains(metric, "extra_info_page_faults"), "gauge",
    contains(metric, "instance_local_time"), "gauge",
    contains(metric, "instance_uptime_estimate_seconds_total"), "gauge",
    contains(metric, "instance_uptime_seconds_total"), "gauge",
    contains(metric, "memory"), "gauge",
    contains(metric, "mongod_global_lock_active_clients"), "gauge",
    contains(metric, "mongod_global_lock_current_queue"), "gauge",
    contains(metric, "mongod_global_lock_total_time_seconds"), "cumulativeCounter",
    contains(metric, "mongod_locks_time_acquiring_global_seconds_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_cursor_open"), "gauge",
    contains(metric, "mongod_metrics_cursor_timed_out_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_document_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_get_last_error_wtime_num"), "gauge",
    contains(metric, "mongod_metrics_get_last_error_wtime_seconds_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_get_last_error_wtimeouts_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_operation_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_query_executor_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_record_moves_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_apply_batches_num_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_apply_batches_seconds_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_apply_ops_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_buffer_count"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_buffer_max_size_bytes_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_buffer_size_bytes"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_executor_queue"), "gauge",
    contains(metric, "mongod_metrics_repl_executor_unsignaled_events"), "gauge",
    contains(metric, "mongod_metrics_repl_network_bytes_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_network_getmores_num_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_network_getmores_seconds_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_network_ops_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_repl_network_readers_created_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_ttl_deleted_documents_total"), "cumulativeCounter",
    contains(metric, "mongod_metrics_ttl_passes_total"), "cumulativeCounter",
    contains(metric, "mongod_op_latencies_histogram"), "gauge",
    contains(metric, "mongod_op_latencies_latency"), "gauge",
    contains(metric, "mongod_op_latencies_ops_total"), "gauge",
    contains(metric, "mongod_storage_engine"), "gauge",
    contains(metric, "mongod_wiredtiger_blockmanager_blocks_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_blockmanager_bytes_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_cache_bytes"), "gauge",
    contains(metric, "mongod_wiredtiger_cache_bytes_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_cache_evicted_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_cache_max_bytes"), "gauge",
    contains(metric, "mongod_wiredtiger_cache_overhead_percent"), "gauge",
    contains(metric, "mongod_wiredtiger_cache_pages"), "gauge",
    contains(metric, "mongod_wiredtiger_cache_pages_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_concurrent_transactions_available_tickets"), "gauge",
    contains(metric, "mongod_wiredtiger_concurrent_transactions_out_tickets"), "gauge",
    contains(metric, "mongod_wiredtiger_concurrent_transactions_total_tickets"), "gauge",
    contains(metric, "mongod_wiredtiger_log_bytes_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_log_operations_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_log_records_scanned_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_log_records_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_session_open_sessions"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_transactions_checkpoint_seconds"), "gauge",
    contains(metric, "mongod_wiredtiger_transactions_checkpoint_seconds_total"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_transactions_running_checkpoints"), "cumulativeCounter",
    contains(metric, "mongod_wiredtiger_transactions_total"), "cumulativeCounter",
    contains(metric, "mongodb_op_counters_repl_total"), "cumulativeCounter",
    contains(metric, "mongodb_op_counters_total"), "cumulativeCounter",
    contains(metric, "network_bytes_total"), "cumulativeCounter",
    contains(metric, "network_metrics_num_requests_total"), "cumulativeCounter",
    contains(metric, "up"), "gauge",
    true, "gauge"
)

make_col unit: case(
  contains(metric, "seconds"), "seconds",
  contains(metric, "bytes"), "bytes"
)

pick_col
    timestamp, metric, value, metricType, endpoint, engine, host, tags,
    mode, observe_env, observe_host, state, type, unit

interface "metric",  metricType: type, metricUnit: unit
```

This will auto-populate the collected mongo metrics in the Observe Metrics Explorer:

![mongo-metrics](screenshots/metrics-observe.png)

### Raw Logs

**Notes:**

- The following OPAL assumes raw datastream collecting vector log data. If you already have a log data transformation pipeline, adjust the following OPAL accordingly:

```js
filter OBSERVATION_KIND = "http" and string(FIELDS.tags.observe_datatype) = "vector_logs"

pick_col
  BUNDLE_TIMESTAMP,
  message:parse_json(string(FIELDS.message)),
  file:string(FIELDS.file),
  host:string(FIELDS.tags.observe_host),
  env:string(FIELDS.tags.observe_env)

filter file ~ mongodb

make_col timestamp:parse_isotime(string(message.t['$date']))
set_valid_from options(max_time_diff:duration_min(15)), timestamp
drop_col BUNDLE_TIMESTAMP

pick_col
  timestamp,
  message,
  host,
  env,
  file

interface "log"
```

This will give you raw logs that can further shaped into Slow Query, Command, and other types of MongoDB logs as appropriate:

![mongo-logs](screenshots/raw-logs-observe.png)
