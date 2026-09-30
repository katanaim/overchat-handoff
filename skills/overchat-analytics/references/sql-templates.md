# SQL-шаблоны Overchat (проверенные, без ловушек)

Все шаблоны уже учитывают: дедуп (`!= 'user_id_pushed'`), отсутствие NULL-ловушки в WHERE, исключение тест-юзеров.

ВЕЗДЕ первым CTE вставлять `excluded_users` из `assets/excluded_users_cte.sql` (вербатим, диапазон дат под период). Ниже в шаблонах он обозначен как `{{EXCLUDED_USERS_CTE}}` — заменять на полный блок.

Даты: `{{LAST_START}}`/`{{LAST_END}}` — прошлая неделя, `{{PREV_START}}`/`{{PREV_END}}` — позапрошлая (как считать — см. SKILL.md, всегда сначала узнать сегодняшнюю дату).

---

## 1. Палитра всех событий виджета (ПЕРВЫЙ шаг любой воронки)

Высасывает ВСЕ события на лендинге и в продукте за оба периода. Из неё строится воронка. `{{TOKEN_FILTER}}` — широкое условие по `page_location` (общая подстрока названия виджета; для двух URL — через OR). `{{SURFACE_CASE}}` — классификация лендинг/продукт.

```sql
{{EXCLUDED_USERS_CTE}}              -- WITH excluded_users AS (...), диапазон {{PREV_START}}..{{LAST_END}}
, base AS (
  SELECT
    user_pseudo_id,
    event_name,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='page_location') AS pl,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='eventCategory') AS category,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='eventAction')   AS action,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='eventLabel')    AS label,
    CASE
      WHEN _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}' THEN 'LAST'
      WHEN _TABLE_SUFFIX BETWEEN '{{PREV_START}}' AND '{{PREV_END}}' THEN 'PREV'
    END AS period
  FROM `zinc-hour-447409-k5.analytics_469242162.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '{{PREV_START}}' AND '{{LAST_END}}'
    AND user_pseudo_id NOT IN (SELECT user_pseudo_id FROM excluded_users)
    AND event_name != 'user_id_pushed'          -- единственное безопасное SQL-исключение (event_name не NULL)
)
SELECT
  CASE
    WHEN pl LIKE '%/image/<LANDING_SLUG>%' THEN '1_landing'
    WHEN pl LIKE '%/web/<PRODUCT_SLUG_1>%'
      OR pl LIKE '%/web/<PRODUCT_SLUG_2>%' THEN '2_product'   -- второй OR только если две продукт-страницы
    ELSE '3_other'
  END AS surface,
  event_name, category, action, label, period,
  COUNT(*) AS events,
  COUNT(DISTINCT user_pseudo_id) AS users
FROM base
WHERE ( pl LIKE '%<TOKEN>%' )                   -- широкий токен; для одного виджета можно несколько OR
  AND period IS NOT NULL
GROUP BY surface, event_name, category, action, label, period
ORDER BY surface, users DESC
LIMIT 1500
```

Замечания:
- Если у виджета продукт-страница не содержит токен (как attractiveness) — `3_other` будет жирным или продукт пустой; тогда искать продукт отдельно (шаблон 5).
- Если виджет «тонкий» — на лендосе будут только `page_view/session_start/click_openwebapp`, генерация в продукте.

---

## 2. Трафик по ВСЕМ виджетам (две недели рядом) — для сравнения трафика

Сравнение трафика по виджетам показывать в **абсолютных числах** (визиты), + Δ%.

```sql
{{EXCLUDED_USERS_CTE}}              -- диапазон {{PREV_START}}..{{LAST_END}}
SELECT
  REGEXP_EXTRACT(
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='page_location'),
    r'overchat\.ai/((?:image|video|text|chat|models)/[^/?#]+)'
  ) AS widget,
  CASE
    WHEN _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}' THEN 'LAST'
    WHEN _TABLE_SUFFIX BETWEEN '{{PREV_START}}' AND '{{PREV_END}}' THEN 'PREV'
  END AS period,
  COUNT(DISTINCT user_pseudo_id) AS visitors,
  COUNT(*) AS pageviews
FROM `zinc-hour-447409-k5.analytics_469242162.events_*`
WHERE _TABLE_SUFFIX BETWEEN '{{PREV_START}}' AND '{{LAST_END}}'
  AND event_name='page_view'
  AND user_pseudo_id NOT IN (SELECT user_pseudo_id FROM excluded_users)
  AND REGEXP_CONTAINS(
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='page_location'),
    r'overchat\.ai/(?:image|video|text|chat|models)/'
  )
