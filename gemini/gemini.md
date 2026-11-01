# GEMINI.md — DB2 LUW → PostgreSQL migration

This file is loaded into every Gemini CLI prompt for this repository. It defines the
job, the constraints, and the translation rules for migrating this Spring Boot service
from IBM DB2 for LUW to PostgreSQL. Follow it over general knowledge when they conflict.

## 1. The job in one paragraph

Convert this service so it runs against PostgreSQL only. Remove every DB2 dependency,
driver, dialect setting, and DB2-specific SQL. Preserve behavior exactly unless a
semantic difference between DB2 and PostgreSQL makes that impossible — then make the
smallest reasonable change and **highlight it** in `postgres-migration-summary.md`.
The database schema and the data move are owned by the DBA; you own the application code.

## 2. Stack facts (shared by all services using this file)

| Item | Value | Notes |
|---|---|---|
| Build | Maven, single module | `./mvnw` if present, else `mvn` |
| Java | 21 | Text blocks (`"""`) are allowed for SQL strings |
| Spring Boot | 4.x | Spring Framework 7, Jackson 3 (`tools.jackson.*`), modularized starters |
| JPA provider | Hibernate ORM 7.x (Boot 4.0 manages 7.2) | Jakarta Persistence 3.2, `jakarta.persistence.*` imports |
| Data access | Spring Data JPA repositories | Heavy use of `@Query(nativeQuery = true)` written for DB2 |
| Schema management | None in the repo — DBA-owned DDL | No Flyway/Liquibase; `ddl-auto` stays `none` |
| Profiles | Not used | Edit the default `application.properties` / `application.yml` only |
| Source DB | DB2 for LUW | |
| Target DB | PostgreSQL, modern release (version TBD, assume ≥ 15) | Driver `org.postgresql:postgresql`, version managed by Boot |
| DB-backed tests | None exist today | See Phase 5 for what verification is possible |
| Target Postgres schema | **Not available yet** | Work from type-parity assumptions (§6) and record every assumption |

**Confirm, don't assume, the exact versions** in Phase 0:
`mvn -q dependency:tree -Dincludes=org.springframework.boot,org.hibernate.orm,com.ibm.db2,org.postgresql`
and the `<parent>` version in `pom.xml`.

If a file named `db2-ddl.sql`, `postgres-ddl.sql`, or `ddl/` exists at the repo root,
read it before anything else — it overrides the assumptions in §6.

## 3. Rules of engagement

1. **Migration changes only.** No refactors, renames, dependency upgrades, formatting
   passes, or "while I'm here" fixes. If you see a bug unrelated to the migration, note it
   in the summary under *Observations* and leave the code alone.
2. **Preserve semantics.** When DB2 and PostgreSQL differ (see §9 watchlist), pick the
   translation that keeps the existing behavior. If that is impossible, choose the
   closest behavior and record it as a **⚠ CRITICAL** item in the summary.
3. **Never invent schema.** You do not know the Postgres column types yet. Where a
   translation depends on them, follow §6 (type parity) and tag the code with
   `// TODO(pg-migration): depends on DDL — <what>` plus a summary entry.
4. **Public contracts are sacred.** REST/JSON shapes, message payloads, file exports,
   scheduled-job outputs, and anything another system consumes must not change. If a
   change is unavoidable (e.g., an aggregate that was INTEGER in DB2 becomes a decimal),
   it goes at the top of the summary under **⚠ CRITICAL**.
5. **Keep diffs reviewable.** Edit in place; don't rewrite whole files when a few lines
   change. You may convert a SQL string you are already rewriting into a Java text block
   for readability; do not reformat queries you are not changing.
6. **One phase at a time** (§5). Finish a phase, run the phase's verification, update the
   summary, then stop and report. Don't start the next phase in the same turn unless told to.
7. **Compile often.** `mvn -q -DskipTests compile` after each batch of edits;
   `mvn -q test` at the end of every phase.
8. **Ask before**: deleting files, changing anything under `src/main/resources` other than
   the datasource/JPA properties, touching CI/Docker/deployment files, or running any
   `git` command that rewrites history. Plain `git add`/`git commit` on a feature branch is fine.
9. **Commits.** One commit per phase, message prefix `pg-migration:`; never commit a
   non-compiling tree.
10. **When unsure, prefer the standard-SQL form** (`CAST(x AS type)`, `FETCH FIRST n ROWS
    ONLY`, `CURRENT_TIMESTAMP`) over PostgreSQL shorthand (`x::type`, `LIMIT`, `now()`).
    Standard forms survive Hibernate's named-parameter parser and future tooling.

## 4. Working style for this model (Gemini 3.8 Flash, Gemini CLI)

- You have a 1M-token context window but a 64K-token output limit per turn. Read widely,
  write incrementally: edit file-by-file and stop at a safe, compiling point when a turn
  is getting long. Say "continuing in next turn" and pick up where you left off.
- Use targeted searches (`grep -rn`, `rg`) rather than reading every file; the grep
  patterns in Phase 0 find nearly all DB2 coupling.
- Phase 3 (native SQL) deserves the highest thinking level available; mechanical phases
  (1, 6) do not.
- After editing a query, re-read the whole method it lives in — parameters, return type,
  `@Modifying`, `countQuery`, and callers — not just the string.
- Keep a running list of **assumptions** and **open DBA questions** in the summary; add
  to it the moment one arises, not at the end.
- Do not paste large file contents into the summary. Reference `path:line`.

## 5. Phases

Work the phases in order. Each phase ends with: verification run → summary updated → commit → stop.

### Phase 0 — Inventory (read-only, no edits)

1. Confirm versions (Boot, Hibernate, DB2 driver) with the dependency-tree command in §2.
2. Grep for DB2 coupling and record every hit as `path:line` in the summary's *Inventory*
   section (`rg` = ripgrep; if it is not installed, use `grep -rnE` / `grep -rniE`):

   ```
   # dependencies & config
   rg -n "com\.ibm|db2|DB2|jcc" pom.xml src/main/resources
   # dialect / hibernate settings
   rg -n "dialect|database-platform|globally_quoted|default_schema|ddl-auto|non_contextual" src/main/resources
   # native SQL and Hibernate SQL annotations
   rg -n "nativeQuery\s*=\s*true|createNativeQuery|@NamedNativeQuery|@SQLRestriction|@Where\(|@Formula|@Subselect|@SQLInsert|@SQLUpdate|@SQLDelete|@ColumnTransformer|@Filter" src/main/java
   # DB2 SQL idioms
   rg -n -i "SYSIBM|SYSDUMMY1|WITH UR|WITH CS|WITH RS|WITH RR|FOR READ ONLY|FOR FETCH ONLY|OPTIMIZE FOR|CURRENT DATE|CURRENT TIMESTAMP|CURRENT TIME\b|CURRENT SCHEMA|NEXT VALUE FOR|NEXTVAL FOR|PREVIOUS VALUE FOR|IDENTITY_VAL_LOCAL|FINAL TABLE|TIMESTAMPDIFF|\bDAYS\(|\bDAYS\b|MONTHS\b|YEARS\b|HOURS\b|MINUTES\b|SECONDS\b|VARCHAR_FORMAT|TIMESTAMP_FORMAT|LISTAGG|XMLAGG|\bVALUE\(|\bLOCATE\(|\bPOSSTR\(|\bUCASE\(|\bLCASE\(|\bDIGITS\(|\bDECIMAL\(|\bINTEGER\(|\bINT\(|\bCHAR\(|\bDOUBLE\(|FOR BIT DATA|CLOB|BLOB|DBCLOB|GRAPHIC|DECFLOAT|TRUNCATE TABLE|GLOBAL TEMPORARY|SESSION\.|RID\(|GENERATE_UNIQUE|MERGE INTO|COUNT_BIG|ROWNUMBER\(" src/main/java src/main/resources
   # Java-side coupling
   rg -n "SQLCODE|getErrorCode\(|-803|-911|-913|-530|-204|DB2Diagnosable|java\.sql\.Clob|java\.sql\.Blob|getClob|getBlob|setClob|setBlob|createClob|createBlob|@Lob" src/main/java
   ```
