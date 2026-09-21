# Forte for coding agents

Keeps Claude Code, Codex, and Cursor accurate on Forte's CLI, SDK, auth, and deployment workflows. Your agent uses it automatically when you work with Forte.

## Claude Code

```
/plugin marketplace add forteplatforms/claude-plugin
/plugin install forte@forteplatforms
```

## Codex

```bash
codex plugin marketplace add forteplatforms/claude-plugin
```

Then run `/plugins` in Codex, install **Forte**, and start a new session.

## Cursor

```bash
npx skills@1 add forteplatforms/claude-plugin -a cursor
```

## Any other agent

Run from the root of your repository, then commit the generated `.agents/skills/forte/` directory:

```bash
curl -fsSL https://forteplatforms.com/claude-skill/install.sh | bash -s -- agents
```
