# Oracle JDBC Catalog Design

---

## 1. Background and Goals

### 1.1 Background

Gravitino already supports MySQL, PostgreSQL, Doris, StarRocks, and ClickHouse as JDBC catalogs. Oracle is one of the most widely used relational databases in enterprise settings, so adding Oracle support will cover many more users.

This document focuses on **Oracle 11g Release 2** (11.2.x). Oracle 12c+ added CDB/PDB multi-tenant architecture and Identity Columns (auto-increment), but 11g does not have those features. Many enterprises still run 11g, so it needs special handling.

### 1.2 Goals

- Implement Oracle as a Gravitino JDBC catalog
- Support Schema (User) and Table CRUD operations
- Implement two-way type and default value conversion between Oracle and Gravitino
- Handle Oracle 11g specific limits (no AUTO_INCREMENT, 30-char identifier limit, empty string equals NULL, etc.)
- Clearly document which Gravitino features are not supported in Oracle

---

## 2. Key Differences Between Oracle and Other Databases

Oracle differs from MySQL and PostgreSQL in several important ways:

| Topic | MySQL | PostgreSQL | Oracle 11g |
|-------|-------|-----------|------------|
| Schema concept | DATABASE = Schema | SCHEMA lives inside DATABASE | **User = Schema**. A schema is the set of all objects owned by a user |
| Create schema | `CREATE DATABASE` | `CREATE SCHEMA` | `CREATE USER ... IDENTIFIED BY ...` |
| Drop schema | `DROP DATABASE` | `DROP SCHEMA CASCADE` | `DROP USER CASCADE` |
| AUTO_INCREMENT | Native support | `SERIAL` / `IDENTITY` | **Not supported** (need SEQUENCE + TRIGGER; IDENTITY added in 12c) |
| Identifier case | Case-insensitive (stored lowercase) | Case-insensitive (stored lowercase) | **Case-insensitive (stored UPPERCASE)** |
| Max identifier length | 64 bytes | 63 bytes | **30 bytes** (strict limit in 11g) |
| Empty string | `''` is different from NULL | `''` is different from NULL | **`''` is the same as NULL** |
| String type | `VARCHAR` | `VARCHAR` | `VARCHAR2` (recommended) / `VARCHAR` (avoid) |
| DATE type | Date only | Date only | **Date + time** (seconds precision) |
| COMMENT syntax | Inline in `CREATE TABLE` | Separate `COMMENT ON ... IS '...'` statement | Separate `COMMENT ON ... IS '...'` statement |
| JDBC URL | `jdbc:mysql://host/db` | `jdbc:postgresql://host/db` | `jdbc:oracle:thin:@host:port:SID` or `@//host:port/service` |
| Partitioning | Basic | Basic | **Rich** (Range, List, Hash, Composite, Interval, Reference) |

---

## 3. Module Layout

### 3.1 Directory Structure

```
catalogs-contrib/catalog-jdbc-oracle/
├── build.gradle.kts
└── src/
    ├── main/java/org/apache/gravitino/catalog/oracle/
    │   ├── OracleCatalog.java
    │   ├── OracleCatalogCapability.java
    │   ├── OracleTablePropertiesMetadata.java
    │   ├── converter/
    │   │   ├── OracleTypeConverter.java
    │   │   ├── OracleColumnDefaultValueConverter.java
    │   │   └── OracleExceptionConverter.java
    │   └── operation/
    │       ├── OracleDatabaseOperations.java
    │       └── OracleTableOperations.java
    └── test/
        ├── java/.../catalog/oracle/
        │   ├── operation/
        │   │   ├── TestOracleDatabaseOperations.java   (unit tests)
        │   │   └── TestOracleTableOperations.java      (unit tests)
        │   └── OracleCatalogIT.java                    (integration tests, requires Docker)
        └── resources/
            └── oracle-init.sql
```

### 3.2 Class Hierarchy

```
JdbcCatalog
└── OracleCatalog
      ├── OracleDatabaseOperations  (extends JdbcDatabaseOperations)
      ├── OracleTableOperations     (extends JdbcTableOperations)
      ├── OracleTypeConverter       (extends JdbcTypeConverter)
      ├── OracleColumnDefaultValueConverter (extends JdbcColumnDefaultValueConverter)
      └── OracleExceptionConverter  (extends JdbcExceptionConverter)
```

---

## 4. Schema (User) Operations

### 4.1 Schema Equals User in Oracle

In Oracle, **Schema = User**. When a user is created, Oracle automatically creates a schema with the same name. All objects (tables, views, sequences, etc.) that the user creates belong to that schema.

This means:
- `CREATE SCHEMA` maps to `CREATE USER ... IDENTIFIED BY ...`
- `DROP SCHEMA` maps to `DROP USER ... CASCADE`
- `LIST SCHEMAS` queries `ALL_USERS` and filters out system users

### 4.2 Required Permissions

The Gravitino connection account needs these permissions:

```sql
-- Create and drop users
GRANT CREATE USER TO gravitino_admin;
GRANT DROP USER TO gravitino_admin;

-- Read system views
GRANT SELECT ON DBA_USERS TO gravitino_admin;
GRANT SELECT ON DBA_TABLES TO gravitino_admin;
GRANT SELECT ON DBA_TAB_COLUMNS TO gravitino_admin;
GRANT SELECT ON DBA_INDEXES TO gravitino_admin;
GRANT SELECT ON DBA_COMMENTS TO gravitino_admin;
```

If DBA-level permissions are not available, use `ALL_USERS` / `ALL_TABLES` views instead (no DBA permission required). However, `listDatabases()` will then only show schemas visible to the current user.

**Design decision**: The first version will use `ALL_USERS` / `ALL_TABLES` by default. Using `DBA_*` views can be an option added later.

