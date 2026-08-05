# Network ACL Emergency Lockdown Demo

Automated CVE-driven network lockdown using Red Hat Ansible Automation Platform and Event-Driven Ansible, designed for the [Network Automation Workshop](https://github.com/rhpds/zt-network-automation-workshop) environment.

## What This Demo Shows

A vulnerability scanner detects a critical CVE. A webhook fires to Event-Driven Ansible, which triggers a workflow that:

1. **Backs up** router configurations
2. **Applies emergency ACLs** blocking the exploit vector on Cisco IOS devices
3. **Validates** the lockdown is active on every device

On failure, the workflow automatically **rolls back** by removing the ACL.

```
Scanner Alert ──> EDA Webhook ──> AAP Workflow
                                      │
                              Backup Config
                                      │
                               ACL Lockdown
                              /            \
                       (on success)    (on failure)
                            │               │
                    Validate Lockdown   ACL Remove
                                       (Rollback)
```

## Prerequisites

- **Network Automation Workshop** provisioned from [demo.redhat.com](https://demo.redhat.com) (catalog item: `zt-network-automation-workshop`)
- The workshop provides: AAP Controller, Cisco IOS-XE router (rtr1) via containerlab, and all required credentials and inventories

### Workshop Resources Used

| Resource | Name |
|----------|------|
| Organization | `Red Hat network organization` |
| Inventory | `Workshop Inventory` |
| Credential | `Workshop Credential` |
| Execution Environment | `Network EE` |
| Host Group | `cisco` (rtr1 — Cisco IOS-XE) |

## Setup

### 1. Create the Project

In AAP Controller, navigate to **Projects** > **Create Project**:

| Field | Value |
|-------|-------|
| Name | `Network Security Demo` |
| Organization | `Red Hat network organization` |
| Source Control Type | Git |
| Source Control URL | `https://github.com/joebrown-RH/network.security.demo.git` |
| Update Revision on Launch | Enabled |

### 2. Create and run the Setup Job Template

Navigate to **Templates** > **Create Job Template**:

| Field | Value |
|-------|-------|
| Name | `Setup - Network Security Demo` |
| Job Type | Run |
| Inventory | `Workshop Inventory` |
| Project | `Network Security Demo` |
| Playbook | `playbooks/setup_demo.yml` |
| Execution Environment | `Network EE` |
| Credentials | `Controller Credential` |

Launch the template. It creates:
- 4 job templates (Backup Config, ACL Lockdown, Validate Lockdown, ACL Remove)
- 1 workflow template (Network - CVE Emergency Lockdown)
- 1 EDA rulebook activation (CVE Emergency Network Lockdown)

All resources default to the workshop environment. Override with extra vars if needed:

```yaml
org: Red Hat network organization
project_name: Network Security Demo
network_inventory: Workshop Inventory
network_credential: Workshop Credential
execution_environment: Network EE
```

<details>
<summary>Manual Setup (Alternative)</summary>

#### Create Job Templates

Create the following job templates using the `Network Security Demo` project:

| Template Name | Playbook | Inventory | Credentials |
|---|---|---|---|
| Network - Backup Config | `playbooks/network_acl_backup.yml` | Workshop Inventory | Workshop Credential |
| Network - ACL Lockdown | `playbooks/network_acl_lockdown.yml` | Workshop Inventory | Workshop Credential |
| Network - Validate Lockdown | `playbooks/network_acl_validate.yml` | Workshop Inventory | Workshop Credential |
| Network - ACL Remove | `playbooks/network_acl_remove.yml` | Workshop Inventory | Workshop Credential |

For all templates, set the **Execution Environment** to `Network EE` and enable **Prompt on launch** for Extra Variables.

#### Create the Workflow Template

1. Navigate to **Templates** > **Create Workflow Job Template**
2. **Name**: `Network - CVE Emergency Lockdown`
3. **Organization**: `Red Hat network organization`
4. Build the workflow:

```
[Backup Config]
       │ (on success)
       v
[ACL Lockdown]
       │ (on success)          │ (on failure)
       v                       v
[Validate Lockdown]     [ACL Remove (Rollback)]
```

5. Enable **Prompt on launch** for Extra Variables
6. Check **Pass extra variables to nodes** so `cve_id`, `blocked_port`, and `blocked_protocol` flow through

#### Configure Event-Driven Ansible (Optional)

1. Log in to the EDA Controller
2. Create a **Project** pointing to this repo
3. Create a **Rulebook Activation**:
   - **Name**: `CVE Emergency Lockdown`
   - **Project**: Network Security Demo
   - **Rulebook**: `rulebooks/cve_webhook_lockdown.yml`
   - **Decision Environment**: Default Decision Environment
   - **Credential**: Controller Credential
4. Enable the activation

</details>

## Usage

### Manual Launch

Run the **Network - CVE Emergency Lockdown** workflow template with:

```yaml
cve_id: CVE-2017-0144
blocked_port: "445"
blocked_protocol: tcp
```

### Via EDA Webhook

```bash
curl -X POST http://<eda-host>:5000/endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "cve_id": "CVE-2017-0144",
    "severity": "critical",
    "action": "lockdown",
    "port": "445",
    "protocol": "tcp"
  }'
```

See `vars/acl_profiles.yml` for additional pre-built scenarios (RDP, HTTP app, DNS amplification, Telnet).

### Clean Up

Run the **Network - ACL Remove** job template to remove the emergency ACL from all interfaces and delete the ACL definition.

## File Structure

```
playbooks/
  setup_demo.yml                # CaC setup: creates all templates + workflow + EDA
  network_acl_backup.yml        # Backup running config before changes
  network_acl_lockdown.yml      # Apply emergency ACLs to block exploit vector
  network_acl_validate.yml      # Verify ACLs are applied and active
  network_acl_remove.yml        # Remove emergency ACLs post-patch
  servicenow_create_change.yml  # Create ServiceNow change request (not wired into workflow)

config_as_code/
  controller_templates.yml           # Job template definitions (4 templates)
  controller_templates_workflow.yml  # Workflow template definition
  eda_rulebook_activations.yml       # EDA rulebook activation definition

rulebooks/
  cve_webhook_lockdown.yml      # EDA rulebook: webhook trigger -> workflow

vars/
  acl_profiles.yml              # Pre-built lockdown profiles for common CVEs
```

## Required Collections

These are included in the `Network EE` execution environment:

- `cisco.ios` >= 5.0.0
- `infra.aap_configuration` (for the setup playbook)
- `ansible.eda` >= 1.0.0 (for the decision environment)

## Roadmap

- **ServiceNow integration** — emergency change request creation for audit trail (playbook exists, needs credential and workflow wiring)
- **Multi-vendor support** — extend lockdown playbooks to Arista EOS and Juniper Junos devices

## Campaign Alignment

This demo maps to the **Mitigate** phase of the Security in the AI Era campaign workflow:

> *Deploys targeted controls at scale when patches aren't available — breaking exploit chains in hours.*

Key talking points:
- MTTE has flipped from a 44-day head start to a 7-day deficit
- 45% of vulnerabilities remain unpatched after 12 months
- Automation closes the gap between discovery and remediation
- Governed execution: RBAC, Vault, audit trails, approval workflows
