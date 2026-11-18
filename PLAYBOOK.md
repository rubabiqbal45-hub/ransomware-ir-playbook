# Ransomware Incident Response Playbook

Nov 18, 2026 · @Rubab

A hands-on incident response playbook for a ransomware attack, built on the NIST SP 800-61 incident-handling lifecycle, then stress-tested against a tabletop exercise and revised based on what broke.

## Scope & Assumptions

**Scenario:** A ransomware strain (e.g. LockBit/Akira-style) is delivered via a phishing email with a malicious attachment, gains a foothold on a single employee workstation, then spreads laterally across the internal network before encrypting files on shared drives and triggering a ransom note.

**Environment assumed:**

- A small-to-mid-size organization running Windows Active Directory (domain controller + domain-joined endpoints)
- Endpoint monitoring (EDR or at minimum Sysmon + centralized logging)
- A SIEM collecting Windows Security and Sysmon events
- Nightly backups, though their integrity has not been independently verified before this playbook

**Roles referenced throughout:**

- **Incident Commander (IC):** owns the decision to contain, coordinates everyone else
- **IT/SysAdmin:** executes technical containment, eradication, and recovery steps
- **SOC Analyst:** monitors detections, confirms scope, hands findings to the IC
- **Communications Lead:** handles internal updates and any external/regulatory notification
- **Leadership:** makes business-impact decisions (e.g. whether to pay a ransom — this playbook assumes **no**, consistent with FBI/CISA guidance, and focuses entirely on recovery without payment)

This playbook assumes the organization does **not** negotiate with or pay the attacker. Every containment and recovery step is written to stand on its own without that option on the table.

## 1. Preparation

What needs to already be true before an incident happens — none of this is done *during* a ransomware attack, it's the groundwork that determines whether the rest of this playbook is even possible to execute.

- **Backups follow the 3-2-1 rule** (3 copies, 2 different media types, 1 offline/offsite) and at least one copy is genuinely offline or immutable — ransomware that reaches a backup server connected to the network can encrypt backups too
- **Backup restores are tested on a schedule** (e.g. quarterly), not just assumed to work — a backup nobody has restored from is a hope, not a plan
- **Asset inventory exists**: a current list of critical systems, their owners, and dependencies, so "what's affected" can be answered in minutes, not hours
- **EDR/antivirus is deployed and centrally monitored** on all endpoints and servers
- **Logging is centralized** (SIEM or equivalent) so investigators aren't pulling logs off individual machines one at a time mid-incident
- **IR contact list is current**: internal responders, leadership, legal counsel, cyber insurance provider, and (if retained) an external IR firm — with phone numbers, since email/Slack may be unavailable during the incident
- **An out-of-band communication channel exists** (e.g. a phone tree, a Signal group) for when the primary network and email are the thing that's compromised
- **This playbook itself is stored somewhere reachable without the network** — a PDF on someone's phone, a printed copy — not only on a file share that ransomware might encrypt
- An Incident Commander is named in advance, with a backup and succession order — so if the first-choice IC is unreachable when the incident starts, authority to act doesn't stall waiting for them specifically.

## 2. Detection & Analysis

**How this typically gets noticed:**

