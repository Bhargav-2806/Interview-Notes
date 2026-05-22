# DevOps Question 1 — How Do You Handle Incident Management in DevOps?

> **Section:** DevOps Miscellaneous &nbsp;|&nbsp; **Topic:** Incident Management / Monitoring / Alerting &nbsp;|&nbsp; **Level:** Mid (2–5 yrs) &nbsp;|&nbsp; **Frequency:** Very High
>
> _Part of the DevOps & DevSecOps Interview Preparation series._

---

## ❓ The Question

> **"Walk me through how you manage a P1 incident — from detection to resolution."**

You may also hear this phrased as:

- "How does your team handle on-call incidents?"
- "What monitoring and alerting tools do you use?"
- "What is your incident response process?"
- "Tell me about a time a production service went down — what did you do?"
- "How do you set up alerting for a new microservice?"
- "What happens after an incident is resolved?"

---

## 🎯 Why Interviewers Ask This

This question is asked in nearly every DevOps, SRE, and Platform Engineering interview because incident management is the heartbeat of operational excellence. The interviewer wants to validate:

- You understand the **full incident lifecycle** — not just "fix the thing" but detect → notify → triage → resolve → review → prevent.
- You have hands-on experience with **monitoring and alerting stacks** (CloudWatch, Prometheus + Alertmanager, Grafana, PagerDuty, etc.).
- You can own **on-call responsibilities** — you're not just a code pusher; you keep things running at 2 AM.
- You drive **continuous improvement** through postmortems — closing the learning loop.
- You understand **reliability vs. velocity** trade-offs and how SLOs/SLAs inform alert thresholds.

For DevSecOps roles, there's an additional lens: do you treat incidents as potential **security events** and handle them through that lens?

---

## 📖 Key Terminologies

| Term | Plain-English Meaning |
|------|----------------------|
| **Incident** | An unplanned interruption to a service or degradation in quality of a service. |
| **P1 / SEV-1** | Highest severity incident — critical service fully down, affecting all/most users. |
| **P2 / SEV-2** | Major functionality impaired; significant user impact, but partial degradation. |
| **P3 / SEV-3** | Minor issue; limited user impact, workaround available. |
| **On-call Engineer** | The engineer designated to respond to alerts outside business hours. |
| **Alert** | A notification triggered when a metric crosses a defined threshold. |
| **Monitoring** | Continuous collection and analysis of system metrics and logs. |
| **Observability** | The ability to understand internal system state from external outputs (metrics, logs, traces). |
| **SLO** | Service Level Objective — internal reliability target (e.g., 99.9% uptime per month). |
| **SLA** | Service Level Agreement — contractual commitment to customers. |
| **SLI** | Service Level Indicator — the actual measured metric (e.g., error rate, latency p99). |
| **Error Budget** | Allowed downtime/errors within an SLO period. When it runs out, freeze deployments. |
| **RCA / Root Cause Analysis** | Identifying the underlying cause of an incident. |
| **Postmortem** | A blameless review after an incident to understand what happened and prevent recurrence. |
| **MTTR** | Mean Time to Resolve — average time from incident detection to resolution. |
| **MTTD** | Mean Time to Detect — how long before the monitoring system catches the issue. |
| **MTBF** | Mean Time Between Failures — average time between outages. |
| **PagerDuty / OpsGenie** | On-call management platforms that route alerts to the right engineer. |
| **Runbook** | A documented set of steps for responding to a specific type of incident. |
| **War Room** | A dedicated (virtual or physical) space where responders coordinate during major incidents. |
| **Escalation Policy** | Rules defining who gets notified if the primary on-call doesn't acknowledge within N minutes. |
| **Blameless Culture** | Focus on systemic failures, not individual mistakes, during postmortems. |

---

## 🗣️ How to Answer (Structured)

**The STAR+R Framework for incident questions:**

1. **Situation** — Describe the service and its criticality.
2. **Trigger** — What caused the alert to fire?
3. **Action** — Walk through your exact response steps.
4. **Resolution** — How did you fix it?
5. **Reflection** — What changed afterward (postmortem output)?

**Model Answer (for the interview):**

> "In our organisation, our applications emit metrics captured by monitoring systems — CloudWatch for AWS-native resources and Prometheus for our Kubernetes workloads. We define alert rules on thresholds tied to our SLOs — for example, error rate > 1% for 5 minutes or p99 latency > 2 seconds for 10 minutes. When an alert fires, Alertmanager routes it to our on-call rotation in PagerDuty, which pages the engineer immediately by phone call. Supplemental notifications go to a dedicated Slack incident channel.
>
> The on-call engineer acknowledges the page, pulls up the Grafana dashboard, and begins triage using runbooks stored in our wiki. For a P1, we open a war room Slack channel, page in a second engineer, and notify the incident commander. Once resolved, we do a blameless postmortem within 48 hours — identifying root cause, contributing factors, and action items with owners and due dates."

---

## 📊 The Incident Management Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      INCIDENT MANAGEMENT LIFECYCLE                      │
└─────────────────────────────────────────────────────────────────────────┘

Phase 1: PREPARE          Phase 2: DETECT           Phase 3: RESPOND
─────────────────         ─────────────────         ─────────────────
• Define SLOs/SLIs        • Metric threshold        • Acknowledge alert
• Write runbooks          • Log anomaly             • Assess severity
• Set up on-call rota     • Synthetic monitor       • Page additional
• Configure alerts        • User report               responders
• Create dashboards       • APM trace spike         • Open war room
• Load test systems       • Security event          • Communicate status

Phase 4: INVESTIGATE      Phase 5: RESOLVE          Phase 6: REVIEW
─────────────────         ─────────────────         ─────────────────
• Check dashboards        • Apply fix               • Write postmortem
• Correlate logs          • Verify recovery         • Identify root cause
• Trace requests          • Monitor for relapse     • Assign action items
• Identify root cause     • Notify stakeholders     • Update runbooks
• Test hypotheses         • Declare all-clear       • Improve alerts
• Use runbook             • Document timeline       • Prevent recurrence
```

---

## 🔍 Monitoring Fundamentals

### The Three Pillars of Observability

```
                    OBSERVABILITY
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    METRICS           LOGS            TRACES
        │                │                │
  "What's the        "What happened"   "Where did
   current state?"   "What errors?"    time go?"
        │                │                │
  CloudWatch         CloudWatch Logs   AWS X-Ray
  Prometheus         Loki              Jaeger
  Datadog            Elasticsearch     Zipkin
  Grafana            Splunk            OpenTelemetry
```

### Metrics: The Numbers That Tell a Story

Metrics are time-series numeric values sampled at regular intervals. Every production system should emit four **golden signals** (Google SRE Book):

| Signal | Description | Example Metric |
|--------|-------------|----------------|
| **Latency** | Time to serve a request | `http_request_duration_seconds` |
| **Traffic** | Request rate / throughput | `http_requests_total` |
| **Errors** | Rate of failed requests | `http_errors_total` |
| **Saturation** | Resource utilisation | `cpu_usage_percent`, `memory_used_bytes` |

Additional USE method metrics (for infrastructure):
- **Utilisation** — % time resource is busy
- **Saturation** — how much work is queued / waiting
- **Errors** — count of error events

### Monitoring Tools Compared

| Tool | Type | Best For | Managed Option |
|------|------|----------|----------------|
| **CloudWatch** | Cloud-native | AWS-native resources (EC2, RDS, Lambda, ECS) | Yes (fully managed) |
| **Prometheus** | Pull-based, TSDB | Kubernetes, microservices | Amazon Managed Prometheus (AMP) |
| **Grafana** | Visualisation | Multi-source dashboards | Amazon Managed Grafana (AMG) |
| **Datadog** | SaaS, full-stack | Multi-cloud, APM, logs in one | Yes (SaaS) |
| **New Relic** | SaaS, APM-first | Application performance | Yes (SaaS) |
| **Dynatrace** | AI-powered | Automatic anomaly detection | Yes (SaaS) |
| **Nagios** | Legacy, passive | On-premises, legacy systems | No |
| **Zabbix** | Open-source, active | Hybrid infra, SNMP devices | No |

---

## 📡 CloudWatch Deep Dive

### CloudWatch Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                         AWS CLOUDWATCH                                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  DATA SOURCES             CLOUDWATCH CORE         ACTIONS               │
│  ────────────             ──────────────          ──────                │
│  EC2 instances    ──────► Metrics          ──────► CloudWatch Alarms    │
│  RDS, ElastiCache ──────► Logs             ──────► SNS → Email/SMS      │
│  Lambda functions ──────► Events           ──────► SNS → PagerDuty      │
│  ECS/EKS          ──────► Dashboards       ──────► SNS → Slack (Lambda) │
│  ALB/NLB          ──────► Insights (query) ──────► Auto Scaling         │
│  API Gateway      ──────► Container Insight──────► Lambda function       │
│  Custom app code  ──────► Application Insight────► Systems Manager      │
│  (PutMetricData)                                                         │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

### CloudWatch Metric Namespaces

```bash
# Built-in AWS namespaces (partial list)
AWS/EC2          # CPUUtilization, NetworkIn, NetworkOut, DiskReadOps
AWS/RDS          # DatabaseConnections, ReadLatency, WriteLatency, FreeStorageSpace
AWS/ApplicationELB  # RequestCount, HTTPCode_Target_5XX_Count, TargetResponseTime
AWS/Lambda       # Invocations, Errors, Duration, Throttles, ConcurrentExecutions
AWS/ECS          # CPUUtilization, MemoryUtilization (per cluster/service)
AWS/EKS          # (pod metrics via Container Insights)
AWS/ApiGateway   # Count, 5XXError, 4XXError, Latency, IntegrationLatency
AWS/SQS          # NumberOfMessagesSent, ApproximateAgeOfOldestMessage
AWS/S3           # BucketSizeBytes, NumberOfObjects, AllRequests

