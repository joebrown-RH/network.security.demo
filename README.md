# Network ACL Emergency Lockdown Demo

Automated CVE-driven network lockdown using Red Hat Ansible Automation Platform and Event-Driven Ansible. Part of the **Security in the AI Era** campaign.

## What This Demo Shows

A vulnerability scanner detects a critical CVE. A webhook fires to Event-Driven Ansible, which triggers a workflow that:

1. **Backs up** router configurations
2. **Applies emergency ACLs** blocking the exploit vector across all Cisco IOS devices
3. **Validates** the lockdown is active on every device
4. **Creates a ServiceNow change request** for audit trail

The entire flow runs in minutes with zero manual CLI access.

```
Scanner Alert ──> EDA Webhook ──> AAP Workflow
                                      │
                    ┌─────────────────┼─────────────────────┐
                    v                 v                     v
              Backup Config    ACL Lockdown ──> Validate ──> ServiceNow CR
                                    │
                                 (on failure)
                                    v
                              Restore Backup
```

## RHDP Lab Setup

### Prerequisites

- **RHDP Catalog Item**: "Ansible Network Automation" from [demo.redhat.com](https://demo.redhat.com)
- The lab provides: AAP Controller, EDA Controller, Cisco IOS-XE routers

### Step 1: Create the Project in AAP

1. Log in to AAP Controller
2. Navigate to **Projects** > **Add**
3. Configure:
   - **Name**: `Network Security Demo`
   - **Organization**: Default
   - **Source Control Type**: Git
   - **Source Control URL**: *(your git repo URL for this project)*
   - **Update Revision on Launch**: Enabled

### Step 2: Create Job Templates

Create the following job templates, all using the `Network Security Demo` project:

| Template Name | Playbook | Inventory | Credentials |
|---|---|---|---|
| Network - Backup Config | `playbooks/network_acl_backup.yml` | Network Inventory | Network Credential |
| Network - ACL Lockdown | `playbooks/network_acl_lockdown.yml` | Network Inventory | Network Credential |
| Network - Validate Lockdown | `playbooks/network_acl_validate.yml` | Network Inventory | Network Credential |
| Network - Create ServiceNow CR | `playbooks/servicenow_create_change.yml` | Demo Inventory | ServiceNow Credential |
| Network - ACL Remove | `playbooks/network_acl_remove.yml` | Network Inventory | Network Credential |

For **Network - ACL Lockdown**, enable **Prompt on launch** for Extra Variables so the workflow can pass `cve_id`, `blocked_port`, and `blocked_protocol`.

### Step 3: Create the Workflow Template

1. Navigate to **Templates** > **Add** > **Add workflow template**
2. **Name**: `Network - CVE Emergency Lockdown`
3. Build the workflow:

```
[Network - Backup Config]
        │ (on success)
        v
[Network - ACL Lockdown]
        │ (on success)          │ (on failure)
        v                       v
[Network - Validate Lockdown]  [Restore from backup - manual]
        │ (on success)
        v
[Network - Create ServiceNow CR]
```

4. Enable **Prompt on launch** for Extra Variables on the workflow template
5. Check **Pass extra variables to nodes** so `cve_id`, `blocked_port`, and `blocked_protocol` flow through

### Step 4: Configure Event-Driven Ansible

1. Log in to the EDA Controller
2. Create a **Project** pointing to this repo
3. Create a **Rulebook Activation**:
   - **Name**: `CVE Emergency Lockdown`
   - **Project**: Network Security Demo
   - **Rulebook**: `rulebooks/cve_webhook_lockdown.yml`
   - **Decision Environment**: Default Decision Environment
   - **Credential**: AAP Controller Credential
4. Enable the activation

### Step 5: Test the Demo

Trigger the lockdown with a simulated scanner alert:

```bash
# Simulate a critical CVE alert (SMB/EternalBlue scenario)
curl -X POST http://<eda-controller>:5000/endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "cve_id": "CVE-2017-0144",
    "severity": "critical",
    "action": "lockdown",
    "port": "445",
    "protocol": "tcp",
    "description": "SMB remote code execution - worm propagation vector"
  }'
```

Then watch the workflow execute in the AAP Controller UI.

See `vars/acl_profiles.yml` for additional pre-built scenarios (RDP, HTTP app, DNS amplification, Telnet).

### Step 6: Clean Up

After the demo, remove the emergency ACLs:

1. Run the **Network - ACL Remove** job template, or
2. Launch manually from the CLI:
   ```bash
   ansible-playbook playbooks/network_acl_remove.yml -i inventory
   ```

## File Structure

```
playbooks/
  network_acl_backup.yml        # Backup running config before changes
  network_acl_lockdown.yml      # Apply emergency ACLs to block exploit vector
  network_acl_validate.yml      # Verify ACLs are applied and active
  network_acl_remove.yml        # Remove emergency ACLs post-patch
  servicenow_create_change.yml  # Create ServiceNow emergency change request

rulebooks/
  cve_webhook_lockdown.yml      # EDA rulebook: webhook trigger -> workflow

vars/
  acl_profiles.yml              # Pre-built lockdown profiles for common CVEs

blog/
  network-acl-lockdown-blog.md  # Blog post about this demo
```

## Required Collections

These should be available in the execution environment:

- `cisco.ios` >= 5.0.0
- `servicenow.itsm` >= 2.4.0
- `ansible.eda` >= 1.0.0 (for the decision environment)

## Campaign Alignment

This demo maps to the **Mitigate** phase of the Security in the AI Era campaign workflow:

> *Deploys targeted controls at scale when patches aren't available -- breaking exploit chains in hours.*

Key talking points:
- MTTE has flipped from a 44-day head start to a 7-day deficit
- 45% of vulnerabilities remain unpatched after 12 months
- Automation closes the gap between discovery and remediation
- Governed execution: RBAC, Vault, audit trails, approval workflows
