# Kafka Consumer Action Strategy & Roadmap

> **Document Purpose**: Strategic analysis and improvement roadmap for the `yahoo.kafka_consumer` action architecture.  
> **Created**: February 2026  
> **Status**: Planning

---

## Table of Contents

1. [Current State Analysis](#current-state-analysis)
2. [The Core Problem](#the-core-problem)
3. [Strategic Recommendations](#strategic-recommendations)
   - [Level 1: Quick Wins](#level-1-quick-wins)
   - [Level 2: New Action Types](#level-2-new-action-types)
   - [Level 3: Parallel Processing](#level-3-parallel-processing-architecture)
   - [Level 4: Agentic AI Workflows](#level-4-agentic-ai-workflows)
4. [Implementation Roadmap](#implementation-roadmap)
5. [Decision Matrix](#decision-matrix)
6. [Implementation Sketches](#implementation-sketches)

---

## Current State Analysis

### Action Types Inventory

The `action_runner.py` currently supports **16 action types** organized into 4 categories:

| Category | Actions | Purpose |
|----------|---------|---------|
| **Remediation** | `screwdriver` | Run CI/CD jobs to fix issues |
| **Notification** | `alert_slack`, `incident_slack`, `comment_on_incident`, `tag_incident` | Communicate status |
| **Validation/Scanning** | `check_alert_volume`, `check_outage`, `scan_big_panda`, `check_service_now_changes`, `scan_service_now_ops`, `scan_rootly_alerts` | Pre-flight checks |
| **Escalation** | `rootly_escalation`, `ops_escalation`, `unassign_incident`, `snooze_incident`, `webhook` | Route to humans/systems |

### Current Screwdriver Action Capabilities

The existing `screwdriver` action type:
- Triggers a Screwdriver pipeline job
- Extracts parameters from alerts (`alert_parameters`, `alert_tag_parameters`, `extra_parameters`)
- Polls for job completion with configurable `polling_interval` and `polling_timeout`
- Tracks failed build URLs for debugging
- Blocks until the job completes or times out

---

## The Core Problem

### Sequential Alert Processing

Alerts are processed one at a time in `automation_consumer.py`:

```python
def _process_all_alerts(self, supported_alerts, matched_condition, incident_id):
    for alert in supported_alerts:
        ok = self.initiate_alert_actions(alert, matched_condition, incident_id)
        # waits for each to complete before continuing...
```

### Synchronous Screwdriver Waits

The Screwdriver action blocks on each job until completion:

```python
# Wait for screwdriver job event to complete - this blocks!
event_successful = sd_tools.sd_event_done_wait(event_id, action_def['polling_interval'])
```

### The Impact

**Example Scenario**:
- 4 alerts in an incident
- Each Screwdriver job takes ~10 minutes
- **Total time: 40 minutes** (4 × 10 min, sequential)

**Desired State**:
- Process alerts in parallel or batch
- **Target time: ~10 minutes** (parallel execution)

---

## Strategic Recommendations

### Level 1: Quick Wins

*Low effort, immediate impact*

#### 1. Batch Screwdriver Action (`screwdriver_batch`)

Collect all hosts/parameters from all alerts and pass them as a single Screwdriver job that handles parallelism internally.

**YAML Configuration**:
```yaml
- name: handle_oozie_batch
  type: screwdriver_batch
  job_name: oozie_batch_remediation
  auth_secret: sd-token
  pipeline_id: 1089736
  polling_interval: 30
  polling_timeout: 900
  # Parameters collected from ALL alerts, not just one
  batch_alert_tag_parameters:
    - host
    - oozie_pipeline
    - alert_tags_coordname
    - alert_tags_coordid
```

**Benefits**:
- 4 alerts → 1 job (~10 minutes vs 40 minutes)
- Reuses existing Screwdriver infrastructure
- Minimal changes to consumer

**Trade-offs**:
- Requires modifying Screwdriver job to accept batch parameters
- All-or-nothing failure mode (one job for all alerts)

---

#### 2. Fire-and-Forget Screwdriver (`screwdriver_async`)

For non-critical remediation where you don't need to wait for job completion.

**YAML Configuration**:
```yaml
- name: async_oozie_fix
  type: screwdriver_async
  job_name: oozie_remediation
  auth_secret: sd-token
  pipeline_id: 1089736
  # No polling - trigger and move on
  notify_on_failure: true
  failure_webhook_url: "$secret:slack-webhook"
```

**Benefits**:
- Near-instant completion from consumer perspective
- Screwdriver job runs in background
- Consumer can process more incidents

**Trade-offs**:
- No immediate feedback on success/failure
- Requires robust fallback/escalation paths
- May need separate job status monitoring

---

### Level 2: New Action Types

*Medium effort, high value*

#### 3. Direct SSH Execution (`ssh_script`)

Bypass Screwdriver entirely for simple script reruns.

**YAML Configuration**:
```yaml
- name: restart_oozie_coordinator
  type: ssh_script
  target_host_parameter: host  # Pull from alert tag
  script: |
    #!/bin/bash
    cd /opt/oozie && ./restart_coordinator.sh {{coordinator_id}}
  timeout: 60
  auth_method: athenz  # Use existing Athenz certs
  retry_count: 2
  retry_delay: 5
```

**Benefits**:
- No CI/CD overhead (seconds vs minutes)
- Direct execution on target host
- Can leverage existing Athenz certificates

**Trade-offs**:
- Requires SSH access from consumer → target hosts
- Less audit trail than Screwdriver
- Security review needed for script execution

**Recommended Library**: `asyncssh` for async SSH operations

---

#### 4. Ansible Playbook Execution (`ansible`)

Use Ansible for complex multi-step remediation with built-in parallelism.

**YAML Configuration**:
```yaml
- name: remediate_oozie_failures
  type: ansible
  playbook: playbooks/fix_oozie.yml
  inventory_from_alerts: host  # Dynamic inventory from alert hosts
  extra_vars:
    pipeline: "{{oozie_pipeline}}"
    coordinator_id: "{{alert_tags_coordid}}"
  forks: 10  # Parallel execution limit
  timeout: 300
  become: false  # Run as current user
```

**Benefits**:
- Native parallel execution across hosts (`forks`)
- Rich playbook ecosystem
- Idempotent operations
- Built-in retry logic

**Trade-offs**:
- Requires Ansible installation on consumer
- Playbook maintenance overhead
- SSH access required

---

#### 5. AWX/Ansible Tower API (`awx`)

Trigger Ansible jobs via AWX API if you have it deployed.

**YAML Configuration**:
```yaml
- name: awx_oozie_remediation
  type: awx
  awx_url: https://awx.internal.yahoo.com
  job_template_id: 42
  auth_secret: awx-api-token
  timeout: 600
  poll_interval: 10
  extra_vars_from_alerts:
    hosts: host
    coordinator_id: alert_tags_coordid
  limit_from_alerts: host  # Dynamic host limit
```

**Benefits**:
- Managed Ansible execution environment
- No direct SSH from consumer
- Centralized audit logging
- Credential management built-in

**Trade-offs**:
- Requires AWX/Tower infrastructure
- API latency overhead
- Dependency on AWX availability

---

#### 6. Kubernetes Job Execution (`k8s_job`)

Trigger Kubernetes Jobs for containerized remediation scripts.

**YAML Configuration**:
```yaml
- name: k8s_remediation
  type: k8s_job
  namespace: sre-automation
  job_template: remediation-job
  image: internal-registry/sre-tools:latest
  command: ["/scripts/fix_oozie.sh"]
  args_from_alerts:
    - host
    - oozie_pipeline
  timeout: 300
  ttl_seconds_after_finished: 3600
```

**Benefits**:
- No SSH required
- Isolated execution environment
- Easy to scale
- Works with existing Omega/K8s infrastructure

**Trade-offs**:
- Job startup overhead
- Need container image with tools
- Network connectivity to target systems

---

### Level 3: Parallel Processing Architecture

*Medium-high effort*

#### 7. Async Alert Processing

Refactor alert processing to use `asyncio` for concurrent execution.

**Current Flow** (Sequential):
```
Alert 1 → Wait → Alert 2 → Wait → Alert 3 → Wait → Alert 4 → Wait
[=========10min=========][=========10min=========][=========10min=========][=========10min=========]
Total: 40 minutes
```

**Proposed Flow** (Parallel with bounded concurrency):
```
Alert 1 → Wait ─┐
Alert 2 → Wait ─┼─→ All complete
Alert 3 → Wait ─┤
Alert 4 → Wait ─┘
[=================10min=================]
Total: ~10 minutes
```

**Key Configuration Options**:
```yaml
incident_conditions:
  - name: oozie_failures
    # ... other config ...
    parallel_alert_processing: true
    max_concurrent_alerts: 4
    fail_fast: false  # Continue other alerts if one fails
```

**Implementation Notes**:
- Use `asyncio.Semaphore` for bounded parallelism
- Run blocking Screwdriver calls in `ThreadPoolExecutor`
- Handle mixed success/failure scenarios
- Update action history atomically

---

### Level 4: Agentic AI Workflows

*High effort, transformational*

#### 8. LLM-Powered Runbook Selection (`llm_runbook`)

Use an LLM to analyze alert context and select the best remediation approach.

**YAML Configuration**:
```yaml
- name: intelligent_remediation
  type: llm_runbook
  model: claude-3-5-sonnet  # or internal Yahoo model
  model_endpoint: https://ai.internal.yahoo.com/v1
  auth_secret: llm-api-key
  
  runbook_library:
    - name: restart_service
      description: "Restart a failed service"
      action: ssh_script
      script: "systemctl restart {{service_name}}"
    
    - name: clear_cache
      description: "Clear application cache"
      action: ssh_script
      script: "/opt/tools/clear_cache.sh {{host}}"
    
    - name: scale_up
      description: "Increase resource allocation"
      action: webhook
      url: "https://autoscaler.internal/scale"
    
    - name: rollback_deploy
      description: "Rollback recent deployment"
      action: screwdriver
      job_name: rollback

    - name: escalate_human
      description: "Page on-call engineer"
      action: rootly_escalation
  
  context_from_incident:
    - description
    - host
    - severity
    - property
    - recent_changes
  
  confidence_threshold: 0.85  # Only auto-execute if confident
  require_human_approval: false
  fallback_action: escalate_human
```

**How It Works**:
1. Build context from incident/alert data
2. Call LLM with available runbooks and context
3. LLM returns recommended runbook + confidence score
4. If confidence ≥ threshold, execute the runbook
5. If confidence < threshold, escalate to humans

**Benefits**:
- Dynamic decision making
- Handles novel scenarios
- Learns from patterns
- Reduces runbook maintenance

**Trade-offs**:
- LLM latency (1-5 seconds)
- Requires prompt engineering
- Trust/validation concerns
- Cost per API call

---

#### 9. Autonomous Remediation Agent (`agent`)

A full agentic workflow with tool-calling capabilities - essentially an "AI SRE".

**YAML Configuration**:
```yaml
- name: sre_agent
  type: agent
  model: claude-3-5-sonnet
  model_endpoint: https://ai.internal.yahoo.com/v1
  auth_secret: llm-api-key
  
  system_prompt: |
    You are an SRE agent responsible for investigating and resolving 
    infrastructure incidents. Analyze alerts, gather context using the 
    available tools, and execute remediation actions. Always verify 
    your fixes before closing incidents.
  
  tools:
    - name: ssh_execute
      description: "Run a shell command on a target host"
      parameters:
        host: string
        command: string
    
    - name: check_metrics
      description: "Query Prometheus for host metrics"
      parameters:
        query: string
        duration: string
    
    - name: query_logs
      description: "Search Splunk for related logs"
      parameters:
        query: string
        time_range: string
    
    - name: restart_service
      description: "Restart a service on target host"
      parameters:
        host: string
        service_name: string
    
    - name: check_dependencies
      description: "Check status of dependent services"
      parameters:
        service_name: string
    
    - name: escalate
      description: "Page the on-call engineer with context"
      parameters:
        urgency: string
        summary: string
  
  max_iterations: 10
  max_tool_calls_per_iteration: 3
  timeout: 300
  require_human_approval: false
  
  # Safety guardrails
  prohibited_actions:
    - "rm -rf"
    - "shutdown"
    - "reboot"
  allowed_hosts_pattern: "*.internal.yahoo.com"
```

**Agent Flow**:
```
1. Receive Alert
       ↓
2. Analyze Alert Context
       ↓
3. Query Logs/Metrics (tools)
       ↓
4. Identify Root Cause
       ↓
5. Select Remediation Action
       ↓
6. Execute Fix (tools)
       ↓
7. Verify Resolution (tools)
       ↓
8. Close or Escalate
```

**Benefits**:
- Handles complex, multi-step investigations
- Can adapt to unexpected situations
- Gathers context dynamically
- Provides reasoning for actions

**Trade-offs**:
- Higher complexity
- More LLM API calls = higher cost
- Needs extensive safety guardrails
- Requires monitoring of agent decisions

---

## Implementation Roadmap

| Phase | Timeline | Deliverables | Impact |
|-------|----------|--------------|--------|
| **Phase 1** | 2 weeks | `screwdriver_batch` + `screwdriver_async` action types | 4x speed improvement for batch scenarios |
| **Phase 2** | 4 weeks | Parallel alert processing with asyncio | 4x speed improvement for all multi-alert incidents |
| **Phase 3** | 6 weeks | `ssh_script` or `ansible` action type | 10x speed improvement for simple remediations |
| **Phase 4** | 8-12 weeks | `llm_runbook` for intelligent action selection | Dynamic remediation, reduced manual runbook updates |
| **Phase 5** | Q3 2026 | Full `agent` action type with tool-calling | Autonomous incident resolution |

### Phase 1 Details (Weeks 1-2)

**Week 1**:
- [ ] Implement `run_sd_batch_automation()` method
- [ ] Add `screwdriver_batch` to action type registry
- [ ] Update config validator for new action type
- [ ] Create test Screwdriver job that accepts batch parameters

**Week 2**:
- [ ] Implement `run_sd_async_automation()` method
- [ ] Add callback/webhook for async job completion notification
- [ ] Write unit tests for new action types
- [ ] Deploy to staging and test with real incidents

### Phase 2 Details (Weeks 3-6)

**Weeks 3-4**:
- [ ] Refactor `_process_all_alerts` to use asyncio
- [ ] Add `ThreadPoolExecutor` for blocking calls
- [ ] Implement bounded concurrency with semaphores
- [ ] Add `parallel_alert_processing` config option

**Weeks 5-6**:
- [ ] Handle partial failure scenarios
- [ ] Update action history for parallel execution
- [ ] Performance testing and tuning
- [ ] Production rollout with feature flag

---

## Decision Matrix

Use this table to choose the right action type for your remediation needs:

| Scenario | Recommended Action | Why |
|----------|-------------------|-----|
| Fast, simple script rerun | `ssh_script` | No CI/CD overhead, direct execution |
| Complex multi-step remediation | `ansible` | Playbooks with built-in parallelism |
| Multiple alerts, same fix | `screwdriver_batch` | Reuse existing SD jobs, batch execution |
| Don't need immediate result | `screwdriver_async` | Fire-and-forget, async notification |
| Unknown/variable fix needed | `llm_runbook` | AI selects best approach dynamically |
| Complex investigation + fix | `agent` | AI investigates, decides, and executes |
| Containerized tools needed | `k8s_job` | Isolated execution, no SSH required |
| Existing Ansible infrastructure | `awx` | Managed execution, central audit log |

---

## Implementation Sketches

### Batch Screwdriver Implementation

```python
def run_sd_batch_automation(self, alerts: list[dict], action_def: dict) -> bool:
    """
    Collect params from all alerts and trigger ONE Screwdriver job.
    The job is expected to handle parallelism internally.
    """
    logging.info("Running batch Screwdriver automation for %d alerts", len(alerts))
    
    # Collect parameters from all alerts
    batch_params = []
    for alert in alerts:
        params = self.extract_alert_params(alert, action_def)
        batch_params.append(params)
    
    # Build single SD job params with batch data
    sd_params = {
        action_def['job_name']: {
            'batch_mode': 'true',
            'hosts': json.dumps([p.get('host') for p in batch_params]),
            'alert_params': json.dumps(batch_params),
            'alert_count': str(len(batch_params))
        }
    }
    
    # Get Screwdriver token
    sd_token = self.secret_manager.get_secret_value(action_def['auth_secret'])
    sd_tools = sdv4_tools(action_def['pipeline_id'], sd_token, action_def['polling_timeout'])
    
    # Trigger single job
    event_id = sd_tools.trigger_sd_job(action_def['job_name'], sd_params)
    logging.info("Triggered batch SD job, event: %s", event_id)
    
    # Wait for completion
    try:
        return sd_tools.sd_event_done_wait(event_id, action_def['polling_interval'])
    except Exception as e:
        logging.error("Batch SD job failed: %s", e)
        return False
```

### Async SSH Implementation

```python
import asyncssh
import asyncio

async def run_ssh_script_async(
    self, 
    host: str, 
    script: str, 
    timeout: int = 60
) -> tuple[bool, str]:
    """
    Execute script on target host via SSH asynchronously.
    Returns (success, output).
    """
    try:
        async with asyncssh.connect(
            host,
            username='sre-automation',
            client_keys=[self.key],
            known_hosts=None,
            connect_timeout=10
        ) as conn:
            result = await asyncio.wait_for(
                conn.run(script, check=False),
                timeout=timeout
            )
            success = result.returncode == 0
            output = result.stdout if success else result.stderr
            return success, output
    except asyncio.TimeoutError:
        return False, f"SSH command timed out after {timeout}s"
    except Exception as e:
        return False, f"SSH error: {str(e)}"

def run_ssh_script(self, alert: dict, action_def: dict) -> bool:
    """Synchronous wrapper for SSH script execution."""
    host = self._get_host_from_alert(alert, action_def)
    script = self.replace_variables(
        action_def['script'],
        self.extract_alert_params(alert, action_def)
    )
    timeout = action_def.get('timeout', 60)
    
    success, output = asyncio.run(
        self.run_ssh_script_async(host, script, timeout)
    )

    if success:
        logging.info("SSH script succeeded on %s: %s", host, output[:200])
    else:
        logging.error("SSH script failed on %s: %s", host, output)
    
    return success
```

### Parallel Alert Processing

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AutomationConsumer:
    def __init__(self, yaml_config):
        # ... existing init ...
        self.executor = ThreadPoolExecutor(max_workers=10)
    
    async def _process_all_alerts_parallel(
        self,
        supported_alerts: list[dict],
        matched_condition: dict,
        incident_id: str
    ) -> bool:
        """Process alerts concurrently with bounded parallelism."""
        max_concurrent = matched_condition.get('max_concurrent_alerts', 4)
        semaphore = asyncio.Semaphore(max_concurrent)
        
        async def process_one(alert):
            async with semaphore:
                loop = asyncio.get_event_loop()
                return await loop.run_in_executor(
                    self.executor,
                    self.initiate_alert_actions,
                    alert, matched_condition, incident_id
                )

        tasks = [process_one(a) for a in supported_alerts]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Log results
        for alert, result in zip(supported_alerts, results):
            if isinstance(result, Exception):
                logging.error("Alert %s failed with exception: %s", alert['id'], result)
            elif not result:
                logging.warning("Alert %s returned failure", alert['id'])
        
        # Return True only if all succeeded
        fail_fast = not matched_condition.get('continue_on_failure', False)
        if fail_fast:
            return all(r is True for r in results)
        else:
            return any(r is True for r in results)
    
    def _process_all_alerts(
        self,
        supported_alerts: list[dict],
        matched_condition: dict,
        incident_id: str
    ) -> bool:
        """Dispatch to parallel or sequential processing based on config."""
        if matched_condition.get('parallel_alert_processing', False):
            return asyncio.run(
                self._process_all_alerts_parallel(
                    supported_alerts, matched_condition, incident_id
                )
            )
        else:
            # Original sequential processing
            return self._process_all_alerts_sequential(
                supported_alerts, matched_condition, incident_id
            )
```

### LLM Runbook Selection

```python
import json
from typing import Optional

def run_llm_runbook(
    self, 
    incident_id: str, 
    alerts: list[dict],
    incident_tags: list[dict],
    action_def: dict
) -> bool:
    """Use LLM to select and execute the best remediation runbook."""
    
    # 1. Build context from incident and alerts
    context = {
        'incident_id': incident_id,
        'alert_count': len(alerts),
        'alerts': [
            {
                'id': a.get('id'),
                'host': a.get('host'),
                'description': next(
                    (t['value'] for t in a.get('tags', []) if t['name'] == 'description'),
                    'N/A'
                ),
                'severity': next(
                    (t['value'] for t in a.get('tags', []) if t['name'] == 'severity'),
                    'N/A'
                ),
            }
            for a in alerts
        ],
        'incident_tags': {t['id']: t['value'] for t in incident_tags}
    }
    
    # 2. Build prompt with available runbooks
    runbooks = action_def.get('runbook_library', [])
    runbook_descriptions = "\n".join(
        f"- {r['name']}: {r['description']}"
        for r in runbooks
    )
    
    prompt = f"""You are an SRE automation system. Analyze this incident and recommend the best remediation action.

Available runbooks:
{runbook_descriptions}

Incident context:
{json.dumps(context, indent=2)}

Respond with JSON only:
{{
    "recommended_runbook": "runbook_name",
    "confidence": 0.0 to 1.0,
    "reasoning": "Brief explanation of why this runbook is appropriate"
}}
"""
    
    # 3. Call LLM
    try:
        response = self._call_llm(action_def, prompt)
        decision = json.loads(response)
    except Exception as e:
        logging.error("LLM call failed: %s", e)
        return self._execute_fallback(action_def, incident_id)
    
    # 4. Check confidence threshold
    confidence = decision.get('confidence', 0)
    threshold = action_def.get('confidence_threshold', 0.85)
    
    logging.info(
        "LLM recommended '%s' with confidence %.2f (threshold: %.2f). Reasoning: %s",
        decision.get('recommended_runbook'),
        confidence,
        threshold,
        decision.get('reasoning')
    )
    
    if confidence < threshold:
        logging.warning("Confidence below threshold, escalating to fallback")
        return self._execute_fallback(action_def, incident_id)
    
    # 5. Execute the recommended runbook
    runbook_name = decision.get('recommended_runbook')
    runbook = next((r for r in runbooks if r['name'] == runbook_name), None)

    if not runbook:
        logging.error("Recommended runbook '%s' not found", runbook_name)
        return False
    
    return self._execute_runbook(runbook, alerts, action_def)

def _call_llm(self, action_def: dict, prompt: str) -> str:
    """Call LLM API and return response text."""
    import requests
    
    endpoint = action_def.get('model_endpoint')
    api_key = self.secret_manager.get_secret_value(action_def['auth_secret'])
    model = action_def.get('model', 'claude-3-5-sonnet')
    
    response = requests.post(
        f"{endpoint}/chat/completions",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        },
        json={
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 500
        },
        timeout=30
    )
    response.raise_for_status()
    return response.json()['choices'][0]['message']['content']
```

---

## Appendix: Config Schema Updates

### New Action Type Schemas

```yaml
# screwdriver_batch schema
- name: string (required)
  type: screwdriver_batch (required)
  job_name: string (required)
  auth_secret: string (required)
  pipeline_id: integer (required)
  polling_interval: integer (default: 30)
  polling_timeout: integer (default: 600)
  batch_alert_parameters: list[string] (optional)
  batch_alert_tag_parameters: list[string] (optional)

# ssh_script schema
- name: string (required)
  type: ssh_script (required)
  target_host_parameter: string (required)
  script: string (required)
  timeout: integer (default: 60)
  auth_method: enum[athenz, key] (default: athenz)
  retry_count: integer (default: 0)
  retry_delay: integer (default: 5)

# llm_runbook schema
- name: string (required)
  type: llm_runbook (required)
  model: string (required)
  model_endpoint: string (required)
  auth_secret: string (required)
  runbook_library: list[object] (required)
  context_from_incident: list[string] (optional)
  confidence_threshold: float (default: 0.85)
  require_human_approval: boolean (default: false)
  fallback_action: string (optional)
```

---

## References

- [CLAUDE.md](../CLAUDE.md) - Project overview and architecture
- [action_runner.py](../src/yahoo/kafka_consumer/action_runner.py) - Current action implementation
- [automation_consumer.py](../src/yahoo/kafka_consumer/automation_consumer.py) - Consumer processing logic
- [example-config.yaml](../src/yahoo/kafka_consumer/example-config.yaml) - Configuration examples
