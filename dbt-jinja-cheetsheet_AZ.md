# Jinja və dbt Cheat‑Sheet — Sadə, Dərin və Praktik

Bu sənəd iki hissədən ibarətdir:

1. **Jinja2 templating** — əsaslardan qabağa doğru, praktik nümunələrlə.
2. **dbt (Data Build Tool)** — Jinja ilə birgə necə işləyir, makrolar, adapter çağırışları, nümunələr.

Məqsəd: **çox funksiyalı**, **asan başa düşülən**, və **doğru** bir cheat‑sheet.

---

# Bölmə A — Jinja2 (əsas + praktika)

> Qısa giriş: Jinja2 — HTML/SQL və digər mətnləri templating üçün istifadə olunan Python-based templating dili. dbt makroları Jinja sintaksisi ilə yazılır.

## 1. Əsas sintaksis

* Dəyişənlər: `{{ var }}`
* Məntiq blokları: `{% ... %}` (for, if, macro və s.)
* Şərhlər: `{# bu şərhdir #}`

```jinja
{{ user_name }}
{% if active %}
  Active
{% endif %}
```

## 2. Dəyişənlər və ifadələr

* Sadə dəyişən: `{{ my_var }}`
* Expression (hesablama): `{{ x + 1 }}`
* Funksiya/filtr: `{{ name | upper }}`

**Praktik məsləhət:** dbt-də `{{ ref('my_model') }}` və `{{ source('a','b') }}` kimi çağırışlar tez-tez görünür — bunlar da Jinja ifadələridir.

## 3. Control structures (for, if)

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

* Whitespace control: `{%-` və `-%}` boşluqları kəsmək üçün istifadə olunur. Məsələn: `{% for x in list -%}`.

## 4. Filters — faydalı transformasiyalar

* `| upper`, `| lower`, `| title`
* `| replace('a','b')`
* `| default('d')` — dəyər yoxdursa default verir
* `| join(', ')` — listi stringə çevirir

Nümunə:

```jinja
{{ ['a','b','c'] | join(' / ') }}  --> "a / b / c"
```

## 5. Tests — yoxlamalar

* `is none`, `is defined`, `is sequence`, `is string`

```jinja
{% if var is none %}no value{% endif %}
```

## 6. Macros — təkrar istifadə olunan template funksiyaları

* Macro tərtibi:

```jinja
{% macro greet(name) %}
Hello {{ name }}
{% endmacro %}

{{ greet('Rafael') }}
```

* Macro içindən `return` yoxdur — son yazılan render olunan mətn macro nəticəsidir.

## 7. Includes və import

* `include` — başqa template faylını daxil edir.
* `import` — macro set-i import edir və alias ilə istifadə edir.

```jinja
{% import 'macros.sql' as m %}
{{ m.greet('R') }}
```

## 8. Error handling və debugging (praktik)

* Jinja səhvlərini tapmaq üçün **aşağıdakılara diqqət et**:

  * `dbt compile` nəticəsində yaranan `target/compiled/...` faylları oxu — orada tam SQL görəcəksən.
  * `undefined` xətası: dəyişən mövcud deyil. `{{ var | default('X') }}` istifadə et.
  * Səhvlər çox vaxt loop və ya filter səhvindən gəlir — dövrə daxil edilən obyektləri logla yoxla (dbt compile + inspect).

## 9. Praktik nümunələr (SQL generator üçün)

### 9.1 Kolonlarla dövr etmək və şərt yaratmaq

```jinja
SELECT * FROM {{ model }} WHERE
{% for col in columns -%}
  {{ col }} IS NULL OR
{% endfor %}
  FALSE
```

Bu pattern user‑un verdiyi macro nümunəsinə oxşardır — `FALSE` son `OR`-u ləğv edir.

### 9.2 Safer variant (qayda ilə parenthesis qoymaq)

```jinja
SELECT * FROM {{ model }} WHERE (
{% for col in columns -%}
  {{ col }} IS NULL{% if not loop.last %} OR{% endif %}
{% endfor -%}
)
```

Bu üsul sonuncu `OR`-u çıxarır və daha oxunaqlıdır.

## 10. Best practices — Jinja

* Hər zaman `dbt compile` ilə nəticəni yoxla.
* Loops içində `loop.index` və `loop.last`-dan istifadə et.
* `| default()` və `is defined` ilə undefined xətalarını azaldın.
* Macro-ları kiçik, testedilə bilən saxlayın.

---

# Bölmə B — dbt (Jinja ilə birgə praktika)

> Qısa giriş: dbt SQL faylları Jinja ilə tərtib olunur. dbt `ref`, `source`, `adapter`, `this` və makrolar vasitəsilə meta-informasiya və helper funksiyalar verir.

## 1. Əsas anlayışlar

