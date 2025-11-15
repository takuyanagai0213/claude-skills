# Incident Response Guide

A systematic workflow for investigating production alerts and incidents using GCP observability tools.

## Overview

This skill provides a step-by-step investigation workflow that:
- **Read-only operations** - No resource modifications, safe to use
- **MCP integration** - Uses gcloud and observability MCP tools
- **Systematic approach** - From alert confirmation to root cause analysis
- **Documentation templates** - Includes incident report template

## Features

### Investigation Workflow

1. **Environment Setup** - Verify GCP project context
2. **Alert Confirmation** - Check which alerts fired
3. **Investigation Phases**:
   - Phase 1: Logs review (first 5 minutes)
   - Phase 2: Traces, metrics, service status (5-15 minutes)
   - Phase 3: Pattern recognition (15-30 minutes)
4. **Root Cause Analysis** - Common patterns and debugging frameworks
5. **Documentation** - Create incident reports
6. **Escalation Criteria** - When and how to escalate

### MCP Tools Used

**Observability MCP:**
- `list_log_entries` - Search logs
- `list_traces` / `get_trace` - Analyze traces
- `list_time_series` - Check metrics
- `list_alerts` - View alerts

**gcloud MCP:**
- `run_gcloud_command` - Check service status, deployments, etc.

## Installation

1. Copy this directory to your project:

```bash
cp -r incident-response /path/to/your/project/.claude/skills/
```

2. Restart Claude Code or start a new conversation

## Usage

Activate the skill by requesting:
- "Investigate this alert"
- "Troubleshoot this incident"
- "アラート対応して" (Japanese)

The skill will guide you through the investigation process step-by-step.

## Customization

You can optionally add project-specific information to the SKILL.md file:
- Project IDs (production, staging)
- Common service names
- Escalation contacts

See the "Customization (Optional)" section in SKILL.md.

## Safety

This skill performs **read-only operations only**:
- ✅ Log queries
- ✅ Metric retrieval
- ✅ Trace analysis
- ✅ Service status checks
- ❌ No deployments
- ❌ No scaling
- ❌ No resource modifications

## Files

- `SKILL.md` - Main skill workflow
- `incident-report-template.md` - Template for documenting findings
- `README.md` - This file

## Requirements

- Claude Code with MCP support
- GCP project with:
  - Cloud Logging enabled
  - Cloud Monitoring enabled
  - Cloud Trace enabled (optional)
- MCP servers:
  - `gcloud` MCP
  - `observability` MCP

## License

MIT License