3. Build a **native-query inventory table** in the summary: file, method, one-line purpose,
   DB2 idioms present, difficulty (Easy = clause removal / function rename; Medium =
   date math, string functions, casts; Hard = MERGE, temp tables, recursion, RETURNING,
   semantics-changing).
4. List entity fields that are LOBs, boolean-like (`CHAR(1)` Y/N, `SMALLINT` flags),
   identity/sequence IDs, and temporal types.
5. Write the *Inventory* and *Assumptions* sections of the summary. No code changes.

### Phase 1 — Dependencies and configuration

- `pom.xml`: remove the DB2 driver (`com.ibm.db2:jcc` or `com.ibm.db2.jcc:db2jcc4`);
  add `org.postgresql:postgresql` with `<scope>runtime</scope>` and **no version**.
- Properties: switch URL to `jdbc:postgresql://HOST:5432/DBNAME?currentSchema=SCHEMA`
  (keep host/db/schema as placeholders or env vars exactly as the DB2 ones were
  parameterized); remove `driver-class-name` (Boot detects it) or set
  `org.postgresql.Driver`; **delete** `spring.jpa.database-platform` /
  `hibernate.dialect` (Hibernate auto-detects; if a dialect must be explicit use
  `org.hibernate.dialect.PostgreSQLDialect`, never a versioned one — they were removed);
  remove DB2-only Hikari `data-source-properties` (`currentSchema`, `clientProgramName`,
  `retrieveMessagesFromServerOnGetMessage`, …); keep `spring.jpa.hibernate.ddl-auto=none`
  (or leave unset); remove `hibernate.globally_quoted_identifiers` if present (§7).
- Delete any custom `DB2Dialect` subclasses, DB2 `SQLExceptionTranslator` beans, or
  DB2-specific `DataSource` configuration classes.
- Verify: `mvn -q -DskipTests compile` passes; `rg -n "db2|ibm" pom.xml src` returns nothing
  except comments you intend to remove later.

### Phase 2 — Entities and mappings

Apply §7 rules to every `@Entity`, `@Embeddable`, `@MappedSuperclass`, and `orm.xml`:
LOB mappings, boolean columns, `columnDefinition`, ID generation, temporal types,
quoted identifiers, `@Formula` / `@SQLRestriction` / `@Subselect` SQL, `@NamedNativeQuery`.
Verify: compile + `mvn -q test`.

### Phase 3 — Native SQL in repositories

Repository by repository, in inventory order (Easy → Hard). For each query:
1. Translate using §8 (the rulebook). Rewrite the whole query coherently; don't patch tokens.
2. Check parameter types and nullability (§7.6). Check `countQuery` for paged queries.
3. Check result mapping: interface/DTO projections, `@SqlResultSetMapping`, scalar return
   types (§7.7).
4. If a translation changes semantics, mark the code `// TODO(pg-migration): SEMANTIC —
   <what>` and add a **⚠ CRITICAL** summary entry.
5. If the translation depends on an unknown column type, mark
   `// TODO(pg-migration): depends on DDL — <what>` and add an *Assumptions* entry.
Verify after each repository: compile; after the phase: `mvn -q test` and the SQL syntax
check in Phase 5 step 2 if Python is available.

### Phase 4 — Java-side DB2 coupling

- Replace SQLCODE / `getErrorCode()` checks with Spring's translated exceptions
  (`DuplicateKeyException`, `DataIntegrityViolationException`,
  `CannotAcquireLockException`, `DeadlockLoserDataAccessException`) or SQLSTATE
  checks (§8.11). Remove `com.ibm.*` imports.
- Replace `java.sql.Clob`/`Blob` field types and `getClob`/`getBlob`/`createClob` JDBC
  calls with `String`/`byte[]` and `getString`/`getBytes` (§7.1).
- Fix any code that reads result-set labels or depends on identifier case (§7.7).
- Remove any DB2-specific `JdbcTemplate` SQL, `SET CURRENT SCHEMA` calls, or
  `WITH UR` appended programmatically.
Verify: compile, `mvn -q test`, `rg -n "com\.ibm" src` empty.

### Phase 5 — Verification

What is possible depends on what exists. Do all that apply, in order:
1. `mvn -q clean test`.
2. **Offline PostgreSQL syntax check** of every native query (no database needed) using
   `pglast` (Python bindings to the real PostgreSQL parser). If `python3` is available:
   `pip install pglast`, then run the script in §10. Every query must parse; record the
   result per query in the summary.
3. **If a Postgres DDL is available**: stand up `postgres:16` (or the target version) in
   Docker, apply the DDL, then (a) run the app once with
   `spring.jpa.hibernate.ddl-auto=validate` to catch mapping mismatches (then set it back
   to `none`), and (b) `EXPLAIN` every native query against the schema with placeholder
   parameters. Add Testcontainers-based repository tests per §10 if asked.
4. Walk the §9 watchlist and confirm each item is either not applicable or recorded.
5. Reconcile the summary: every TODO tag in the code must have a summary entry, and vice versa.

### Phase 6 — Cleanup and handoff

Remove dead DB2 code paths and comments, confirm `rg -in "db2|ibm|sysibm"` is clean,
finalize the summary (status, DBA questions, remaining TODOs), final commit.

## 6. Type-parity assumptions (until the DDL arrives)

Assume the DBA maps DB2 types 1:1 to their natural PostgreSQL equivalents and creates
identifiers **unquoted** (so PostgreSQL folds them to lowercase). Record every place
your translation would differ if an assumption is wrong.

| DB2 LUW | Assumed PostgreSQL | Java type to use | Notes |
|---|---|---|---|
| `SMALLINT` | `smallint` | `Short` (or `Boolean` via converter if it is a flag) | If DDL makes flags `boolean`, see §7.2 |
| `INTEGER` / `BIGINT` | `integer` / `bigint` | `Integer` / `Long` | |
| `DECIMAL(p,s)`, `DECFLOAT` | `numeric(p,s)` / `numeric` | `BigDecimal` | |
| `REAL` / `DOUBLE` | `real` / `double precision` | `Float` / `Double` | |
| `CHAR(n)` | `char(n)` | `String` | PG `char(n)` drops trailing spaces when cast to text (`\|\|`, `LIKE`) — §9 |
| `CHAR(1)` Y/N flag | `char(1)` **or** `boolean` | depends — §7.2 | Ask the DBA; this is the most likely divergence |
| `VARCHAR(n)` | `varchar(n)` | `String` | DB2 length is bytes, PG is characters — only ever more room |
| `GRAPHIC` / `VARGRAPHIC` | `varchar` | `String` | |
| `CLOB` / `DBCLOB` | `text` | `String` (no `@Lob`) | §7.1 |
| `BLOB`, `… FOR BIT DATA` | `bytea` | `byte[]` (no `@Lob`) | §7.1 |
| `DATE` / `TIME` | `date` / `time` | `LocalDate` / `LocalTime` | |
| `TIMESTAMP(p)` | `timestamp(p)` without time zone, p ≤ 6 | `LocalDateTime` | DB2 allows p up to 12; PG max 6 — §9 |
| `BOOLEAN` | `boolean` | `Boolean` | |
| `XML` | `xml` | `String` | functions differ; case by case |
| Identity column | `GENERATED … AS IDENTITY` | `@GeneratedValue(strategy = IDENTITY)` | §7.4 |
| Sequence | sequence, same name, same `INCREMENT BY` | `@SequenceGenerator` with matching `allocationSize` | §7.4 — ask DBA for increment |
| `ROWID`, `RID()` | none (`ctid` is not stable) | — | flag any use as **⚠ CRITICAL** |

