# Key Activity: активность по ключам

Раздел отслеживания расходов и использования API-ключей.

## Какие данные отображаются

- Список ключей с расходом за 30 дней
- Глобальный расход по дням (с фильтрацией по ключу)
- Логи запросов с фильтрацией по ключу
- Топ ключей по расходу

## Эндпоинты

### GET `/spend/keys`

Список всех API-ключей.

**Таблица:** `LiteLLM_VerificationToken` (find_all)

**Фильтрация:**
- Admin видит все ключи
- INTERNAL_USER видит только ключи где `user_id == caller_user_id`

### GET `/global/spend/keys`

Топ-N ключей по расходу за 30 дней.

**Таблица (VIEW):** `Last30dKeysBySpend`

**VIEW SQL:**
```sql
SELECT L."api_key", V."key_alias", V."key_name",
       SUM(L."spend") AS total_spend
FROM "LiteLLM_SpendLogs" L
LEFT JOIN "LiteLLM_VerificationToken" V ON L."api_key" = V."token"
WHERE startTime >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY L."api_key", V."key_alias", V."key_name"
ORDER BY total_spend DESC
```

**Ответ:** `[{api_key, total_spend, key_alias, key_name}]`

### GET `/global/spend/logs`

Глобальный расход по дням за 30 дней.

**Три ветки:**
- Prometheus подключен → данные из Prometheus
- Admin без api_key → `MonthlyGlobalSpend`
- Admin с api_key → `MonthlyGlobalSpendPerKey`
- INTERNAL_USER → `MonthlyGlobalSpendPerUserPerKey`

**Ответ:** `[{date, spend}]`

### GET `/spend/logs/ui` (и `/spend/logs/v2`)

Сырые логи запросов с пагинацией и фильтрами.

**Таблица:** `LiteLLM_SpendLogs`

**Фильтры:** api_key, user_id, request_id, team_id, model, model_group, key_alias, end_user, error_code, min_spend, max_spend, status_filter, sort_by.

**Сортировка:** spend, total_tokens, startTime, endTime, request_duration_ms, model, ttft_ms.

**Фильтрация по ролям:**
- Admin — полный доступ
- TEAM_MEMBER — логи своей команды + свои
- INTERNAL_USER — только свои

**Ответ:** `{data: [...], total, page, page_size, total_pages}`

## Breakdown по ключам

В `BreakdownMetrics.api_keys` данные разбиваются по API-ключам:

```json
{
  "api_keys": {
    "sk-xxx": {
      "metrics": {"spend": 1.5, "prompt_tokens": 5000, ...},
      "metadata": {"key_alias": "test-key", "team_id": "team-1"}
    }
  }
}
```

Каждое другое измерение (models, providers, mcp_servers) также содержит `api_key_breakdown` — подразбивку по ключам внутри измерения.
