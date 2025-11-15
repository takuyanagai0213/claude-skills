---
name: incident-response-guide
description: Incident response workflow assistant for investigating alerts and system issues. Automatically activates when user requests incident investigation, alert analysis, or troubleshooting guidance.
---

# Incident Response Guide Skill

This skill assists with systematic incident investigation and response by providing step-by-step guidance and templates.

## When to Use

This skill automatically activates when the user makes requests like:
- "Investigate this alert"
- "アラート対応して"
- "Troubleshoot this incident"
- "How to investigate this error"
- "インシデント調査"

## Customization (Optional)

This skill works out-of-the-box with any GCP project. Optionally, you can add project-specific information to this section:

- **Project IDs**: List your production/staging project IDs
- **Common Services**: Frequently investigated service names
- **Escalation Contacts**: On-call rotation, team leads

**Note**: Detailed information (alert policies, service names, load balancers) should be retrieved dynamically using MCP tools during investigation to avoid stale information.

---

## Prerequisites: Environment Setup

**IMPORTANT: Before starting investigation, ensure you're in the correct GCP context.**

### Step -1: Verify GCP Project Context

**Check Current Project:**

Ask Claude to verify which project you're currently using:
```
"What project am I currently using?"
"Show current gcloud configuration"
```

Claude will use:
```
run_gcloud_command(args=["config", "get-value", "project"])
run_gcloud_command(args=["config", "list"])
```

**Expected Output:**
Verify the project ID matches the environment you want to investigate.

**If you need to switch to a different project:**

```
"Switch to project [PROJECT_ID]"
```

Claude will use:
```
run_gcloud_command(args=["config", "set", "project", "[PROJECT_ID]"])
```

Or if you use gcloud configurations:
```bash
# List your available configurations
gcloud config configurations list

# Activate the appropriate configuration
gcloud config configurations activate [YOUR_CONFIG_NAME]

# Verify
gcloud config get-value project
```

**⚠️ Common Mistake:**
- Investigating the wrong environment (dev/staging instead of production)
- Using personal project instead of organization project
- Not authenticating before investigation

---

## Execution Steps

### 0. Alert Confirmation (First Step)

**Confirm which alerts fired and their current status:**

**Using Observability MCP:**

Ask Claude to list recent alerts:
```
"Show me recent alerts for project [PROJECT_ID]"
"Show alerts in the last hour"
"Show OPEN alerts ordered by severity"
```

Claude will use:
```
list_alerts(
  parent="projects/[PROJECT_ID]",
  filter='state="OPEN"',
  orderBy="open_time desc"
)
```

**Key Information to Extract:**
- [ ] Alert policy name
- [ ] Alert state (OPEN/CLOSED)
- [ ] Severity/priority
- [ ] When it opened (first occurrence)
- [ ] Affected resources (service, region, instance)
- [ ] Condition that triggered it

**Example:**
```
You: "Show me alerts that fired in the last hour"

Claude returns:
- Alert: "High Error Rate - onetag-tracker"
- State: OPEN
- Opened: 2025-11-13T22:09:31Z
- Resource: Cloud Run service "onetag-tracker"
- Condition: Error rate > 5% for 5 minutes
```

### 1. Initial Assessment

Using the alert information from Step 0, gather context:

**Alert Information (from Step 0):**
- [✓] Alert name/title
- [✓] Severity level (Critical/High/Medium/Low)
- [✓] First occurrence time
- [✓] Affected service/component
- [✓] Error message or symptom

**Quick Impact Check:**
- [ ] Is the service down?
- [ ] Are users affected?
- [ ] Is data at risk?
- [ ] Is this recurring or first-time?

**Check Alert History:**
```
"Has this alert fired before in the last week?"
"Show alert history for [ALERT_POLICY_NAME]"
```

### 2. Investigation Checklist

Follow these steps systematically:

#### Phase 1: Immediate Context (First 5 minutes)

**Logs Review:**
```
1. Check application logs around incident time
2. Check error logs for stack traces
3. Check access logs for unusual patterns
4. Note any ERROR or CRITICAL level messages
```

**Using Observability MCP:**

Ask Claude to search logs using the `list_log_entries` tool:
```
"Search logs for errors in [SERVICE_NAME] between [START_TIME] and [END_TIME]"
```

Claude will use:
```
list_log_entries(
  resourceNames=["projects/[PROJECT_ID]"],
  filter='resource.labels.service_name="[SERVICE_NAME]" severity>=ERROR timestamp>="[START_TIME]" timestamp<="[END_TIME]"',
  orderBy="timestamp desc"
)
```

