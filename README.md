# Audiobookshelf Homelab

Architecture notes from Chris Clark’s personal Audiobookshelf setup: a media library on Apple Silicon, container-based storage, automated ingestion, and a small email integration for ebook delivery.

The interesting part is how the pieces fit together: keeping media separate from application state, handling macOS permissions, and planning for backups without modifying the upstream application.

> **A reference setup, not a universal recipe.** The hardware and configuration below describe my own homelab. Your CPU architecture, operating system, storage, paths, permissions, ports, email provider, and backup needs may differ. Service names and schedules are examples, not a live inventory or a guarantee of current operating status.

## At a glance

| Component | My reference setup | What may differ in yours |
| --- | --- | --- |
| Host | Mac mini running macOS on Apple Silicon | Linux, another Mac, or other supported hardware; check image compatibility |
| Container runtime | Docker Desktop | Docker Engine or another compatible environment |
| External storage | G-RAID Thunderbolt storage | A different disk, NAS, or storage layout with suitable access and capacity |
| Media storage | A Docker named volume called `storyteller_data` in the original notes | Your volume or bind mount; the name does not identify a particular physical drive |
| Application state | Separate config and metadata volumes | Back up and restore the actual volumes used by your deployment |
| Email | A private SMTP relay forwarding through Gmail | Your provider, sender policy, and delivery requirements |
| Ingestion | A scheduled macOS workflow | Your scheduler, permissions model, and ingest process |

An external drive and a Docker named volume are different layers. On Docker Desktop, a named volume is managed within Docker’s storage environment unless you have explicitly configured otherwise. Do not assume a volume is stored on an external disk just because one is attached.

## What is in this repository

- A description of the reference architecture and the reasoning behind it.
- An independent [Audiobookshelf Compose example](examples/compose.yaml), with [placeholder settings](examples/.env.example).
- Notes on ingestion, logging, backups, and the email workaround used in this setup.

The custom SMTP relay, ingestion scripts, macOS app wrapper, and backup jobs are **not included**. The Compose example runs only the library application; it does not reproduce the full homelab. No media files, application databases, account credentials, or live deployment settings are included.

## Architecture

```text
Incoming media
     │
     ▼
Scheduled ingest workflow ──────► Local logs
     │
     ▼
Media storage ──────► Audiobookshelf
                          │
                          ├────► Separate config and metadata storage
                          │
                          └────► Private SMTP relay ───► Mail provider ───► Reader

Media + application state ──────► Separate backup storage
```

The application uses the upstream Audiobookshelf image. Ingestion and email adaptations sit around it rather than requiring a fork.

## Adapt the Compose example

The example is for a **new, separate deployment**. It is not a migration procedure for an existing library. It uses a media bind mount plus dedicated config and metadata volumes; this intentionally differs from the named media volume in the original reference setup.

1. Copy `examples/.env.example` to `examples/.env`.
2. Replace `REPLACE_WITH_RELEASE` with an image release you have chosen and reviewed. Use a versioned tag or digest so updates are deliberate.
3. Set `MEDIA_DIRECTORY` to the absolute path of your media directory. It must exist and be accessible to Docker. The example refuses to create a missing host directory.
4. Adjust the host port and timezone for your environment.
5. Validate before starting:

   ```sh
   docker compose --env-file examples/.env -f examples/compose.yaml config --quiet
   ```

6. After reviewing the resolved configuration, start the example:

   ```sh
   docker compose --env-file examples/.env -f examples/compose.yaml up -d
   ```

With the example port unchanged, open `http://127.0.0.1:13378` on the Docker host and complete the application’s initial setup. If that port is already used, choose another in your private `.env` file.

The example binds to loopback. Network access from other devices or a remote connection requires a separately reviewed routing and authentication setup. It does not configure a tunnel, email, ingestion, or backups. Do not point a second writable deployment at an existing library or application-state volume without planning the migration.

## Ingestion and macOS permissions

The original workflow checked an `_incoming` directory every five minutes, merged incoming content into the library without overwriting existing files, and removed source copies after successful transfer. That describes the intended workflow; its implementation is not distributed here.

During setup, background tasks encountered macOS permission restrictions around removable storage. The approach used a LaunchAgent to invoke an app wrapper, with appropriate filesystem access granted to that wrapper. File operations then ran in the configured container storage context.

This is an observation from one environment, not a blanket statement that macOS background processes cannot access external drives. The result depends on the macOS version, responsible process, permissions, mount layout, and Docker file-sharing configuration. A container does not automatically bypass host permissions or gain access to an attached disk.

If you implement your own ingest job, verify duplicate handling, partial-copy recovery, and destination integrity before deleting source files. Test with disposable sample files first.

## Logging

The reference setup kept a local operational log and attempted to mirror it to external storage. A failed mirror was intended to leave ingestion running while retaining the local log.

Choose paths and retention for your environment. Logs can include filenames, filesystem paths, or personal library details, so they belong outside the repository. The Compose example does not configure log mirroring.

## Backup and recovery

The original notes described monthly compressed media snapshots, two retained media archives, and a separate copy of metadata backups. Those settings describe that setup; they are not a recommended retention policy for every library and have not been reverified as current by this guide.

A useful recovery plan accounts for:

- Media files and any external datasets they depend on.
- Application config, metadata, and database state.
- The image version and configuration needed to reconstruct the service.
- A separate backup destination and a tested restore procedure.

Use a consistent application/database backup method. A live volume archive alone does not establish database consistency. Validate a restore in an isolated environment before relying on a backup, and choose frequency and retention based on how much change you can afford to lose.

## Email integration

The reference setup used a private relay to rewrite the subject to `Convert` for ebook messages addressed to Kindle recipients, then forwarded them through Gmail using STARTTLS. This was an integration workaround used in that environment, not a claim that every Kindle delivery or Audiobookshelf release requires it.

The relay is not included in this repository. If you build a similar integration, check the current requirements of your application, mail provider, and receiving service. Sender authorization, supported file types, and conversion behavior may vary.

Keep real SMTP credentials, app passwords, and recipient addresses in private runtime configuration. The public Compose example does not request or embed email credentials.

## Public documentation and privacy

Hardware descriptions are intentionally shared. Actual account identifiers, credentials, library contents, databases, logs, and runtime configuration are not. See [PUBLICATION.md](PUBLICATION.md) for the release scope and checks.

This is an independent personal homelab guide, not the official Audiobookshelf project. For application support and current image/release documentation, use the [upstream project](https://github.com/advplyr/audiobookshelf).
