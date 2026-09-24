# Model Activity: активность по моделям

Раздел отслеживания расходов и использования LLM-моделей.

## Какие данные отображаются

- Расход ($) по моделям за период
- Число API-запросов по моделям
- Токены (prompt/completion) по моделям
- Графики: линейные chart по дням для каждой модели
- Топ-10 моделей по расходу

## Источник данных

Два типа запросов:

**Сырые логи** (`LiteLLM_SpendLogs`) — каждая строка = один API-запрос. Используется старыми эндпоинтами.

**Агрегированные daily-таблицы** (`LiteLLM_DailyUserSpend` и др.) — одна строка на уникальную комбинацию `(entity_id, date, api_key, model, provider, mcp_tool, endpoint)`. Используется новыми эндпоинтами.

## Эндпоинты

### GET `/global/activity`

Активность (API-запросы и токены) по дням.

**Таблица:** `LiteLLM_SpendLogs`

**SQL:**
```sql
SELECT date_trunc('day', "startTime") AS date,
       COUNT(*) AS api_requests,
       SUM(total_tokens) AS total_tokens
FROM "LiteLLM_SpendLogs"
WHERE startTime BETWEEN $1 AND $2
GROUP BY date_trunc('day', "startTime")
```

**Фильтрация:** INTERNAL_USER видят только свои запросы (`AND "user" = $3`).

**Ответ:**
```json
{
  "daily_data": [{"date": "Jan 22", "api_requests": 10, "total_tokens": 2000}],
  "sum_api_requests": 20,
  "sum_total_tokens": 2012
}
```

### GET `/global/activity/model`

Активность по моделям — топ-10 моделей по числу запросов.

**Таблица:** `LiteLLM_SpendLogs`

**SQL:**
```sql
SELECT model_group,
       date_trunc('day', "startTime") AS date,
       COUNT(*) AS api_requests,
       SUM(total_tokens) AS total_tokens
FROM "LiteLLM_SpendLogs"
WHERE startTime BETWEEN $1 AND $2
GROUP BY model_group, date_trunc('day', "startTime")
```

**Ответ:** Массив по top-10 моделям:
```json
[{
  "model": "gpt-4",
  "daily_data": [{"date": "Jan 22", "api_requests": 10, "total_tokens": 2000}],
  "sum_api_requests": 20,
  "sum_total_tokens": 2012
}]
```

### GET `/global/spend/models`

Топ-N моделей по расходу за 30 дней.

**Таблица (VIEW):** `Last30dModelsBySpend`

**VIEW SQL:**
```sql
SELECT "model", SUM("spend") AS total_spend
FROM "LiteLLM_SpendLogs"
WHERE startTime >= CURRENT_DATE - INTERVAL '30 days' AND "model" != ''
GROUP BY "model" ORDER BY total_spend DESC
```

**Ответ:** `[{model, total_spend, total_tokens}]`

### GET `/global/spend/report`

Детальный отчёт с drill-down (только premium). Поддерживает `group_by`: `team`, `customer`, `api_key`.

### GET `/global/activity/cache_hits`

Cache hits vs misses по ключам и моделям.

**Таблица:** `LiteLLM_SpendLogs` JOIN `LiteLLM_VerificationToken`

**Метрики:** `cache_hit_true_rows`, `cached_completion_tokens`, `generated_completion_tokens`

## Breakdown по моделям (в daily activity)

В `BreakdownMetrics.models` данные разбиваются по моделям:

```json
{
  "models": {
    "gpt-4": {
      "metrics": {"spend": 1.5, "prompt_tokens": 5000, "completion_tokens": 2000, ...},
      "metadata": {},
      "api_key_breakdown": {
        "sk-xxx": {"metrics": {"spend": 1.0, ...}, "metadata": {"key_alias": "test", "team_id": "..."}}
      }
    }
  }
}
```

Топ-15 моделей отображается на странице Usage.
