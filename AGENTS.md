# Apache Calcite Working Copy

## Focus

Fix for CALCITE-5212: Decimal with scale specified but precision not specified returns scale=0.

## Setup (mise)

```bash
# Activar tools del proyecto (ya configurado en .mise.toml)
mise use java@21.0.2 gradle@8.14.4

# O activar en shell actual
eval "$(mise activate bash)"
```

## Build & Test

```bash
# Usar 'mise exec --' antes de comandos gradle
mise exec -- ./gradlew build -x test

mise exec -- ./gradlew :core:test

mise exec -- ./gradlew :core:test --tests "*SqlTypeFactoryImpl*"

# Code style check
mise exec -- ./gradlew style
mise exec -- ./gradlew autostyleCheck checkstyleAll
```

## Bug Location

- `core/src/main/java/org/apache/calcite/sql/type/BasicSqlType.java:197-208`
- `generateTypeString()` - scale only included when precision is specified

## Scope

Modify only:
- `core/src/main/java/org/apache/calcite/sql/type/BasicSqlType.java`
- `core/src/test/java/org/apache/calcite/rel/type/*Test.java` (add tests)

DO NOT modify:
- `site/`, `example/`, any adapter modules (unless the Jira explicitly covers them)
- Any file outside the scope agreed on the Jira

**`gradle.properties`:** for a PR to `apache/calcite`, keep `calcite.version` aligned with upstream (e.g. `1.42.0`). A distinct suffix (e.g. `1.42.0-cybira`) is only for local `publishToMavenLocal` / fork artifacts and must not be part of the upstream patch unless release managers request it.

## Verification

Must pass before commit:
- `./gradlew :core:test --tests "*Decimal*" --tests "*SqlTypeFactoryImpl*"`
- `./gradlew style`
- `./gradlew autostyleCheck checkstyleAll`

## Commit/PR

- Branch: `CALCITE-5212-fix`
- Message: "CALCITE-5212: [description]"
- PR: https://github.com/apache/calcite

## Notes

- Use `mise` to manage Java/Gradle (see Setup section above)
- Generated sources: `./gradlew generateSources` if parser files missing
- Integration tests require external VMs (not needed for this fix)
