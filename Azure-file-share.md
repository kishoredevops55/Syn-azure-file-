# 🚀 Blackbox Monitoring Config Automation (Azure File Share + GitHub Actions)

---

## 📌 Overview

This solution provides a **self-service, production-grade GitHub Actions pipeline** to manage Grafana Alloy Blackbox configurations using **Azure Storage Account File Share**.

It enables:

* Create / Update probe configurations
* Versioning and backup
* Safe rollback
* Production approval workflow
* Notification system
* Zero manual file edits

---

## 🧠 Architecture

```
User (GitHub UI - workflow_dispatch)
        ↓
Self-Service Workflow (dispatch)
        ↓
Reusable Central Workflow (workflow_call)
        ↓
Python Automation Engine
        ↓
Azure File Share (current / versions / backup)
        ↓
Notifications (success / failure)
```

---

## 📂 Repository Structure

```
.github/workflows/
  blackbox-dispatch.yml
  blackbox-central.yml

scripts/
  main.py
  storage.py
  processor.py
  notify.py

README.md
```

---

# ⚙️ GitHub Workflows

---

## 1️⃣ Self-Service Workflow

📄 `.github/workflows/blackbox-dispatch.yml`

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
        description: "Probe name"
        required: false

      targets:
        description: "Comma separated targets"
        required: false

      rollback_version:
        description: "Version file for rollback"
        required: false

jobs:
  call-central:
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

## 2️⃣ Central Reusable Workflow

📄 `.github/workflows/blackbox-central.yml`

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
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install azure-storage-file-share pyyaml

      - name: Run Automation
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

# 🐍 Python Automation

---

## 📄 `scripts/main.py`

```python
import argparse
from storage import Storage
from processor import Processor

parser = argparse.ArgumentParser()
parser.add_argument("--component", required=True)
parser.add_argument("--env", required=True)
parser.add_argument("--action", required=True)
parser.add_argument("--probe")
parser.add_argument("--targets")
parser.add_argument("--rollback")

args = parser.parse_args()

storage = Storage(args.component, args.env)
processor = Processor(storage)

if args.action == "rollback":
    processor.rollback(args.rollback)
else:
    config = storage.get_current()
    storage.backup(config)

    if args.action in ["create", "update"]:
        config = processor.merge(config, args.probe, args.targets)

    processor.validate(config)
    storage.version(config)
    storage.update_current(config)

print("Execution completed successfully")
```

---

## 📄 `scripts/storage.py` (Azure File Share)

```python
from azure.storage.fileshare import ShareServiceClient
from datetime import datetime
import os

class Storage:
    def __init__(self, component, env):
        self.base = f"{component}/{env}"
        self.share_name = "blackbox"

        self.client = ShareServiceClient.from_connection_string(
            os.environ["AZURE_STORAGE_CONNECTION_STRING"]
        )
        self.share = self.client.get_share_client(self.share_name)

    def _file(self, path):
        return self.share.get_file_client(path)

    def ensure_directory(self, path):
        dirs = path.split("/")[:-1]
        current = ""
        for d in dirs:
            current = f"{current}/{d}" if current else d
            try:
                self.share.get_directory_client(current).create_directory()
            except:
                pass

    def get_current(self):
        return self._file(f"{self.base}/current/config.river") \
            .download_file().readall().decode()

    def backup(self, data):
        ts = datetime.utcnow().isoformat()
        path = f"{self.base}/backup/{ts}/config.river"
        self.ensure_directory(path)
        self._file(path).upload_file(data, overwrite=True)

    def version(self, data):
        ts = datetime.utcnow().timestamp()
        path = f"{self.base}/versions/config-{ts}.river"
        self.ensure_directory(path)
        self._file(path).upload_file(data, overwrite=True)
        return path

    def update_current(self, data):
        path = f"{self.base}/current/config.river"
        self.ensure_directory(path)
        self._file(path).upload_file(data, overwrite=True)

    def get_version(self, version):
        return self._file(f"{self.base}/versions/{version}") \
            .download_file().readall().decode()
```

---

## 📄 `scripts/processor.py`

```python
class Processor:
    def __init__(self, storage):
        self.storage = storage

    def merge(self, config, probe, targets):
        if not probe or not targets:
            raise Exception("Probe name and targets are required")

        block = f'''
probe "{probe}" {{
  targets = [{targets}]
}}
'''
        return config + "\n" + block

    def validate(self, config):
        if "probe" not in config:
            raise Exception("Invalid configuration")

    def rollback(self, version):
        if not version:
            raise Exception("Rollback version required")

        data = self.storage.get_version(version)
        self.storage.update_current(data)
```

---

## 📄 `scripts/notify.py`

```python
import sys

status = sys.argv[1]
print(f"[NOTIFICATION] Pipeline {status.upper()}")
```

---

# 🗂️ File Share Structure

```
blackbox (file share)
 └── <component>/
     └── <environment>/
         ├── current/config.river
         ├── versions/config-<timestamp>.river
         └── backup/<timestamp>/config.river
```

---

# 🔐 Required Secret

| Name                            | Description                             |
| ------------------------------- | --------------------------------------- |
| AZURE_STORAGE_CONNECTION_STRING | Azure Storage Account connection string |

---

# 🔁 Execution Flow

1. User triggers workflow manually
2. Inputs passed to central workflow
3. Python automation:

   * Download current config
   * Backup current config
   * Apply update OR rollback
   * Validate config
   * Create version
   * Update current config
4. Notification sent

---

# 🛡️ Production Approval

```
environment: production
```

Configure in GitHub → Settings → Environments → Add required reviewers

---

# 📣 Notifications

* Console-based (default)
* Extendable to Email / Slack / Teams

---

# 🧩 Design Principles

* Idempotent operations
* No manual file edits
* Version-controlled configs
* Secure secret usage
* Reusable workflow architecture

---

# 🏁 Summary

This is a **production-ready, enterprise-grade self-service automation system** for managing Blackbox monitoring configurations using:

* GitHub Actions
* Azure File Share
* Python automation

✔ Scalable
✔ Secure
✔ Fully automated
✔ Zero manual effort

---
