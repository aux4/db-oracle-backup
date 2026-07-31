# aux4/db-oracle-backup

Backup and restore provider commands for Oracle Database. This package extends
the `db:oracle` profile (owned by `aux4/db-oracle`) with `backup` and `restore`
commands that wrap Oracle Data Pump (`expdp` and `impdp`).

It is packaged **separately** from `aux4/db-oracle` on purpose: the core
`aux4/db-oracle` query commands (`execute`, `stream`) use the Node driver and
need no external binaries. Only backup/restore need the Data Pump CLIs, so that
system dependency is isolated here — install this package only when you need
dump/restore.

Because it declares the same `db:oracle` profile, its commands merge into the
existing profile: after installing both packages, `aux4 db oracle --help` shows
the query commands **and** `backup`/`restore` together.

## Important: server-side Data Pump model

Oracle Data Pump (`expdp`/`impdp`) is fundamentally different from client-side
tools like `mysqldump` or `pg_dump`. Data Pump runs **on the Oracle server**.
Dump files are written to and read from an **Oracle DIRECTORY** — a named
server-side filesystem path, not a local directory on the client machine.

- **`--dir`** maps to an Oracle DIRECTORY object name (e.g. `DATA_PUMP_DIR`),
  not a local filesystem path. The DBA creates directories with
  `CREATE DIRECTORY dir_name AS '/path/on/server'` and grants read/write access.
- **`--file`** is the dump file name within that directory (e.g. `mydb.dmp`).
- The `--path` flag is a shorthand that splits on the last `/` into `--dir` and
  `--file`, but using `--dir` + `--file` explicitly is recommended for clarity.

The default Oracle DIRECTORY `DATA_PUMP_DIR` exists in most installations and
typically points to `$ORACLE_BASE/admin/<SID>/dpdump/`.

## Installation

```bash
aux4 aux4 pkger install aux4/db-oracle-backup
```

This pulls in `aux4/db-oracle` as a dependency, so the full `db:oracle` profile
(query commands + backup/restore) is available.

## Configuration

Connection details are supplied through a `config.yaml` profile so credentials
stay out of any backup catalog:

```yaml
config:
  prod:
    host: 192.168.1.100
    port: 1521
    serviceName: ORCL
    user: system
    password: secret
```

Use it with `--configFile config.yaml --config prod`. Individual values can also
be passed directly with `--host`, `--port`, `--serviceName`, `--user`,
`--password`, which take precedence over config values. The `--database` flag is
used as a fallback when `--serviceName` is not set.

### Export options

What goes into the dump is controlled by the same mechanism, so options can be
set once per profile and overridden per run on the command line:

```yaml
config:
  prod:
    host: 192.168.1.100
    serviceName: ORCL
    user: system
    password: secret
    schemas: HR,FINANCE
    content: all
    compression: all
    parallel: 4
    options: "exclude=statistics"
```

| Option | Default | Effect |
|--------|---------|--------|
| `schemas` | — | Export specific schemas (`expdp schemas=`) |
| `tables` | — | Export specific tables as `SCHEMA.TABLE` (`expdp tables=`) |
| `full` | — | Full database export (`expdp full=y`) |
| `content` | `all` | What to include: `all`, `data_only`, or `metadata_only` |
| `compression` | `all` | Compression: `all`, `data_only`, `metadata_only`, or `none` |
| `parallel` | — | Degree of parallelism |
| `logFile` | `export.log` | Log file name within the Oracle directory |
| `options` | — | Additional raw `expdp` options, appended verbatim |

## backup

Creates a logical export of an Oracle database using `expdp`.

The destination is specified as:

- **`--dir <directory>` + `--file <name>`** — Oracle DIRECTORY name and dump
  file name (recommended).
- **`--path <dir/file>`** — shorthand, split on the last `/`.

If neither is given, the command fails fast. When the file name has no
extension, `.dmp` is appended.

On success it prints a result manifest as a single line of JSON to stdout:

```bash
aux4 db oracle backup --configFile config.yaml --config prod --dir DATA_PUMP_DIR --file hr_export.dmp --schemas HR
```

```text
{"path":"DATA_PUMP_DIR/hr_export.dmp","status":"success","format":"oracle-datapump"}
```

The manifest does **not** include `bytes` or `checksum` because the dump file
lives on the Oracle server, not on the client machine.

## restore

Loads a Data Pump dump file back into an Oracle database using `impdp`. The dump
file must already exist in the specified Oracle DIRECTORY on the server.

```bash
aux4 db oracle restore --configFile config.yaml --config prod --dir DATA_PUMP_DIR --file hr_export.dmp
```

```text
{"path":"DATA_PUMP_DIR/hr_export.dmp","database":"ORCL","status":"success","action":"restore"}
```

### Import options

| Option | Default | Effect |
|--------|---------|--------|
| `remapSchema` | — | Remap schema during import (`impdp remap_schema=OLD:NEW`) |
| `remapTablespace` | — | Remap tablespace during import (`impdp remap_tablespace=OLD:NEW`) |
| `content` | `all` | What to include: `all`, `data_only`, or `metadata_only` |
| `tableExistsAction` | `replace` | Action when table exists: `skip`, `append`, `truncate`, or `replace` |
| `logFile` | `import.log` | Log file name within the Oracle directory |
| `options` | — | Additional raw `impdp` options, appended verbatim |

## Connection string

Both commands build the connection string as:

```
user/password@host:port/serviceName
```

If `--serviceName` is not set, `--database` is used as the fallback (the
`aux4/db-oracle` package uses `database` as the service name field).

## Binary resolution

Both commands locate the `expdp`/`impdp` binaries via `command -v`. If the
binary is not on `PATH`, the command fails with an install hint. Unlike MySQL
where Homebrew provides a keg-only fallback, Oracle Instant Client is typically
installed manually — ensure the Oracle client `bin` directory is on your `PATH`.

## System dependency

Requires Oracle Data Pump tools (`expdp` and `impdp`):

- **Linux:** Install `oracle-instantclient-tools` from the Oracle YUM/APT repo.
- **macOS:** Download Oracle Instant Client from oracle.com and add the `bin`
  directory to your `PATH`.
- **Docker:** Use an Oracle Database image (e.g. `gvenzl/oracle-xe`) which
  includes Data Pump by default.
