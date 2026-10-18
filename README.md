# Forte for coding agents

Connects Claude, ChatGPT, Codex, Cursor and VS Code to your Forte account, and keeps your agent accurate on Forte's CLI, SDK, auth, and deployment workflows. Your agent uses it automatically when you work with Forte.

## Connect Forte to your AI tools

Forte runs a remote MCP server at `https://mcp.forteplatforms.com/mcp`. Once connected, your assistant can create services, websites and databases, check deploys, read build logs, request logs and metrics, look up costs, connect services, websites and databases to each other, and redeploy. The first time it calls a Forte tool, your browser opens a Forte sign-in and consent screen where you pick what it can do and which projects it can see. You can disconnect it anytime under Organization → Connected apps in the Forte console.

- **Claude (claude.ai, desktop):** [add Forte as a connector](https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=Forte&connectorUrl=https%3A%2F%2Fmcp.forteplatforms.com%2Fmcp).
- **Claude Code:**

  ```bash
  claude mcp add --transport http forte https://mcp.forteplatforms.com/mcp
  ```

- **Codex:**

  ```bash
  codex mcp add forte --url https://mcp.forteplatforms.com/mcp
  codex mcp login forte
  ```

- **ChatGPT:** go to chatgpt.com/plugins, select **+**, then **Add custom MCP server**, and paste `https://mcp.forteplatforms.com/mcp`. In a chat, type `@` and pick Forte.
- **Cursor:** [install the Forte MCP server](https://cursor.com/install-mcp?name=forte&config=eyJ1cmwiOiJodHRwczovL21jcC5mb3J0ZXBsYXRmb3Jtcy5jb20vbWNwIn0%3D).
- **VS Code:** [install the Forte MCP server](https://insiders.vscode.dev/redirect/mcp/install?name=forte&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fmcp.forteplatforms.com%2Fmcp%22%7D).

Installing the plugin below also connects the MCP server.

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
