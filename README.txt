# Integration of Governance In Dashboard

## Current Status
We are on the discovery / refinement phase

## Quick Links
- [Project Brief](01-context/project-brief.md)
- [Customer Interviews](02-discovery/Interviews.md) 
- [Epics & Features](03-delivery/Epics.md)
<<<<<<< HEAD

## This Week's Focus


## Key Decisions Made
- Drop the standalone Governance PoC and integrate Governance natively into Tyk Dashboard
- Use Vacuum as the linting engine (Spectral-compatible rulesets, faster on large specs)
- v1 never blocks deployments. Per-ruleset "Deployment Warning" toggle is OFF by default — non-compliant APIs deploy silently and just show as "non-compliant". Turning it ON adds a soft warning on Save with "Deploy Anyway" or "Go Back"
- Governance ships as an add-on entitlement with Read / Write / Deny RBAC
- Rulesets applied to APIs via Categories
- v1 scope: OAS APIs + MCP. GraphQL, AsyncAPI, shadow/redundant API detection are post-v1

## Questions/Blockers
- API contract mismatch: FE ticket TT-17023 references `/api/governance/rulesets` while BE tickets TT-17019–22 use `/api/rulesets` with different param shapes — needs reconciling before parallel work
- TT-17024 (Create UI) and TT-17025 (View/Edit/Delete UI) still empty stubs
- Vacuum fork decision pending (TT-17026) — keep the fork or move to upstream v0.26.1?
- Compliance score formula not yet defined (Epic 3 open question)
- Real ruleset templates needed — current OWASP + Recommended are placeholder examples; validate with customers
- Should customers be able to opt-in to blocking on non-compliance, or stay warn-only?
- Northwestern Mutual & Barclays asked for read-only shareable dashboard snapshots for federated teams — in or out of v1?
=======
>>>>>>> a5da9556da28622821b7d1cf8ec98ac2cfc8fbfd
