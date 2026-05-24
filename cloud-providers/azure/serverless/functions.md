# Azure Functions

> Event-driven serverless compute platform. Run code without managing infrastructure.

## Overview

Azure Functions lets you run event-triggered code without provisioning or managing servers. Write functions in your preferred language, and Azure handles scaling, hosting, and infrastructure.

## Key Concepts

| Term | Definition |
|------|------------|
| Function App | Container for one or more functions (deployment unit) |
| Function | Individual piece of code triggered by an event |
| Trigger | Event that causes a function to run |
| Binding | Declarative connection to data (input/output) |
| Host | Runtime that executes functions |
| Plan | Hosting option (Consumption, Premium, Dedicated) |

## Supported Languages

| Language | Version | Process Model |
|----------|---------|---------------|
| C# | .NET 6, 7, 8 | In-process or Isolated |
| JavaScript/TypeScript | Node.js 18, 20 | Isolated |
| Python | 3.9, 3.10, 3.11 | In-process |
| Java | 8, 11, 17, 21 | Isolated |
| PowerShell | 7.2, 7.4 | In-process |
| Custom | Any | Custom handler |

## Hosting Plans

| Plan | Scaling | Max Instances | Timeout | Use Case |
|------|---------|---------------|---------|----------|
| **Consumption** | Auto (event-driven) | 200 | 5 min (default), 10 min max | Sporadic workloads |
| **Premium** | Auto (pre-warmed) | 100 (Windows), 20-100 (Linux) | 60 min | No cold starts, VNet |
| **Dedicated (App Service)** | Manual/Auto | 10-30 | Unlimited | Existing App Service |
| **Container Apps** | Auto (KEDA) | 300 | Unlimited | Containerized functions |

### Plan Comparison

| Feature | Consumption | Premium | Dedicated |
|---------|-------------|---------|-----------|
| Cold starts | Yes | No (pre-warmed) | No |
| VNet integration | No | Yes | Yes |
| Always Ready | No | Yes | Yes |
| Private endpoints | No | Yes | Yes |
| Scale to zero | Yes | No (min 1) | No |
| Per-second billing | Yes | Yes (pre-warmed cost) | No |

## Triggers and Bindings

### Common Triggers

| Trigger | Description |
|---------|-------------|
| **HTTP** | REST API endpoint |
| **Timer** | Scheduled execution (CRON) |
| **Blob Storage** | File added/modified |
| **Queue Storage** | Message in queue |
| **Service Bus** | Message in topic/queue |
| **Event Hub** | Event stream processing |
| **Event Grid** | Event-driven architecture |
| **Cosmos DB** | Document changes |

### Bindings Example

```json
{
  "bindings": [
    {
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    },
    {
      "type": "blob",
      "direction": "in",
      "name": "inputBlob",
      "path": "input/{name}",
      "connection": "StorageConnection"
    },
    {
      "type": "queue",
      "direction": "out",
      "name": "outputQueue",
      "queueName": "myqueue",
      "connection": "StorageConnection"
    }
  ]
}
```

## Function Code Examples

### HTTP Trigger (Python)

```python
import azure.functions as func
import logging

app = func.FunctionApp()

@app.route(route="hello")
def hello(req: func.HttpRequest) -> func.HttpResponse:
    name = req.params.get('name', 'World')
    return func.HttpResponse(f"Hello, {name}!")
```

### Timer Trigger (JavaScript)

```javascript
const { app } = require('@azure/functions');

app.timer('timerTrigger', {
    schedule: '0 */5 * * * *', // Every 5 minutes
    handler: (myTimer, context) => {
        context.log('Timer function executed at:', new Date().toISOString());
    }
});
```

### Blob Trigger (C#)

```csharp
[Function("BlobTrigger")]
public void Run(
    [BlobTrigger("samples/{name}", Connection = "StorageConnection")] string content,
    string name,
    FunctionContext context)
{
    var logger = context.GetLogger("BlobTrigger");
    logger.LogInformation($"Blob {name} processed. Content length: {content.Length}");
}
```

## Timer Trigger CRON Expressions

```
┌───────────── second (0-59)
│ ┌───────────── minute (0-59)
│ │ ┌───────────── hour (0-23)
│ │ │ ┌───────────── day of month (1-31)
│ │ │ │ ┌───────────── month (1-12)
│ │ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │ │
* * * * * *

Examples:
0 */5 * * * *     Every 5 minutes
0 0 * * * *       Every hour
0 0 9 * * *       Every day at 9:00 AM
0 0 9 * * 1-5     Weekdays at 9:00 AM
0 0 0 1 * *       First of every month
```

## Durable Functions

Stateful functions for complex workflows.

### Patterns

```
1. Function Chaining
   F1 → F2 → F3 → F4

2. Fan-out/Fan-in
        ┌→ F1 ─┐
   Start ├→ F2 ─┼→ Aggregate
        └→ F3 ─┘

3. Async HTTP APIs
   POST → Start → GET /status → GET /result

4. Monitor
   Loop: Check condition → Wait → Repeat

5. Human Interaction
   Request → Wait for approval → Continue
```

### Durable Functions Example

```csharp
// Orchestrator
[Function("Orchestrator")]
public static async Task<List<string>> RunOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var outputs = new List<string>();
    outputs.Add(await context.CallActivityAsync<string>("Activity", "Tokyo"));
    outputs.Add(await context.CallActivityAsync<string>("Activity", "Seattle"));
    outputs.Add(await context.CallActivityAsync<string>("Activity", "London"));
    return outputs;
}

// Activity
[Function("Activity")]
public static string SayHello([ActivityTrigger] string city)
{
    return $"Hello {city}!";
}
```

