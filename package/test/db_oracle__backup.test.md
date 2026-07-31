# db oracle backup and restore

Provider commands (contributed by `aux4/db-oracle-backup`) that wrap Oracle Data
Pump `expdp`/`impdp`. Connection is supplied via `--config` (a `config.yaml`
profile) and the destination via `--dir` (Oracle DIRECTORY name) + `--file`
(dump file name). `backup` prints a result manifest to stdout; `restore` loads
the dump back into the database.

**Prerequisites:** These tests require a running Oracle Database instance with
Data Pump configured. The default Oracle DIRECTORY `DATA_PUMP_DIR` must exist
and the test user must have read/write access to it. A typical local setup uses
an Oracle XE container:

```
docker run -d --name oracle -p 1521:1521 -e ORACLE_PASSWORD=oracle gvenzl/oracle-free:23-slim
```

```file:config.yaml
config:
  test:
    host: 127.0.0.1
    port: 1521
    database: FREEPDB1
    user: system
    password: oracle
    dir: DATA_PUMP_DIR
  schemaonly:
    host: 127.0.0.1
    port: 1521
    database: FREEPDB1
    user: system
    password: oracle
    dir: DATA_PUMP_DIR
    content: metadata_only
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "BEGIN EXECUTE IMMEDIATE 'DROP TABLE bkptest_items'; EXCEPTION WHEN OTHERS THEN NULL; END;"
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "CREATE TABLE bkptest_items (id NUMBER PRIMARY KEY, name VARCHAR2(100), qty NUMBER)"
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "INSERT INTO bkptest_items VALUES (1, 'apple', 10)"
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "INSERT INTO bkptest_items VALUES (2, 'pear', 20)"
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "INSERT INTO bkptest_items VALUES (3, 'plum', 30)"
```

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "COMMIT"
```

```afterAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "BEGIN EXECUTE IMMEDIATE 'DROP TABLE bkptest_items'; EXCEPTION WHEN OTHERS THEN NULL; END;"
```

## backup with --dir and --file

### should export and print a manifest

```timeout
120000
```

```execute
aux4 db oracle backup --configFile config.yaml --config test --dir DATA_PUMP_DIR --file bkptest_export.dmp --schemas SYSTEM
```

```expect:regex
\{"format":"oracle-datapump","path":"DATA_PUMP_DIR/bkptest_export\.dmp","status":"success"\}
```

## backup with --path shorthand

### should split path into dir and file

```timeout
120000
```

```execute
aux4 db oracle backup --configFile config.yaml --config test --path DATA_PUMP_DIR/bkptest_path.dmp --schemas SYSTEM
```

```expect:regex
\{"format":"oracle-datapump","path":"DATA_PUMP_DIR/bkptest_path\.dmp","status":"success"\}
```

## backup auto-appends .dmp extension

### should add .dmp when the file name has no extension

```timeout
120000
```

```execute
aux4 db oracle backup --configFile config.yaml --config test --dir DATA_PUMP_DIR --file bkptest_noext --schemas SYSTEM
```

```expect:regex
\{"format":"oracle-datapump","path":"DATA_PUMP_DIR/bkptest_noext\.dmp","status":"success"\}
```

## backup with config-driven content option

### should accept content from the config profile

```timeout
120000
```

```execute
aux4 db oracle backup --configFile config.yaml --config schemaonly --file bkptest_meta.dmp --schemas SYSTEM
```

```expect:regex
\{"format":"oracle-datapump","path":"DATA_PUMP_DIR/bkptest_meta\.dmp","status":"success"\}
```

## backup with no dir or file

### should fail fast when neither dir/file nor path is given

```timeout
120000
```

```execute
aux4 db oracle backup --configFile config.yaml --config test
```

```error:partial
Error: provide --dir (Oracle directory name) and --file (dump file name)
```

## restore with --dir and --file

```beforeAll
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "BEGIN EXECUTE IMMEDIATE 'DROP TABLE bkptest_items'; EXCEPTION WHEN OTHERS THEN NULL; END;"
```

### should import the dump and print an outcome

```timeout
120000
```

```execute
aux4 db oracle restore --configFile config.yaml --config test --dir DATA_PUMP_DIR --file bkptest_export.dmp --tableExistsAction replace
```

```expect:regex
\{"action":"restore","database":"FREEPDB1","path":"DATA_PUMP_DIR/bkptest_export\.dmp","status":"success"\}
```

### should bring the rows back

```timeout
120000
```

```execute
aux4 db oracle execute --host 127.0.0.1 --port 1521 --database FREEPDB1 --user system --password oracle --query "SELECT * FROM bkptest_items ORDER BY id" | jq -c .
```

```expect
[{"ID":1,"NAME":"apple","QTY":10},{"ID":2,"NAME":"pear","QTY":20},{"ID":3,"NAME":"plum","QTY":30}]
```

## restore with remap

### should accept remapSchema option

```timeout
120000
```

```execute
aux4 db oracle restore --configFile config.yaml --config test --dir DATA_PUMP_DIR --file bkptest_export.dmp --remapSchema SYSTEM:SYSTEM --tableExistsAction replace
```

```expect:regex
\{"action":"restore","database":"FREEPDB1","path":"DATA_PUMP_DIR/bkptest_export\.dmp","status":"success"\}
```

## restore with no dir or file

### should fail fast when neither dir/file nor path is given

```timeout
120000
```

```execute
aux4 db oracle restore --configFile config.yaml --config test
```

```error:partial
Error: provide --dir (Oracle directory name) and --file (dump file name)
```
