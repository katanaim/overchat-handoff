# Overchat — схема данных GA4 в BigQuery

Полный справочник по таблице, схеме событий и ловушкам. Читать перед написанием любого запроса.

## Таблица

```
`zinc-hour-447409-k5.analytics_469242162.events_*`
```

- Это **GA4-экспорт, только WEB**. Данных мобильных приложений (iOS/Android) в BigQuery НЕТ — они в Adapty/сторах, не здесь.
- Партиции по дате: фильтровать через `_TABLE_SUFFIX BETWEEN 'YYYYMMDD' AND 'YYYYMMDD'` (формат `20260605`).
- GA4-экспорт догоняет с лагом ~2-3 дня. Поэтому всегда брать буфер от сегодняшней даты (см. SKILL.md).

## ⚠️ Двойная схема событий — ГЛАВНЫЙ источник ошибок

Семантика события лежит **в разных местах** в зависимости от типа страницы:

### Лендинг-страницы (`/image/<slug>`, `/video/<slug>`, `/text/<slug>`, `/chat/<slug>`, `/models/<slug>`)
- Семантика — **в самом `event_name`** (snake_case), например:
  - `ai_face_rater_click_upload`, `ai_face_rater_upload_success`, `ai_face_rater_click_generate`
  - `looksmax_image_upload_attempt`, `looksmax_click_google`, `looksmax_click_close_paywall`
  - `image_combiner_upload`, `image_combiner_click_combine`, `image_combiner_prompt_typed`
  - `ai_attractiveness_test_click_upload`, `ai_attractiveness_test_upload_success`
- `eventCategory` / `eventAction` / `eventLabel` тут **NULL**.
- Имена событий у каждого виджета СВОИ. **Никогда не угадывать** — сначала вытащить палитру (см. sql-templates.md → palette query).

### Продуктовые страницы (`/web/...`)
- `event_name = 'overchat'`
- Семантика — в `event_params`: `eventCategory` / `eventAction` / `eventLabel`.
- Примеры: `chat/pop-up/get stars view`, `chat/pop-up/get stars click`, `chat/pop-up/get feature view`, `chat/request/web-application`, `login/registration/email`, `purchase/<payment>/<plan>`.

### Дубли каждого overchat-события (дедуп обязателен)
Каждое продуктовое событие логируется в BigQuery **трижды**:
1. `event_name='overchat'` с заполненными cat/act/label — **это берём**.
2. `event_name='user_id_pushed'` — дубль, **отбрасывать** (`AND event_name != 'user_id_pushed'`).
3. Слитое имя, напр. `chat_pop-up_get feature view` (cat/act/label = NULL) — дубль, **отбрасывать при обработке** (в Python по паттерну, см. ниже).

## ⚠️ Ловушка NULL в фильтрах (роняет половину событий молча!)

В BigQuery `NULL = 'что-то'` → `NULL`, а `NOT NULL` → `NULL`, и строка с `NULL` в `WHERE` **молча выпадает**.

Поэтому фильтр вида:
```sql
AND NOT (eventAction = 'create new chat')   -- ❌ ОПАСНО
```
выкинет ВСЕ строки где `eventAction IS NULL` — а это все `page_view`, `session_start` и ВСЕ лендинг-события. Воронка развалится без ошибки.

**Правило:** не фильтровать по `eventAction`/`eventCategory` в `WHERE` через `NOT(...)` или `!=`, если эти поля бывают NULL. Чистить шум (UUID-чаты, дубли) **в Python при обработке**, а не в SQL. Единственное безопасное SQL-исключение — `event_name != 'user_id_pushed'` (event_name не бывает NULL).

## Дедуп и шум — чистить в Python (не в SQL)

При обработке выгруженного JSON отбрасывать:
- **UUID-чаты**: `eventLabel` соответствует UUID (`^[0-9a-f]{8}-[0-9a-f]{4}-...`) — это `create new chat` с id чата, сотни уникальных = шум. Можно схлопнуть в одну строку «create new chat (uuid)».
- **Слитые дубли**: `event_name` содержит структуру вида `chat_pop-up_*`, `login_registration_*`, `*_timer-armed`, `*_impression*`, `*_tab_switch*`, `__` и т.п. ПРИ `cat/act/label = NULL` — это дубль overchat-события.
- **Дедуп по юзерам**: при сведении одного логического события брать `MAX(users)` (а не сумму), т.к. event пишется в нескольких формах.

