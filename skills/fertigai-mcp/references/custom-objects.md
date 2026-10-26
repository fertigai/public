# Custom objects and contacts (fertigai_custom_object_*, fertigai_contacts_*)

## Overview
**Custom objects** are record kinds the workspace defines itself: vendors, contracts, locations, anything agents and tasks need to look up. A **custom object type** (`cot_...`) has a `key`, a `name`, an optional `description` and a `field_schema`; its **records** (`cor_...`) carry a `name`, an optional `external_id` and `data`, the values keyed by field key.

**Contacts** (`con_...`) are the people the workspace talks to: callers, task requesters, mail senders. Every contact has the same built-in fields (name, email, phone, company, notes, external id) plus workspace-defined `custom_fields`.

Shared conventions (ids, pagination, errors) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide. List tools take `cursor?` and `page_size?` (1 to 100, default 20) and return the newest entries first.

**Permissions**
- Custom objects: Custom-Objects-View for every read, Custom-Objects-Manage for record writes, Custom-Objects-Admin to create, edit or delete types.
- Contacts need the Tasks product, plus Contacts-View for reads (including the field schema, duplicates and a contact's tasks, conversations and calls) or Contacts-Manage for create, update, delete and merge. Changing the contact field schema needs Contacts-Fields-Manage.

## Tools
**Custom object types**
| Tool | Args |
|---|---|
| `fertigai_custom_object_types_list` | `search?`, `cursor?`, `page_size?` |
| `fertigai_custom_object_types_get` | `key` |
| `fertigai_custom_object_types_create` | `key`, `name`, `description?`, `field_schema?` |
| `fertigai_custom_object_types_update` | `key`, `name?`, `description?`, `field_schema?` (replaces the whole list) |
| `fertigai_custom_object_types_delete` | `key` |

**Custom object records**
| Tool | Args |
|---|---|
| `fertigai_custom_object_records_list` | `key` (the type), `external_id?`, `search?`, `match?`, `filters?`, `cursor?`, `page_size?` |
| `fertigai_custom_object_records_get` | `key`, `id` |
| `fertigai_custom_object_records_create` | `key`, `name`, `external_id?`, `data?` |
| `fertigai_custom_object_records_update` | `key`, `id`, `name?`, `external_id?`, `data?` (merges per field key) |
| `fertigai_custom_object_records_delete` | `key`, `id` |

**Contacts**
| Tool | Args |
|---|---|
| `fertigai_contacts_list` | `search?`, `company?`, `email_domain?`, `phone?`, `cursor?`, `page_size?` |
| `fertigai_contacts_get` | `id` |
| `fertigai_contacts_create` | `name?`, `email?`, `phone?`, `company?`, `notes?`, `external_id?`, `custom_fields?` |
| `fertigai_contacts_update` | `id`, then any of the create fields |
| `fertigai_contacts_delete` | `id` |
| `fertigai_contacts_field_schema_get` | none |
| `fertigai_contacts_field_schema_update` | `fields` (replaces the whole list) |
| `fertigai_contacts_duplicates` | `id?`, `cursor?`, `page_size?` |
| `fertigai_contacts_merge` | `id` (the contact that stays), `loser_id`, `name`, `email`, `phone`, `company`, `notes`, `custom_fields?` |
| `fertigai_contacts_tasks`, `fertigai_contacts_conversations`, `fertigai_contacts_calls` | `id`, `cursor?`, `page_size?` |

Contact import, export, avatars and erasing a contact's personal data are not available over MCP: use the portal.

## Field schema
A custom object type's `field_schema` and the contact field schema's `fields` are arrays of field definitions, the same shape task fields use (see `references/tasks.md`):

```json
{ "key": "tier", "label": "Tier", "type": "DROPDOWN", "required": false,
  "help_text": "Support tier", "placeholder": null, "default": null,
  "config": { "options": [ { "value": "gold", "label": "Gold" }, { "value": "silver", "label": "Silver" } ] } }
```

- `type`: `SHORT_TEXT`, `LONG_TEXT`, `NUMBER`, `DATE`, `DATE_TIME`, `DROPDOWN`, `MULTI_SELECT`, `CHECKBOX`, `EMAIL`, `PHONE`, and, on custom objects only, `LOOKUP`. Input is case-insensitive; responses use upper case.
- `DROPDOWN` and `MULTI_SELECT` need `config.options` with at least one `{ value, label }`.
- Keys must be unique in the list (a duplicate is a 422).
- The server gives every field an `id` (`fld_...`). Records and contacts address values by field **key**, so ids matter only when you edit a schema.

**Editing a custom object type's schema**: `field_schema` replaces the whole list, so get the type, change the returned list and send all of it back.
- A field sent without an `id` keeps the id of the stored field with the same key; a new key gets a new id. An `id` the type does not hold is a 422.
- A field you leave out is dropped. Dropping a lookup field deletes its links.
- A lookup cannot be pointed at another type while records still reference it.

**Editing the contact field schema**: `fields` also replaces the whole list. A field sent without an `id` gets a new id each time; that is harmless, because contact values are stored by key. Removing a field keeps its values on the contacts.

## Type keys
`key` matches `^[a-z][a-z0-9_]{0,62}$`, is unique among the workspace's types, and never changes (update ignores it). After a type is deleted, its key can be reused. Type and record names and record `external_id`s are at most 255 bytes.

## Lookups
A `LOOKUP` field links a record to a record of another type (or of the same type):

```json
{ "key": "vendor", "label": "Vendor", "type": "LOOKUP", "config": { "target_type_key": "vendor" } }
```

- `target_type_key` must name a type of the workspace.
- The value is the target record's id (`cor_...`). Reads return it in `data.<key>`; the key is absent when no record is linked.
- On a record update, `null` clears the link (not for a required field).

## Record values
| Field type | Value in `data` |
|---|---|
| `SHORT_TEXT`, `LONG_TEXT`, `PHONE` | string |
| `NUMBER` | number (a numeric string is accepted) |
| `CHECKBOX` | `true` / `false` |
| `DATE` | `YYYY-MM-DD` |
| `DATE_TIME` | RFC 3339 with an offset, e.g. `2026-10-26T14:00:00+01:00` |
| `EMAIL` | an email address |
| `DROPDOWN` | one option `value` |
| `MULTI_SELECT` | an array of option values |
| `LOOKUP` | a `cor_...` id |

- **Create**: every required field must be present; an unknown key is a 422.
- **Update**: `data` merges per key; `null` clears one value (a required field cannot be cleared). An empty `name` leaves the name unchanged. `external_id: ""` or `null` clears it.
- `external_id` is unique per type (a taken one is a 409) and can be reused after the record is deleted. Use it to keep records in sync with an outside system.

```json
{ "key": "contract", "name": "ACME 2026", "external_id": "crm-1042",
  "data": { "vendor": "cor_...", "value": 12000, "start": "2026-01-01", "tier": "gold" } }
```

## Finding records
`fertigai_custom_object_records_list` combines:
- `external_id`: exact match;
- `search`: substring of the record name;
- `match`: fuzzy match on the name (sounds-alike and close spellings). It returns one ranked page with no next page; use it to find a record from a spoken or misspelled name;
- `filters`: an object of `{ field_key: value }`. Each filter is an exact match and all must match. A checkbox takes `true`/`false`, a multi-select one option (matches records whose selection contains it), a lookup a `cor_...` id. A `null` value drops that filter. An unknown key, or a value that does not fit the field type, is a 422.

```json
{ "key": "contract", "filters": { "tier": "gold", "vendor": "cor_..." } }
```

## Deleting
- **A type** can be deleted only when it has no records left; otherwise the call fails with `409: ... still has N record(s); delete them first`. Lookups in other types that point at it stay in their schemas.
- **A record**: its own links are removed, and every lookup that pointed at it is cleared.
- **A contact**: `fertigai_contacts_delete` removes it permanently.

## Contacts
- A contact needs an email or a phone (otherwise 422). Email (case-insensitive) and `external_id` are unique in the workspace (409); several contacts may share a phone.
- **Update**: only the fields you send change, and `""` clears one. The result must keep an email or a phone. `custom_fields` replaces the whole object: get the contact, change it, send all of it back.
- `custom_fields` is keyed by field key. Keys in the contact field schema are checked and converted to their type; other keys are stored as given.
- **List filters**: `search` (part of the name, email or phone), `company` (exact), `email_domain` (the part after `@`, case-insensitive), `phone` (exact after normalization).
- **`fertigai_contacts_tasks`**: tasks the contact requested, most recently active first, limited to the tasks you can see. **`fertigai_contacts_conversations`**: conversations from the contact's phone number. **`fertigai_contacts_calls`**: calls from or to it. Both are empty when the contact has no phone.

### Duplicates and merging
Contacts are duplicates when they share a phone number.
- `fertigai_contacts_duplicates` with `id`: `{ "contacts": [...] }`, the other contacts with that contact's phone (empty when it has none).
- Without `id`: `{ "groups": [ { "phone", "contacts": [...] } ], "next_cursor", "has_more" }` for the whole workspace; pass `next_cursor` as `cursor` for the next page.

`fertigai_contacts_merge` folds `loser_id` into `id`. **It cannot be undone.**
- The loser is deleted, and its tasks move to the contact that stays.
- The contact that stays is overwritten with exactly the `name`, `email`, `phone`, `company` and `notes` you send. A field you leave out is cleared, so start from both contacts' values and send every field. Its `external_id` stays. `custom_fields` is written only when you send it.
- The result needs an email or a phone (422). Merging a contact into itself is a 400; an email held by a third contact is a 409.

Typical flow: `fertigai_contacts_duplicates` → `fertigai_contacts_get` both contacts → show the user the combined values and get a confirmation → `fertigai_contacts_merge` with every field set.
