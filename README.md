# Purpose

Prometheus is awesome, but the human mind doesn't work in PromQL. The intention of this repository is to become a simple place for people to provide examples of queries they've found useful.
We encourage all to contribute so that this can become something valuable to the community.

Simple or complex, all input is welcome.

## Further Reading

* [Prometheus Main Site](https://prometheus.io/)
* [Prometheus Docs](https://prometheus.io/docs/introduction/overview/)
* [Prometheus Alert Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
* [Robust Perception](https://www.robustperception.io/blog/)



# PromQL Examples

Please ensure all examples are submitted in the same format, we'd like to keep this nice and easy to read and maintain.
The examples may contain some metric names and labels that aren't present on your system, if you're looking to re-use these then make sure validate the labels and metric names match your system.

---

**Show Overall CPU usage for a server**
```
100 * (1 - avg by(instance)(irate(node_cpu{mode='idle'}[5m])))
```
*Summary:* Often useful to newcomers to Prometheus looking to replicate common host CPU checks. This query ultimately provides an overall metric for CPU usage, per instance. It does this by a calculation based on the `idle` metric of the CPU, working out the overall percentage of the other states for a CPU in a 5 minute window and presenting that data per `instance`.

---

**Track http error rates as a proportion of total traffic**
```
 rate(demo_api_request_duration_seconds_count{status="500",job="demo"}[5m]) * 50
> on(job, instance, method, path)
    rate(demo_api_request_duration_seconds_count{status="200",job="demo"}[5m])
```
*Summary:* This query selects the 500-status rate for any job, instance, method, and path combinations for which the 200-status rate is not at least 50 times higher than the 500-status rate. The rate function has been used here as it's designed to be used with the counters in this query.

*link:* [Julius Volz - Tutorial](https://www.digitalocean.com/community/tutorials/how-to-query-prometheus-on-ubuntu-14-04-part-2)

---

**90th Percentile latency**
```
 histogram_quantile(0.9, rate(demo_api_request_duration_seconds_bucket{job="demo"}[5m])) > 0.05
and
    rate(demo_api_request_duration_seconds_count{job="demo"}[5m]) > 1
```
*Summary:*  Select any HTTP endpoints that have a 90th percentile latency higher than 50ms (0.05s) but only for the dimensional combinations that receive more than one request per second. We use the `histogram_quantile()` function for the percentile calculation here. It calculates the 90th percentile latency for each sub-dimension. To filter the resulting bad latencies and retain only those that receive more than one request per second. `histogram_quantile` is only suitable for usage with a Histogram metric.

*link:* [Julius Volz - Tutorial](https://www.digitalocean.com/community/tutorials/how-to-query-prometheus-on-ubuntu-14-04-part-2)

---

**HTTP request rate, per second.. an hour ago**
```
rate(api_http_requests_total{status=500}[5m] offset 1h)
```

*Summary:*  The `rate()` function calculates the per-second average rate of time series in a range vector. Combining all the above tools, we can get the rates of HTTP requests of a specific timeframe. The query calculates the per-second rates of all HTTP requests that occurred in the last 5 minutes, an hour ago. Suitable for usage on a `counter` metric.

*Link:* [Tom Verelst - Ordina](https://ordina-jworks.github.io/monitoring/2016/09/23/Monitoring-with-Prometheus.html)

---

**Kubernetes Container Memory Usage**
```
sum by(kubernetes_pod_name) (container_memory_usage_bytes{kubernetes_namespace="kube-system"})
```

*Summary:* How much memory are the tools in the kube-system namespace using? Break it down by Pod and NameSpace!

*Link:* [Joe Bowers - CoreOS](https://coreos.com/blog/monitoring-kubernetes-with-prometheus.html)

---

**Most expensive time series**
```
topk(10, count by (__name__)({__name__=~".+"}))
```

*Summary:* Which are your most expensive time series to store? When tuning Prometheus, these quries can help you monitor your most expensive metrics. Be cautious, this query is expensive to run.

*Link:* [Brian Brazil - Robust Perception](https://www.robustperception.io/which-are-my-biggest-metrics/)

---

**Most expensive time series**
```
topk(10, count by (job)({__name__=~".+"}))
```

*Summary:* Which of your jobs have the most timeseries? Be cautious, this query is expensive to run.

*Link:* [Brian Brazil - Robust Perception](https://www.robustperception.io/which-are-my-biggest-metrics/)

---

**Which Alerts have been firing?**
```
sum(sort_desc(sum_over_time(ALERTS{alertstate=`firing`}[24h]))) by (alertname)
```

*Summary:* Which of your Alerts have been firing the most? Useful to track alert trends.

---


# Alert Rules Examples

These are examples of rules you can use with Prometheus to trigger the firing of an event, usually to the Prometheus alertmanager application. You can refer to the [official documentation](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) for more information.

```yaml
- alert: <alert name>
  expr: <expression>
  for: <duration>
  labels:
    label_name: <label value>
  annotations:
    annotation_name: <annotation value>
```

**Disk Will Fill in 4 Hours**
```yaml
- alert: PreditciveHostDiskSpace
  expr: predict_linear(node_filesystem_free{mountpoint="/"}[4h], 4 * 3600) < 0
  for: 30m
  labels:
    severity: warning
  annotations:
    description: 'Based on recent sampling, the disk is likely to will fill on volume
      {{ $labels.mountpoint }} within the next 4 hours for instace: {{ $labels.instance_id
      }} tagged as: {{ $labels.instance_name_tag }}'
    summary: Predictive Disk Space Utilisation Alert
```
*Summary:* Asks Prometheus to predict if the hosts disks will fill within four hours, based upon the last hour of sampled data. In this example, we are returning AWS EC2 specific labels to make the alert more readable.

---

**Alert on High Memory Load**
```yaml
- expr: (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers + node_memory_Cached) ) / sum(node_memory_MemTotal) * 100 > 85
```
*Summary:* Trigger an alert if the memory of a host is almost full. This is done by deducting the total memory by the free, buffered and cached memory and dividing it by total again to obtain a percentage. The `> 85` will only return when the resulting value is above 85.

*Link:* [Stefan Prodan - Blog](https://stefanprodan.com/2016/a-monitoring-solution-for-docker-hosts-containers-and-containerized-services/)

---

**Alert on High CPU utilisation**
```yaml
- alert: HostCPUUtilisation
  expr: 100 - (avg by(instance) (irate(node_cpu{mode="idle"}[5m])) * 100) > 70
  for: 20m
  labels:
    severity: warning
  annotations:
    description: 'High CPU utilisation detected for instance {{ $labels.instance_id
      }} tagged as: {{ $labels.instance_name_tag }}, the utilisation is currently:
      {{ $value }}%'
    summary: CPU Utilisation Alert
```
*Summary:* Trigger an alert if a host's CPU becomes over 70% utilised for 20 minutes or more.

---

**Alert if Prometheus is throttling**
```yaml
- alert: PrometheusIngestionThrottling
  expr: prometheus_local_storage_persistence_urgency_score > 0.95
  for: 1m
  labels:
    severity: warning
  annotations:
    description: Prometheus cannot persist chunks to disk fast enough. It's urgency
      value is {{$value}}.
    summary: Prometheus is (or borderline) throttling ingestion of metrics
```
*Summary:* Trigger an alert if Prometheus begins to throttle its ingestion. If you see this, some TLC is required.

---
