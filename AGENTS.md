# Apache Calcite - Agent Rules & Starting Point

This is a fork of Apache Calcite for bug investigation and fix development.

## Repo Setup

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `git@github.com:AlexisCubilla/calcite-cybira.git` | Your fork |
| `upstream` | `https://github.com/apache/calcite.git` | Apache official |

### Branches

| Branch | Description |
|--------|-------------|
| `main` | Synced with `upstream/main` |
| `cybira/dev` | Internal tooling + combined fixes |
| `CALCITE-5212-decimal-digest-scale` | Fix for DECIMAL scale in type digest |
| `CALCITE-7524-jdbc-decimal-zero-precision` | Fix for JDBC DECIMAL precision 0 |
| `CALCITE-7483-fix-select-star-qualified` | Fix for ambiguous column references in RelNode-to-SQL |

## Build System

- **Java:** 21 (JDK 8, 11, 17, 21, or 23 also supported)
- **Build tool:** Gradle 8.7 — always use the wrapper `./gradlew`
- **Version:** Defined in `gradle.properties` as `calcite.version`

### Essential Commands

```bash
# Build (skip tests for speed)
./gradlew build -x test

# Run core tests only (no external DB needed)
./gradlew :core:test

# Run a specific test
./gradlew :core:test --tests "*ClassName*"
./gradlew :core:test --tests "*ClassName.methodName*"

# Style checks (required for PR)
./gradlew style
./gradlew autostyleCheck checkstyleAll

# Auto-fix style issues
./gradlew autostyleApply

# Generate sources (parser, etc.)
./gradlew generateSources

# Publish to local Maven repo
./gradlew :linq4j:publishToMavenLocal :core:publishToMavenLocal :testkit:publishToMavenLocal \
  :file:publishToMavenLocal :example:csv:publishToMavenLocal :server:publishToMavenLocal
```

## Module Structure

| Module | Purpose |
|--------|---------|
| `core/` | SQL parser, validator, optimizer, type system, RelNode-to-SQL |
| `linq4j/` | LINQ-like query framework |
| `testkit/` | Shared test utilities |
| `babel/` | Additional SQL dialects |
| `arrow/` | Apache Arrow adapter |
| `server/` | Calcite server extensions |
| `plus/` | Additional features |
| `file/`, `cassandra/`, `druid/`, `elasticsearch/`, `mongodb/`, `redis/`, `kafka/`, `spark/`, `splunk/`, `geode/`, `innodb/`, `pig/`, `piglet/`, `sqlline/` | Data source adapters |

## Key Source Locations

### RelNode-to-SQL (SQL Generation)
- `core/src/main/java/org/apache/calcite/rel/rel2sql/SqlImplementor.java` — Core SQL generation
- `core/src/main/java/org/apache/calcite/sql/dialect/PostgresqlSqlDialect.java` — PostgreSQL dialect
- `core/src/main/java/org/apache/calcite/sql/SqlDialect.java` — Base dialect interface

### Type System
- `core/src/main/java/org/apache/calcite/sql/type/BasicSqlType.java` — SQL type generation
- `core/src/main/java/org/apache/calcite/rel/type/RelDataTypeFactoryImpl.java` — Type factory with `canonize()`
- `core/src/main/java/org/apache/calcite/rel/type/RelDataTypeImpl.java` — Type `equals()` and `hashCode()`

### JDBC
- `core/src/main/java/org/apache/calcite/jdbc/JdbcSchema.java` — JDBC schema metadata

### Tests
- `core/src/test/java/org/apache/calcite/rel/rel2sql/RelToSqlConverterTest.java` — RelNode-to-SQL tests
- `core/src/test/java/org/apache/calcite/sql/type/SqlTypeFactoryTest.java` — Type system tests

## Workflow for Fixing a Bug

1. **Identify the bug** — Read the Jira issue, understand expected vs actual behavior
2. **Find the code** — Use the key source locations above or grep for relevant classes
3. **Write a failing test** — Add a test case in the appropriate `*Test.java` file that reproduces the bug
4. **Implement the fix** — Modify the source code
5. **Verify** — Run `./gradlew :core:test --tests "*TestName*"` and ensure the test passes
6. **Style check** — Run `./gradlew style autostyleCheck checkstyleAll`
7. **Commit** — Use format: `[CALCITE-XXXX] Short description`
8. **Push and PR** — Push to `origin`, create PR targeting `upstream/main`

## PR Rules for Apache Calcite

1. **One change per PR** — Do not mix unrelated fixes
2. **Jira issue required** — Title must include `[CALCITE-XXXX]`
3. **Tests required** — Every fix must have a test case
4. **Style must pass** — `./gradlew style` must be green
5. **Version for upstream** — Use `calcite.version=1.42.0` (no `cybira` suffix) in `gradle.properties` for the PR branch
6. **ICLA** — First-time ASF contributors need an [ICLA](https://www.apache.org/licenses/contributor-agreements.html)

## Consuming Patched Calcite Locally

Before the PR is merged, publish to local Maven and use in consumer projects:

```bash
cd calcite-main
./gradlew :core:publishToMavenLocal
```

In the consumer's `pom.xml`, set:
```xml
<calcite.patch.version>1.42.0-cybira-SNAPSHOT</calcite.patch.version>
```

## Active Bug Reports

| Jira | Branch | Status |
|------|--------|--------|
| CALCITE-5212 | `CALCITE-5212-decimal-digest-scale` | Fix implemented |
| CALCITE-7524 | `CALCITE-7524-jdbc-decimal-zero-precision` | Fix implemented |
| CALCITE-7483 (regression) | `CALCITE-7483-fix-select-star-qualified` | Fix implemented, bug report in `BUG_REPORT_CALCITE-7483-REGRESSION.md` |

## Documentation

- `site/_docs/howto.md` — Build instructions
- `.github/workflows/main.yml` — CI pipeline (check before PR)
- `README.md` — Project overview
