---
title: "Cloud Network Supervision on GCP: From Flow Logs to BigQuery"
date: 2026-04-20 14:00:00 +0100
categories: [Projects, Cloud]
tags: [gcp, cloud, networking, bigquery, vpc-flow-logs, cloud-monitoring, looker-studio, observability]
---

# Summary

For my Cloud Computing course I built a full network supervision pipeline on Google Cloud: a multi-region virtual network, monitored in real time, with every network connection logged, exported to BigQuery, and analyzed with SQL. The scenario is a company moving its network monitoring to the cloud across three continents.

The whole thing ran on GCP's free tier plus student credits, and the total bill came to **less than one cent** — which is itself part of the point: a complete observability stack that would be serious infrastructure on-prem costs almost nothing when you build it from managed cloud services and shut it off when you're done.

The pipeline, end to end:

- **Infrastructure** — a custom VPC with three subnets, one each in the US, Europe and Asia, and an `e2-micro` VM in each.
- **Real-time monitoring** — the [Cloud Ops Agent](https://cloud.google.com/monitoring/agent/ops-agent) on every VM feeding [Cloud Monitoring](https://cloud.google.com/monitoring) and [Cloud Logging](https://cloud.google.com/logging).
- **Traffic analysis** — [VPC Flow Logs](https://cloud.google.com/vpc/docs/flow-logs) exported to [BigQuery](https://cloud.google.com/bigquery) and queried with SQL.
- **Alerting** — Cloud Monitoring policies on CPU and bandwidth, with email notifications.
- **Visualization** — a [Looker Studio](https://lookerstudio.google.com/) dashboard reading straight from BigQuery.

Everything was done from Cloud Shell with the `gcloud` and `bq` CLIs.

# The infrastructure

A custom-mode VPC, `supervision-vpc`, with three regional subnets so I had full control over the address ranges:

| Subnet | Region | IP range | Flow Logs |
|---|---|---|---|
| subnet-us | us-west1 | 10.0.1.0/24 | Enabled |
| subnet-eu | europe-west1 | 10.0.2.0/24 | Enabled |
| subnet-asia | asia-east1 | 10.0.3.0/24 | Enabled |

```bash
gcloud compute networks subnets create subnet-us \
  --network=supervision-vpc --region=us-west1 \
  --range=10.0.1.0/24 --enable-flow-logs
```

That `--enable-flow-logs` flag is the one that makes the whole analysis half of the project possible: it tells GCP to record metadata for every connection on the subnet — source, destination, port, protocol, bytes, timestamps.

Then one `e2-micro` Debian VM per subnet, and three firewall rules for SSH, HTTP and ICMP. I'll flag now that those rules allow `0.0.0.0/0` — fine for a throwaway lab, not fine for anything real, and I come back to it in the audit.

# Real-time monitoring

The Cloud Ops Agent went on all three VMs with a two-line install. It's the unified agent that collects both metrics and logs and ships them to Cloud Monitoring and Cloud Logging:

```bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

All three agents passed their startup checks (ports, network, API), and I built a Cloud Monitoring dashboard with CPU utilization, bytes sent, and a memory gauge. Combined memory across the three VMs sat at about **356 MiB** — worth noting, because the agent's own footprint is a real cost on an `e2-micro`.

# The interesting half: Flow Logs in BigQuery

Real-time dashboards tell you what's happening now. To ask questions about history, I piped the flow logs into BigQuery.

A dataset to hold them, then a **log sink** that routes every VPC Flow Log entry from Cloud Logging into that dataset automatically:

```bash
bq mk --dataset --location=US network-supervision:network_logs

gcloud logging sinks create vpc-flow-to-bigquery \
  bigquery.googleapis.com/projects/network-supervision/datasets/network_logs \
  --log-filter='resource.type="gce_subnetwork" AND log_id("compute.googleapis.com/vpc_flows")'
```

BigQuery auto-creates a table from the logs' schema, and from then on every connection on the network becomes a queryable row. I generated some traffic with `ping` and `curl` between the VMs to populate it, which also gave me the cross-region latencies: **~156 ms US→EU** and **~125 ms US→Asia**, both at 0% packet loss.

Then the actual analysis, in SQL. A few queries that stood out:

**Most-used destination ports:**

| Port | Protocol | Connections | What it is |
|---|---|---|---|
| 443 | TCP | 96 | HTTPS — the Ops Agent talking to GCP APIs |
| 22 | TCP | 61 | SSH — my admin sessions |
| 45684 | TCP | 23 | ephemeral outbound |

The thing I like about this result is that the pipeline caught itself. The single biggest source of traffic on the network wasn't my test pings — it was the monitoring agents phoning home to Google's APIs over HTTPS. The longest-duration sessions (around 4 seconds) were the same Ops Agent heartbeats. The observability layer is itself the heaviest talker on the network, and the flow logs make that visible.

Over one day the network produced **311 flow log entries**, with `vm-us` the most active node at 121 of them.

# Alerting

Two Cloud Monitoring alert policies:

- **High CPU** — fires when any VM's CPU goes above 80% for two minutes.
- **High bandwidth** — fires when sent bytes exceed 1 GB/s.

The CPU alert actually triggered during the VM setup phase (installing packages spiked CPU), and the email landed the next day. The notification was delayed by a few hours, which is its own small lesson: Cloud Monitoring email alerts are not instant, so they're for "something's been wrong for a while," not for anything you need to react to in seconds.

# Visualization

A Looker Studio dashboard connected directly to the BigQuery `network_logs` dataset: a time series of connection volume by hour, a bar chart of destination ports, a table of top source IPs, and a scorecard of total flow entries. Because it reads BigQuery live, it updates as new logs arrive rather than being a static snapshot.

# The audit — what I'd fix

Part of the project was auditing what I'd built. The honest findings:

- **SSH open to the world.** The `allow-ssh` rule permits `0.0.0.0/0` on port 22. The first real fix is to scope that to known IP ranges. (HTTP and ICMP are wide open too.)
- **Flow Logs at 100% sampling.** Fine for a lab generating 311 entries a day; on a busy network that's a lot of BigQuery ingestion. Production would sample at something like 50%.
- **Broad IAM.** The BigQuery dataset inherited project-level editor access. Least privilege means scoping it to the analysts who actually need it.
- **No log archival.** For compliance you'd sink older flow logs to Cloud Storage beyond BigQuery's retention.
- **Agent overhead on e2-micro.** ~356 MiB for monitoring is a meaningful slice of an `e2-micro`; `e2-small` would give it room.
- A Web Application Firewall (Cloud Armor) in front of the HTTP-exposed services.

# What I took from it

- **Managed services make serious observability cheap.** A metrics pipeline, a log-analytics warehouse and a BI dashboard, for under a cent, because you only pay for what you use and you tear it down after.
- **Flow Logs + BigQuery is a genuinely powerful combination.** Turning every network connection into a SQL-queryable row lets you ask questions a dashboard can't answer.
- **A monitored system's own telemetry is part of its traffic.** Most of what this network carried was the monitoring talking to Google. Worth remembering when you read any observability data: some of what you see is the observer.

*Cloud Computing & Network Services course project, IT Business School, April 2026.*
