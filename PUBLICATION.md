# Publication scope

This repository shares architecture notes and an independent, configurable example. It does not distribute the author's running environment or promise a complete deployment of the described integrations.

## Intentionally public

- The use of a Mac mini, Apple Silicon, macOS, Docker Desktop, and G-RAID Thunderbolt storage.
- The reasoning behind separating media from application state.
- Reference service/volume names and historical ingest/backup schedules.
- Placeholder paths, a loopback port example, and general integration notes.

Hardware, network layout, software versions, volume mappings, permissions, schedules, and provider requirements vary. Readers should adapt and verify these choices for their own systems.

## Keep outside Git

- Filled `.env` files, SMTP credentials, app passwords, tokens, and private keys.
- Media libraries, personal filenames, recipient lists, and application databases.
- Logs, backup archives, private screenshots, and live filesystem inventories.

`.gitignore` is a convenience, not a history scrubber. Review staged files before committing. If a real credential is ever committed, revoke or rotate it and review its history exposure rather than only deleting the current file.

## Review record

On November 10, 2026, the original two-commit repository was reviewed for publication. It contained only a README, with placeholder mail credentials; the secret scan found no credential-pattern matches. The preparation revision replaces the incomplete relay-dependent Compose snippet with a standalone example and makes environment-specific and historical claims explicit.

The preparation review covers committed content and fetched Git history. It does not certify Issues, pull-request discussions, Actions artifacts, releases, external systems, or future changes. No running Audiobookshelf service or storage is changed by this documentation update. The example is configuration-validated, not started against the author's media.
