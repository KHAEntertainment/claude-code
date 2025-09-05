Claude Code JSON View – DB‑backed Blueprint
Background

Claude Code writes all conversation history, tool calls, pasted documents, images, and configuration into a single claude.json. This file can quickly grow to several megabytes, reducing the effective context window and making manual cleanup fragile. The SDK provides a cleanupPeriodDays setting to purge old transcripts
docs.anthropic.com
, but it does not summarise or compact the data. Meanwhile, Claude Code uses hierarchical CLAUDE.md files to store long‑term memory and instructions at the enterprise, project, and user levels
docs.anthropic.com
—these are separate from the session transcripts and should remain untouched.

Problem

A monolithic JSON file grows without bound, bloating the context window.

Every built‑in slash command (/mcp, /memory, /compact, etc.) can mutate claude.json, making it brittle to clean manually
docs.anthropic.com
.

While you often only work on one project at a time, the JSON contains the entire session history for all projects.

Goals

Separate storage and view. Persist all session transcripts, messages, and metadata in a durable database while generating a minimal JSON file that contains only what Claude needs: core system config and the active project.

Preserve existing behaviour. Claude Code must continue to read and write claude.json at the same path without modification to its core.

Facilitate summarisation and retention. Provide tools to summarise old messages, expire clipboard data, and store embeddings in an external vector store.

Hook into Claude Code natively. Use Claude Code’s hook system to run maintenance scripts on events like session start, pre‑compact, and session end
docs.anthropic.com
.

Document and version control the architecture. Record decisions via ADRs and maintain changelogs and READMEs.

Design Overview
Database Layer

Use SQLite in WAL mode to store:

core table: small key/value pairs for global system config.

projects table: project IDs, metadata, and last‑opened timestamps.

messages table: project ID, timestamp, role, and content (JSON) for each message.

snapshots table (optional): saved versions of documents or transcripts.

Index tables by project ID and timestamp to support efficient retrieval of recent messages.

JSON View

The daemon composes a minimal JSON file at the path Claude Code expects (claude.json or similar) with this structure:

{
  "version": 3,
  "core": { /* loaded from DB */ },
  "activeProject": {
    "id": "abc123",
    "meta": { /* name, description, repo info */ },
    "state": { /* buffers, window positions, toggles */ },
    "conversation": [ /* sliding window of recent messages */ ]
  }
}


The conversation window is a token‑budgeted slice of recent messages plus any pinned or system prompts. Old messages remain in the database and can be summarised.

Hook Integration

Rather than polling claude.json for changes, the system uses Claude Code’s hooks. Hooks are configured in .claude/settings.json and run user‑defined scripts when certain events occur
docs.anthropic.com
. Each hook receives JSON on stdin with fields like session_id, cwd, and transcript_path
docs.anthropic.com
.

Configure hooks as follows:

{
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "./scripts/cc-jsonview-maintain.sh" } ] }
    ],
    "PreCompact": [
      { "hooks": [ { "type": "command", "command": "./scripts/cc-jsonview-maintain.sh" } ] }
    ],
    "SessionEnd": [
      { "hooks": [ { "type": "command", "command": "./scripts/cc-jsonview-maintain.sh" } ] }
    ]
  }
}


The cc-jsonview-maintain.sh script should:

Read the incoming JSON from stdin to get transcript_path, hook_event_name, and other context.

Parse the transcript file (JSONL), ingest new messages into the database, and detect changes triggered by slash commands.

Summarise old messages and purge expired clipboard entries if necessary.

Regenerate the minimal JSON view and atomically write it to the expected path.

Slash‑Command Awareness

Built‑in slash commands like /mcp, /memory, /compact, and /clear modify project state and transcript files
docs.anthropic.com
. Our script monitors these events in two ways:

Transcript ingestion – When a hook runs, the script reads the latest messages from the transcript file. If a /mcp command is detected, the corresponding .mcp.json is imported into the DB. If /memory is used, memory files remain untouched (they’re outside the DB scope). /compact triggers summarisation and resets the conversation window.

File watcher (optional) – For immediate responsiveness between hooks, a lightweight watcher can detect changes to specific config files (*.mcp.json, CLAUDE.md) and update the database. This can be implemented with fs.watch or a similar mechanism.

Session Maintenance Agent

On SessionStart and SessionEnd, a maintenance routine can perform tasks such as:

Summarising older message batches and storing the summary in the database.

Removing large paste or clipboard data beyond a retention period.

Syncing summaries and embeddings to an external vector or knowledge graph.

Generating a “since you were away” digest summarising what changed since the last session.

The maintenance agent operates asynchronously and does not block the interactive session.

Implementation Guidelines

Atomic writes – Use a temporary file and fs.rename() to ensure the JSON view is updated atomically.

Schema versioning – Store a version field in the JSON view and migrate the database schema as needed.

Testing – Write unit and integration tests for the script. Simulate crashes during writes to verify durability.

CLI – Provide a small CLI (cc-jsonview) with commands to import the original claude.json, list projects, activate a project, and configure the token budget.

Documentation – Record architecture decisions via ADRs. Maintain this blueprint alongside the code. Use semantic versioning and update CHANGELOG.md on each release.

Future Work

Evaluate using a FUSE or Dokany virtual file to emulate claude.json directly, eliminating the need for a pre‑exported file.

Investigate integration with Claude Code’s PreToolUse and PostToolUse hooks to capture tool inputs and outputs in real time.

Develop heuristics for automatic summarisation (e.g., vector‑based summarisation, hierarchical summarisation) to keep context windows lean.

Implement a background service to re‑index or compact the database and rotate old logs.
