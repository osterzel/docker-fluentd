# Docker-Fluentd: the Container to Log Other Containers' Logs

## What

By running this container with the following command, one can aggregate the logs of Docker containers running on the same host:

```
docker run -d -v /var/lib/docker/containers:/var/lib/docker/containers kiyoto/docker-fluentd
```

By default, the container logs are stored in /var/log/docker/yyyyMMdd.log inside this logging container. The data is buffered, so you may also see buffer files like /var/log/docker/20141114.b507c71e6fe540eab.log where "b507c71e6fe540eab" is a hash identifier. You can mount that container volume back to host. Also, by modifying `fluent.conf` and rebuilding the Docker image, you can stream up your logs to Elasticsearch, Amazon S3, MongoDB, Treasure Data, etc.

The output log looks exactly like Docker's JSON formatted logs, except each line also has the "container_id" field:

```
{"log":"Fri Nov 14 01:03:19 UTC 2014\r\n","stream":"stdout","container_id":"db18480728ed247a64bf6df49684cb246a38bbe11f14276d4c2bb84f56255ff4"}
```

## How

`docker-fluentd` uses [Fluentd](https://www.fluentd.org) inside to tail log files that are mounted on `/var/lib/docker/containers/<CONTAINER_ID>/<CONTAINER_ID>-json.log`. It uses the [tail input plugin](https://docs.fluentd.org/articles/in_tail) to tail JSON-formatted log files that each Docker container emits.

Then, Fluentd adds the "container_id" field using the [record reformer](https://github.com/sonots/fluent-plugin-record-reformer) before writing out to local files using the [file output plugin](https://docs.fluentd.org/articles/out_file).

Fluentd has [a lot of plugins](https://www.fluentd.org/plugins) and can probably support your data output destination. For example, if you want to output your logs to Elasticsearch instead of files, then, add the following line to `Dockerfile`

```
RUN ["/usr/local/bin/gem", "install", "fluent-plugin-record-reformer", "--no-rdoc", "--no-ri"]
```

right before "ENTRYPOINT" and update `fluent.conf` as follows.


```
<source>
  type tail
  path /var/lib/docker/containers/*/*-json.log
  pos_file /var/log/fluentd-docker.pos
  time_format %Y-%m-%dT%H:%M:%S 
  tag docker.*
  format json
</source>

<match docker.var.lib.docker.containers.*.*.log>
  type record_reformer
  container_id ${tag_parts[5]}
  tag docker.all
</match>

<match docker.all>
  type elasticsearch
  log_level info
  host YOUR_ES_HOST
  port YOUR_ES_PORT
  include_tag_key true 
  logstash_format true
  flush_intercal 5s
</match>
```

## What's Next?

- [Fluentd website](https://www.fluentd.org)
- [Fluentd's Repo](https://github.com/fluent/fluentd)
- [Kubernetes's Logging Pod](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/contrib/logging)
