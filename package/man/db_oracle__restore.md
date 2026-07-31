#### Description

The `restore` command loads an Oracle Data Pump dump file back into a database by invoking the `impdp` (Data Pump Import) system CLI. It is the counterpart of the `backup` provider command and uses the same connection and directory model.

This command is contributed by the `aux4/db-oracle-backup` package, which extends the `db:oracle` profile owned by `aux4/db-oracle`. It is packaged separately so the `expdp`/`impdp` system dependency is only required when backup/restore is actually used.

**Server-side operation.** Like `expdp`, the `impdp` utility runs on the Oracle server and reads dump files from an **Oracle DIRECTORY**. The `--dir` parameter is an Oracle DIRECTORY object name (e.g. `DATA_PUMP_DIR`), and `--file` is the dump file name within that directory. The dump file must already exist on the server.

Connection details come from a `config.yaml` profile (via `--config`/`--configFile`). The dump file location is specified as:

- **`--dir <directory>` + `--file <name>`** — Oracle DIRECTORY name and dump file name (recommended).
- **`--path <dir/file>`** — shorthand, split on the last `/` into `--dir` and `--file`.

If neither `--dir`/`--file` nor `--path` is provided, the command fails fast with a clear error and a non-zero exit code.

**Import options.** How the import behaves is controlled by variables:

| Option | Default | impdp parameter |
|--------|---------|-----------------|
| `--remapSchema` | — | `remap_schema=OLD:NEW` |
| `--remapTablespace` | — | `remap_tablespace=OLD:NEW` |
| `--content` | `all` | `content=` |
| `--tableExistsAction` | `replace` | `table_exists_action=` |
| `--logFile` | `import.log` | `logfile=` |
| `--options` | — | appended verbatim |

The default `tableExistsAction` is `replace`, which drops and recreates existing tables during import. Use `skip` to leave existing tables untouched, `append` to insert rows into existing tables, or `truncate` to empty existing tables before loading.

On success `restore` prints a small outcome JSON to stdout:

```json
{
  "path": "<dir>/<file>",
  "database": "<serviceName>",
  "status": "success",
  "action": "restore"
}
```

**Binary resolution:** the command locates `impdp` via `command -v impdp`. If not found on `PATH`, it fails with an install hint. Oracle Instant Client is typically installed manually — ensure the Oracle client `bin` directory is on your `PATH`.

**System dependency:** requires the `impdp` utility from Oracle Instant Client tools or a full Oracle Database installation.

#### Usage

```bash
aux4 db oracle restore --configFile config.yaml --config <profile> --dir <directory> --file <name>
aux4 db oracle restore --configFile config.yaml --config <profile> --path <dir/file>
```

--host              Database host (default: localhost)
--port              Database port (default: 1521)
--database          Database service name, used when --serviceName is not set (default: XEPDB1)
--user              Database user (default: system)
--password          Database password
--serviceName       Oracle service name (takes precedence over --database)
--dir               Oracle DIRECTORY object name on the server (e.g. DATA_PUMP_DIR)
--file              Dump file name within the Oracle directory (e.g. mydb.dmp)
--path              Shorthand for dir/file, split on the last slash
--remapSchema       Remap schema during import (OLD:NEW)
--remapTablespace   Remap tablespace during import (OLD:NEW)
--content           What to include: all, data_only, or metadata_only
--tableExistsAction Action when table exists: skip, append, truncate, or replace

Configuration file:

```yaml
config:
  prod:
    host: 192.168.1.100
    port: 1521
    serviceName: ORCL
    user: system
    password: secret
```

#### Example

```bash
aux4 db oracle restore --configFile config.yaml --config prod --dir DATA_PUMP_DIR --file hr_export.dmp
```

```text
{"path":"DATA_PUMP_DIR/hr_export.dmp","database":"ORCL","status":"success","action":"restore"}
```
