# intent-store

Agent-native development loop: developers state intent in a store, a swarm of
agents produces candidates, gates and selection narrow them, humans approve.
Prototype evidence: ~/personal-projects/swarm-replay (README.md).

## Agent skills

### Issue tracker

GitHub Issues on evangstav/intent-store, via `gh`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the root, created lazily. See `docs/agents/domain.md`.
