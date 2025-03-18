# [Schema guard](/sg1/)
The [declarative](https://github.com/vkrinitsyn/schema_guard) Rust based portion of the [project](https://www.dbinvent.com/rdbm/) discuss bellow.

Almost every website, from the simplest blogs to complex platforms, relies on databases. Of course, databases are no longer accessed directly from the code that generates the website, as was common in the past with PHP and JSP. Modern websites are typically structured in two or three layers:  
Static or pseudo-static CRM content, which either doesn’t use a database or interacts with it as a third-party service.  

## Why Schema Guard Matters

The core principle of Schema Guard is ensuring that only verified and approved changes reach production (deployed by version number and tag).
### Key aspects include:  
- Deployment Flow (CI/CD Process): Deployment and testing occur before production, ensuring stability.  
- No Manual Interference in Production: We avoid manual changes or rollbacks in production environments.  
- Admin Script Review for Production: All scripts are reviewed by an admin before deployment to production.  
- File Checksums: Checksums ensure that files haven’t been tampered with or substituted.
- Separation of Schema Modification Access.
Access to schema changes is separated between the user application and the utility:  
- Different Users for DDL and DML: DDL (Data Definition Language) access is restricted and unavailable to the user application, while DML (Data Manipulation Language) is handled separately.  

### Declarative (YAML Schema Definition) vs. Imperative (SQL):  
- Declarative Approach: Preferred during **development** for easier schema change management. However, it allows unrestricted modifications in local databases, with SQL generated dynamically based on the database state. The text-based YAML format enables convenient schema comparison.  
- Imperative Approach: Preferred in **production** for precise control over the exact SQL executed (plus checksum validation). The SQL deployed in development is guaranteed to match what’s deployed in production.


Dynamic frontend, which directly interfaces with:  
Backend or API, which handles direct database interactions for retrieving and persisting changes.
Data includes everything from login credentials, login attempts, and user profile preferences to other dynamic information. The first step in development is creating the database, defining tables, and populating them with initial data.
There are many ways to create database tables or entire schemas. Among the numerous approaches, several key methods stand out:  
Using features of the chosen programming language and utilities to generate SQL scripts from classes or objects used for data access. This is convenient when supported by the language, but such support is more often absent than present.  
Using ERP systems for graphical schema modeling and SQL generation. This is visually intuitive, but the generated scripts are limited by the database’s supported features. Not all database-specific capabilities can be generated, and manual script modifications prevent regeneration after schema changes.  
Writing SQL scripts manually. This is the simplest and most common approach, but also the most time-intensive.  
Using a declarative method, which generates the required SQL from a simple text-based format. This is the method we’ll explore next.

### Getting Started
Let’s begin:  

```bash
curl https://www.dbinvent.com/dist/rdbm-unix-latest.zip -o rdbm-latest.zip && unzip ./rdbm-latest.zip
```
This downloads a console application in binary format, ready to run on Linux or Windows (via WSL). The most convenient configuration method is creating and using a text file:
```bash
echo "db_host=localhost
db_name=postgres
db_port=5432
db_user=postgres
db_password=xxx" > db.cfg
```
If your database already contains tables, you can generate a schema snapshot:  
```bash
./rdbm -c db.cfg --snapshot_to=schema.yaml snapshot
```
Next, you can edit or create a schema description file in YAML format, such as:  
```yaml
# databaseChangeLog
---
database:
- schema: # changeSet
  schemaName: public
  owner: postgres
  tables: # set of changes for tables
    - table: # will do a createTable or alter table
      tableName: test_samples
      description: Sample table comments
      columns:
      - column: # column definition
        name: id
        type: serial
        constraint: # changes on PK & NULL won’t apply
          primaryKey: true
          nullable: false
      - column:
        name: is_deleted
        type: BOOLEAN
        defaultValue: "false"
        constraint:
          nullable: false
      - column:
        name: text_value
        description: "my column comments"
        type: VARCHAR(256)
```

This text represents a highly readable and user-friendly YAML format, though it’s sensitive to indentation and formatting.
Save the file as ```./S1__schema.yaml```. Then run:  
```bash
./rdbm -c db.cfg --source_dir=. --dry_run=y migrate
```

This attempts to connect to the database and generates scripts to create the table without executing them.
If the database doesn’t exist, the ```--createdb=y``` parameter will create it. Without ```--dry_run=y``` (or with ```--dry_run=n```),
the SQL executes, creating the table and logging the results in the ```public.rdbm_history``` history table. 
This table can be overridden, e.g., to deploy_history.deploys, using two parameters: ```--schema=deploy_history``` ```--table=deploys```.
Notably, re-running the command won’t alter the database, even if the history table is missing, but this applies only to scripts of this type.
For detailed help, run:  

```bash
./rdbm help migrate
```

Next, add to the script:  
```yaml
- column:
  name: created_on
  type: timestamp
  defaultValue: current_timestamp
```
Running the migration again adds the new column to the database. If the column is missing from an existing table, it’s added; if the table doesn’t exist, a new table is created with all specified columns.

###  Foreign Key Constraints
Relational database schemas often involve constraints. Here’s an example with an additional table:  
```yaml
   - table:
      tableName: test_ref
      columns:
      - column:
          name: id
          type: serial
          constraint:
            primaryKey: true
            nullable: false
      - column:
          name: sample
          type: INT
          constraint:
            nullable: false
            foreignKey:
              references: test_sample
              sql: DEFERABLE
```
The sql field is optional and defined in column_constraint per PostgreSQL [documentation](https://www.postgresql.org/docs/current/sql-createtable.html 
```text
(e.g., [ MATCH FULL | MATCH PARTIAL | MATCH SIMPLE ] [ ON DELETE referential_action ] [ ON UPDATE referential_action ] [ DEFERRABLE | NOT DEFERRABLE ] [ INITIALLY DEFERRED | INITIALLY IMMEDIATE ]).
```

Extended SQL at the Column Level
Example:  
```yaml
- column:
  name: name
  type: VARCHAR(50)
  sql: UNIQUE
```
The SQL format aligns with column_constraint from PostgreSQL’s CREATE TABLE [documentation](https://www.postgresql.org/docs/current/sql-createtable.html,
excluding ```NULL, PRIMARY KEY, and REFERENCES```.
Constraints and Extended SQL at the Table Level
Similar to columns, you can define SQL fragments for table creation:  
```yaml
- table:
  tableName: users
  constraint: UNIQUE (iap, oauth_id)
  sql: PARTITION BY ...
```
Table constraint: Matches table_constraint in PostgreSQL’s CREATE TABLE [documentation](https://www.postgresql.org/docs/current/sql-createtable.html:
```text
{ CHECK ( expression ) [ NO INHERIT ] | UNIQUE ( column_name [, ... ] ) index_parameters | EXCLUDE [ USING index_method ] ( ... ) index_parameters [ WHERE ( predicate ) ] }, 
```
excluding primary and foreign keys.  
Table SQL: Follows formats like:
```text
[ INHERITS ( parent_table [, ... ] ) ] [ PARTITION BY { RANGE | LIST | HASH } ( ... ) ] [ USING method ] [ WITH ( storage_parameter [= value] [, ... ] ) | WITHOUT OIDS ] [ ON COMMIT { PRESERVE ROWS | DELETE ROWS | DROP } ] [ TABLESPACE tablespace_name ].
```
Triggers
The final table component is trigger definition, which requires a function (function creation is covered in a next article):  
```yaml
- table:
  tableName: emails   
  triggers:
    - trigger:
      name: ensure_lower_email_trg
      event: before update or insert
      when: for each row
      proc: make_lower_email() -- you have to have this function created
```

> [!IMPORTANT]
> Current YAML Limitations
> - Adding or modifying sql or constraint fields in YAML for an existing table won’t trigger ALTER TABLE commands, except for foreign keys (but not additional fk.sql).  
> - Composite primary keys (multi-column) are unsupported, including foreign keys referencing such tables.

> [!IMPORTANT]
> Schema Snapshot Restrictions
> - When creating a snapshot of an existing database schema, the YAML includes:  
> Table and column names, types, foreign keys, defaults, and nullable constraints.
> However, it excludes:  
> - Extended table/column definitions (constraint & sql), indexes, procedures, and functions.


## [Part Two - ](/sg2/)