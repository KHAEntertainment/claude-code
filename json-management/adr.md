ADR‑0001: Adopt a Database‑backed Storage Layer for Claude Code
Status

Accepted – 2025‑08‑28

Context

Claude Code stores all session transcripts and project state in a single claude.json file. This file grows continuously as every message, code snippet, document, and image is appended, leading to slow loading times and quickly consuming the context window. The SDK exposes a cleanupPeriodDays setting to control retention of chat transcripts
docs.anthropic.com
, but it does not summarise or prune messages. Meanwhile, Claude Code loads long‑term instructions from CLAUDE.md files at different levels (enterprise, project, user)
docs.anthropic.com
. These memory files must remain intact.

Claude Code provides a hook mechanism in settings.json that can run arbitrary scripts on events such as SessionStart, PreCompact, and SessionEnd
docs.anthropic.com
. Hooks receive the current transcript_path and other context via stdin
docs.anthropic.com
, allowing external programs to inspect and modify the conversation.

Decision

We will replace direct writes to a monolithic claude.json with a database‑backed storage layer that persists all transcripts, messages, and project metadata. The key elements of this decision are:

Database – Use SQLite (WAL mode) as a lightweight, embedded database. It supports transactions, concurrency, and durability without requiring a server. The schema will include tables for core configuration, projects, messages, and snapshots.

JSON View – Maintain a generated JSON file at the original path so that Claude Code continues to read and write as it always has. The JSON view contains only the current core config and active project with a sliding window of recent messages.

Hook Integration – Configure Claude Code hooks (SessionStart, PreCompact, SessionEnd) to run a maintenance script that ingests new messages into the database, summarises old ones, and regenerates the JSON view. This avoids file polling and integrates seamlessly with Claude Code’s lifecycle.

Summarisation and Retention – Implement summarisation of old messages and expiration of large paste data in the maintenance script. Summaries can be stored in the database and used to reconstruct context when needed.

Backward Compatibility – Leave CLAUDE.md memory files untouched. All memory management continues through Claude Code’s standard mechanisms and slash commands.

Consequences

Improved Performance – The JSON file remains small, allowing Claude Code to load sessions quickly and preserve more tokens for new messages.

Enhanced Durability – A proper database reduces the risk of data corruption and allows atomic updates. Backups and replication can be added later.

Complexity – A daemon and hook scripts must be maintained. Developers need to set up the database and manage migrations.

Extensibility – With a database, it becomes easier to integrate summarisation, vector embeddings, and analytics. External tools can query the DB without parsing JSON.

Alternatives Considered

Continue using a single JSON file – This approach requires manual compaction and suffers from corruption risk when the file grows large. It does not scale and makes concurrency difficult.

Use multiple JSON files (sharding) – Splitting the JSON into smaller files by project could reduce file size, but it still lacks transactions and is error‑prone. Versioning and migrations become harder to manage.

Adopt an external server‑based database (Postgres) – While a server DB offers more features, it introduces deployment complexity. SQLite offers sufficient performance and durability for this use case.

Outcome

This ADR is accepted. We will proceed with implementing the database layer and JSON view, updating the documentation accordingly, and preparing the repository for version control.
