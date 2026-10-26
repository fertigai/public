# Knowledge base (fertigai_knowledge_base_*)

## Overview
The **knowledge base** is the workspace's library of documents that agents read during a conversation. It is a tree of **items** (`kbi_...`): each item has a `type`, either `FOLDER` or `FILE`. Folders only organise; files hold the content. A branch uses a folder or a file once it is attached to the branch or to one of its workflow nodes (see "Attaching to a branch" below).

Shared conventions (ids, pagination, errors) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`).

**Permissions**: listing works for every workspace member. Every write (folders, uploads, rename, move, delete) needs Integrations-Manage.

## Tools
| Tool | Args | Notes |
|---|---|---|
| `fertigai_knowledge_base_list` | `parent_id?`, `search?`, `cursor?`, `page_size?` | without `parent_id`: every item, at all levels |
| `fertigai_knowledge_base_folders_create` | `name`, `parent_id?` | returns the new folder |
| `fertigai_knowledge_base_files_create` | `name`, `file_name`, `content_text?` or `content_base64?`, `content_type?`, `parent_id?` | upload, at most 5 MB |
| `fertigai_knowledge_base_items_rename` | `id`, `name` | folder or file |
| `fertigai_knowledge_base_items_move` | `id`, `parent_id?` | omit `parent_id` to move to the root |
| `fertigai_knowledge_base_items_delete` | `id` | permanent, see below |

Knowledge base files cannot be downloaded over MCP: open them in the portal.

## Items
Every item, as list and create return it:

| Field | Notes |
|---|---|
| `id` | `kbi_...` |
| `name` | display name |
| `type` | `FOLDER` or `FILE` |
| `parent_id` | the folder it sits in; `null` at the root |
| `file_format`, `file_size` | files only: the format (from the file name's extension) and the size in bytes |
| `index_status` | files only: `NOT_INDEXED`, `INDEXING`, `INDEXED` or `FAILED` |
| `index_error_code`, `index_error_message` | why indexing `FAILED` |
| `created_at`, `updated_at` | timestamps |

A new or renamed file is indexed in the background: `index_status` moves from `INDEXING` to `INDEXED` (or `FAILED`). List the item again to follow it.

## Listing
- **Without `parent_id`**, the list returns every item of the workspace, folders and files at every level, not only the root. Build the tree from each item's `parent_id`.
- **With `parent_id`** (a folder's `kbi_...` id), the list returns only that folder's direct children.
- `search` is a case-insensitive substring match on the name.
- The list is paginated (`cursor`, `page_size` 1 to 100, default 20).

## Folders
`fertigai_knowledge_base_folders_create { name, parent_id? }` creates a folder at the root, or inside `parent_id`.
- `name`: 1 to 255 bytes after trimming. The same rule applies to file names and renames.
- `parent_id` must be an existing folder: an unknown id is a `404` (`parent folder not found`), and a file id is a `400` (`parent must be a folder`). Uploads and moves follow the same rule.

## Uploading a file
`fertigai_knowledge_base_files_create` uploads one file, at the root or into the folder `parent_id`.

| Arg | Notes |
|---|---|
| `name` | display name, 1 to 255 bytes after trimming |
| `file_name` | the file name with its extension. The extension sets the format |
| `content_text` | the file's text (UTF-8), uploaded as its bytes. Use it for `txt`, `md`, `markdown`, `html` and `htm` |
| `content_base64` | the file, base64-encoded (standard alphabet). Use it for binary formats: `pdf`, `docx`, `epub` |
| `content_type` | optional MIME type (default `application/octet-stream`). Informational only: the format always comes from `file_name` |
| `parent_id` | optional folder to upload into |

- Send **exactly one** of `content_text` and `content_base64`. Both, or neither, is refused before anything is uploaded.
- **Allowed formats** (the `file_name` extension): `pdf`, `docx`, `epub`, `txt`, `md`, `markdown`, `html`, `htm`. Any other extension is a `400` that lists the allowed formats.
- **Size**: at most 5 MB (5242880 bytes) after decoding. A larger file is refused before anything is uploaded, with an error naming the limit. Upload larger files in the portal.
- Send the file's content, never a file path or a URL: there is no upload from a URL.
- When the new file lands inside a folder that a branch has attached, that branch picks up the new file.

```
fertigai_knowledge_base_files_create {
  "name": "Opening hours FAQ",
  "file_name": "opening-hours.md",
  "content_text": "# Opening hours\n\nWe are open Monday to Friday, 9:00 to 17:00.",
  "parent_id": "kbi_..."
}
```

## Rename, move, delete
- **Rename** (`fertigai_knowledge_base_items_rename { id, name }`) works on folders and files. A renamed file is indexed again (`index_status` goes back to `INDEXING`), and branches that use it pick up the change.
- **Move** (`fertigai_knowledge_base_items_move { id, parent_id? }`) puts the item into the folder `parent_id`, or at the root when `parent_id` is omitted. A folder cannot move into itself or one of its subfolders (`400`). Branches that used the item through its old folder, and branches that have its new folder attached, both pick up the change.
- **Delete** (`fertigai_knowledge_base_items_delete { id }`) is **permanent** and cannot be undone. Deleting a folder deletes everything inside it. The stored files are removed, and the deleted items are detached from every branch and workflow node that had them attached; those branches continue without them. The tool returns `{}`. Before deleting, check which branches use the item (`fertigai_agent_attachments_get`) and tell the user.

## Attaching to a branch
There is no separate knowledge base attach tool. Attach an item (a folder or a file) with `fertigai_agent_branch_configure`, by putting its `kbi_...` id in the `attachments.knowledge_bases` section, at branch level or on a workflow node. That section replaces the whole knowledge base attachment state, so read the current attachments first with `fertigai_agent_attachments_get` and send them back with the new id added. The full shape and the read-modify-write loop are in `references/attachments.md`.

## Example: upload a document and attach it
1. Create a folder: `fertigai_knowledge_base_folders_create { "name": "Product docs" }` returns `{ "id": "kbi_A", ... }`.
2. Upload a PDF into it: `fertigai_knowledge_base_files_create { "name": "Price list", "file_name": "prices.pdf", "content_base64": "JVBERi0...", "parent_id": "kbi_A" }`.
3. Check the upload: `fertigai_knowledge_base_list { "parent_id": "kbi_A" }` shows the file with its `index_status`; on `FAILED`, `index_error_message` says why.
4. Read the branch's attachments: `fertigai_agent_attachments_get { "agent_id": "agt_...", "branch_id": "br_..." }`.
5. Add `"kbi_A"` to `knowledge_bases.branch`, keep everything else, and send the whole `attachments` object back with `fertigai_agent_branch_configure`.

## Common mistakes
- Expecting `fertigai_knowledge_base_list` without `parent_id` to return only the root: it returns every item at every level. Pass a folder's id to see one level.
- Sending both `content_text` and `content_base64`, or neither.
- Base64-encoding a text file into `content_text`, or sending a binary file as `content_text`: text goes in `content_text` as is, binary files go in `content_base64`.
- A `file_name` without one of the allowed extensions, or relying on `content_type` to set the format.
- Uploading a file over 5 MB over MCP: use the portal.
- Deleting a folder that branches still rely on: the delete is permanent and detaches every item inside it from every branch.
- Attaching with a partial `attachments` object: it replaces the whole state and can detach every function. Read first, then send everything back.
