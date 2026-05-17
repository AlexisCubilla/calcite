# Jira Bug Report Template

## Title
```
[CALCITE-7483 Regression] Ambiguous column reference in generated SQL for JOINs with FETCH/LIMIT on PostgreSQL
```

## Description

Calcite 1.42.0-SNAPSHOT generates SQL with ambiguous column references when converting a RelNode containing JOINs with FETCH/LIMIT clauses to PostgreSQL dialect. The query fails at execution with:

```
ERROR: column reference "id" is ambiguous
```

This is a regression introduced by commit 8a7ee97f6 (CALCITE-7483), which extended the `supportGenerateSelectStar` mechanism to the `builder()` method.

## Environment

- Calcite version: 1.42.0-SNAPSHOT (main branch after commit 8a7ee97f6)
- Target dialect: PostgreSQL
- Works in: Calcite 1.41.0

## Root Cause

When a dialect returns `false` for `supportGenerateSelectStar()` (PostgreSQL does this for JOINs with duplicate field names per CALCITE-5583), Calcite expands `SELECT *` to explicit column references. The expansion logic determines whether to qualify column names based on:

```java
boolean qualified = !dialect.hasImplicitTableAlias() || aliases.size() > 1;
```

For PostgreSQL (`hasImplicitTableAlias() = true`) with a single subquery alias (`aliases.size() = 1`), this evaluates to `qualified = false`. The expanded columns are left unqualified, and when the underlying JOIN produces duplicate column names (e.g., multiple `id` columns from different tables), the resulting subquery contains duplicate unqualified column names that PostgreSQL cannot resolve.

This affects two methods in `SqlImplementor.java`:
- `builder()` method (~line 2016)
- `maybeExpandStar()` method (~line 2392)

## Reproduction

### Schema

```sql
CREATE TABLE cred_solicitud (
    id_solicitud INT PRIMARY KEY,
    process_id VARCHAR(50),
    estado_solicitud INT,
    fecha_aprob_comite DATE
);

CREATE TABLE v_usuario_alta (
    process_id VARCHAR(50),
    id_usuario INT
);

CREATE TABLE cred_solicitante_x_solicitud (
    id_solicitud INT,
    dat_per_departamento INT,
    dat_per_ciudad INT,
    dat_per_distrito INT
);

CREATE TABLE departamento (
    id INT PRIMARY KEY,
    nombre VARCHAR(100)
);

CREATE TABLE distrito_ips (
    id INT PRIMARY KEY,
    departamentoid INT
);

CREATE TABLE localidad_ips (
    id INT PRIMARY KEY,
    distritoid INT,
    departamentoid INT
);
```

### Query (RelNode equivalent)

A query joining these tables with a FETCH/LIMIT clause:

```sql
SELECT s.id_solicitud, s.process_id, s.estado_solicitud,
       ua.id_usuario,
       d.id AS dept_id, d.nombre AS dept_nombre,
       di.id AS dist_id, di.departamentoid,
       l.id AS loc_id, l.distritoid, l.departamentoid AS loc_dept_id
FROM cred_solicitud s
INNER JOIN v_usuario_alta ua ON s.process_id = ua.process_id
INNER JOIN cred_solicitante_x_solicitud cs ON s.id_solicitud = cs.id_solicitud
LEFT JOIN departamento d ON cs.dat_per_departamento = d.id
LEFT JOIN distrito_ips di ON cs.dat_per_ciudad = di.id AND d.id = di.departamentoid
LEFT JOIN localidad_ips l ON cs.dat_per_distrito = l.id AND di.id = l.distritoid AND di.departamentoid = l.departamentoid
WHERE s.estado_solicitud = 129 AND s.fecha_aprob_comite >= DATE '2026-05-01'
FETCH NEXT 10000 ROWS ONLY;
```

### Actual Output (1.42.0-SNAPSHOT) - FAILS

