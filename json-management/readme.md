Claude Code JSON View

This repository contains code and documentation for a project to manage Claude Code’s conversation transcripts and configuration state more efficiently. Claude Code writes all conversation history, tool outputs, and configuration into a single claude.json file. Over time this file balloons in size and consumes the entire context window. The goal of this project is to replace the monolithic file with a database‑backed storage layer and a thin JSON view so that Claude Code only sees the core configuration and the currently active project.

Overview

The core of our design is a SQLite database that stores all historical messages, snapshots, and project metadata, and a daemon that exports a minimal JSON file at the same location Claude Code expects its transcript. The JSON view contains:

A version field for schema evolution.

A core section with the global system configuration.

An activeProject section with the active project ID, metadata, and a sliding window of recent messages.

Claude Code continues to read and write the JSON file as usual, but behind the scenes our daemon synchronises the database and regenerates the view. Hooks configured in .claude/settings.json trigger our maintenance script on events like SessionStart, PreCompact, and SessionEnd
docs.anthropic.com
.

Repository Contents

PRJ-claude-json-view-db-blueprint.md – A comprehensive blueprint outlining the DB‑backed design, integration with Claude Code hooks, and considerations for memory management and slash‑command awareness.

ADR‑0001-adopt-db-layer.md – An Architecture Decision Record detailing the rationale for adopting a database rather than using a single JSON file.

CHANGELOG.md – A chronologically ordered list of changes using semantic versioning.

answer.js – Starter code for generating slides with PptxGenJS (unrelated to the JSON view but part of the repository).

Next Steps

Initialise a Git repository in this directory (git init) to track changes. Use semantic versioning tags (e.g., v0.1.0) and Conventional Commits for messages.

Review the blueprint and ADR documents to familiarise yourself with the design.

Implement the daemon and hook scripts as described in the blueprint. Test them with Claude Code in both interactive and headless modes.

For more details, see the blueprint document.