## 7. Hibernate 7 / Spring Data rules

### 7.1 LOBs — the #1 runtime trap
In Hibernate 6+, `@Lob` on a `String` or `byte[]` maps to PostgreSQL **large objects
(`oid`)**, not `text`/`bytea`. Reading a `text` column through `@Lob` fails at runtime
("Bad value for type long"). Therefore:
- **Remove `@Lob`** from every `String` and `byte[]` field.
- `String` → `text`: use `@JdbcTypeCode(SqlTypes.LONG32VARCHAR)` (import
  `org.hibernate.annotations.JdbcTypeCode`, `org.hibernate.type.SqlTypes`) or
  `@Column(length = Length.LONG32)`. Plain `String` with no length also works for
  reading a `text` column; prefer the explicit annotation so intent is visible.
- `byte[]` → `bytea`: plain `byte[]` (Hibernate maps to `VARBINARY` → `bytea`); use
  `@JdbcTypeCode(SqlTypes.LONG32VARBINARY)` if you want it explicit.
- Replace `java.sql.Clob` / `java.sql.Blob` field types with `String` / `byte[]`;
  replace JDBC `getClob/getBlob/createClob/createBlob` with `getString/getBytes/setString/setBytes`.
- Delete `hibernate.jdbc.lob.non_contextual_creation` from properties unless a
  `createClob() not yet implemented` warning actually appears.
- Remove DB2 `columnDefinition = "CLOB(…)"` / `"BLOB(…)"`.
- Native SQL: PostgreSQL `text` compares, `LIKE`s, and `DISTINCT`s directly — drop the
  `CAST(clob_col AS VARCHAR(n))` wrappers DB2 required (unless they truncate on purpose).

### 7.2 Boolean-like columns (`CHAR(1)` Y/N, `SMALLINT` 0/1)
- If the Postgres column keeps the DB2 type (default assumption), keep the Java type and
  any existing `AttributeConverter`; native SQL like `flag = 'Y'` or `active = 1` still works.
  Prefer Hibernate's built-in converters when you touch one:
  `@Convert(converter = org.hibernate.type.YesNoConverter.class)` (Y/N),
  `TrueFalseConverter` (T/F), `NumericBooleanConverter` (0/1).
- If the DDL makes the column `boolean`: change the field to `Boolean`, remove the
  converter, and rewrite native predicates to `WHERE active` / `WHERE NOT active` /
  `= TRUE`. `boolean = integer` and `boolean = varchar` (a bound `String` param) are
  errors in PostgreSQL. Each such column is a *depends on DDL* item.

### 7.3 `columnDefinition`, quoting, naming
- Remove `columnDefinition` values that spell DB2 types (`VARCHAR(10) FOR BIT DATA`,
  `TIMESTAMP(12)`, `CLOB(1M)`, `DECIMAL(15,2)` …). The DBA owns DDL; Hibernate only
  needs the Java type. If the definition carried a constraint the code relies on, note it.
- Remove `hibernate.globally_quoted_identifiers=true` and any explicitly quoted names
  (`@Table(name = "\"CUSTOMER\"")`). Quoted uppercase names will not match the DBA's
  unquoted (lowercase-folded) tables. Unquoted `@Table(name = "CUSTOMER")` is fine.
- `@Column(name = …)` case does not matter when unquoted. Do not rename columns.

### 7.4 Identifiers and sequences
- Identity columns: `@GeneratedValue(strategy = GenerationType.IDENTITY)` works unchanged.
- Sequences: `@SequenceGenerator(sequenceName = …, allocationSize = N)` — `N` **must
  equal** the sequence's `INCREMENT BY` in PostgreSQL. Record the current value as a DBA
  question; do not change it.
- Never leave `GenerationType.AUTO`: on PostgreSQL Hibernate 6+ resolves it to a
  sequence named `<entity>_SEQ` with increment 50, which the DBA has not created.
  Replace with the strategy the DB2 column actually used (identity vs. named sequence).
- `IDENTITY_VAL_LOCAL()` / `SELECT … FROM FINAL TABLE (INSERT …)` → prefer Hibernate's
  generated-key handling, else `INSERT … RETURNING id`.

### 7.5 Temporal types and time zones
- Match Java types to column types: `LocalDate`↔`date`, `LocalTime`↔`time`,
  `LocalDateTime`↔`timestamp`, `Instant`/`OffsetDateTime`↔`timestamptz`.
- Hibernate 6+ maps `Instant` to `timestamp with time zone`. If the DBA creates
  `timestamp` (no tz) columns — the DB2-parity default — keep `LocalDateTime` fields or
  set `spring.jpa.properties.hibernate.timezone.default_storage=NORMALIZE_UTC` and
  record the choice. Do not silently switch a field from `LocalDateTime` to `Instant`.
- Replace `java.sql.Timestamp`/`java.util.Date` + `@Temporal` only if the field is
  already being changed; otherwise leave it (Hibernate 7 still supports them).
- `CURRENT_TIMESTAMP` in PostgreSQL is `timestamptz`; `LOCALTIMESTAMP` is the DB2-like
  `timestamp`. Use `LOCALTIMESTAMP` when assigning to `timestamp` columns (§8.5).
- The JDBC driver sends the JVM default zone as the session time zone; note the JVM
  `-Duser.timezone` for every environment in *DBA/ops questions*.

### 7.6 Parameters — PostgreSQL infers types strictly
- `WHERE (:p IS NULL OR col = :p)` fails with *could not determine data type of
  parameter* when `p` is null. Write `CAST(:p AS <type>) IS NULL OR col = :p`, or split
  into two queries / use `Specification`.
- Bound Java types must match column types: `String` param against an `integer` column
  → *operator does not exist: integer = character varying*. Fix the Java type, or
  `CAST(:p AS INTEGER)`.
- Collections: `IN (:ids)` is fine (Hibernate expands); an **empty** collection renders
  invalid SQL — guard in Java if the caller can pass an empty list.
- `FETCH FIRST :n ROWS ONLY` and `LIMIT :n` both accept a bound parameter; prefer
  `Pageable` for anything user-facing.
- Do not use `::` casts inside `@Query` strings; use `CAST(x AS type)` (parser-safe).

### 7.7 Result mapping and identifier case
- DB2 returned unquoted column labels in UPPERCASE; PostgreSQL returns them lowercase.
  Interface/DTO projections on native queries look aliases up by name and may not
  tolerate the case change — quote camelCase aliases so the label is exact:
  `SELECT c.first_name AS "firstName" …`. Audit every projection.
