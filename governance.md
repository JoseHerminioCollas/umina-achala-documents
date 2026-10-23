# Governance & Security

## Boutique Shop Phase
Umiña Achala begins as a boutique shop owned and operated by the project founder.  
- The boutique acts as the sole buyer and seller of stones.  
- Provenance records originate at the boutique level, ensuring consistency.  
- Governance is simplified: all operational keys are held by the founder, secured in a vault.  
- Compliance checks (REINFO, RUC, HS Code, VUCE) are performed directly before minting.  
This phase establishes a controlled environment before expanding to multi‑vendor participation.

## Key Storage
- Keys should be secured in a vault or hardware security module (HSM).
- Access policies must enforce least privilege and audit logging.

## Pause/Wipe Usage
- Governance rules define when tokens can be paused or wiped.
- Pause Key: Used to temporarily halt transfers in case of compliance or security issues.
- Wipe Key: Used to remove tokens from accounts if regulatory violations occur.
- All actions must be logged in Hedera Consensus Service (HCS) for transparency.

## Multi-Signature Controls
- Sensitive roles (Admin, Metadata, Fee Schedule) consider multi-signature approval.

## Audit & Compliance
- Customs officers and buyers can query the HCS topic for a full lifecycle audit trail.
- Governance decisions must be documented and accessible for compliance checks.
