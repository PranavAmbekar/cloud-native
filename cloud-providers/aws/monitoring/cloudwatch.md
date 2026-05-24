# Amazon CloudWatch

> Monitoring and observability service for AWS resources and applications.

---

## Core Components

| Component | Purpose |
|-----------|---------|
| Metrics | Numerical data points over time |
| Logs | Text-based log data |
| Alarms | Automated actions on metric thresholds |
| Dashboards | Visualizations |
| Events/EventBridge | React to state changes |
| Insights | Query and analyze logs |
| Synthetics | Canary monitoring |

---

## Metrics

### Concepts
| Term | Definition |
|------|------------|
| Namespace | Container for metrics (e.g., AWS/EC2) |
| Metric | Time-ordered data points (e.g., CPUUtilization) |
| Dimension | Name/value pair to identify metric (e.g., InstanceId) |
| Statistic | Aggregation (Average, Sum, Min, Max, p99) |
| Period | Time granularity (1s to 1 day) |
| Resolution | Standard (1 min) or High (1 sec) |

### Common AWS Metrics

**EC2**
- CPUUtilization, NetworkIn/Out, DiskReadOps
- Note: Memory and disk space NOT included (need agent)

**RDS**
- CPUUtilization, DatabaseConnections, FreeStorageSpace

**Lambda**
- Invocations, Errors, Duration, Throttles

**ALB**
- RequestCount, TargetResponseTime, HTTPCode_Target_2XX

### Custom Metrics

```bash
# Put custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ActiveUsers" \
  --value 150 \
  --dimensions Environment=Production,Service=API
```

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp',
    MetricData=[
        {
            'MetricName': 'ProcessedOrders',
            'Value': 42,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': 'Environment', 'Value': 'Production'}
            ]
        }
    ]
)
```

### High Resolution Metrics
- Standard: 1-minute resolution
- High resolution: 1-second resolution
- Higher cost for high resolution

---

## CloudWatch Agent

Collect system-level metrics and logs from EC2.

### Metrics Collected
- Memory utilization
- Disk space
- CPU (detailed)
- Network (detailed)
- Process metrics

### Installation
```bash
# Download and install
sudo yum install amazon-cloudwatch-agent

# Configure (wizard)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Or use config file
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

### Agent Config Example
```json
{
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"]
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/"]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/myapp/*.log",
            "log_group_name": "/myapp/logs"
          }
        ]
      }
    }
  }
}
```

---

## CloudWatch Logs

### Structure
```
Log Group: /aws/lambda/my-function
+-- Log Stream: 2024/01/15/[$LATEST]abc123
|   +-- Log Event: timestamp + message
|   +-- Log Event: timestamp + message
|   +-- ...
+-- Log Stream: 2024/01/15/[$LATEST]def456
    +-- ...
```

### Retention
- Default: Never expire
- Configurable: 1 day to 10 years
- Set per log group

### Log Insights
Query language for analyzing logs.

```sql
-- Find errors
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

-- Stats by status code
fields @timestamp, status_code
| stats count(*) by status_code

-- P99 latency
fields @timestamp, latency
| stats pct(latency, 99) as p99_latency by bin(5m)

-- Parse JSON logs
fields @timestamp, @message
| parse @message '{"userId": *, "action": *}' as userId, action
| filter action = "login"
```

### Metric Filters
Extract metrics from log data.

```
Log Group -> Metric Filter -> CloudWatch Metric

Filter Pattern: [ERROR]
-> Creates metric: ErrorCount

Filter Pattern: { $.statusCode = 500 }
-> Creates metric: Server500Errors
```

### Subscription Filters
Stream logs to other services.

```
Log Group -> Subscription Filter -> Lambda / Kinesis / OpenSearch
```

### Export
- S3 (batch export)
- Takes up to 12 hours

### Live Tail
Real-time log viewing.

---

## CloudWatch Alarms

### States
| State | Description |
|-------|-------------|
| OK | Metric within threshold |
| ALARM | Metric breached threshold |
| INSUFFICIENT_DATA | Not enough data |

### Alarm Types

**Metric Alarm**
```
Metric: CPUUtilization
Condition: > 80%
Period: 5 minutes
Evaluation Periods: 3
-> Trigger if CPU > 80% for 3 consecutive 5-min periods
```