### 4.3 System Schema Filter List

Oracle 11g ships with many built-in system users. These must be filtered out in `createSysDatabaseNameSet()`:

```java
private static final Set<String> ORACLE_SYSTEM_SCHEMAS = ImmutableSet.of(
    "SYS", "SYSTEM", "OUTLN", "DBA", "DBSNMP", "MGMT_VIEW",
    "SYSMAN", "ANONYMOUS", "APEX_030200", "APEX_PUBLIC_USER",
    "APPQOSSYS", "BI", "CTXSYS", "EXFSYS", "FLOWS_FILES",
    "HR", "IX", "MDDATA", "MDSYS", "OE", "OLAPSYS",
    "ORACLE_OCM", "ORDDATA", "ORDPLUGINS", "ORDSYS", "OWB",
    "OWBSYS", "OWBSYS_AUDIT", "PM", "SCOTT", "SH",
    "SPATIAL_CSW_ADMIN_USR", "SPATIAL_WFS_ADMIN_USR",
    "SYSBACKUP", "SYSDG", "SYSKM",
    "WKPROXY", "WKSYS", "WK_TEST", "WMSYS",
    "XDB", "XS$NULL"
);
```

### 4.4 Create Schema SQL

Oracle `CREATE USER` requires a password. This is a problem because Gravitino's schema creation interface has no password parameter.

**Options:**

1. **Random password** (more secure): Generate a random password using `UUID.randomUUID()`. The created user cannot log in directly (password is not exposed), and Gravitino manages its objects. This is the safest approach.

2. **Configurable default password**: Read a password from catalog properties, for example `oracle.new-schema.default-password`.

3. **Disallow schema creation** (most conservative): Only allow using existing schemas; do not support CREATE.

**First version recommendation**: Option 2. Catalog properties provide the default password, and the documentation explains the limitation.

```java
// OracleDatabaseOperations.generateCreateDatabaseSql()
@Override
protected String generateCreateDatabaseSql(
    String databaseName, String comment, Map<String, String> properties) {

  String password = properties.getOrDefault(
      OracleConstants.NEW_SCHEMA_PASSWORD, defaultPassword);
  if (StringUtils.isBlank(password)) {
    throw new IllegalArgumentException(
        "Oracle requires a password to create a schema (user). "
        + "Set 'oracle.new-schema.default-password' in catalog properties.");
  }

  // Uppercase the name to match Oracle's storage behavior
  return String.format(
      "CREATE USER \"%s\" IDENTIFIED BY \"%s\"",
      databaseName.toUpperCase(), password);
}
```

After the user is created, basic permissions must also be granted. This should be done inside the `create()` method (not `generateCreateDatabaseSql()`):

```java
@Override
public void create(String databaseName, String comment, Map<String, String> properties) {
  try (Connection conn = getConnection()) {
    JdbcConnectorUtils.executeUpdate(conn, generateCreateDatabaseSql(...));
    JdbcConnectorUtils.executeUpdate(conn,
        String.format("GRANT CREATE SESSION TO \"%s\"", databaseName.toUpperCase()));
    JdbcConnectorUtils.executeUpdate(conn,
        String.format("GRANT CREATE TABLE TO \"%s\"", databaseName.toUpperCase()));
    JdbcConnectorUtils.executeUpdate(conn,
        String.format("GRANT UNLIMITED TABLESPACE TO \"%s\"", databaseName.toUpperCase()));

    // Oracle has no COMMENT ON SCHEMA syntax, so schema-level comments are not stored.
  }
}
```

### 4.5 Drop Schema SQL

```sql
-- CASCADE removes all objects (tables, views, sequences, etc.) owned by the user
DROP USER "schema_name" CASCADE
```

```java
@Override
protected String generateDropDatabaseSql(String databaseName, boolean cascade) {
  // Always use CASCADE to match Gravitino's drop behavior
  return String.format("DROP USER \"%s\" CASCADE", databaseName.toUpperCase());
}
```

### 4.6 Schema Comment Support

Oracle does not have a `COMMENT ON SCHEMA` statement, and there is no system view that stores schema-level comments.

```java
@Override
protected boolean supportSchemaComment() {
  return false; // Oracle 11g has no schema-level comment support
}
```

### 4.7 listDatabases Implementation

```java
@Override
public List<String> listDatabases() {
  // Use ALL_USERS (no DBA permission needed)
  // Filter out system schemas before returning
  String sql = "SELECT USERNAME FROM ALL_USERS ORDER BY USERNAME";
  // ...
}
```

### 4.8 loadDatabase Implementation

Since Oracle stores no schema-level comment, always return an empty comment:

```java
@Override
public JdbcSchema load(String databaseName) throws NoSuchSchemaException {
  // Query ALL_USERS WHERE USERNAME = UPPER(databaseName)
  // Throw NoSuchSchemaException if not found
  return JdbcSchema.builder()
      .withName(databaseName)
      .withComment("")   // Oracle does not support schema comments
      .withProperties(ImmutableMap.of())
      .withAuditInfo(AuditInfo.EMPTY)
      .build();
}
```

---

## 5. Type Conversion

### 5.1 Deterministic Mapping Policy (No Ambiguous Fallbacks)

To keep behavior deterministic, this design uses strict rules:

1. If a type has an exact and stable Gravitino mapping, use it.
2. If exact mapping is not possible but raw Oracle semantics can be preserved, use `ExternalType` (preferred).
3. If neither is reliable for DDL generation, reject with `UnsupportedOperationException`.

No implicit “best effort” widening fallback is allowed in v1.

### 5.2 Oracle -> Gravitino Mapping (`toGravitino`)

