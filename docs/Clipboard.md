# Sivnox — clipboard contract

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L13`, `R33`–`R36`,
`T7`–`T8`).
**Component:** `sivnoxd`, the only long-lived Rust daemon.
**Picker:** Quickshell/QML, edited from Ryoku.
**Source:** Ryoku clipboard behaviour, ported to Rust.

---

## 1. Scope

### 1.1 In scope

- Text clipboard history.
- Deduplication by content.
- Pins that survive eviction.
- Bounded storage: 100 entries / 1 MB by default.
- Oldest-unpinned eviction.
- A local picker socket.
- Copy-back via `wl-copy`.
- SQLite persistence across restarts.

### 1.2 Out of scope

- Image/binary clipboard rendering in v0.1. The `kind` column exists for later.
- Clipboard sync between machines.
- History export UI.
- A Rust picker UI.
- A second clipboard daemon.

---

## 2. Lifecycle

### 2.1 Start

- `sivnoxd.socket` listens at `$XDG_RUNTIME_DIR/sivnox/clipboard.sock`.
- The first picker connection activates `sivnoxd.service`.
- If socket activation is rejected at W0 (`P5`), the picker starts the daemon
  explicitly and the service file is adjusted.
- The daemon creates the runtime directory with mode `0700` if missing.
- Consequence: history begins when the service first starts. If the user wants
  continuous capture from login, they can enable `sivnoxd.service` at boot.
  This is the deliberate v0.1 trade for a lower idle footprint.

### 2.2 Stay running

- Keeps one SQLite connection in WAL mode.
- Supervises a `wl-paste --watch` child.
- Serves picker requests.
- Exit is explicit: `systemctl --user stop sivnoxd`, or a fatal database error.

### 2.3 Restart

- On crash, `systemd` restarts the daemon with backoff.
- The database is durable; no in-memory history is lost beyond the last write
  not yet committed.
- The watcher is restarted with backoff if it exits.

### 2.4 Shutdown

- Close the database cleanly.
- Terminate the watcher child.
- Remove the socket on clean exit; a stale socket is unlinked on start.

---

## 3. Storage

### 3.1 Path

```
~/.local/share/sivnox/clipboard.db
```

- Directory mode `0700`.
- Database mode `0600`.
- SQLite WAL, `busy_timeout = 5000`, `synchronous = NORMAL`.
- `PRAGMA user_version = 1`.

### 3.2 Schema

```sql
CREATE TABLE IF NOT EXISTS entries (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    hash        BLOB NOT NULL UNIQUE,   -- blake3 of content
    content     TEXT NOT NULL,
    kind        TEXT NOT NULL DEFAULT 'text',
    size        INTEGER NOT NULL,       -- bytes of content
    created_at  INTEGER NOT NULL,       -- unix ms
    last_used_at INTEGER NOT NULL,      -- unix ms, for eviction order
    pinned      INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS idx_entries_evict
    ON entries (pinned, last_used_at);

CREATE INDEX IF NOT EXISTS idx_entries_pinned
    ON entries (pinned, last_used_at DESC);
```

### 3.3 Deduplication

- On ingest, compute `blake3(content)`.
- If the hash exists, update `last_used_at` and keep one row.
- Do not create a duplicate.
- A pinned entry stays pinned when re-copied.

### 3.4 Limits

From `config.toml`:

```toml
[clipboard]
max_entries = 100
max_bytes = 1048576
pins_persist = true
```

- `max_entries`: maximum rows.
- `max_bytes`: maximum sum of `size`.
- `pins_persist = true`: pins are never evicted. If false, pins are treated as
  normal entries for eviction but remain marked pinned. The default is true.
- Limits are validated on read; invalid values fall back to defaults.

### 3.5 Eviction

When `count > max_entries` or `sum(size) > max_bytes`:

1. Select unpinned entries ordered by `last_used_at ASC` (oldest first).
2. Delete until both limits are satisfied.
3. Never delete a pinned entry when `pins_persist = true`.
4. If all entries are pinned and the limit is still exceeded, stop evicting and
   log a warning. The picker shows the over-limit state.

`T7` and `T8` are the gate.

### 3.6 Corruption and recovery

- On open, run `PRAGMA integrity_check`.
- If the database is corrupt, rename it to `clipboard.db.broken-<timestamp>`,
  create a new database, and log. Do not silently delete user data.
- A failed write is retried once; if it fails again, the daemon stays up and
  reports the error through `status`.

---

## 4. Watcher

- Child process: `wl-paste --watch --no-newline` (text type by default), so the
  stored value matches the clipboard exactly.
- The daemon reads each new clipboard value and ingests it.
- Ignore empty strings and values identical to the current row.
- Ignore a value that is already the active selection to avoid a feedback loop
  when `wl-copy` writes the same content.
- If `wl-paste` is missing, the daemon starts, serves history, and reports
  `watcher: unavailable`.
- If the watcher dies, restart with exponential backoff (1 s, 2 s, 4 s, max
  30 s).

---

## 5. Socket contract

### 5.1 Transport

- Path: `$XDG_RUNTIME_DIR/sivnox/clipboard.sock`.
- Mode `0600`.
- Unix stream socket.
- One JSON request per line, one JSON reply per line.
- The connection stays open for multiple requests; the picker may send several.

### 5.2 Request envelope

```json
{ "v": 1, "op": "list", "args": { "limit": 100 } }
```

### 5.3 Reply envelope

```json
{ "v": 1, "ok": true, "result": { } }
{ "v": 1, "ok": false, "error": { "code": "invalid_request", "message": "…" } }
```

### 5.4 Operations

| Op | Args | Result |
|---|---|---|
| `list` | `{ limit?, include_pinned? }` | `{ entries: [entry], total, pinned_count }` |
| `get` | `{ id }` | `{ entry }` |
| `select` | `{ id }` | `{ copied: true }` — copies via `wl-copy` |
| `pin` | `{ id }` | `{ entry }` |
| `unpin` | `{ id }` | `{ entry }` |
| `delete` | `{ id }` | `{ deleted: true }` |
| `clear_unpinned` | `{}` | `{ deleted: n }` |
| `status` | `{}` | `{ entries, bytes, pinned, max_entries, max_bytes, watcher }` |

`entry`:

```json
{
  "id": 12,
  "preview": "first 120 chars…",
  "size": 42,
  "kind": "text",
  "created_at": 1760000000000,
  "last_used_at": 1760000000000,
  "pinned": false
}
```

`get` and `select` may include full `content`; `list` uses `preview` only. The
picker asks for `get` when it needs the full text.

### 5.5 Errors

| Code | Meaning |
|---|---|
| `invalid_request` | malformed JSON or missing field |
| `not_found` | id does not exist |
| `db_error` | SQLite failure |
| `wl_copy_failed` | `wl-copy` failed |
| `unsupported` | op not implemented in this version |

### 5.6 Versioning

- The socket is local and short-lived; `v: 1` is enough.
- A request with an unknown `v` gets an `invalid_request` error.
- The daemon and picker ship together; no backward-compatibility promise beyond
  the current release.

---

## 6. Picker contract

The picker is QML (`R36`), edited from Ryoku.

### 6.1 Open

- `sivnox open clipboard` asks Quickshell to show the picker.
- The picker connects to the socket; socket activation starts `sivnoxd` if
  needed.
- The list is ordered pinned-first, then `last_used_at DESC`.

### 6.2 Display

- Show a one-line preview, size, and relative age.
- Pinned entries have a visible pin state, not only a colour.
- Over-limit state is visible.
- Empty state explains that copying text will populate the history.

### 6.3 Actions

- Select → `select` op → daemon copies with `wl-copy` → picker closes.
- Pin/unpin → daemon updates and returns the entry.
- Delete → daemon deletes and returns success.
- Clear unpinned → confirmation in the picker, then `clear_unpinned`.

### 6.4 Rules

- The picker never opens the SQLite file directly.
- The picker never writes clipboard content itself; `select` owns the copy.
- The picker unloads when closed (`Loader`).
- The picker does not poll; it requests once on open and updates after actions.

---

## 7. Security and privacy

- The database is plaintext, user-owned, mode `0600`.
- No network, no sync, no telemetry.
- No clipboard content is logged. Logs contain counts and error codes only.
- The socket is user-only; another user cannot connect.
- Secrets copied by the user are stored until evicted. The picker can delete an
  entry; `clear_unpinned` clears history.
- A future encrypted-store option is out of scope for v0.1.

---

## 8. Service units (`P5`)

```ini
# sivnoxd.socket
[Unit]
Description=Sivnox clipboard socket

[Socket]
ListenStream=%t/sivnox/clipboard.sock
SocketMode=0600
DirectoryMode=0700

[Install]
WantedBy=sockets.target
```

```ini
# sivnoxd.service
[Unit]
Description=Sivnox clipboard daemon
Requires=sivnoxd.socket

[Service]
ExecStart=%h/.local/bin/sivnoxd
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

`P5` confirms the unit names. `sivnoxd.socket` is the recommended way to satisfy
"start on first picker open."

---

## 9. Testing

| Test | Requirement |
|---|---|
| T7 | A pinned entry survives eviction. |
| T8 | At the limit, the oldest unpinned entry is dropped. |
| — | Dedup: copying the same text twice creates one row. |
| — | Restart: history and pins survive a daemon restart. |
| — | Corruption: a broken DB is renamed, not silently lost. |
| — | Socket: every op returns the documented envelope. |
| — | Watcher: a dead `wl-paste` restarts with backoff. |
| — | Privacy: no clipboard content appears in logs. |

Integration tests live in `sivnox-tests/`. The acceptance procedure is in
`Testing.md`.
