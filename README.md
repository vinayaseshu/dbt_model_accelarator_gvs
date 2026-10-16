# dbt Model Accelerator v2.3

Turns a source-to-target mapping spreadsheet into a dbt project slice that follows dbt Labs'
recommended structure: CTE-style models, YAML docs and tests, an optional staging layer,
optional contracts, and CI drift checks.

## Quick start
    pip install openpyxl
    python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output              # no staging layer
    python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output --staging    # with staging layer
    # strict "production" profile
    python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
        --enforce-contracts --surrogate-key --audit-columns --freshness-field _loaded_at

Copy `dbt_output/models/*` into your project, merge `dbt_project_snippet.yml` into `dbt_project.yml`,
add `packages.yml` (if generated), then run `dbt deps && dbt build`.

## What gets generated
    dbt_output/
      models/<staging folder>/_<source>__sources.yml    sources (+ freshness)
      models/<staging folder>/_<folder>__models.yml     staging docs            (only if staging used)
      models/<staging folder>/stg_<source>__<table>.sql one per source table  (only if staging used)
      models/<target folder>/<model>.sql                one per MODEL_NAME
      models/<target folder>/_<folder>__models.yml      descriptions, data_type, tests, contracts, meta
      packages.yml                                     dbt_utils, only when needed
      dbt_project_snippet.yml                          folder-level materialization defaults
      .sqlfluff                                        lint rules matching the generated style
      generation_report.md                             validation results + lineage

## Best practices built in
| Practice | How |
|---|---|
| Staging layer (optional) | Off by default, so target models select from `source()` directly. `--staging` turns it on for every model, and the USE_STAGING column (Y/N) overrides the flag per model. When on, one `stg_<source>__<table>` view per source table selects only the columns the opted-in models use (including join, filter and `table.column` expression references), and those models `ref()` it. |
| Import / final CTE style | Each upstream is an import CTE, logic sits in `final`, ends with `select * from final`. |
| Folder and file conventions | Configurable per model/table (see Folders below); `_<folder>__models.yml` / `_<source>__sources.yml` naming. |
| Documentation | MODEL_DESCRIPTION / COLUMN_DESCRIPTION go into YAML; defaults describe the lineage (`Mapped from hr.emp.ename via upper(ename)`). `meta` records the generator version and mapping file. |
| Tests | Keys get `not_null` + `unique` automatically (composite keys: `dbt_utils.unique_combination_of_columns`). COLUMN_TESTS adds `not_null`, `unique`, `accepted_values(A|B)`, `relationships(model.col)` or any package test. Uses `data_tests:` + `arguments:` (dbt 1.10+); `--legacy-test-syntax` for older versions. |
| Model contracts | `--enforce-contracts` adds `contract: {enforced: true}` with `data_type` from TARGET_DATATYPE. A model with any untyped column is warned about and left uncontracted. |
| Surrogate keys | `--surrogate-key` replaces composite keys with `<model>_sk` built by `dbt_utils.generate_surrogate_key` and uses it as `unique_key`, so incremental merges stay on one key. |
| Audit columns | `--audit-columns` adds `dbt_loaded_at` and `dbt_invocation_id` for traceability back to the dbt run. |
| Source freshness | `--freshness-field <col>` adds warn 24h / error 48h freshness to every source. |
| Selectors and environments | TAGS -> `config(tags=...)`, SCHEMA -> `config(schema=...)`, TARGET_TABLE -> `alias` when it differs from the model name. |
| Incremental safety | `merge` strategy, `on_schema_change='append_new_columns'`, `is_incremental()` filter from INCREMENTAL_FILTER. |
| Linting | `.sqlfluff` with the dbt templater, lower-case keywords and explicit aliasing (`--dialect`, default snowflake). |
| CI drift check | `--check` regenerates in memory and fails if committed models differ from the mapping, so nobody hand-edits generated SQL. |
| Clean regeneration | `--clean` removes `OUTPUT/models` first, so models deleted from the mapping don't linger. |
| Validation gate | Errors (missing join, bad key, duplicate column, conflicting settings, alias clash) skip the model and exit 1. `--force` writes it with a TODO marker. |

## Mapping columns
Required: SOURCE_TABLE, SOURCE_COLUMN, TARGET_COLUMN, MODEL_NAME.

