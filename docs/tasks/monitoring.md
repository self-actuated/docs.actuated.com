# Task: Monitoring Actuated

!!! info "Our team monitors actuated around the clock, on your behalf"
    The actuated team proactively monitors your servers and build queue for issues. We remediate them on your behalf and for anything cannot be fixed remotely, we'll be in touch via Slack or email.

The [actuated CLI](/tasks/cli) should be used for support, to query the agent's logs, or the logs of individual VMs.

If you would also like to do your own monitoring, you can purchase a monitoring add-on, which will expose metrics for your own Prometheus instance. You can then set up a Grafana dashboard to view the metrics.

The monitoring add-on provides:

* Control-plane metrics such as queue-depth, delayed VM launches, and failed jobs.
* Server metrics such as VMs running, load averages, and network I/O.

To opt-in, follow the instructions in the dashboard.

## Scrape the metrics

Metrics are currently made available through [Prometheus](https://prometheus.io/) federation. Prometheus can be run with Docker, as a Kubernetes deployment, or as a standalone binary.

You can add a scrape target to your own Prometheus instance, or you can use the Grafana Agent to do that and ship off the metrics to Grafana Cloud.

Here is a sample scrape config for Prometheus:

```yaml
scrape_configs:
  - job_name: "actuated"

    bearer_token: "xyz"
    scheme: https
    metrics_path: /api/v1/metrics/federate

    # Note: this value cannot be changed lower than 60s due to rate-limiting
    scrape_interval: 60s
    scrape_timeout: 5s
    static_configs:
    - targets: ["tbc:443"]
```

The `bearer_token` is a secret, and unique per customer. Only a bcrypt hash is stored in the control-plane, along with a mapping between GitHub organisations and the token.

The `scrape_interval` must be `60s`, or higher to avoid rate-limiting.

Contact the support team on Slack for the value for the `targets` field.

### Control-plane metrics

Check the pending build queue depth:

```
actuated_controller_db_queued_status>0
```

Check for delayed VM launches (due to insufficient capacity):

```
sum by(owner) ( rate(actuated_controller_vm_launch_delayed_total
[$__rate_interval]) )>0
```

Failed jobs:

```
sum by (owner) (actuated_controller_job_failed_total{}) > 0
```

Rate of jobs queued:

```
sum by(owner) ( rate(actuated_controller_job_queued_total[$__rate_interval]) ) > 0   
```

Rate of VMs launched:

```
sum by(owner) (rate(actuated_controller_vm_launch_total [$__rate_interval]))>0
```

### Server metrics

VMs running by host:

```
sum by(job) (actuated_vm_running_gauge) > 0 
```

Free RAM by host:

```
actuated_system_memory_available_bytes{}
```

VM launch total:

```
sum by(job) ( actuated_vm_launch_total )
```

Load averages by host:

```
actuated_system_load_avg_1

actuated_system_load_avg_5

actuated_system_load_avg_15
```

Net I/O from egress adapter:

```
sum by( job) (  actuated_system_egress_rx )

sum by( job) (  actuated_system_egress_tx )
```
