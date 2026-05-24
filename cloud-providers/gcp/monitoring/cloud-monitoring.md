# Google Cloud Monitoring (Operations Suite)

> Full-stack monitoring for applications and infrastructure on Google Cloud.

## Overview

Cloud Monitoring collects metrics, events, and metadata from Google Cloud, AWS, hosted uptime probes, and application instrumentation. It provides dashboards, alerting, and insights to help you understand the health and performance of your applications.

## Key Concepts

| Term | Definition |
|------|------------|
| Metric | Time-series measurement (CPU, memory, etc.) |
| Resource | Entity being monitored (VM, database, etc.) |
| Dashboard | Visual display of metrics |
| Alert Policy | Condition-based notifications |
| Uptime Check | External availability monitoring |
| Service | Logical grouping for SLIs/SLOs |
| SLI | Service Level Indicator |
| SLO | Service Level Objective |

## Architecture

```
+---------------------------------------------------------------+
|                      Cloud Monitoring                         |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                     Data Sources                        |  |
|  |  +---------+ +---------+ +---------+ +---------+        |  |
|  |  |   GCE   | |   GKE   | |  Cloud  | |  Custom |        |  |
|  |  |         | |         | |   SQL   | | Metrics |        |  |
|  |  +---------+ +---------+ +---------+ +---------+        |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                              v                                |
|  +---------------------------------------------------------+  |
|  |                    Metrics & Logs                       |  |
|  |  +------------------+  +----------------------------+   |  |
|  |  |  Metrics Store   |  |      Cloud Logging         |   |  |
|  |  |  (Time Series)   |  |    (Log-based metrics)     |   |  |
|  |  +------------------+  +----------------------------+   |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                              v                                |
|  +---------------------------------------------------------+  |
|  |                  Analysis & Actions                     |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  |  |Dashboards| |  Alerts  | |  Uptime  | |   SLOs   |    |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## Metrics

### Metric Types

| Type | Description |
|------|-------------|
| **Built-in** | Automatic from GCP services |
| **Agent** | From monitoring agent |
| **Custom** | User-defined metrics |
| **Log-based** | Derived from logs |

### Common Metrics

| Service | Key Metrics |
|---------|-------------|
| Compute Engine | CPU, Memory, Disk, Network |
| GKE | Container CPU/Memory, Node status |
| Cloud SQL | CPU, Memory, Connections, Queries |
| Cloud Storage | Request count, Latency |
| Cloud Functions | Execution count, Duration, Errors |

### Metrics Explorer

```
Metric Query:
+-- Resource Type: gce_instance
+-- Metric: compute.googleapis.com/instance/cpu/utilization
+-- Filter: zone = "us-central1-a"
+-- Aggregation: mean
+-- Group By: instance_name
```

### Custom Metrics

```python
from google.cloud import monitoring_v3

client = monitoring_v3.MetricServiceClient()
project_name = f"projects/my-project"

# Create time series
series = monitoring_v3.TimeSeries()
series.metric.type = "custom.googleapis.com/my_metric"
series.metric.labels["environment"] = "production"
series.resource.type = "global"

# Add data point
now = time.time()
interval = monitoring_v3.TimeInterval(
    {"end_time": {"seconds": int(now)}}
)
point = monitoring_v3.Point({
    "interval": interval,
    "value": {"double_value": 42.0}
})
series.points = [point]

# Write
client.create_time_series(name=project_name, time_series=[series])
```

## Dashboards

### Create Dashboard (JSON)

```json
{
  "displayName": "My Dashboard",
  "gridLayout": {
    "columns": 2,
    "widgets": [
      {
        "title": "CPU Utilization",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
                "aggregation": {
                  "perSeriesAligner": "ALIGN_MEAN",
                  "crossSeriesReducer": "REDUCE_MEAN",
                  "groupByFields": ["resource.label.instance_id"]
                }
              }
            }
          }]
        }
      },
      {
        "title": "Memory Usage",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"agent.googleapis.com/memory/percent_used\""
              }
            }
          }]
        }
      }
    ]
  }
}
```

```bash
gcloud monitoring dashboards create --config-from-file=dashboard.json
```

## Alert Policies

### Conditions

| Type | Description |
|------|-------------|
| Metric threshold | Metric above/below value |
| Metric absence | No data received |
| Log-based | Log entries matching filter |
| Forecast | Predicted future value |

### Create Alert Policy

```bash
# Create notification channel first
gcloud beta monitoring channels create \
  --display-name="Email Channel" \
  --type=email \
  --channel-labels=email_address=admin@example.com

# Create alert policy
gcloud alpha monitoring policies create \
  --policy-from-file=alert-policy.json
