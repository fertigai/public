# Conversation tests (fertigai_conversation_test_*, fertigai_conversation_tests_*, fertigai_agent_branch_conversation_tests_*)

## Overview
A **conversation test** checks how an agent behaves: what it replies, which function it calls, or how a whole simulated conversation goes. Tests live in the workspace **library** and run against an agent branch:
- **Folders** (`ctfd_...`) organise the library. They have no effect on what a test does.
- **Tests** (`ctst_...`) hold a name, an optional folder and a `definition`.
- A test runs only on a branch it is **attached** to. One test can be attached to many branches.
- A **run** (`ctrn_...`) executes some or all of a branch's attached tests and collects one result per test.

Shared conventions (ids, pagination, errors) are in the skill index (SKILL.md). Every tool here also takes an optional `workspace` slug, required only when the connection is org-wide (see SKILL.md / `fertigai_whoami`). List tools take `cursor?` and `page_size?` (1 to 100, default 20) and return the newest entries first.

**Permissions**: reads need Agents-View; creating, changing, deleting, attaching, detaching and starting runs need Agents-Edit. When conversation tests are not enabled for the workspace, every tool here answers `403` with `conversational-agent-tests-not-rolled-out`; they cannot be enabled over MCP.

## Tools
**Folders**
| Tool | Args |
|---|---|
| `fertigai_conversation_test_folders_list` | none (every folder, not paginated) |
| `fertigai_conversation_test_folders_create` | `name`, `parent_id?` |
| `fertigai_conversation_test_folders_update` | `id`, `name`, `parent_id?` (full replace) |
| `fertigai_conversation_test_folders_delete` | `id` |

**Tests**
| Tool | Args |
|---|---|
| `fertigai_conversation_tests_list` | `search?`, `folder_id?`, `cursor?`, `page_size?` |
| `fertigai_conversation_tests_get` | `id` |
| `fertigai_conversation_tests_create` | `name`, `folder_id?`, `definition` |
| `fertigai_conversation_tests_update` | `id`, `name`, `folder_id?`, `definition` (full replace) |
| `fertigai_conversation_tests_delete` | `id` |

**Tests on a branch**
| Tool | Args |
|---|---|
| `fertigai_agent_branch_conversation_tests_list` | `agent_id`, `branch_id` |
| `fertigai_agent_branch_conversation_tests_attach` | `agent_id`, `branch_id`, `test_id` |
| `fertigai_agent_branch_conversation_tests_detach` | `agent_id`, `branch_id`, `test_id` |

**Runs**
| Tool | Args |
|---|---|
| `fertigai_conversation_test_runs_start` | `agent_id`, `branch_id`, `test_ids?` |
| `fertigai_conversation_test_runs_list` | `agent_id`, `branch_id`, `cursor?`, `page_size?` |
| `fertigai_conversation_test_runs_get` | `agent_id`, `branch_id`, `run_id` |

## Folders
- The folder list is flat: every folder of the workspace with its `parent_id` (`null` at the root). Build the tree from `parent_id`.
- `name`: 1 to 255 bytes after trimming, unique among the folders that share its parent (`409` otherwise).
- **Update is a full replace**: send `name`, and `parent_id` unless the folder belongs at the root. A missing `parent_id` moves the folder to the root. Moving a folder into itself or one of its subfolders is a `400`.
- **Delete** works only on an empty folder. A folder that still holds folders or tests answers `409`: move or delete its contents first.

## Tests
Each test has `id`, `folder_id` (absent at the root), `name`, `definition`, `attached_branch_count` (how many branches it is attached to), `created_at` and `updated_at`.

- **List**: `folder_id` takes `root` (tests outside any folder), a `ctfd_...` id (that folder only), or nothing (every test). `search` is a case-insensitive substring of the name and spans every folder unless `folder_id` narrows it.
- **Create** saves the test in the library. It does not run it: attach it to a branch, then start a run.
- **Update is a full replace** of `name`, `folder_id` and the whole `definition`; a missing `folder_id` moves the test to the root. Read the test with `fertigai_conversation_tests_get`, edit it, and send all of it back.
- **Delete** is permanent. It detaches the test from every branch. Results of past runs keep the test's name.
- A definition that breaks a rule below answers `400` with `validation-error` and a pointer to the field. Every `FUNCTION` and `INTEGRATION` the definition references must exist in the workspace.
- `definition` is a JSON object; a JSON string of the same object is accepted too.

