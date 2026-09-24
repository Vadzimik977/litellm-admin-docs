# Team Activity: активность по командам

Раздел отслеживания расходов и использования по командам.

## Какие данные отображаются

- Расход ($) по командам за 30 дней
- График: stack area/bar chart по командам
- Топ-10 команд по расходу
- Breakdown по моделям, провайдерам, ключам внутри каждой команды

## Эндпоинты

### GET `/global/spend/teams`

Расходы по командам за 30 дней.

**Таблица:** `LiteLLM_SpendLogs` LEFT JOIN `LiteLLM_TeamTable`

**SQL:**
```sql
SELECT t.team_alias, DATE(s."startTime") AS spend_date,
       SUM(s.spend) AS total_spend
FROM "LiteLLM_SpendLogs" s
LEFT JOIN "LiteLLM_TeamTable" t ON s.team_id = t.team_id
WHERE s."startTime" >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY t.team_alias, DATE(s."startTime")
ORDER BY spend_date
```

**Ответ:**
```json
{
  "daily_spend": [{"date": "2024-01-22", "team-1": 10.5, "team-2": 5.3}],
  "teams": ["team-1", "team-2"],
  "total_spend_per_team": [{"team_id": "team-1", "total_spend": 150.5}]
}
```

### GET `/team/daily/activity`

Детальная активность команды с breakdown.

**Таблица:** `LiteLLM_DailyTeamSpend`

**Параметры:** `team_ids` (comma-separated), `start_date`, `end_date`, `model`, `api_key`, `page`, `page_size`.

**Фильтрация по ролям:**
- Admin — полный доступ ко всем командам
- Team admin или обладатель permission — полный доступ к своей команде
- Team member — фильтрация по собственным API-ключам

**Формат:** `SpendAnalyticsPaginatedResponse` — аналогичен User Activity.

## Breakdown по командам

В `BreakdownMetrics.entities` данные разбиваются по командам (при team_id_field):

```json
{
  "entities": {
    "team-1": {
      "metrics": {"spend": 10.5, "prompt_tokens": 50000, ...},
      "metadata": {},
      "api_key_breakdown": {
        "sk-abc": {"metrics": {"spend": 5.0, ...}, "metadata": {"key_alias": "team-key", "team_id": "team-1"}}
      }
    }
  }
}
```

## Фильтрация

- Admin видит все команды
- INTERNAL_USER видит только команды где он состоит (`user_info.teams`)
- Team admin видит данные своей команды
