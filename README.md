# Apex Agent Pipeline — Claude Code Implementation
A working subagent pipeline for Claude Code, built from your Apex spec. This is real and runnable, with the fictional parts of the original JSON (message bus, 1000-cycle autonomous loop, self-reported 95+ scores) replaced with mechanisms that actually exist.
Setup
Copy .claude/ and queue/ into your project root.
Merge .claude/settings.snippet.json into your .claude/settings.json.
Install jq (apt install jq / brew install jq) — the gate hook needs it to actually enforce gating. Without it, the hook logs a warning and allows everything through (fails open, not closed — check the hook output).
Restart your Claude Code session so it picks up the new agents in .claude/agents/.
Running a feature through the pipeline
Set a feature slug for the session so the gate hook can track it:
export APEX_FEATURE_SLUG="billing-tier-upgrade"
Then in Claude Code, drive it stage by stage. This is NOT fully autonomous — you are the CEO/orchestrator, reading each stage's summary and deciding whether to proceed. That's a deliberate design choice, not a limitation to work around:
Use the pm agent to spec: "add a subscription billing tier"
[read the PRD it produces]

Use the design agent on docs/prd/billing-tier-upgrade.md
[read the design doc]

Use the frontend agent and the backend agent on the above (can run in parallel — invoke both in one message and Claude Code will parallelize)

Use the database agent if backend flagged schema needs

Use the security agent to audit the billing-tier-upgrade changes
[if FAIL, it names which agent owns each fix — go back to that agent]

Use the qa agent to test billing-tier-upgrade
[if FAIL, same — go back to owning agent]

Use the devops agent to deploy billing-tier-upgrade to staging
The gate hook will hard-block security/qa/devops from running out of order if you (or Claude) try to skip a stage — it exits with code 2 and Claude sees the block reason.
What this actually does vs. the original spec
Original spec claimed
What's real here
Agents communicate via message schema
Agents share a JSON state file; there is no live agent-to-agent channel. The parent (you) routes results.
Autonomous loop, max_cycles 1000
Does not exist. Each stage requires a human (or a script you write) to trigger the next invocation. Building a real unattended loop means writing an external script that calls claude -p headlessly and checks pipeline_state.json — not included here, ask if you want it.
Security/performance/accessibility scores ≥95
The security and qa agents are explicitly instructed to never fabricate a score — they report "not measured" if no real tool produced the number. Wire in real tools (Lighthouse CI, semgrep, npm audit) for actual scores.
8 agents running in a coordinated org chart
8 subagent definitions exist and work, but Claude Code's own guidance is that 3-4 active roles per pipeline is the practical ceiling before coordination overhead and token cost outweigh the benefit. Use frontend/backend/database in parallel, but don't expect the full 8 to run concurrently and converge cleanly.
Known gaps you'd need to fill for production use
No CEO agent file — deliberately. The orchestrating role is you, in the main session, because subagents can't currently spawn or gate other subagents themselves in a loop; only the parent conversation can sequence them. Making a "ceo.md" subagent wouldn't gain you anything Claude Code's main session doesn't already do.
gate.sh uses grep/sed fallbacks if jq is missing — this is not robust, install jq.
No real metrics tooling wired in — you'll want to add actual npx lighthouse, semgrep --config=auto, npm audit --json calls inside the qa/security agents' Bash usage once you tell me what stack the target project actually uses.
No unattended loop — say the word if you want the headless-script version; it's a different piece (a bash/python driver script, not a Claude Code primitive) and changes the safety profile since it'd run without a human checkpoint per stage.
File map
.claude/agents/pm.md          — Product Manager (PRD, user stories)
.claude/agents/design.md      — UI/UX (design system, wireframe specs)
.claude/agents/frontend.md    — Next.js/React/TS implementation
.claude/agents/backend.md     — API/business logic implementation
.claude/agents/database.md    — Schema/migrations
.claude/agents/security.md    — Audits only, reports findings, doesn't fix
.claude/agents/qa.md          — Runs real tests, reports real results
.claude/agents/devops.md      — CI/CD/deploy, gated on security+qa PASS
.claude/hooks/gate.sh         — Enforces stage ordering
.claude/settings.snippet.json — Hook wiring to merge into your settings.json
queue/pipeline_state.json     — Shared state / audit trail
