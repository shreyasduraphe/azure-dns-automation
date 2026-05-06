# Enterprise GitOps Architecture: Scalable Private DNS Management in Azure

This architectural blueprint outlines the transformation of our Azure Private DNS infrastructure from a slow, monolithic, serial process into a highly parallel, segmented, and event-driven GitOps framework.

---

## 1. Executive Summary & Core Engineering Challenges

Our management model faces severe operational limitations when scaled to **92 Private DNS Zones** and **200 Virtual Networks (VNets)**, resulting in up to **18,400 potential Virtual Network Links**. 

### Explicit Bottleneck Remediation Analysis

* **Slow Discovery Operations:**
    * *The Problem:* The current approach executes ~90 sequential `Get-AzPrivateDnsVirtualNetworkLink` calls. Because these calls poll the Azure control plane back-to-back, the total discovery time scales linearly ($O(N)$), causing severe upstream delays.
    * *The Solution:* We eliminate global runtime polling. By utilizing segmented state files, individual pipeline runners only interrogate their specific, regional subset of links. Real-time drift detection is shifted entirely to event-driven push notifications.
* **Full Scan Overhead:**
    * *The Problem:* The engine evaluates all ~90 zones against ~200 VNets (~18,400 combinations) during every single scheduled run, even when zero infrastructure changes have occurred in the subscription.
    * *The Solution:* We shift from a polling model to a reactive model using **Azure Resource Notifications (ARN)** and **Azure Event Grid**. The automation engine remains completely dormant until an explicit creation or deletion event is broadcast by the Azure platform.
* **Long Write Latency:**
    * *The Problem:* Each virtual network link creation requires roughly 90 seconds to fully clear the cloud provider's network fabric. Running these sequentially locks the execution thread for hours.
    * *The Solution:* We max out resource deployment concurrency by appending the `-parallelism=50` execution flag to our Terraform orchestration layer, letting us process up to 50 links simultaneously.
* **Runner Timeouts:**
    * *The Problem:* Due to the combined overhead of slow discovery, full scans, and sequential writes, the total execution time routinely breaches maximum job runtime limits on our GitHub Actions runners, abruptly killing active deployments.
    * *The Solution:* **Blast Radius Reduction via State Segmentation.** Breaking our single state file into distinct regional workspaces caps individual pipeline runs to a tiny fraction of the global matrix, keeping execution times under 3 minutes.
* **Concurrent Run & Overlap Conflicts:**
    * *The Problem:* The existing PowerShell script is scheduled via a rigid 1-hour CRON loop. If a previous long-running write execution times out or runs over its block, subsequent loops overlap, creating state file lock contention and API race conditions.
    * *The Solution:* We eliminate scheduled loops completely. Self-healing and compliance evaluations are atomic and reactive—triggered instantly on-demand via **GitHub Repository Dispatch** webhooks fired from our event pipeline.
* **CI Setup Overhead:**
    * *The Problem:* Every scheduled run wastes roughly 5 minutes downloading, extracting, and installing the heavy Azure PowerShell modules (`Az.Network`, `Az.Resources`) and script dependencies onto a raw runner environment.
    * *The Solution:* We utilize custom, pre-baked **Dockerized GitHub Runner Images**. All binary toolsets—including Terraform, the Azure CLI, and specialized Python SDK dependencies—are baked into the container image, dropping pipeline cold-start overhead to near zero.

---

## 2. System Architecture & Request Flows

The architecture handles three distinct operational scenarios: **Authorized CI/CD Changes**, **Unauthorized Manual Additions**, and **Unauthorized Manual Deletions**.

### Scenario 1: Authorized GitOps Lifecycle (GitHub Issue Form)
This path represents the standard, authorized mechanism for adding zones, removing zones, or connecting a newly introduced hub/spoke network.

1. **User Submits Form:** The user submits a GitHub Issue Form requesting a new network hub or zone.
2. **Python Automation Script:** Parses input data and targets a specific region.
3. **Appends Configuration:** Appends configuration to a segmented regional `terraform.tfvars` file and raises a GitHub Pull Request.
4. **Engineer Approval:** A peer reviews and merges the PR.
5. **Execution:** A GitHub Actions runner initiates using a custom, pre-baked Docker container image.
6. **Deploy:** Runs `terraform apply -parallelism=50` to process infrastructure modifications concurrently.
7. **Compliance Audit:** A final step packages execution metrics and uploads a standardized JSON log directly to Azure Blob Storage.

### Scenario 2: Reactive Manual Creation Interception (Nuke & Destroy)
When an unauthorized operator creates a Private DNS Zone or Virtual Network Link through the Azure Portal or Azure CLI without the mandatory `managed-terraform` tag.