```sql
SELECT "t8"."id_solicitud", "t8"."process_id", "t8"."estado_solicitud",
       "t8"."id_usuario", "t8"."id", "t8"."id", "t8"."departamentoid",
       "t9"."id", "t9"."distritoid", "t9"."departamentoid"
FROM (
  SELECT "t6"."id_solicitud", "t6"."process_id", "t6"."estado_solicitud",
         "t6"."id_usuario", "t6"."id", "t7"."id", "t7"."departamentoid"
  FROM (
    SELECT "t0"."id_solicitud", "t0"."process_id", "t0"."estado_solicitud",
           "t2"."id_usuario", "t5"."id"
    FROM "cred_solicitud" AS "t0"
    INNER JOIN "v_usuario_alta" AS "t2" ON "t0"."process_id" = "t2"."process_id"
    INNER JOIN "cred_solicitante_x_solicitud" AS "t3" ON "t0"."id_solicitud" = "t3"."id_solicitud"
    LEFT JOIN "departamento" AS "t5" ON "t3"."dat_per_departamento" = "t5"."id"
    WHERE "t0"."estado_solicitud" = 129 AND "t0"."fecha_aprob_comite" >= DATE '2026-05-01'
    FETCH NEXT 10000 ROWS ONLY
  ) AS "t6"
  LEFT JOIN "distrito_ips" AS "t7" ON "t6"."dat_per_ciudad" = "t7"."id" AND "t6"."id" = "t7"."departamentoid"
  FETCH NEXT 10000 ROWS ONLY
) AS "t8"
LEFT JOIN "localidad_ips" AS "t9"
  ON "t8"."dat_per_distrito" = "t9"."id" AND "t8"."id" = "t9"."distritoid" AND "t8"."departamentoid" = "t9"."departamentoid"
FETCH NEXT 10000 ROWS ONLY
```

PostgreSQL error:
```
ERROR: column reference "id" is ambiguous
Position: 2214
```

The subquery `"t6"` contains `"t6"."id"` (from `departamento`) and `"t7"."id"` (from `distrito_ips`), both unqualified in the outer select list. PostgreSQL cannot determine which `id` column is intended.

### Expected Output (1.41.0) - WORKS

```sql
SELECT "t1"."id_solicitud", "t1"."process_id", "t1"."estado_solicitud",
       "t2"."id_usuario",
       "t5"."id" AS "dept_id", "t5"."nombre",
       "t7"."id" AS "dist_id", "t7"."departamentoid",
       "t9"."id" AS "loc_id", "t9"."distritoid", "t9"."departamentoid"
FROM "cred_solicitud" AS "t1"
INNER JOIN "v_usuario_alta" AS "t2" ON "t1"."process_id" = "t2"."process_id"
INNER JOIN "cred_solicitante_x_solicitud" AS "t3" ON "t1"."id_solicitud" = "t3"."id_solicitud"
LEFT JOIN "departamento" AS "t5" ON "t3"."dat_per_departamento" = "t5"."id"
LEFT JOIN "distrito_ips" AS "t7" ON "t3"."dat_per_ciudad" = "t7"."id" AND "t5"."id" = "t7"."departamentoid"
LEFT JOIN "localidad_ips" AS "t9" ON "t3"."dat_per_distrito" = "t9"."id" AND "t7"."id" = "t9"."distritoid" AND "t7"."departamentoid" = "t9"."departamentoid"
WHERE "t1"."estado_solicitud" = 129 AND "t1"."fecha_aprob_comite" >= DATE '2026-05-01'
FETCH NEXT 10000 ROWS ONLY
```

## Key Differences

| Aspect | 1.41.0 | 1.42.0-SNAPSHOT |
|--------|--------|-----------------|
| Subquery wrapping | None | Nested subqueries per FETCH |
| SELECT list | `SELECT *` preserved | Expanded to explicit columns |
| Column qualification | N/A (star) | Unqualified, causing ambiguity |
| FETCH placement | Single at top level | Pushed to each subquery |

## Suggested Fix

### Option 1: Force qualified column expansion

In `SqlImplementor.java`, force `qualified = true` when expanding `SELECT *` in both `builder()` and `maybeExpandStar()`:

```java
// builder() method (~line 2031)
if (!dialect.supportGenerateSelectStar(rel.getInput(0))) {
  final Context expandContext = aliasContext(
      newAliases != null ? newAliases : aliases, true);
  ...
}

// maybeExpandStar() method (~line 2391)
if (expectedRel != null && !expectedRel.getInputs().isEmpty()
    && select.getSelectList().equals(SqlNodeList.SINGLETON_STAR)
    && !dialect.supportGenerateSelectStar(expectedRel.getInput(0))) {
  final Context ctx = aliasContext(aliases, true);
  ...
}
```

### Option 2: Generate unique aliases for duplicate column names

When expanding `SELECT *` for a subquery with duplicate field names, generate unique aliases:

```sql
-- Instead of:
SELECT "t6"."id", "t7"."id", "t7"."departamentoid"
-- Generate:
SELECT "t6"."id" AS "id", "t7"."id" AS "id0", "t7"."departamentoid" AS "departamentoid"
```

This requires detecting duplicate field names during expansion and appending numeric suffixes.

## Related Issues

- CALCITE-5583: Introduced `supportGenerateSelectStar()` — PostgreSQL returns `false` for JOINs with duplicate fields
- CALCITE-7483: Extended `supportGenerateSelectStar` to `builder()` method (regression source)

## Files Affected

- `core/src/main/java/org/apache/calcite/rel/rel2sql/SqlImplementor.java`
  - `builder()` method
  - `maybeExpandStar()` method