# Custom namespace (your application)
MyApp/Orders     # PutMetricData with Namespace=MyApp/Orders
```

### Creating CloudWatch Alarms (CLI)

```bash
# 1. Alarm: EC2 CPU > 80% for 2 consecutive 5-minute periods → SNS
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-WebServer" \
  --alarm-description "EC2 CPU utilization exceeds 80% for 10 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:incident-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789012:incident-resolved \
  --treat-missing-data breaching

# 2. Alarm: ALB 5xx error rate > 5% for 5 minutes → P1 alert
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-HighErrorRate-P1" \
  --alarm-description "ALB 5xx errors above 5% — P1 incident trigger" \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=app/my-alb/abc123 \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:p1-pagerduty \
  --treat-missing-data notBreaching

# 3. Alarm using Metric Math — error rate percentage
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-ErrorRate-Percentage" \
  --alarm-description "5xx error rate > 1% of total requests" \
  --metrics '[
    {
      "Id": "e1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "HTTPCode_Target_5XX_Count",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/my-alb/abc123"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "r1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "RequestCount",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/my-alb/abc123"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "errorRate",
      "Expression": "e1/r1*100",
      "Label": "ErrorRatePercent",
      "ReturnData": true
    }
  ]' \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:p1-pagerduty

# 4. CloudWatch Log-based alarm — count "ERROR" in application logs
aws cloudwatch put-metric-alarm \
  --alarm-name "AppLogs-ErrorSpike" \
  --metric-name "ErrorCount" \
  --namespace "MyApp/Logs" \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:incident-alerts

# Create the metric filter to extract log data
aws logs put-metric-filter \
  --log-group-name "/aws/app/production" \
  --filter-name "ErrorFilter" \
  --filter-pattern "[timestamp, requestId, level=ERROR, ...]" \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp/Logs,metricValue=1,defaultValue=0
```

### CloudWatch Composite Alarms (Reduce Alert Noise)

```bash
# Composite alarm: only page if BOTH CPU is high AND error rate is elevated
# (prevents false positives from a single metric spike)
aws cloudwatch put-composite-alarm \
  --alarm-name "P1-Composite-WebTier" \
  --alarm-description "Page only if high CPU AND high error rate simultaneously" \
  --alarm-rule "ALARM(\"HighCPU-WebServer\") AND ALARM(\"ALB-HighErrorRate-P1\")" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:p1-pagerduty \
  --ok-actions arn:aws:sns:us-east-1:123456789012:incident-resolved
```

---

## 🔥 Prometheus + Alertmanager Deep Dive

### Prometheus Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PROMETHEUS STACK                                   │
│                                                                        │
│  TARGET DISCOVERY          PROMETHEUS SERVER      ALERTMANAGER         │
│  ────────────────          ────────────────       ────────────         │
│                                                                        │
│  Kubernetes SD    ──────►  /scrape config         Receives alerts      │
│  EC2 SD           ──────►  TSDB storage           ──────────────       │
│  Consul SD        ──────►  PromQL engine   ──────► Grouping            │
│  Static config    ──────►  Rule evaluation ──────► Deduplication       │
│                            (alert.rules)   ──────► Inhibition          │
│                                            ──────► Silencing           │
│  EXPORTERS                                         │                   │
│  ─────────                 GRAFANA        ◄──────  Routing             │
│  node_exporter    ──────►  (dashboards)           │                   │
│  blackbox_exporter         (alerting UI)          ▼                   │
│  kube-state-metrics                         Email / Slack / PD         │
│  custom /metrics endpoint                   Phone / OpsGenie / Teams   │
└──────────────────────────────────────────────────────────────────────┘
```

### Prometheus Metrics Types

```yaml
# Counter — always increasing (resets on restart)
# Use for: request count, error count, bytes processed
http_requests_total{method="GET", status="200"} 1547

# Gauge — can go up or down
# Use for: current active connections, memory, temperature
active_connections 42
memory_usage_bytes 536870912

# Histogram — distributes observations into buckets
# Use for: request durations, response sizes
http_request_duration_seconds_bucket{le="0.1"} 24054
http_request_duration_seconds_bucket{le="0.25"} 33444
http_request_duration_seconds_bucket{le="0.5"} 100392
http_request_duration_seconds_bucket{le="1.0"} 129389
http_request_duration_seconds_bucket{le="+Inf"} 133988
http_request_duration_seconds_sum 53423
http_request_duration_seconds_count 133988

# Summary — pre-calculated quantiles (less flexible, no aggregation)
rpc_duration_seconds{quantile="0.5"}  0.012607
rpc_duration_seconds{quantile="0.9"}  0.026479
rpc_duration_seconds{quantile="0.99"} 0.029336
```

### PromQL Queries for Alerting

```promql
# Error rate over 5 minutes — percentage of 5xx responses
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100

# p99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Pod restart count in last 15 minutes
increase(kube_pod_container_status_restarts_total[15m]) > 5

# Node memory utilisation > 90%
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90

# SLO burn rate alert (multi-window, multi-burn-rate)
# Short window: 14.4× burn rate over 1 hour (fast burn)
(
  job:slo_errors:rate1h{job="api"} / on(job) group_left()
  job:slo_error_budget:ratio{job="api"}
) > 14.4
AND
# Long window: 14.4× burn rate over 5 minutes (to avoid false positives)
(
  job:slo_errors:rate5m{job="api"} / on(job) group_left()
  job:slo_error_budget:ratio{job="api"}
) > 14.4
```

### Prometheus Alert Rules (YAML)

```yaml
# /etc/prometheus/rules/application.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # P1: Service completely down (no successful requests)
      - alert: ServiceDown
        expr: up{job="my-api"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
          priority: P1
        annotations:
          summary: "Service {{ $labels.instance }} is DOWN"
          description: "{{ $labels.job }} on {{ $labels.instance }} has been down for more than 1 minute."
          runbook_url: "https://wiki.company.com/runbooks/service-down"
          dashboard_url: "https://grafana.company.com/d/abc123"

      # P1: High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) * 100 > 5
        for: 5m
        labels:
          severity: critical
          priority: P1
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanize }}% over the last 5 minutes on {{ $labels.service }}."
          runbook_url: "https://wiki.company.com/runbooks/high-error-rate"

      # P2: High latency
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 10m
        labels:
          severity: warning
          priority: P2
        annotations:
          summary: "P99 latency high for {{ $labels.service }}"
          description: "P99 response time is {{ $value | humanize }}s for {{ $labels.service }}."

      # P2: Pod crash-looping
      - alert: PodCrashLooping
        expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
        for: 5m
        labels:
          severity: warning
          priority: P2
        annotations:
          summary: "Pod {{ $labels.pod }} is crash-looping"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted {{ $value }} times in 15 minutes."
          runbook_url: "https://wiki.company.com/runbooks/pod-crashloop"

      # P3: High memory usage
      - alert: NodeMemoryHigh
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 15m
        labels:
          severity: warning
          priority: P3
        annotations:
          summary: "Node {{ $labels.instance }} memory > 85%"
          description: "Memory utilization is {{ $value | humanize }}% on {{ $labels.instance }}."

  - name: slo_alerts
    rules:
      # SLO burn rate — page-worthy (14.4× over 1h = 2% error budget burned in 1h)
      - alert: ErrorBudgetBurnRateFast
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          ) / (1 - 0.999) > 14.4
        for: 2m
        labels:
          severity: critical
          priority: P1
        annotations:
          summary: "Fast SLO burn on {{ $labels.service }}"
          description: "At current burn rate, error budget will be exhausted in ~1 hour."

      # Slow burn — P2 (ticket, not page)
      - alert: ErrorBudgetBurnRateSlow
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
            /
            sum(rate(http_requests_total[6h])) by (service)
          ) / (1 - 0.999) > 1
        for: 60m
        labels:
          severity: warning
          priority: P2
        annotations:
          summary: "Slow SLO burn on {{ $labels.service }}"
          description: "Error budget is being consumed. Investigate during business hours."
```

### Alertmanager Configuration

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
  pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

# Route tree: define who gets what
route:
  group_by: ['alertname', 'service', 'cluster']
  group_wait: 30s        # wait before sending first notification
  group_interval: 5m     # wait before sending updated notifications
  repeat_interval: 4h    # wait before re-sending unresolved alerts
  receiver: 'default-slack'

  routes:
    # P1 alerts → PagerDuty (phone call) + Slack
    - match:
        priority: P1
      receiver: 'p1-pagerduty'
      group_wait: 10s
      repeat_interval: 30m
      continue: true  # also send to Slack

    # P1 security alerts → Security team
    - match:
        priority: P1
        team: security
      receiver: 'security-team'
      group_wait: 5s

    # P2 alerts → Slack only (business hours)
    - match:
        priority: P2
      receiver: 'p2-slack'
      repeat_interval: 2h

    # P3 alerts → Low-priority Slack channel (informational)
    - match:
        priority: P3
      receiver: 'p3-slack'
      repeat_interval: 24h

    # Silenced during maintenance windows
    - match_re:
        alertname: '.*'
      matchers:
        - maintenance="true"
      receiver: 'null'

