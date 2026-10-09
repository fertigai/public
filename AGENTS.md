# public - agent guide

Public Agent Skills for driving a fertig.ai workspace through its MCP server. Markdown only: no build, no CI, no deploy. Everything here is read by customers.

## Layout

- `skills/fertigai-mcp/SKILL.md` the skill entry: frontmatter (`name`, `description`), connection and auth, `fertigai_whoami`, the `fertigai_*` tool catalogue table, shared conventions.
- `skills/fertigai-mcp/references/*.md` per-domain guides, one row each in the SKILL.md catalogue table.
- `README.md` repo overview with a reference table and connection instructions.

## Editing

- Document only customer-visible behaviour: tool names, arguments, fields, limits, errors and workflows as a workspace user sees them. No internal implementation details, service names, infrastructure, environment variables or issue-tracker references in the skill text.
- Keep `SKILL.md` lean; depth goes into the matching `references/<domain>.md`. When a tool is added, removed or renamed, update the catalogue table in `SKILL.md` and the reference that covers it.
- A new reference file needs a row in the `SKILL.md` table and in the `README.md` table. Keep file names stable: `SKILL.md` points readers at a hosted copy by file name.
- Keep the frontmatter `name` equal to the folder name and the `description` a single "Use when ..." sentence.

## Validate

No automated checks. Before pushing:

- Every file under `references/` is named in `SKILL.md`: `for f in skills/fertigai-mcp/references/*.md; do grep -q "$(basename "$f")" skills/fertigai-mcp/SKILL.md || echo "missing: $f"; done` prints nothing.
- Every `fertigai_*` name you wrote matches a real tool, and examples (JSON, JavaScript) are valid.
- Read the diff once as a customer would: nothing internal, nothing that only makes sense inside the company.

## Rules

- One branch and one pull request per change; squash merge. PR title `docs: subject` (append the tracking issue id in parentheses when there is one).
- No AI-tool mentions or `Co-Authored-By` trailers in commits or PRs. Stage files by path, never `git add -A`.
- Never commit credentials, API keys or real workspace data in examples; use placeholders such as `<workspace-slug>` and `wsk_...`.
