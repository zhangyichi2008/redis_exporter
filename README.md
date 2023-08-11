# Prometheus Redis Metrics Exporter

[![Build Status](https://drone-github.21zoo.com/api/badges/oliver006/redis_exporter/status.svg)](https://drone-github.21zoo.com/oliver006/redis_exporter)
 [![Coverage Status](https://coveralls.io/repos/github/oliver006/redis_exporter/badge.svg?branch=master)](https://coveralls.io/github/oliver006/redis_exporter?branch=master) [![codecov](https://codecov.io/gh/oliver006/redis_exporter/branch/master/graph/badge.svg)](https://codecov.io/gh/oliver006/redis_exporter) [![docker_pulls](https://img.shields.io/docker/pulls/oliver006/redis_exporter.svg)](https://img.shields.io/docker/pulls/oliver006/redis_exporter.svg) [![Stand With Ukraine](https://raw.githubusercontent.com/vshymanskyy/StandWithUkraine/main/badges/StandWithUkraine.svg)](https://stand-with-ukraine.pp.ua)

Prometheus exporter for Redis metrics.\
Supports Redis 2.x, 3.x, 4.x, 5.x, 6.x, and 7.x

**v1.51.1 / 2023-08-07**

**[ADD] 增加对sentinel集群状态的监控**

- HELP redis_sentinel_up sentinel_up metric
- TYPE redis_sentinel_up gauge
- redis_sentinel_up 1
- HELP redis_sentinel_master_sentinels The number of sentinels monitoring this master
- TYPE redis_sentinel_master_sentinels gauge
- redis_sentinel_master_sentinels{master_address="10.1.1.1:6379",master_name="mymaster"} 3
- HELP redis_sentinel_master_slaves The number of slaves of the master
- TYPE redis_sentinel_master_slaves gauge
- redis_sentinel_master_slaves{master_address="10.1.1.1:6379",master_name="mymaster"} 2
- HELP redis_sentinel_master_status Master status on Sentinel
- TYPE redis_sentinel_master_status gauge
- redis_sentinel_master_status{master_address="10.1.1.1:6379",master_name="mymaster",master_status="ok"} 1
- HELP redis_sentinel_masters The number of masters this sentinel is watching
- TYPE redis_sentinel_masters gauge
- redis_sentinel_masters 1
- HELP redis_sentinel_running_scripts Number of scripts in execution right now
- TYPE redis_sentinel_running_scripts gauge
- redis_sentinel_running_scripts 0
- HELP redis_sentinel_scripts_queue_length Queue of user scripts to execute
- TYPE redis_sentinel_scripts_queue_length gauge
- redis_sentinel_scripts_queue_length 0
- HELP redis_sentinel_simulate_failure_flags Failures simulations
- TYPE redis_sentinel_simulate_failure_flags gauge
- redis_sentinel_simulate_failure_flags 0
- HELP redis_sentinel_tilt Sentinel is in TILT mode
- TYPE redis_sentinel_tilt gauge
- redis_sentinel_tilt 0
- HELP redis_sentinel_last_scrape_error The last scrape sentinel error status.
- TYPE redis_sentinel_last_scrape_error gauge
- redis_sentinel_last_scrape_error{err=""} 0


**[DEL] 移除以go_开头的33个监控项,具体如下：**

- go_gc_duration_seconds{quantile="0"}
- go_gc_duration_seconds{quantile="0.25"}
- go_gc_duration_seconds{quantile="0.5"}
- go_gc_duration_seconds{quantile="0.75"}
- go_gc_duration_seconds{quantile="1"}
- go_gc_duration_seconds_sum
- go_gc_duration_seconds_count
- go_goroutines
- go_info{version="go1.19.1"}
- go_memstats_alloc_bytes
- go_memstats_alloc_bytes_total
- go_memstats_buck_hash_sys_bytes
- go_memstats_frees_total
- go_memstats_gc_sys_bytes
- go_memstats_heap_alloc_bytes
- go_memstats_heap_idle_bytes
- go_memstats_heap_inuse_bytes
- go_memstats_heap_objects
- go_memstats_heap_released_bytes
- go_memstats_heap_sys_bytes
- go_memstats_last_gc_time_seconds
- go_memstats_lookups_total
- go_memstats_mallocs_total
- go_memstats_mcache_inuse_bytes
- go_memstats_mcache_sys_bytes
- go_memstats_mspan_inuse_bytes
- go_memstats_mspan_sys_bytes
- go_memstats_next_gc_bytes
- go_memstats_other_sys_bytes
- go_memstats_stack_inuse_bytes
- go_memstats_stack_sys_bytes
- go_memstats_sys_bytes
- go_threads


## Building and running the exporter

### Build and run locally

```sh
cd redis_exporter
go build .
./redis_exporter --version
```


### Pre-build binaries

For pre-built binaries please take a look at [the releases](https://github.com/oliver006/redis_exporter/releases).


### Basic Prometheus Configuration

Add a block to the `scrape_configs` of your prometheus.yml config file:

```yaml
scrape_configs:
  - job_name: redis_exporter
    static_configs:
    - targets: ['<<REDIS-EXPORTER-HOSTNAME>>:9121']
```

and adjust the host name accordingly.


### Kubernetes SD configurations

To have instances in the drop-down as human readable names rather than IPs, it is suggested to use [instance relabelling](https://www.robustperception.io/controlling-the-instance-label).

For example, if the metrics are being scraped via the pod role, one could add:

```yaml
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: instance
            regex: (.*redis.*)
```

as a relabel config to the corresponding scrape config. As per the regex value, only pods with "redis" in their name will be relabelled as such.

Similar approaches can be taken with [other role types](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config) depending on how scrape targets are retrieved.


### Prometheus Configuration to Scrape Multiple Redis Hosts

The Prometheus docs have a [very informative article](https://prometheus.io/docs/guides/multi-target-exporter/) on how multi-target exporters are intended to work.

Run the exporter with the command line flag `--redis.addr=` so it won't try to access the local instance every time the `/metrics` endpoint is scraped. Using below config instead of the /metric endpoint the /scrape endpoint will be used by prometheus. As an example the first target will be queried with this web request:
http://exporterhost:9121/scrape?target=first-redis-host:6379

```yaml
scrape_configs:
  ## config for the multiple Redis targets that the exporter will scrape
  - job_name: 'redis_exporter_targets'
    static_configs:
      - targets:
        - redis://first-redis-host:6379
        - redis://second-redis-host:6379
        - redis://second-redis-host:6380
        - redis://second-redis-host:6381
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <<REDIS-EXPORTER-HOSTNAME>>:9121

  ## config for scraping the exporter itself
  - job_name: 'redis_exporter'
    static_configs:
      - targets:
        - <<REDIS-EXPORTER-HOSTNAME>>:9121
```

The Redis instances are listed under `targets`, the Redis exporter hostname is configured via the last relabel_config rule.\
If authentication is needed for the Redis instances then you can set the password via the `--redis.password` command line option of
the exporter (this means you can currently only use one password across the instances you try to scrape this way. Use several
exporters if this is a problem). \
You can also use a json file to supply multiple targets by using `file_sd_configs` like so:

```yaml

scrape_configs:
  - job_name: 'redis_exporter_targets'
    file_sd_configs:
      - files:
        - targets-redis-instances.json
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <<REDIS-EXPORTER-HOSTNAME>>:9121

  ## config for scraping the exporter itself
  - job_name: 'redis_exporter'
    static_configs:
      - targets:
        - <<REDIS-EXPORTER-HOSTNAME>>:9121
```

The `targets-redis-instances.json` should look something like this:

```json
[
  {
    "targets": [ "redis://redis-host-01:6379", "redis://redis-host-02:6379"],
    "labels": { }
  }
]
```

Prometheus uses file watches and all changes to the json file are applied immediately.


### Command line flags

| Name                    | Environment Variable Name              | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|-------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| redis.addr              | REDIS_ADDR                             | Address of the Redis instance, defaults to `redis://localhost:6379`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| sentinel.addr           | SENTINEL_ADDR                          | Address of the Sentinel instance, defaults to `host:port ex:10.66.14.104:26379`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| redis.user              | REDIS_USER                             | User name to use for authentication (Redis ACL for Redis 6.0 and newer).                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| redis.password          | REDIS_PASSWORD                         | Password of the Redis instance, defaults to `""` (no password).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| redis.password-file     | REDIS_PASSWORD_FILE                    | Password file of the Redis instance to scrape, defaults to `""` (no password file).                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| check-keys              | REDIS_EXPORTER_CHECK_KEYS              | Comma separated list of key patterns to export value and length/size, eg: `db3=user_count` will export key `user_count` from db `3`. db defaults to `0` if omitted. The key patterns specified with this flag will be found using [SCAN](https://redis.io/commands/scan).  Use this option if you need glob pattern matching; `check-single-keys` is faster for non-pattern keys. Warning: using `--check-keys` to match a very large number of keys can slow down the exporter to the point where it doesn't finish scraping the redis instance. |
| check-single-keys       | REDIS_EXPORTER_CHECK_SINGLE_KEYS       | Comma separated list of keys to export value and length/size, eg: `db3=user_count` will export key `user_count` from db `3`. db defaults to `0` if omitted.  The keys specified with this flag will be looked up directly without any glob pattern matching.  Use this option if you don't need glob pattern matching;  it is faster than `check-keys`.                                                                                                                                                                                           |
| check-streams           | REDIS_EXPORTER_CHECK_STREAMS           | Comma separated list of stream-patterns to export info about streams, groups and consumers. Syntax is the same as `check-keys`.                                                                                                                                                                                                                                                                                                                                                                                                                   |
| check-single-streams    | REDIS_EXPORTER_CHECK_SINGLE_STREAMS    | Comma separated list of streams to export info about streams, groups and consumers. The streams specified with this flag will be looked up directly without any glob pattern matching.  Use this option if you don't need glob pattern matching;  it is faster than `check-streams`.                                                                                                                                                                                                                                                              |
| check-keys-batch-size   | REDIS_EXPORTER_CHECK_KEYS_BATCH_SIZE   | Approximate number of keys to process in each execution. This is basically the COUNT option that will be passed into the SCAN command as part of the execution of the key or key group metrics, see [COUNT option](https://redis.io/commands/scan#the-count-option). Larger value speeds up scanning. Still Redis is a single-threaded app, huge `COUNT` can affect production environment.                                                                                                                                                       |
| count-keys              | REDIS_EXPORTER_COUNT_KEYS              | Comma separated list of patterns to count, eg: `db3=sessions:*` will count all keys with prefix `sessions:` from db `3`. db defaults to `0` if omitted. Warning: The exporter runs SCAN to count the keys. This might not perform well on large databases.                                                                                                                                                                                                                                                                                        |
| script                  | REDIS_EXPORTER_SCRIPT                  | Comma separated list of path(s) to Redis Lua script(s) for gathering extra metrics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| debug                   | REDIS_EXPORTER_DEBUG                   | Verbose debug output                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| log-format              | REDIS_EXPORTER_LOG_FORMAT              | Log format, valid options are `txt` (default) and `json`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| namespace               | REDIS_EXPORTER_NAMESPACE               | Namespace for the metrics, defaults to `redis`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| connection-timeout      | REDIS_EXPORTER_CONNECTION_TIMEOUT      | Timeout for connection to Redis instance, defaults to "15s" (in Golang duration format)                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| web.listen-address      | REDIS_EXPORTER_WEB_LISTEN_ADDRESS      | Address to listen on for web interface and telemetry, defaults to `0.0.0.0:9121`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| web.telemetry-path      | REDIS_EXPORTER_WEB_TELEMETRY_PATH      | Path under which to expose metrics, defaults to `/metrics`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| redis-only-metrics      | REDIS_EXPORTER_REDIS_ONLY_METRICS      | Whether to also export go runtime metrics, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| include-config-metrics  | REDIS_EXPORTER_INCL_CONFIG_METRICS     | Whether to include all config settings as metrics, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| include-system-metrics  | REDIS_EXPORTER_INCL_SYSTEM_METRICS     | Whether to include system metrics like `total_system_memory_bytes`, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| redact-config-metrics   | REDIS_EXPORTER_REDACT_CONFIG_METRICS   | Whether to redact config settings that include potentially sensitive information like passwords.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ping-on-connect         | REDIS_EXPORTER_PING_ON_CONNECT         | Whether to ping the redis instance after connecting and record the duration as a metric, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| is-tile38               | REDIS_EXPORTER_IS_TILE38               | Whether to scrape Tile38 specific metrics, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| is-cluster              | REDIS_EXPORTER_IS_CLUSTER              | Whether this is a redis cluster (Enable this if you need to fetch key level data on a Redis Cluster).                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| export-client-list      | REDIS_EXPORTER_EXPORT_CLIENT_LIST      | Whether to scrape Client List specific metrics, defaults to false.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| export-client-port      | REDIS_EXPORTER_EXPORT_CLIENT_PORT      | Whether to include the client's port when exporting the client list. Warning: including the port increases the number of metrics generated and will make your Prometheus server take up more memory                                                                                                                                                                                                                                                                                                                                               |
| skip-tls-verification   | REDIS_EXPORTER_SKIP_TLS_VERIFICATION   | Whether to to skip TLS verification when the exporter connects to a Redis instance                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| tls-client-key-file     | REDIS_EXPORTER_TLS_CLIENT_KEY_FILE     | Name of the client key file (including full path) if the server requires TLS client authentication                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| tls-client-cert-file    | REDIS_EXPORTER_TLS_CLIENT_CERT_FILE    | Name the client cert file (including full path) if the server requires TLS client authentication                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| tls-server-key-file     | REDIS_EXPORTER_TLS_SERVER_KEY_FILE     | Name of the server key file (including full path) if the web interface and telemetry should use TLS                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| tls-server-cert-file    | REDIS_EXPORTER_TLS_SERVER_CERT_FILE    | Name of the server certificate file (including full path) if the web interface and telemetry should use TLS                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| tls-server-ca-cert-file | REDIS_EXPORTER_TLS_SERVER_CA_CERT_FILE | Name of the CA certificate file (including full path) if the web interface and telemetry should use TLS                                                                                                                                                                                                                                                                                                                                                                                                                 |
| tls-server-min-version  | REDIS_EXPORTER_TLS_SERVER_MIN_VERSION  | Minimum TLS version that is acceptable by the web interface and telemetry when using TLS, defaults to `TLS1.2` (supports `TLS1.0`,`TLS1.1`,`TLS1.2`,`TLS1.3`).                                                                                                                                                                                                                                                                                                                                                                                    |
| tls-ca-cert-file        | REDIS_EXPORTER_TLS_CA_CERT_FILE        | Name of the CA certificate file (including full path) if the server requires TLS client authentication                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| set-client-name         | REDIS_EXPORTER_SET_CLIENT_NAME         | Whether to set client name to redis_exporter, defaults to true.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| check-key-groups        | REDIS_EXPORTER_CHECK_KEY_GROUPS        | Comma separated list of [LUA regexes](https://www.lua.org/pil/20.1.html) for classifying keys into groups. The regexes are applied in specified order to individual keys, and the group name is generated by concatenating all capture groups of the first regex that matches a key. A key will be tracked under the `unclassified` group if none of the specified regexes matches it.                                                                                                                                                            |
| max-distinct-key-groups | REDIS_EXPORTER_MAX_DISTINCT_KEY_GROUPS | Maximum number of distinct key groups that can be tracked independently *per Redis database*. If exceeded, only key groups with the highest memory consumption within the limit will be tracked separately, all remaining key groups will be tracked under a single `overflow` key group.                                                                                                                                                                                                                                                         |
| config-command          | REDIS_EXPORTER_CONFIG_COMMAND          | What to use for the CONFIG command, defaults to `CONFIG`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |

Redis instance addresses can be tcp addresses: `redis://localhost:6379`, `redis.example.com:6379` or e.g. unix sockets: `unix:///tmp/redis.sock`.\
SSL is supported by using the `rediss://` schema, for example: `rediss://azure-ssl-enabled-host.redis.cache.windows.net:6380` (note that the port is required when connecting to a non-standard 6379 port, e.g. with Azure Redis instances).\

Command line settings take precedence over any configurations provided by the environment variables.


### Authenticating with Redis

If your Redis instance requires authentication then there are several ways how you can supply
a username (new in Redis 6.x with ACLs) and a password.

You can provide the username and password as part of the address, see [here](https://www.iana.org/assignments/uri-schemes/prov/redis) for the official documentation of the `redis://` scheme.
You can set `-redis.password-file=sample-pwd-file.json` to specify a password file, it's used whenever the exporter connects to a Redis instance,
no matter if you're using the `/scrape` endpoint for multiple instances or the normal `/metrics` endpoint when scraping just one instance.
It only takes effect when `redis.password == ""`.  See the [contrib/sample-pwd-file.json](contrib/sample-pwd-file.json) for a working example, and make sure to always include the `redis://` in your password file entries.

An example for a URI including a password is: `redis://<<username (optional)>>:<<PASSWORD>>@<<HOSTNAME>>:<<PORT>>`

Alternatively, you can provide the username and/or password using the `--redis.user` and `--redis.password` directly to the redis_exporter.

If you want to use a dedicated Redis user for the redis_exporter (instead of the default user) then you need enable a list of commands for that user.
You can use the following Redis command to set up the user, just replace `<<<USERNAME>>>` and `<<<PASSWORD>>>` with your desired values.
```
ACL SETUSER <<<USERNAME>>> +client +ping +info +config|get +cluster|info +slowlog +latency +memory +select +get +scan +xinfo +type +pfcount +strlen +llen +scard +zcard +hlen +xlen +eval allkeys on > <<<PASSWORD>>>
```
