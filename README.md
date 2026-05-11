# Easy8 MCP Server

Connect your AI assistant directly to Easy8. The Easy8 MCP server lets Claude, Cursor, GitHub Copilot, and any other Model Context Protocol client read and update your projects, issues, and comments — using your own Easy8 permissions, against your own Easy8 instance.

The server is built into Easy8. There is nothing to install, host, or maintain — you enable it in admin settings and point your AI client at `https://your-easy8.example.com/mcp`.

> **Status:** early access. The protocol surface and tool catalog will expand in future Easy8 releases.

## Supported clients

The Easy8 MCP server speaks the standard Model Context Protocol over Streamable HTTP, so any compliant client works. Verified setup instructions below are provided for:

- **Claude Code** — see [Claude Code setup](#claude-code).
- **Cursor** — see [Cursor setup](#cursor).
- **Visual Studio Code (GitHub Copilot)** — see [VS Code setup](#vs-code-github-copilot).
- **Claude Desktop and other stdio-only clients** — via the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) proxy, see [stdio clients setup](#claude-desktop-and-other-stdio-only-clients).

Any other MCP client that supports remote HTTP servers with custom request headers will work out of the box — point it at `https://your-easy8.example.com/mcp` and send your Easy8 API key in the `X-Redmine-API-Key` header.

## Before you start

You need three things:

1. **An Easy8 instance** you can reach from your AI client (cloud or self-hosted).
2. **MCP enabled** in Easy8 administration: *Settings → AI → Enable MCP → Save*. MCP is disabled by default. While disabled, `/mcp` returns `404 Not Found`.
3. **A personal Easy8 API key.** Generate it from your Easy8 user profile. The MCP server uses standard Easy8 API authentication and inherits all of your account's permissions and project visibility.

That's it. There is no separate install, no extra service to run, no OAuth app to register.

## Endpoint and discovery

| | |
|---|---|
| MCP endpoint | `https://your-easy8.example.com/mcp` |
| Transport | Streamable HTTP |
| Authentication | API key in the `X-Redmine-API-Key` request header |
| Discovery card | `https://your-easy8.example.com/.well-known/mcp/server-card.json` |

The discovery card is public metadata advertising the transport and capabilities. The `/mcp` endpoint itself still requires MCP to be enabled and a valid API key.

## What the AI can do

The current tool catalog focuses on issue and project workflows.

| Tool | What it does |
|---|---|
| `easy8_projects_list` | List visible projects. Filter by name or free-text query, paginate with `limit` / `offset`. |
| `easy8_issues_list` | List visible issues. Filter by project, assignee, status (`open` / `closed`), subject, due-date period, or free-text query. |
| `easy8_issues_get` | Fetch one issue by ID, including metadata, description, and comment count. |
| `easy8_issues_comments_list` | List issue journal notes, newest first. |
| `easy8_issues_create` | Create an issue. Subject and a project selector are required. Supports tracker, description, assignee, priority, status, dates, and custom fields. |
| `easy8_issues_update` | Update an existing issue. Supports field edits and appending a new journal note via `notes`. |

Tools are discovered dynamically via the standard MCP `tools/list` method, so your client always sees the live, up-to-date schema.

### Example workflows

- *"Find all open issues in the Q2 marketing project assigned to me and summarize what's blocked."*
- *"Read the latest comments on issue #1842 and draft a status update for the customer."*
- *"Create a follow-up task in the Onboarding project, assign it to Petra, due Friday, with the right tracker and priority."*
- *"List every issue in the Helpdesk project closed last week and group them by tracker."*

## Client setup

Replace `https://your-easy8.example.com` with your Easy8 URL and `YOUR_API_KEY` with your personal Easy8 API key.

### Claude Code

```bash
claude mcp add --transport http easy8 https://your-easy8.example.com/mcp \
  --header "X-Redmine-API-Key: YOUR_API_KEY"
```

### Cursor

Add to `~/.cursor/mcp.json` (or your project's `.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "easy8": {
      "url": "https://your-easy8.example.com/mcp",
      "headers": {
        "X-Redmine-API-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### VS Code (GitHub Copilot)

Add to your workspace's `.vscode/mcp.json`:

```json
{
  "servers": {
    "easy8": {
      "type": "http",
      "url": "https://your-easy8.example.com/mcp",
      "headers": {
        "X-Redmine-API-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### Claude Desktop (and other stdio-only clients)

Claude Desktop does not yet speak remote HTTP MCP natively. Use the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) proxy. Edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "easy8": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://your-easy8.example.com/mcp",
        "--header",
        "X-Redmine-API-Key: YOUR_API_KEY"
      ]
    }
  }
}
```

Restart the client after editing the config.

## How it works

The Easy8 MCP server implements the standard JSON-RPC methods of the Model Context Protocol:

- `initialize` — opens a session and returns server metadata and capabilities.
- `tools/list` — returns the live tool catalog with full JSON schemas.
- `tools/call` — invokes a tool with structured arguments and returns a structured response.
- `ping` — health check.
- `notifications/initialized` — accepted as the standard post-initialization notification from the client.

Full method-level reference, request/response examples, and error semantics live in the [Easy8 developer documentation](https://developers.easy8.com/AI/mcp-server/).

## Security and permissions

- **Your permissions, your data.** Every tool call runs with the permissions of the API key owner. The MCP server does not bypass Easy8 authorization, visibility rules, or required-field validation.
- **TLS in transit.** Always use HTTPS for `/mcp`.
- **Treat API keys like passwords.** Don't commit them to source control or share them in chat. Rotate from your Easy8 user profile if leaked.
- **Off by default.** Easy8 administrators enable MCP explicitly per instance.
- **Use a dedicated service account.** Create an Easy8 user with only the permissions the integration actually needs (often read-only) and generate the API key from that account instead of from a personal admin account.

## Troubleshooting

**`/mcp` returns `404 Not Found`**
MCP is not enabled. Go to *Settings → AI → Enable MCP* in Easy8 administration and save.

**`POST /mcp` returns `401 Unauthorized`**
The request is missing a valid API key. Check that the `X-Redmine-API-Key` header is present and that the key has not been revoked or regenerated.

**Tool call returns a permission error**
The MCP server enforces Easy8 authorization. Check the permissions of the API key owner on the project or issue in question.

**`GET /mcp` returns `405 Method Not Allowed`**
Expected. The endpoint is JSON-RPC over POST; GET is not supported when MCP is enabled.

## Support and feedback

- Product questions and feature requests: [support@easy8.com](mailto:support@easy8.com)
- Developer reference: [developers.easy8.com/AI/mcp-server](https://developers.easy8.com/AI/mcp-server/)
- Issues with this README or the configuration snippets: open an issue in this repository.

## Disclaimer

The Easy8 MCP server is provided as part of Easy8 under your existing Easy8 license and terms. Tool behavior, available methods, and the protocol version may change between Easy8 releases. Always test against your own non-production instance before granting an AI client write access to production data.
