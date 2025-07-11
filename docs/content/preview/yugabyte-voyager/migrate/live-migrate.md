---
title: Steps to perform live migration of your database using YugabyteDB Voyager
headerTitle: Live migration
linkTitle: Live migration
headcontent: Steps for a live migration using YugabyteDB Voyager
description: Run the steps to ensure a successful live migration using YugabyteDB Voyager.
menu:
  preview_yugabyte-voyager:
    identifier: migrate-live
    parent: migration-types
    weight: 103
tags:
  feature: tech-preview
type: docs
---

The following instructions describe the steps to perform and verify a successful live migration to YugabyteDB, including changes that continuously occur on the source.

## Live migration workflow

The following workflows illustrate how you can perform data migration including changes happening on the source simultaneously. With the export data command, you can first export a snapshot and then start continuously capturing changes occurring on the source to an event queue on the disk. Using the import data command, you similarly import the snapshot first, and then continuously apply the exported change events on the target.

Eventually, the migration process reaches a steady state where you can [cutover to the target database](#cutover-to-the-target). You can stop your applications from pointing to your source database, let all the remaining changes be applied on the target YugabyteDB database, and then restart your applications pointing to YugabyteDB.

The following illustration describes how the data export and import operations are simultaneously handled by YugabyteDB Voyager.

![Live migration short](/images/migrate/live-migration-short-new.png)

The following illustration shows the steps in a live migration using YugabyteDB Voyager.

![Live migration workflow](/images/migrate/live-migration-workflow-new.png)

| Phase | Step | Description |
| :---- | :--- | :---|
| PREPARE |[Install voyager](../../install-yb-voyager/#install-yb-voyager) | yb-voyager supports RHEL, CentOS, Ubuntu, and macOS, as well as airgapped and Docker-based installations. |
| | [Prepare&nbsp;source DB](#prepare-the-source-database) | Create a new database user with READ access to all the resources to be migrated. |
| | [Prepare target DB](#prepare-the-target-database) | Deploy a YugabyteDB database and create a user with necessary privileges. |
| ASSESS | [Assess Migration](#assess-migration) | Assess the migration complexity, and get schema changes, data distribution, and cluster sizing recommendations using the `yb-voyager assess-migration` command. |
| SCHEMA | [Export](#export-schema) | Convert the database schema to PostgreSQL format using the `yb-voyager export schema` command. |
| | [Analyze](#analyze-schema) | Generate a _Schema&nbsp;Analysis&nbsp;Report_ using the `yb-voyager analyze-schema` command. The report suggests changes to the PostgreSQL schema to make it appropriate for YugabyteDB. |
| | [Modify](#manually-edit-the-schema) | Using the report recommendations, manually change the exported schema. |
| | [Import](#import-schema) | Import the modified schema to the target YugabyteDB database using the `yb-voyager import schema` command. |
| LIVE&nbsp;MIGRATION | Start | Start the phases: export data first, followed by import data and archive changes simultaneously. |
| | [Export data](#export-data-from-source) | The export data command first exports a snapshot and then starts continuously capturing changes from the source.|
| | [Import data](#import-data-to-target) | The import data command first imports the snapshot, and then continuously applies the exported change events on the target. |
| | [Archive changes](#archive-changes-optional) | Continuously archive migration changes to limit disk utilization. |
| CUTOVER | [Initiate cutover](#cutover-to-the-target) | Perform a cutover (stop streaming changes) when the migration process reaches a steady state where you can stop your applications from pointing to your source database, allow all the remaining changes to be applied on the target YugabyteDB database, and then restart your applications pointing to YugabyteDB. |
| | [Wait for cutover to complete](#cutover-to-the-target) | Monitor the wait status using the [cutover status](../../reference/cutover-archive/cutover/#cutover-status) command. |
| | [Verify target DB](#verify-migration) | Check if the live migration is successful. |
| END | [End migration](#end-migration) | Clean up the migration information stored in export directory and databases (source and target). |

Before proceeding with migration, ensure that you have completed the following steps:

- [Install yb-voyager](../../install-yb-voyager/#install-yb-voyager).
- Review the [guidelines for your migration](../../known-issues/).
- Review [data modeling](../../../develop/data-modeling/) strategies.
- [Prepare the source database](#prepare-the-source-database).
- [Prepare the target database](#prepare-the-target-database).

## Prepare the source database

Create a new database user, and assign the necessary user permissions.

<ul class="nav nav-tabs-alt nav-tabs-yb custom-tabs">
  <li>
    <a href="#oracle" class="nav-link active" id="oracle-tab" data-bs-toggle="tab"
      role="tab" aria-controls="oracle" aria-selected="true">
      <i class="icon-oracle" aria-hidden="true"></i>
      Oracle
    </a>
  </li>
  <li >
    <a href="#pg" class="nav-link" id="pg-tab" data-bs-toggle="tab"
      role="tab" aria-controls="pg" aria-selected="false">
      <i class="icon-postgres" aria-hidden="true"></i>
      PostgreSQL
    </a>
  </li>
</ul>
<div class="tab-content">
  <div id="oracle" class="tab-pane fade show active" role="tabpanel" aria-labelledby="oracle-tab">

{{< tabpane text=true >}}

  {{% tab header="Standalone Oracle Container Database" %}}

1. Ensure that your database log_mode is `archivelog` as follows:

    ```sql
    SELECT LOG_MODE FROM V$DATABASE;
    ```

    ```output
    LOG_MODE
    ------------
    ARCHIVELOG
    ```

    If log_mode is NOARCHIVELOG (that is, not enabled), run the following command:

    ```sql
    sqlplus /nolog
    SQL>alter system set db_recovery_file_dest_size = 10G;
    SQL>alter system set db_recovery_file_dest = '<oracle_path>/oradata/recovery_area' scope=spfile;
    SQL> connect / as sysdba
    SQL> Shutdown immediate
    SQL> Startup mount
    SQL> Alter database archivelog;
    SQL> Alter database open;
    ```

1. Create the tablespaces as follows:

    1. Connect to Pluggable database (PDB) as sysdba and run the following command:

        ```sql
        CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/ORCLPDB1/logminer_tbs.dbf'
          SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
        ```

    1. Connect to Container database (CDB) as sysdba and run the following command:

        ```sql
        CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/logminer_tbs.dbf'
          SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
        ```

1. Run the following commands from CDB as sysdba:

    ```sql
    CREATE USER c##ybvoyager IDENTIFIED BY password
      DEFAULT TABLESPACE logminer_tbs
      QUOTA UNLIMITED ON logminer_tbs
      CONTAINER=ALL;

    GRANT CREATE SESSION TO c##ybvoyager CONTAINER=ALL;
    GRANT SET CONTAINER TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$DATABASE to c##ybvoyager CONTAINER=ALL;
    GRANT FLASHBACK ANY TABLE TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ANY TABLE TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT_CATALOG_ROLE TO c##ybvoyager CONTAINER=ALL;
    GRANT EXECUTE_CATALOG_ROLE TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ANY TRANSACTION TO c##ybvoyager CONTAINER=ALL;
    GRANT LOGMINING TO c##ybvoyager CONTAINER=ALL;

    GRANT CREATE TABLE TO c##ybvoyager CONTAINER=ALL;
    GRANT LOCK ANY TABLE TO c##ybvoyager CONTAINER=ALL;
    GRANT CREATE SEQUENCE TO c##ybvoyager CONTAINER=ALL;

    GRANT EXECUTE ON DBMS_LOGMNR TO c##ybvoyager CONTAINER=ALL;
    GRANT EXECUTE ON DBMS_LOGMNR_D TO c##ybvoyager CONTAINER=ALL;

    GRANT SELECT ON V_$LOG TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$LOG_HISTORY TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$LOGMNR_LOGS TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$LOGMNR_PARAMETERS TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$LOGFILE TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$ARCHIVED_LOG TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$TRANSACTION TO c##ybvoyager CONTAINER=ALL;

    GRANT SELECT ON V_$MYSTAT TO c##ybvoyager CONTAINER=ALL;
    GRANT SELECT ON V_$STATNAME TO c##ybvoyager CONTAINER=ALL;
    ```

1. Enable supplemental logging in the database as follows:

    ```sql
    ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
    ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
    ```

  {{% /tab %}}

  {{% tab header="RDS Oracle" %}}

**Note** that the following steps assume you're using SQL*Plus or a compatible Oracle client that supports `EXEC`. If your client doesn't support `EXEC`, use the standard SQL CALL syntax instead.

1. Ensure that your database log_mode is `archivelog` as follows:

    ```sql
    SELECT LOG_MODE FROM V$DATABASE;
    ```

    ```output
    LOG_MODE
    ------------
    ARCHIVELOG
    ```

    If log_mode is NOARCHIVELOG (that is, not enabled), run the following command:

    ```sql
    exec rdsadmin.rdsadmin_util.set_configuration('archivelog retention hours',24);
    ```

1. Connect to your database as an admin user, and create the tablespaces as follows:

    ```sql
    CREATE TABLESPACE logminer_tbs DATAFILE SIZE 25M AUTOEXTEND ON MAXSIZE UNLIMITED;
    ```

1. Run the following commands connected to the admin or privileged user:

    ```sql
    CREATE USER ybvoyager IDENTIFIED BY password
      DEFAULT TABLESPACE logminer_tbs
      QUOTA UNLIMITED ON logminer_tbs;

    GRANT CREATE SESSION TO YBVOYAGER;
    begin rdsadmin.rdsadmin_util.grant_sys_object(
          p_obj_name  => 'V_$DATABASE',
          p_grantee   => 'YBVOYAGER',
          p_privilege => 'SELECT');
    end;
    /

    GRANT FLASHBACK ANY TABLE TO YBVOYAGER;
    GRANT SELECT ANY TABLE TO YBVOYAGER;
    GRANT SELECT_CATALOG_ROLE TO YBVOYAGER;
    GRANT EXECUTE_CATALOG_ROLE TO YBVOYAGER;
    GRANT SELECT ANY TRANSACTION TO YBVOYAGER;
    GRANT LOGMINING TO YBVOYAGER;

    GRANT CREATE TABLE TO YBVOYAGER;
    GRANT LOCK ANY TABLE TO YBVOYAGER;
    GRANT CREATE SEQUENCE TO YBVOYAGER;


    begin rdsadmin.rdsadmin_util.grant_sys_object(
          p_obj_name => 'DBMS_LOGMNR',
          p_grantee => 'YBVOYAGER',
          p_privilege => 'EXECUTE',
          p_grant_option => true);
    end;
    /

    begin rdsadmin.rdsadmin_util.grant_sys_object(
          p_obj_name => 'DBMS_LOGMNR_D',
          p_grantee => 'YBVOYAGER',
          p_privilege => 'EXECUTE',
          p_grant_option => true);
    end;
    /

    begin rdsadmin.rdsadmin_util.grant_sys_object(
          p_obj_name  => 'V_$LOG',
          p_grantee   => 'YBVOYAGER',
          p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$LOG_HISTORY',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$LOGMNR_LOGS',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$LOGMNR_CONTENTS',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$LOGMNR_PARAMETERS',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$LOGFILE',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$ARCHIVED_LOG',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$ARCHIVE_DEST_STATUS',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$TRANSACTION',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$MYSTAT',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /

    begin
        rdsadmin.rdsadmin_util.grant_sys_object(
            p_obj_name  => 'V_$STATNAME',
            p_grantee   => 'YBVOYAGER',
            p_privilege => 'SELECT');
    end;
    /
    ```

1. Enable supplemental logging in the database as follows:

    ```sql
    exec rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD');

    begin
        rdsadmin.rdsadmin_util.alter_supplemental_logging(
            p_action => 'ADD',
            p_type   => 'ALL');
    end;
    /
    ```

  {{% /tab %}}

{{< /tabpane >}}

If you want yb-voyager to connect to the source database over SSL, refer to [SSL Connectivity](../../reference/yb-voyager-cli/#ssl-connectivity).

{{< note title="Connecting to Oracle instances" >}}

You can use only one of the following arguments in the `source` parameter (configuration file) or CLI flag to connect to your Oracle instance:

| `source` section parameters (configuration file)  | CLI Flag | Description |
|---|---|---|
|`db-schema`|`--source-db-schema`|Schema name of the source database.|
|`oracle-db-sid`|`--oracle-db-sid`|Oracle System Identifier you can use while exporting data from Oracle instances.|
|`oracle-tns-alias`|`--oracle-tns-alias`|TNS (Transparent Network Substrate) alias configured to establish a secure connection with the server.|

{{< /note >}}

  </div>
  <div id="pg" class="tab-pane fade" role="tabpanel" aria-labelledby="pg-tab">

{{< tabpane text=true >}}

  {{% tab header="Standalone PostgreSQL" %}}

1. yb_voyager requires `wal_level` to be logical. You can check this using following the steps:

    1. Run the command `SHOW wal_level` on the database to check the value.

    1. If the value is anything other than logical, run the command `SHOW config_file` to know the path of your configuration file.

    1. Modify the configuration file by uncommenting the parameter `wal_level` and set the value to logical.

    1. Restart PostgreSQL.

1. Create user `ybvoyager` for the migration using the following command:

    ```sql
    CREATE USER ybvoyager PASSWORD 'password';
    ```

1. Grant permissions for migration. Use the `yb-voyager-pg-grant-migration-permissions.sql` script (in `/opt/yb-voyager/guardrails-scripts/` or, for brew, check in `$(brew --cellar)/yb-voyager@<voyagerversion>/<voyagerversion>`) to grant the required permissions as follows:

    _Warning_: This script transfers ownership of all tables in the specified schemas to the specified replication group. The migration user and the original owner of the tables will be added to the replication group.

    ```sql
    psql -h <host> \
          -d <database> \
          -U <username> \ # A superuser or a privileged user with enough permissions to grant privileges
          -v voyager_user='ybvoyager' \
          -v schema_list='<comma_separated_schema_list>' \
          -v is_live_migration=1 \
          -v is_live_migration_fall_back=0 \
          -v replication_group='<replication_group>' \
          -f <path_to_the_script>
    ```

    The `ybvoyager` user can now be used for migration.

  {{% /tab %}}

  {{% tab header="RDS PostgreSQL" %}}

1. yb_voyager requires `wal_level` to be logical. This is controlled by a database parameter `rds.logical_replication` which needs to be set to 1. You can check this using following the steps:

    1. Run the command `SHOW rds.logical_replication` on the database to check whether the parameter is set.

    1. If the parameter is not set, you can change the parameter value to 1 from the RDS console of the database; navigate to **Configuration** > **Parameter group** > `rds.logical_replication`.

    1. If the `rds.logical_replication` errors out (after the change), create a new parameter group with the value as 1, and assign it to the database instance from the **Modify** option on the RDS console.

    1. Restart RDS.

1. Create user `ybvoyager` for the migration using the following command:

    ```sql
    CREATE USER ybvoyager PASSWORD 'password';
    ```

1. Grant permissions for migration. Use the `yb-voyager-pg-grant-migration-permissions.sql` script (in `/opt/yb-voyager/guardrails-scripts/` or, for brew, check in `$(brew --cellar)/yb-voyager@<voyagerversion>/<voyagerversion>`) to grant the required permissions as follows:

    _Warning_: This script transfers ownership of all tables in the specified schemas to the specified replication group. The migration user and the original owner of the tables will be added to the replication group.

    ```sql
    psql -h <host> \
          -d <database> \
          -U <username> \ # A superuser or a privileged user with enough permissions to grant privileges
          -v voyager_user='ybvoyager' \
          -v schema_list='<comma_separated_schema_list>' \
          -v is_live_migration=1 \
          -v is_live_migration_fall_back=0 \
          -v replication_group='<replication_group>' \
          -f <path_to_the_script>
    ```

    The `ybvoyager` user can now be used for migration.

  {{% /tab %}}

{{< /tabpane >}}

If you want yb-voyager to connect to the source database over SSL, refer to [SSL Connectivity](../../reference/yb-voyager-cli/#ssl-connectivity).

</div>

## Prepare the target database

Prepare your target YugabyteDB database cluster by creating a database, and a user for your cluster.

{{<note title="Important">}}

Add the following flags to the cluster before starting migration, and revert them after the migration is complete.

For the target YugabyteDB versions `2.18.5.1` and `2.18.6.0` (or later minor versions), set the following flag:

```sh
ysql_pg_conf_csv = yb_max_query_layer_retries=0
```

For all the other target YugabyteDB versions, set the following flags:

```sh
ysql_max_read_restart_attempts = 0
ysql_max_write_restart_attempts = 0
```

Turn off the [read-committed](../../../explore/transactions/isolation-levels/#read-committed-isolation) isolation level on the target YugabyteDB cluster during the migration.

{{</note>}}

### Create the target database

Create the target YugabyteDB database in your YugabyteDB cluster. The database name can be the same or different from the source database name.

If you don't provide the target YugabyteDB database name during import, yb-voyager assumes the target YugabyteDB database name is `yugabyte`. To specify the target YugabyteDB database name during import, use the `db-name`  parameter under the `target` section of the config file or  `--target-db-name` CLI flag with the yb-voyager import commands

```sql
CREATE DATABASE target_db_name;
```

### Create a user

Create a user with [`SUPERUSER`](../../../api/ysql/the-sql-language/statements/dcl_create_role/#syntax) role.

- For a local YugabyteDB cluster or YugabyteDB Anywhere, create a user and role with the superuser privileges using the following command:

     ```sql
     CREATE USER ybvoyager SUPERUSER PASSWORD 'password';
     ```

- For YugabyteDB Aeon, create a user with [`yb_superuser`](../../../yugabyte-cloud/cloud-secure-clusters/cloud-users/#admin-and-yb-superuser) role using the following command:

     ```sql
     CREATE USER ybvoyager PASSWORD 'password';
     GRANT yb_superuser TO ybvoyager;
     ```

If you want yb-voyager to connect to the target YugabyteDB database over SSL, refer to [SSL Connectivity](../../reference/yb-voyager-cli/#ssl-connectivity).

## Create an export directory

yb-voyager keeps all of its migration state, including exported schema and data, in a local directory called the _export directory_.

Before starting migration, you should create the export directory on a file system that has enough space to keep the entire source database. Ideally, this export directory should be placed inside a parent folder named after your migration for better organization. Next, you should provide the path to the export directory using the mandatory parameter `export-dir` (configuration file) or `--export-dir` flag (CLI) with each invocation of the yb-voyager command.

```sh
mkdir -p $HOME/<migration-name>/export-dir
```

The export directory has the following sub-directories and files:

- `reports` directory contains the generated _Schema Analysis Report_.
- `schema` directory contains the source database schema translated to PostgreSQL. The schema is partitioned into smaller files by the schema object type such as tables, views, and so on.
- `data` directory contains CSV (Comma Separated Values) files that are passed to the COPY command on the target YugabyteDB database.
- `metainfo` and `temp` directories are used by yb-voyager for internal bookkeeping.
- `logs` directory contains the log files for each command.

## Set up a configuration file

You can use a [configuration file](../../reference/configuration-file/) to specify the parameters required when running Voyager commands (v2025.6.2 or later).

To get started, copy the `live-migration.yaml` template configuration file from one of the following locations to the migration folder you created (for example, `$HOME/my-migration/`):

{{< tabpane text=true >}}

  {{% tab header="Linux (apt/yum/airgapped)" lang="linux" %}}

```bash
/opt/yb-voyager/config-templates/live-migration.yaml
```

  {{% /tab %}}

  {{% tab header="MacOS (Homebrew)" lang="macos" %}}

```bash
$(brew --cellar)/yb-voyager@<voyager-version>/<voyager-version>/config-templates/live-migration.yaml
```

Replace `<voyager-version>` with your installed Voyager version, for example, `2025.5.2`.

  {{% /tab %}}

{{< /tabpane >}}

Set the export-dir, source, and target arguments in the configuration file:

```yaml
# Replace the argument values with those applicable for your migration.

export-dir: <absolute-path-to-export-dir>

source:
  db-type: <source-db-type>
  db-host: <source-db-host>
  db-port: <source-db-port>
  db-name: <source-db-name>
  db-schema: <source-db-schema> # Not applicable for MySQL
  db-user: <source-db-user>
  db-password: <source-db-password> # Enclose the password in single quotes if it contains special characters.

target:
  db-host: <target-db-host>
  db-port: <target-db-port>
  db-name: <target-db-name>
  db-schema: <target-db-schema> # MySQL and Oracle only
  db-user: <target-db-username>
  db-password: <target-db-password> # Enclose the password in single quotes if it contains special characters.
```

Refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template for more information on the available global, source, and target configuration parameters supported by Voyager.

## Assess migration

This step is optional and only applicable to PostgreSQL and Oracle database migrations. Assess migration analyzes the source database, captures essential metadata, and generates a report with recommended migration strategies and cluster configurations for optimal performance with YugabyteDB. You run assessments using the `yb-voyager assess-migration` command.

Refer to [Migration assessment](../../migrate/assess-migration/) for details.

## Migrate your database to YugabyteDB

Proceed with schema and data migration using the following steps:

### Export and analyze schema

To begin, export the schema from the source database. Once exported, analyze the schema and apply any necessary manual changes.

#### Export schema

The `yb-voyager export schema` command extracts the schema from the source database, converts it into PostgreSQL format (if the source database is Oracle or MySQL), and dumps the SQL DDL files in the `EXPORT_DIR/schema/*` directories.

The `db-schema` key inside the `source` section parameters (configuration file), or the `--source-db-schema` flag (CLI), is used to specify the schema(s) to migrate from the source database.

For Oracle, `source-db-schema` (CLI) or `db-schema` (configuration file) can take only one schema name and you can migrate _only one_ schema at a time.

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager export schema --config-file <path-to-config-file>
```

You can specify additional `export schema` parameters in the `export-schema` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager export schema --export-dir <EXPORT_DIR> \
        --source-db-type <SOURCE_DB_TYPE> \
        --source-db-host <SOURCE_DB_HOST> \
        --source-db-user <SOURCE_DB_USER> \
        --source-db-password <SOURCE_DB_PASSWORD> \ # Enclose the password in single quotes if it contains special characters.
        --source-db-name <SOURCE_DB_NAME> \
        --source-db-schema <SOURCE_DB_SCHEMA> # Not applicable for MySQL
```

  {{% /tab %}}

{{< /tabpane >}}

Refer to [export schema](../../reference/schema-migration/export-schema/) for more information on the use of the command.

Note that if the source database is PostgreSQL and you haven't already run `assess-migration`, the schema is also assessed and a migration assessment report is generated.

#### Analyze schema

The schema exported in the previous step may not yet be suitable for importing into YugabyteDB. Even though YugabyteDB is PostgreSQL compatible, given its distributed nature, you may need to make minor manual changes to the schema.

The `yb-voyager analyze-schema` command analyses the PostgreSQL schema dumped in the [export schema](#export-schema) step, and prepares a report that lists the DDL statements which need manual changes.

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

Add output format argument to the config file:

```yaml
...
analyze-schema:
  output-format: <FORMAT>
...
```

Then run the command as follows:

```sh
yb-voyager analyze-schema --config-file <path-to-config-file>
```

You can specify additional `analyze-schema` parameters in the `analyze-schema` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager analyze-schema --export-dir <EXPORT_DIR> --output-format <FORMAT>
```

  {{% /tab %}}

{{< /tabpane >}}

The preceding command generates a report file under the `EXPORT_DIR/reports/` directory.

Refer to [analyze schema](../../reference/schema-migration/analyze-schema/) for more information on the use of the command.

#### Manually edit the schema

Fix all the issues listed in the generated schema analysis report by manually editing the SQL DDL files from the `EXPORT_DIR/schema/*`.

After making the manual changes, re-run the `yb-voyager analyze-schema` command. This generates a fresh report using your changes. Repeat these steps until the generated report contains no issues.

To learn more about modelling strategies using YugabyteDB, refer to [Data modeling](../../../develop/data-modeling/).

{{< note title="Manual schema changes" >}}

Include the primary key definition in the `CREATE TABLE` statement. Primary Key cannot be added to a partitioned table using the `ALTER TABLE` statement.

{{< /note >}}

Refer to the [Manual review guideline](../../known-issues/) for a detailed list of limitations and suggested workarounds associated with the source databases when migrating to YugabyteDB Voyager.

### Import schema

Import the schema using the `yb-voyager import schema` command.

The `db-schema` key inside the `target` section parameters (configuration file), or the `--target-db-schema` flag (CLI), is used to specify the schema in the target YugabyteDB database where the source schema will be imported.`yb-voyager` imports the source database into the `public` schema of the target YugabyteDB database. By specifying this argument during import, you can instruct `yb-voyager` to create a non-public schema and use it for the schema/data import.

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager import schema --config-file <path-to-config-file>
```

You can specify additional `import schema` parameters in the `import-schema` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager import schema --export-dir <EXPORT_DIR> \
        --target-db-host <TARGET_DB_HOST> \
        --target-db-user <TARGET_DB_USER> \
        --target-db-password <TARGET_DB_PASSWORD> \ # Enclose the password in single quotes if it contains special characters..
        --target-db-name <TARGET_DB_NAME> \
        --target-db-schema <TARGET_DB_SCHEMA>
```

  {{% /tab %}}

{{< /tabpane >}}

Refer to [import schema](../../reference/schema-migration/import-schema/) for more information on the use of the command.

{{< note title="NOT VALID constraints are not imported" >}}

Currently, `import schema` does not import NOT VALID constraints exported from source, because this could lead to constraint violation errors during the import if the source contains the data that is violating the constraint.

To add the constraints back, you run the `finalize-schema-post-data-import` command after data import. See [Cutover to the target](#cutover-to-the-target).

{{< /note >}}

yb-voyager applies the DDL SQL files located in the `schema` sub-directory of the [export directory](#create-an-export-directory) to the target YugabyteDB database. If `yb-voyager` terminates before it imports the entire schema, you can rerun it by adding the `ignore-exist` argument (configuration file), or using the `--ignore-exist` flag (CLI).

### Export data from source

Begin exporting data from the source database into the `EXPORT_DIR/data` directory using the yb-voyager export data from source command:

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager export data from source --config-file <path-to-config-file>
```

You can specify additional `export data` parameters in the `export-data` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager export data from source --export-dir <EXPORT_DIR> \
        --source-db-type <SOURCE_DB_TYPE> \
        --source-db-host <SOURCE_DB_HOST> \
        --source-db-user <SOURCE_DB_USER> \
        --source-db-password <SOURCE_DB_PASSWORD> \ # Enclose the password in single quotes if it contains special characters.
        --source-db-name <SOURCE_DB_NAME> \
        --source-db-schema <SOURCE_DB_SCHEMA> \
        --export-type snapshot-and-changes
```

  {{% /tab %}}

{{< /tabpane >}}

{{< note title="PostgreSQL and parallel jobs" >}}
For PostgreSQL, make sure that no other processes are running on the source database that can try to take locks; with more than one parallel job, Voyager will not be able to take locks to dump the data.
{{< /note >}}

{{< note title= "Migrating source databases with large row sizes" >}}
If a table's row size on the source database is too large, and exceeds the default [RPC message size](../../../reference/configuration/all-flags-yb-master/#rpc-max-message-size), import data will fail with the error `ERROR: Sending too long RPC message..`. So, you need to migrate those tables separately after removing the large rows.
{{< /note >}}

The export data command first ensures that it exports a snapshot of the data already present on the source database. Next, you start a streaming phase (CDC phase) where you begin capturing new changes made to the data on the source after the migration has started. Some important metrics such as the number of events, export rate, and so on, is displayed during the CDC phase similar to the following:

```output
| ---------------------------------------  |  ----------------------------- |
| Metric                                   |                          Value |
| ---------------------------------------  |  ----------------------------- |
| Total Exported Events                    |                         123456 |
| Total Exported Events (Current Run)      |                         123456 |
| Export Rate(Last 3 min)                  |                      22133/sec |
| Export Rate(Last 10 min)                 |                      21011/sec |
| ---------------------------------------  |  ----------------------------- |
```

Note that the CDC phase will start only after a snapshot of the entire table-set is completed.
Additionally, the CDC phase is restartable. So, if yb-voyager terminates when data export is in progress, it resumes from its current state after the CDC phase is restarted.

{{<note title="Important">}}
yb-voyager creates a replication slot in the source database where disk space can be used up rapidly. To avoid this, execute the [Cutover to the target](#cutover-to-the-target) or [End Migration](#end-migration) steps to delete the replication slot. If you choose to skip these steps, then you must delete the replication slot manually to reduce disk usage.
{{</note>}}

#### Caveats

- Some data types are unsupported. For a detailed list, refer to [datatype mappings](../../reference/datatype-mapping-oracle/).
- For Oracle where sequences are not attached to a column, resume value generation is unsupported.
- `parallel-jobs` config parameter under the `export-data-from-source` section or the `--parallel-jobs` CLI argument (specifies the number of tables to be exported in parallel from the source database at a time) has no effect on live migration.

Refer to [export data](../../reference/data-migration/export-data/#export-data) for more information on the use of the command.

The options passed to the command are similar to the [`yb-voyager export schema`](#export-schema) command. To export only a subset of tables, provide a comma-separated list of table names using the `table-list` argument (configuration file), or pass it via the `--table-list` flag (CLI).

{{<note title="Table list cannot be changed during migration resumption">}}
In any resumption scenario using the `export data` command, altering the list of tables to be migrated is not allowed. The command will result in an error if the table set is modified, thereby preventing unnecessary failure.
{{</note>}}

#### get data-migration-report

To get a consolidated report of the overall progress of data migration concerning all the databases involved (source and target), you can run the `yb-voyager get data-migration-report` command. You specify the `<EXPORT_DIR>` to push data in using `export-dir` parameter (configuration file), or `--export-dir` flag (CLI).

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager get data-migration-report --config-file <path-to-config-file>
```

You can specify additional `get data-migration-report` parameters in the `get-data-migration-report` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
 yb-voyager get data-migration-report --export-dir <EXPORT_DIR>
```

{{% /tab %}}

{{< /tabpane >}}

Refer to [get data-migration-report](../../reference/data-migration/export-data/#get-data-migration-report) for more information.

### Import data to target

After you have successfully imported the schema in the target YugabyteDB database, you can start importing the data using the yb-voyager import data to target command:

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager import data to target --config-file <path-to-config-file>
```

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager import data to target --export-dir <EXPORT_DIR> \
        --target-db-host <TARGET_DB_HOST> \
        --target-db-user <TARGET_DB_USER> \
        --target-db-password <TARGET_DB_PASSWORD> \ # Enclose the password in single quotes if it contains special characters.
        --target-db-name <TARGET_DB_NAME> \
        --target-db-schema <TARGET_DB_SCHEMA> \ # MySQL and Oracle only.
```

  {{% /tab %}}

{{< /tabpane >}}

Refer to [import data](../../reference/data-migration/import-data/) for more information.

For the snapshot exported, yb-voyager splits the data dump files (from the $EXPORT_DIR/data directory) into smaller batches. yb-voyager concurrently ingests the batches such that all nodes of the target YugabyteDB database cluster are used. After the snapshot is imported, a similar approach is employed for the CDC phase, where concurrent batches of change events are applied on the target YugabyteDB database cluster.

Some important metrics such as the number of events, ingestion rate, and so on, is displayed during the CDC phase similar to the following:

```output
| -----------------------------  |  ----------------------------- |
| Metric                         |                          Value |
| -----------------------------  |  ----------------------------- |
| Total Imported events          |                         272572 |
| Events Imported in this Run    |                         272572 |
| Ingestion Rate (last 3 mins)   |               14542 events/sec |
| Ingestion Rate (last 10 mins)  |               14542 events/sec |
| Time taken in this Run         |                      0.83 mins |
| Remaining Events               |                        4727427 |
| Estimated Time to catch up     |                          5m42s |
| -----------------------------  |  ----------------------------- |
```

The entire import process is designed to be *restartable* if yb-voyager terminates when the data import is in progress. If restarted, the data import resumes from its current state.

{{< note title="Note">}}
The arguments `table-list` and `exclude-table-list` are not supported in live migration.
For details about the arguments, refer to the [arguments table](../../reference/data-migration/import-data/#arguments).
{{< /note >}}

{{< tip title="Importing large datasets" >}}

When importing a very large database, run the import data command in a `screen` session, so that the import is not terminated when the terminal session stops.

If the `yb-voyager import data to target` command terminates before completing the data ingestion, you can re-run it with the same arguments and the command will resume the data import operation.

{{< /tip >}}

##### Migrating Oracle source databases with large row sizes

When migrating from Oracle source, when the snapshot import process, the default row size limit for data import is 32MB. If a row exceeds this limit but is smaller than the `batch-size * max-row-size`, you can increase the limit for the import data process by setting the `csv-reader-max-buffer-size-bytes` parameter in the `import-data-to-target` (configuration file) or export the environment variable `CSV_READER_MAX_BUFFER_SIZE_BYTES` with the value.

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```yaml
import-data-to-target:
  csv-reader-max-buffer-size-bytes: <MAX_ROW_SIZE_IN_BYTES>
```

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
export CSV_READER_MAX_BUFFER_SIZE_BYTES = <MAX_ROW_SIZE_IN_BYTES>
```

{{< /tab >}}

{{< /tabpane >}}

#### get data-migration-report

To get a consolidated report of the overall progress of data migration concerning all the databases involved (source or target), you can run the `yb-voyager get data-migration-report` command. You specify the `<EXPORT_DIR>` to push data in using `export-dir` parameter (configuration file), or `--export-dir` flag (CLI).

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager get data-migration-report --config-file <path-to-config-file>
```

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager get data-migration-report --export-dir <EXPORT_DIR> --target-db-password <TARGET_DB_PASSWORD>
```

  {{% /tab %}}

{{< /tabpane >}}

Refer to [get data-migration-report](../../reference/data-migration/import-data/#get-data-migration-report) for more information.

### Archive changes (Optional)

As the migration continuously exports changes on the source database to the `EXPORT-DIR`, disk use continues to grow. To prevent the disk from filling up, you can optionally use the `archive changes` command:


Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

```sh
yb-voyager archive changes --config-file <path-to-config-file>
```

You can specify additional `archive changes` parameters in the `archive-changes` section of the configuration file. For more details, refer to the [live-migration.yaml](https://github.com/yugabyte/yb-voyager/blob/{{< yb-voyager-release >}}/yb-voyager/config-templates/live-migration.yaml) template.

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager archive changes --export-dir <EXPORT-DIR> --move-to <DESTINATION-DIR>
```

  {{% /tab %}}

{{< /tabpane >}}

Refer to [archive changes](../../reference/cutover-archive/archive-changes/) for more information.

### Cutover to the target

Cutover is the last phase, where you switch your application over from the source database to the target YugabyteDB database.

Keep monitoring the metrics displayed for export data and import data processes. After you notice that the import of events is catching up to the exported events, you are ready to perform a cutover. You can use the "Remaining events" metric displayed in the import data process to help you determine the cutover.

Perform the following steps as part of the cutover process:

1. Quiesce your source database, that is stop application writes.
1. Perform a cutover after the exported events rate ("Export rate" in the metrics table) drops to 0 using `cutover to target` command (CLI) or using the configuration file.

    <br/>{{< tabpane text=true >}}

{{% tab header="Config file" lang="config" %}}

```sh
yb-voyager initiate cutover to target --config-file <path-to-config-file>
```

{{% /tab %}}

{{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager initiate cutover to target --export-dir <EXPORT_DIR> --prepare-for-fall-back false
```

{{% /tab %}}

    {{< /tabpane >}}

    Refer to [cutover to target](../../reference/cutover-archive/cutover/#cutover-to-target) for more information.

    The initiate cutover to target command stops the `export data from source` phase. After this, the `import data to target` phase continues and completes by importing all the exported events into the target YugabyteDB database.

1. Wait for the cutover process to complete. Monitor the status of the cutover process using the `cutover status` command (CLI) or configuration file.

    <br/>{{< tabpane text=true >}}

{{% tab header="Config file" lang="config" %}}

```sh
yb-voyager cutover status --config-file <path-to-config-file>
```

{{% /tab %}}

{{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager cutover status --export-dir <EXPORT_DIR>
```

{{% /tab %}}

    {{< /tabpane >}}

    Refer to [cutover status](../../reference/cutover-archive/cutover/#cutover-status) for more information.

1. If there are any NOT VALID constraints on the source, create them after the import data command is completed by using the `finalize-schema-post-data-import` command. If there are [Materialized views](../../../explore/ysql-language-features/advanced-features/views/#materialized-views) in the target YugabyteDB database, you can refresh them by setting the `refresh-mviews` parameter in the `finalize-schema-post-data-import` (configuration file) or use `--refresh-mviews` flag (CLI) with the value true. Run the command as follows:

    <br/>{{< tabpane text=true >}}

{{% tab header="Config file" lang="config" %}}

```sh
yb-voyager finalize-schema-post-data-import --config-file <path-to-config-file>
```

{{% /tab %}}

{{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager finalize-schema-post-data-import --export-dir <EXPORT_DIR> \
    --target-db-host <TARGET_DB_HOST> \
    --target-db-user <TARGET_DB_USER> \
    --target-db-password <TARGET_DB_PASSWORD> \ # Enclose the password in single quotes if it contains special characters.
    --target-db-name <TARGET_DB_NAME> \
    --target-db-schema <TARGET_DB_SCHEMA> \ # MySQL and Oracle only

```

{{% /tab %}}

    {{< /tabpane >}}


     {{< note title ="Note" >}}
The `import schema --post-snapshot-import` command is deprecated. Use [finalize-schema-post-data-import](../../reference/schema-migration/finalize-schema-post-data-import/) instead.
    {{< /note >}}

### Verify migration

After the schema and data import is complete, the automated part of the database migration process is considered complete. You should manually run validation queries on both the source and target YugabyteDB database to ensure that the data is correctly migrated. A sample query to validate the databases can include checking the row count of each table.

{{< warning title = "Caveat associated with rows reported by get data-migration-report" >}}

Suppose you have a scenario where,

- [import data](#import-data-to-target) or [import data file](../bulk-data-load/#import-data-files-from-the-local-disk) command fails.
- To resolve this issue, you delete some of the rows from the split files.
- After retrying, the import data command completes successfully.

In this scenario, [get data-migration-report](#get-data-migration-report) command reports an incorrect number of imported rows, because it doesn't take into account the deleted rows.

For more details, refer to GitHub issue [#360](https://github.com/yugabyte/yb-voyager/issues/360).

{{< /warning >}}

## End migration

To complete the migration, you need to clean up the export directory (export-dir), and Voyager state ( Voyager-related metadata) stored in the target YugabyteDB database.


The `yb-voyager end migration` command performs the cleanup, and backs up the schema, data, migration reports, and log files by providing the backup related arguments.

Run the command as follows:

{{< tabpane text=true >}}

  {{% tab header="Config file" lang="config" %}}

Specify the following parameters in the `end-migration` section of the configuration file:

```yaml
...
end-migration:
  backup-schema-files: <true, false, yes, no, 1, 0>
  backup-data-files: <true, false, yes, no, 1, 0>
  save-migration-reports: <true, false, yes, no, 1, 0>
  backup-log-files: <true, false, yes, no, 1, 0>
  # Set optional argument to store a back up of any of the above arguments.
  backup-dir: <BACKUP_DIR>
...
```

Run the command:

```sh
yb-voyager end migration --config-file <path-to-config-file>
```

  {{% /tab %}}

  {{% tab header="CLI" lang="cli" %}}

```sh
# Replace the argument values with those applicable for your migration.
yb-voyager end migration --export-dir <EXPORT_DIR> \
        --backup-log-files <true, false, yes, no, 1, 0> \
        --backup-data-files <true, false, yes, no, 1, 0> \
        --backup-schema-files <true, false, yes, no, 1, 0> \
        --save-migration-reports <true, false, yes, no, 1, 0> \
        # Set optional argument to store a back up of any of the above arguments.
        --backup-dir <BACKUP_DIR>
```

  {{% /tab %}}

{{< /tabpane >}}

Note that after you end the migration, you will *not* be able to continue further.

If you want to back up the schema, data, log files, and the migration reports (`analyze-schema` report, `get data-migration-report` output) for future reference, use the `backup-dir` argument (configuration file) or `--backup-dir` flag (CLI), and provide the path of the directory where the backup content needs to be saved (based on what you choose to back up).

Refer to [end migration](../../reference/end-migration/) for more information.

### Delete the ybvoyager user (Optional)

After migration, all the migrated objects (tables, views, and so on) are owned by the `ybvoyager` user. Transfer the ownership of the objects to some other user (for example, `yugabyte`) and then delete the `ybvoyager` user. For example, do the following:

```sql
REASSIGN OWNED BY ybvoyager TO yugabyte;
DROP OWNED BY ybvoyager;
DROP USER ybvoyager;
```

## Limitations

- Schema changes on the source database will not be recognized during the live migration.
- Adding or deleting partitions of a partitioned table is not supported during the live migration.
- Tables without primary key are not supported.
- Truncating a table on the source database is not taken into account; you need to manually truncate tables on your YugabyteDB cluster.
- Some Oracle data types are unsupported - User Defined Types (UDT), NCHAR, NVARCHAR, VARRAY, BLOB, CLOB, and NCLOB.
- Case-sensitive table names or column names are partially supported. YugabyteDB Voyager converts them to case-insensitive names. For example, an "Accounts" table in a source Oracle database is migrated as `accounts` (case-insensitive) to a YugabyteDB database.
- For Oracle source databases, schema, table, and column names with more than 30 characters are not supported.
- Sequences that are not associated with any column or are attached to columns of non-integer types are not supported for resuming value generation. These sequences must be manually resumed during the cutover phase.
- For Oracle, only the values of identity columns on the migrating tables will be restored. The user will have to resume other sequences manually.