## Связь лендинг ↔ продукт (URL разные!)

Имя виджета в `/image/...` НЕ совпадает с именем в `/web/...`. Карта известных:

| Виджет | Лендинг | Продукт (`/web/...`) |
|---|---|---|
| rate-my-face | `/image/rate-my-face` | `/web/ai-rate-my-face` |
| looksmax | `/image/looksmaxing-ai` | `/web/looksmax` |
| image-combiner | `/image/ai-image-combiner` | `/web/ai-image-combiner` **и** `/web/image-generator/combine-images` |
| baby-face | `/image/baby-face-generator` | `/web/image-generator/baby-generator` |
| aspect-ratio | `/image/aspect-ratio-changer` | `/web/aspect-ratio` (или `/web/image-generator/...`) |
| attractiveness-test | `/image/ai-attractiveness-test` | продукт-страница НЕ содержит «attractiveness» — юзер уходит в общий `/web/c/<id>` или `/web/ai-rate-my-face` |

**Как находить продукт-URL когда он неизвестен:** брать широкий токен (общая подстрока названия) в `LIKE '%токен%'` на `page_location`, классифицировать surface по `/image/` vs `/web/`, и смотреть блок `3_other`. Если продуктовые события не находятся под токеном виджета (как у attractiveness) — продукт-страница называется иначе; найти её отдельным запросом «куда уходят» (LEAD по page_location).

**Толстый vs тонкий лендинг:**
- «Толстый» (rate-my-face, looksmax, image-combiner, attractiveness) — загрузка/генерация ПРЯМО на лендосе, свои события.
- «Тонкий» (baby, aspect) — лендинг это просто витрина + кнопка `click_openwebapp`; вся работа и генерация в продукте.

## Покупки и пейволлы

- **Покупка**: `event_name='overchat'`, `eventCategory='purchase'`, `eventAction`=способ оплаты (`apple` = Apple Pay, `universal`/`stripe` = карта), `eventLabel`=план.
  - ⚠️ У rate-my-face 100% покупок идут через `apple` (Apple Pay) — норма для него, не баг.
- **Пейволлы** (`eventCategory='chat'`, `eventAction='pop-up'`):
  - `get stars view` / `get stars click` — пейволл «звёзды» (валюта)
  - `get feature view` / `get feature click` — пейволл «фича»
  - `credits paywall view` — пейволл кредитов (видео-генерация). ⚠️ Замечен возможный зацикленный показ (33 показа одному юзеру) — флагать аномалии «много показов на 1 юзера».
- **Регистрация**: `eventCategory='login'`, `eventAction='registration'` (label = `email`/`google`/`apple`). Вход — `eventAction='authorization'`.
- **Генерация в продукте**: `chat/request/web-application`.
- **Загрузка в продукте**: `event_name='upload-attempt'`, `eventCategory='chat'`, `eventAction='upload_attempt'`.

## Пост-онбординг модалка

После генерации/онбординга часть виджетов показывает кросс-промо модалку (увести в другой виджет). События вида:
- `<widget>-post-onboarding` (event_name, слитое) ИЛИ `eventCategory='<widget>-post-onboarding'`
- действия: `timer-armed` (модалка взведена), `impression` (показана), `cta-<target>` (клик на промо целевого виджета), `view-<target>` (показ промо-карточки)
- Имена варьируются по виджетам — **вытаскивать всё, что содержит `post-onboarding`**, и строить под-воронку (см. SKILL.md → пост-онбординг).

## Кросс-фановые джойны

Связывать лендинг ↔ продукт по `user_pseudo_id` + порядок по `event_timestamp` (микросекунды). Для атрибуции покупки к виджету — last-click среди визитов лендосов до момента покупки.
