# Private DNS Automation — Implementation Plan

## Background

We currently manage 92+ Private DNS Zones and ~200 VNets using a scheduled PowerShell
script (`run.ps1`) that runs hourly. The script has several critical issues:

- Removing a zone from `zones.json` silently destroys it on the next run
- No structured intake process — engineers edit files directly
- Sequential link creation causes runs to exceed 60 minutes
- Console-created zones are silently deleted
- No audit trail for who requested a zone or why
- Infoblox sync is a separate manual step (to be addressed in a future phase)

---

## Goal

Replace the PowerShell script with a GitOps pipeline where:

- Git is the single source of truth
- Zone requests come through a structured form, not file edits
- Accidental deletions are blocked, intentional deletions require explicit approval
- The pipeline scales to handle 200+ VNet links without timing out
- Console-created or manually deleted zones are detected and remediated automatically via Event Grid

---

## Architecture

### Repository Layout

```
azure-dns-governance/
├── registry.json                  # Zone → Resource Group mapping (source of truth)
├── hubs.json                      # Hub VNet inventory for spoke discovery
├── Dockerfile                     # Consistent execution environment (TF + Python + AZ CLI)
│
├── 01-core-zones/                 # Terraform workspace — manages DNS zones only
│   ├── main.tf
│   ├── variables.tf
│   ├── backend.tf
│   └── zones.auto.tfvars
│
├── 02-links-batch-01/             # Terraform workspace — VNet links, batch 1 of 5
├── 03-links-batch-02/             # Batch 2
├── 04-links-batch-03/             # Batch 3
├── 05-links-batch-04/             # Batch 4
├── 06-links-batch-05/             # Batch 5
│
└── scripts/
    ├── discover_spokes.py         # Crawls hub peerings to build full VNet inventory
    ├── process_dns_request.py     # Parses GitHub Issue, validates, edits registry.json
    ├── check_zone_drift.py        # Pre-apply safety gate
    └── detect_unmanaged.py        # Nightly scan for console-created resources
```

---

## Why Batched State Files

A single Terraform state file managing 200 VNet links hits two problems:

1. **Azure API throttling** — too many concurrent requests return 429 errors
2. **GitHub runner timeout** — a single job can exceed the 6-hour limit

The solution is to split VNet links across multiple workspaces, each with its own
state file. Each batch runs as a separate parallel GitHub Actions job:

```
PR merged
    │
    ▼
Job 1: 01-core-zones    → creates/deletes DNS zones
    │
    ▼ (on success)
Jobs 2-6 (parallel):
    02-links-batch-01   → links 1-40
    03-links-batch-02   → links 41-80
    04-links-batch-03   → links 81-120
    05-links-batch-04   → links 121-160
    06-links-batch-05   → links 161-200
```

Total time: zone creation (~1 min) + link batches in parallel (~4 min) = ~5 minutes
vs the current sequential approach of 60+ minutes.

---

## State File Layout

Each workspace gets its own blob in Azure Storage:

```
Storage Account: tfstatedns
└── Container: terraform-state
    ├── core-zones.tfstate
    ├── links-batch-01.tfstate
    ├── links-batch-02.tfstate
    ├── links-batch-03.tfstate
    ├── links-batch-04.tfstate
    └── links-batch-05.tfstate
```

Isolated state files also limit blast radius — a failed batch only affects its own
40 links, not the entire zone inventory.

---

## Source of Truth — registry.json

Instead of a flat zone list, `registry.json` maps each zone to its resource group
and any metadata needed for governance:

```json
{
  "zones": [
    {
      "fqdn": "app.internal.contoso.com",
      "resource_group": "RG-PrivateDNS",
      "environment": "production",
      "owner": "platform-team",
      "ticket": "ADO-1234",
      "created": "2026-05-01"
    }
  ]
}
```

This replaces both `zones.json` and `zones.auto.tfvars` as the canonical record.
The Python script reads `registry.json` and generates the tfvars files at pipeline
runtime — engineers never edit tfvars directly.

---

## Hub and Spoke Discovery

Rather than maintaining a static VNet list, a Python crawler builds the full
VNet inventory dynamically at pipeline runtime:

```
hubs.json (7 hub VNets)
    │
    ▼
discover_spokes.py
    │  Calls Get-AzVirtualNetworkPeering for each hub
    │  Builds complete list of peered spokes
    ▼
Full VNet inventory (200 VNets)
    │
    ▼
Split into 5 batches of 40
    │
    ▼
Each batch workspace gets its own tfvars file
```

When a new VNet is peered to any hub, it is automatically discovered and linked
on the next pipeline run — no manual inventory update needed.

---

## Zone Request Flow

### Creating a Zone

```
1. Engineer opens GitHub Issue using the DNS Zone Request form
   Fields: Zone FQDN, ADO/Incident number, owner, environment, justification

2. GitHub Actions workflow fires (triggered by dns-zone-request label)
   - Validates FQDN format and naming convention
   - Checks Azure: does zone already exist?
   - If exists → comments on issue, closes it
   - If not → continues

3. process_dns_request.py updates registry.json
   - Adds zone entry with all metadata
   - Branch name = ticket number (ADO-1234)

4. PR auto-created and linked back to issue
   - Title: [DNS Create] app.internal.com (ADO-1234)
   - DNS team reviews and approves

5. On merge:
   - 01-core-zones applies → zone created in Azure
   - Batches 01-05 apply in parallel → 200 VNet links created
   - Issue auto-closes with confirmation comment
```

### Deleting a Zone

```
1. Engineer opens GitHub Issue (action = Delete)

2. process_dns_request.py moves zone to pending_deletion in registry.json

3. PR requires 2 approvers (branch protection rule)

4. check_zone_drift.py verifies deletion is intentional before apply

5. On merge: zone and all its VNet links destroyed
```

---

## Drift Prevention

### Accidental Deletion Guard — Pipeline Level

`check_zone_drift.py` runs before every `terraform apply`. It compares
`registry.json` against Terraform state. If a zone was removed from the registry
without being explicitly marked for deletion, the pipeline exits with an error
and apply never runs.

### Event Grid — Two Subscriptions, Two Guards

All zone lifecycle events are monitored via Event Grid at the subscription scope.
Two separate subscriptions handle creation and deletion independently, both
routing to Logic Apps that cross-reference `registry.json`.

#### Guard 1 — Console Creation (Auto-Nuke)

```
Zone created in Azure (portal, CLI, ARM template, or any tool outside pipeline)
    │
    ▼
Event Grid fires: Microsoft.Resources.ResourceWriteSuccess
Filter: resourceType == Microsoft.Network/privateDnsZones
    │
    ▼
Logic App: check-zone-creation triggered
    │
    ├── Zone exists in registry.json AND has tag managed-by: terraform
    │     → Created by pipeline — no action
    │
    └── Zone NOT in registry.json OR missing managed-by tag
          → Unmanaged zone detected
          │
          ├── Post to Teams: "⚠ Unmanaged zone created: <fqdn> — deleting"
          ├── Delete the zone via Azure REST API
          └── Log action to Storage Account audit log
```

Any zone not provisioned through the pipeline is removed within minutes.
This eliminates shadow IT and prevents configuration drift at the source.

#### Guard 2 — Accidental Deletion (Self-Healing)

```
Zone deleted in Azure (portal, CLI, accidental Terraform destroy, or any tool)
    │
    ▼
Event Grid fires: Microsoft.Resources.ResourceDeleteSuccess
Filter: resourceType == Microsoft.Network/privateDnsZones
    │
    ▼
Logic App: check-zone-deletion triggered
    │
    ├── Zone NOT in registry.json
    │     → Intentional deletion (went through pipeline) — no action
    │
    └── Zone IS in registry.json AND not in pending_deletion
          → Managed zone deleted outside pipeline — discrepancy detected
          │
          ├── Post to Teams: "🔴 Managed zone deleted: <fqdn> — restoring"
          │
          └── Trigger GitHub Actions pipeline via repository_dispatch event
                    │
                    ▼
                Pipeline runs terraform apply on 01-core-zones
                    │
                    ▼
                Terraform detects zone missing from Azure but present in state
                Recreates zone
                    │
                    ▼
                Batch workspaces 01-05 run in parallel
                Recreates all VNet links
                    │
                    ▼
                Teams notification: "✅ Zone <fqdn> restored successfully"
                Log restoration to Storage Account audit log
```

