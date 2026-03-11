## Prometheus and blackbox validation

Important blackbox rule

For blackbox jobs, do not use **up** to judge service health.

* `up{job="blackbox_http"}` only means Prometheus can scrape blackbox exporter.
* `probe_success{job="blackbox_http"}` tells you whether the HTTP probe itself succeeded.

### Correct queries

Open Prometheus:

http://127.0.0.1:9090/graph

Use these queries:

```promql
probe_success{job="blackbox_dns"}
probe_success{job="blackbox_http"}
probe_success{job="blackbox_tcp"}
up{job="snmp_iosv"}
```

Healthy baseline:

* DNS = 1
* HTTP = 1
* TCP = 1
* SNMP targets up

### Blackbox debug URLs

These are the fastest way to test service checks:

```text
http://127.0.0.1:9115/probe?target=10.20.10.10:53&module=dns_tcp_lab&debug=true
http://127.0.0.1:9115/probe?target=http://10.20.10.10/&module=http_2xx_lab&debug=true
http://127.0.0.1:9115/probe?target=10.20.10.10:80&module=tcp_80_lab&debug=true
```


Look for:

* probe_success 1

or

* probe_success 0

---

## Why ICMP blackbox is not used as a primary signal

Windows blackbox ICMP was noisy / unreliable in this lab. The project kept:

* DNS
* HTTP
* TCP/80
* SNMP
* syslog
* and de-emphasized Windows ICMP blackbox.

---

## Loki / syslog validation

### Important design note

Loki in this project only ingests router syslog.

It does not ingest Linux host logs from S1-SVC unless you explicitly add that later.

That means:

* use Loki for router events (LINK, LINEPROTO, OSPF, BGP)
* use Prometheus blackbox for nginx / dnsmasq service failures

### Validate syslog path

On R2-BRANCH:

```cisco
conf t
interface Ethernet0/2
 shutdown
 no shutdown
end
```

Then in Grafana Explore with Loki:

```promql
{job="syslog", source="cml"}
```

Then narrow:

```prompql
{job="syslog", source="cml"} |= `Ethernet0/2`
{job="syslog", source="cml"} |= `UPDOWN`
{job="syslog", source="cml"} |= `LINEPROTO`
```

Why Alloy uses raw syslog ingestion

Alloy's RFC3164 Cisco parsing produced parser errors like:

expecting a sequence number

The final working solution was to use:

* syslog_format = "raw"
* --stability.level=experimental

That reliably ingested Cisco router syslog into Loki.