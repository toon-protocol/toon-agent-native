# toon-agent-native — Context

Where Agent-Native (Builder.io's open-source TypeScript agent framework) and TOON fit together, if anywhere. TOON's own vocabulary is in [toon-meta's glossary](https://github.com/toon-protocol/toon-meta/blob/main/context/glossary.md); this file adds only the terms this effort needs.

## Glossary

**Fit Verdict** — the written go/no-go this effort ends in: whether any Overlap is worth pursuing, ranked, with the strongest one or two named. Not a spec; a spec is a later, separate effort.

**Overlap** — one candidate way the two projects connect, judged by its Direction. *Avoid*: integration (implies it is already decided to build).

**Direction** — who the TOON user is in an Overlap:
- *Earning*: an Agent-Native app author is paid per Action call, with a connector in front of the app.
- *Paying*: an Agent-Native agent pays TOON apps (relay, store, gas-station, swap) through an agent node.
- *Reaching*: TOON gains reach into Agent-Native's agent-to-agent world (A2A, Dispatch) next to the NIP-90 job market.

**Action** — Agent-Native's term: one typed capability that the agent calls as a tool and the UI calls from code. Not a TOON term; TOON's unit of payment is the packet, not the Action.
