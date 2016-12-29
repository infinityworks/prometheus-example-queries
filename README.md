# Purpose

Prometheus is awesome, but the human mind doesn't work in PromQL. The intention of this repository is to become a simple place for people to provide examples of queries they've found useful.
We encourage all to contribute so that this can become something valuable to the community.

Simple or complex, all input is welcome.

## Further Reading

* [Prometheus Main Site](https://prometheus.io/)
* [Prometheus Docs](https://prometheus.io/docs/introduction/overview/)
* [Prometheus Alert Rules](https://prometheus.io/docs/alerting/rules/)
* [Robust Perception](https://www.robustperception.io/blog/)



# PromQL Examples

Please ensure all examples are submitted in the same format, we'd like to keep this nice and easy to read and maintain.
The examples may contain some metric names and labels that aren't present on your system, if your looking to re-use these then make sure validate the labels and metric names match your system.

---

**Show Overall CPU usage for a server**
```
100 * (1 - avg by(instance)(irate(node_cpu{mode='idle'}[5m])))
```
*Summary:* Often useful to newcomers to Prometheus looking to replicate common host CPU checks. This query ultimately provides a overall metric for CPU usage, per instance. It does this by calculation based on the `idle` metric of the CPU, working out the overall percentage of the other states for a CPU in a 5 minute window and presenting that data per `instance`.

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

*Summary:*  The `rate()` function calculates the per-second average rate of time series in a range vector. Combining all the above tools, we can get the rates of HTTP requests of a specific timeframe. The query calculates the per-second rates of all HTTP requests that occurred in the last 5 minutes,  an hour ago. Suitable for usage on a `counter` metric.

*Link:* [Tom Verelst - Ordina](https://ordina-jworks.github.io/monitoring/2016/09/23/Monitoring-with-Prometheus.html)

---

**Kubernetes Container Memory Usage**
```
sum by(kubernetes_pod_name) (container_memory_usage_bytes{kubernetes_namespace="kube-system"})
```

*Summary:* How much memory are the tools in the kube-system namespace using? Break it down by Pod and NameSpace!

*Link:* [Joe Bowers - CoreOS](https://coreos.com/blog/monitoring-kubernetes-with-prometheus.html)


# Alert Rules Examples

These are examples of rules you can use with Prometheus to trigger the firing of an event, usually to the Prometheus alertmanager application.
Each alert's are usually defined with the syntax below, in the examples we just highlight the query section.

```
ALERT <alert name>
  IF <expression>
  [ FOR <duration> ]
  [ LABELS <label set> ]
  [ ANNOTATIONS <label set> ]
``` 

**Disk Will Fill in 4 Hours**
```
predict_linear(node_filesystem_free[1h], 4*3600)
```
*Summary:* Asks Prometheus to predict if the hosts disks will fill within four hours, based upon the last hour of sampled data.

*Link:* [Mônica Ribeiro - Medium](https://medium.com/quick-mobile/monitoring-containers-with-prometheus-ffde286c17f7#.87umnk8zv)

---

**Alert on High Memory Load**
```
  IF (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers + node_memory_Cached) ) / sum(node_memory_MemTotal) * 100 > 85
```
*Summary:* Trigger an alert if a host memory is almost full. This is done by deducting the total memory by the free, buffered and cached memory and dividing it by total again to obtain a percentage. The `> 85` then only returns when the resulting value is above 85.

*Link:* [Stefan Prodan - Blog](https://stefanprodan.com/2016/a-monitoring-solution-for-docker-hosts-containers-and-containerized-services/)

---

