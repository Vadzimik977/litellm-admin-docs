# MCP Server Activity: активность по MCP-серверам

Раздел отслеживания использования MCP (Model Context Protocol) серверов.

## Как это работает

MCP-серверы отслеживаются через поле `mcp_namespaced_tool_name` во всех таблицах daily spend:

- `LiteLLM_DailyUserSpend.mcp_namespaced_tool_name`
- `LiteLLM_DailyTeamSpend.mcp_namespaced_tool_name`
- `LiteLLM_DailyOrganizationSpend.mcp_namespaced_tool_name`
- `LiteLLM_DailyEndUserSpend.mcp_namespaced_tool_name`
- `LiteLLM_DailyAgentSpend.mcp_namespaced_tool_name`
- `LiteLLM_DailyTagSpend.mcp_namespaced_tool_name`
- `LiteLLM_SpendLogs.mcp_namespaced_tool_name`

## Какие данные отображаются

- Расход ($) по MCP-серверам за период
- Токены по MCP-серверам
- Число запросов по MCP-серверам
- Breakdown по API-ключам внутри каждого MCP-сервера

## Breakdown

В `BreakdownMetrics.mcp_servers` данные разбиваются по MCP-серверам:

```json
{
  "mcp_servers": {
    "my-mcp-server/tool-name": {
      "metrics": {"spend": 1.5, "prompt_tokens": 5000, "completion_tokens": 2000, ...},
      "metadata": {},
      "api_key_breakdown": {
        "sk-xxx": {"metrics": {"spend": 1.0, ...}, "metadata": {"key_alias": "test", "team_id": "..."}}
      }
    }
  }
}
```

## SQL GROUPING SETS

В `get_daily_activity_aggregated()` MCP включены в GROUPING SETS:

```sql
GROUP BY GROUPING SETS (
    (date, mcp_namespaced_tool_name),
    (date, mcp_namespaced_tool_name, api_key),
    ...
)
```

Bitmask-значения:
- `_GROUP_DATE_MCP = 61` (0b0111101)
- `_GROUP_DATE_MCP_API_KEY = 29` (0b0011101)

## Доступные эндпоинты

MCP-данные доступны через все daily activity эндпоинты:

| Эндпоинт | Описание |
| -------- | -------- |
| `/user/daily/activity` | Активность пользователя с breakdown по MCP |
| `/team/daily/activity` | Активность команды с breakdown по MCP |
| `/organization/daily/activity` | Активность организации с breakdown по MCP |
| `/tag/daily/activity` | Активность по тегу с breakdown по MCP |
| `/agent/daily/activity` | Активность агента с breakdown по MCP |

## Пример ответа

```json
{
  "results": [{
    "date": "2024-01-22",
    "metrics": {"spend": 1.5, "prompt_tokens": 5000, ...},
    "breakdown": {
      "mcp_servers": {
        "filesystem/read_file": {
          "metrics": {"spend": 0.8, "prompt_tokens": 2000, ...},
          "metadata": {},
          "api_key_breakdown": {
            "sk-abc": {"metrics": {"spend": 0.5, ...}, "metadata": {"key_alias": "dev", "team_id": "..."}}
          }
        },
        "database/query": {
          "metrics": {"spend": 0.7, "prompt_tokens": 3000, ...},
          "metadata": {},
          "api_key_breakdown": {...}
        }
      }
    }
  }]
}
```