| Oracle Type | Gravitino Type | Deterministic Rule |
|-------------|----------------|--------------------|
| `NUMBER(p)` (`s=0`) | `Byte` / `Short` / `Integer` / `Long` / `Decimal(p,0)` | Map by fixed precision thresholds only. |
| `NUMBER(p,s)` (`s>0`) | `Decimal(p,s)` | Exact mapping. |
| `NUMBER(p,s)` (`s<0`) | `ExternalType("NUMBER(p,s)")` | Negative scale is not representable in canonical Gravitino decimal mapping. |
| `NUMBER` (precision/scale unspecified) | `ExternalType("NUMBER")` | Do not guess as `Decimal(38,18)`. |
| `FLOAT(p)` | `DoubleType` | Exact in v1 policy. |
| `BINARY_FLOAT` | `FloatType` | Exact mapping. |
| `BINARY_DOUBLE` | `DoubleType` | Exact mapping. |
| `CHAR(n)` | `FixedCharType(n)` | Exact mapping. |
| `VARCHAR2(n)` / `VARCHAR(n)` | `VarCharType(n)` | Use length-preserving mapping. |
| `NCHAR(n)` | `ExternalType("NCHAR(n)")` | Keep national-character semantics explicitly. |
| `NVARCHAR2(n)` | `ExternalType("NVARCHAR2(n)")` | Keep national-character semantics explicitly. |
| `DATE` | `TimestampType.withoutTimeZone()` | Oracle DATE contains time (to seconds); no fractional precision, so no precision set. |
| `TIMESTAMP(p)` | `TimestampType.withoutTimeZone(p)` | Preserve fractional-second precision `p`. |
| `TIMESTAMP(p) WITH TIME ZONE` | `ExternalType("TIMESTAMP(p) WITH TIME ZONE")` | Preserve Oracle-specific semantics; do not collapse. |
| `TIMESTAMP(p) WITH LOCAL TIME ZONE` | `ExternalType("TIMESTAMP(p) WITH LOCAL TIME ZONE")` | Preserve Oracle-specific semantics; do not collapse. |
| `CLOB` / `NCLOB` | `StringType` | Exact in v1 policy. |
| `BLOB` / `RAW(n)` / `LONG RAW` | `BinaryType` | Exact in v1 policy. |
| `INTERVAL ...` / `ROWID` / `XMLTYPE` | `ExternalType(...)` | Preserve source type name; unsupported for canonical DDL. |

`NUMBER(p)` precision thresholds (strict):

- `1 <= p <= 3` -> `ByteType`
- `4 <= p <= 5` -> `ShortType`
- `6 <= p <= 10` -> `IntegerType`
- `11 <= p <= 18` -> `LongType`
- `p >= 19` -> `DecimalType(p, 0)`

#### 5.2.1 Why `NCHAR` / `NVARCHAR2` Must Use `ExternalType`

`NCHAR` and `NVARCHAR2` use Oracle national character semantics (`NLS_NCHAR_CHARACTERSET`), while `CHAR`/`VARCHAR2` use database character semantics (`NLS_CHARACTERSET`).  
They are not interchangeable in collation/charset behavior, so mapping them to Gravitino `VarCharType` would hide semantics. Therefore, v1 keeps them as `ExternalType`.

#### 5.2.2 DATE and TIMESTAMP Determinism

Oracle `DATE` and `TIMESTAMP(p)` map differently:

- `DATE` → `TimestampType.withoutTimeZone()` (no precision set, since Oracle DATE has only second-level precision with no fractional digits).
- `TIMESTAMP(p)` → `TimestampType.withoutTimeZone(p)` (precision `p` is preserved).

DDL generation rules:

- `TimestampType.withoutTimeZone()` (no precision set) **always** generates `TIMESTAMP(6)` (not `DATE`).
- `TimestampType.withoutTimeZone(p)` (precision set) generates `TIMESTAMP(p)`.
- If users require Oracle `DATE` exactly, they must use `ExternalType("DATE")`.

For timezone variants:

- `TIMESTAMP(p) WITH TIME ZONE` and `TIMESTAMP(p) WITH LOCAL TIME ZONE` are kept as `ExternalType` on read.
- `TimestampType.withTimeZone()` canonical output is `TIMESTAMP(6) WITH TIME ZONE`.
- To generate `... WITH LOCAL TIME ZONE`, use `ExternalType("TIMESTAMP(6) WITH LOCAL TIME ZONE")`.

### 5.3 Gravitino -> Oracle Mapping (`fromGravitino`)

Strict unsupported policy for `fromGravitino`:

- If a Gravitino type cannot be mapped to a deterministic Oracle type, throw `UnsupportedOperationException` directly.
- Do not silently coerce to another Oracle type.
- Suggest explicit alternatives via `ExternalType` in error messages when possible.

| Gravitino Type | Oracle DDL Type | Rule |
|----------------|-----------------|------|
| `ByteType` | `NUMBER(3)` | Canonical mapping. |
| `ShortType` | `NUMBER(5)` | Canonical mapping. |
| `IntegerType` | `NUMBER(10)` | Canonical mapping. |
| `LongType` | `NUMBER(19)` | Canonical mapping. |
| `DecimalType(p,s)` | `NUMBER(p,s)` | Exact mapping. |
| `FloatType` | `BINARY_FLOAT` | Canonical mapping. |
| `DoubleType` | `BINARY_DOUBLE` | Canonical mapping. |
| `BooleanType` | `NUMBER(1)` | Canonical v1 representation (0/1). |
| `StringType` | `CLOB` | Canonical mapping. |
| `VarCharType(n)` | `VARCHAR2(n)` | Canonical mapping. |
| `FixedCharType(n)` | `CHAR(n)` | Canonical mapping. |
| `TimestampType.withoutTimeZone()` (no precision) | `TIMESTAMP(6)` | Default canonical output when no precision is set (e.g. originates from Oracle `DATE`). |
| `TimestampType.withoutTimeZone(p)` (precision set) | `TIMESTAMP(p)` | Preserves fractional-second precision; round-trips `TIMESTAMP(p)` correctly. |
| `TimestampType.withTimeZone()` (no precision) | `TIMESTAMP(6) WITH TIME ZONE` | Default canonical output. |
| `TimestampType.withTimeZone(p)` (precision set) | `TIMESTAMP(p) WITH TIME ZONE` | Preserves fractional-second precision. |
| `BinaryType` | `BLOB` | Canonical mapping. |
| `DateType` | Unsupported in v1 canonical DDL | Throw unsupported; suggest `ExternalType("DATE")` if exact Oracle DATE is required. |
| `TimeType`, `ListType`, `MapType`, `StructType` | Unsupported | Throw `UnsupportedOperationException`. |

