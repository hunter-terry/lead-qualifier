# CLAUDE.md

## This project
One-liner: n8n workflow that scores incoming leads with local AI
(intent, urgency, 1-10 qualification score), validates the AI actually
returned usable structured data, and uses native n8n branching (Switch
node, not custom code) to route hot leads one way and cold/invalid
ones another. Second n8n portfolio piece — see CONTEXT.md for why this
one specifically and how it differs from inquiry-triage.

## How I work - read this first
- Plain language. No jargon. Explain technical words right after.
- Show me the plan before changing any files. I approve first.
- Test before handing me anything. Show proof it works.
- Preview first. Never delete or overwrite without a clear warning.
- Don't make things up. If unsure, say so.
- Done and shipped beats fancy.