## Application Settings & Configuration

```bash
# Local development: local.settings.json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "MY_SETTING": "value"
  }
}

# Access in code
import os
my_setting = os.environ['MY_SETTING']
```

### Key Vault References

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/)
@Microsoft.KeyVault(VaultName=myvault;SecretName=mysecret)
```

## Networking

### VNet Integration (Premium/Dedicated)

```
┌─────────────────────────────────────────────────────┐
│                        VNet                          │
│  ┌─────────────────────────────────────────────────┐│
│  │              Integration Subnet                  ││
│  │  ┌──────────────┐                               ││
│  │  │ Function App │ ──────▶ Private Resources    ││
│  │  │ (outbound)   │         (SQL, Storage, etc.) ││
│  │  └──────────────┘                               ││
│  └─────────────────────────────────────────────────┘│
│                                                      │
│  ┌─────────────────────────────────────────────────┐│
│  │              Private Endpoint                    ││
│  │  ┌──────────────┐                               ││
│  │  │ Function App │ ◀────── Inbound traffic      ││
│  │  │ (inbound)    │         (private access)     ││
│  │  └──────────────┘                               ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

## Deployment Options

| Method | Use Case |
|--------|----------|
| **VS Code** | Local development |
| **Azure CLI** | CI/CD scripts |
| **GitHub Actions** | Automated deployments |
| **Azure DevOps** | Enterprise CI/CD |
| **ZIP Deploy** | Package deployment |
| **Container** | Custom dependencies, Linux |

```bash
# Deploy via CLI
func azure functionapp publish <FunctionAppName>

# Deploy from ZIP
az functionapp deployment source config-zip \
  --resource-group myRG \
  --name myFunctionApp \
  --src ./function.zip
```

## Monitoring

### Application Insights

```python
import logging

def main(req):
    logging.info('Function triggered')      # Trace
    logging.warning('Something unusual')    # Warning
    logging.error('Something failed')       # Error

    # Custom metrics/events via OpenTelemetry
```

### Key Metrics

- **Invocations**: Number of function executions
- **Execution Time**: Duration of executions
- **Success/Failure Rate**: Error tracking
- **Active Instances**: Concurrent instances

## CLI Quick Reference

```bash
# Create function app
az functionapp create \
  --resource-group myRG \
  --name myFunctionApp \
  --storage-account mystorageaccount \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4

# Create with Premium plan
az functionapp create \
  --resource-group myRG \
  --name myFunctionApp \
  --storage-account mystorageaccount \
  --plan myPremiumPlan \
  --runtime node \
  --runtime-version 20

# Deploy code
func azure functionapp publish myFunctionApp

# Get function URL
az functionapp function show \
  --resource-group myRG \
  --name myFunctionApp \
  --function-name MyFunction \
  --query invokeUrlTemplate

# View logs
func azure functionapp logstream myFunctionApp

# Set app setting
az functionapp config appsettings set \
  --resource-group myRG \
  --name myFunctionApp \
  --settings "MY_SETTING=value"
```

## Pricing

| Plan | Compute | Executions | Storage |
|------|---------|------------|---------|
| **Consumption** | $0.000016/GB-s | $0.20/million | Included |
| **Premium** | ~$0.173/vCPU-hr | Included | Included |
| **Dedicated** | App Service pricing | Included | Included |

### Cost Formula (Consumption)

```
Monthly Cost = (GB-s × $0.000016) + (Executions/1M × $0.20)

Free tier: 400,000 GB-s + 1M executions/month

Example: 1M executions, 1GB memory, 500ms avg
= 1M × 1 × 0.5 = 500,000 GB-s
= (500,000 × $0.000016) + (1 × $0.20)
= $8.00 + $0.20 = $8.20/month
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Consumption vs Premium**: Premium for VNet, no cold starts, longer timeout
2. **Consumption timeout**: 5 min default, 10 min max (not 15 like AWS Lambda)
3. **Durable Functions**: Use for stateful, long-running workflows
4. **Triggers vs Bindings**: Triggers invoke function; bindings connect to data
5. **CRON format**: 6 fields (includes seconds), unlike standard 5-field CRON
6. **Premium always-ready**: Minimum 1 instance always warm
7. **Deployment slots**: Available in Premium and Dedicated plans
8. **Managed Identity**: Use for secure access to other Azure services
9. **Function Keys**: host key (all functions), function key (specific function)
10. **Isolated process**: Recommended for .NET (better control, updates)

## Gotchas

- Consumption plan timeout is 10 min max (not 15 like Lambda)
- Timer trigger uses UTC by default (set WEBSITE_TIME_ZONE)
- Consumption plan may have cold starts after ~20 minutes of inactivity
- Function app name must be globally unique (becomes part of URL)
- Python functions are Linux only
- Durable Functions require Azure Storage account
- VNet integration only in Premium/Dedicated plans
- local.settings.json is NOT deployed to Azure
- Premium plan has minimum 1 always-ready instance (costs money even when idle)

## Limits

| Resource | Consumption | Premium | Dedicated |
|----------|-------------|---------|-----------|
| Timeout | 10 min | 60 min | Unlimited |
| Max instances | 200 | 100 | 10-30 |
| Memory | 1.5 GB | 3.5-14 GB | Varies |
| Max deployment size | 1 GB | 1 GB | 1 GB |
| Functions per app | Unlimited | Unlimited | Unlimited |
| App settings | 4 KB total | 4 KB total | 4 KB total |