`ExternalType` pass-through policy in v1:

- If `ExternalType` string is in the Oracle allow-list (`DATE`, `NCHAR(n)`, `NVARCHAR2(n)`, `TIMESTAMP(...) WITH LOCAL TIME ZONE`, etc.), emit as-is.
- Otherwise reject with `UnsupportedOperationException` to avoid unsafe SQL generation.

---

## 6. Column Default Values

### 6.1 How Oracle Stores Default Values

The `COLUMN_DEF` field returned by `DatabaseMetaData.getColumns()` looks different from MySQL and PostgreSQL:

```
MySQL:   CURRENT_TIMESTAMP
PgSQL:   now()   or   'value'::type
Oracle:  SYSDATE   or   0   or   'text'   or   NULL
```

Oracle stores the raw SQL expression without any type-cast suffix.

### 6.2 Common Default Value Formats

| Oracle Default Value | Gravitino Expression |
|----------------------|----------------------|
| `NULL` | `Literals.NULL` |
| `0`, `1`, `42` | `Literals.integerLiteral(value)` |
| `3.14` | `Literals.decimalLiteral(value)` |
| `'hello'` | `Literals.stringLiteral("hello")` |
| `SYSDATE` | `FunctionExpression.of("SYSDATE")` |
| `SYSTIMESTAMP` | `FunctionExpression.of("SYSTIMESTAMP")` |
| `SYS_GUID()` | `FunctionExpression.of("SYS_GUID")` |
| `USER` | `FunctionExpression.of("USER")` |
| `CURRENT_DATE` | `FunctionExpression.of("CURRENT_DATE")` |
| `CURRENT_TIMESTAMP` | `FunctionExpression.of("CURRENT_TIMESTAMP")` |
| Any other expression | `UnparsedExpression` |

### 6.3 toGravitino Logic

```java
@Override
public Expression toGravitino(
    JdbcTypeConverter.JdbcTypeBean typeBean,
    String columnDefaultValue,
    boolean isExpression,
    boolean nullable) {

  if (columnDefaultValue == null) {
    return Column.DEFAULT_VALUE_NOT_SET;
  }

  // Oracle JDBC may include leading or trailing whitespace in COLUMN_DEF
  String defaultVal = columnDefaultValue.trim();

  if ("NULL".equalsIgnoreCase(defaultVal)) {
    return nullable ? Literals.NULL : Column.DEFAULT_VALUE_NOT_SET;
  }

  if ("SYSDATE".equalsIgnoreCase(defaultVal)) {
    return FunctionExpression.of("SYSDATE");
  }
  if ("SYSTIMESTAMP".equalsIgnoreCase(defaultVal)) {
    return FunctionExpression.of("SYSTIMESTAMP");
  }
  if ("CURRENT_TIMESTAMP".equalsIgnoreCase(defaultVal)) {
    return FunctionExpression.of("CURRENT_TIMESTAMP");
  }
  if ("CURRENT_DATE".equalsIgnoreCase(defaultVal)) {
    return FunctionExpression.of("CURRENT_DATE");
  }
  if (defaultVal.toUpperCase().startsWith("SYS_GUID")) {
    return FunctionExpression.of("SYS_GUID");
  }

  // String literal: Oracle format is 'value'
  if (defaultVal.startsWith("'") && defaultVal.endsWith("'")) {
    String content = defaultVal.substring(1, defaultVal.length() - 1)
                               .replace("''", "'"); // Oracle escapes single quote as ''
    return Literals.stringLiteral(content);
  }

  // Numeric literal
  String typeName = typeBean.getTypeName().toUpperCase();
  if ("NUMBER".equals(typeName) || "BINARY_FLOAT".equals(typeName)
      || "BINARY_DOUBLE".equals(typeName) || "FLOAT".equals(typeName)) {
    try {
      if (typeBean.getScale() > 0) {
        return Literals.decimalLiteral(new BigDecimal(defaultVal));
      }
      return Literals.longLiteral(Long.parseLong(defaultVal));
    } catch (NumberFormatException e) {
      // fall through
    }
  }

  return UnparsedExpression.of(defaultVal);
}
```

### 6.4 fromGravitino Logic

```java
@Override
public String fromGravitino(Expression defaultValue) {
  if (defaultValue == Column.DEFAULT_VALUE_NOT_SET) return null;
  if (defaultValue == Literals.NULL) return "NULL";

  if (defaultValue instanceof FunctionExpression) {
    String funcName = ((FunctionExpression) defaultValue).functionName().toUpperCase();
    switch (funcName) {
      case "SYSDATE":           return "SYSDATE";
      case "SYSTIMESTAMP":      return "SYSTIMESTAMP";
      case "CURRENT_TIMESTAMP": return "CURRENT_TIMESTAMP";
      case "CURRENT_DATE":      return "CURRENT_DATE";
      case "SYS_GUID":          return "SYS_GUID()";
      default:                  return funcName + "()";
    }
  }

  if (defaultValue instanceof Literals.StringLiteral) {
    String val = ((Literals.StringLiteral) defaultValue).value();
    return "'" + val.replace("'", "''") + "'"; // Oracle escapes single quote as ''
  }

  if (defaultValue instanceof Literals.NumericLiteral) {
    return defaultValue.toString();
  }

  if (defaultValue instanceof UnparsedExpression) {
    return ((UnparsedExpression) defaultValue).unparsedExpression();
  }

  return super.fromGravitino(defaultValue);
}
```

