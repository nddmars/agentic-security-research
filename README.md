# CxSAST AI Triage: Design Pack (v2)

Author: Surya

Start with `00-MAIN-DESIGN.md` (index, shared contracts, decisions). Then read the feature file you are implementing.

## Reading and build order
1. `00-MAIN-DESIGN.md`: scope, contracts (finding record, Evidence Pack, verdict schema), hard rules.
2. `14-rollout-plan.md` section 8: suggested build sequence.
3. Feature files in build order: 01, 02, 03, 06, 10, 07 (+05), 08 (+11), 09, 15, 04, with 12 and 13 in parallel.

## What changed in v2
- Joern is the only code-analysis engine; ripgrep and path rules fill gaps; no tree-sitter in v1 (optional phase-2 fallback by measured data).
- 03 builds an **Evidence Pack** with citable snippet IDs; 07 and 08 work only from it.
- 07 is **facts-first** with decision-table playbooks; 08 derives the allowed verdict from the facts and can only downgrade.
- 02 anchors branch resolution at scan start and computes a per-finding `source_match_score`.
- 15 triages Checkmarx pattern-only findings (no path) with secret masking. Checkmarx already holds the regex rules, so there is no independent pattern scan.
- 13 targets OpenShift (random non-root UID, `restricted-v2` SCC, NetworkPolicy, Routes).
- 01, 04, 06, 09, 10, 11 and 12 were brought in line with the above.

## Files in this folder
`00` main design, `01` ingestion, `02` source acquisition, `03` code intelligence, `04` framework detection, `05` sanitizer KB, `06` orchestrator, `07` triage agent, `08` validator and policy, `09` Checkmarx writeback, `10` dashboard and metrics, `11` evaluation harness, `12` security and isolation, `13` infrastructure (OpenShift), `14` rollout plan, `15` pattern findings triage (Checkmarx only), `17` phase 2 Git mapping and history, `16` modules, functions and implementation libraries (the code layout index), plus this README.

## Open items before coding
- Source is archive-first: the scan comment holds a build URL for a zip. Git mapping and history are phase 2 (17).
- CxSAST 9.7 API spike (update endpoint, assignment, notes append and length).
- Decided: the central CISO analyst team is the reviewer queue; the agent only sets Proposed Not Exploitable, otherwise comment only.
- Golden set comes from a Checkmarx XML report (Reports UI). **The coding agent reads the sample XML export and matching zips first**, builds the parser against them, and verifies `XmlReportSourceLinesRange` and `AddQueryMetaDataToXmlReport` on the 9.7 server.
- SSO and hosting standards for the dashboard on OpenShift.
- Analysts must verify sanitizer KB seed entries against official documentation before enabling.
