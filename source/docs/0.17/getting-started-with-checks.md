---
version: 0.17
category: "Getting Started Guide"
title: "Getting Started with Checks"
next:
  url: "getting-started-with-handlers"
  text: "Getting Started with Handlers"
---

# Overview

The purpose of this guide is to help Sensu users create monitoring checks. At the conclusion of this guide, you - the user - should have several Sensu checks in place to monitor and measure machine resources, applications, and services. Each Sensu monitoring check in this guide demonstrates one or more check definition features, for more information please refer to the [Sensu checks reference documentation](checks).

## Objectives

What will be covered in this guide:

- Creation of **standard** checks (functional tests)
- Creation of **metric collection** checks (server resources, etc)
- Creation of **metric analysis** checks (querying time series data, etc)

# What are Sensu checks? {#what-are-sensu-checks}

Sensu checks allow you to monitor server resources, services, and application health, as well as collect & analyze metrics; they are executed on servers running the Sensu client. Checks are essentially commands (or scripts) that output data to `STDOUT` or `STDERR` and produce an exit status code to indicate a state. The common exit status codes used are `0` for `OK`, `1` for `WARNING`, `2` for `CRITICAL`, and `3` or greater to indicate `UNKNOWN` or `CUSTOM`. **Sensu checks use the same specification as Nagios**, therefore, Nagios check plugins may be used with Sensu.

# Create a standard check

Standard Sensu checks are used to determine the health of server resources, services and applications. A standard check will query a resource for information to determine its state. Once a standard check has determined the resource state, it outputs a human readable message, and exits with the appropriate exit status code to indicate its state/severity (`OK`, `WARNING`, etc.).

## Monitor the cron service

The following instructions install the check dependencies and configure the Sensu check definition in order to monitor the Cron service.

### Install dependencies

The `check-procs` Sensu plugin can reliably detect if a service such as Cron is running or not. The following instructions will install the `check-procs` Sensu check plugin (written in Ruby) to `/etc/sensu/plugins/check-procs.rb`.

~~~ shell
sudo wget -O /etc/sensu/plugins/check-procs.rb http://sensuapp.org/docs/0.17/files/check-procs.rb
sudo chmod +x /etc/sensu/plugins/check-procs.rb
~~~

The `check-procs` Sensu plugin requires a Ruby runtime and the `sensu-plugin` Ruby gem. Install Ruby from the distribution repository and `sensu-plugin` from Rubygems:

_NOTE: the following Ruby installation steps may differ depending on your platform._

#### Ubuntu/Debian

~~~ shell
sudo apt-get update
sudo apt-get install ruby ruby-dev
sudo gem install sensu-plugin
~~~

#### CentOS/RHEL

~~~ shell
sudo yum install ruby ruby-devel
sudo gem install sensu-plugin
~~~

### Create the check definition for Cron