## The test definition
```json
{
  "kind": "RESPONSE",
  "history": [ { "role": "USER", "message": "Are you open on Saturdays?" } ],
  "dynamic_variables": { "customer_tier": "gold" },
  "response": { "success_condition": "The agent says the office is closed on Saturdays." }
}
```
- `kind`: `RESPONSE`, `TOOL_CALL` or `SIMULATION`. Set exactly the criteria object that matches it (`response`, `tool_call` or `simulation`) and no other.
- `history`: the conversation so far, as turns `{ role, message, tool_calls? }`. `role` is `USER` or `AGENT`; a turn needs a `message` or at least one tool call. For `RESPONSE` and `TOOL_CALL` the history is required and must end with a `USER` turn: the test judges the agent's next step after it. For `SIMULATION` it is optional.
- A tool call in the history is `{ function, arguments, result, is_error }`: a function reference, its arguments object, and what it returned.
- `dynamic_variables`: `{ "name": "value" }` overrides of the branch's dynamic variables for this test. Every other variable the branch declares uses its default. Names must not be empty or reserved.

### Function references
`{ "type": "FUNCTION" | "SYSTEM_TOOL" | "INTEGRATION", "id": "..." }`:
- `FUNCTION`: a workspace function, `fn_...`.
- `SYSTEM_TOOL`: a system tool slug such as `end_call`, `language_detection` or `transfer_to_number`. `transfer_to_agent` cannot be tested.
- `INTEGRATION`: an integration key from the catalogue.

### Parameter checks
`{ "path": "customer.email", "type": "EXACT", "value": "jane@example.com" }` checks one argument of a function call:
- `path`: a dot path into the call's arguments.
- `type`: `ANY` (present, with any value), `EXACT`, `REGEX` (the pattern must compile) or `LLM` (a natural-language criterion; `value` is required).

### RESPONSE: judge the next reply
`response: { success_condition, success_examples?, failure_examples? }`. `success_condition` is required: what a good reply does. The examples are sample replies that pass or fail.