* `ref('model_name')` — modelə referans; build zamanı modelin tam adını qaytarır.
* `source('source_name','table_name')` — raw data source referansı.
* `this` — cari modelin tam relation-ı (schema + name).
* `adapter` — DB adapteri vasitəsilə connection və metadata çağırışları (Postgres, MySQL, Snowflake və s.).

## 2. Macro yazmaq (struktur)

```jinja
{% macro my_macro(arg1, arg2='def') %}
-- Jinja + SQL burada
{% endmacro %}

-- istifadə
{{ my_macro(ref('some_model')) }}
```

**Qeyd:** Macro `sql` render edir — nəticəni bir `select` və ya `where` kimi istifadə et.

## 3. Faydalı adapter funksiyaları (tez istifadə olunanlar)

* `adapter.get_columns_in_relation(relation)` — relation-un sütunlarını qaytarır (list of column objects).
* `adapter.get_relation(database, schema, identifier)` və ya `adapter.get_relation(relation)` — relation metadata.
* `run_query(sql)` — (context-dən asılı) compile zaman sorgu çalışdırmaq üçün istifadə olunur; (bu metod fərqli dbt versiyalarında fərqli davranış göstərə bilər) — *işləndiyi mühitə diqqət et.*

> **Praktik:** `get_columns_in_relation` macro-larda sütun siyahısı almaq üçün çox faydalıdır.

## 4. Macro nümunəsinin yaxşılaşdırılmış versiyası

Orijinal:

```jinja
{% macro no_nulls_in_columns(model) %}
SELECT * FROM {{ model }} WHERE
{% for col in adapter.get_columns_in_relation(model) -%}
{{ col.column }} IS NULL OR
{% endfor %}
FALSE
{% endmacro %}
```

**Problemlər / potensial yaxşılaşma:**

* `adapter.get_columns_in_relation(model)` bəzən `model`-un bir `Relation` object olub olmadığını gözləyir. `ref()` ilə çağırmaq daha stabildir.
* `OR`-un sonuna `FALSE` əlavə edilməsi işləyir, amma oxunaqlı deyil.

**Təkmilləşmiş:**

```jinja
{% macro no_nulls_in_columns(relation) %}
{# relation gözlənilir: ref('my_model') və ya this #}
{% set cols = adapter.get_columns_in_relation(relation) %}
SELECT * FROM {{ relation }} WHERE (
{% for c in cols -%}
  {{ c.column }} IS NULL{% if not loop.last %} OR{% endif %}
{% endfor -%}
)
{% endmacro %}
```

**İstifadə:** `{{ no_nulls_in_columns(ref('my_model')) }}`

## 5. profiles.yml nümunəsi (MySQL örnəyi)

`~/.dbt/profiles.yml` içində (və ya CI-də env\_var ilə):

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

**Qeyd:** `env_var()` dbt compile zamanı environment dəyişənlərini oxuyur — CI-də GitHub Secrets-ı buraya map elə.

## 6. DBT test workflow (CI) — qısa xülasə

* GitHub Actions-da `services:` bölməsində MySQL/ Postgres konteyneri işə salınır.
* `secrets` ilə parolları ötürün.
* İş addımları: `pip install dbt-core dbt-mysql`, `dbt deps`, `dbt seed`, `dbt run`, `dbt test`.
* `dbt compile` — generated SQL-ləri yoxlamaq üçün istifadə edin.

## 7. Faydalı patternlər və nümunələr

### 7.1 Column-list generator (sütunları qaytarmaq)

```jinja
{% macro column_list(relation, sep=', ') -%}
{% set cols = adapter.get_columns_in_relation(relation) -%}
{{ cols | map(attribute='column') | join(sep) }}
{%- endmacro %}

-- istifadə: {{ column_list(ref('my_model')) }}
```

### 7.2 Null check test qaydası (count-based)

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

Bu macro `dbt test` içində `severity` və s. ilə istifadə edilə bilər.

## 8. Debugging dbt+Jinja

* `dbt compile` → `target/compiled/...` içində yaranan SQL-i aç.
* `dbt run --models my_model --debug` → daha çox logging.
* Macro daxilində `{{ log('message') }}`: `log()` bəzi helper paketlərdə var; general yol **compile et və compiled SQL-i yoxla**.

## 9. Best practices — dbt makrolar üçün

* Makroları kiçik saxla — bir funksiya bir iş görsün.
* `ref()` istifadə edərək model referanslarını ver — bu dependency graph yaradır.
* `adapter.get_columns_in_relation()`-dan sonra `if not cols` yoxlamasını unutma.
* `profiles.yml`-də parolları env\_var ilə saxla; heç vaxt həssas məlumatı repo‑da saxlamayın.


---

# Qısa glossary (əl ilə yığılmış)

* **Jinja** — templating dili ({{ }}, {% %})
* **macro** — reusable template block
* **ref()** — dbt model reference
* **source()** — dbt source reference
* **adapter.get\_columns\_in\_relation()** — relation-un sütunlarını alır
* **profiles.yml** — dbt connection konfiqurasiyası