# Inhibition rules — suppress lower severity when higher is firing
inhibit_rules:
  - source_matchers:
      - severity = 'critical'
    target_matchers:
      - severity = 'warning'
    equal: ['service', 'cluster']

receivers:
  - name: 'default-slack'
    slack_configs:
      - channel: '#alerts-general'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        icon_url: 'https://avatars3.githubusercontent.com/u/3380462'
        send_resolved: true

  - name: 'p1-pagerduty'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_ROUTING_KEY'
        description: '{{ template "pagerduty.description" . }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        client: 'Alertmanager'
        client_url: '{{ template "pagerduty.client_url" . }}'
        details:
          runbook: '{{ .CommonAnnotations.runbook_url }}'
          dashboard: '{{ .CommonAnnotations.dashboard_url }}'
    slack_configs:
      - channel: '#incidents-p1'
        title: '🚨 P1 INCIDENT: {{ .CommonAnnotations.summary }}'
        text: |
          *Alert:* {{ .CommonAnnotations.summary }}
          *Description:* {{ .CommonAnnotations.description }}
          *Runbook:* {{ .CommonAnnotations.runbook_url }}
          *Dashboard:* {{ .CommonAnnotations.dashboard_url }}
        color: '#FF0000'
        send_resolved: true

  - name: 'p2-slack'
    slack_configs:
      - channel: '#alerts-p2'
        title: '⚠️ P2 Alert: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        color: '#FF8C00'
        send_resolved: true

  - name: 'p3-slack'
    slack_configs:
      - channel: '#alerts-p3'
        title: 'ℹ️ {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        color: '#36A64F'
        send_resolved: true

  - name: 'security-team'
    pagerduty_configs:
      - routing_key: 'SECURITY_PAGERDUTY_KEY'
        severity: critical
    email_configs:
      - to: 'security@company.com'
        from: 'alerts@company.com'
        smarthost: 'smtp.company.com:587'
        auth_username: 'alerts@company.com'
        auth_password: 'SECRET'
        subject: '[P1-SECURITY] {{ .CommonAnnotations.summary }}'

  - name: 'null'   # black hole for silenced alerts

# Custom templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

### Alertmanager Slack Message Template

```
# /etc/alertmanager/templates/slack.tmpl
{{ define "slack.title" }}
  [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
{{ end }}

{{ define "slack.text" }}
  {{ range .Alerts }}
    *Alert:* {{ .Annotations.summary }}
    *Description:* {{ .Annotations.description }}
    *Severity:* {{ .Labels.severity }}
    *Runbook:* {{ .Annotations.runbook_url }}
    *Dashboard:* {{ .Annotations.dashboard_url }}
    *Started:* {{ .StartsAt.Format "2006-01-02 15:04:05 UTC" }}
  {{ end }}
{{ end }}
```

---

## 📱 On-Call Engineering

### On-Call Rotation Best Practices (2024)

A healthy on-call programme looks like this:

```
ROTATION DESIGN
───────────────
• 5-day business week + 2-day weekend = 7-day shift
• Minimum 2 people on-call: Primary + Secondary (escalation backup)
• Rotation frequency: weekly (not daily — context switching is too high)
• Handover: structured 15-min sync + written summary

ESCALATION POLICY
─────────────────
1. Page PRIMARY on-call — wait 5 minutes for acknowledgement
2. If no ack → page SECONDARY on-call — wait 5 more minutes
3. If no ack → escalate to TEAM LEAD
4. If no ack → escalate to ENGINEERING MANAGER
5. For P1: simultaneously notify INCIDENT COMMANDER at step 1

TOIL BUDGET
───────────
Google SRE rule: On-call burden should be <50% of engineer time.
If the on-call pager fires >5 times/shift → system needs improvement, not more engineers.
Track pager fatigue: weekly alert count, MTTD, MTTR per service.
```

### PagerDuty Integration (via EventsV2 API)

```bash
# Trigger a PagerDuty alert (e.g., from a custom script or Lambda)
curl -X POST https://events.pagerduty.com/v2/enqueue \
  -H "Content-Type: application/json" \
  -d '{
    "routing_key": "YOUR_PD_INTEGRATION_KEY",
    "event_action": "trigger",
    "dedup_key": "prod-api-down-2024-001",
    "payload": {
      "summary": "Production API returning 100% 5xx errors",
      "severity": "critical",
      "source": "my-monitoring-system",
      "component": "api-gateway",
      "group": "production",
      "class": "availability",
      "custom_details": {
        "environment": "production",
        "region": "us-east-1",
        "error_rate": "100%",
        "runbook": "https://wiki.company.com/runbooks/api-down"
      }
    },
    "links": [
      {
        "href": "https://grafana.company.com/d/api-dashboard",
        "text": "Grafana Dashboard"
      }
    ]
  }'

# Acknowledge an alert
curl -X POST https://events.pagerduty.com/v2/enqueue \
  -H "Content-Type: application/json" \
  -d '{
    "routing_key": "YOUR_PD_INTEGRATION_KEY",
    "event_action": "acknowledge",
    "dedup_key": "prod-api-down-2024-001"
  }'

# Resolve an alert
curl -X POST https://events.pagerduty.com/v2/enqueue \
  -H "Content-Type: application/json" \
  -d '{
    "routing_key": "YOUR_PD_INTEGRATION_KEY",
    "event_action": "resolve",
    "dedup_key": "prod-api-down-2024-001"
  }'
```

### SNS → PagerDuty via Lambda (AWS Native Flow)

```python
# lambda_function.py — converts CloudWatch alarm to PagerDuty event
import json
import urllib.request
import os

PD_ROUTING_KEY = os.environ['PAGERDUTY_ROUTING_KEY']
PD_API_URL = "https://events.pagerduty.com/v2/enqueue"

def lambda_handler(event, context):
    # Parse SNS message from CloudWatch alarm
    for record in event['Records']:
        message = json.loads(record['Sns']['Message'])
        
        alarm_name = message['AlarmName']
        new_state = message['NewStateValue']
        reason = message['NewStateReason']
        region = message['Region']
        
        # Map CloudWatch state to PagerDuty action
        if new_state == 'ALARM':
            event_action = 'trigger'
            severity = 'critical' if 'P1' in alarm_name else 'warning'
        elif new_state == 'OK':
            event_action = 'resolve'
            severity = 'info'
        else:  # INSUFFICIENT_DATA
            return {'statusCode': 200, 'body': 'Skipping INSUFFICIENT_DATA'}
        
        # Build PagerDuty payload
        pd_payload = {
            "routing_key": PD_ROUTING_KEY,
            "event_action": event_action,
            "dedup_key": f"cloudwatch-{alarm_name}-{region}",
            "payload": {
                "summary": f"{alarm_name}: {reason[:200]}",
                "severity": severity,
                "source": f"CloudWatch/{region}",
                "component": alarm_name,
                "group": "AWS",
                "custom_details": {
                    "alarm_name": alarm_name,
                    "state": new_state,
                    "reason": reason,
                    "region": region,
                    "account_id": message.get('AWSAccountId', 'unknown')
                }
            }
        }
        
        # Send to PagerDuty
        req = urllib.request.Request(
            PD_API_URL,
            data=json.dumps(pd_payload).encode('utf-8'),
            headers={'Content-Type': 'application/json'},
            method='POST'
        )
        
        with urllib.request.urlopen(req) as response:
            print(f"PagerDuty response: {response.status} {response.read()}")
    
    return {'statusCode': 200, 'body': 'OK'}
```

---

## 🚨 The Incident Response Process (Step by Step)

### Step 1: Detection

```
Detection Sources (in order of speed):
──────────────────────────────────────
1. Synthetic monitoring (fastest — 1 min)
   → Blackbox exporter pinging /health every 30s
   → CloudWatch Synthetics canary
   
2. Metric threshold alert (fast — 2-5 min)
   → Error rate spike, latency jump, memory OOM
   
3. Log anomaly (medium — 5-10 min)
   → CloudWatch Logs Insights pattern match
   → ELK/Splunk anomaly detection
   
4. User report (slow — 5-30 min)
   → Support ticket, social media, status page complaint
   → Internal Slack message "the app is down"
   
5. Business metric anomaly (slowest)
   → "Orders dropped 80% in the last hour"
```

### Step 2: Severity Classification

