# Jinja and dbt Cheat‑Sheet — Simple, Deep, and Practical

This document has two sections:

1. **Jinja2 templating** — from basics to advanced, with practical examples.
2. **dbt (Data Build Tool)** — how it works with Jinja, macros, adapter calls, examples.

Goal: a **multi-functional**, **easy-to-understand**, and **accurate** cheat-sheet.

---

# Section A — Jinja2 (Basics + Practice)

> Quick intro: Jinja2 is a Python-based templating language used to generate HTML, SQL, or other text. dbt macros are written using Jinja syntax.

## 1. Basic Syntax

* Variables: `{{ var }}`
* Logic blocks: `{% ... %}` (for, if, macro, etc.)
* Comments: `{# this is a comment #}`

```jinja
{{ user_name }}
{% if active %}
  Active
{% endif %}
```

## 2. Variables and Expressions

* Simple variable: `{{ my_var }}`
* Expression (calculation): `{{ x + 1 }}`
* Function/filter: `{{ name | upper }}`

**Practical tip:** In dbt, `{{ ref('my_model') }}` and `{{ source('a','b') }}` are common examples — these are also Jinja expressions.

## 3. Control Structures (for, if)

* `for` loop:

```jinja
{% for col in columns %}
  {{ col }}
{% endfor %}
```

* `if`/`elif`/`else`:

```jinja
{% if x > 0 %}
  positive
{% elif x == 0 %}
  zero
{% else %}
  negative
{% endif %}
```

* Whitespace control: `{%-` and `-%}` remove unnecessary spaces. Example: `{% for x in list -%}`.

## 4. Filters — Useful Transformations

* `| upper`, `| lower`, `| title`
* `| replace('a','b')`
* `| default('d')` — returns a default value if variable is undefined
* `| join(', ')` — joins a list into a string

Example:

```jinja
{{ ['a','b','c'] | join(' / ') }}  --> "a / b / c"
```

## 5. Tests — Checks

* `is none`, `is defined`, `is sequence`, `is string`

```jinja
{% if var is none %}no value{% endif %}
```

## 6. Macros — Reusable Template Functions

* Macro definition:

```jinja
{% macro greet(name) %}
Hello {{ name }}
{% endmacro %}

{{ greet('Rafael') }}
```

* Macros do not use `return`; the last rendered text is the result.

## 7. Includes and Imports

* `include` — include another template file.
* `import` — import a macro file and use with an alias.

```jinja
{% import 'macros.sql' as m %}
{{ m.greet('R') }}
```

## 8. Error Handling and Debugging (Practical)

* Use `dbt compile` to inspect rendered SQL in `target/compiled/...`.
* `undefined` error → use `{{ var | default('X') }}`.
* Loops and filters are common sources of errors — inspect compiled SQL.

## 9. Practical Examples (SQL Generator)

### 9.1 Loop over columns to create a WHERE clause

```jinja
SELECT * FROM {{ model }} WHERE
{% for col in columns -%}
  {{ col }} IS NULL OR
{% endfor %}
  FALSE
```

### 9.2 Cleaner variant using loop.last

```jinja
SELECT * FROM {{ model }} WHERE (
{% for col in columns -%}
  {{ col }} IS NULL{% if not loop.last %} OR{% endif %}
{% endfor -%}
)
```

## 10. Best Practices — Jinja

* Always check with `dbt compile`.
* Use `loop.index` and `loop.last` in loops.
* Use `| default()` and `is defined` to avoid undefined errors.
* Keep macros small and testable.

---

# Section B — dbt (with Jinja Practice)

> dbt SQL files are templated with Jinja. dbt provides `ref`, `source`, `adapter`, `this`, and macros.

## 1. Core Concepts

* `ref('model_name')` — reference a dbt model; returns the full table name at build time.
* `source('source_name','table_name')` — reference a raw source table.
* `this` — the full relation of the current model.
* `adapter` — provides connection and metadata methods (Postgres, MySQL, Snowflake, etc.).

## 2. Macro Structure

```jinja
{% macro my_macro(arg1, arg2='def') %}
-- SQL + Jinja here
{% endmacro %}

-- usage
{{ my_macro(ref('some_model')) }}
```

## 3. Useful Adapter Functions

* `adapter.get_columns_in_relation(relation)` — returns a list of column objects.
* `adapter.get_relation(database, schema, identifier)` — gets relation metadata.
* `run_query(sql)` — executes a query (context dependent).

> Practical: `get_columns_in_relation` is very useful for macros iterating over columns.

## 4. Improved null-check macro

```jinja
{% macro no_nulls_in_columns(relation) %}
{% set cols = adapter.get_columns_in_relation(relation) %}
SELECT * FROM {{ relation }} WHERE (
{% for c in cols -%}
  {{ c.column }} IS NULL{% if not loop.last %} OR{% endif %}
{% endfor -%}
)
{% endmacro %}

-- usage: {{ no_nulls_in_columns(ref('my_model')) }}
```

## 5. profiles.yml example (MySQL)

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: mysql
      host: {{ env_var('MYSQL_HOST') }}
      port: {{ env_var('MYSQL_PORT', '3306') }}
      user: {{ env_var('MYSQL_USER') }}
      password: {{ env_var('MYSQL_PASSWORD') }}
      schema: {{ env_var('MYSQL_SCHEMA') }}
      database: {{ env_var('MYSQL_DATABASE') }}
```

## 6. dbt Test Workflow (CI) Summary

* Use GitHub Actions `services:` for MySQL/Postgres.
* Pass passwords via `secrets`.
* Steps: `pip install dbt-core dbt-mysql`, `dbt deps`, `dbt seed`, `dbt run`, `dbt test`.
* `dbt compile` to inspect generated SQL.

## 7. Useful Patterns and Examples

### 7.1 Column-list generator

```jinja
{% macro column_list(relation, sep=', ') -%}
{% set cols = adapter.get_columns_in_relation(relation) -%}
{{ cols | map(attribute='column') | join(sep) }}
{%- endmacro %}

-- usage: {{ column_list(ref('my_model')) }}
```

### 7.2 Null check count-based test

```jinja
{% macro assert_no_nulls(relation) %}
{% set cols = adapter.get_columns_in_relation(relation) %}
select count(*) as null_count from {{ relation }} where (
  {% for c in cols -%}
    {{ c.column }} is null{% if not loop.last %} or{% endif %}
  {% endfor -%}
)
{% endmacro %}
```

## 8. Debugging dbt + Jinja

* `dbt compile` → check generated SQL in `target/compiled/...`
* `dbt run --models my_model --debug` → verbose logging
* Macro debugging: inspect compiled SQL rather than runtime prints.

## 9. Best Practices — dbt Macros

* Keep macros small; one function per macro.
* Use `ref()` to define model dependencies.
* After `adapter.get_columns_in_relation()`, check if `cols` is empty.
* Use environment variables for passwords; never store secrets in repo.

---

# Quick Glossary

* **Jinja** — templating language (`{{ }}`, `{% %}`)
* **macro** — reusable template block
* **ref()** — dbt model reference
* **source()** — dbt source reference
* **adapter.get\_columns\_in\_relation()** — get table columns
* **profiles.yml** — dbt connection configuration
