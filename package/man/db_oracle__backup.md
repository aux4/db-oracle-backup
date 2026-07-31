#### Description

The `backup` command creates a logical export of an Oracle database by invoking the `expdp` (Data Pump Export) system CLI. It is a backup **provider**: connection details come from a `config.yaml` profile (via `--config`/`--configFile`) so credentials stay out of any catalog, and the destination is an Oracle DIRECTORY on the server.

This command is contributed by the `aux4/db-oracle-backup` package, which extends the `db:oracle` profile owned by `aux4/db-oracle`. It is packaged separately so the `expdp`/`impdp` system dependency is only required when backup/restore is actually used — the core `aux4/db-oracle` query commands need no external CLIs.

**Server-side operation.** Oracle Data Pump is fundamentally different from client-side tools like `mysqldump`. The `expdp` utility runs on the Oracle server and writes dump files to an **Oracle DIRECTORY** — a named server-side filesystem path. The `--dir` parameter is an Oracle DIRECTORY object name (e.g. `DATA_PUMP_DIR`), not a local directory. The `--file` parameter is the dump file name within that directory.

The destination is specified as:

- **`--dir <directory>` + `--file <name>`** — Oracle DIRECTORY name and dump file name (recommended).
- **`--path <dir/file>`** — shorthand, split on the last `/` into `--dir` and `--file`.

If neither `--dir`/`--file` nor `--path` is provided, the command fails fast with a clear error and a non-zero exit code. When the file name has no extension, `.dmp` is appended — this lets `aux4/backup` pass an extension-less base name and leave artifact naming to the provider, while an explicit `--file export.dmp` is used exactly as given.

**Export options.** What goes into the dump is controlled by variables, so they can be set once in the `config.yaml` profile (and overridden per run on the command line):

| Option | Default | expdp parameter |
|--------|---------|-----------------|
| `--schemas` | — | `schemas=` |
| `--tables` | — | `tables=` |
| `--full` | — | `full=y` |
| `--content` | `all` | `content=` |
| `--compression` | `all` | `compression=` |
| `--parallel` | — | `parallel=` |
| `--logFile` | `export.log` | `logfile=` |
| `--options` | — | appended verbatim |

After a successful export, `backup` prints a **result manifest** as a single line of JSON to stdout:

```json
{
  "path": "<dir>/<file>",
  "status": "success",
  "format": "oracle-datapump"
}
```

The manifest does not include `bytes` or `checksum` because the dump file lives on the Oracle server, not on the client machine. The manifest is consumed by the `aux4/backup` orchestrator to catalog each run.

**Binary resolution:** the command locates `expdp` via `command -v expdp`. If not found on `PATH`, it fails with an install hint. Oracle Instant Client is typically installed manually — ensure the Oracle client `bin` directory is on your `PATH`.

**System dependency:** requires the `expdp` utility from Oracle Instant Client tools or a full Oracle Database installation.

#### Usage

```bash
aux4 db oracle backup --configFile config.yaml --config <profile> --dir <directory> --file <name>
aux4 db oracle backup --configFile config.yaml --config <profile> --path <dir/file>
```

--host         Database host (default: localhost)
--port         Database port (default: 1521)
--database     Database service name, used when --serviceName is not set (default: XEPDB1)
--user         Database user (default: system)
--password     Database password
--serviceName  Oracle service name (takes precedence over --database)
--dir          Oracle DIRECTORY object name on the server (e.g. DATA_PUMP_DIR)
--file         Dump file name within the Oracle directory (e.g. mydb.dmp)
--path         Shorthand for dir/file, split on the last slash

Export options: --schemas, --tables, --full, --content, --compression, --parallel, --logFile, --options

Configuration file — export options live alongside the connection settings:

```yaml
config:
  prod:
    host: 192.168.1.100
    port: 1521
    serviceName: ORCL
    user: system
    password: secret
    schemas: HR
    compression: all
```

#### Example

```bash
aux4 db oracle backup --configFile config.yaml --config prod --dir DATA_PUMP_DIR --file hr_export.dmp --schemas HR
```

```text
{"path":"DATA_PUMP_DIR/hr_export.dmp","status":"success","format":"oracle-datapump"}
```
