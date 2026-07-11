# lead-qualifier

A self-hosted n8n workflow that scores incoming sales leads with local
AI (intent, urgency, a 1-10 qualification score), checks that the AI
actually returned usable structured data, and uses n8n's native Switch
node — not custom code — to route hot leads one way and cold/invalid
ones another.

Second n8n portfolio piece, alongside
[`inquiry-triage`](https://github.com/hunter-terry/inquiry-triage).
Same "never trust the raw AI reply" pattern, different shape: this one
leans on n8n's own visual building blocks (Schedule Trigger, native
file nodes, Switch) instead of Code nodes doing all the work.

Everything runs locally: n8n self-hosted, AI via
[Ollama](https://ollama.com) on-device. No lead data reaches a cloud AI
provider. Nothing routes anywhere real automatically — this polls a
local "leads inbox" folder that stands in for a CRM export or a
form-to-file integration, so it's fully demoable with no real CRM,
Slack, or email credentials. Swap the file nodes for your CRM
connector to make it real.

## What it does

1. **Schedule Trigger** polls on an interval (demoed at 1 minute; set
   it to whatever fits your real inbox).
2. **Read/Write Files from Disk** (native n8n node) reads every lead
   file waiting in the inbox folder.
3. **Extract from File** (native n8n node) parses each file's JSON
   into structured fields — no Code node needed for this step.
4. **Ollama Score Lead** — a local model (`llama3.1:8b`) reads the
   lead's name/company/source/message and returns an intent summary,
   an urgency level, and a 1-10 qualification score.
5. **Parse & Validate** — the model's reply is checked before it goes
   anywhere:
   - SCORE must be a real whole number from 1-10
   - URGENCY must be low, medium, or high
   - INTENT and SUMMARY must actually be present
   - The lead itself must have a name and a message (rejects
     malformed/empty CRM exports)
   - Rejects cleanly (not a crash) if Ollama is unreachable, and gives
     a distinct message if the model just isn't pulled yet
6. **Switch: Route Lead** (native n8n Switch node, not an if/else in
   Code) routes on a single `route` field: `hot`, `cold`, or `invalid`,
   with a built-in fallback output for anything unexpected.
7. Passing leads land in `hot-leads/` or `cold-leads/`; anything that
   failed validation lands in `flagged/` with the specific reason(s).
   The original inbox file is archived to `processed/` either way, so
   the next poll doesn't reprocess it.

```
Schedule Trigger --> Read Lead Files --> Extract Lead Data --> Ollama Score Lead
     (poll)          (native, disk)      (native, JSON)         (local AI call)
                                                                        |
                                                                        v
                                                              Parse & Validate
                                                               (rule checks)
                                                                        |
                                                                        v
                                                            Switch: Route Lead
                                                          (native, 3-way + fallback)
                                                          /        |         \
                                                    hot-leads/  cold-leads/  flagged/
```

The full workflow is in [`workflow.json`](workflow.json) — importable
directly into n8n.

## Proven, not assumed

Tested against 6 real cases — see
[`test-leads/test-results.md`](test-leads/test-results.md) for the
full record, including a real bug found and fixed (n8n's newer
file-access restriction blocking the native file-read node) and a
prompt tightening made after watching the model's actual behavior on
edge cases.

The standout test: two deliberately bad lead files (one with blank
required fields, one that wasn't even valid JSON) both got the local
model to produce a confident-looking score and summary anyway — it
didn't refuse. Validation caught both and routed them to `flagged/`
regardless of what the model said. That's the same lesson
inquiry-triage's prompt-injection test proved: the model's compliance
is never the safety net, the check downstream is.

## How to run it

Requirements: [n8n](https://n8n.io) (self-hosted, no account needed),
[Ollama](https://ollama.com) with a model pulled.

### Choosing a model for your machine

The workflow defaults to `llama3.1:8b`. Same rule of thumb as
inquiry-triage:

| Total RAM | Suggested model |
|---|---|
| ~8GB | `llama3.2:3b` |
| ~16GB | `llama3.1:8b` (default) |
| 32GB+ | a larger model, e.g. `llama3.1:70b` or `mixtral` |

To use a different model, edit the `model` field inside the "Ollama
Score Lead" node's JSON body.

```
ollama pull llama3.1:8b
```

The Code nodes (validation + save/archive) need Node's `fs`, `path`,
and `os` built-ins — n8n blocks these by default, so allow them before
starting. This n8n version also defaults to restricting the native
file nodes to `~/.n8n-files` only, so the leads-data folder needs to be
explicitly allowlisted too:

Windows (PowerShell):
```
$env:NODE_FUNCTION_ALLOW_BUILTIN = "fs,path,os"
$env:N8N_RESTRICT_FILE_ACCESS_TO = "~/.n8n-files;~/lead-qualifier-data"
npx n8n
```

macOS/Linux:
```
NODE_FUNCTION_ALLOW_BUILTIN=fs,path,os N8N_RESTRICT_FILE_ACCESS_TO="~/.n8n-files;~/lead-qualifier-data" npx n8n
```

Then, with n8n running:
```
npx n8n import:workflow --input=workflow.json
npx n8n update:workflow --id=lead-qualifier-001 --active=true
```
(Restart n8n once after activating — CLI-side workflow changes need a
restart to take effect. Editing in the n8n UI instead doesn't have this
limitation.)

All data lives under `~/lead-qualifier-data/`, resolved from the home
directory at runtime — no username or machine-specific path is ever
baked into the workflow:

- `inbox/` — drop lead files here (JSON: `name`, `company`, `source`,
  `message`) to simulate a CRM export landing
- `hot-leads/` / `cold-leads/` — passing leads, split by score
- `flagged/` — anything that failed validation, with reasons
- `processed/` — archived copies of every inbox file once handled, so
  the poller doesn't reprocess it

Try it — copy one of the files from `test-leads/` into the inbox and
wait for the next poll:
```
cp test-leads/lead-hot-01.json ~/lead-qualifier-data/inbox/
```

## Safety guarantees

- Local AI only — no lead data leaves the machine.
- Nothing auto-sends or auto-imports anywhere real. Every output is a
  local file for a human (or the next integration) to pick up.
- Every AI output is checked against hard rules before it moves on —
  the model's compliance with the requested format is never assumed.
- Failures (bad input, AI unreachable, rule violations) are logged
  with a specific, plain-language reason and routed to `flagged/` —
  never silently dropped, never crash the workflow.
- No real CRM/Slack/email credentials are used or required.

## Folders

- `workflow.json` — the exported n8n workflow (the actual deliverable)
- `test-leads/` — the real test cases and results
- `docs/` — architecture notes (if added)
- Runtime output (`hot-leads/`, `cold-leads/`, `flagged/`,
  `processed/`, `inbox/`) lives under `~/lead-qualifier-data/`, outside
  this repo — nothing runtime is shipped with the repo.
