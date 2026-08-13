# fertig.ai Agent Skills

Agent Skills for driving a [fertig.ai](https://fertig.ai) workspace through its **MCP** (Model Context Protocol) server. Load them into an MCP-capable AI assistant alongside the workspace MCP connection, and the assistant gets grounded, accurate guidance for managing your workspace with the `fertigai_*` tools.

## What's here

One [Agent Skill](https://agentskills.io): `skills/fertigai-mcp`. Its `SKILL.md` is the index (endpoint, auth, ids, pagination, permissions, errors, the full `fertigai_*` tool catalogue), and detailed per-domain guides are bundled under `references/`:

| Reference | Covers |
|---|---|
| `agents.md` (+ `agents-workflow.md`) | Creating and configuring agents, branches, the conversation workflow graph, and setting groups |
| `attachments.md` | Attaching functions, system tools, and knowledge bases to a branch or workflow node |
| `functions.md` | Custom JavaScript function tools that agents can call during a conversation |
| `actions.md` | Automations that run in response to a conversation (for example post-call steps) |
| `scripting.md` | The shared JavaScript/TypeScript environment for functions and actions: `run(ctx)`, built-in primitives, limits |
| `ticket-templates.md` | Support-ticket field and status schemas |
| `mail-templates.md` | Reusable HTML email templates |
| `secrets.md` | Workspace secrets (write-only) |
| `conversations.md` | Reading and searching conversation history |

## Connecting to the MCP

The workspace MCP endpoint is:

```
https://<your-org-host>/api/v1/workspaces/<workspace-slug>/mcp
```

Authenticate with a workspace API key (`wsk_...`, created in workspace settings under API keys), sent as an `Authorization: Bearer <wsk_...>` header. Transport is MCP Streamable HTTP. See `skills/fertigai-mcp/SKILL.md` for the shared conventions and the full `fertigai_*` tool catalogue.

## Using the skill

1. Add the workspace MCP endpoint (URL + `wsk_` key) to your MCP client.
2. Make the skill available to the assistant: copy (or upload) the single `skills/fertigai-mcp` folder, including its `references/` directory, into your assistant's skills location.

The assistant reads `SKILL.md` first and pulls in a bundled reference file when a task calls for it.