**Composite Alarm**
```
ALARM when:
  (Alarm1 = ALARM) AND (Alarm2 = ALARM)
OR
  Alarm3 = ALARM
```

### Actions
| Target | Use Case |
|--------|----------|
| SNS | Notifications |
| Auto Scaling | Scale EC2 |
| EC2 | Stop, terminate, reboot |
| Lambda | Custom actions |
| Systems Manager | Run automation |

### Example
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-xxx \
  --alarm-actions arn:aws:sns:us-east-1:xxx:alerts
```

---

## CloudWatch Dashboards

### Widgets
- Line graph
- Stacked area
- Number
- Gauge
- Text
- Logs
- Alarms

### Cross-Account/Region
- View metrics from multiple accounts
- View metrics from multiple regions

### Automatic Dashboards
- Pre-built dashboards per service
- Container Insights dashboards

---

## Container Insights

Metrics for containers.

```
ECS/EKS -> Container Insights -> CloudWatch
                                    |
                              +-- Cluster metrics
                              +-- Service metrics
                              +-- Task/Pod metrics
                              +-- Container metrics
```

Metrics:
- CPU, Memory utilization
- Network I/O
- Storage I/O
- Container restarts

---

## Lambda Insights

Enhanced monitoring for Lambda.

Metrics:
- Memory usage
- CPU time
- Network, disk I/O
- Cold starts
- Init duration

---

## Synthetics (Canaries)

Automated tests for endpoints.

```python
# Canary script
def handler(event, context):
    # Check website availability
    response = requests.get('https://example.com/health')
    assert response.status_code == 200

    # Check API response time
    start = time.time()
    response = requests.get('https://api.example.com/status')
    latency = time.time() - start
    assert latency < 2
```

Features:
- Run every 1-60 minutes
- Visual monitoring (screenshots)
- HAR files
- Alarms on failures

---

## Anomaly Detection

ML-based anomaly detection for metrics.

```
Historical data -> ML model -> Expected band
                                  |
Current value outside band -> Alarm
```

```bash
aws cloudwatch put-anomaly-detector \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxx \
  --stat Average
```

---

## Contributor Insights

Analyze high-cardinality data.

```
VPC Flow Logs -> Contributor Insights -> Top talkers
CloudTrail -> Contributor Insights -> Top API callers
```

---

## CLI Quick Reference

```bash
# Get metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxx \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average

# List metrics
aws cloudwatch list-metrics --namespace AWS/Lambda

# Put metric data
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name Errors \
  --value 1

# Create log group
aws logs create-log-group --log-group-name /myapp/logs

# Put log events
aws logs put-log-events \
  --log-group-name /myapp/logs \
  --log-stream-name stream1 \
  --log-events timestamp=$(date +%s000),message="Hello"

# Query logs (Insights)
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | limit 10'

# Describe alarms
aws cloudwatch describe-alarms

# Set alarm state (testing)
aws cloudwatch set-alarm-state \
  --alarm-name "HighCPU" \
  --state-value ALARM \
  --state-reason "Testing"
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Metrics (first 10K) | Free |
| Metrics (10K-250K) | $0.30/metric/month |
| Custom metrics | $0.30/metric/month |
| High resolution | $0.30/metric/month |
| Dashboards | $3/dashboard/month |
| Alarms | $0.10/alarm/month |
| Logs ingested | $0.50/GB |
| Logs stored | $0.03/GB/month |
| Logs Insights | $0.005/GB scanned |
| Canary runs | $0.0012/run |

---

## Exam Tips

1. **Basic metrics** - 5-minute intervals, free
2. **Detailed metrics** - 1-minute intervals, extra cost
3. **Custom metrics** - high resolution up to 1-second
4. **CloudWatch Agent** - required for memory, disk metrics
5. **Metric filters** - extract metrics from logs
6. **Log Insights** - query logs with SQL-like syntax
7. **Composite alarms** - combine multiple alarms
8. **Anomaly detection** - ML-based, creates bands
9. **Subscription filters** - stream logs to Lambda/Kinesis
10. **Unified CloudWatch Agent** - both metrics and logs
11. **Cross-account dashboards** - centralized monitoring
12. **Container Insights** - ECS/EKS metrics
