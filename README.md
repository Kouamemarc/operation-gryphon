# Operation Gryphon — Red Team Challenge (M1-06)

Build, attack and defensive analysis of an Active Directory domain `meridian.local` (fictional bank "Meridian").
Solo project. Tier: baseline (domain controller only).

> **Disclaimer.** Educational repository. No real secrets (passwords, hashes) are committed: all sensitive values are redacted. The techniques shown must only be used against systems you are explicitly authorized to test.

---

## Summary

Starting from a single compromised user account, the attack escalates to full domain compromise and data exfiltration by abusing a misconfigured service account. The deferred defensive analysis then measures, from the domain controller logs, what would have been detected and what would have gone unnoticed.

**Attack chain (MITRE ATT&CK):**

| Phase | Technique | Action |
|-------|-----------|--------|
| 1 | T1087.002 | Reconnaissance and domain account enumeration |
| 2 | T1558.003 / T1110.002 | Kerberoasting of svc_sql then offline cracking |
| 3 | T1078.002 | Privilege escalation: svc_sql is Domain Admin |
| 5 | T1003.006 | DCSync: replication of domain secrets |
| 6 | T1550.002 | Pass-the-Hash of the Administrator account |
| 7 | T1039 | Exfiltration from the target share |

**Key defensive finding:** DCSync is strongly detected (48 Sigma hits), but Kerberoasting goes unnoticed because the attack uses AES256 (Windows Server 2025) while the standard Sigma rule only watches for RC4. A custom Sigma rule closes this blind spot.

---

## Repository layout

```
operation-gryphon/
├── README.md                          # this file
├── 01-scoping/
│   └── scoping-note.pdf               # scope, tier, milestones
├── 02-build/
│   ├── lab-build-notes.md             # architecture + reproducibility
│   └── promote-domain.ps1             # domain build script
├── 03-attack/
│   └── realtime-journal.csv           # timestamped action journal
├── 04-defense/
│   ├── defensive-analysis.pdf         # ATT&CK matrix + findings
│   └── sigma/
│       └── kerberoasting_aes.yml      # custom Sigma rule
├── 05-report/
│   └── Gryphon_Report.docx            # narrative report (25 pages)
└── 06-screenshots/
    ├── Cap00-domain-structure.png
    ├── Cap01..Cap07 (attack)
    ├── Cap08..cap13 (defensive analysis)
    └── cap14, cap15 (BloodHound)
```

---

## Reproducing the lab

Full steps are in `02-build/lab-build-notes.md`. In short:

1. **Deploy the target**: Windows Server 2025 VM (official image), here on GCP. At the firewall, open only to the attacker's IP: RDP, SMB, LDAP, Kerberos, RPC (including the dynamic range 49152-65535) and DNS.
2. **Promote the domain**: run `promote-domain.ps1` (AD DS role, forest `meridian.local`), then part 2 of the script (OUs, accounts, SPN, attack paths, share).
3. **Verify** the build (domain, accounts, SPN, Domain Admins membership, share).
4. **Attack** from Kali by replaying the commands in the journal (`03-attack/realtime-journal.csv`).
5. **Analyze**: install Sysmon, export the logs, replay the Sigma rules with Chainsaw.

---

## Intentional attack paths

Weaknesses deliberately introduced at build time (realistic misconfigurations):

- `svc_sql`: SPN + weak password + member of Domain Admins → Kerberoasting then escalation.
- `mservice`: Kerberos pre-authentication disabled → AS-REP Roasting.
- `FinancesConf`: share readable by Administrator only → anomalous access is detectable.

---

## Deliverables (mapping to the assignment)

| Required deliverable | File |
|----------------------|------|
| Scoping note | `01-scoping/` |
| Narrative Red Team report | `05-report/Gryphon_Report.docx` |
| Annotated ATT&CK matrix | report §8.3 and `04-defense/defensive-analysis.pdf` |
| Detection recommendations (Sigma + IOCs) | report §9 and `04-defense/sigma/` |
| Real-time journal | `03-attack/realtime-journal.csv` |
| Lab build notes | `02-build/lab-build-notes.md` |
| Deferred defensive analysis | `04-defense/` and report §8 |

---

## Repository hygiene

Add a `.gitignore` that excludes any secret, and never commit real credentials:

```
*.kirbi
*.ccache
*secret*
*password*
kerb*.txt
```

Author: Marc BOHOUSSOU — 2026.
