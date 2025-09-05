# Goal

Keep Claude Code fully unmodified while replacing its giant `claude.json` with a **managed minimal JSON view** backed by a durable database. The app still reads/writes the same path, but we only materialize:

* **core**: small, persistent system configuration
* **activeProject**: metadata + a trimmed, token‑budgeted conversation window

Everything else (other projects, long history, archives) lives in the DB and never inflates the on‑disk file or the model context window.

---

## External Interface (what Claude Code sees)

**File path:** `<original>/claude.json`

**Shape (illustrative):**

```json
{
  "version": 3,
  "core": { /* small configuration subset */ },
  "activeProject": {
    "id": "<uuid-or-path>",
    "meta": { /* name, repo, tags, etc. */ },
    "state": { /* ephemeral toggles UI expects */ },
    "conversation": [
      // last N messages + pinned system prompts (policy-defined window)
    ]
  }
}
```

**Contract:**

* The JSON view is **authoritative** for fields we export. Edits by Claude are **validated** and **merged** back into the DB.
* We **ignore** unknown/new fields (forward‑compatible) but persist if allow‑listed.
* Writes are **atomic**: `write → fsync → rename`.

---

## Internal Architecture

### State Daemon

* Owns the file at `claude.json`
* Watches for app writes (fs events)
* Validates + merges allowed changes into DB
* Re‑exports normalized minimal view after merges or internal updates

### SQLite (WAL) + JSON1

* Single portable DB file, transactional, crash‑safe, backup‑friendly
* Tables for `core`, `projects`, `messages`, `snapshots`, `kv`

### Command‑Aware Mutation Layer

Claude Code sometimes mutates `claude.json` directly via slash‑commands.

* **Examples:**

  * `/mcp server add <NAME> <URL>` → `mcpServers[NAME]`
  * `/memory <text>` → append rule to active project
* **Strategy:** allow‑listed handlers map JSON pointer → DB op. Only idempotent upserts and tail‑appends accepted.
* **Events:** emit `mcpServerAdded`, `memoryAdded`, etc.

### Message Store, Summarization & Retention

* **Ingest:** every message/paste/image saved; large blobs stored externally by reference
* **Summarize:** background worker compacts old runs into abstracts (hierarchical summaries)
* **Retention:** clipboard blobs TTL; dedupe tool results; purge old raw docs once summarized
* **Redaction:** secrets filtered before indexing

### Vector & Knowledge‑Graph Sync

* Chunk messages by type; embed via PgVector/LanceDB/etc.
* Build entity/relation edges from summaries (optional GraphRAG)
* Sync runs asynchronously; failures never block JSON export

### Session‑Start Maintenance Agent

Runs at each new session:

* Compacts messages, applies TTL cleanup
* Refreshes vector index
* Writes a digest into project metadata (`sessionDigest`)
* Resource‑bounded (e.g., 5s CPU)

### Concurrency & Race Handling

* Debounce import events (\~300ms)
* Lockfile for serialization
* Detect partial writes → ignore until stable; re‑normalize

---

## DB Schema

```sql
CREATE TABLE core (key TEXT PRIMARY KEY, value JSON);
CREATE TABLE projects (id TEXT PRIMARY KEY, meta JSON, last_opened_at TEXT);
CREATE TABLE messages (
  project_id TEXT, ts INTEGER, role TEXT, content JSON,
  PRIMARY KEY(project_id, ts)
);
CREATE TABLE snapshots (
  project_id TEXT, version INTEGER, doc JSON,
  PRIMARY KEY(project_id, version)
);
CREATE TABLE kv (namespace TEXT, key TEXT, value JSON,
  PRIMARY KEY(namespace, key));
```

---

## Export/Import Algorithm

**Export:**

1. Load core + active project
2. Fetch project meta + recent messages (policy window)
3. Compose object
4. Atomic write to `claude.json`

**Import:**

1. Parse incoming file
2. Validate vs JSON Schema
3. Merge only allow‑listed changes
4. Normalize & re‑export

**Conflict policy:** DB is source of truth. Prefer DB unless incoming is strictly newer (monotonic ts) or is an idempotent command write.

---

## JSON Schema (excerpt)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["version", "core"],
  "properties": {
    "version": {"type": "integer"},
    "core": {"type": "object"},
    "activeProject": {
      "type": ["object", "null"],
      "properties": {
        "id": {"type": "string"},
        "conversation": {"type": "array"}
      }
    }
  }
}
```

---

## CLI UX

```bash
# Import from legacy JSON
cc-jsonview import from-json ./claude-legacy.json

# List & activate projects
cc-jsonview projects list
cc-jsonview projects activate <project_id>

# Manage MCP servers (mirrors /mcp)
cc-jsonview mcp add <name> <url> [--type stdio|sse] [--arg k=v]
cc-jsonview mcp rm <name>

# Manage project memory (mirrors /memory)
cc-jsonview memory add <project_id> "Always lint before commit"
cc-jsonview memory ls <project_id>

# Export policy tuning
cc-jsonview export --policy tokens --budget 12000