```
P1 — Critical (page immediately, 24/7)
────────────────────────────────────────
• Complete service outage affecting all users
• Revenue-generating features non-functional
• Data loss or corruption occurring NOW
• Security breach in progress
• SLA breach imminent (< 30 minutes remaining)
Response time target: Acknowledge < 5 min, MTTR < 1 hour

P2 — Major (page immediately during business hours, off-hours: email+slack)
────────────────────────────────────────────────────────────────────────────
• Core feature degraded but partially functional
• >10% of users impacted
• Performance degradation >3x normal latency
• Non-critical service completely down
Response time target: Acknowledge < 30 min, MTTR < 4 hours

P3 — Minor (Slack + ticket, resolve next business day)
──────────────────────────────────────────────────────
• Edge case failures affecting <1% users
• Non-critical feature broken, workaround exists
• Performance degradation but within SLA
Response time target: Same day, MTTR < 24 hours

P4 — Informational (backlog ticket)
────────────────────────────────────
• Cosmetic issues, minor bugs
• Potential future issue (warning trend)
Response time target: Sprint planning
```

### Step 3: Incident Communication Template

```
# Slack war room initial message (post within 5 minutes of P1 declaration)

🚨 *INCIDENT DECLARED: INC-2024-042*
*Status:* Investigating
*Severity:* P1
*Affected:* Production API — users cannot log in
*Detected:* 14:32 UTC via CloudWatch Alarm (HTTPCode_5XX > 100/min)
*IC (Incident Commander):* @alice
*Tech Lead:* @bob
*Comms:* @charlie (updates status page)

*Initial Symptoms:*
• ALB 5xx errors: 100% since 14:30 UTC
• ECS service healthy check: FAILING
• Last deploy: 14:15 UTC (PR #1234 — auth refactor)

*Current Actions:*
• [ ] @bob — checking ECS task logs
• [ ] @alice — reviewing last deployment
• [ ] @charlie — posted "investigating" on status page

*Next update in:* 15 minutes or on change
```

### Step 4: Triage Checklist

```bash
# Systematic triage — run these in order

# 1. Check service health
aws ecs describe-services \
  --cluster production \
  --services my-api \
  --query 'services[0].{Running:runningCount,Desired:desiredCount,Pending:pendingCount}'

# 2. Check recent ECS task failures
aws ecs list-tasks \
  --cluster production \
  --service-name my-api \
  --desired-status STOPPED \
  --query 'taskArns' \
  --output text | head -5

aws ecs describe-tasks \
  --cluster production \
  --tasks <task-arn> \
  --query 'tasks[0].{Status:lastStatus,StopCode:stopCode,StopReason:stoppedReason,Exit:containers[0].exitCode}'

# 3. Tail application logs (CloudWatch Logs Insights)
aws logs start-query \
  --log-group-name "/ecs/my-api" \
  --start-time $(date -d '30 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 50'

# 4. Check ALB target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123:targetgroup/my-api/abc \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}'

# 5. Recent deployments / events
aws ecs describe-service-deployments \
  --service-deployment-arns $(aws ecs list-service-deployments \
    --cluster production --service my-api --query 'serviceDeploymentArns[0]' --output text)

# 6. Infrastructure events
aws ec2 describe-instance-status \
  --filters Name=instance-state-name,Values=running \
  --include-all-instances \
  --query 'InstanceStatuses[?Events!=null].{ID:InstanceId,Events:Events}'

# 7. RDS health (if DB-related)
aws rds describe-events \
  --source-identifier my-database \
  --source-type db-instance \
  --duration 60 \
  --query 'Events[*].{Time:Date,Message:Message}'
```

### Step 5: Common Resolution Patterns

```bash
# Pattern 1: Bad deployment → Rollback
# ECS rolling deployment rollback
aws ecs update-service \
  --cluster production \
  --service my-api \
  --task-definition my-api:PREVIOUS_REVISION \
  --force-new-deployment

# Blue/Green deployment rollback via CodeDeploy
aws deploy stop-deployment \
  --deployment-id d-ABC123 \
  --auto-rollback-enabled

# Pattern 2: Pod crashloop → Kubernetes rollback
kubectl rollout undo deployment/my-api -n production
kubectl rollout status deployment/my-api -n production --timeout=5m

# Check rollout history
kubectl rollout history deployment/my-api -n production

# Pattern 3: Database connection exhaustion
# Check current connections (PostgreSQL)
# Via RDS Performance Insights or:
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=my-database \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Maximum

# Restart RDS Proxy to flush stale connections (if using RDS Proxy)
aws rds reboot-db-instance --db-instance-identifier my-database-proxy

# Pattern 4: ASG capacity insufficient → manual scale-out
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 20 \
  --honor-cooldown

# Pattern 5: Memory leak → ECS task restart (preserves capacity)
aws ecs update-service \
  --cluster production \
  --service my-api \
  --force-new-deployment

# Pattern 6: Certificate expired → renew ACM
aws acm request-certificate \
  --domain-name api.company.com \
  --validation-method DNS \
  --subject-alternative-names "*.api.company.com"

# Pattern 7: S3 bucket policy blocking access
aws s3api get-bucket-policy --bucket my-bucket
# Fix and apply corrected policy
aws s3api put-bucket-policy --bucket my-bucket --policy file://fixed-policy.json
```

---

## 📝 Postmortem Best Practices

### Blameless Postmortem Culture

> **Key principle:** Systems fail, not people. We build better systems to prevent recurrence. Blame creates fear of reporting, which leads to delayed detection and hidden problems.

### Postmortem Template

```markdown
# Postmortem: [Incident Title] — INC-2024-042
**Date:** 2024-03-15
**Severity:** P1
**Duration:** 47 minutes (14:30 UTC – 15:17 UTC)
**Incident Commander:** Alice
**Authors:** Bob, Charlie
**Status:** Action items in progress

---

## Impact
- 100% of users unable to log in for 47 minutes
- Estimated 12,000 affected users
- $47,000 estimated revenue impact (2,800 failed transactions × $16.80 avg order)
- No data loss or security breach

---

## Timeline (UTC)
| Time | Event |
|------|-------|
| 14:15 | Deployment INC-PR-1234 merged and deployed to production (auth refactor) |
| 14:22 | First elevated error rate detected by Prometheus (2% — below alert threshold) |
| 14:30 | Alert fires: ALB 5xx > 100/min for 5 minutes |
| 14:31 | PagerDuty pages on-call engineer (Bob) |
| 14:33 | Bob acknowledges, begins investigation |
| 14:35 | P1 declared; Alice joins as IC; Charlie updates status page |
| 14:40 | Root cause identified: JWT library breaking change in v2.1.0 |
| 14:45 | Decision made: rollback deployment vs. hotfix |
| 14:48 | Rollback initiated via `aws ecs update-service` |
| 15:02 | New tasks running — errors begin to decrease |
| 15:17 | Error rate back to baseline; all-clear declared |
| 15:20 | Status page updated: resolved |

---

## Root Cause
The deployment updated the `jwt-go` library from v2.0.4 to v2.1.0. This version introduced a breaking change in token validation: it now rejects tokens with the `alg` claim set to `none` (a security improvement), but our legacy tokens used during session refresh included this claim. All token refreshes failed with a 401, which cascaded to 5xx at the ALB because the application returned an unhandled error.

## Contributing Factors
1. **Inadequate test coverage** — the integration test suite did not cover the token refresh flow.
2. **Dependency update not flagged as breaking** — semantic versioning minor bump (2.0 → 2.1) was not treated as a potential breaking change.
3. **Alert threshold too high** — the initial error rate spike (2-5% for 8 minutes) did not trigger an alert. Only caught at 5% threshold.
4. **No canary deployment** — the change was deployed to 100% of production traffic simultaneously.

## What Went Well
- PagerDuty routing worked correctly; on-call responded in <2 minutes.
- Status page was updated within 5 minutes of P1 declaration.
- Rollback was available and executed cleanly.
- Incident Commander kept the war room focused.

## Action Items
| Item | Owner | Due Date | Priority |
|------|-------|----------|----------|
| Add integration test for token refresh flow | Bob | 2024-03-22 | P1 |
| Lower error rate alert threshold from 5% to 1% | Platform team | 2024-03-18 | P1 |
| Implement canary deployments (10% traffic first) | Alice | 2024-04-01 | P1 |
| Add Dependabot + changelog review gate to CI | Charlie | 2024-03-22 | P2 |
| Update runbook with JWT debugging steps | Bob | 2024-03-20 | P2 |
| Add synthetic monitor for /auth/refresh endpoint | Platform team | 2024-03-25 | P2 |

---

## Lessons Learned
1. Minor version bumps in security-sensitive libraries should be treated with the same rigor as major versions.
2. Multi-window burn rate alerting (fast + slow burn) would have caught this 8 minutes earlier.
3. Canary deployments are non-negotiable for auth service changes.
```

---

## 🏗️ Infrastructure as Code: Monitoring Stack

### Terraform: CloudWatch Alarms Module

