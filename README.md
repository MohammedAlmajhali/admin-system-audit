# admin-system-audit

A skill for Claude Code, Codex, Cursor, and Google Antigravity that audits an existing admin dashboard against the system it actually runs. It reads the backend, the roles, the workflows, and the rendered screens, then writes the reports before anyone changes code.

Created by Mohammed Khaled Saleh AL-Majhali ([@MohammedAlmajhali](https://github.com/MohammedAlmajhali)).

## Install

For Claude Code, install it as a plugin. Run this in a terminal to get it in every project:

```bash
claude plugin install admin-system-audit --marketplace MohammedAlmajhali/admin-system-audit
```

On an older Claude Code without `--marketplace`, the same in two steps:

```bash
claude plugin marketplace add MohammedAlmajhali/admin-system-audit
claude plugin install admin-system-audit@MohammedAlmajhali
```

To get it in a single project only, run the install from that project's root folder and add `--scope local`. To turn it on for everyone who works in the project, add `--scope project` instead and commit `.claude/settings.json`. Each teammate then runs this once in their own copy of the project:

```bash
claude plugin marketplace add MohammedAlmajhali/admin-system-audit
claude plugin install admin-system-audit@MohammedAlmajhali --scope project
```

Then start a new Claude Code session, or run `/reload-plugins` in one that is already open.

To update, run `claude plugin update admin-system-audit@MohammedAlmajhali`.

For Codex, copy the skill into `~/.agents/skills/`, or into `.agents/skills/` inside a single project.

```bash
git clone https://github.com/MohammedAlmajhali/admin-system-audit
mkdir -p ~/.agents/skills && cp -r admin-system-audit/skills/admin-system-audit ~/.agents/skills/
```

For Cursor, copy it into your personal skills folder so it is available in every project. Start a new agent chat after copying.

```bash
git clone https://github.com/MohammedAlmajhali/admin-system-audit
mkdir -p ~/.cursor/skills && cp -r admin-system-audit/skills/admin-system-audit ~/.cursor/skills/
```

For a single Cursor project, copy it into `.cursor/skills/` inside that project instead.

For Google Antigravity, copy the skill into your skills folder.

```bash
git clone https://github.com/MohammedAlmajhali/admin-system-audit
mkdir -p ~/.gemini/config/skills && cp -r admin-system-audit/skills/admin-system-audit ~/.gemini/config/skills/
```

## Use

In Claude Code, type `/admin-system-audit` with what you want. In Codex, type `$admin-system-audit` the same way. In Cursor, ask in a new agent chat. Or just ask in your own words and the agent picks the skill up by itself.

```
audit the admin dashboard
admin system audit for this repo, report only
```

Phases 1–5 never change application code. They write five reports under `docs/admin-audit/` and stop for approval. Phase 6, the code changes, starts only after you name the tasks you approve. If you say "audit only" or "report only", it stops after the reports.

## What it does

- **Discovery.** Maps the stack, endpoints, roles, and the business workflows the dashboard is supposed to run.
- **Coverage.** Builds a traceability matrix of each business capability against the admin UI.
- **Usability.** Reviews navigation, names, task flows, and the rendered screens, including Arabic and RTL when the product needs them.
- **Correctness.** Runs the tests and walks the workflows, and marks anything that was not run.
- **Plan.** Writes a blueprint and an implementation plan, then waits. Code changes happen only after explicit approval.

## License

[CC BY 4.0](LICENSE). Use it, change it, and share it, including commercially, as long as you credit Mohammed Khaled Saleh AL-Majhali ([@MohammedAlmajhali](https://github.com/MohammedAlmajhali)) and link back to this repository.