### 6.5 Known Oracle JDBC Driver Issue with Default Values

Some versions of the Oracle JDBC driver return `COLUMN_DEF` with **leading or trailing spaces or newlines**. Always call `.trim()` before parsing.

A more reliable approach is to query `ALL_TAB_COLUMNS` directly instead of using the JDBC metadata API:

```sql
SELECT COLUMN_NAME, DATA_DEFAULT
FROM ALL_TAB_COLUMNS
WHERE OWNER = UPPER(:schema)
  AND TABLE_NAME = UPPER(:table)
ORDER BY COLUMN_ID
```

**Recommendation**: Override `getColumns()` in `OracleTableOperations` to query `ALL_TAB_COLUMNS` directly.

---

## 7. Table Operations

### 7.1 COMMENT Syntax

In Oracle, comments are **separate DDL statements**, not part of `CREATE TABLE`:

```sql
COMMENT ON TABLE "SCHEMA"."TABLE_NAME" IS 'table comment';
COMMENT ON COLUMN "SCHEMA"."TABLE_NAME"."COLUMN_NAME" IS 'column comment';
```

**Effect on `create()`**: Run `CREATE TABLE` first, then run `COMMENT ON TABLE` and `COMMENT ON COLUMN` statements separately:

```java
@Override
public void create(String databaseName, String tableName, ...) {
  try (Connection conn = getConnection(databaseName)) {
    // 1. Run CREATE TABLE (no COMMENT clause)
    JdbcConnectorUtils.executeUpdate(conn, generateCreateTableSql(...));

    // 2. Set table comment
    if (StringUtils.isNotEmpty(comment)) {
      JdbcConnectorUtils.executeUpdate(conn, String.format(
          "COMMENT ON TABLE \"%s\".\"%s\" IS '%s'",
          databaseName.toUpperCase(), tableName.toUpperCase(),
          comment.replace("'", "''")));
    }

    // 3. Set column comments
    for (JdbcColumn column : columns) {
      if (StringUtils.isNotEmpty(column.comment())) {
        JdbcConnectorUtils.executeUpdate(conn, String.format(
            "COMMENT ON COLUMN \"%s\".\"%s\".\"%s\" IS '%s'",
            databaseName.toUpperCase(), tableName.toUpperCase(),
            column.name().toUpperCase(),
            column.comment().replace("'", "''")));
      }
    }
  }
}
```

### 7.2 getTableProperties Implementation

The JDBC `DatabaseMetaData.getTables()` API is not reliable for Oracle. Query `ALL_TABLES` and `ALL_TAB_COMMENTS` directly:

```sql
SELECT t.TABLE_NAME, t.TABLESPACE_NAME, t.PARTITIONED,
       c.COMMENTS AS TABLE_COMMENT
FROM ALL_TABLES t
LEFT JOIN ALL_TAB_COMMENTS c
  ON c.OWNER = t.OWNER AND c.TABLE_NAME = t.TABLE_NAME
WHERE t.OWNER = UPPER(:schema)
  AND t.TABLE_NAME = UPPER(:table)
```

**Table properties to expose:**

| Property Key | Source | Description |
|--------------|--------|-------------|
| `tablespace` | `ALL_TABLES.TABLESPACE_NAME` | Storage tablespace name |
| `partitioned` | `ALL_TABLES.PARTITIONED` | `YES` or `NO` |
| `row_movement` | `ALL_TABLES.ROW_MOVEMENT` | `ENABLED` or `DISABLED` |
| `compression` | `ALL_TABLES.COMPRESSION` | `ENABLED` or `DISABLED` |

### 7.3 getColumns Implementation

Query `ALL_TAB_COLUMNS` and `ALL_COL_COMMENTS` directly instead of using the JDBC metadata API:

```sql
SELECT
  col.COLUMN_NAME,
  col.DATA_TYPE,
  col.DATA_LENGTH,
  col.DATA_PRECISION,
  col.DATA_SCALE,
  col.NULLABLE,
  col.DATA_DEFAULT,
  col.COLUMN_ID,
  cmt.COMMENTS AS COLUMN_COMMENT
FROM ALL_TAB_COLUMNS col
LEFT JOIN ALL_COL_COMMENTS cmt
  ON cmt.OWNER = col.OWNER
  AND cmt.TABLE_NAME = col.TABLE_NAME
  AND cmt.COLUMN_NAME = col.COLUMN_NAME
WHERE col.OWNER = UPPER(:schema)
  AND col.TABLE_NAME = UPPER(:table)
ORDER BY col.COLUMN_ID
```

Benefits:
1. Reliably gets column comments (JDBC API usually does not return Oracle column comments)
2. Reliably gets `DATA_DEFAULT` (avoids JDBC driver bugs)
3. Gets Oracle-specific type fields (`DATA_PRECISION`, `DATA_SCALE`) accurately

### 7.4 CREATE TABLE SQL

```sql
CREATE TABLE "SCHEMA_NAME"."TABLE_NAME" (
    "COL1" NUMBER(10)    NOT NULL,
    "COL2" VARCHAR2(100) DEFAULT 'default_value',
    "COL3" TIMESTAMP(6),
    "COL4" CLOB,
    CONSTRAINT "PK_TABLE_NAME" PRIMARY KEY ("COL1")
) TABLESPACE "USERS"
```

