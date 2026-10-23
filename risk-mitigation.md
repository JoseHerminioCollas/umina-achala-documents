# Risk & Mitigation

Umiña Achala’s deployment on Hedera introduces both technical and regulatory risks. This section outlines anticipated issues and the strategies to mitigate them.

## Technical Risks
- **Key Compromise:** Private keys for Admin, Supply, Metadata, and Fee Schedule could be exposed.
  - *Mitigation:* Store keys in secure vaults; enforce multi‑sig for sensitive operations.
- **HCS Topic Overload:** A single master topic may become congested with thousands of events.
  - *Mitigation:* Monitor throughput; shard into multiple topics if latency exceeds thresholds.
- **IPFS Availability:** Images and scans may become unavailable if not pinned.
  - *Mitigation:* Use redundant pinning services and periodic availability checks.
- **Metadata Drift:** Mutable fields (export status, artisan RNA, timestamps) could diverge from compliance reality.
  - *Mitigation:* Require HCS anchoring for every update; enforce schema validation.

## Compliance Risks
- **REINFO Suspension:** Assets tied to concessions with expired or suspended REINFO become ineligible.
  - *Mitigation:* Automated “Regulatory Handshake” checks Vigente status before minting.
- **Payroll Non‑Compliance:** SUNAFIL inspections may reveal missing electronic payroll records.
  - *Mitigation:* Require miners to submit payroll proofs; block minting until verified.
- **Census Gate Failure:** Miners not registered in the INEI Census lose eligibility.
  - *Mitigation:* Census ID field mandatory in metadata; handshake validation enforced.
- **Cultural Heritage Misclassification:** Stones flagged as *Patrimonio Cultural* may slip through.
  - *Mitigation:* Integrate Ministerio de Cultura datasets; auto‑block Chapter 97 HS codes.

## Market Risks
- **Liquidity Shortfall:** Limited secondary market adoption could reduce token utility.
  - *Mitigation:* Partner with marketplaces; incentivize artisan onboarding.
- **Regulatory Shifts:** Future Peruvian laws may alter compliance requirements.
  - *Mitigation:* Governance council monitors legislation; updates metadata schema accordingly.