- `@SqlResultSetMapping` / `ConstructorResult` column names: unquoted lowercase is fine.
- Scalar results: `COUNT(*)` is `Long` in both; `SUM(integer)` is `Long` in PG (`Integer`
  in DB2 — DB2 `SUM(INT)` is INTEGER, overflow aside); `AVG(integer)` is **`BigDecimal`** in PG
  (`Integer`, truncated, in DB2) — see §9. Check every `getSingleResult()` cast.
- Derived tables need an alias (`FROM (SELECT …) AS t`) on PostgreSQL < 16. Always alias.

### 7.8 JPQL / HQL
JPQL is mostly portable. Check `FUNCTION('name', …)` calls that name DB2 functions,
`@Formula`/`@SQLRestriction` fragments (they are native SQL), and any HQL using
DB2-only functions via `FUNCTION(...)`.

## 8. DB2 LUW → PostgreSQL SQL rulebook

Apply to native `@Query` strings, `@NamedNativeQuery`, `@Formula`, `@SQLRestriction`,
`@Subselect`, `@SQLInsert/Update/Delete`, `createNativeQuery`, `JdbcTemplate` SQL, and
`.sql` resources. "Same" means leave it alone.

### 8.1 Clauses to delete outright
| DB2 | PostgreSQL |
|---|---|
| `WITH UR` / `WITH CS` / `WITH RS` / `WITH RR` (and `USE AND KEEP … LOCKS`) | delete — isolation is per transaction; `WITH UR` removal changes semantics (§9) |
| `FOR READ ONLY` / `FOR FETCH ONLY` | delete |
| `OPTIMIZE FOR n ROWS` | delete |
| `FOR UPDATE WITH RS` / `… OF col` | `FOR UPDATE` |
| `SKIP LOCKED DATA` (z/OS syntax, if it appears) | `FOR UPDATE SKIP LOCKED` |
| `FROM SYSIBM.SYSDUMMY1` | delete the `FROM` entirely (`SELECT 1`, `SELECT CURRENT_DATE`) |
| `VALUES expr` (standalone) | `SELECT expr` |
| `TRUNCATE TABLE t IMMEDIATE` | `TRUNCATE TABLE t` |
| `SET CURRENT SCHEMA x` | `SET search_path TO x` — but prefer the `currentSchema` URL property and delete the statement |

### 8.2 Row limiting and paging
| DB2 | PostgreSQL |
|---|---|
| `FETCH FIRST n ROWS ONLY` / `FETCH FIRST ROW ONLY` | same (standard) — or `LIMIT n` |
| `OFFSET n ROWS FETCH NEXT m ROWS ONLY` | same — or `LIMIT m OFFSET n` |
| `FETCH FIRST :n ROWS ONLY` (bound) | same — PostgreSQL accepts a bare parameter; `LIMIT :n` is equivalent |
| `DELETE FROM (SELECT … FROM t WHERE … FETCH FIRST n ROWS ONLY)` / `UPDATE (SELECT …)` | no DML on a subselect — `DELETE FROM t WHERE id IN (SELECT id FROM t WHERE … LIMIT n)` (use `ctid` if there is no key) |
| `ROWNUMBER() OVER (…)` | `ROW_NUMBER() OVER (…)` |