The following is an example Sensu check definition, a JSON configuration file located at `/etc/sensu/conf.d/check_cron.json`. This check definition uses the check-procs plugin ([installed above](#install-dependencies)) to determine if the Cron service is running. The check is named `cron` and it runs <kbd>/etc/sensu/plugins/check-procs.rb -p cron</kbd> on Sensu clients with the `production` subscription, every `60` seconds (interval).

_NOTE: Sensu services must be restarted in order to pick up configuration changes. Sensu Enterprise can be reloaded._

~~~ json
{
  "checks": {
    "cron": {
      "command": "/etc/sensu/plugins/check-procs.rb -p cron",
      "subscribers": [
        "production"
      ],
      "interval": 60
    }
  }
}
~~~

For a full listing of the `check-procs` command line arguments, run <kbd>/etc/sensu/plugins/check-procs.rb -h</kbd>.

Currently, the Cron check definition requires that check requests be sent to Sensu clients with the `production` subscription. This is known as **check subscription mode**. Optionally, a check may use `standalone` mode, which allows clients to schedule their own check executions. The following is an example of the Cron check using `standalone` mode (`true`). The Cron check will now be executed every `60` seconds on each Sensu client with the check definition. A Sensu check definition with `"standalone": true` does not need to specify `subscribers`.

~~~ json
{
  "checks": {
    "cron": {
      "command": "/etc/sensu/plugins/check-procs.rb -p cron",
      "standalone": true,
      "interval": 60
    }
  }
}
~~~

By default, Sensu checks use the `default` Sensu event handler for events they create. To specify a different Sensu event handler for a check, use the `handler` attribute. The `debug` event handler used in this example will log the Sensu event data to the Sensu server log.

~~~ json
{
  "checks": {
    "cron": {
      "command": "/etc/sensu/plugins/check-procs.rb -p cron",
      "standalone": true,
      "interval": 60,
      "handler": "debug"
    }
  }
}
~~~

#### Using multiple handlers

To specify multiple Sensu event handlers, use the `handlers` attribute (plural).

_NOTE: if both `handler` and `handlers` (plural) check definition attributes are used, `handlers` will take precedence._

~~~ json
{
  "checks": {
    "cron": {
      "command": "/etc/sensu/plugins/check-procs.rb -p cron",
      "standalone": true,
      "interval": 60,
      "handlers": ["default", "debug"]
    }
  }
}
~~~

# Create a metric collection check

Metric collection checks are used to collect measurements and other data (metrics) from server resources, services, and applications. Metric collection checks can output in a variety of metric formats:

- [Graphite plaintext](http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-plaintext-protocol)
- [Nagios Performance Data](http://nagios.sourceforge.net/docs/3_0/perfdata.html)
- [OpenTSDB](http://opentsdb.net/docs/build/html/user_guide/writing.html)
- [Metrics 2.0 wire format](http://metrics20.org/spec/)

## Measuring CPU utilization

### Install dependencies {#cpu-metrics-install-dependencies}

The following instructions install the `cpu-metrics` Sensu plugin (written in Ruby) to `/etc/sensu/plugins/cpu-metrics.rb`. This Sensu plugin will collect CPU metrics and output them in the Graphite plaintext format.

~~~ shell
sudo wget -O /etc/sensu/plugins/cpu-metrics.rb http://sensuapp.org/docs/0.17/files/cpu-metrics.rb
sudo chmod +x /etc/sensu/plugins/cpu-metrics.rb
~~~

### Create the check definition for CPU utilization

The following is an example Sensu check definition, a JSON configuration file located at `/etc/sensu/conf.d/cpu_metrics.json`. This check definition uses the `cpu-metrics` plugin ([installed above](#cpu-metrics-install-dependencies)) to collect CPU utilization metrics and output them in the Graphite plaintext format.

By default, Sensu checks with an exit status code of `0` (for `OK`) do not create events unless they indicate a change in state from a non-zero status to a zero status (i.e. resulting in a `resolve` action; see: [Sensu Events](events#what-are-sensu-events)). Metric collection checks will output metric data regardless of the check exit status code, however, they usually exit `0`. To ensure events are always created for a metric collection check, the check `type` of `metric` is used.

The check is named `cpu_metrics`, and it runs <kbd>/etc/sensu/plugins/cpu-metrics.rb</kbd> on Sensu clients with the `production` subscription, every `10` seconds (interval). The `debug` handler is used to log the CPU utilization metrics to the Sensu server log.

_NOTE: Sensu services must be restarted in order to pick up configuration changes. Sensu Enterprise can be reloaded._

~~~ json
{
  "checks": {
    "cpu_metrics": {
      "type": "metric",
      "command": "/etc/sensu/plugins/cpu-metrics.rb",
      "subscribers": [
        "production"
      ],
      "interval": 10,
      "handler": "debug"
    }
  }
}
~~~

For a full listing of the `cpu-metrics` command line arguments, run <kbd>/etc/sensu/plugins/cpu-metrics.rb -h</kbd>.

# Create a metric analysis check

A metric analysis check analyzes metric data which may or may not have been collected by a [metrics collection check](#metric-collection-checks). By querying external metric stores (e.g. Graphite) to perform data evaluations, metric analysis checks allow you to perform powerful analytics based on trends in metric data rather than a single data point. For example, where monitoring and alerting on on a single CPU utilization data point can result in false positive events based on momentary spikes, monitoring and alerting on CPU utilization data over a specified period of time will improve alerting accuracy.

Because metric analysis checks require interaction with an external metric store, providing a functional example is outside of the scope of this guide. However, assuming the existence of a Graphite installation that is populated with metric data, the following example checks could be used.

The following check uses the `check-data` plugin to query the Graphite API at `localhost:9001`. The check queries Graphite for a calculated moving average (using the last 10 data points) of the load balancer session count. The session count moving average is compared with the provided alert thresholds. A Sensu client running on the Graphite server would be responsible for scheduling and executing this check (`standalone` mode).

_NOTE: Sensu services must be restarted in order to pick up configuration changes. Sensu Enterprise can be reloaded._

~~~ json
{
  "checks": {
    "disk_capacity": {
      "command": "check-data.rb -s localhost:9001 -t 'movingAverage(lb1.assets_backend.session_current,10)' -w 100 -c 200",
      "standalone": true,
      "interval": 30
    }
  }
}
~~~

The following check uses the `check-data` plugin to query the Graphite API at `localhost:9001` for disk capacity metrics. The Graphite API query uses `highestCurrent()` to grab only the highest disk capacity metric, to be compared with the provided alert thresholds. This check will trigger an event (alert) when one or more disks on any machine are at the configured capacity threshold. In this example configuration, the check is configured to **warn** at 85% capacity (`-w 85`), and to raise a **critical** alert at 95% capacity (`-c 95`).

~~~ json
{
  "checks": {
    "disk_capacity": {
      "command": "check-data.rb -s localhost:9001 -t 'highestCurrent(*.disk.*.capacity,1)' -w 85 -c 95 -a 120",
      "standalone": true,
      "interval": 30
    }
  }
}
~~~

The `check-data` plugin can be installed with the following instructions:

~~~ shell
sudo wget -O /etc/sensu/plugins/check-data.rb http://sensuapp.org/docs/0.17/files/check-data.rb
sudo chmod +x /etc/sensu/plugins/check-data.rb
/etc/sensu/plugins/check-data.rb -h
~~~
