# CONTEXT - lead-qualifier
(living "where I'm at" note - keep updating this as you go)

## One-liner
Second n8n portfolio piece, for GitHub/resume/Upwork. Not started yet —
this file IS the approved plan, ready to build in a fresh session.

## Why this project (not just "another n8n workflow")
`inquiry-triage` (github.com/hunter-terry/inquiry-triage) leans almost
entirely on Code nodes for its logic, which undercuts n8n's own selling
point as a visual builder. This project is meant to round out the
story for employers/freelance clients:
- Uses n8n's native Schedule Trigger and Switch/IF branching instead of
  JavaScript-in-a-Code-node for the routing logic.
- Different AI task: structured extraction + scoring (a real number,
  validated as actually a number 1-10) instead of free-text drafting.
  Lead qualification/routing is a commonly-requested SMB automation.
- Realistic without needing real accounts: polls a local "leads inbox"
  folder (stands in for a CRM export / form-to-file integration) rather
  than a live webhook - fully demoable, no real Slack/CRM/Google
  credentials needed. README notes "swap this node for your CRM
  connector."

## Intake brief (approved 2026-07-11, not yet built)
MODE: PERSONAL - portfolio piece, same as hub/listit/auto-summary/inquiry-triage.

WANT: An n8n workflow that scores incoming leads with local AI (intent,
urgency, a 1-10 qualification score), validates the AI actually
returned usable structured data, and uses native n8n branching to route
hot leads one way and cold/invalid ones another.

DONE LOOKS LIKE: Schedule Trigger polls a local "leads inbox" folder ->
Ollama extracts score/urgency/summary -> a validation step rejects
anything where the score isn't a real number 1-10 or required fields
are missing -> an n8n Switch node (visual, in the canvas) routes hot
leads to one output and everything else to another -> tested with real
synthetic leads including at least one that should get rejected by
validation, same discipline as inquiry-triage (see its
test-inquiries/test-results.md for the pattern to follow).

CONSTRAINTS: No real CRM/Slack/email accounts - local files stand in
for those integrations. Local AI only (Ollama, model per
RAM - see inquiry-triage's README for the model-selection table to
reuse). Free stack only.

WORK RULES: Same engineering standard as every other project - preview
before action, validate AI output before it moves anywhere, real
behavioral tests (actually run it, not just read the code) per
playbooks/testing.md.

FIRST MOVE: Scaffold done (this folder). Next: build the Schedule
Trigger -> Ollama scoring step first, before adding branching. Follow
the same n8n build pattern as inquiry-triage: hand-author workflow.json,
`n8n import:workflow`, `n8n update:workflow --active=true`, restart,
test via CLI/PowerShell - see playbooks/fixes-log.md for the n8n
gotchas already solved (IPv4 not localhost for Ollama, Code node
sandbox needs NODE_FUNCTION_ALLOW_BUILTIN, workflow needs an explicit
id, no `process` global in Code nodes, restart required after every
CLI-side edit).

## Where I'm at right now (2026-07-11)
Built, tested, and pushed public:
[github.com/hunter-terry/lead-qualifier](https://github.com/hunter-terry/lead-qualifier). Workflow runs: Schedule Trigger -> Read Lead Files
(native) -> Extract Lead Data (native) -> Ollama Score Lead -> Parse &
Validate -> Switch: Route Lead (native, 3-way + fallback) -> hot-leads/
cold-leads/ flagged/. Polls `~/lead-qualifier-data/inbox/` every 1
minute when n8n + Ollama are running.

6 real test cases run (see `test-leads/test-results.md`): clean hot
lead, clean cold lead, invalid (missing required fields), invalid
(malformed non-JSON file), Ollama unreachable (stopped the process
mid-test), and a filename-collision re-submission (proved the archive
step's "never silently overwrite" handling works). All routed
correctly; none crashed the workflow, nothing was overwritten.

Two real things found and fixed through actual testing, not assumed:
- n8n 2.29's default `N8N_RESTRICT_FILE_ACCESS_TO` blocked the native
  file-read node from reaching `~/lead-qualifier-data` — inquiry-triage
  never hit this since it only used Code-node `fs`. Fixed by explicitly
  allowlisting the folder; documented in the vault's fixes-log.md.
- On both invalid test leads, Ollama didn't refuse — it invented a
  plausible score/summary for garbage/empty input anyway. Validation
  caught both regardless. Prompted a follow-up tightening of the Ollama
  call itself (capped input length, temperature 0, num_predict cap) per
  Hunter's guidance mid-build; re-tested afterward, same clean result.

## Where I'm at right now (update, 2026-07-12)
Built `docs/walkthrough.html` — a plain-language, client-facing demo
page for non-technical Upwork clients. Two real runs captured live
against this workflow (a qualified hot lead scored 9/10; a blank/broken
lead that Ollama still confidently scored until validation caught it
and routed it to flagged). Published as a shareable Claude Artifact,
linked from the vault's `job-search/upwork-profile.md`.

## Steps / plan
- ~~Build Schedule Trigger -> Ollama scoring node, prove the round-trip~~ — done
- ~~Add validation (score is a real number 1-10, required fields present)~~ — done
- ~~Add native Switch node branching (hot vs cold/invalid), not Code-node logic~~ —
  done, 3-way (hot/cold/invalid) plus a fallback output, not just 2-way
- ~~Test with synthetic leads, including one designed to fail validation~~ —
  done, 5 cases including two different validation-failure shapes and
  one Ollama-unreachable case
- Webhook auth not applicable (Schedule Trigger, not webhook-triggered);
  re-checked playbooks/automation-platforms.md rules, nothing else applies
- ~~Export workflow.json, README (client-facing, same tone as inquiry-triage's),
  test-inquiries equivalent folder with real results~~ — done
- ~~QA checklist (standards/qa-publish-checklist.md), push to new public
  GitHub repo~~ — done, 2026-07-11:
  [github.com/hunter-terry/lead-qualifier](https://github.com/hunter-terry/lead-qualifier)

## Where I'm at right now (update, 2026-07-13)
A demo-recording attempt for Terry Studio Goal 1 was made and scrapped
(see vault root CONTEXT.md / `2-work/master-plan-status.md` /
`playbooks/demo-recording.md`) — the `demo/` and `demo-leads/` folders
that existed for it have been deleted, they were scratch work, not
shipped project content. `test-leads/` (the original QA fixtures) is
untouched. Two real, still-valid env fixes came out of that session,
now required every time n8n starts: `NODE_FUNCTION_ALLOW_BUILTIN=fs,path,os`
(n8n 2.29.10 blocks bare `fs`/`path`/`os` in Code nodes by default) and
`N8N_RESTRICT_FILE_ACCESS_TO` must include this project's
`lead-qualifier-data` path.

## Notes
- Reuse the RAM-based model-selection table pattern from the other
  three repos' READMEs.
- Reuse the "model not pulled" vs "Ollama unreachable" distinct-error
  pattern already built into inquiry-triage's Parse & Validate node.