### 8.3 Current values
| DB2 | PostgreSQL |
|---|---|
| `CURRENT DATE` / `CURRENT_DATE` | `CURRENT_DATE` |
| `CURRENT TIME` | `LOCALTIME` |
| `CURRENT TIMESTAMP` | `LOCALTIMESTAMP` (matches DB2's zone-less value); `CURRENT_TIMESTAMP` is `timestamptz` |
| `CURRENT TIMESTAMP(0)` | `LOCALTIMESTAMP(0)` |
| `CURRENT SCHEMA` | `current_schema` |
| `CURRENT USER` / `USER` / `SESSION_USER` | `current_user` / `session_user` |
| `CURRENT TIMEZONE` | no equivalent (`current_setting('TimeZone')` is a name, not an offset) — flag |

### 8.4 Date/time arithmetic (labeled durations are DB2-only)
| DB2 | PostgreSQL |
|---|---|
| `d + 30 DAYS`, `d - 1 DAY` | `d + INTERVAL '30 days'`; for a bound value `d + :n * INTERVAL '1 day'` (or `d + :n` when `d` is a `date` and `:n` an integer) |
| `… n MONTHS / YEARS / HOURS / MINUTES / SECONDS / MICROSECONDS` | `… INTERVAL 'n months'` etc.; bound: `:n * INTERVAL '1 month'` |
| `DAYS(a) - DAYS(b)` | `(CAST(a AS DATE) - CAST(b AS DATE))` → integer days |
| `DAYS(x)` alone | `(CAST(x AS DATE) - DATE '0001-01-01' + 1)` |
| `TIMESTAMPDIFF(16, CHAR(a - b))` (days) | `EXTRACT(EPOCH FROM (a - b)) / 86400` — **DB2's result is an estimate** (30-day months, 365-day years); PG is exact (§9). Codes: 1 µs, 2 s, 4 min, 8 h, 16 d, 32 wk, 64 mo, 128 qtr, 256 yr |
| `a - b` (timestamp duration as `DECIMAL(20,6)` yyyymmddhhmmss) | `a - b` is an `interval`; extract what the code needs (`EXTRACT(EPOCH FROM …)`, `EXTRACT(DAY FROM …)`) — flag any code parsing the DB2 decimal form |
| `TIMESTAMP(d)` / `TIMESTAMP(d, t)` / `TIMESTAMP('…')` | `CAST(d AS TIMESTAMP)` / `d + t` / `TIMESTAMP '…'` (function-call form `TIMESTAMP(...)` is a syntax error in PG) |
| `DATE(x)` / `TIME(x)` | `CAST(x AS DATE)` / `CAST(x AS TIME)` (`DATE(x)` happens to parse in PG; `TIME(x)` does not — use CAST everywhere) |
| `DATE('2024-01-31')` | `DATE '2024-01-31'` |
| `YEAR(x)`, `MONTH(x)`, `DAY(x)`, `HOUR(x)`, `MINUTE(x)`, `SECOND(x)` | `EXTRACT(YEAR FROM x)` … — returns `numeric`; wrap in `CAST(… AS INTEGER)` where an int is compared/returned |
| `DAYOFWEEK(x)` (1 = Sunday) | `EXTRACT(DOW FROM x) + 1` |
| `DAYOFWEEK_ISO(x)` (1 = Monday) | `EXTRACT(ISODOW FROM x)` |
| `DAYOFYEAR(x)` | `EXTRACT(DOY FROM x)` |
| `WEEK_ISO(x)` | `EXTRACT(WEEK FROM x)` (ISO week) |
| `WEEK(x)` (Sunday-start, 1–54) | no built-in equivalent — `FLOOR((EXTRACT(DOY FROM x) + EXTRACT(DOW FROM DATE_TRUNC('year', x)) - 1) / 7) + 1`; verify against DB2 output and flag |
| `QUARTER(x)` | `EXTRACT(QUARTER FROM x)` |
| `DAYNAME(x)` / `MONTHNAME(x)` | `TO_CHAR(x, 'FMDay')` / `TO_CHAR(x, 'FMMonth')` |
| `LAST_DAY(x)` | `(DATE_TRUNC('month', x) + INTERVAL '1 month - 1 day')::date` → write as `CAST(DATE_TRUNC('month', x) + INTERVAL '1 month - 1 day' AS DATE)` |
| `ADD_MONTHS(x, n)` | `x + n * INTERVAL '1 month'` — DB2 snaps end-of-month to end-of-month (Feb 28 + 1 → Mar 31); PG gives Mar 28 (§9) |
| `NEXT_DAY`, `MIDNIGHT_SECONDS`, `JULIAN_DAY`, `TIMESTAMP_ISO`, `DATE(int)` | translate case by case; flag |
| `VARCHAR_FORMAT(ts, fmt)` / `TO_CHAR` | `TO_CHAR(ts, fmt)` — format tokens mostly compatible; check `FF`→`US`, `RRRR` |
| `TIMESTAMP_FORMAT(s, fmt)` / `TO_DATE` | `TO_TIMESTAMP(s, fmt)` (returns `timestamptz`; wrap in `CAST(… AS TIMESTAMP)` for a `timestamp` column) |
| `CHAR(d, ISO)` / `CHAR(d, USA)` / `CHAR(d, EUR)` | `TO_CHAR(d, 'YYYY-MM-DD')` / `'MM/DD/YYYY'` / `'DD.MM.YYYY'` |
| `TIMESTAMP(12)` precision | PG max is 6 — values truncate (§9) |

### 8.5 Strings
| DB2 | PostgreSQL |
|---|---|
| `a \|\| b`, `CONCAT(a, b)` | `\|\|` same (NULL-propagating). PG `CONCAT()` treats NULL as `''` — keep `\|\|` to preserve DB2 NULL behavior |
| `SUBSTR(s, i, n)`, `SUBSTRING(s, i, n)` | same |
| `LOCATE(needle, haystack)` | `STRPOS(haystack, needle)` — **argument order flips**; `POSITION(needle IN haystack)` also works |
| `LOCATE(needle, haystack, start)` | `COALESCE(NULLIF(STRPOS(SUBSTR(haystack, start), needle), 0) + start - 1, 0)` |
| `POSSTR(haystack, needle)` / `INSTR(haystack, needle)` | `STRPOS(haystack, needle)` |
| `UCASE` / `LCASE` | `UPPER` / `LOWER` |
| `STRIP(s)`, `STRIP(s, BOTH, ' ')`, `STRIP(s, L/T, c)` | `TRIM(s)`, `TRIM(BOTH ' ' FROM s)`, `LTRIM/RTRIM(s, c)` |
| `TRANSLATE(s, to, from)` | `TRANSLATE(s, from, to)` — **argument order flips** |
| `LENGTH`, `CHAR_LENGTH`, `CHARACTER_LENGTH` | same (PG `LENGTH(char(n))` ignores padding — §9) |
| `CHAR_LENGTH(s, OCTETS)` / `LENGTH(s, OCTETS)` | `OCTET_LENGTH(s)` |
| `REPLACE`, `REPEAT`, `LEFT`, `RIGHT`, `LPAD`, `RPAD`, `LTRIM`, `RTRIM`, `ASCII`, `CHR`, `INITCAP`, `REVERSE` | same |
| `CHAR(n)` (number → string) / `VARCHAR(x)` | `CAST(x AS VARCHAR)` — DB2 `CHAR(int)` right-pads to the type width; if code depends on the padding, `RPAD(…)` |
| `DIGITS(n)` | `LPAD(CAST(ABS(n) AS TEXT), <precision>, '0')` — precision comes from the column; flag |
| `HEX(x)` | `ENCODE(x, 'hex')` (bytea) / `TO_HEX(n)` (int) |
| `VALUE(a, b, …)`, `NVL`, `IFNULL` | `COALESCE(a, b, …)` |
| `DECODE(x, v1, r1, …, default)` | `CASE x WHEN v1 THEN r1 … ELSE default END` |
| `LISTAGG(x, ',') WITHIN GROUP (ORDER BY y)` | `STRING_AGG(CAST(x AS TEXT), ',' ORDER BY y)` |
| `XMLAGG(XMLELEMENT(…))`-based string aggregation | `STRING_AGG(…)` |
| `REGEXP_LIKE(s, p)` | `s ~ p` (case-insensitive: `~*`) |
| `REGEXP_REPLACE/SUBSTR/COUNT/INSTR` | same names in PG 15+; flag arguments differ (`'i'`/`'g'` flag positions) |
| `LIKE … ESCAPE` | same |
| `UPPER(col) LIKE UPPER(:p)` | same, or `col ILIKE :p` |

### 8.6 Numbers and casts
| DB2 | PostgreSQL |
|---|---|
| `INTEGER(x)` / `INT(x)` / `BIGINT(x)` / `SMALLINT(x)` on a decimal | `CAST(TRUNC(x) AS INTEGER)` — **DB2 truncates, `CAST` in PG rounds** |
| `DECIMAL(x, p, s)` / `DEC(x, p, s)` | `CAST(x AS NUMERIC(p, s))` (rounds in both) |
| `DOUBLE(x)` / `FLOAT(x)` / `REAL(x)` | `CAST(x AS DOUBLE PRECISION)` / `… REAL` |
| `CAST(x AS type)` | same |
| `TRUNCATE(x, n)` / `TRUNC(x, n)` | `TRUNC(CAST(x AS NUMERIC), n)` — two-arg form needs `numeric` |
| `ROUND(x, n)` | `ROUND(CAST(x AS NUMERIC), n)` — two-arg form needs `numeric` |
| `LOG(x)` (natural log in DB2) | `LN(x)` — PG `LOG` is base 10 |
| `LOG10(x)` | `LOG(x)` |
| `MOD(a, b)` | same (or `%`) |
| `RAND()` / `RANDOM()` | `RANDOM()` |
| `MAX(a, b)` / `MIN(a, b)` with 2+ args (scalar) | `GREATEST(a, b)` / `LEAST(a, b)` |
| `COUNT_BIG(*)` | `COUNT(*)` |
| `AVG(int_col)` | same — **result type changes** from truncated integer to `numeric` (§9) |
| `BITAND/BITOR/BITXOR/BITNOT` | `&` / `\|` / `#` / `~` |
| int-column `= '123'` literal | works; int-column `= varchar_col` does not → `CAST` |
| `CASE … THEN 1 ELSE 'N/A'`, `COALESCE(int_col, '')` | type mismatch errors — make branches the same type |

### 8.7 Sequences, identity, RETURNING
| DB2 | PostgreSQL |
|---|---|
| `NEXT VALUE FOR seq` / `NEXTVAL FOR seq` | `NEXTVAL('seq')` |
| `PREVIOUS VALUE FOR seq` / `PREVVAL FOR seq` | `CURRVAL('seq')` |
| `VALUES NEXT VALUE FOR seq` | `SELECT NEXTVAL('seq')` |
| `IDENTITY_VAL_LOCAL()` | `LASTVAL()` — prefer `INSERT … RETURNING id` |
| `SELECT … FROM FINAL TABLE (INSERT/UPDATE …)` | `INSERT/UPDATE … RETURNING …` |
| `SELECT … FROM OLD TABLE (DELETE/UPDATE …)` | `DELETE/UPDATE … RETURNING …` (old values only for DELETE; UPDATE returns new — flag if old values needed) |
| `GENERATE_UNIQUE()` | `GEN_RANDOM_UUID()` — different type/length (§9) |

### 8.8 DML and statements
| DB2 | PostgreSQL |
|---|---|
| `MERGE INTO t USING s ON … WHEN MATCHED THEN UPDATE … WHEN NOT MATCHED THEN INSERT …` | PG 15+: same statement (no `WHEN NOT MATCHED BY SOURCE` before 17, no `RETURNING` before 17). Single-row upsert: `INSERT … ON CONFLICT (key) DO UPDATE SET …` — requires a unique constraint on `key`; confirm with DBA |
| `UPDATE t AS x SET x.col = …` | alias not allowed on the target column: `SET col = …` |
| `UPDATE t SET (a, b) = (SELECT …)` | same (PG ≥ 9.5) |
| `DELETE FROM t AS x WHERE …` | same |
| `WITH t(cols) AS (… UNION ALL …)` recursive | `WITH RECURSIVE t(cols) AS (…)` — keyword is mandatory |
| `SELECT * FROM TABLE(fn(…)) AS t` | `SELECT * FROM fn(…) AS t` |
| `DECLARE GLOBAL TEMPORARY TABLE SESSION.x …` / `SESSION.x` | `CREATE TEMP TABLE x (…) ON COMMIT DROP` and unqualified `x` — pooled connections make session temp tables hazardous; **⚠ CRITICAL** review |
| `LOCK TABLE t IN EXCLUSIVE MODE` | `LOCK TABLE t IN ACCESS EXCLUSIVE MODE` (or `SHARE ROW EXCLUSIVE` if readers must continue) — flag |
| `CALL proc(…)` | `CALL proc(…)` for procedures; `SELECT fn(…)` for functions — DBA decides which the object became |
| `SELECT DISTINCT` on a CLOB column | allowed in PG — no change needed |
| `INSERT INTO t VALUES (…), (…)` | same |
| `EXCEPT` / `INTERSECT` / `UNION [ALL]` | same |
| Window functions, `GROUPING SETS`, `ROLLUP`, `CUBE`, `NULLS FIRST/LAST`, `IS [NOT] DISTINCT FROM`, `EXISTS`, `ANY/ALL` | same |
| Derived table without alias | add `AS t` |
| Oracle-compat `(+)` outer joins | explicit `LEFT/RIGHT JOIN` |

### 8.9 Types in SQL text
`CLOB(n)`/`DBCLOB` → `TEXT`; `BLOB`/`FOR BIT DATA` → `BYTEA`; `DECFLOAT` → `NUMERIC`;
`DOUBLE` → `DOUBLE PRECISION`; `GRAPHIC/VARGRAPHIC` → `VARCHAR`; `TIMESTAMP(p>6)` → `TIMESTAMP(6)`;
`CHAR(n) FOR BIT DATA` → `BYTEA`; `XML` → `XML`; `BOOLEAN` → `BOOLEAN`.

### 8.10 Things that are the same — do not touch
`CASE`, `COALESCE`, `NULLIF`, `CAST`, `||`, `LIKE`, `IN`, `BETWEEN`, `EXISTS`, joins,
CTEs (non-recursive), window functions, `FETCH FIRST n ROWS ONLY`, `OFFSET … ROWS`,
`'' <> NULL` semantics (both distinguish empty string from NULL), `''` quote escaping,
`--` and `/* */` comments, NULL sort order (NULLs high in both), integer division truncation.

### 8.11 Error codes (Java side)
| DB2 SQLCODE | SQLSTATE | Spring exception to catch instead |
|---|---|---|
| -803 duplicate key | 23505 | `DuplicateKeyException` (subclass of `DataIntegrityViolationException`) |
| -530 / -532 FK violation | 23503 | `DataIntegrityViolationException` |
| -407 NOT NULL violation | 23502 | `DataIntegrityViolationException` |
| -545 check constraint | 23514 | `DataIntegrityViolationException` |
| -911 / -913 deadlock or lock timeout | 40P01 / 55P03 | `DeadlockLoserDataAccessException` / `CannotAcquireLockException` |
| -204 object not found | 42P01 | `BadSqlGrammarException` |
| -302 / -433 value too long | 22001 | `DataIntegrityViolationException` |
Delete code that parses DB2 message text or `DB2Diagnosable`. If raw SQLSTATE is needed,
use `((SQLException) e.getCause()).getSQLState()` — never the vendor error code.

## 9. Semantic-change watchlist

Each item below is a behavior difference that compiles fine and fails or drifts at
runtime. In Phase 5, confirm each is either *not applicable* or recorded in the summary
(**⚠ CRITICAL** when it can reach a consumer of this service).

1. **A failed statement aborts the whole PostgreSQL transaction.** DB2 lets you catch an
   error and keep issuing SQL in the same transaction; PostgreSQL answers every later
   statement with *current transaction is aborted*. Grep for `try { … save/saveAndFlush/
   executeUpdate … } catch (DataIntegrityViolationException|DuplicateKeyException|
   PersistenceException …)` inside `@Transactional` methods. Fix with
   `Propagation.REQUIRES_NEW` on the attempted operation, a check-before-insert, or
   `INSERT … ON CONFLICT DO NOTHING`. Highest-priority CRITICAL pattern.
2. **`WITH UR` removal.** Those reads now see only committed data; code polling another
   transaction's uncommitted progress will see nothing until commit. (DB2 CS ≈ PG READ
   COMMITTED otherwise; PG readers never block on writers.)