GROUP BY widget, period
HAVING widget IS NOT NULL
ORDER BY widget, period DESC
```

---

## 3. Строго-вложенная воронка по топ-N виджетам (для скелета/сравнения)

Каждый шаг — подмножество предыдущего по времени. Полезно когда нужна корректная «сужающаяся» воронка, а не частотная. Топ-N определяется сам по трафику.

Ключевые шаги (вложенные): визит лендоса → перешёл в продукт (`page_view` на `/web/` после визита) → увидел пейволл (после перехода) → купил (после пейволла). Подробный шаблон — в разделе ниже; для полной воронки одного виджета чаще используется частотная палитра (шаблон 1) + обработка в Python (см. SKILL.md → построение воронки).

---

## 4. Длительность на странице / в сессии

```sql
WITH vg AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key='ga_session_id') AS session_id,
    event_timestamp,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key='page_location') AS loc
  FROM `zinc-hour-447409-k5.analytics_469242162.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}'
),
sessions_on_page AS (
  SELECT user_pseudo_id, session_id,
    (MAX(event_timestamp)-MIN(event_timestamp))/1e6 AS dwell_sec
  FROM vg
  WHERE session_id IS NOT NULL
  GROUP BY user_pseudo_id, session_id
  HAVING COUNTIF(loc LIKE '%<PRODUCT_OR_PAGE_PATH>%') > 0
)
SELECT
  COUNT(*) AS sessions,
  ROUND(AVG(dwell_sec),1) AS avg_sec,
  ROUND(APPROX_QUANTILES(dwell_sec,100)[OFFSET(50)],1) AS median_sec,
  COUNTIF(dwell_sec=0) AS bounced_0sec,
  ROUND(COUNTIF(dwell_sec=0)/COUNT(*)*100,1) AS bounce_pct
FROM sessions_on_page
```

⚠️ Это длительность всей сессии, содержащей страницу (не чистое время на одной странице). Для большинства задач достаточно; оговаривать при выводе. Среднее >> медианы = длинный хвост (часто из-за ожидания рендера видео/картинок).

---

## 5. Источник входа + что нажимают + куда уходят (анализ одной страницы)

Три блока в одном запросе для страницы без лендоса.

```sql
{{EXCLUDED_USERS_CTE}}              -- диапазон {{LAST_START}}..{{LAST_END}}
, page_users AS (
  SELECT DISTINCT user_pseudo_id
  FROM `zinc-hour-447409-k5.analytics_469242162.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}'
    AND (SELECT value.string_value FROM UNNEST(event_params) WHERE key='page_location') LIKE '%<PAGE_PATH>%'
    AND user_pseudo_id NOT IN (SELECT user_pseudo_id FROM excluded_users)
),
entry AS (
  SELECT e.user_pseudo_id,
    (SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='page_referrer') AS referrer,
    ROW_NUMBER() OVER (PARTITION BY e.user_pseudo_id ORDER BY e.event_timestamp) AS rn
  FROM `zinc-hour-447409-k5.analytics_469242162.events_*` e
  WHERE _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}'
    AND e.event_name='page_view'
    AND (SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='page_location') LIKE '%<PAGE_PATH>%'
    AND e.user_pseudo_id IN (SELECT user_pseudo_id FROM page_users)
)
SELECT '1_ENTRY' AS block,
  CASE
    WHEN referrer IS NULL OR referrer='' THEN '(direct/none)'
    WHEN referrer LIKE '%google%' THEN 'google'
    WHEN referrer LIKE '%overchat.ai%' THEN 'internal overchat.ai'
    ELSE REGEXP_EXTRACT(referrer, r'https?://([^/]+)')
  END AS k1, NULL AS k2,
  COUNT(*) AS cnt, COUNT(DISTINCT user_pseudo_id) AS users
FROM entry WHERE rn=1 GROUP BY k1
UNION ALL
SELECT '2_EVENTS', event_name,
  (SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='eventAction'),
  COUNT(*), COUNT(DISTINCT user_pseudo_id)
FROM `zinc-hour-447409-k5.analytics_469242162.events_*` e
WHERE _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}'
  AND (SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='page_location') LIKE '%<PAGE_PATH>%'
  AND e.user_pseudo_id IN (SELECT user_pseudo_id FROM page_users)
  AND event_name != 'user_id_pushed'
GROUP BY event_name, 3
UNION ALL
SELECT '3_NEXT_PAGE', REGEXP_EXTRACT(next_loc, r'overchat\.ai(/[^?#]*)'), NULL,
  COUNT(*), COUNT(DISTINCT user_pseudo_id)
FROM (
  SELECT e.user_pseudo_id,
    (SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='page_location') AS cur_loc,
    LEAD((SELECT value.string_value FROM UNNEST(e.event_params) WHERE key='page_location'))
      OVER (PARTITION BY e.user_pseudo_id ORDER BY e.event_timestamp) AS next_loc
  FROM `zinc-hour-447409-k5.analytics_469242162.events_*` e
  WHERE _TABLE_SUFFIX BETWEEN '{{LAST_START}}' AND '{{LAST_END}}'
    AND e.event_name='page_view'
    AND e.user_pseudo_id IN (SELECT user_pseudo_id FROM page_users)
)
WHERE cur_loc LIKE '%<PAGE_PATH>%' AND next_loc IS NOT NULL AND next_loc NOT LIKE '%<PAGE_PATH>%'
GROUP BY 2
ORDER BY block, users DESC
LIMIT 250
```

---

## Общие правила, чтобы запрос не падал

1. **Группировка по алиасам:** в `GROUP BY` ссылаться на исходное поле (`p.lbl`), а не на алиас (`p.plan`) с префиксом — BigQuery не видит алиас по префиксу. (Известная ошибка: `Name plan not found inside p`.)
2. **`ARRAY_AGG(... LIMIT 1)[OFFSET(0)]`** для last-click атрибуции — рабочий паттерн.
3. **Никаких `NOT(action=...)` / `action != ...` в WHERE** при NULL-полях (см. database-schema.md).
4. **`event_name != 'user_id_pushed'`** — безопасно, добавлять всегда.
5. **`LIMIT`** ставить с запасом (800-1500 для палитры), UUID-чаты раздувают строки.
6. **`HAVING widget IS NOT NULL`** в трафик-запросе, чтобы убрать не-виджетные страницы.