Key points:
- Use double quotes for all identifiers
- Define PRIMARY KEY as a named `CONSTRAINT` at the end of the column list
- No `COMMENT` clause (run separately after CREATE)
- `TABLESPACE` is optional, read from table properties
- No AUTO_INCREMENT (Oracle 11g)

### 7.5 ALTER TABLE SQL

```sql
-- Add column
ALTER TABLE "schema"."table" ADD ("new_col" VARCHAR2(100) DEFAULT 'val')

-- Change column type, default, or nullability
ALTER TABLE "schema"."table" MODIFY ("col" NUMBER(20) NOT NULL DEFAULT 0)

-- Drop column
ALTER TABLE "schema"."table" DROP COLUMN "col"

-- Rename column (supported in Oracle 11g)
ALTER TABLE "schema"."table" RENAME COLUMN "old_name" TO "new_name"

-- Update table comment (separate statement)
COMMENT ON TABLE "schema"."table" IS 'new comment'

-- Update column comment (separate statement)
COMMENT ON COLUMN "schema"."table"."col" IS 'new comment'

-- Rename table
ALTER TABLE "schema"."old_table" RENAME TO "new_table"
```

**Notes on Oracle MODIFY:**
- Changing a column type has strict limits (e.g., shrinking a `VARCHAR2` fails if existing data is longer)
- Changing `NULL` to `NOT NULL` fails if the column contains null values
- Multiple MODIFY changes can be combined: `MODIFY (col1 ..., col2 ...)`

### 7.6 Index Support

```sql
-- Regular index
CREATE INDEX "IDX_NAME" ON "schema"."table" ("col1", "col2")

-- Unique index
CREATE UNIQUE INDEX "IDX_NAME" ON "schema"."table" ("col1")

-- Primary key (as a constraint)
ALTER TABLE "schema"."table" ADD CONSTRAINT "PK_NAME" PRIMARY KEY ("col1")

-- Function-based index (not supported in v1)
CREATE INDEX "IDX_NAME" ON "schema"."table" (UPPER("col1"))
```

First version supports: `PRIMARY_KEY` and `UNIQUE_KEY` only.

---

## 8. AUTO_INCREMENT

### 8.1 Not Available in Oracle 11g

Oracle 11g has no column-level auto-increment feature. (Oracle 12c introduced IDENTITY COLUMN.)

The traditional workaround is SEQUENCE + TRIGGER:

```sql
-- Create a sequence
CREATE SEQUENCE "SEQ_TABLE_ID" START WITH 1 INCREMENT BY 1 NOCACHE

-- Create a trigger to fill the column before INSERT
CREATE OR REPLACE TRIGGER "TRG_TABLE_ID"
BEFORE INSERT ON "TABLE_NAME"
FOR EACH ROW
WHEN (NEW."ID" IS NULL)
BEGIN
  :NEW."ID" := SEQ_TABLE_ID.NEXTVAL;
END;
```

### 8.2 Decision

**Option A**: Do not support AUTO_INCREMENT.

Throw `UnsupportedOperationException` when `column.autoIncrement() == true`.

**Option B**: Simulate AUTO_INCREMENT using SEQUENCE + TRIGGER.

- Create a SEQUENCE and TRIGGER when the table is created with an auto-increment column
- Naming rule: `SEQ_{TABLE_NAME}_{COL_NAME}` (keep within 30-char limit)
- Drop the SEQUENCE and TRIGGER when the table is dropped
- Detect the TRIGGER in `getTableProperties` to determine whether the column is auto-increment

**First version recommendation**: Option A. Document the limitation clearly. Option B can be added in a later version.

---

## 9. Partitioning

### 9.1 Oracle Partition Types

Oracle 11g has rich partitioning support:

| Partition Type | Syntax Example | Description |
|----------------|----------------|-------------|
| Range | `PARTITION BY RANGE (col)` | Divide rows by value ranges |
| List | `PARTITION BY LIST (col)` | Divide rows by value lists |
| Hash | `PARTITION BY HASH (col)` | Divide rows by hash of column |
| Range-List | `PARTITION BY RANGE ... SUBPARTITION BY LIST` | Two-level partitioning |
| Range-Hash | `PARTITION BY RANGE ... SUBPARTITION BY HASH` | Two-level partitioning |
| Interval | `PARTITION BY RANGE ... INTERVAL (...)` | Auto-creates partitions (11g feature) |
| Reference | `PARTITION BY REFERENCE (fk)` | Partition based on parent table (11g feature) |

### 9.2 Gravitino Transform to Oracle Partition Mapping

| Gravitino Transform | Oracle Partition |
|---------------------|------------------|
| `Transforms.range(col)` | `PARTITION BY RANGE (col)` |
| `Transforms.list(col)` | `PARTITION BY LIST (col)` |
| `Transforms.hash(n, col)` | `PARTITION BY HASH (col) PARTITIONS n` |
| `Transforms.year(col)` | `PARTITION BY RANGE (col)` with year bounds |
| `Transforms.month(col)` | `PARTITION BY RANGE (col)` with month bounds |
| `Transforms.day(col)` | `PARTITION BY RANGE (col)` with day bounds |
| Composite partitions | Not supported in v1 |

### 9.3 First Version Scope

Support only basic **Range, List, and Hash** partitioning (no sub-partitions) in v1:

1. Gravitino's Transform model does not yet fully model Oracle composite partitions
2. Reading sub-partition info requires joining multiple system views, adding complexity
3. The three basic types cover most user needs

**Read partition info from Oracle:**