3. **String ordering / collation.** DB2 LUW defaults to codepoint order (uppercase before
   lowercase, digits before letters, punctuation significant). PostgreSQL uses the
   database's collation; `en_US.UTF-8` sorts linguistically and case-insensitively at the
   first level. `ORDER BY` on text, `<`/`BETWEEN` on strings, and anything exported in
   sorted order can change. Ask the DBA for `datcollate`; if it is not `C`/`C.UTF-8`,
   record every user-visible ordering as CRITICAL (or add `COLLATE "C"` where the old
   order is required).
4. **Aggregate result types.** `AVG(integer)` was a truncated INTEGER in DB2 and is
   `numeric` in PG; `SUM(integer)` is `bigint`. JSON output shape changes (`5` → `5.0…`).
   Fix the Java cast and wrap in `CAST(… AS INTEGER)` only where the old truncation was
   the intended contract.
5. **Decimal arithmetic.** DB2 fixes result precision/scale by rule (and truncates to fit
   31 digits); PG `numeric` is exact and unbounded. Money math done in SQL can differ in
   trailing digits — review `ROUND`/`TRUNC` placement in any monetary query.
6. **Integer conversion.** DB2 `INTEGER(x)`/`INT(x)` truncates; PG `CAST(x AS INTEGER)`
   rounds. Always `CAST(TRUNC(x) AS INTEGER)` (§8.6).
7. **`TIMESTAMPDIFF` and `ADD_MONTHS`.** DB2 estimates (30-day months, 365-day years);
   PG is exact. `ADD_MONTHS` snaps end-of-month in DB2 only. Results near month/year
   boundaries change (§8.4).
8. **Time zones.** `LOCALTIMESTAMP` in PG is evaluated in the session `TimeZone`, which the
   JDBC driver sets from the JVM default. If the app JVM and the DB2 server were in
   different zones, stored "now" values shift. Record the zone of each environment.
