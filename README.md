NGINX-to-Prometheus log file exporter
=====================================

[![Build Status](https://travis-ci.org/martin-helmich/prometheus-nginxlog-exporter.svg?branch=master)](https://travis-ci.org/martin-helmich/prometheus-nginxlog-exporter)
[![Docker Repository on Quay](https://quay.io/repository/martinhelmich/prometheus-nginxlog-exporter/status "Docker Repository on Quay")](https://quay.io/repository/martinhelmich/prometheus-nginxlog-exporter)

Helper tool that continuously reads an NGINX log file and exports metrics to
[Prometheus](prom).

Usage
-----

You can either use a simple configuration, using command-line flags, or create
a configuration file with a more advanced configuration.

Use the command-line:

    ./nginx-log-exporter \
      -format="<FORMAT>" \
      -listen-port=4040 \
      -namespace=nginx \
      [PATHS-TO-LOGFILES...]

Use the configuration file:

    ./nginx-log-exporter -config-file /path/to/config.hcl

Collected metrics
-----------------

This exporter collects the following metrics. This collector can listen on
multiple log files at once and publish metrics in different namespaces. Each
metric uses the labels `method` (containing the HTTP request method) and
`status` (containing the HTTP status code).

- `<namespace>_http_response_count_total` - The total amount of processed HTTP requests/responses.
- `<namespace>_http_response_size_bytes` - The total amount of transferred content in bytes.
- `<namespace>_http_upstream_time_seconds` - A summary vector of the upstream
  response times in seconds. Logging these needs to be specifically enabled in
  NGINX using the `$upstream_response_time` variable in the log format.
- `<namespace>_http_response_time_seconds` - A summary vector of the total
  response times in seconds. Logging these needs to be specifically enabled in
  NGINX using the `$request_time` variable in the log format.

Additional labels can be configured in the configuration file (see below).

Configuration file
------------------

You can specify a configuration file to read at startup. The configuration file
is expected to be in [HCL](hcl) format. Here's an example file:

```hcl
listen {
  port = 4040
  address = "10.1.2.3"
}

consul {
  enable = true
  address = "localhost:8500"
  service {
    id = "nginx-exporter"
    name = "nginx-exporter"
    datacenter = "dc1"
    scheme = "http"
    token = ""
    tags = ["foo", "bar"]
  }
}

namespace "app-1" {
  format = "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" \"$http_x_forwarded_for\""
  source_files = [
    "/var/log/nginx/app1/access.log"
  ]
  labels {
    app = "application-one"
    environment = "production"
    foo = "bar"
  }
}

namespace "app-2" {
  format = "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" \"$http_x_forwarded_for\" $upstream_response_time"
  source_files = [
    "/var/log/nginx/app2/access.log"
  ]
}
```

Experimental features
---------------------

The exporter contains features that are currently experimental and may change without prior notice.
To use these features, either set the `-enable-experimental` flag or add a `enable_experimental` option
to your configuration file.

### Dynamic re-labeling

Re-labeling lets you add arbitrary fields from the parsed log line as labels to your metrics.
To add a dynamic label, add a `relabel` statement to your configuration file:

```hcl
namespace "app-1" {
  // ...

  relabel "host" {
    from = "server_name"
    whitelist = [
      "host-a.com",
      "host-b.de"
    ]
  }
}
```

The `whitelist` property is optional; if set, only the supplied values will be added as label.
All other values will be subsumed under the `"other"` label value. See #16 for a more detailed
discussion around the reasoning.

Dynamic relabeling also allows you to aggregate your metrics by request path (which replaces
the experimental feature originally introduced in #23):

```hcl
namespace "app-1" {
  // ...

  relabel "request_uri" {
    from = "request"
    split = 2

    match "^/users/[0-9]+" {
      replacement = "/users/:id"
    }

    match "^/profile" {
      replacement = "/profile"
    }
  }
}
```

Running the collector
---------------------

### Systemd

You can find an example unit file for this service [in this repository](systemd/prometheus-nginxlog-exporter.service). Simply copy the unit file to `/etc/systemd/system`:

    $ wget -O /etc/systemd/system/prometheus-nginxlog-exporter.service https://raw.githubusercontent.com/martin-helmich/prometheus-nginxlog-exporter/master/systemd/prometheus-nginxlog-exporter.service
    $ systemctl enable prometheus-nginxlog-exporter
    $ systemctl start prometheus-nginxlog-exporter

The shipped unit file expects the binary to be located in `/usr/local/bin/prometheus-nginxlog-exporter` and the configuration file in `/etc/prometheus-nginxlog-exporter.hcl`. Adjust to your own needs.

### Docker

You can also run this exporter from the Docker image `quay.io/martinhelmich/prometheus-nginxlog-exporter`:

    $ docker run --name nginx-exporter -v logs:/mnt/nginxlogs -p 4040:4040 quay.io/martinhelmich/prometheus-nginxlog-exporter mnt/nginxlogs/access.log

Command-line flags and arguments can simply be appended to the `docker run` command, for example to use a
configuration file:

    $ docker run --name nginx-exporter -p 4040:4040 -v logs:/mnt/nginxlogs -v /path/to/config.hcl:/etc/prometheus-nginxlog-exporter.hcl quay.io/martinhelmich/prometheus-nginxlog-exporter -config-file /etc/prometheus-nginxlog-exporter.hcl

Credits
-------

- [tail](https://github.com/hpcloud/tail), MIT license
- [gonx](https://github.com/satyrius/gonx), MIT license
- [Prometheus Go client library](https://github.com/prometheus/client_golang), Apache License
- [HashiCorp configuration language](hcl), Mozilla Public License

[prom]: https://prometheus.io/
[hcl]: https://github.com/hashicorp/hcl
