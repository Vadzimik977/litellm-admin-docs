# User Activity: активность по пользователям

Раздел отслеживания расходов и использования по пользователям.

## Какие данные отображаются

- Расход ($) по пользователям за период
- Токены (prompt/completion) по пользователям
- Breakdown по моделям, провайдерам, ключам внутри каждого пользователя
- Пагинация (page, page_size)

## Эндпоинты

### GET `/spend/users`

Список пользователей с расходом.

**Таблица:** `LiteLLM_UserTable`

**Фильтрация:**
- Admin видит всех
- INTERNAL_USER видит только себя
- Другие — HTTP 403

### GET `/user/daily/activity`

Детальная активность пользователя с breakdown.

**Таблица:** `LiteLLM_DailyUserSpend`

**Параметры:**
- `start_date`, `end_date` — период (YYYY-MM-DD)
- `model` — фильтр по модели
- `api_key` — фильтр по ключу
- `user_id` — фильтр по пользователю (admin only)
- `page` (default 1), `page_size` (default 50, max 1000)
- `timezone` — смещение в минутах

**Фильтрация по ролям:**
- Admin может смотреть любого пользователя или глобально (без user_id)
- INTERNAL_USER видит только свои данные. Если передан user_id != свой — HTTP 403

**Формат:** `SpendAnalyticsPaginatedResponse`

### GET `/user/daily/activity/aggregated`

Агрегированная активность без пагинации. Использует SQL GROUPING SETS для вычисления всех уровней агрегации за один проход.

## Формат ответа (SpendAnalyticsPaginatedResponse)

Одинаков для всех daily activity эндпоинтов:

```json
{
  "results": [
    {
      "date": "2024-01-22",
      "metrics": {
        "spend": 12.5,
        "prompt_tokens": 50000,
        "completion_tokens": 20000,
        "total_tokens": 70000,
        "cache_read_input_tokens": 5000,
        "cache_creation_input_tokens": 1000,
        "api_requests": 150,
        "successful_requests": 145,
        "failed_requests": 5
      },
      "breakdown": {
        "models": {"gpt-4": {"metrics": {...}, "metadata": {}, "api_key_breakdown": {...}}},
        "model_groups": {"gpt-4-group": {"metrics": {...}}},
        "providers": {"openai": {"metrics": {...}}},
        "mcp_servers": {"server/tool": {"metrics": {...}}},
        "endpoints": {"/chat/completions": {"metrics": {...}}},
        "api_keys": {"sk-xxx": {"metrics": {...}, "metadata": {"key_alias": "...", "team_id": "..."}}},
        "entities": {"entity-id": {"metrics": {...}}}
      }
    }
  ],
  "metadata": {
    "total_spend": 125.0,
    "total_tokens": 700000,
    "total_api_requests": 1500,
    "page": 1,
    "total_pages": 5,
    "has_more": true
  }
}
```

## SQL GROUPING SETS

В `get_daily_activity_aggregated()` используется один SQL-запрос с GROUPING SETS:

```sql
SELECT date, api_key, model, model_group, custom_llm_provider,
       mcp_namespaced_tool_name, endpoint,
       GROUPING(...) AS group_level,
       SUM(spend) AS spend, SUM(prompt_tokens) AS prompt_tokens, ...
FROM "{pg_table}"
GROUP BY GROUPING SETS (
    (date),
    (date, api_key),
    (date, model),
    (date, model, api_key),
    (date, model_group),
    (date, model_group, api_key),
    (date, custom_llm_provider),
    (date, custom_llm_provider, api_key),
    (date, mcp_namespaced_tool_name),
    (date, mcp_namespaced_tool_name, api_key),
    (date, endpoint),
    (date, endpoint, api_key),
    ()
)
```

Python dispatch'ит строки по `group_level` (bitmask) в нужные bucket.

## Breakdown по измерениям

| Измерение | Ключ | Описание |
| --------- | ---- | -------- |
| Модель | `models` | По моделям (gpt-4, claude-haiku) |
| Model Group | `model_groups` | По группам моделей |
| Provider | `providers` | По провайдерам (openai, anthropic) |
| MCP Server | `mcp_servers` | По MCP-серверам |
| Endpoint | `endpoints` | По эндпоинтам (/v1/chat/completions) |
| API Key | `api_keys` | По API-ключам |
| Entity | `entities` | По сущностям (user/team/tag) |

Каждое измерение (кроме `api_keys`) содержит `api_key_breakdown` — подразбивку по ключам.

## Materialized Views

| View | Назначение |
| ---- | ---------- |
| `MonthlyGlobalSpend` | Глобальный spend за 30 дней |
| `MonthlyGlobalSpendPerKey` | Spend по дням + api_key |
| `MonthlyGlobalSpendPerUserPerKey` | Spend по дням + user + api_key |
| `Last30dKeysBySpend` | Top ключей по spend |
| `Last30dModelsBySpend` | Top моделей по spend |