```hcl
# modules/cloudwatch-alarms/main.tf
# Reusable alarm module for any ECS service

variable "service_name" {
  description = "ECS service name"
  type        = string
}

variable "cluster_name" {
  description = "ECS cluster name"
  type        = string
}

variable "alb_arn_suffix" {
  description = "ALB ARN suffix (from aws_lb data source)"
  type        = string
}

variable "target_group_arn_suffix" {
  description = "Target group ARN suffix"
  type        = string
}

variable "sns_critical_arn" {
  description = "SNS topic ARN for critical/P1 alerts"
  type        = string
}

variable "sns_warning_arn" {
  description = "SNS topic ARN for P2 warnings"
  type        = string
}

variable "error_rate_threshold" {
  description = "5xx error rate threshold (requests per 5 min)"
  default     = 50
  type        = number
}

variable "latency_threshold_ms" {
  description = "Target response time threshold in milliseconds"
  default     = 2000
  type        = number
}

variable "cpu_threshold_percent" {
  description = "ECS CPU utilisation threshold"
  default     = 85
  type        = number
}

variable "memory_threshold_percent" {
  description = "ECS memory utilisation threshold"
  default     = 90
  type        = number
}

# ─────────────────────────────────────────────────────────
# ALB Error Rate Alarm (P1)
# ─────────────────────────────────────────────────────────
resource "aws_cloudwatch_metric_alarm" "alb_5xx_critical" {
  alarm_name          = "${var.service_name}-alb-5xx-critical"
  alarm_description   = "P1: ALB 5xx errors above threshold for ${var.service_name}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  threshold           = var.error_rate_threshold
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "e1"
    return_data = false
    metric {
      namespace   = "AWS/ApplicationELB"
      metric_name = "HTTPCode_Target_5XX_Count"
      period      = 300
      stat        = "Sum"
      dimensions = {
        LoadBalancer = var.alb_arn_suffix
        TargetGroup  = var.target_group_arn_suffix
      }
    }
  }

  metric_query {
    id          = "r1"
    return_data = false
    metric {
      namespace   = "AWS/ApplicationELB"
      metric_name = "RequestCount"
      period      = 300
      stat        = "Sum"
      dimensions = {
        LoadBalancer = var.alb_arn_suffix
        TargetGroup  = var.target_group_arn_suffix
      }
    }
  }

  metric_query {
    id          = "error_rate"
    expression  = "IF(r1>0, e1/r1*100, 0)"
    label       = "5xx Error Rate %"
    return_data = true
  }

  alarm_actions = [var.sns_critical_arn]
  ok_actions    = [var.sns_critical_arn]

  tags = {
    Service  = var.service_name
    Severity = "critical"
    Priority = "P1"
  }
}

# ─────────────────────────────────────────────────────────
# ALB Latency Alarm (P2)
# ─────────────────────────────────────────────────────────
resource "aws_cloudwatch_metric_alarm" "alb_latency_p2" {
  alarm_name          = "${var.service_name}-alb-latency-p2"
  alarm_description   = "P2: ALB target response time high for ${var.service_name}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "p99"
  threshold           = var.latency_threshold_ms / 1000.0  # convert to seconds
  treat_missing_data  = "notBreaching"

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
    TargetGroup  = var.target_group_arn_suffix
  }

  alarm_actions = [var.sns_warning_arn]
  ok_actions    = [var.sns_warning_arn]

  tags = {
    Service  = var.service_name
    Severity = "warning"
    Priority = "P2"
  }
}

# ─────────────────────────────────────────────────────────
# ECS CPU Alarm (P2)
# ─────────────────────────────────────────────────────────
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "${var.service_name}-ecs-cpu-high"
  alarm_description   = "P2: ECS service CPU utilisation high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = var.cpu_threshold_percent
  treat_missing_data  = "notBreaching"

  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = var.service_name
  }

  alarm_actions = [var.sns_warning_arn]
  ok_actions    = [var.sns_warning_arn]
}

# ─────────────────────────────────────────────────────────
# ECS Memory Alarm (P2)
# ─────────────────────────────────────────────────────────
resource "aws_cloudwatch_metric_alarm" "ecs_memory_high" {
  alarm_name          = "${var.service_name}-ecs-memory-high"
  alarm_description   = "P2: ECS service memory utilisation high — potential OOM risk"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "MemoryUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = var.memory_threshold_percent
  treat_missing_data  = "notBreaching"

  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = var.service_name
  }

  alarm_actions = [var.sns_warning_arn]
  ok_actions    = [var.sns_warning_arn]
}

# ─────────────────────────────────────────────────────────
# Composite Alarm — Page only when multiple signals confirm
# ─────────────────────────────────────────────────────────
resource "aws_cloudwatch_composite_alarm" "service_critical" {
  alarm_name        = "${var.service_name}-composite-critical"
  alarm_description = "P1 composite: confirms critical issue from multiple signals"
  alarm_rule        = <<-EOT
    ALARM("${aws_cloudwatch_metric_alarm.alb_5xx_critical.alarm_name}")
    OR (
      ALARM("${aws_cloudwatch_metric_alarm.ecs_cpu_high.alarm_name}")
      AND ALARM("${aws_cloudwatch_metric_alarm.alb_latency_p2.alarm_name}")
    )
  EOT

  alarm_actions = [var.sns_critical_arn]
  ok_actions    = [var.sns_critical_arn]
}

output "critical_alarm_arn" {
  value = aws_cloudwatch_composite_alarm.service_critical.arn
}
```

### Terraform: SNS Topic with KMS Encryption

```hcl
# modules/alert-sns/main.tf

variable "environment" { type = string }
variable "pagerduty_endpoint" { type = string }
variable "slack_webhook_url" { type = string }

# KMS key for encrypting SNS messages (DevSecOps requirement)
resource "aws_kms_key" "sns_alerts" {
  description             = "KMS key for SNS alert topics"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowSNSService"
        Effect = "Allow"
        Principal = {
          Service = "sns.amazonaws.com"
        }
        Action   = ["kms:GenerateDataKey*", "kms:Decrypt"]
        Resource = "*"
      },
      {
        Sid    = "AllowCloudWatchToPublish"
        Effect = "Allow"
        Principal = {
          Service = "cloudwatch.amazonaws.com"
        }
        Action   = ["kms:GenerateDataKey*", "kms:Decrypt"]
        Resource = "*"
      },
      {
        Sid    = "AllowAccountAdmin"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })

  tags = {
    Environment = var.environment
    Purpose     = "SNS Encryption"
  }
}

resource "aws_kms_alias" "sns_alerts" {
  name          = "alias/sns-alerts-${var.environment}"
  target_key_id = aws_kms_key.sns_alerts.id
}

# Critical/P1 SNS topic
resource "aws_sns_topic" "critical" {
  name              = "alerts-critical-${var.environment}"
  kms_master_key_id = aws_kms_key.sns_alerts.id

  tags = {
    Environment = var.environment
    Priority    = "P1"
  }
}

# SNS topic policy — restrict who can publish
resource "aws_sns_topic_policy" "critical" {
  arn = aws_sns_topic.critical.arn
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudWatchAlarms"
        Effect = "Allow"
        Principal = {
          Service = "cloudwatch.amazonaws.com"
        }
        Action   = "SNS:Publish"
        Resource = aws_sns_topic.critical.arn
        Condition = {
          ArnLike = {
            "aws:SourceArn" = "arn:aws:cloudwatch:*:${data.aws_caller_identity.current.account_id}:alarm:*"
          }
        }
      },
      {
        Sid    = "AllowLambdaPublish"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.alert_lambda.arn
        }
        Action   = "SNS:Publish"
        Resource = aws_sns_topic.critical.arn
      },
      {
        Sid    = "DenyPublicAccess"
        Effect = "Deny"
        Principal = "*"
        Action   = "SNS:Publish"
        Resource = aws_sns_topic.critical.arn
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::${data.aws_caller_identity.current.account_id}:*"
            ]
          }
        }
      }
    ]
  })
}

# HTTPS subscription (no email — all routing via PagerDuty)
resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.critical.arn
  protocol  = "https"
  endpoint  = var.pagerduty_endpoint
  # PagerDuty provides the HTTPS endpoint in their CloudWatch integration setup
}

data "aws_caller_identity" "current" {}
```

### Terraform: Prometheus Stack on EKS (Helm)