1. **Rogue Creation:** A user manually spins up an untagged resource.
2. **Event Broadcast:** Azure Resource Notifications (ARN) streams the live control plane change to an **Azure Event Grid System Topic**.
3. **Advanced Filter Match:** The Event Grid subscription utilizes an Advanced Filter to catch unmanaged resources:
   * `EventType == Microsoft.ResourceNotifications.Resources.CreatedOrUpdated`
   * `data.resourceInfo.tags.managed-terraform StringNotContains true`
4. **Function Wake Up:** An Azure Function (Python) wakes up instantly on the edge match.
5. **Nuke & Log:** The function reads the resource ID from the JSON payload, calls the Azure SDK `.delete()` method, generates a compliance audit log, and drops it into your blob storage.

### Scenario 3: Reactive Manual Deletion Interception (Self-Healing Loop)
When a corporate-managed, legacy, or active GitOps asset is deleted out-of-band directly inside the Azure portal.

1. **Manual Deletion:** An operator accidentally or maliciously deletes a managed link or zone.
2. **Event Broadcast:** ARN streams a `Microsoft.ResourceNotifications.Resources.Deleted` event to Event Grid.
3. **Advanced Filter Match:** Event Grid intercepts the deleted event for the DNS/Link resource types and triggers your Azure Function.
4. **Audit Alert:** The Function logs a `DELETION_DETECTED` audit entry to your blob storage.
5. **Webhook Dispatch:** The Function acts as an automated API broker and fires a POST request to the **GitHub Repository Dispatch API**.
6. **Self-Heal:** GitHub Actions instantly activates a pipeline running `terraform apply -refresh-only` to sync state, followed by an auto-approved apply to completely recreate the missing link within 2-3 minutes.

---

## 3. GitHub Issue Form Configuration & Repository Layout

To transition users completely away from manual Azure portal modifications, GitHub Issue Forms are utilized to enforce data entry constraints before any code changes are generated.

### GitHub Issue Form Template Details (`.github/ISSUE_TEMPLATE/dns_vnet_link.yml`)
```yaml
name: "Connect New Network Architecture"
description: "Submit a request to provision a new Private DNS Zone or attach a new Hub/Spoke Virtual Network."
title: "[NetOps-Provisioning]: "
labels: ["automated-provisioning", "gitops"]
body:
  - type: markdown
    attributes:
      value: "### Infrastructure Automation Request Form"
  - type: dropdown
    id: request_type
    attributes:
      label: "Action Type"
      options:
        - "Add New Private DNS Zone(s)"
        - "Attach New Hub VNet (Link All Zones)"
        - "Remove Infrastructure"
    validations:
      required: true
  - type: input
    id: target_region
    attributes:
      label: "Target Azure Region"
      placeholder: "e.g., eastus, westus, centralus"
    validations:
      required: true
  - type: textarea
    id: resource_identifiers
    attributes:
      label: "Resource List / IDs"
      description: "Provide comma-separated private DNS zone names OR the full Azure Resource ID of the target VNet."
      placeholder: "e.g., privatelink.database.azure.com OR /subscriptions/xxx/resourceGroups/xxx/providers/Microsoft.Network/virtualNetworks/hub-vnet-eastus"
    validations:
      required: true
```

### Segmented Repository Layout
```text
azure-dns-governance/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   └── dns_vnet_link.yml     # Enforced front-end data input form
│   └── workflows/
│       ├── gitops_deploy.yml     # Standard pipeline (triggers on PR merge)
│       └── self_heal.yml         # Fast remediation pipeline (via Repository Dispatch API)
├── .runners/
│   └── Dockerfile                # Custom pre-baked execution runner image
├── 01-core-zones/                # State Key: core-zones.tfstate (Changes rarely)
│   ├── main.tf                   # Declares the 92 base Private DNS Zones
│   └── outputs.tf                # Exposes zone names to downstream consumers
├── 02-links-hub-eastus/          # State Key: links-eastus.tfstate (Changes frequently)
│   ├── main.tf                   # Evaluates ONLY East US VNet links
│   └── terraform.tfvars          # Appended to dynamically by GitHub form engine
└── 03-links-hub-westus/          # State Key: links-westus.tfstate (Changes frequently)
    └── main.tf                   # Evaluates ONLY West US VNet links
```

---

## 4. Terraform Solutions & Asynchronous API Safeguards

### Mitigating Phantom "200 OK / 202 Accepted" False Successes
The Azure API frequently operates asynchronously. When deploying multiple VNet links, the Azure fabric may return an immediate HTTP `200 OK` or `202 Accepted` indicating that the *payload was valid and queued*. However, the resource provisioning process could still fail or time out behind the scenes within Azure's internal network mesh. 

To safeguard against these false positives, our architecture enforces two explicit protections:

1.  **Extended Resource Polling Windows:** We inject explicit `timeouts` blocks directly into our `azurerm_private_dns_zone_virtual_network_link` resource block. This forces the Terraform provider to actively verify that the `provisioningState` transitions completely to `Succeeded` before relinquishing its lock and marking the deployment as complete.
2.  **Asynchronous Background Status Checks:** In tandem with extended timeouts, our Azure Event Grid monitor scans for `Microsoft.ResourceNotifications.Resources.CreatedOrUpdated` payloads where `properties.provisioningState == "Failed"`. If an async background failure occurs after a pipeline has finished, Event Grid catches it and triggers our self-healing GitOps flow to fix it.

### Core Link Matrix Configuration (`02-links-hub-eastus/main.tf`)
```hcl
terraform {
  backend "azurerm" {
    container_name       = "tfstate"
    key                  = "links-eastus.tfstate"
  }
}

# Fetch the 92 zones dynamically via Azure Control Plane Data Sources
data "azurerm_private_dns_zone" "all_zones" {
  for_each            = toset(var.zone_names_list)
  name                = each.key
  resource_group_name = "dns-core-rg"
}

locals {
  # Computes the Cartesian matrix: 92 zones x Number of Regional Hubs
  zone_vnet_pairs = setproduct(keys(data.azurerm_private_dns_zone.all_zones), keys(var.east_us_hubs))

  vlink_matrix = {
    for pair in local.zone_vnet_pairs : "${pair[1]}-${pair[0]}" => {
      zone_name = pair[0]
      vnet_key  = pair[1]
      vnet_id   = var.east_us_hubs[pair[1]]
    }
  }
}

resource "azurerm_private_dns_zone_virtual_network_link" "links" {
  for_each              = local.vlink_matrix
  name                  = "vlink-${each.value.vnet_key}"
  resource_group_name   = "dns-core-rg"
  private_dns_zone_name = data.azurerm_private_dns_zone.all_zones[each.value.zone_name].name
  virtual_network_id    = each.value.vnet_id
  registration_enabled  = false

  # Explicit mitigation parameters for async API behavior
  timeouts {
    create = "15m"  # Overrides default allowances to ensure true back-end provisioning confirmation
    delete = "15m"
  }
}
```

---

## 5. Standardized Compliance Reporting Framework

All execution engines write to a centralized, **Immutable WORM (Write Once, Read Many)** storage container utilizing a single, schema-enforced JSON standard.

### The Standardized Schema (`audit-compliance.json`)
```json
{
  "audit_id": "UUID-v4-generated-string",
  "timestamp": "2026-05-06T17:42:00Z",
  "environment": "hub-spoke-network",
  "trigger_source": "GITHUB_ACTIONS | EVENT_GRID_FUNCTION",
  "execution_mode": "AUTHORIZED_PIPELINE | ENFORCEMENT_NUKE | AUTO_REMEDIATION",
  "actor": {
    "identity": "user-email@company.com | automation-service-principal@tenant.com",
    "type": "User | ServicePrincipal"
  },
  "action_summary": {
    "operation": "CREATE | DELETE | MODIFY | SELF_HEAL",
    "status": "SUCCESS | FAILED",
    "target_count": 92
  },
  "resources_affected": [
    {
      "resource_id": "/subscriptions/xxx/resourceGroups/dns-core-rg/providers/Microsoft.Network/privateDnsZones/privatelink.database.azure.com/virtualNetworkLinks/vlink-eastus-hub-2",
      "type": "Microsoft.Network/privateDnsZones/virtualNetworkLinks",
      "action_taken": "DEPLOYED | DESTROYED | RECREATED",
      "associated_issue_form": "https://github.com/your-org/your-repo/issues/104"
    }
  ],
  "raw_system_metadata": {}
}
```

---

## 6. Future Scope: Infoblox Core DNS Hybrid Integration

As part of our multi-cloud networking strategy, upcoming iterations of this automation engine will incorporate a core integration module with our hybrid corporate DNS solution (**Infoblox**).

### Infoblox WAPI Zone Forwarding Module Design
When a new Private DNS Zone is provisioned via our Azure GitOps pipeline, on-premises workloads and hybrid cloud data centers connected via ExpressRoute or VPN gateways must remain able to resolve those Azure private endpoints.

To orchestrate this, our GitHub Actions post-deployment step will communicate securely with the internal Infoblox grids via the **Infoblox WAPI (RESTful API)**. This automated workflow dynamically instantiates an on-premises **Conditional Forward Zone** that points directly to our dedicated Azure inbound resolver IPs, seamlessly bridging on-premises resolution to the Azure fabric architecture without human administrative latency.

```text
[Azure GitOps Deploys New Zone]
               │
               ▼
 [GitHub Actions Pipeline Webhook]
               │
               ▼ (Secure Outbound Auth Call)
 [Infoblox REST WAPI Endpoint] ──> [Creates Conditional Forwarder Row Instantly]
```