**Manual GCP Logging Queries (fallback):**
```
# Application errors
resource.labels.service_name="[SERVICE_NAME]"
severity>=ERROR
timestamp >= "[INCIDENT_TIME - 5min]"
timestamp <= "[INCIDENT_TIME + 5min]"

# Load balancer logs
resource.type="http_load_balancer"
timestamp >= "[INCIDENT_TIME - 5min]"
timestamp <= "[INCIDENT_TIME + 5min]"
```

#### Phase 2: Detailed Analysis (5-15 minutes)

**Trace Analysis:**
- [ ] Find trace ID from logs
- [ ] Get detailed trace information
- [ ] Identify slow or failed spans
- [ ] Check service dependencies

**Using Observability MCP for Traces:**

Ask Claude to analyze traces:
```
"List traces for [PROJECT_ID] with errors between [START_TIME] and [END_TIME]"
"Get details for trace [TRACE_ID]"
```

Claude will use:
```
list_traces(
  projectId="[PROJECT_ID]",
  startTime="[START_TIME]",
  endTime="[END_TIME]",
  filter="latency:500ms"  # or other filters
)

get_trace(
  projectId="[PROJECT_ID]",
  traceId="[TRACE_ID]"
)
```

**Metrics Check:**
- [ ] CPU/Memory usage around incident time
- [ ] Request rate and latency
- [ ] Error rate spike
- [ ] Database connection pool

**Using Observability MCP for Metrics:**

Ask Claude to fetch metrics:
```
"Show me CPU usage for [SERVICE] around [TIME]"
"Get error rate for [SERVICE] in the last hour"
```

Claude will use:
```
list_time_series(
  name="projects/[PROJECT_ID]",
  filter='metric.type="compute.googleapis.com/instance/cpu/usage_time"',
  interval={
    "startTime": "[START_TIME]",
    "endTime": "[END_TIME]"
  }
)
```

**Related Systems:**
- [ ] Database status (PostgreSQL, BigQuery)
- [ ] Cache status (Redis)
- [ ] External API status
- [ ] Network connectivity

**Using gcloud MCP for Service Status:**

Ask Claude to check service status:
```
"Check Cloud Run service status for [SERVICE_NAME]"
"List Cloud Functions in region [REGION]"
"Show GKE workloads in cluster [CLUSTER_NAME]"
```

Claude will use:
```
run_gcloud_command(args=["run", "services", "describe", "[SERVICE_NAME]", "--region=[REGION]"])
run_gcloud_command(args=["functions", "list", "--region=[REGION]"])
run_gcloud_command(args=["container", "clusters", "get-credentials", "[CLUSTER_NAME]"])
```

#### Phase 3: Pattern Recognition (15-30 minutes)

**Historical Comparison:**
- [ ] Has this happened before?
- [ ] Check similar incidents
- [ ] Review recent deployments
- [ ] Check configuration changes

**Scope Assessment:**
- [ ] Single instance or multiple?
- [ ] Single region or multiple?
- [ ] Specific user segment affected?
- [ ] Time pattern (specific hours/days)?

### 3. Root Cause Analysis

Use this framework to identify the root cause:

**Common Patterns:**

1. **Network Issues**
   - DNS resolution failure
   - Connection timeout
   - Firewall/routing problems
   - Regional connectivity issues

2. **Resource Exhaustion**
   - Out of memory
   - CPU throttling
   - Connection pool exhausted
   - Disk space full

3. **Code Issues**
   - Unhandled exceptions
   - Null pointer errors
   - Infinite loops
   - Memory leaks

4. **External Dependencies**
   - Third-party API failure
   - Database unavailability
   - CDN issues
   - Authentication service down

5. **Configuration Issues**
   - Wrong environment variables
   - Incorrect feature flags
   - Bad deployment
   - Schema migration problems

### 4. Document Findings

**Investigation Documentation:**
- [ ] Create incident report using the template below
- [ ] Document root cause analysis
- [ ] Record timeline of events
- [ ] Save relevant logs, traces, and metrics
- [ ] Notify stakeholders of investigation findings

**Note:** This skill focuses on investigation and analysis only. For incident remediation (rollback, scaling, configuration changes), refer to your organization's incident response procedures or dedicated remediation workflows.

### 5. Escalation Criteria

**Escalate immediately if:**
- Service is completely down
- Data loss or corruption suspected
- Security breach detected
- Multiple critical systems affected
- Unable to determine cause within 30 minutes

**Escalation Contacts:**
- On-call engineer: [Contact info]
- Team lead: [Contact info]
- Platform team: [Contact info]

