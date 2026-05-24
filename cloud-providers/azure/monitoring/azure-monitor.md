# Azure Monitor

> Comprehensive monitoring solution for collecting, analyzing, and acting on telemetry from cloud and on-premises environments.

## Overview

Azure Monitor collects and aggregates data from every layer of your application stack—infrastructure, platform, and application. It provides insights through metrics, logs, and traces to help you understand performance and proactively identify issues.

## Key Concepts

| Term | Definition |
|------|------------|
| Metrics | Numerical time-series data (CPU, memory, etc.) |
| Logs | Event and trace data stored in Log Analytics |
| Alerts | Notifications based on conditions |
| Action Groups | Who to notify and what actions to take |
| Workbooks | Interactive reports and dashboards |
| Insights | Pre-built monitoring solutions |

## Architecture

```
+-------------------------------------------------------------------+
|                        Data Sources                               |
|  +---------+ +---------+ +---------+ +---------+ +----------+     |
|  |   VMs   | |   AKS   | |  Apps   | | Storage | | Custom   |     |
|  +----+----+ +----+----+ +----+----+ +----+----+ +----+-----+     |
+-------|-----------|-----------|-----------|-----------|-----------+
        |           |           |           |           |
        v           v           v           v           v
+-------------------------------------------------------------------+
|                       Azure Monitor                               |
|                                                                   |
|  +---------------------+    +-----------------------------+       |
|  |       Metrics       |    |          Logs               |       |
|  |   (Time-series DB)  |    |   (Log Analytics Workspace) |       |
|  |   - Platform metrics|    |   - Resource logs           |       |
|  |   - Custom metrics  |    |   - Activity logs           |       |
|  |   - Guest metrics   |    |   - Application logs        |       |
|  +---------+-----------+    +-------------+---------------+       |
|            |                              |                       |
|            +---------------+--------------+                       |
|                            v                                      |
|  +---------------------------------------------------------+      |
|  |                     Analyze & Act                       |      |
|  |  - Alerts   - Dashboards   - Workbooks   - Autoscale    |      |
|  +---------------------------------------------------------+      |
+-------------------------------------------------------------------+
```

## Metrics

### Types

| Type | Source | Retention |
|------|--------|-----------|
| **Platform Metrics** | Azure resources (auto) | 93 days |
| **Guest Metrics** | VM agent | 93 days |
| **Custom Metrics** | Your application | 93 days |
| **Prometheus Metrics** | AKS, containers | Configurable |

### Common Platform Metrics

| Resource | Key Metrics |
|----------|-------------|
| VMs | CPU %, Memory %, Disk IOPS |
| Storage | Transactions, Latency, Capacity |
| App Service | Requests, Response Time, Errors |
| SQL Database | DTU %, CPU %, Storage |
| AKS | Node CPU, Pod count, Network |

### Metrics Explorer

```
Query:
Resource: myVM
Metric: Percentage CPU
Aggregation: Average
Time range: Last 24 hours
Split by: Instance

Chart:
100% |
 75% |     /--\     /---\
 50% |  /--    \---/     \--
 25% |--                    --
  0% +----+----+----+----+----
     12:00  18:00  00:00  06:00
```

## Logs (Log Analytics)

### Data Types

| Table | Content |
|-------|---------|
| AzureActivity | Azure control plane operations |
| AzureDiagnostics | Resource diagnostic logs |
| AzureMetrics | Metrics (if routed to logs) |
| Heartbeat | Agent heartbeat |
| Perf | Performance counters |
| Event | Windows event logs |
| Syslog | Linux syslog |
| ContainerLog | Container stdout/stderr |
| AppTraces | Application Insights traces |

### KQL (Kusto Query Language)

```kusto
// Basic query
AzureActivity
| where TimeGenerated > ago(1h)
| where OperationName contains "Create"
| project TimeGenerated, Caller, OperationName, ResourceGroup

// Aggregation
Perf
| where CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart

// Join tables
Heartbeat
| where TimeGenerated > ago(1h)
| distinct Computer
| join kind=inner (
    Perf
    | where CounterName == "% Processor Time"
    | summarize AvgCPU = avg(CounterValue) by Computer
) on Computer

// Top N
AzureActivity
| where TimeGenerated > ago(7d)
| summarize count() by OperationName
| top 10 by count_
```

### Common KQL Operators

| Operator | Purpose |
|----------|---------|
| `where` | Filter rows |
| `project` | Select columns |
| `summarize` | Aggregate data |
| `extend` | Add calculated columns |
| `join` | Combine tables |
| `render` | Visualize results |
| `ago()` | Relative time |
| `bin()` | Time buckets |

## Alerts

### Alert Types

| Type | Trigger | Use Case |
|------|---------|----------|
| **Metric Alert** | Metric threshold | CPU > 80% |
| **Log Alert** | Log query results | Error count > 10 |
| **Activity Log Alert** | Azure operations | VM deleted |
| **Smart Detection** | ML-based anomalies | Performance degradation |

### Alert Rule Structure

```
Alert Rule:
|-- Scope: Which resource(s)
|-- Condition: When to fire
|   |-- Signal: Metric or Log query
|   |-- Threshold: Value to compare
|   +-- Frequency: How often to check
|-- Action Group: What to do
+-- Severity: 0 (Critical) to 4 (Verbose)
```

### Metric Alert Example

```bash
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group myRG \
  --scopes /subscriptions/.../virtualMachines/myVM \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action myActionGroup \
  --severity 2
```

### Log Alert Example (KQL)

