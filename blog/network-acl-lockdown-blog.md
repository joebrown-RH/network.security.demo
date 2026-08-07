# A Critical CVE Just Dropped and There's No Patch. Now What?

It's 2pm on a Tuesday. A critical CVE hits the wire affecting a protocol your entire network depends on. Your vulnerability scanner lights up, every router in the fleet is exposed. You check the vendor advisory: no patch available yet. Estimated timeline: weeks.

You know what needs to happen. Someone has to log into every affected router, write an ACL to block the exploited port, apply it to the right interfaces, verify it took, and document the whole thing for the change advisory board. Across dozens or hundreds of devices, without breaking production traffic. If your team is doing this by hand, that's days of work. Days where every one of those devices is sitting there waiting to be exploited.

## The real problem isn't writing the ACL

Network engineers know how to write ACLs. That part is straightforward. What slows everything down is the operational overhead around it — who's making the change, on which devices, in what order? Did someone already handle the east coast routers? Did every device get exactly the same ACL, or did someone fat-finger a sequence number on router 47? Did we back up the running config first? If this ACL breaks something in production, can we actually revert without making it worse?

And then there's the audit trail. When the compliance team comes asking what was changed, when, and who approved it, the answer can't be "check Dave's terminal history."

Scale that across a large fleet under time pressure and mistakes pile up fast. A missed interface here, a skipped device there — each one is a hole in your perimeter.

## Automate the response, not just the detection

We've gotten good at detection. Scanners, SIEM rules, threat feeds — the alert side is fast. The response side? Still a lot of SSH sessions and spreadsheets tracking who did what. That's the gap.

This demo connects detection directly to response using Ansible Automation Platform:

```
┌──────────────────┐
│  Vulnerability   │
│    Scanner       │
│ (Qualys/Tenable/ │
│   Rapid7/etc.)   │
└────────┬─────────┘
         │ webhook
         v
┌──────────────────┐
│   Event-Driven   │     "Is this critical?"
│     Ansible      │──── Non-critical: log for manual review
│                  │
└────────┬─────────┘
         │ critical + lockdown
         v
┌──────────────────────────────────────────────────┐
│           AAP Workflow Engine                     │
│                                                  │
│  ┌──────────┐    ┌──────────┐    ┌────────────┐ │
│  │  Backup  │───>│ Lockdown │───>│  Validate  │ │
│  │  Config  │    │   ACL    │    │  Lockdown  │ │
│  └──────────┘    └─────┬────┘    └────────────┘ │
│                        │                         │
│                   (on failure)                    │
│                        │                         │
│                        v                         │
│                  ┌───────────┐                   │
│                  │  Rollback │                   │
│                  │ (Remove   │                   │
│                  │   ACL)    │                   │
│                  └───────────┘                   │
└──────────────────────────────────────────────────┘
```

The scanner finds the vulnerability and fires a webhook. Event-Driven Ansible looks at the severity — critical threats trigger the lockdown workflow automatically, everything else gets logged for a human to triage.

Here's what the workflow actually does:

First, it backs up the running config on every target router. If anything goes sideways, there's a known-good state to revert to.

Then it pushes an extended ACL blocking the exploited port and protocol on all routed interfaces. The ACL only blocks the specific attack vector, deny the bad traffic,and permit everything else. Every deny rule has logging enabled and a remark with the CVE ID baked in, so six months from now when someone sees `EMERGENCY-CVE-BLOCK` in the config, they know exactly why it's there.

After the lockdown, the workflow actually verifies it worked. It gathers the ACL state from each device, checks that the rules exist, confirms they're applied to the right interfaces, and pulls hit counters. It doesn't just assume the change took — it proves it.

If the lockdown step fails on any device, the workflow doesn't leave things half-done. It automatically removes the ACL and restores normal operation.

The whole thing runs in minutes.

## Why this approach works

Under the hood, it's the exact same `ip access-list extended` config a network engineer would write. The automation doesn't do anything clever with the ACL itself, it just makes sure it gets applied identically on every device without the typos and missed interfaces that come from doing it manually under pressure at 2am.

The lockdown playbook takes the CVE ID, port, and protocol as variables, so the same workflow handles different attack vectors: SMB worms on 445, RDP exploits on 3389, rogue HTTP services on 8080, DNS amplification on 53/udp. You don't rebuild the automation for each incident, you just pass different parameters.

On the governance side, AAP handles the parts that usually require a spreadsheet and a prayer. RBAC controls who can trigger a lockdown. Credentials live in Vault, not in a playbook someone committed to Git. Every execution is logged with exactly what changed on which devices. When compliance asks questions, the answers already exist.

And when the actual patch is ready and deployed? A separate job template strips the emergency ACL from all interfaces and cleans up the rules. You're not left with stale emergency ACLs cluttering configs for the next three years because nobody remembers if it's safe to remove them.

## The gap between "we found it" and "we fixed it"

There's a window between when your scanner finds a vulnerability and when a patch actually lands in production. Sometimes that's days. Sometimes it's months, 45% of vulnerabilities in large environments are still sitting unpatched a year later. That window is where attackers live.

The tooling to detect is already fast. The tooling to patch is getting better. But that middle ground, the "we know about it but can't fix it yet" phase is where most organizations are still flying manual. That's the gap this automation fills. Your scanner finds the problem in seconds; the network-level mitigation deploys in minutes. No waiting for a change window. No war room. No hoping nobody exploits it before Friday.

---

*This demo runs on the Network Automation Workshop available from the Red Hat Demo Platform. The playbooks, EDA rulebook, containerlab topology, and setup instructions are in the [network.security.demo](https://github.com/joebrown-RH/network.security.demo) repository.*