```sql
-- Check if table is partitioned and get partition type
SELECT PARTITIONING_TYPE, SUBPARTITIONING_TYPE, PARTITION_COUNT
FROM ALL_PART_TABLES
WHERE OWNER = UPPER(:schema) AND TABLE_NAME = UPPER(:table)

-- Get partition key columns
SELECT COLUMN_NAME, COLUMN_POSITION
FROM ALL_PART_KEY_COLUMNS
WHERE OWNER = UPPER(:schema) AND NAME = UPPER(:table)
ORDER BY COLUMN_POSITION
```

---

## 10. Exception Mapping

### 10.1 Oracle Error Codes

Oracle uses `ORA-NNNNN` format error codes. `SQLException.getErrorCode()` returns the numeric part.

| Oracle Error Code | Message | Gravitino Exception |
|-------------------|---------|---------------------|
| ORA-01918 | user does not exist | `NoSuchSchemaException` |
| ORA-01920 | user name conflicts with another user or role | `SchemaAlreadyExistsException` |
| ORA-00942 | table or view does not exist | `NoSuchTableException` |
| ORA-00955 | name is already used by an existing object | `TableAlreadyExistsException` |
| ORA-01031 | insufficient privileges | `UnauthorizedException` |
| ORA-01017 | invalid username/password | `ConnectionFailedException` |
| ORA-12541 | TNS: no listener | `ConnectionFailedException` |
| ORA-12514 | TNS: listener does not know of service | `ConnectionFailedException` |
| ORA-12505 | TNS: listener does not know of SID | `ConnectionFailedException` |
| ORA-00001 | unique constraint violated | `GravitinoRuntimeException` |
| ORA-01400 | cannot insert NULL | `GravitinoRuntimeException` |

```java
@Override
public GravitinoRuntimeException toGravitinoException(SQLException se) {
  switch (se.getErrorCode()) {
    case 1918:  return new NoSuchSchemaException(se.getMessage(), se);
    case 1920:  return new SchemaAlreadyExistsException(se.getMessage(), se);
    case 955:   return new TableAlreadyExistsException(se.getMessage(), se);
    case 942:   return new NoSuchTableException(se.getMessage(), se);
    case 1017:
    case 12541:
    case 12514:
    case 12505: return new ConnectionFailedException(se.getMessage(), se);
    default:    return new GravitinoRuntimeException(se.getMessage(), se);
  }
}
```

---

## 11. JDBC Connection

### 11.1 JDBC URL Formats

Oracle 11g supports two URL formats:

```
# SID format (traditional, common in 11g)
jdbc:oracle:thin:@<host>:<port>:<SID>
jdbc:oracle:thin:@192.168.1.100:1521:ORCL

# Service name format (recommended)
jdbc:oracle:thin:@//<host>:<port>/<service_name>
jdbc:oracle:thin:@//192.168.1.100:1521/orcl.example.com
```

### 11.2 JDBC Driver

The Oracle JDBC driver is **not published to Maven Central** due to licensing. Users must provide it manually.

- Driver class: `oracle.jdbc.OracleDriver`
- JAR file: `ojdbc8.jar` (recommended for Oracle 11g with JDBC 4.2)

**build.gradle.kts note:**

```kotlin
// Users must place ojdbc8.jar under {gravitino.home}/libs/
// Do not declare it as a regular dependency
compileOnly("com.oracle.database.jdbc:ojdbc8:21.9.0.0")
```

### 11.3 Connection and Schema Switching

Oracle does not support `connection.setCatalog()` (there is no database-level catalog concept like MySQL). To switch the active schema, use:

```sql
ALTER SESSION SET CURRENT_SCHEMA = "schema_name"
```

```java
@Override
protected Connection getConnection(String schemaName) throws SQLException {
  Connection conn = dataSource.getConnection();
  if (StringUtils.isNotBlank(schemaName)) {
    try (Statement stmt = conn.createStatement()) {
      stmt.execute(String.format(
          "ALTER SESSION SET CURRENT_SCHEMA = \"%s\"",
          schemaName.toUpperCase()));
    }
  }
  return conn;
}
```

---

## 12. Other Corner Cases

### 12.1 Empty String Equals NULL

Oracle treats `''` (empty string) the same as `NULL`. This is different from MySQL and PostgreSQL.

**Effect**: A column default of `''` behaves the same as `NULL`.

**Handling**: When `toGravitino` sees an empty string default value, convert it to `Literals.NULL`. Document this behavior.

### 12.2 NUMBER Precision Loss via JDBC Driver

When Gravitino maps `IntegerType` to `NUMBER(10)`, some Oracle JDBC driver versions may return the column type as `NUMBER` (no precision) when reading it back. This causes the round-trip type to become `DecimalType(38, 18)` instead of `IntegerType`.

**Fix**: Query `ALL_TAB_COLUMNS.DATA_PRECISION` directly instead of relying on JDBC metadata.

### 12.3 LONG and LONG RAW (Deprecated)

Oracle deprecated `LONG` and `LONG RAW`, but they still exist in 11g. Each table can only have one `LONG` column, and it cannot be indexed or used in WHERE clauses.

- `fromGravitino`: never generate these types
- `toGravitino`: recognize and map them; mark as deprecated in documentation

### 12.4 Gravitino StringIdentifier and Oracle COMMENT

Gravitino embeds a `StringIdentifier` in object comments to track its managed objects. Oracle's `COMMENT ON TABLE` and `COMMENT ON COLUMN` support up to 4000 bytes, which is enough for a StringIdentifier.

Getting comments in Oracle is different from MySQL. Use these views:

```sql
-- Get table comment
SELECT COMMENTS FROM ALL_TAB_COMMENTS
WHERE OWNER = UPPER(:schema) AND TABLE_NAME = UPPER(:table)

-- Get column comment
SELECT COMMENTS FROM ALL_COL_COMMENTS
WHERE OWNER = UPPER(:schema) AND TABLE_NAME = UPPER(:table)
  AND COLUMN_NAME = UPPER(:column)
```