```hcl
# monitoring/prometheus-stack.tf

resource "helm_release" "kube_prometheus_stack" {
  name             = "kube-prometheus-stack"
  repository       = "https://prometheus-community.github.io/helm-charts"
  chart            = "kube-prometheus-stack"
  version          = "56.6.2"  # pin version; check for updates quarterly
  namespace        = "monitoring"
  create_namespace = true
  timeout          = 600

  values = [
    yamlencode({
      # Prometheus configuration
      prometheus = {
        prometheusSpec = {
          retention                = "30d"
          retentionSize            = "50GB"
          storageSpec = {
            volumeClaimTemplate = {
              spec = {
                storageClassName = "gp3"
                resources = {
                  requests = { storage = "50Gi" }
                }
              }
            }
          }
          # Scrape additional services
          additionalScrapeConfigs = [
            {
              job_name       = "application-services"
              kubernetes_sd_configs = [{ role = "pod" }]
              relabel_configs = [
                {
                  source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
                  action        = "keep"
                  regex         = "true"
                },
                {
                  source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_path"]
                  action        = "replace"
                  target_label  = "__metrics_path__"
                  regex         = "(.+)"
                }
              ]
            }
          ]
          # External labels to identify this cluster in multi-cluster setups
          externalLabels = {
            cluster     = var.cluster_name
            environment = var.environment
            region      = var.aws_region
          }
          # Resources
          resources = {
            requests = { cpu = "500m", memory = "2Gi" }
            limits   = { cpu = "2000m", memory = "4Gi" }
          }
        }
      }

      # Alertmanager configuration
      alertmanager = {
        alertmanagerSpec = {
          storage = {
            volumeClaimTemplate = {
              spec = {
                storageClassName = "gp3"
                resources = {
                  requests = { storage = "10Gi" }
                }
              }
            }
          }
        }
        config = {
          global = {
            resolve_timeout = "5m"
          }
          route = {
            group_by       = ["alertname", "service", "namespace"]
            group_wait     = "30s"
            group_interval = "5m"
            repeat_interval = "4h"
            receiver       = "default"
            routes = [
              {
                match    = { severity = "critical" }
                receiver = "pagerduty-critical"
              },
              {
                match    = { severity = "warning" }
                receiver = "slack-warning"
              }
            ]
          }
          receivers = [
            {
              name = "default"
              slack_configs = [
                {
                  api_url  = var.slack_webhook_url
                  channel  = "#alerts"
                  title    = "{{ .CommonAnnotations.summary }}"
                  text     = "{{ .CommonAnnotations.description }}"
                  send_resolved = true
                }
              ]
            },
            {
              name = "pagerduty-critical"
              pagerduty_configs = [
                {
                  routing_key = var.pagerduty_routing_key
                  description = "{{ .CommonAnnotations.summary }}"
                  severity    = "critical"
                }
              ]
              slack_configs = [
                {
                  api_url  = var.slack_webhook_url
                  channel  = "#incidents-p1"
                  title    = "🚨 CRITICAL: {{ .CommonAnnotations.summary }}"
                  text     = "{{ .CommonAnnotations.description }}"
                  color    = "#FF0000"
                  send_resolved = true
                }
              ]
            },
            {
              name = "slack-warning"
              slack_configs = [
                {
                  api_url  = var.slack_webhook_url
                  channel  = "#alerts-warning"
                  title    = "⚠️ {{ .CommonAnnotations.summary }}"
                  text     = "{{ .CommonAnnotations.description }}"
                  color    = "#FF8C00"
                  send_resolved = true
                }
              ]
            }
          ]
        }
      }

      # Grafana configuration
      grafana = {
        enabled       = true
        adminPassword = var.grafana_admin_password

        persistence = {
          enabled          = true
          storageClassName = "gp3"
          size             = "20Gi"
        }

        # Grafana datasources
        additionalDataSources = [
          {
            name      = "CloudWatch"
            type      = "cloudwatch"
            jsonData = {
              authType      = "default"
              defaultRegion = var.aws_region
            }
          }
        ]

        # Grafana plugins
        plugins = [
          "grafana-piechart-panel",
          "grafana-clock-panel",
          "grafana-worldmap-panel"
        ]

        # Grafana ingress
        ingress = {
          enabled = true
          annotations = {
            "kubernetes.io/ingress.class"                = "alb"
            "alb.ingress.kubernetes.io/scheme"          = "internal"
            "alb.ingress.kubernetes.io/certificate-arn" = var.acm_certificate_arn
          }
          hosts = ["grafana.internal.company.com"]
          tls   = [{ hosts = ["grafana.internal.company.com"], secretName = "grafana-tls" }]
        }
      }

      # Node exporter for host-level metrics
      nodeExporter = {
        enabled = true
        hostNetwork = true
        hostPID     = true
      }

      # kube-state-metrics for Kubernetes object metrics
      kubeStateMetrics = {
        enabled = true
      }
    })
  ]
}
```

---

## 📊 Grafana Dashboards

### Essential Dashboard Panels

```json
// dashboard-api-service.json (excerpt — key panels)
{
  "title": "API Service — Production Dashboard",
  "uid": "api-service-prod",
  "panels": [
    {
      "title": "Request Rate (RPS)",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{service=\"my-api\"}[5m]))",
        "legendFormat": "Requests/s"
      }]
    },
    {
      "title": "Error Rate (%)",
      "type": "gauge",
      "fieldConfig": {
        "thresholds": {
          "steps": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 0.5},
            {"color": "red", "value": 1}
          ]
        }
      },
      "targets": [{
        "expr": "(sum(rate(http_requests_total{service=\"my-api\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"my-api\"}[5m]))) * 100",
        "legendFormat": "Error Rate %"
      }]
    },
    {
      "title": "Latency Percentiles (p50, p95, p99)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{service=\"my-api\"}[5m])) by (le))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=\"my-api\"}[5m])) by (le))",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"my-api\"}[5m])) by (le))",
          "legendFormat": "p99"
        }
      ]
    },
    {
      "title": "Error Budget Remaining",
      "type": "gauge",
      "targets": [{
        "expr": "(1 - (sum(rate(http_requests_total{service=\"my-api\",status=~\"5..\"}[30d])) / sum(rate(http_requests_total{service=\"my-api\"}[30d])))) / (1 - 0.999) * 100",
        "legendFormat": "Error Budget %"
      }]
    }
  ]
}
```

---

## 🔐 Security Perspective (DevSecOps)

### Security Alerting: Treat Incidents as Potential Breaches

```
SECURITY INCIDENT INDICATORS IN MONITORING
───────────────────────────────────────────
• Sudden spike in 401/403 errors → credential stuffing or broken auth
• Unusual API path patterns (/../, %00, SQL syntax) → injection attempts
• Anomalous egress traffic volume → data exfiltration
• CloudTrail: unusual IAM role assumptions from unknown IPs
• WAF block rate spike → active attack
• Lambda invocation from unexpected source ARN
• GuardDuty finding → escalate to security team immediately
```

### AWS GuardDuty + CloudWatch Integration

```bash
# GuardDuty findings → EventBridge → SNS → Security Team
# This is separate from application alerting

# Create EventBridge rule to capture GuardDuty HIGH/CRITICAL findings
aws events put-rule \
  --name "GuardDuty-High-Severity" \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{"numeric": [">=", 7]}]
    }
  }' \
  --state ENABLED

# Route to Security SNS topic
aws events put-targets \
  --rule "GuardDuty-High-Severity" \
  --targets '[
    {
      "Id": "SecuritySNS",
      "Arn": "arn:aws:sns:us-east-1:123456789012:security-incidents"
    }
  ]'
```

### Terraform: GuardDuty Alerting

```hcl
# security/guardduty-alerting.tf

resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name        = "guardduty-high-severity-findings"
  description = "Capture GuardDuty findings severity >= 7 (HIGH/CRITICAL)"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_to_sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_high.name
  target_id = "SecuritySNSTopic"
  arn       = aws_sns_topic.security.arn

  # Transform the event to include only relevant fields
  input_transformer {
    input_paths = {
      severity    = "$.detail.severity"
      type        = "$.detail.type"
      description = "$.detail.description"
      account     = "$.detail.accountId"
      region      = "$.region"
      findingId   = "$.detail.id"
    }
    input_template = <<-EOT
      {
        "AlarmName": "GuardDuty-Finding",
        "Severity": <severity>,
        "Type": "<type>",
        "Description": "<description>",
        "AccountId": "<account>",
        "Region": "<region>",
        "FindingId": "<findingId>",
        "ConsoleLink": "https://console.aws.amazon.com/guardduty/home?region=<region>#/findings?macros=current&fId=<findingId>"
      }
    EOT
  }
}

# Security team SNS topic (separate from application alerts)
resource "aws_sns_topic" "security" {
  name              = "security-incidents"
  kms_master_key_id = aws_kms_key.sns_security.id

  tags = {
    Classification = "Security"
    DataSensitivity = "High"
  }
}

resource "aws_sns_topic_subscription" "security_email" {
  topic_arn = aws_sns_topic.security.arn
  protocol  = "email"
  endpoint  = "security-team@company.com"
}

# Also integrate with PagerDuty security rotation
resource "aws_sns_topic_subscription" "security_pagerduty" {
  topic_arn = aws_sns_topic.security.arn
  protocol  = "https"
  endpoint  = var.security_pagerduty_endpoint
}
```

### IAM Least-Privilege for Monitoring

```hcl
# IAM role for monitoring Lambda (pushes custom metrics, reads logs)
resource "aws_iam_role" "monitoring_lambda" {
  name = "monitoring-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "monitoring_lambda" {
  name = "monitoring-lambda-policy"
  role = aws_iam_role.monitoring_lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow publishing custom metrics
      {
        Sid      = "CloudWatchMetrics"
        Effect   = "Allow"
        Action   = ["cloudwatch:PutMetricData"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "cloudwatch:namespace" = ["MyApp/Monitoring", "MyApp/SLO"]
          }
        }
      },
      # Allow reading logs for analysis
      {
        Sid    = "CloudWatchLogsRead"
        Effect = "Allow"
        Action = [
          "logs:GetLogEvents",
          "logs:FilterLogEvents",
          "logs:StartQuery",
          "logs:GetQueryResults"
        ]
        Resource = "arn:aws:logs:*:${data.aws_caller_identity.current.account_id}:log-group:/aws/app/*"
      },
      # Allow publishing to SNS alert topics
      {
        Sid      = "SNSPublish"
        Effect   = "Allow"
        Action   = ["sns:Publish"]
        Resource = [
          aws_sns_topic.critical.arn,
          aws_sns_topic.warning.arn
        ]
      },
      # Allow using KMS key for SNS encryption
      {
        Sid    = "KMSForSNS"
        Effect = "Allow"
        Action = ["kms:GenerateDataKey*", "kms:Decrypt"]
        Resource = aws_kms_key.sns_alerts.arn
      },
      # CloudWatch Logs for Lambda execution
      {
        Sid    = "LambdaLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/aws/lambda/monitoring-*"
      }
    ]
  })
}
```