9. **Timestamp precision.** DB2 `TIMESTAMP(7..12)` truncates to 6 digits in PG; any
   uniqueness or ordering that relied on sub-microsecond precision breaks.
10. **`CHAR(n)` trailing spaces.** PG strips them when casting to text, so `||`, `LIKE`,
    `LENGTH` and `SUBSTR` on `char(n)` columns differ from DB2; `=` comparison is unaffected.
11. **Implicit casting is strict.** `int_col = varchar_col`, `COALESCE(int_col, '')`, mixed
    `CASE` branch types, and a `String` parameter bound to a numeric column all fail at
    **runtime**. Every native query must be executed at least once before go-live; list
    the ones no test or syntax check covered under *Remaining work*.
12. **Result-label case.** Unquoted aliases come back lowercase (were uppercase). Interface
    projections and any `rs.getObject("ALIAS")`-by-name code on camelCase aliases need
    quoted aliases (§7.7).
13. **Sequence allocation.** `allocationSize` ≠ sequence `INCREMENT BY` → duplicate keys
    or large gaps. Never guess; it's a DBA question.
14. **Upserts.** `INSERT … ON CONFLICT` needs a unique constraint/index on the conflict
    target; `MERGE` without one can insert duplicates where DB2 updated.
15. **Temp tables.** `SESSION.` tables were per-connection in DB2 too, but pooled PG
    connections keep temp tables alive across requests unless `ON COMMIT DROP`.
16. **`GENERATE_UNIQUE()` → `gen_random_uuid()`** changes the type (13-byte `CHAR FOR
    BIT DATA` → `uuid`) and any column/API that carried it.
17. **`LOG(x)`** is natural log in DB2, base 10 in PG (§8.6).
18. **`CONCAT()`** treats NULL as empty in PG; keep `||` for DB2 NULL semantics (§8.5).
19. **Lock modes.** `LOCK TABLE … IN EXCLUSIVE MODE` has different blocking semantics in
    PG; `FOR UPDATE` on joined/aggregated queries has different restrictions. Review each.
20. **Boolean flags.** If the DDL turns `CHAR(1)`/`SMALLINT` flags into `boolean`, every
    predicate comparing them to `'Y'`/`1` must change (§7.2).

## 10. Verification tooling

### 10.1 Offline syntax check with the real PostgreSQL parser (no database required)
`pglast` wraps libpg_query, the actual PostgreSQL grammar. Save the script below as
`tools/pg_syntax_check.py` (creating `tools/` is allowed), then:

```
pip install pglast        # or: pip install --user pglast
python3 tools/pg_syntax_check.py src/main/java
```

It extracts SQL from `@Query(nativeQuery = true)`, `@NamedNativeQuery`,
`createNativeQuery(...)`, `@SQLRestriction`/`@Where`, `@Formula`, and `@Subselect`,
replaces JPA parameters (`:name`, `?1`, `:#{…}`) with `$n`, and parses each. A PASS
means the statement is valid PostgreSQL *syntax* — it does not prove the columns exist or
the types agree. Queries held in constants or built dynamically are not followed; list
those and check them by hand. DB2 leftovers such as `WITH UR`, `+ 30 DAYS`,
`OPTIMIZE FOR`, bare `VALUES 1`, `DAYS(CURRENT DATE)` and function-style `TIMESTAMP(x)`
fail here. It does **not** catch analysis-time errors — a missing `RECURSIVE`,
alias-qualified `SET x.col`, `SYSIBM.SYSDUMMY1` (parses as an ordinary table), unknown
columns, type mismatches — those need the database-backed checks in §10.2.

```python
#!/usr/bin/env python3
"""Parse every native SQL string under src/main/java with the real PostgreSQL grammar.
Usage: python3 tools/pg_syntax_check.py [src/main/java]   (exit 1 if anything fails)"""
import re, sys, pathlib
try:
    from pglast import parse_sql
except ImportError:
    sys.exit("pglast is not installed: pip install pglast")

ROOT = pathlib.Path(sys.argv[1] if len(sys.argv) > 1 else "src/main/java")

# marker -> (attribute names that hold SQL, wrapper for fragments)
TARGETS = {
    "@Query(":            (("value", "countQuery"), None),
    "@NamedNativeQuery(": (("query",), None),
    "createNativeQuery(": (("value",), None),
    "@SQLRestriction(":   (("value",), "SELECT 1 WHERE {}"),
    "@Where(":            (("clause",), "SELECT 1 WHERE {}"),
    "@Formula(":          (("value",), "SELECT {}"),
    "@Subselect(":        (("value",), None),
}

def unescape(s):
    return s.replace('\\"', '"').replace("\\n", "\n").replace("\\t", "\t").replace("\\\\", "\\")

def balanced_body(src, i):
    """src[i] == '('. Return the text inside the matching parentheses (string-aware)."""
    depth, j, n = 0, i, len(src)
    while j < n:
        if src.startswith('"""', j):
            k = src.find('"""', j + 3); j = (k + 3) if k >= 0 else n; continue
        if src[j] == '"':
            j += 1
            while j < n and src[j] != '"':
                j += 2 if src[j] == "\\" else 1
            j += 1; continue
        if src[j] == "(":
            depth += 1
        elif src[j] == ")":
            depth -= 1
            if depth == 0:
                return src[i + 1:j]
        j += 1
    return src[i + 1:]

def split_attrs(body):
    """Top-level comma-split of an annotation/call body -> [(attr_name, concatenated_sql)]."""
    chunks, cur, lits, depth, i, n = [], [], [], 0, 0, len(body)
    while i < n:
        if body.startswith('"""', i):
            k = body.find('"""', i + 3); k = n if k < 0 else k
            lits.append(body[i + 3:k]); cur.append("S"); i = k + 3; continue
        if body[i] == '"':
            j = i + 1
            while j < n and body[j] != '"':
                j += 2 if body[j] == "\\" else 1
            lits.append(unescape(body[i + 1:j])); cur.append("S"); i = j + 1; continue
        c = body[i]
        if c in "([{":
            depth += 1
        elif c in ")]}":
            depth -= 1
        elif c == "," and depth == 0:
            chunks.append(("".join(cur), "".join(lits))); cur, lits = [], []; i += 1; continue
        cur.append(c); i += 1
    chunks.append(("".join(cur), "".join(lits)))
    out = []
    for text, sql in chunks:
        m = re.match(r"\s*(\w+)\s*=", text)
        out.append((m.group(1) if m else "value", sql))
    return out

def to_pg_params(sql):
    n = [0]
    def repl(_m):
        n[0] += 1
        return f"${n[0]}"
    sql = re.sub(r"\{h-\w+\}", "", sql)                 # Hibernate {h-schema} placeholders
    sql = re.sub(r"\?\d+", repl, sql)                   # ?1 positional
    sql = re.sub(r"(?<!:):#\{[^}]*\}", repl, sql)       # :#{spel}
    sql = re.sub(r"(?<!:):([A-Za-z_]\w*)", repl, sql)   # :name   (leaves ::casts alone)
    return sql

total = failed = 0
for path in sorted(ROOT.rglob("*.java")):
    src = path.read_text(encoding="utf-8", errors="replace")
    for marker, (attrs, wrapper) in TARGETS.items():
        for m in re.finditer(re.escape(marker), src):
            body = balanced_body(src, m.end() - 1)
            if marker == "@Query(" and not re.search(r"nativeQuery\s*=\s*true", body):
                continue
            line = src.count("\n", 0, m.start()) + 1
            parts = split_attrs(body)
            if marker == "createNativeQuery(":
                parts = parts[:1]
            for name, sql in parts:
                if name not in attrs or not sql.strip():
                    continue
                stmt = to_pg_params(sql)
                if wrapper:
                    stmt = wrapper.format(stmt)
                total += 1
                try:
                    parse_sql(stmt)
                    print(f"PASS  {path}:{line}  {name}")
                except Exception as e:  # pglast.parser.ParseError
                    failed += 1
                    print(f"FAIL  {path}:{line}  {name}  -> {e}")
print(f"\n{total} statements checked, {failed} failed")
sys.exit(1 if failed else 0)
```

