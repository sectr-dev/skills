# Sectr agent skills

Agent skills for [Sectr](https://sectr.dev), a PaaS for hosting AI agents.
Give your coding agent (Claude Code, Codex, Cursor, and others) the Sectr
docs so it can build, run, and deploy Sectr agents with you.

## Install

```sh
npx skills add sectr-dev/skills
```

Add `-g` to install for your user instead of the current project, and
`--agent <name>` to pick an agent (repeat it for several), e.g.
`npx skills add sectr-dev/skills --agent claude-code`. Update later with
`npx skills update`.

## Skills

- [`sectr`](sectr/SKILL.md): the Sectr CLI, Python SDK and framework
  adapters, tool approvals, connecting a frontend over AG-UI, and the
  public HTTP API.