- A user reports files won't open, or sees a ransom note (`README_RESTORE.txt`, a changed desktop wallpaper) — often the *first* signal in less mature environments
- EDR/antivirus alerts on known ransomware behavior (mass file renaming, shadow copy deletion, suspicious encryption-related process activity)
- SIEM alerts on specific Windows Event IDs tied to ransomware behavior:
  - **Event 4688** (process creation) showing `vssadmin delete shadows` or `wbadmin delete catalog` — ransomware deleting Volume Shadow Copies so Windows' own built-in recovery can't be used
  - A spike in **Event 4663** (object access) or file-system activity showing thousands of files being renamed/modified in a short window
  - **Event 4624/4625** showing unusual logons (off-hours, from an account that doesn't normally touch that host) just before the encryption activity starts

**Initial analysis questions, in order:**

1. **What is patient zero?** Which host showed the first sign of compromise — check EDR/SIEM timestamps across hosts to find the earliest.
2. **How did it get in?** Phishing email, exposed RDP, a vulnerable public-facing service, a compromised credential? This drives both containment (close that specific door) and eradication later.
3. **How far has it spread?** Check for the same indicators (shadow-copy deletion commands, the same file extensions appearing) on other hosts, and check Active Directory logs for lateral movement (unusual logons from the patient-zero host to other machines).
4. **Is this still actively spreading, or has it already finished encrypting?** This determines how urgent containment is versus whether the priority shifts straight to scoping for recovery.

**Severity classification:** if more than one host is affected, or any domain controller / backup server shows signs of compromise, this is escalated to a **critical** incident and the Incident Commander activates the full response team immediately rather than handling it as routine IT ticket triage.

## 3. Containment

**Short-term containment (first 30-60 minutes, stop the bleeding):**

1. **Isolate affected hosts from the network** — disconnect the network cable or disable the NIC, or isolate via EDR's network-isolation feature if available. Do **not** power the machine off — this destroys memory-resident evidence (running processes, encryption keys that may still be in RAM) that could matter for forensics or even recovery.
2. **Disable the compromised user account(s)** at the domain level to stop any further lateral movement using those credentials.
3. **Block identified command-and-control (C2) indicators** (malicious IPs/domains) at the firewall if any were identified during detection.
4. **Disable or heavily restrict SMB/network shares** temporarily if encryption is actively spreading across shared drives — this buys time even before every affected host is individually isolated.

**Longer-term containment (next few hours):** 5. **Force a password reset for all potentially exposed accounts**, not just the one confirmed compromised — assume credential reuse until proven otherwise. 6. **Segment the network further** if lateral movement crossed VLANs, to box the incident into the smallest possible area. 7. **Preserve evidence**: take a forensic disk image (or at minimum full memory + disk snapshot) of patient zero and any other key affected hosts *before* any cleanup begins — needed both for root-cause analysis and for any legal/insurance requirements.

**Who has the authority to isolate a production system immediately, without going through normal change management?** This must be named explicitly, in advance, in the Preparation phase — otherwise containment stalls waiting for a ticket to be approved while the ransomware keeps spreading. (This playbook assumes the Incident Commander has that standing authority once an incident is declared critical.)

## 4. Eradication

**The goal here is not just "remove the malware" — it's removing every way the attacker could get back in.**

1. **Identify and close the initial access vector** found during Detection & Analysis (e.g. disable the exploited RDP exposure, reset the phished account's credentials and enforce MFA, patch the exploited vulnerability).
2. **Do not trust an "in-place clean"** of a compromised host. Scanning and removing the known ransomware binary is not sufficient — attackers routinely leave backdoors, scheduled tasks, or additional accounts behind. The default assumption for any confirmed-compromised host is **full wipe and rebuild from a known-clean image**, not disinfection.
3. **Check for persistence mechanisms** across the environment, not just on patient zero: new scheduled tasks, new or modified service accounts, new local admin accounts, modified Group Policy, unexpected entries in startup folders/registry run keys.
4. **Rotate every credential that could plausibly have been exposed** — not just the accounts directly involved, but any account whose credentials might have been cached on an affected host (e.g. a domain admin who happened to log into the compromised machine for an unrelated reason).
5. **Re-scan the full environment** with updated EDR/AV signatures before declaring eradication complete, since a secondary/backup malware implant is common in ransomware operations specifically to survive the obvious cleanup.

## 5. Recovery

1. **Verify backup integrity before trusting it.** Before restoring anything, confirm the backup itself predates the compromise and was not also encrypted or tampered with — check backup timestamps against the known infection timeline, and test-restore a sample file to a clean, isolated system first.
2. If a public decryptor exists for the identified strain, verify it before trusting it. Confirm it through a reputable source (e.g. the No More Ransom project, the EDR/AV vendor, or a recognized IR firm) rather than a random download link — fake or outdated decryptors circulate and can themselves be malicious. Prefer a verified, recent, clean backup over a decryptor when both are available; a decryptor only reverses encryption, it does not confirm the host is otherwise clean, so eradication steps still apply in full even after successful decryption.
3. **Restore in priority order**: critical business systems and anything needed to resume core operations first, lower-priority systems after. The asset inventory from Preparation is what makes this prioritization possible to do quickly rather than debated in the moment.
4. **Restore onto rebuilt (not merely cleaned) systems**, per the Eradication phase — recovery and eradication happen together, not recovery after eradication is "done."
5. **Keep restored systems network-isolated initially** and monitor them closely for a defined period (e.g. 48-72 hours) before fully reconnecting them to production — watching specifically for the same indicators that flagged the original infection, in case eradication missed something.
6. **Change all passwords again post-recovery**, even ones already rotated during containment/eradication, as a final safety margin before declaring the incident closed.
7. **Formally declare recovery complete** only once the Incident Commander confirms: no new detections for the monitoring period, all priority systems restored and verified functional, and all affected credentials rotated.

## 6. Post-Incident Activity & Reporting

**Lessons-learned review** — held within a week of the incident closing, while details are still fresh, with every role from the Scope section present:

- What actually happened, in a plain timeline (first compromise → detection → containment → recovery)
- What worked
- What didn't work, or took longer than it should have
- What changes to this playbook, tooling, or training come out of it

**Report structure, for leadership and (if applicable) regulators/insurers:**

1. **Executive summary** — what happened, impact, current status, in a few sentences a non-technical reader can act on
2. **Timeline** — dated/timestamped sequence of events from initial compromise through recovery
3. **Root cause** — exactly how the attacker got in (the specific vulnerability, phishing email, or exposed service)
4. **Impact** — systems affected, data affected (and whether any was exfiltrated, not just encrypted — many modern ransomware operations steal data before encrypting it, which changes legal/notification obligations), downtime, estimated cost
5. **Remediation actions taken** — what was fixed, patched, or changed as a direct result
6. **Recommendations** — what should change going forward to reduce the chance of recurrence

**Legal/regulatory note:** if personal data was confirmed or suspected to be exfiltrated (not just encrypted in place), this may trigger breach-notification obligations depending on jurisdiction and data type — the Communications Lead and legal counsel make this call together, not IT alone.

## Tabletop Exercise: Scenario Walkthrough

Ran this solo, playing every role (IC, SOC Analyst, SysAdmin, Communications Lead) in turn, working strictly from the playbook above as written — no shortcuts, no knowledge not actually written down in it.

**Inject (the scenario given to start):** *"It's 9:14 AM. An employee in Finance reports their desktop wallpaper changed to a ransom note. Two other Finance workstations report the same within the next 10 minutes. Shared drive files are inaccessible."*

**9:14 — Detection & Analysis.** Per the playbook: identify patient zero. Checked EDR timestamps across the three affected hosts — Finance-WS-03 shows the earliest suspicious process activity, 8:52 AM, a `vssadmin delete shadows` command (the exact Event 4688 indicator the playbook names). Confirmed via the Received-header-equivalent for this scenario (the email gateway log): Finance-WS-03's user opened an attachment named `Invoice_8841.pdf.exe` at 8:50 AM from a phishing email. Initial access vector identified: phishing attachment, not RDP exposure.

**9:20 — Severity classification.** Three hosts, one being on a shared drive used company-wide → classified critical per playbook's own threshold ("more than one host affected"). IC role activated the full response.

**9:25 — Containment.** Attempted to isolate Finance-WS-03 via EDR network isolation. **This is where the exercise found its first real gap** — see Findings below. Proceeded with the fallback (physical network disconnect) and disabled the three affected user accounts at the domain level.

**9:40 — Containment continued.** Disabled SMB access to the affected shared drive org-wide as a stopgap, per playbook step 4. Began forensic disk imaging of Finance-WS-03 before any cleanup, per playbook step 7.

**10:15 — Eradication.** Attempted to identify persistence mechanisms per playbook step 3. **Second gap found** — see Findings below.

**11:00 — Recovery.** Checked backup timestamps against the 8:50 AM infection time — last clean backup was 2:00 AM that day, before infection. Test-restored a sample file to an isolated VM successfully. Began phased restore of the shared drive from that backup.

**11:45 — Post-Incident.** Drafted the report structure per playbook step 6, walked through what data exposure assessment would actually require.

Full exercise, start to finish: roughly 1 hour of simulated-incident time, compressed into about 50 minutes of actual tabletop time.

## Findings From the Tabletop & Fixes Made

The point of running this exercise was to find exactly these kinds of gaps — a playbook nobody has stress-tested just *looks* complete. Two real ones turned up, and both are already reflected in the sections above (this section documents what changed and why, rather than leaving the reader to guess).

**Gap 1 — Containment had no fallback if EDR isolation itself fails.** The first draft's Containment section said only "isolate via EDR's network-isolation feature." Walking through the exercise, the obvious next question was: *what if the EDR console is slow, unreachable, or the agent on that host has already been killed by the malware?* The original draft had no answer — it assumed the easy path always works. **Fix:** Containment now explicitly lists the physical/manual fallback (disconnect the NIC or network cable directly) as an equal first-class option, not an afterthought, and the "who has authority to isolate immediately" question was added precisely because this gap surfaced it — if EDR isolation fails and someone has to physically pull a cable, that requires someone with physical access and the standing authority to act without waiting for approval.

**Gap 2 — "Check for persistence mechanisms" was too vague to actually execute under pressure.** The first draft's Eradication section said to check for persistence mechanisms, full stop. During the walkthrough, acting as the SysAdmin role, it became clear that instruction gives no actual starting point when you're moving fast — *check where, with what, looking for what specifically?* **Fix:** Eradication step 3 was rewritten to name the concrete, specific things to check (new scheduled tasks, new/modified service accounts, new local admin accounts, modified Group Policy, registry run keys) rather than the vague original instruction — something a responder can tick through directly instead of having to invent a checklist mid-incident.

**Broader takeaway:** both gaps were in steps that *sounded* complete on paper — "isolate the host," "check for persistence" — but fell apart the moment someone tried to actually execute them without additional judgment calls the original playbook hadn't made explicit. That's exactly the kind of gap a tabletop surfaces and a read-through never would.

**Gap 3 — Preparation never actually named who the Incident Commander is.** Containment says authority to isolate "must be named explicitly, in advance, in the Preparation phase" — but the Preparation checklist itself had no item saying so. Running this live, the question of *who specifically* makes the isolation call, and who's the backup if they're unreachable, wasn't actually written down anywhere. **Fix:** added a Preparation item naming the IC and a backup/succession order explicitly.

**Gap 4 — No guidance on when a public decryptor is safe to use.** Recovery only covered restoring from backup. A live inject involving a known strain with a public decryptor available exposed that the playbook gave no criteria for trusting it versus a fake/scam decryptor, or how it fits alongside backup restoration. **Fix:** added decryptor-specific guidance to Recovery — verify against a trusted source before using one, prefer a clean backup when available, and treat decryption as separate from eradication.