**TARGET_COLUMN forms**
| Form | Example | Output column |
|---|---|---|
| Plain name (rename) | `STATE_CODE` | `state_code` |
| Expression | `upper(channel)` | named after SOURCE_COLUMN (`channel`) |
| Expression with alias | `quantity * order_items.unit_price as gross_amount` | `gross_amount` |
| Jinja / cross-db macro | `{{ dbt.datediff('orders.order_date', 'orders.ship_date', 'day') }} as days_to_ship` | `days_to_ship` |

Only a trailing top-level `as <name>` counts as an alias: `cast(x as int)` is still an un-aliased expression.
Reference columns other than SOURCE_COLUMN as `table.column`, so they reach the staging model and are
unambiguous in joins.
Core: TARGET_TABLE, TARGET_DATATYPE (blank = no cast), MATERIALIZATION, UNIQUE_KEY, JOIN_CONDITION.
Optional: JOIN_TYPE, INCREMENTAL_FILTER, SOURCE_NAME, SCHEMA, TAGS, MODEL_DESCRIPTION,
COLUMN_DESCRIPTION, COLUMN_TESTS, USE_STAGING (Y/N), STAGING_MODEL_FOLDER, TARGET_MODEL_FOLDER.
(`MODEL_FOLDER` from older templates still works as TARGET_MODEL_FOLDER.) Model-level fields only need filling on a model's first row.
The template's README tab explains each column with an example.

## Flags
| Flag | Default | Purpose |
|---|---|---|
| `--sheet` | STM_MAPPING, else first sheet | Worksheet to read |
| `--source-name` / `--source-schema` | raw / - | Default source when SOURCE_NAME is blank |
| `--freshness-field` | off | Source freshness `loaded_at_field` |
| `--staging` / `--no-staging` | off | Default staging behaviour; USE_STAGING overrides per model |
| `--marts-dir` | marts | Default target folder when TARGET_MODEL_FOLDER is blank |
| `--staging-dir` | staging | Default staging root; tables go to `<staging-dir>/<source>` when STAGING_MODEL_FOLDER is blank |
| `--enforce-contracts` | off | Enforced model contracts |
| `--surrogate-key` | off | Hashed key for composite keys |
| `--audit-columns` | off | dbt_loaded_at, dbt_invocation_id |
| `--no-composite-test` | test on | Skip unique_combination_of_columns |
| `--legacy-test-syntax` | off | `tests:` with top-level args (dbt < 1.10) |
| `--project-name` / `--dialect` | my_dbt_project / snowflake | For the snippet and .sqlfluff |
| `--force` / `--clean` / `--check` | off | Run modes (see above) |

## Folders
All paths are relative to the `models/` root. Nested paths such as `marts/finance/reporting` are allowed,
and a leading `models/` is ignored.

| Column | Scope | Blank means | Example |
|---|---|---|---|
| TARGET_MODEL_FOLDER | per model (first row) | `--marts-dir` (default `marts`) | `marts/hr/reporting` |
| STAGING_MODEL_FOLDER | per source table (any row for that table) | `<--staging-dir>/<SOURCE_NAME>` (default `staging/<source>`) | `staging/hr/reference` |

- **Staging folder only matters for staged tables.** STAGING_MODEL_FOLDER is used only when a model with staging on reads that table.
- **Where sources.yml goes.** `_<source>__sources.yml` sits in the source's staging folder when all its tables share one; otherwise it goes in `<staging-dir>/<source>`.
- **Staging and target models in the same folder.** This is allowed. The staging models get an explicit `config(materialized='view')`, and their docs go in `_<folder>__staging_models.yml`.
- **dbt_project_snippet.yml.** Sets `+materialized: view` on staging folders and `table` on target folders, without repeating a setting a parent folder already applies.
- **Validation.** An absolute path, `..` or invalid characters is an error, and that model is skipped. A source table given two different STAGING_MODEL_FOLDER values is a warning, and the first value is used.

## Staging decision order
1. USE_STAGING on the model's first row: `Y` = staging, `N` = direct from source.
2. If blank, the `--staging` / `--no-staging` flag.
3. If neither is set, no staging.

The report has a Staging column per model, plus the staging model (if any) for each source table.

Verified with dbt-core 1.12 + dbt-duckdb, including nested custom staging and target folders, in all three modes (default, `--staging`, `--no-staging`), with a
per-model `Y` mixed in: all flags on gives PASS=20 on `dbt build` (6 models, 14 tests,
contracts enforced), and the incremental model merges new rows on a second run.