### Event Grid Setup Summary

| Subscription Name | Event Type | Resource Filter | Logic App | Action |
|---|---|---|---|---|
| `dns-zone-created` | `ResourceWriteSuccess` | `privateDnsZones` | `check-zone-creation` | Auto-nuke if not in registry |
| `dns-zone-deleted` | `ResourceDeleteSuccess` | `privateDnsZones` | `check-zone-deletion` | Trigger restore pipeline if in registry |

Both Logic Apps:
- Post to the same DNS Operations Teams channel
- Write every action to a Storage Account audit log (zone name, event type, action taken, timestamp)
- Are scoped to the subscription level so they catch events across all resource groups

---

## Execution Environment — Docker

All CI/CD jobs use a custom Docker image stored in Azure Container Registry (ACR).
This ensures every runner uses identical tool versions and dependencies are pulled
over the Azure backbone, not the public internet.

```dockerfile
FROM ubuntu:22.04

# Terraform
RUN apt-get install -y terraform=1.7.0

# Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt
```

```yaml
# In each GitHub Actions job
container:
  image: <acr-name>.azurecr.io/dns-automation:latest
  credentials:
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}
```

---

## Teams Integration

Two notification points:

1. **PR opened** — Teams card sent to DNS channel with zone details and a link
   to review the PR in GitHub. One-click approve.

2. **Apply complete** — Teams card confirms zone is live in Azure,
   or surfaces an error with a link to the pipeline logs.

3. **Event Grid alerts** — Teams card when an unmanaged zone is detected and
   deleted, or when a managed zone is restored after accidental deletion.

Future state: Logic App Adaptive Card with Approve/Reject buttons directly in
Teams, eliminating the need to open GitHub for approvals.

---

## Rollout Plan

### Phase 1 — Foundation
- Replace `run.ps1` with Terraform (`01-core-zones`)
- Single state file, single batch (no parallelism yet)
- Import all existing zones into Terraform state
- Validate: `terraform plan` shows zero changes

### Phase 2 — Intake Form
- GitHub Issue form live
- `process_dns_request.py` auto-creates PRs
- Teams webhook notifications on PR open and apply complete
- Remove direct file editing access from non-DNS-team members

### Phase 3 — Parallel Batching
- Split VNet links across 5 batch workspaces
- Docker execution environment in ACR
- Spoke auto-discovery via `discover_spokes.py`
- Target: full apply in under 5 minutes

### Phase 4 — Governance
- Azure Policy in Audit mode → escalate to Deny
- Event Grid subscriptions for zone create and delete events
- Logic App: auto-nuke unmanaged zones, self-heal deleted managed zones
- Nightly `detect_unmanaged.py` scan with Teams alert

### Phase 5 — Enterprise Integration (Future)
- Migrate intake form to ServiceNow Service Catalog
- Move pipeline to Azure DevOps
- ServiceNow RITM auto-closure on apply success
- Logic App Teams approval cards

### Phase 6 — Infoblox Integration (Future)
- Service account provisioned for WAPI access
- `infoblox_sync.py` added to pipeline — runs after Terraform apply
- Zone created in Infoblox atomically with Azure in the same pipeline run
- Deletion blocked if zone has active DNS records in Infoblox
- Nightly orphan record scan across all Infoblox views

---

## Scaling Reference

| Scenario | Approach |
|---|---|
| VNets double (400+) | Add batch folders 07-10, update discovery script |
| Zones grow (200+) | Increase TF parallelism within existing batches |
| New hub VNet | Add to hubs.json, discovery picks up all peered spokes automatically |
| New subscription | Update registry.json mapping, pipeline logic unchanged |

---

## Open Items

- [ ] Decide on naming convention enforcement (suffix requirement for zone FQDNs)
- [ ] Define batch split logic — static assignment vs dynamic based on VNet count
- [ ] Confirm ACR location and access policy
- [ ] ServiceNow instance URL and catalog category for Phase 5
- [ ] Infoblox service account permissions for WAPI zone management (Phase 6)
- [ ] Infoblox WAPI version to target (Phase 6)