```bash
az monitor scheduled-query create \
  --name "Error Alert" \
  --resource-group myRG \
  --scopes /subscriptions/.../workspaces/myWorkspace \
  --condition "count 'AzureDiagnostics | where Level == \"Error\"' > 10" \
  --window-size 15 \
  --evaluation-frequency 5 \
  --action-groups myActionGroup
```

## Action Groups

### Action Types

| Action | Description |
|--------|-------------|
| Email/SMS | Notify people |
| Azure Function | Run custom code |
| Logic App | Workflow automation |
| Webhook | HTTP callback |
| ITSM | ServiceNow, etc. |
| Runbook | Azure Automation |
| Event Hub | Stream alerts |

### Example

```bash
az monitor action-group create \
  --name myActionGroup \
  --resource-group myRG \
  --short-name myAG \
  --email admin admin@example.com \
  --sms oncall +15551234567 \
  --webhook alertwebhook https://example.com/alert
```

## Insights

### Pre-built Solutions

| Insight | Monitors |
|---------|----------|
| **VM Insights** | VM performance, dependencies |
| **Container Insights** | AKS, containers |
| **Application Insights** | Application performance |
| **Network Insights** | Network topology, metrics |
| **Storage Insights** | Storage account health |
| **SQL Insights** | SQL database performance |

### Application Insights

```
Application Insights:
|-- Performance
|   |-- Server response times
|   |-- Failed requests
|   +-- Dependency calls
|-- Usage
|   |-- Users and sessions
|   |-- Page views
|   +-- Custom events
|-- Failures
|   |-- Exceptions
|   +-- Failed dependencies
+-- Availability
    +-- URL ping tests
```

### Enabling VM Insights

```bash
# Enable for single VM
az vm extension set \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --vm-name myVM \
  --resource-group myRG

# Or use Azure Monitor Agent (AMA)
az monitor data-collection-rule create \
  --name myDCR \
  --resource-group myRG \
  --location eastus \
  --rule-file dcr.json
```

## Data Collection Rules (DCR)

Control what data is collected and where it goes.

```json
{
  "dataSources": {
    "performanceCounters": [
      {
        "name": "perfCounterDataSource",
        "streams": ["Microsoft-Perf"],
        "samplingFrequencyInSeconds": 60,
        "counterSpecifiers": [
          "\\Processor(_Total)\\% Processor Time",
          "\\Memory\\Available Bytes"
        ]
      }
    ],
    "windowsEventLogs": [
      {
        "name": "eventLogsDataSource",
        "streams": ["Microsoft-Event"],
        "xPathQueries": ["System!*[System[(Level=1 or Level=2)]]"]
      }
    ]
  },
  "destinations": {
    "logAnalytics": [
      {
        "name": "myWorkspace",
        "workspaceResourceId": "/subscriptions/.../workspaces/myLA"
      }
    ]
  }
}
```

## Autoscale

Scale resources based on metrics.

```bash
az monitor autoscale create \
  --name myAutoscale \
  --resource-group myRG \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --min-count 2 \
  --max-count 10 \
  --count 4

# Add scale-out rule
az monitor autoscale rule create \
  --autoscale-name myAutoscale \
  --resource-group myRG \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Add scale-in rule
az monitor autoscale rule create \
  --autoscale-name myAutoscale \
  --resource-group myRG \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1
```

## CLI Quick Reference

```bash
# List metrics for resource
az monitor metrics list \
  --resource /subscriptions/.../myVM \
  --metric "Percentage CPU" \
  --interval PT1H

# Query logs
az monitor log-analytics query \
  --workspace myWorkspace \
  --analytics-query "AzureActivity | take 10"

# Create alert
az monitor metrics alert create \
  --name "CPU Alert" \
  --resource-group myRG \
  --scopes /subscriptions/.../myVM \
  --condition "avg Percentage CPU > 80" \
  --action myActionGroup

# List alerts
az monitor alert list --resource-group myRG

# Create diagnostic setting
az monitor diagnostic-settings create \
  --name myDiagnostics \
  --resource /subscriptions/.../myResource \
  --workspace /subscriptions/.../myWorkspace \
  --logs '[{"category":"AuditLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

## Exam Tips (AZ-104, AZ-305)

1. **Metrics vs Logs**: Metrics = real-time, numeric; Logs = detailed, queryable
2. **Log Analytics**: Central repository for all log data
3. **KQL**: Query language for logs (similar to SQL)
4. **Action Groups**: Reusable notification/action configuration
5. **Alert severity**: 0 = Critical, 4 = Verbose
6. **Diagnostic settings**: Enable logs/metrics collection per resource
7. **Data Collection Rules**: Configure what agents collect
8. **VM Insights**: Requires Azure Monitor Agent
9. **Autoscale**: Based on metrics or schedule
10. **93 days**: Default metric retention

## Gotchas

- Platform metrics are free; custom metrics have cost
- Log Analytics has ingestion and retention costs
- Alert evaluation has minimum frequency limits
- Action groups have rate limits (1 SMS/5 min)
- Diagnostic settings must be enabled per resource
- Some metrics take time to appear (up to 3 min delay)
- KQL is case-sensitive for table/column names
- VM Insights requires Log Analytics workspace
- Alert state (fired/resolved) is separate from alert rules
- Legacy agents (MMA, Dependency Agent) being replaced by AMA

## Limits

| Resource | Limit |
|----------|-------|
| Metric alerts per subscription | 5,000 |
| Log alert rules per subscription | 1,000 |
| Action groups per subscription | 2,000 |
| Actions per action group | 10 of each type |
| Log Analytics query timeout | 10 minutes |
| Log Analytics query size | 64 MB |
| Custom metrics per resource | 10 dimensions |
| Metrics retention | 93 days |
| Log retention | 730 days (2 years) max |