### TOOL_CALL: the next step calls a function or reaches a node
```json
{
  "kind": "TOOL_CALL",
  "history": [ { "role": "USER", "message": "Please book me in for Tuesday at 10." } ],
  "tool_call": {
    "function": { "type": "FUNCTION", "id": "fn_..." },
    "parameters": [ { "path": "date", "type": "LLM", "value": "a Tuesday" } ],
    "expect_absent": false,
    "any_call_matches": false
  }
}
```
- Set exactly one of `function` and `workflow_node_id` (the `id` of a node in the branch's workflow, see agents-workflow.md).
- `parameters`: optional checks on the call's arguments.
- `expect_absent: true`: the test passes when the function is NOT called.
- `any_call_matches: true`: the test passes when any one of several calls matches. Otherwise more than one call fails the test.

### SIMULATION: a simulated caller talks to the agent
```json
{
  "kind": "SIMULATION",
  "simulation": {
    "scenario": "You are a customer who wants to cancel tomorrow's appointment and is in a hurry.",
    "success_conditions": [ "The agent confirms the cancellation.", "The agent does not offer a new appointment unasked." ],
    "max_turns": 8,
    "mocks": [ { "function": { "type": "FUNCTION", "id": "fn_..." }, "result": "{\"cancelled\": true}", "is_error": false, "when": [] } ],
    "real_functions": []
  }
}
```
- `scenario` (required): the situation and persona of the simulated caller.
- `success_conditions`: 1 to 10 non-empty criteria; all of them are judged.
- `max_turns`: 1 to 20, default 5.
- `mocks`: canned results `{ function, result, is_error?, when? }`. `when` is a list of parameter checks that must all match for the mock to answer; empty matches every call.
- `real_functions`: functions that really run during the simulation. Every other function is mocked, and a call no mock matches answers with an error. A function cannot be both mocked and real.

Prefer mocks over `real_functions`: a real function has real effects (a booking, an email, a ticket).

## Attaching tests to a branch
- `fertigai_agent_branch_conversation_tests_attach` and `_detach` are idempotent. Detaching keeps the test in the library.
- A branch holds at most 50 tests; attaching one more answers `409`.
- `fertigai_agent_branch_conversation_tests_list` returns the branch's tests, not paginated.
- A branch created as a copy of another (`fertigai_agent_branch_create` with `from_branch_id`) keeps the source's tests.

## Runs
### Starting a run
`fertigai_conversation_test_runs_start { agent_id, branch_id, test_ids? }` runs the listed tests, or every attached test when `test_ids` is empty or missing. It returns at once with the run (`ctrn_...`) in status `PENDING`; the tests run in the background.

A test the branch cannot run is not run: it reports `NOT_APPLICABLE` when it references a function the branch does not have attached, or targets a workflow node the branch does not have.

| Answer | Why |
|---|---|
| `400` | the branch has no tests attached, or a test in `test_ids` is not attached to it |
| `409` | the branch runs on agent engine V2, where test runs are not available yet |
| `429` | more than 10 runs started in the workspace within one minute |
| `503` | the run could not be started; it stays in the run history with status `ERROR` |

Runs cost usage, and simulations cost the most. Start runs on purpose, not in a loop.

### Waiting for the result
Poll `fertigai_conversation_test_runs_get { agent_id, branch_id, run_id }` every few seconds until `status` is `COMPLETED` or `ERROR`. Reading a `PENDING` run is what fetches its progress: its status changes only when you read it, so keep polling.

The run `status` is the lifecycle only:
- `PENDING`: still running.
- `COMPLETED`: every test has a result. Failing tests do not make the run fail: read the counts.
- `ERROR`: the run itself failed; `error_message` is `The test run could not be started.` or `The test run did not finish in time.` (a run still `PENDING` after 15 minutes ends as `ERROR`).

### Reading results
A run carries `passed_count`, `failed_count`, `not_applicable_count`, `error_count`, `total_count`, `cost_usd_micros` (the run's cost in millionths of a US dollar), `created_at` and `completed_at`.

Only `fertigai_conversation_test_runs_get` returns `results`, one per test:
- `test_id`, `test_name` (the name at run time), `kind`;
- `status`: `PENDING`, `PASSED`, `FAILED`, `NOT_APPLICABLE` or `ERROR`;
- `summary` and `reasons[]`: why the test passed or failed;
- `transcript[]`: what happened, as turns in the same shape as `history`. For a simulation it is the whole conversation. A function call that was not executed during the test reads `skipped in test` as its result.

`fertigai_conversation_test_runs_list` returns a branch's runs with their status and counts, never their `results`.

## Example: test a branch
1. Create the test: `fertigai_conversation_tests_create { "name": "Closed on Saturday", "definition": { ... } }` returns `{ "id": "ctst_A", ... }`.
2. Attach it: `fertigai_agent_branch_conversation_tests_attach { "agent_id": "agt_...", "branch_id": "br_...", "test_id": "ctst_A" }`.
3. Run it: `fertigai_conversation_test_runs_start { "agent_id": "agt_...", "branch_id": "br_...", "test_ids": ["ctst_A"] }` returns `{ "id": "ctrn_B", "status": "PENDING", ... }`.
4. Poll `fertigai_conversation_test_runs_get { "agent_id": "agt_...", "branch_id": "br_...", "run_id": "ctrn_B" }` every few seconds until `status` is `COMPLETED` or `ERROR`.
5. Report the counts, and for each `FAILED` result its `summary` and `reasons`.

## Common mistakes
- Starting a run with a test that is only in the library: attach it to the branch first (`400` otherwise).
- Treating `COMPLETED` as "all tests passed": check `failed_count`, `error_count` and `not_applicable_count`.
- Looking for per-test results in `fertigai_conversation_test_runs_list`: only `_get` has them.
- Stopping the poll early, or waiting without reading: a run's status changes only when it is read.
- A `RESPONSE` or `TOOL_CALL` history that is empty or ends with an `AGENT` turn.
- Setting a criteria object that does not match `kind`, or both `function` and `workflow_node_id` on a `TOOL_CALL`.
- Updating a test or folder with only the changed fields: update replaces everything, and a missing `folder_id` or `parent_id` moves it to the root.
- Deleting a folder that is not empty (`409`).