### 12.5 Recycle Bin

Oracle 11g has a Recycle Bin. When a table is dropped without `PURGE`, it is not deleted immediately. Instead, Oracle renames it to something like `BIN$xxxx` and keeps it in the recycle bin.

**Effect**: `listTables()` querying `ALL_TABLES` will see these `BIN$...` tables.

**Fix options:**

Option A — Filter in `listTables()`:
```java
// Add to WHERE clause
AND table_name NOT LIKE 'BIN$%'
```

Option B — Always use `PURGE` when dropping:
```sql
DROP TABLE "schema"."table" PURGE
-- PURGE skips the recycle bin and deletes immediately
```

**Recommendation**: Use both. Always add `PURGE` to drop SQL, and also filter `BIN$` names in list queries as a safety net.

### 12.6 Synonyms

Oracle supports public synonyms (`PUBLIC SYNONYM`) and private synonyms (`SYNONYM`). These can appear when querying `ALL_TABLES`.

**Fix**: In `listTables()`, filter to only return rows where `OBJECT_TYPE = 'TABLE'`, excluding synonyms and views.

### 12.7 Uppercase Identifiers and User Experience

Because Oracle stores unquoted identifiers in uppercase, all table names and column names returned by Gravitino will be uppercase. Users coming from MySQL or PostgreSQL may find this unexpected.

**Document clearly:**
- Oracle does not distinguish uppercase from lowercase in names (unless double-quoted)
- Names created through Gravitino will be returned in uppercase
- Mixed-case identifiers (wrapped in double quotes) are not supported in v1

---

## 13. Unsupported Features in v1

| Feature | Reason |
|---------|--------|
| AUTO_INCREMENT | Oracle 11g has no native support. SEQUENCE+TRIGGER simulation is complex; defer to v2 |
| Schema comment | Oracle has no `COMMENT ON SCHEMA` syntax |
| `TimeType` column | Oracle has no standalone TIME column type |
| `BooleanType` direct mapping | Oracle table columns have no BOOLEAN type (use NUMBER(1) instead) |
| Composite partitions (sub-partitions) | Gravitino's Transform model does not yet fully support them |
| Function-based indexes | Out of scope for v1 |
| LOB storage options | Such as CACHE/NOCACHE, LOGGING, ENABLE STORAGE IN ROW |
| CHECK constraints | Gravitino's table interface does not yet support them |
| XMLTYPE / SDO_GEOMETRY | Not supported in v1 |
| INTERVAL types | Gravitino's type system has no matching type |
| CDB/PDB multi-tenant | Oracle 12c+ feature; not applicable to 11g |
| Case-sensitive identifiers | Simplified to always uppercase in v1 |

---

## 14. Integration Tests

### 14.1 Docker Image

Oracle does not provide an official Docker image for 11g, but there are third-party options:

- `gvenzl/oracle-xe:11` — Oracle XE 11g, lightweight, good for testing
- `gvenzl/oracle-xe:21` — More stable and better maintained; use this if exact version is not required

**Recommendation**: Use `gvenzl/oracle-xe:11` to match the target version.

### 14.2 Test Cases

Follow the patterns in `CatalogMysqlIT.java` and `CatalogClickHouseClusterIT.java`:

- `testCreateAndLoadSchema` — create a schema, load it, verify properties
- `testCreateAndLoadTable` — create a table with various Oracle types, load and verify
- `testAlterTable` — add / modify / drop / rename columns, update comments
- `testDropTable` — verify PURGE behavior and that the table is gone after drop
- `testListTables` — verify recycle bin entries (`BIN$...`) are filtered out
- `testPartitionedTable` — create Range, List, and Hash partitioned tables
- `testDefaultValues` — verify SYSDATE, NULL, string, and numeric defaults round-trip correctly
- `testNumberTypeMapping` — verify precision boundaries: `NUMBER`, `NUMBER(10)`, `NUMBER(10,2)`
- `testNameLengthLimit` — verify that names longer than 30 characters throw an error

---

## 15. Implementation Priority

| Priority | Item |
|----------|------|
| P0 | `OracleTypeConverter` — NUMBER, VARCHAR2, DATE, TIMESTAMP, CLOB, BLOB |
| P0 | `OracleDatabaseOperations` — list, create, drop, load schema |
| P0 | `OracleTableOperations` — create, drop, load, list tables; `getColumns` via `ALL_TAB_COLUMNS` |
| P0 | `OracleColumnDefaultValueConverter` — SYSDATE, NULL, string and number literals |
| P0 | `OracleExceptionConverter` — ORA-942, ORA-955, ORA-1918, ORA-1920, connection errors |
| P1 | Full ALTER TABLE support — add, modify, drop, rename columns and comments |
| P1 | Basic partitioning — Range, List, Hash |
| P1 | Integration tests — using `gvenzl/oracle-xe` Docker image |
| P2 | AUTO_INCREMENT simulation via SEQUENCE + TRIGGER |
| P2 | Regular (non-primary, non-unique) index support |
| P3 | Composite partitioning — Range-List, Range-Hash |

---

## 16. References

- Oracle 11g SQL Reference: https://docs.oracle.com/cd/E11882_01/server.112/e41084/toc.htm
- Oracle JDBC Developer's Guide: https://docs.oracle.com/cd/E11882_01/java.112/e16548/toc.htm
- Oracle Data Types: https://docs.oracle.com/cd/E11882_01/server.112/e41084/sql_elements001.htm
- Gravitino JDBC Framework: `catalogs/catalog-jdbc-common/`
- PostgreSQL Catalog (closest to Oracle's schema concept): `catalogs/catalog-jdbc-postgresql/`