## Investigation Report Template

A template file `incident-report-template.md` is provided in this directory.

Copy it to create a new investigation report:

```bash
cp .claude/skills/incident-response-guide/incident-report-template.md researche-YYYY-MM-DD.md
```

Then fill in the sections with your findings.

## GCP Investigation URLs

**Quick Access Links:**

**Cloud Logging:**
```
https://console.cloud.google.com/logs/query
```

**Cloud Trace:**
```
https://console.cloud.google.com/traces/list
```

**Error Reporting:**
```
https://console.cloud.google.com/errors
```

**Cloud Monitoring:**
```
https://console.cloud.google.com/monitoring
```

**Cloud Functions:**
```
https://console.cloud.google.com/functions/list
```

**GKE Workloads:**
```
https://console.cloud.google.com/kubernetes/workload
```

## MCP Tools Quick Reference

### Observability MCP Tools

**Search Logs:**
```
"Search logs for [query] in project [PROJECT_ID]"
```
Uses: `list_log_entries(resourceNames, filter, orderBy)`

**List Traces:**
```
"Show traces for [PROJECT_ID] between [START] and [END]"
```
Uses: `list_traces(projectId, startTime, endTime, filter)`

**Get Trace Details:**
```
"Get trace details for [TRACE_ID]"
```
Uses: `get_trace(projectId, traceId)`

**Check Metrics:**
```
"Show [METRIC_TYPE] for [RESOURCE] between [START] and [END]"
```
Uses: `list_time_series(name, filter, interval)`

**List Alerts:**
```
"Show recent alerts for project [PROJECT_ID]"
```
Uses: `list_alerts(parent, filter, orderBy)`

**List Alert Policies:**
```
"Show alert policies for project [PROJECT_ID]"
```
Uses: `list_alert_policies(name, filter)`

### gcloud MCP Tools

**Cloud Run Services:**
```
"Describe Cloud Run service [SERVICE_NAME] in [REGION]"
```
Uses: `run_gcloud_command(args=["run", "services", "describe", ...])`

**Cloud Functions:**
```
"List Cloud Functions in [REGION]"
```
Uses: `run_gcloud_command(args=["functions", "list", ...])`

**GKE Clusters:**
```
"Show GKE cluster [CLUSTER_NAME] details"
```
Uses: `run_gcloud_command(args=["container", "clusters", "describe", ...])`

**Recent Deployments:**
```
"Show recent Cloud Run revisions for [SERVICE_NAME]"
```
Uses: `run_gcloud_command(args=["run", "revisions", "list", ...])`

## Useful Log Queries

**Recent errors across all services:**
```
severity>=ERROR
timestamp >= "[START_TIME]"
```

**Specific service errors:**
```
resource.labels.service_name="[SERVICE_NAME]"
severity>=ERROR
```

**Load balancer 5xx errors:**
```
resource.type="http_load_balancer"
httpRequest.status>=500
```

**Function execution errors:**
```
resource.type="cloud_function"
severity>=ERROR
```

**Tracker service logs:**
```
resource.type="k8s_container"
resource.labels.container_name="onetag-tracker"
```

## Tips for Effective Investigation

### Do's ✅
- Start with broad queries, then narrow down
- Document timeline of events
- Save relevant log entries and traces
- Compare with normal behavior
- Check recent changes (deployments, configs)
- Look at the big picture (not just one error)

### Don'ts ❌
- Don't jump to conclusions without evidence
- Don't ignore context (time, region, user)
- Don't forget to check dependencies
- Don't skip documentation
- Don't panic - follow the checklist

## Common Investigation Patterns

### Pattern 1: Single Failed Request

**Using MCP Tools:**
1. **Find error in logs:**
   ```
   "Search logs for errors around [TIMESTAMP] in [SERVICE_NAME]"
   ```
2. **Get trace details:**
   ```
   "Get trace details for [TRACE_ID]"
   ```
3. **Check service logs:**
   ```
   "Show logs for [SERVICE_NAME] at [TIMESTAMP]"
   ```
4. **Determine pattern:**
   ```
   "Search for similar errors in the last hour"
   ```

**Example Commands:**
```
"Search logs for severity>=ERROR in onetag-tracker between 2025-11-13T22:09:00Z and 2025-11-13T22:10:00Z"
"Get trace e021813b4a7019c52db0bd308f4ef688 details"
```

### Pattern 2: Service Degradation

**Using MCP Tools:**
1. **Check error rate:**
   ```
   "Show error rate for [SERVICE_NAME] in the last hour"
   ```