# Maintenance (like session agent)
cc-jsonview maintain --compact --ttl-clipboard 14d --reindex

# Status
cc-jsonview status

# Dagger-based test rig
pnpm dagger:up   # spins Claude headless + daemon
pnpm dagger:down # optional cleanup
```

```bash
# Import from legacy JSON
cc-jsonview import from-json ./claude-legacy.json

# List & activate projects
cc-jsonview projects list
cc-jsonview projects activate <project_id>

# Manage MCP servers (mirrors /mcp)
cc-jsonview mcp add <name> <url> [--type stdio|sse] [--arg k=v]
cc-jsonview mcp rm <name>

# Manage project memory (mirrors /memory)
cc-jsonview memory add <project_id> "Always lint before commit"
cc-jsonview memory ls <project_id>

# Export policy tuning
cc-jsonview export --policy tokens --budget 12000

# Maintenance (like session agent)
cc-jsonview maintain --compact --ttl-clipboard 14d --reindex

# Status
cc-jsonview status
```

---

## Operational Details

* Daemon owns writes, atomic fsync/rename
* Snapshots: rotate `claude.json.YYYYMMDD-HHMMSS`
* Structured logs + optional Prometheus metrics
* **Containerization:** Prefer Dagger + container-use for an isolated Claude headless env and a sidecar daemon. Only mount `/workspace`, a throwaway `~/.claude`, and a `/data` volume; wire hooks via the mounted `.claude/settings.json` to pass `transcript_path` to the daemon.
* Daemon owns writes, atomic fsync/rename
* Snapshots: rotate `claude.json.YYYYMMDD-HHMMSS`
* Structured logs + optional Prometheus metrics

---

## Security & Privacy

* Secrets detector (regex+entropy) masks sensitive data
* Optional per‑project encryption at rest
* Audit log for `/mcp` & `/memory` mutations

---

## Testing

* Kill process mid‑export → file still valid
* Simulate rapid `/mcp add` edits → idempotent
* Fuzz large paste events → only references exported
* Race tests: concurrent Claude write + daemon export → DB‑canonical end state
* **Container tests:** run the Dagger plan in CI against fixtures; assert exported view size, token window, and DB row counts per run
* Kill process mid‑export → file still valid
* Simulate rapid `/mcp add` edits → idempotent
* Fuzz large paste events → only references exported
* Race tests: concurrent Claude write + daemon export → DB‑canonical end state

---

## Repo Layout

```
claude-jsonview/
├─ packages/daemon
├─ packages/cli
├─ packages/schema
├─ config/ (launchd/systemd)
├─ dagger/                 # container-use + Dagger plan
│  ├─ dagger.ts           # spins claude + sidecar
│  └─ container-use.hcl   # optional module config/pins
├─ tests/
├─ ADRs/
└─ README.md
```

---

## Next Quick Wins

* **Watcher**: log when Claude writes to sensitive paths (`/mcpServers`, `/projects/*/memories`)
* **Token‑budget window**: heuristic + model tokenizer fallback
* **Clipboard TTL**: external blob store, replace old with refs

---

## Containerized Test Environment (Dagger + container-use)

**Objective:** run a **clean, reproducible Claude Code test rig** that can’t pollute your main hive environment.

### Approach

* Use **Dagger** with the **container-use** module to spin two containers per run:

  1. **Claude headless** container with an isolated `$HOME/.claude` (bind-mount a throwaway dir).
  2. **DB/exporter daemon** sidecar (our project) with its own `/data` volume for SQLite + blobs.
* Mount only three paths:

  * `/workspace` → a fixture or repo snapshot (read-only is fine)
  * `/home/worker/.claude` → throwaway state for Claude test runs
  * `/data` → SQLite DB + blob store (safe to nuke between runs)
* Configure **hooks** in the mounted `.claude/settings.json` to call our importer/exporter on `SessionStart`, `PreCompact`, and `SessionEnd` with `transcript_path`.

### Example (high-level)

* **dagger.ts** creates both containers, mounts volumes, and runs:

  * sidecar: `node dist/daemon.js --tokens 12000 --ttl 14d`
  * claude: `claude --headless --prompt "smoke test" --output-format json`
* After run, inspect:

  * `.claude-test/claude.json` → confirm minimal view
  * `state/db.sqlite` → schemas, message counts, summaries

### CI

* Add a GitHub Actions job to execute the same Dagger plan against fixtures.
* Artifacts: exported minimal `claude.json`, SQLite DB, logs.

### Benefits

* **Isolation** from your hive build
* **Reproducibility**: pinned bases and immutable recipe
* **Parallelism**: multiple sessions without cross-contamination

---

## Notes from Scaffold

* Grounded in user‑supplied structure: `mcpServers`, `projects{}` with histories, etc.filecite55†claude-scaffold-null.json56†claude-scaffold.json
* Keep idempotent MCP upserts, append‑only project memories
* Only surface active project in the JSON view
* Expire or summarize noisy history to preserve context budget

(End of document)