### 10.2 With a Postgres DDL (when the DBA provides it)
1. `docker run --rm -d --name pgmig -e POSTGRES_PASSWORD=pg -p 5432:5432 postgres:16`
   (use the real target version once known), then `psql` the DDL in.
2. Run the app once with `spring.jpa.hibernate.ddl-auto=validate` pointed at it. Fix
   genuine mapping errors; note that `validate` is strict about `varchar` vs `text` and
   can flag harmless differences — decide each, then set `ddl-auto` back to `none`.
3. `EXPLAIN` every native query with placeholder values (via `psql` or a scratch
   `JdbcTemplate` test). This catches missing columns, type mismatches and ambiguous
   references that the syntax check cannot.
4. Optional, only if asked: Testcontainers smoke tests. Boot 4 dependencies (test scope,
   versions managed by Boot): `spring-boot-starter-test`, `spring-boot-testcontainers`,
   `org.testcontainers:testcontainers-postgresql` (Testcontainers 2.x name; if Boot's
   managed version is 1.x it is `org.testcontainers:postgresql`). Boot 4 moved the test
   slices into per-technology modules — if `@DataJpaTest` does not resolve, add the
   matching `spring-boot-starter-data-jpa-test` starter after confirming the artifact in
   the Boot BOM. Skeleton:

   ```java
   @DataJpaTest
   @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
   @Testcontainers
   @Sql("/postgres-ddl.sql")              // DBA's DDL copied to src/test/resources
   class NativeQuerySmokeTest {
       @Container @ServiceConnection
       static PostgreSQLContainer postgres = new PostgreSQLContainer("postgres:16"); // add <?> on Testcontainers 1.x
       @Autowired SomeRepository repo;
       @Test void eachNativeQueryExecutes() { repo.findActiveSince(LocalDate.now(), PageRequest.of(0, 5)); }
   }
   ```
   One test method per native query, with representative (including null) parameters.

## 11. `postgres-migration-summary.md` — required deliverable

Maintain this file at the repository root from Phase 0 onward. It is written for a human
reviewer who has not read the diff. Update it **in the same turn** as the code change it
describes; never let code and summary drift.

```markdown
# PostgreSQL migration summary
Last updated: YYYY-MM-DD · Current phase: N · Compile: pass/fail · Tests: pass/fail · Syntax check: n/m

## ⚠ CRITICAL — contract changes and external impacts
<!-- Anything a consumer of this service, another system, a report, a job, or an operator could notice. -->
- **What changed** — where (`path:line`) — **who is affected** (API consumer / downstream system / report / scheduled job / ops) — why unavoidable — recommended action / decision needed

## Assumptions pending DDL
| Assumption | Affected code | If wrong |
|---|---|---|

## Open questions for DBA / ops
- [ ] …

## Changes by phase
### Phase 1 — Dependencies & configuration
### Phase 2 — Entities & mappings
### Phase 3 — Native SQL
| File:line | Method | DB2 idioms | Translation notes | Difficulty | Verified by |
|---|---|---|---|---|---|
### Phase 4 — Java-side coupling

## Inventory (Phase 0)
## Verification log
| Date | Command | Result |
|---|---|---|
## Observations (not changed — outside migration scope)
## Remaining work
```

Rules for the summary:
- **CRITICAL stays at the top** and is the first thing you update. Typical CRITICAL
  entries: a JSON field changing numeric type or precision (§9.4), sort order of any
  list or export (§9.3), timestamps shifting zones (§9.8), a `WITH UR` read that fed a
  status endpoint (§9.2), a UUID replacing `GENERATE_UNIQUE` (§9.16), any `try/catch`
  recovery pattern inside a transaction (§9.1), a changed error response because
  SQLCODE mapping changed (§8.11), any REST/DTO/message/export change at all.
- Every `// TODO(pg-migration): …` in code has a summary line and vice versa. When a TODO
  is resolved, mark the line `✅` with the date instead of deleting it.
- Entries are one to three lines with `path:line`; no code dumps, no restating the diff.
- Record what was verified and how (compile, unit tests, syntax check, EXPLAIN, validate
  run) — and explicitly what was **not** verifiable yet.
- The *Observations* section is for non-migration issues you noticed and deliberately
  did not touch.

## 12. Seed questions for the DBA / ops

Add these to the summary in Phase 0 and refine as you go:

1. Target PostgreSQL major version and hosting (self-managed, RDS, Aurora, Cloud SQL…)?
2. Identifier policy: unquoted (lowercase-folded) table and column names? Schema name
   and whether the app should set `currentSchema`/`search_path`?
3. Database collation (`datcollate` / `datctype`)? Is `C` or `C.UTF-8` acceptable to
   preserve DB2 ordering?
4. Type policy for `CHAR(1)` Y/N and `SMALLINT` flags — preserved or `boolean`?
5. `CLOB`/`BLOB` → `text`/`bytea` confirmed (not large objects)?
6. Timestamp precision used (DB2 columns > 6 digits?) and the server `TimeZone` setting.
7. For each sequence: name and `INCREMENT BY`; for each identity column: `ALWAYS` vs `BY DEFAULT`.
8. Unique constraints/indexes available for every upsert target.
9. Any stored procedures or functions ported? As `PROCEDURE` or `FUNCTION`?
10. Indexes supporting `LIKE 'prefix%'` under a non-`C` collation (`text_pattern_ops`)?
11. Connection limits for pool sizing; whether `pgbouncer`/pooling in transaction mode is
    in front of the database (affects temp tables, `SET`, prepared statements).
12. Is there a copy of the DDL the app team can use for local verification (§10.2)?

## 13. Definition of done

- `rg -in "db2|ibm|sysibm|jcc" pom.xml src` returns nothing.
- `mvn -q clean test` passes; the syntax check passes for every extracted statement.
- Every native query, `@Formula`, `@SQLRestriction`, and `@Subselect` has been rewritten
  and appears in the Phase 3 table with a *Verified by* value.
- Every `@Lob` is gone; every ID strategy is explicit (no `AUTO`); no DB2 `columnDefinition`.
- No `com.ibm.*` imports, SQLCODE checks, or JDBC LOB API calls remain.
- Every §9 item is marked *n/a* or recorded in the summary.
- `postgres-migration-summary.md` is current, with CRITICAL items and DBA questions on top.

## 14. Project-specific notes

<!-- Optional. Fill in per service; everything above applies to all services on this stack. -->
- Service: _(name / one-line purpose)_
- Postgres schema name: _(…)_ · Target version: _(…)_
- DDL location, if provided: _(path)_
- Paths that hold most native SQL: _(e.g., src/main/java/…/repository)_
- Anything excluded from this migration: _(…)_
