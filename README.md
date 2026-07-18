# fertig.ai Agent Skills

Agent Skills for driving a [fertig.ai](https://fertig.ai) workspace through its **MCP** (Model Context Protocol) server. Load them into an MCP-capable AI assistant alongside the workspace MCP connection, and the assistant gets grounded, accurate guidance for managing your workspace with the `fertigai_*` tools.

## What's here

Each folder under `skills/` is one [Agent Skill](https://agentskills.io) (a `SKILL.md`).

| Skill | Use it for |
|---|---|
| `fertigai-mcp` | Index and shared conventions: endpoint, auth, ids, pagination, permissions, errors |
| `fertigai-agents` | Creating and configuring agents, branches, the conversation workflow, and setting groups |
| `fertigai-functions` | Custom JavaScript function tools that agents can call during a conversation |
| `fertigai-actions` | Automations that run in response to a conversation (for example post-call steps) |
| `fertigai-scripting` | The shared JavaScript environment for functions and actions: `run(ctx)`, built-in primitives, limits |
| `fertigai-ticket-templates` | Support-ticket field and status schemas |
| `fertigai-mail-templates` | Reusable HTML email templates |
| `fertigai-secrets` | Workspace secrets (write-only) |
| `fertigai-conversations` | Reading and searching conversation history |

`fertigai-agents` also ships a `workflow-reference.md` companion with the full workflow node/edge model and every setting-group field.

## Connecting to the MCP

The workspace MCP endpoint is:

```
https://<your-org-host>/api/v1/workspaces/<workspace-slug>/mcp
```

Authenticate with a workspace API key (`wsk_...`, created in workspace settings under API keys), sent as an `Authorization: Bearer <wsk_...>` header. Transport is MCP Streamable HTTP. See `skills/fertigai-mcp/SKILL.md` for the shared conventions and the full `fertigai_*` tool catalogue.

## Using the skills

1. Add the workspace MCP endpoint (URL + `wsk_` key) to your MCP client.
2. Make these skills available to the assistant (for example, copy the folders into your assistant's skills directory).

The `fertigai-mcp` index skill is the entry point; the assistant loads a domain skill when a task calls for it.
