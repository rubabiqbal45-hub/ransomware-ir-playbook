# Ransomware Incident Response Playbook

A hands-on incident response playbook for a ransomware attack, built on the **NIST SP 800-61** incident-handling lifecycle (Preparation → Detection & Analysis → Containment → Eradication → Recovery → Post-Incident Activity), then actually stress-tested with a live tabletop exercise rather than left as a theoretical document.

This isn't a template filled in from a NIST checklist and left untested. After writing the playbook, I ran a solo tabletop exercise against it — played every role (Incident Commander, SOC Analyst, SysAdmin, Communications Lead) in turn, working strictly from what the playbook said, with no shortcuts and no knowledge that wasn't actually written down in it. That exercise found four real gaps in the playbook, documented below, each with the fix that closed it.

## Why a tabletop exercise, and why it matters

A playbook nobody has stress-tested just *looks* complete. Steps like "isolate the host" or "check for persistence mechanisms" read fine on paper but fall apart the moment someone tries to actually execute them under pressure, without the extra judgment calls the original draft assumed. Running the scenario live — reacting to one inject at a time, in real time, instead of reading a finished document start to finish — is what actually surfaces those gaps. That's the same reason real SOC/IR teams run tabletops: to find out where the plan breaks *before* a real incident does.

## The scenario

A ransomware strain delivered via a phishing email with a malicious attachment, gaining a foothold on a single employee workstation, then spreading laterally across the network before encrypting files on shared drives and dropping a ransom note. The playbook assumes the organization does **not** pay the ransom, consistent with FBI/CISA guidance, and focuses entirely on recovery without payment.

## Four real gaps found, and the fixes made

| # | Gap found | Fix made |
|---|---|---|
| 1 | Containment named only EDR network-isolation, with no fallback if the EDR console is slow, unreachable, or the agent was already killed by the malware | Added physical/manual disconnect (pull the NIC or cable) as an equal first-class containment option, and named who has standing authority to do it immediately |
| 2 | "Check for persistence mechanisms" gave no actual starting point under pressure | Rewritten to name concrete items: new scheduled tasks, new/modified service accounts, new local admin accounts, modified Group Policy, registry run keys |
| 3 | Containment referenced an Incident Commander "named explicitly in advance" — but Preparation never actually listed that as a checklist item | Added an explicit Preparation item: IC named in advance, with a backup/succession order |
| 4 | Recovery only covered restoring from backup — no criteria for when a public decryptor is safe to trust vs. a scam/fake one | Added decryptor-specific guidance: verify against a trusted source (e.g. the No More Ransom project) before using one, prefer a clean backup when available, and treat decryption as separate from eradication |

Gaps 1 and 2 came from a first authoring pass working through the scenario solo. Gaps 3 and 4 came from a second, live run-through where I answered each inject cold, in my own words, before checking what the playbook actually said — which is a more honest test than grading your own pre-written example.

## Skills demonstrated

- Incident response process design, grounded in NIST SP 800-61
- Tabletop exercise facilitation and self-assessment against a written plan
- Ransomware-specific technical knowledge: Windows Event IDs tied to ransomware behavior (4688 shadow-copy deletion, 4663 mass file modification, 4624/4625 unusual logons), initial-access-vector analysis, persistence mechanism hunting
- Containment/eradication/recovery sequencing under realistic constraints (EDR failure modes, backup integrity verification, decryptor risk)
- Incident reporting structure for both technical and non-technical (leadership/legal) audiences
- Documentation discipline: capturing what broke and why, not just a clean final version

## Files

- [`PLAYBOOK.md`](PLAYBOOK.md) — the full playbook: all six NIST phases, the tabletop scenario walkthrough, and the findings/fixes section