```

```json
{
  "displayName": "High CPU Alert",
  "conditions": [{
    "displayName": "CPU > 80%",
    "conditionThreshold": {
      "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" AND resource.type=\"gce_instance\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.8,
      "duration": "300s",
      "aggregations": [{
        "alignmentPeriod": "60s",
        "perSeriesAligner": "ALIGN_MEAN"
      }]
    }
  }],
  "notificationChannels": ["projects/my-project/notificationChannels/123"],
  "alertStrategy": {
    "autoClose": "1800s"
  }
}
```

### Notification Channels

| Channel | Description |
|---------|-------------|
| Email | Email notifications |
| SMS | Text messages |
| Slack | Slack channel messages |
| PagerDuty | Incident management |
| Webhook | HTTP callbacks |
| Pub/Sub | Publish to topic |

## Uptime Checks

### Create Uptime Check

```bash
gcloud monitoring uptime create my-uptime-check \
  --display-name="Website Check" \
  --http \
  --uri="https://example.com" \
  --check-interval=60 \
  --timeout=10 \
  --selected-regions=usa,europe-west1
```

### Check Types

| Type | Description |
|------|-------------|
| HTTP(S) | Check URL response |
| TCP | Check port availability |
| Custom | Custom protocol checks |

## SLIs and SLOs

### Create SLO

```bash
# Create service
gcloud monitoring services create my-service \
  --display-name="My Service"

# Create SLO
gcloud monitoring slos create \
  --service=my-service \
  --display-name="Availability SLO" \
  --goal=0.999 \
  --rolling-period-days=28 \
  --request-based-sli='{
    "goodTotalRatio": {
      "goodServiceFilter": "metric.type=\"loadbalancing.googleapis.com/https/request_count\" resource.type=\"https_lb_rule\" metric.label.response_code_class=\"200\"",
      "totalServiceFilter": "metric.type=\"loadbalancing.googleapis.com/https/request_count\" resource.type=\"https_lb_rule\""
    }
  }'
```

### SLI Types

| Type | Description |
|------|-------------|
| Request-based | Good requests / total requests |
| Windows-based | Good time windows / total windows |

## Cloud Logging Integration

### Log-Based Metrics

```bash
# Create counter metric
gcloud logging metrics create error-count \
  --description="Count of error logs" \
  --log-filter="severity>=ERROR"

# Create distribution metric
gcloud logging metrics create latency \
  --description="Request latency" \
  --log-filter="resource.type=gae_app" \
  --value-extractor='EXTRACT(jsonPayload.latency)'
```

## Ops Agent

Install on VMs for detailed metrics and logging.

```bash
# Install Ops Agent
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Configure
cat > /etc/google-cloud-ops-agent/config.yaml << EOF
logging:
  receivers:
    myapp_logs:
      type: files
      include_paths:
        - /var/log/myapp/*.log
  service:
    pipelines:
      myapp:
        receivers: [myapp_logs]
metrics:
  receivers:
    nginx:
      type: nginx
      stub_status_url: http://localhost/status
  service:
    pipelines:
      nginx:
        receivers: [nginx]
EOF

sudo systemctl restart google-cloud-ops-agent
```

## CLI Quick Reference

```bash
# List metrics
gcloud monitoring metrics list --filter="metric.type:compute.googleapis.com"

# List dashboards
gcloud monitoring dashboards list

# Describe dashboard
gcloud monitoring dashboards describe DASHBOARD_ID

# List alert policies
gcloud alpha monitoring policies list

# Describe alert policy
gcloud alpha monitoring policies describe POLICY_ID

# List uptime checks
gcloud monitoring uptime list-configs

# List notification channels
gcloud beta monitoring channels list

# Query metrics (MQL)
gcloud monitoring query \
  --filter='
    fetch gce_instance
    | metric "compute.googleapis.com/instance/cpu/utilization"
    | filter zone = "us-central1-a"
    | group_by 5m, [mean(value.utilization)]'
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Metrics scope**: Can include multiple projects
2. **Built-in metrics**: Free, automatic collection
3. **Custom metrics**: Charged per data point
4. **Ops Agent**: For detailed VM metrics and logs
5. **Alert conditions**: Can use MQL or filters
6. **Uptime checks**: Global, probe from multiple regions
7. **SLOs**: Define reliability targets
8. **Log-based metrics**: Derive metrics from logs
9. **Notification channels**: Multiple types available
10. **Dashboards**: Shareable, version controlled

## Gotchas

- Metrics have different granularities
- Custom metrics incur charges
- Alert policies require notification channels
- Log-based metrics have ingestion delay
- Uptime checks count as ingress
- Ops Agent replaces legacy agents
- Dashboard widgets have size limits
- MQL has learning curve
- SLO burn rate alerts for error budgets
- Cross-project monitoring needs metrics scope

## Limits

| Resource | Limit |
|----------|-------|
| Custom metrics per project | 10,000 |
| Labels per metric | 30 |
| Label key length | 100 characters |
| Label value length | 1024 characters |
| Time series per write | 200 |
| Alert policies per project | 500 |
| Conditions per policy | 6 |
| Notification channels per project | 4,000 |
| Uptime checks per project | 100 |
| Dashboards per project | 1,000 |
| Widgets per dashboard | 40 |