---

## 📟 Runbook: P1 Incident Response

```markdown
# Runbook: P1 — Production Service Outage
**Last Updated:** 2024-03-01 | **Owner:** Platform Team

## Pre-conditions
- You have been paged by PagerDuty for a P1 alert
- This runbook assumes access to AWS Console, kubectl (if K8s), and Slack

## Step 1: Acknowledge (< 5 minutes)
1. Acknowledge the PagerDuty alert
2. Join #incidents-p1 Slack channel (or create: /incident create P1 <description>)
3. Post: "IC: @yourusername — investigating [service] — ETA for first update: 15 min"
4. Page Incident Commander if this is not you: @oncall-ic

## Step 2: Assess Scope (< 10 minutes)
```bash
# Quick scope check
# What is broken?
aws elbv2 describe-target-health --target-group-arn <TG_ARN>
# When did it start?
aws cloudwatch get-metric-statistics --metric-name HTTPCode_Target_5XX_Count ...
# What changed recently?
aws codedeploy list-deployments --application-name <app> --create-time-range start=$(date -d '-2h' --iso-8601=seconds),end=$(date --iso-8601=seconds)
```

## Step 3: Check Recent Changes (< 15 minutes)
- [ ] Was there a deployment in the last 2 hours? → Consider rollback immediately
- [ ] Was there a config change (Terraform apply)? → Check Terraform state history
- [ ] Did any database migration run? → Check migration logs
- [ ] Any scheduled job that runs at this time? → Check EventBridge rules
- [ ] AWS service health event? → Check https://health.aws.amazon.com/health/status

## Step 4: Rollback (if applicable)
```bash
# ECS rollback
aws ecs update-service --cluster prod --service <name> --task-definition <name>:<previous_rev> --force-new-deployment
# Kubernetes rollback
kubectl rollout undo deployment/<name> -n production
```

## Step 5: Communicate Every 15 Minutes
- Update #incidents-p1: current status, what you're investigating, ETA
- If >30 minutes: notify VP Engineering
- If >1 hour: notify CTO + post on status page

## Step 6: Resolve
- Confirm error rate returns to baseline
- Confirm CloudWatch alarms return to OK
- Run synthetic tests manually
- Post resolution message to Slack + update status page
- File postmortem ticket (24-hour deadline)
```

---

## 🤖 AI & 2024 Trends in Incident Management

### 1. AIOps — ML-Driven Anomaly Detection

Modern platforms use machine learning to baseline normal behaviour and alert on **deviations** rather than static thresholds. This dramatically reduces false positives.

```bash
# AWS DevOps Guru — AI-powered operational insights
# Enable DevOps Guru for your account (covers all supported services)
aws devops-guru update-service-integration \
  --service-integration '{
    "OpsCenter": {"OptInStatus": "ENABLED"},
    "LogsAnomalyDetection": {"OptInStatus": "ENABLED"}
  }'

# DevOps Guru automatically:
# • Learns normal metric patterns using ML
# • Groups related anomalies into a single "insight"
# • Predicts impending issues before they become outages
# • Recommends specific runbook actions

# List anomalous behavior detected by DevOps Guru
aws devops-guru list-insights \
  --status-filter '{"Any": {"Type": "REACTIVE", "StartTimeRange": {"FromTime": 1700000000, "ToTime": 1700100000}}}' \
  --max-results 10
```

### 2. CloudWatch Anomaly Detection

```bash
# Create anomaly detection band on a metric (no static threshold needed)
aws cloudwatch put-anomaly-detector \
  --namespace "AWS/ApplicationELB" \
  --metric-name "TargetResponseTime" \
  --dimensions '[{"Name": "LoadBalancer", "Value": "app/my-alb/abc"}]' \
  --stat "Average" \
  --configuration '{"ExcludedTimeRanges": [], "MetricTimezone": "UTC"}'

# Alarm on anomaly (outside the expected band × 2 std deviations)
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-Latency-Anomaly" \
  --comparison-operator GreaterThanUpperThreshold \
  --evaluation-periods 2 \
  --metrics '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "TargetResponseTime",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/my-alb/abc"}]
        },
        "Period": 300,
        "Stat": "Average"
      }
    },
    {
      "Id": "ad1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "Label": "LatencyAnomalyBand"
    }
  ]' \
  --threshold-metric-id ad1 \
  --alarm-actions arn:aws:sns:us-east-1:123:incident-alerts
```

### 3. OpenTelemetry — Vendor-Neutral Observability (2024 Standard)

```yaml
# otel-collector-config.yaml — OTEL Collector as sidecar/DaemonSet
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  # Auto-instrument Kubernetes
  kubeletstats:
    collection_interval: 20s
    auth_type: "serviceAccount"
    endpoint: "${env:K8S_NODE_NAME}:10250"

  # Host metrics
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu: {}
      memory: {}
      disk: {}
      network: {}

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  
  memory_limiter:
    limit_percentage: 75
    check_interval: 1s
  
  # Add Kubernetes metadata
  k8sattributes:
    auth_type: serviceAccount
    extract:
      metadata:
        - k8s.pod.name
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.node.name

exporters:
  # AWS CloudWatch
  awscloudwatch:
    region: us-east-1
    namespace: "OTEL/Application"
    log_group_name: "/otel/application"
  
  # Prometheus remote write (to Amazon Managed Prometheus)
  prometheusremotewrite:
    endpoint: "https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write"
    auth:
      authenticator: sigv4auth
  
  # AWS X-Ray for traces
  awsxray:
    region: us-east-1

service:
  extensions: [sigv4auth]
  pipelines:
    metrics:
      receivers: [otlp, hostmetrics, kubeletstats]
      processors: [memory_limiter, batch, k8sattributes]
      exporters: [awscloudwatch, prometheusremotewrite]
    
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, k8sattributes]
      exporters: [awsxray]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [awscloudwatch]
```

### 4. Incident Management Platforms (2024)

| Platform | Unique Value | Best For |
|----------|-------------|----------|
| **PagerDuty** | Industry standard, complex routing | Enterprises, mature on-call |
| **OpsGenie** (Atlassian) | Jira integration, on-call schedules | Teams using Jira |
| **FireHydrant** | Automated war room, retroactive timelines | Startups, fast-growing teams |
| **Incident.io** | Slack-native, streamlined flow | Modern, Slack-first teams |
| **Rootly** | AI-powered postmortems | Teams wanting auto-summaries |
| **VictorOps** (Splunk) | Log-metric correlation | Splunk shops |

---

## 🎓 SLO-Based Alerting (Google SRE Approach)

### Why SLO Alerting is Better Than Threshold Alerting

Traditional alerting: "CPU > 80% for 5 minutes" → Many false positives  
SLO alerting: "Error budget burning 14× faster than normal" → Only page when it really matters

### Defining SLOs in Code

```yaml
# slo.yaml — SLO definitions (can be consumed by sloth or pyrra tools)
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: my-api-slo
  namespace: monitoring
spec:
  service: "my-api"
  labels:
    team: "platform"
    env: "production"

  slos:
    # Availability SLO: 99.9% of requests succeed
    - name: "availability"
      objective: 99.9
      description: "99.9% of all requests return non-5xx responses"
      sli:
        events:
          errorQuery: |
            sum(rate(http_requests_total{service="my-api",status=~"5.."}[{{.window}}]))
          totalQuery: |
            sum(rate(http_requests_total{service="my-api"}[{{.window}}]))
      alerting:
        name: MyApiAvailabilityAlert
        labels:
          category: "availability"
        annotations:
          summary: "My API availability SLO burning"
          runbook: "https://wiki.company.com/runbooks/api-availability"
        pageAlert:
          labels:
            priority: P1
        ticketAlert:
          labels:
            priority: P2

    # Latency SLO: 99% of requests served in < 500ms
    - name: "latency"
      objective: 99.0
      description: "99% of requests served within 500ms"
      sli:
        events:
          errorQuery: |
            sum(rate(http_request_duration_seconds_bucket{service="my-api",le="0.5"}[{{.window}}]))
          totalQuery: |
            sum(rate(http_request_duration_seconds_count{service="my-api"}[{{.window}}]))
      alerting:
        name: MyApiLatencyAlert
        labels:
          category: "latency"
        pageAlert:
          labels:
            priority: P1
        ticketAlert:
          labels:
            priority: P2
```

---

## 🔄 Consistency Across Teams

### Standardisation Strategy

