
---

# ✅ FINAL – SELF-SERVICE WORKFLOW (CLEAN)

👉 `.github/workflows/blackbox-dispatch.yml`

```yaml
name: Blackbox Self-Service

on:
  workflow_dispatch:
    inputs:
      component:
        description: "Probe type"
        required: true
        type: choice
        options:
          - http
          - dns
          - tcp
          - ping
          - traceroute

      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - dev
          - perf
          - prod

      action:
        description: "Action to perform"
        required: true
        type: choice
        options:
          - create
          - update
          - rollback

      probe_name:
        description: "Probe name (required for create/update)"
        required: false

      targets:
        description: "Targets (comma separated)"
        required: false

      rollback_version:
        description: "Version file name (for rollback)"
        required: false

jobs:
  call-central-workflow:
    uses: your-org/central-repo/.github/workflows/blackbox-central.yml@main
    with:
      component: ${{ inputs.component }}
      environment: ${{ inputs.environment }}
      action: ${{ inputs.action }}
      probe_name: ${{ inputs.probe_name }}
      targets: ${{ inputs.targets }}
      rollback_version: ${{ inputs.rollback_version }}
    secrets: inherit
```

---

# ✅ FINAL – CENTRAL WORKFLOW (REUSABLE)

👉 `.github/workflows/blackbox-central.yml`

```yaml
name: Blackbox Central Pipeline

on:
  workflow_call:
    inputs:
      component:
        type: string
        required: true
      environment:
        type: string
        required: true
      action:
        type: string
        required: true
      probe_name:
        type: string
        required: false
      targets:
        type: string
        required: false
      rollback_version:
        type: string
        required: false

jobs:
  process-config:
    runs-on: self-hosted

    environment: ${{ inputs.environment == 'prod' && 'production' || 'nonprod' }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install azure-storage-blob pyyaml

      - name: Execute automation
        env:
          AZURE_STORAGE_CONNECTION_STRING: ${{ secrets.AZURE_STORAGE_CONNECTION_STRING }}
        run: |
          python scripts/main.py \
            --component "${{ inputs.component }}" \
            --env "${{ inputs.environment }}" \
            --action "${{ inputs.action }}" \
            --probe "${{ inputs.probe_name }}" \
            --targets "${{ inputs.targets }}" \
            --rollback "${{ inputs.rollback_version }}"

      - name: Notify Success
        if: success()
        run: python scripts/notify.py success

      - name: Notify Failure
        if: failure()
        run: python scripts/notify.py failure
```

---

# ✅ FINAL – PRODUCTION-READY README.md

👉 **Fully clean, no formatting issues, copy-paste safe**

# Blackbox Monitoring Config Automation

## Overview

This project provides a **self-service GitHub Actions pipeline** to manage Grafana Alloy Blackbox configuration.

It enables:

* Create / update probes
* Version control of configs
* Backup and rollback
* Production approval workflow
* Notification on execution

---

## Architecture

User triggers workflow manually → Dispatch Workflow → Central Workflow → Python Automation → Azure Blob Storage → Notification

---

## Workflow Design

### 1. Self-Service Workflow

* Triggered using `workflow_dispatch`
* Accepts user inputs
* Calls reusable central workflow

### 2. Central Workflow

* Executes core logic
* Runs Python automation
* Handles validation, backup, versioning
* Sends notifications

---

## Inputs

| Input Name       | Description                          |
| ---------------- | ------------------------------------ |
| component        | http / dns / tcp / ping / traceroute |
| environment      | dev / perf / prod                    |
| action           | create / update / rollback           |
| probe_name       | Required for create/update           |
| targets          | Comma-separated endpoints            |
| rollback_version | Required for rollback                |

---

## Storage Structure

```
<component>/<environment>/
  current/config.river
  versions/config-<timestamp>.river
  backup/<timestamp>/config.river
```

---

## Execution Flow

1. User triggers workflow manually
2. Dispatch workflow passes inputs to central workflow
3. Python script performs:

   * Download current config
   * Backup current config
   * Apply changes OR rollback
   * Validate configuration
   * Create version
   * Update current config
4. Notification sent on completion

---

## Production Approval

* Production environment is protected using GitHub Environments
* Workflow pauses for approval before execution

```
environment: production
```

---

## Secrets Required

| Secret Name                     | Description                  |
| ------------------------------- | ---------------------------- |
| AZURE_STORAGE_CONNECTION_STRING | Azure Blob connection string |

---

## Notifications

* Success and failure notifications supported
* Extendable to email / Slack / Teams

---

## Repository Structure

```
.github/workflows/
  blackbox-dispatch.yml
  blackbox-central.yml

scripts/
  main.py
  storage.py
  processor.py
  validator.py
  notify.py
```

---

## Key Design Principles

* No manual config edits
* Idempotent operations
* Full version history
* Secure secret handling
* Environment-based control
* Reusable workflow design

---

## Future Enhancements

* Slack / Teams integration
* Schema validation
* UI portal for self-service
* GitOps-based config management
* Parallel execution per probe type

---

## Summary

This solution provides a **scalable, secure, and production-ready automation system** for managing blackbox monitoring configurations with zero manual effort.

---

👉 **V3 = Multi-job parallel (http/dns/tcp split) + schema validation + PR-based GitOps**

Just say: **“upgrade to V3”** 🚀