2. **Check recent deployments:**
   ```
   "List recent Cloud Run revisions for [SERVICE_NAME]"
   ```
3. **Check resource usage:**
   ```
   "Show CPU and memory usage for [SERVICE_NAME] around [TIME]"
   ```
4. **List recent alerts:**
   ```
   "Show recent alerts for project [PROJECT_ID]"
   ```

**Example Commands:**
```
"List Cloud Run revisions for onetag-tracker --region=asia-northeast1"
"Show alerts in the last 24 hours"
```

### Pattern 3: Complete Outage

**Using MCP Tools:**
1. **Check service status:**
   ```
   "Describe Cloud Run service [SERVICE_NAME] in [REGION]"
   ```
2. **Check load balancer logs:**
   ```
   "Search load balancer logs for 5xx errors in the last 10 minutes"
   ```
3. **Check cluster status:**
   ```
   "Show GKE cluster [CLUSTER_NAME] status"
   ```
4. **Check recent changes:**
   ```
   "List recent deployments for [SERVICE_NAME]"
   ```

**Example Commands:**
```
"Describe Cloud Run service onetag-tracker --region=asia-northeast1"
"Search http_load_balancer logs with status>=500 in the last 10 minutes"
```

## Post-Investigation Recommendations

After completing the investigation, consider the following:

**Documentation & Communication:**
- [ ] Document lessons learned
- [ ] Share findings with team
- [ ] Schedule post-mortem (if major incident)
- [ ] Close related tickets

**Follow-up Considerations (separate workflows):**
- Review if monitoring alerts need adjustment
- Identify potential preventive measures
- Consider runbook updates based on findings

**Note:** Implementation of changes (alert updates, preventive measures, etc.) should follow your organization's change management process.

---

## MCP Tools Usage Tips

### Best Practices

**1. Use Natural Language:**
- You can ask Claude in natural language
- No need to memorize MCP tool parameters
- Claude will translate to appropriate MCP calls

**2. Start Broad, Then Narrow:**
```
✅ Good: "Show me errors in the last hour" → then narrow down
❌ Bad: Immediately searching for specific trace IDs
```

**3. Leverage Multiple Tools:**
```
Step 1: Use observability MCP to find errors
Step 2: Use gcloud MCP to check service status
Step 3: Use observability MCP to get traces
```

**4. Save Important Data:**
- Ask Claude to save trace IDs, log entries
- Document findings as you go
- Use the incident report template

**5. Time Format:**
- Use RFC 3339 format: `2025-11-13T22:09:31Z`
- Or relative: "in the last hour", "since 10 minutes ago"

### Example Investigation Flow

**Scenario: You receive an alert notification**

```
Step -1: Environment Setup
You: "What project am I using?"
→ Claude uses run_gcloud_command(["config", "get-value", "project"])
→ Returns: d2f821de-867e-4f48-9761

You: "Show current gcloud configuration"
→ Claude uses run_gcloud_command(["config", "list"])
→ Returns: Current project confirmed

Step 0: Confirm Alert
You: "Show me alerts in the last hour"
→ Claude uses list_alerts()
→ Returns: "High Error Rate - onetag-tracker" (OPEN, started 22:09 UTC)

Step 1: Initial Assessment
You: "Search for errors in onetag-tracker in the last 10 minutes"
→ Claude uses list_log_entries()
→ Returns: TypeError: fetch failed at 22:09:31

Step 2: Get Trace Details
You: "Get trace details for e021813b4a7019c52db0bd308f4ef688"
→ Claude uses get_trace()
→ Returns: Request failed at fetch step, 0.148s duration

Step 3: Check Service Status
You: "Check Cloud Run service status for onetag-tracker"
→ Claude uses run_gcloud_command()
→ Returns: Service is healthy, revision 00003 serving 100%

Step 4: Check Recent Changes
You: "Show recent deployments for onetag-tracker"
→ Claude uses run_gcloud_command()
→ Returns: No deployments in last 24 hours

Conclusion: Single transient network error, not related to deployment
```

### Advantages of Using MCP Tools

- ✅ **Faster**: No manual console clicking
- ✅ **Documented**: All queries are in chat history
- ✅ **Reproducible**: Can re-run exact same queries
- ✅ **Comprehensive**: Combines logs, traces, metrics in one place
- ✅ **Automated**: Claude can correlate data across tools

---

**Remember:**
- Systematic approach is better than random debugging
- Document as you investigate
- It's okay to ask for help
- Every incident is a learning opportunity
- **Use MCP tools to speed up investigation**