```
ORGANISATION-WIDE MONITORING STANDARDS
───────────────────────────────────────
1. NAMING CONVENTIONS
   • Alarm names: {service}-{metric}-{severity}
   • Example: my-api-error-rate-critical
   • Slack channel: #alerts-{severity} (p1, p2, p3)
   • Incident channel: #inc-{YYYY-MM-DD}-{description}

2. REQUIRED METRICS (every service MUST emit)
   • http_requests_total{method, path, status}
   • http_request_duration_seconds (histogram)
   • process_cpu_seconds_total
   • process_resident_memory_bytes
   • custom_errors_total{type, service}
   • db_query_duration_seconds (histogram)

3. REQUIRED DASHBOARDS (every service MUST have)
   • Service overview (golden signals)
   • Error budget / SLO dashboard
   • Infrastructure metrics
   • Database metrics (if applicable)

4. REQUIRED RUNBOOKS (every P1 alarm MUST have)
   • Link in annotation: runbook_url
   • Contents: what the alert means, immediate steps, escalation path

5. ALERT HYGIENE
   • Review alert noise monthly
   • Auto-silence repeated alerts after X occurrences (investigate instead)
   • Alert coverage target: mean time to detect <5 minutes
```

### Service Mesh Monitoring (Istio / AWS App Mesh)

```yaml
# Istio generates L7 metrics automatically — no code changes needed
# These PromQL queries work for any Istio-instrumented service

# Request rate per service
sum(rate(istio_requests_total{destination_service_name="my-api"}[5m])) by (destination_service_name)

# Error rate (5xx from upstream)
sum(rate(istio_requests_total{destination_service_name="my-api",response_code=~"5.."}[5m]))
/
sum(rate(istio_requests_total{destination_service_name="my-api"}[5m]))

# p99 latency between services
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{destination_service_name="my-api"}[5m])) by (le)
)
```

---

## 🛡️ Security Controls Checklist for Monitoring

```
MONITORING SECURITY CHECKLIST
──────────────────────────────
[ ] CloudWatch logs encrypted with KMS (CMK, not default key)
[ ] SNS topics encrypted with KMS
[ ] Prometheus scrape endpoints protected (mTLS or network policy)
[ ] Grafana behind SSO (SAML/OIDC — not username/password)
[ ] Grafana datasource connections use IAM roles (not long-lived keys)
[ ] Alert notification webhooks use HTTPS + secret validation
[ ] PagerDuty integration uses v2 Events API (not deprecated v1)
[ ] CloudTrail logging enabled for all CloudWatch API calls
[ ] No sensitive data (PII, credentials) in metric labels or log messages
[ ] Metric namespaces use SCPs to restrict who can publish
[ ] Dashboards with cost/revenue data access-controlled (Grafana RBAC)
[ ] GuardDuty enabled and findings routed to security team
[ ] Security Hub + AWS Config enabled for compliance monitoring
[ ] VPC Flow Logs enabled (for network anomaly detection)
[ ] CloudWatch agent uses instance profile (not access keys on instances)
```

---

## 📋 Interview Q&A

**Q1: What's the difference between monitoring and observability?**

> Monitoring is about tracking known failure modes — you define metrics, set thresholds, and alert on breaches. Observability is broader: it's the ability to understand the internal state of a complex system from its external outputs (metrics, logs, and traces together). Monitoring answers "is the system healthy?" while observability answers "why is the system behaving this way?" Modern distributed systems need observability because you can't predict all failure modes in advance.

**Q2: How do you reduce alert fatigue?**

> Several strategies: First, use **composite alarms** so a page only fires when multiple signals confirm a real problem. Second, adopt **SLO-based burn rate alerting** — only page when the error budget is burning fast enough to matter. Third, regularly review alert history and tune or silence alerts that fire without requiring human action. Fourth, implement **runbook auto-remediation** for known issues (Lambda auto-restart a crashed service). Fifth, distinguish between "page" alerts (requiring human action now) and "ticket" alerts (fix next business day). The goal is: every page requires human judgment; everything else creates a ticket.

**Q3: What's in a good postmortem?**

> A good postmortem is blameless and focused on systemic improvement. It covers: (1) impact quantified in business terms, (2) precise timeline reconstructed from logs and metrics, (3) root cause AND contributing factors (usually there are 3-5 factors, not just one), (4) what went well, (5) action items with specific owners and due dates. The most important part is the action items being acted on — postmortems that produce no change are wasted effort.

**Q4: How do you instrument a new microservice for observability?**

> The standard approach in 2024 is OpenTelemetry. I instrument the application with the OTEL SDK, which auto-instruments common frameworks (HTTP, database, gRPC) and allows custom spans. The OTEL Collector aggregates telemetry and exports to our backends — CloudWatch for AWS-native metrics, Amazon Managed Prometheus for time-series, and AWS X-Ray for traces. I ensure the service emits the four golden signals from day one. Before the service reaches production, I create a Grafana dashboard and at minimum two CloudWatch alarms (error rate + latency), and write a runbook for each.

**Q5: How do you handle an alert that fires at 3 AM but resolves itself before you can investigate?**

> Three actions: (1) Acknowledge the alert so the rotation isn't woken up again. (2) Document the time, duration, and any metrics around it. (3) During business hours, investigate the root cause — a self-healing incident often means an auto-scaling event covered a capacity problem, or a retry loop recovered a transient failure. Self-resolving alerts are a signal that something **almost** went wrong. Track them in a weekly review. If it's happening frequently, either fix the underlying issue or add automation so the system recovers faster and the human pager fires only when the automation fails.

**Q6: What is SLO burn rate alerting and why is it better?**

> SLO burn rate alerting measures how fast you're consuming your error budget relative to the rate that would deplete it over the SLO window. For a 99.9% SLO (monthly), your error budget is 43.8 minutes/month. If you're burning errors at 14× the "normal" rate, you'll exhaust the budget in ~3 hours. That's worth waking someone up. If you're burning at 2× the normal rate, you'll exhaust in ~10 days — worth a ticket but not a 3 AM page. This approach is better because it aligns alerting with actual user impact and business commitments, rather than arbitrary CPU/latency thresholds that may not reflect real user pain.

**Q7: Describe how you'd set up monitoring for a new AWS EKS cluster from scratch.**

> I follow this sequence: (1) Deploy kube-prometheus-stack via Helm — this gives Prometheus, Alertmanager, node-exporter, kube-state-metrics, and Grafana. (2) Enable Amazon Managed Prometheus (AMP) and configure remote write from Prometheus for long-term retention. (3) Deploy the OTEL Collector as a DaemonSet to collect host metrics and forward to CloudWatch Container Insights. (4) Enable CloudWatch Container Insights in the EKS cluster for AWS-console visibility. (5) Deploy the AWS Load Balancer Controller and configure Ingress with ALB for external traffic. (6) Create CloudWatch alarms on the key EKS and ALB metrics. (7) Import the official Grafana Kubernetes dashboards (IDs 315, 6417, 13770). (8) Define alert rules for pod crash-loops, node resource pressure, and SLO burn rates. (9) Test the alerting end-to-end by deliberately crashing a pod and confirming PagerDuty receives the alert.

---

## 💡 30-Second Summary (For the Interview)

```
INCIDENT MANAGEMENT — THE COMPLETE LOOP

Application / Infrastructure
        │
        ├─ Emits METRICS ──────────────────► CloudWatch / Prometheus
        │                                          │
        ├─ Emits LOGS ────────────────────► CloudWatch Logs / Loki
        │                                          │
        └─ Emits TRACES ──────────────────► AWS X-Ray / Jaeger
                                                   │
                                         ALERT RULES evaluate
                                         thresholds / SLOs
                                                   │
                                         ALERTMANAGER / CloudWatch
                                         routes to correct receiver
                                                   │
                          ┌────────────────────────┴───────────────────────┐
                          ▼                                                 ▼
                   PagerDuty                                          Slack channel
                 (phone call)                                    (context + runbook)
                          │
                          ▼
                   On-call engineer
                   acknowledges → investigates → resolves
                          │
                          ▼
                   POSTMORTEM (within 48h)
                   → Root cause
                   → Action items
                   → Improved system
                   → Reduced MTTR / MTTD
```

---

## 🔗 Reference Links & Further Reading

| Resource | URL |
|----------|-----|
| Google SRE Book (free online) | https://sre.google/sre-book/table-of-contents/ |
| AWS CloudWatch Documentation | https://docs.aws.amazon.com/cloudwatch/ |
| Prometheus Best Practices | https://prometheus.io/docs/practices/ |
| Alertmanager Configuration | https://prometheus.io/docs/alerting/latest/configuration/ |
| OpenTelemetry Documentation | https://opentelemetry.io/docs/ |
| AWS DevOps Guru | https://aws.amazon.com/devops-guru/ |
| PagerDuty Incident Response Guide | https://response.pagerduty.com/ |
| KodeKloud DevOps Interview Course | https://learn.kodekloud.com/user/courses/devops-interview-preparation-course/module/013fead8-a37c-42eb-83ec-e691a1238d08/lesson/4ce58a54-f444-4e19-a86d-d4239bb2c1e6 |

---

> **Images from KodeKloud:**
> ![Monitoring and Alerting System Flowchart](https://kodekloud.com/kk-media/image/upload/v1752873336/notes-assets/images/DevOps-Interview-Preparation-Course-DevOps-Question-1/monitoring-alerting-system-flowchart.jpg)
> ![Incident Management Handwritten Flowchart](https://kodekloud.com/kk-media/image/upload/v1752873337/notes-assets/images/DevOps-Interview-Preparation-Course-DevOps-Question-1/handwritten-flowchart-process-analysis.jpg)

---

*Next: DevOps-Q2.md — [Next Topic in DevOps Miscellaneous Series]*
