# dbt Model Accelerator

**Version 2.4.0** · `generate_dbt_models.py`

Generates dbt models from a source-to-target mapping (STM) spreadsheet. Each row in the mapping
describes one target column. The accelerator turns those rows into:

- one `.sql` model per `MODEL_NAME`, in CTE style, with casts, joins and incremental logic
- one `.yml` file per model with descriptions, data types, tests and (optionally) contracts
- source definitions and an optional staging layer
- project scaffolding (`packages.yml`, folder config snippet, `.sqlfluff`) and a validation report

It targets **Snowflake, PostgreSQL, BigQuery, Databricks and Redshift**. Datatypes and incremental
strategy are translated for the platform you choose.

---

## Contents
1. [Requirements and installation](#1-requirements-and-installation)
2. [Quick start](#2-quick-start)
3. [End-to-end workflow](#3-end-to-end-workflow)
4. [Command-line reference](#4-command-line-reference)
5. [Mapping template reference](#5-mapping-template-reference)
6. [What gets generated](#6-what-gets-generated)
7. [Target databases](#7-target-databases)
8. [Staging layer and folders](#8-staging-layer-and-folders)
9. [Command recipes](#9-command-recipes)
10. [Validation, errors and exit codes](#10-validation-errors-and-exit-codes)
11. [CI/CD integration](#11-cicd-integration)
12. [Testing the generated models](#12-testing-the-generated-models)
13. [Troubleshooting](#13-troubleshooting)
14. [Limitations](#14-limitations)

---

## 1. Requirements and installation

| Need | Version | Why |
|---|---|---|
| Python | 3.9+ (tested on 3.11–3.13) | Runs the generator |
| openpyxl | any recent | Reads `.xlsx` mappings (not needed for `.csv`) |
| dbt-core + your adapter | 1.10+ recommended | Runs the generated models (not needed to generate them) |
| dbt_utils package | 1.1+ | Only if you use composite keys or `--surrogate-key` (added to `packages.yml` automatically) |

```bash
pip install openpyxl

# plus the adapter for your platform, when you want to run the models
pip install dbt-snowflake      # or dbt-postgres / dbt-bigquery / dbt-databricks / dbt-redshift
```

The generator is a single file with no other dependencies. Copy `generate_dbt_models.py` anywhere and
run it with Python.

```bash
python generate_dbt_models.py --version     # 2.4.0
python generate_dbt_models.py --help        # all options
```

---

## 2. Quick start

```bash
# 1. Fill in the STM_MAPPING tab of the template (one row per target column)
# 2. Generate models for your platform
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output --database snowflake

# 3. Read the report, then copy the models into your dbt project
cat dbt_output/generation_report.md
cp -r dbt_output/models/* /path/to/your_dbt_project/models/

# 4. Run them
cd /path/to/your_dbt_project
dbt deps        # only needed if dbt_output/packages.yml was generated
dbt build
```

> **Windows (PowerShell):** use `python` the same way. Replace the `\` line continuations in the
> multi-line examples below with a backtick (`` ` ``), or put the command on one line. Use
> `Copy-Item -Recurse dbt_output\models\* C:\path\to\project\models\` instead of `cp -r`.

---

## 3. End-to-end workflow

```
 ┌─────────────────────┐   python generate_dbt_models.py   ┌──────────────────────────┐
 │ STM mapping (.xlsx) │ ────────────────────────────────▶ │ dbt_output/              │
 │  one row per column │       --database <platform>       │  models/ (.sql + .yml)   │
 └─────────────────────┘                                   │  packages.yml            │
                                                           │  dbt_project_snippet.yml │
                                                           │  .sqlfluff               │
                                                           │  generation_report.md    │
                                                           └────────────┬─────────────┘
                                                      copy into your dbt project
                                                                        ▼
                                                     dbt deps  →  dbt build
```

1. **Fill in the mapping.** Use the `STM_MAPPING` tab of `STM_dbt_Mapping_Template.xlsx`. Its `README`
   tab explains every column, and the grey rows are examples you can delete.
2. **Generate.** Run the generator with the options you need ([section 4](#4-command-line-reference)).
3. **Check the report.** `generation_report.md` lists every model, its status (OK, SKIPPED or
   WRITTEN with errors), upstream sources, staging decision, folder and datatype translations.
   Skipped models have errors that cite the spreadsheet row numbers to fix.
4. **Merge into your project.**
   - Copy `dbt_output/models/*` into your project's `models/`.
   - Merge `dbt_project_snippet.yml` into `dbt_project.yml`, under `models:`.
   - If `packages.yml` was generated, add its entry to your project's `packages.yml`.
   - Optionally copy `.sqlfluff` to the project root.
5. **Run dbt.** `dbt deps && dbt build`.
6. **Change things in the mapping, not the SQL.** Regenerate after every mapping change. `--check` in CI catches hand edits ([section 11](#11-cicd-integration)).

---

## 4. Command-line reference

```
python generate_dbt_models.py MAPPING [options]
```

### 4.1 Input and output

| Option | Default | Description |
|---|---|---|
| `MAPPING` *(required)* | - | Path to the mapping file: `.xlsx` or `.csv`. |
| `-o`, `--output DIR` | `dbt_output` | Output directory. Created if missing. Existing files are overwritten, but files no longer produced are not deleted (see `--clean`). |
| `--sheet NAME` | `STM_MAPPING`, else first sheet | Worksheet to read from an `.xlsx`. Ignored for `.csv`. |

```bash
python generate_dbt_models.py mapping.xlsx                       # -> ./dbt_output
python generate_dbt_models.py mapping.xlsx -o build/dbt          # custom output folder
python generate_dbt_models.py mapping.xlsx --sheet Finance_STM   # read a different tab
python generate_dbt_models.py mapping.csv  -o dbt_output         # CSV input (same headers)
```

### 4.2 Target platform

| Option | Default | Description |
|---|---|---|
| `--database {snowflake,postgres,bigquery,databricks,redshift}` | `snowflake` | Platform to generate for. Translates `TARGET_DATATYPE` in casts and YAML `data_type`, picks the incremental strategy, types the surrogate-key and audit columns, and sets the sqlfluff dialect. Alias: `--target-db`. |
| `--incremental-strategy NAME` | per platform | Overrides the default strategy for incremental models. It's validated against what the platform supports (table below); an unsupported value stops with an error. |
| `--dialect NAME` | from `--database` | Overrides only the sqlfluff dialect written to `.sqlfluff`. |

| Platform | Default strategy | Allowed `--incremental-strategy` values |
|---|---|---|
| snowflake | `merge` | merge, append, delete+insert, insert_overwrite, microbatch |
| bigquery | `merge` | merge, insert_overwrite, microbatch |
| databricks | `merge` (+ `file_format='delta'`) | merge, append, insert_overwrite, replace_where, microbatch |
| postgres | `delete+insert` | delete+insert, append, merge, microbatch |
| redshift | `delete+insert` | delete+insert, append, merge, microbatch |

```bash
python generate_dbt_models.py mapping.xlsx -o out_sf  --database snowflake
python generate_dbt_models.py mapping.xlsx -o out_pg  --database postgres
python generate_dbt_models.py mapping.xlsx -o out_bq  --database bigquery
python generate_dbt_models.py mapping.xlsx -o out_dbx --database databricks
python generate_dbt_models.py mapping.xlsx -o out_rs  --database redshift

# Postgres 15+ with a recent dbt-postgres can use merge instead of delete+insert
python generate_dbt_models.py mapping.xlsx -o out_pg --database postgres --incremental-strategy merge
```

### 4.3 Sources

| Option | Default | Description |
|---|---|---|
| `--source-name NAME` | `raw` | dbt source name used when a row's `SOURCE_NAME` is blank, as in `{{ source('raw', 'orders') }}`. |
| `--source-schema SCHEMA` | *(none)* | Physical schema written to the default source's YAML (`schema:`). Use it when the schema name differs from the source name. |
| `--freshness-field COLUMN` | *(off)* | Adds source freshness to every source, using this column as `loaded_at_field`: warn after 24h, error after 48h. Check it with `dbt source freshness`. |

```bash
python generate_dbt_models.py mapping.xlsx --source-name erp --source-schema RAW_ERP
python generate_dbt_models.py mapping.xlsx --freshness-field _loaded_at
```

### 4.4 Model structure and best practices

| Option | Default | Description |
|---|---|---|
| `--staging` / `--no-staging` | `--no-staging` | Default for models whose `USE_STAGING` cell is blank. With staging on, each source table gets a `stg_<source>__<table>` view and the model reads it with `ref()`. Without it, models read `source()` directly. `USE_STAGING` = Y/N in the mapping always wins. |
| `--staging-dir DIR` | `staging` | Root for staging models when `STAGING_MODEL_FOLDER` is blank. Files go to `models/<DIR>/<source>/`. |
| `--marts-dir DIR` | `marts` | Folder for target models when `TARGET_MODEL_FOLDER` is blank (`models/<DIR>/`). |
| `--yaml-layout {model,folder}` | `model` | `model`: one `<model>.yml` next to each `<model>.sql`. `folder`: one combined `_<folder>__models.yml` per folder (the older layout). |
| `--enforce-contracts` | off | Adds `contract: {enforced: true}` and a translated `data_type` for every column. A model with any column missing `TARGET_DATATYPE` is warned about and left without a contract. Ephemeral models never get contracts. |
| `--surrogate-key` | off | For models with a composite `UNIQUE_KEY`, adds a `<model>_sk` column built with `dbt_utils.generate_surrogate_key`, uses it as `unique_key`, and tests it `unique` + `not_null`. |
| `--audit-columns` | off | Adds `dbt_loaded_at` (load timestamp) and `dbt_invocation_id` (the dbt run id) to every target model. |
| `--no-composite-test` | test on | Skips the `dbt_utils.unique_combination_of_columns` test that composite keys get by default. |
| `--legacy-test-syntax` | off | Writes tests in the dbt < 1.10 form: `tests:` with arguments at the top level. The default is `data_tests:` with `arguments:` (dbt 1.10+). |

```bash
python generate_dbt_models.py mapping.xlsx --staging                         # stg_ layer for every model
python generate_dbt_models.py mapping.xlsx --staging-dir stg --marts-dir core
python generate_dbt_models.py mapping.xlsx --yaml-layout folder              # one YAML per folder
python generate_dbt_models.py mapping.xlsx --enforce-contracts --surrogate-key --audit-columns
python generate_dbt_models.py mapping.xlsx --legacy-test-syntax              # dbt 1.8 / 1.9 projects
```

### 4.5 Project scaffolding

| Option | Default | Description |
|---|---|---|
| `--project-name NAME` | `my_dbt_project` | Your dbt project's `name:`. It's used as the key in `dbt_project_snippet.yml`, so set it to match your project. |

```bash
python generate_dbt_models.py mapping.xlsx --project-name analytics_warehouse
```

### 4.6 Run modes

| Option | Description |
|---|---|
| `--force` | Also writes models that fail validation. A missing join becomes `on 1 = 1 -- TODO`. Useful for drafting, not for production. |
| `--clean` | Deletes `OUTPUT/models/` before generating, so models removed from the mapping (or old-layout YAML files) don't linger. |
| `--check` | CI mode. Generates in a temporary folder and compares it with `OUTPUT`. Nothing is written. Exits 1 and lists each file that differs. |
| `-v`, `--verbose` | Debug logging. |
| `--version` / `-h`, `--help` | Show the version or the help text. |

```bash
python generate_dbt_models.py mapping.xlsx -o dbt_output --clean            # full regeneration
python generate_dbt_models.py mapping.xlsx -o dbt_output --force            # draft incomplete models
python generate_dbt_models.py mapping.xlsx -o dbt_output --check            # CI: fail on drift
```

> `--check` must use **the same options** as the run that produced `OUTPUT`. Otherwise every
> difference in options shows up as drift.

---

## 5. Mapping template reference

One row per target column. All rows with the same `MODEL_NAME` form one model. **Model-level**
columns only need filling on the model's first row; other rows can be blank or repeat the same value.
Headers are case-insensitive, and common variants are accepted (e.g. `SORUCE_TABLE`, `MODEL_FOLDER`).

| Column | Scope | Required | What to enter | Example |
|---|---|---|---|---|
| SOURCE_TABLE | row | **yes** | Source table. Its lower-case name becomes the SQL alias. | `ORDERS` |
| SOURCE_COLUMN | row | **yes** | Source column. | `ORDER_DATE` |
| TARGET_COLUMN | row | **yes** | Output column name, or an expression (see 5.1). | `order_date` |
| MODEL_NAME | row | **yes** | Model file name; `.sql` is optional. | `fct_orders.sql` |
| TARGET_TABLE | row | recommended | Target table. Becomes the dbt `alias` if it differs from the model name. | `FCT_ORDERS` |
| TARGET_DATATYPE | row | optional | Type to cast to; blank means no cast. Translated per `--database`. | `decimal(18,2)` |
| MATERIALIZATION | model | **yes** | `table`, `view`, `incremental` or `ephemeral`. Default `view`. | `incremental` |
| UNIQUE_KEY | model | recommended | Key column(s), comma-separated; must be target column names. | `order_id,line_number` |
| JOIN_CONDITION | model | when 2+ sources | Join predicates using lower-case table aliases; separate several with `;`. | `order_items.order_id = orders.order_id` |
| JOIN_TYPE | model | optional | `inner` (default), `left`, `right`, `full`. | `left` |
| INCREMENTAL_FILTER | model | optional | `where` predicate used on incremental runs. | `orders.updated_at > (select max(updated_at) from {{ this }})` |
| SOURCE_NAME | row | optional | dbt source / system name. Blank means `--source-name`. | `sales` |
| SCHEMA | model | optional | Custom schema, written as `config(schema=...)`. | `analytics` |
| TAGS | model | optional | Comma-separated dbt tags. | `sales,daily` |
| MODEL_DESCRIPTION | model | recommended | Model description for the YAML docs. | `Order-line sales fact.` |
| COLUMN_DESCRIPTION | row | recommended | Column description for the YAML docs. | `Net revenue after discount` |
| COLUMN_TESTS | row | optional | Column tests (see 5.2). | `not_null, relationships(dim_product.product_id)` |
| USE_STAGING | model | optional | `Y` = read sources via `stg_` views; `N` = direct. Blank means the CLI default. | `Y` |
| STAGING_MODEL_FOLDER | row | optional | Folder for this source table's staging model. Blank means `<staging-dir>/<source>`. | `staging/erp` |
| TARGET_MODEL_FOLDER | model | optional | Folder for the model's `.sql` and `.yml`. Blank means `--marts-dir`. | `marts/sales/facts` |

### 5.1 TARGET_COLUMN forms

| Form | Example | Output column |
|---|---|---|
| Same name | `ORDER_DATE` | `order_date` |
| Rename | `STATE_CODE` (source column `STATE`) | `state_code` |
| Expression | `upper(channel)` | named after SOURCE_COLUMN: `channel` |
| Expression + alias | `quantity * order_items.unit_price as gross_amount` | `gross_amount` |
| dbt macro + alias | `{{ dbt.datediff('orders.order_date', 'orders.ship_date', 'day') }} as days_to_ship` | `days_to_ship` |

- **What counts as an alias.** Only a trailing, top-level `as <name>` is treated as an alias. `cast(x as int)` stays an expression without one.
- **Referencing other columns.** Refer to columns other than SOURCE_COLUMN as `table.column`. They are then added to the staging model and stay unambiguous in joins.
- **Join qualification.** In multi-source models, the bare SOURCE_COLUMN inside an expression is qualified automatically.

### 5.2 COLUMN_TESTS syntax

Comma-separated. Key columns automatically get `not_null`, and `unique` when the key is a single column.

| Entry | Generates |
|---|---|
| `not_null`, `unique` | built-in tests |
| `accepted_values(A\|B\|C)` | `accepted_values` with values A, B, C |
| `relationships(dim_customer.customer_id)` | FK test to `ref('dim_customer')`, field `customer_id` |
| `dbt_utils.not_empty_string` (any `package.test`) | that generic test, without arguments |

---

## 6. What gets generated

```
dbt_output/
├── models/
│   ├── staging/<source>/                         (folder: STAGING_MODEL_FOLDER or <staging-dir>/<source>)
│   │   ├── _<source>__sources.yml                source + table + column definitions (+ freshness)
│   │   ├── stg_<source>__<table>.sql             only for tables read by staged models
│   │   └── _<source>__models.yml                 staging model docs
│   └── marts/.../                                (folder: TARGET_MODEL_FOLDER or --marts-dir)
│       ├── <model>.sql                           one per MODEL_NAME
│       └── <model>.yml                           one per model (or _<folder>__models.yml with --yaml-layout folder)
├── packages.yml                                  dbt_utils - only when needed
├── dbt_project_snippet.yml                       folder-level +materialized defaults to merge into dbt_project.yml
├── .sqlfluff                                     lint config for the generated style and dialect
└── generation_report.md                          models, status, lineage, folders, type translation, warnings
```

**Shape of a target model**
```sql
{{ config(materialized='incremental', unique_key=['order_id', 'line_number'],
          incremental_strategy='merge', on_schema_change='append_new_columns', tags=['sales']) }}

with

order_items as ( select * from {{ ref('stg_sales__order_items') }} ),   -- or source() without staging
orders      as ( select * from {{ ref('stg_sales__orders') }} ),

final as (
    select
        cast(order_items.order_id as int) as order_id,
        ...
    from order_items
    inner join orders on order_items.order_id = orders.order_id
    {% if is_incremental() %}
    where orders.updated_at > (select max(updated_at) from {{ this }})
    {% endif %}
)

select * from final
```

**Shape of a model YAML** (`<model>.yml`)
```yaml
version: 2
models:
  - name: fct_order_lines
    description: Sales fact at order-line grain.
    config:
      meta: {generated_by: dbt_accelerator v2.4.0, mapping_file: ..., target_database: snowflake}
      contract: {enforced: true}           # with --enforce-contracts
    columns:
      - name: product_id
        data_type: int
        description: FK to dim_product
        data_tests:
          - not_null
          - relationships:
              arguments: {to: "ref('dim_product')", field: product_id}
```

---

## 7. Target databases

Write `TARGET_DATATYPE` once in any common spelling. The generator translates it for `--database`:

| Mapping type | snowflake | postgres | redshift | bigquery | databricks |
|---|---|---|---|---|---|
| int / integer | int | integer | integer | int64 | int |
| bigint / number(38,0) | bigint | bigint | bigint | int64 | bigint |
| decimal(18,2) / numeric / number(18,2) | number(18,2) | numeric(18,2) | decimal(18,2) | numeric(18,2) ¹ | decimal(18,2) |
| decimal(38,12) | number(38,12) | numeric(38,12) | decimal(38,12) | bignumeric | decimal(38,12) |
| varchar(n) | varchar(n) | varchar(n) | varchar(n) ² | string | string |
| text / string | varchar | text | varchar(max) | string | string |
| char(n) | char(n) | char(n) | char(n) | string | string |
| float / double | float | double precision | double precision | float64 | double |
| boolean | boolean | boolean | boolean | bool | boolean |
| date | date | date | date | date | date |
| time | time | time | time | time | string ³ |
| timestamp | timestamp_ntz | timestamp | timestamp | timestamp | timestamp |
| timestamp_ntz | timestamp_ntz | timestamp | timestamp | datetime | timestamp_ntz |
| timestamp_tz / timestamptz | timestamp_tz | timestamptz | timestamptz | timestamp | timestamp |
| variant / json | variant | jsonb | super | json | string |

1. BigQuery casts to `numeric(18,2)`, but YAML contracts use plain `numeric`, because BigQuery tables don't keep type parameters.
2. Redshift varchar lengths above 65535 are capped, with a warning.
3. Databricks has no TIME type; time maps to `string`, with a warning.

- **Unrecognised types** (e.g. `geography`) are passed through unchanged, with a warning.
- **Surrogate-key and audit columns** are typed per platform as well.
- **`.sqlfluff`** uses the matching dialect.
- **The report** has a *Datatype translation* table showing exactly what each mapping type became.

---

## 8. Staging layer and folders

### Staging: who decides
1. `USE_STAGING` on the model's first row: `Y` means staging, `N` means read the source directly.
2. If the cell is blank, the `--staging` / `--no-staging` flag decides.
3. If neither is set, there's no staging.

A staged model gets one `stg_<source>__<table>` view per source table, selecting only the columns
that staged models use, including columns referenced in joins, filters and `table.column` expressions.
Unstaged models read `{{ source() }}` directly. One run can mix both.

### Folders (relative to `models/`; nesting allowed; a leading `models/` is ignored)

| Column | Applies to | Blank means |
|---|---|---|
| TARGET_MODEL_FOLDER | the model's `.sql` and `.yml` | `--marts-dir` (default `marts`) |
| STAGING_MODEL_FOLDER | that row's source table (stg model; sources.yml when all tables of the source agree) | `<--staging-dir>/<SOURCE_NAME>` (default `staging/<source>`) |

- **Shared folders.** Staging and target models may share a folder. The staging models then get an explicit `config(materialized='view')`.
- **Rejected paths.** Absolute paths, `..` and invalid characters are errors.
- **Conflicting staging folders.** If one source table is given two different STAGING_MODEL_FOLDER values, the first one is used, with a warning.

---

## 9. Command recipes

```bash
# Minimal: Snowflake, models read sources directly, one YAML per model
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output

# Production profile for Snowflake: staging, contracts, surrogate keys, audit columns, freshness
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
    --database snowflake --staging \
    --enforce-contracts --surrogate-key --audit-columns \
    --freshness-field _loaded_at --project-name analytics_warehouse --clean

# BigQuery, with sources from a differently named dataset
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
    --database bigquery --source-name raw --source-schema raw_landing

# Databricks with custom folder roots
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
    --database databricks --staging-dir bronze --marts-dir gold

# Redshift, old dbt (1.8/1.9) project, combined YAML per folder
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
    --database redshift --legacy-test-syntax --yaml-layout folder

# Postgres 15+ using merge
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output \
    --database postgres --incremental-strategy merge

# Generate the same mapping for every platform (bash)
for db in snowflake postgres bigquery databricks redshift; do
  python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o out/$db --database $db --clean
done

# Draft mode: write everything, even models with validation errors
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o draft --force

# CI: verify the committed models still match the mapping (same options as the generating run)
python generate_dbt_models.py STM_dbt_Mapping_Template.xlsx -o dbt_output --database snowflake --check
```

After generating:
```bash
cp -r dbt_output/models/* <your_project>/models/
# merge dbt_output/dbt_project_snippet.yml into <your_project>/dbt_project.yml (under models:)
# add dbt_output/packages.yml entries to <your_project>/packages.yml (if generated)
cd <your_project>
dbt deps
dbt build                                   # all models + tests
dbt build -s tag:sales                      # only models tagged 'sales' (TAGS column)
dbt build -s fct_order_lines+               # one model and everything downstream
dbt build -s fct_order_lines --full-refresh # rebuild an incremental model from scratch
dbt source freshness                        # if --freshness-field was used
```

---

## 10. Validation, errors and exit codes

Every model is validated before it's written. **Errors** skip the model (unless `--force` is used).
**Warnings** are written but listed. Both go to the console and to `generation_report.md`, with
spreadsheet row numbers.

| Severity | Condition |
|---|---|
| ERROR | Mapping file, sheet or a required header missing (the run stops with exit code 2) |
| ERROR | Model has 2+ source tables but no `JOIN_CONDITION`, or a source isn't referenced by any join |
| ERROR | Duplicate target column in one model; `UNIQUE_KEY` column isn't a target column |
| ERROR | Conflicting model-level values (MATERIALIZATION, JOIN_TYPE, SCHEMA, folder, USE_STAGING…) or unsupported ones |
| ERROR | `USE_STAGING` not Y/N; folder absolute, contains `..` or invalid characters |
| ERROR | Same table name from two different `SOURCE_NAME`s in one model (alias clash) |
| WARN | Rows in one model point to different `TARGET_TABLE`s (usually a wrong MODEL_NAME) |
| WARN | Incremental model without `UNIQUE_KEY` or `INCREMENTAL_FILTER` (a TODO filter is generated) |
| WARN | Contract skipped because a column has no `TARGET_DATATYPE` |
| WARN | Datatype not recognised, or capped/changed for the target database |
| WARN | Non-snake_case model or column names; conflicting STAGING_MODEL_FOLDER for a table |

| Exit code | Meaning |
|---|---|
| `0` | Every model was written; for `--check`, no drift |
| `1` | At least one model was skipped because of errors; for `--check`, drift found |
| `2` | Invalid input or usage: mapping file not found, unknown `--sheet`, required headers missing, openpyxl not installed for .xlsx, unknown option, unsupported `--database` / `--incremental-strategy`, bad folder flag. Nothing is generated. |

---

## 11. CI/CD integration

Commit the mapping **and** the generated models. In CI, fail the build when they disagree, then run dbt:

```yaml
# .github/workflows/dbt.yml (excerpt)
- name: Install
  run: pip install openpyxl dbt-snowflake
- name: Generated models match the mapping
  run: |
    python tools/generate_dbt_models.py mappings/STM.xlsx -o dbt_output \
      --database snowflake --staging --enforce-contracts --check
- name: Build and test
  run: |
    dbt deps
    dbt build --target ci
```

Typical team flow:
1. **Change the mapping** spreadsheet.
2. **Regenerate** with `--clean` and the agreed options.
3. **Commit** the mapping and the generated models together.
4. **Let CI check**: `--check` catches hand-edited SQL or a forgotten regeneration, and `dbt build` runs the tests and contracts.

---

## 12. Testing the generated models

- **Locally without a warehouse:** `pip install dbt-duckdb`, then run the models against a
  DuckDB file. The `sales_example/test_kit/run_tests.sh` script in the package does this end to end
  with dummy sales data, including a second incremental batch.
- **Offline compile for any platform:** `validation/offline_compile.py <project_dir>` runs a real
  `dbt compile` with that adapter's macros but no connection (it treats it as a first run).
  `validation/check_compiled.py <project_dir> <dialect>` then syntax-checks every compiled model.
  You need the adapter installed and a `profiles.yml` with placeholder credentials.
- **Before production:** run `dbt build` once against a dev target on the real platform.

```bash
python validation/offline_compile.py my_project
python validation/check_compiled.py  my_project bigquery
```

---

## 13. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Mapping is missing required column(s)` | The header row must contain SOURCE_TABLE, SOURCE_COLUMN, TARGET_COLUMN and MODEL_NAME. Check `--sheet`. |
| A model is *SKIPPED* | Read its row in `generation_report.md`. The error names the spreadsheet rows to fix. |
| dbt: *model defined twice* / duplicate patch | The old `_<folder>__models.yml` files are still there next to the new `<model>.yml`. Regenerate with `--clean`, or delete the old files. |
| dbt: `'dbt_utils' is undefined` | Add the generated `packages.yml` entry and run `dbt deps`. |
| Contract error: data type mismatch | Check the *Datatype translation* table in the report. Adjust `TARGET_DATATYPE` or generate with the right `--database`. |
| `--check` reports drift right after generating | `--check` was run with different options than the generating run. Use identical options. |
| Models land in the wrong project key | Set `--project-name` to your `dbt_project.yml` `name:`. |
| Tests fail on dbt 1.8/1.9 with `arguments` | Regenerate with `--legacy-test-syntax`. |
| A staging view is missing a column used in an expression | Reference it as `table.column` in the expression (e.g. `customers.last_name`). |

---

## 14. Limitations

- **Your expressions aren't translated between platforms.** Expressions in TARGET_COLUMN, JOIN_CONDITION and INCREMENTAL_FILTER are used as written. Keep them portable, or use dbt's cross-database macros (`dbt.datediff`, `dbt.concat`, `dbt.safe_cast`, `dbt.dateadd`…).
- **No aggregation.** The mapping has no aggregation or GROUP BY; each model selects at the grain of its joined sources.
- **One join type per model.** JOIN_TYPE applies to every join in a model.
- **Dimension surrogate keys need an explicit join.** To look up a dimension's surrogate key in a fact, add the dimension as another source table with a JOIN_CONDITION.
- **No partitioning or clustering.** Platform-specific physical options (e.g. BigQuery `partition_by`, Snowflake `cluster_by`) aren't generated. Add them in `dbt_project.yml` or extend `render_config`.
