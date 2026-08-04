# Attackers Move in Hours. Your Network Lockdown Should Too.

In 2020, security teams had a 44-day head start between a vulnerability being published and the first exploit appearing in the wild. By 2025, that gap had collapsed into a **7-day deficit** -- meaning attackers are now building working exploits *before* most organizations even begin patching. AI-powered tools like frontier models can identify and weaponize 30-year-old zero-days in hours. The game has fundamentally changed.

So what happens when a critical CVE drops on a Tuesday afternoon, there is no patch available yet, and your vulnerability scanner just flagged 200 network devices as exposed?

If you are manually SSH-ing into routers to push ACLs, you are looking at days of work -- and days of exposure. If you have Ansible Automation Platform and Event-Driven Ansible, you are looking at **minutes**.

## The Scenario

A vulnerability scanner (Qualys, Tenable, Rapid7 -- pick your tool) detects a critical CVE affecting a widely-used network protocol. No vendor patch exists yet. The scanner fires a webhook to Event-Driven Ansible, which kicks off an automated lockdown workflow across your Cisco IOS infrastructure.

Here is what happens in the next 3 minutes -- with zero human CLI access required:

**1. Backup** -- The current running configuration on every affected router is backed up automatically. If anything goes wrong, we have a rollback point.

**2. Lockdown** -- An emergency ACL is pushed to all routed interfaces, blocking the exploited port and protocol. Every deny rule includes logging and a remark tagging the specific CVE ID for traceability. The ACL permits all other traffic -- we are surgically blocking the attack vector, not taking the network offline.

**3. Validate** -- The workflow verifies the ACL exists, confirms it is bound to the correct interfaces, and checks hit counters. If validation fails on any device, the workflow flags it immediately.

**4. Audit Trail** -- A ServiceNow emergency change request is created automatically with full details: which CVE, what was blocked, which devices were affected, when the lockdown was applied. Your change advisory board has a complete record without anyone filling out a form.

The entire flow -- from scanner alert to network-wide lockdown with a ServiceNow ticket -- runs without a human touching a CLI.

## Why This Matters for the AI Era

The security landscape has shifted in two critical ways that make this kind of automation non-negotiable.

First, **the volume is unmanageable without automation**. Annual CVE counts have grown 2.6x since 2020. Security teams cannot manually triage and respond to every alert, especially when 45% of vulnerabilities in large organizations remain unpatched after 12 months. The backlog is growing faster than teams can work through it.

Second, **patching alone is not enough**. Even when a patch exists, getting from patch availability to production deployment takes most organizations 31-90 days. That is an eternity when exploits are weaponized in hours. You need the ability to **mitigate** -- to deploy targeted controls that break the exploit chain while you evaluate and test the actual fix. That is exactly what this workflow does.

## The Automation Architecture

The solution uses three components of Red Hat Ansible Automation Platform working together:

- **Event-Driven Ansible** listens for webhook events from your vulnerability scanner and applies rules to determine the appropriate response. Critical severity triggers the lockdown workflow. Non-critical alerts are logged for manual review.

- **AAP Controller** orchestrates the multi-step workflow with full governance -- RBAC controls who can run what, Ansible Vault protects device credentials, the automation mesh ensures secure execution across network zones, and every action is logged in the audit trail.

- **Certified Content Collections** (`cisco.ios`, `servicenow.itsm`) provide idempotent, tested modules that handle the actual device configuration and ITSM integration. No custom scripts. No fragile screen-scraping.

The workflow is built as reusable job templates connected in a workflow template. The lockdown playbook accepts the CVE ID, port, and protocol as variables, so the same automation handles SMB worms, RDP exploits, DNS amplification -- any network-level attack vector where blocking a port buys time.

## From Reactive to Proactive

This pattern represents a fundamental shift. Instead of a security team receiving an alert, scheduling a change window, writing ACL changes, testing them, and manually deploying across the fleet -- the response is **immediate, consistent, and auditable**.

And when the patch is ready? A separate "ACL Remove" job template cleanly reverses the emergency rules, the ServiceNow change request gets closed, and the network returns to normal operations. The entire lifecycle is governed.

The Mean Time To Exploit is now measured in hours. Your Mean Time To Mitigate should be too.

---

*This demo can be built and tested using the Network Automation Lab available on the Red Hat Demo Platform (demo.redhat.com). The playbooks, EDA rulebook, and full setup instructions are available in the `network.security.demo` repository.*
