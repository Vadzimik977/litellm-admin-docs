# Страница Usage: расходы и аналитика

Страница `/ui/usage` — центральный инструмент мониторинга расходов в админ-панели LiteLLM. Построена как набор вкладок (подразделов) в боковой панели страницы.

## Подразделы Usage

| Вкладка | Что показывает | Права |
| ------- | -------------- | ----- |
| [Global Usage](admin-usage-global.md) | Общая аналитика по всему прокси: сводки, графики, топы | Админ |
| [Your Usage](admin-usage-your.md) | Аналитика собственной учётки (для не-админа это заголовок вкладки Global) | Все |
| [Organization Usage](admin-usage-organization.md) | Расходы по организациям | Админ |
| [Team Usage](admin-usage-team.md) | Расходы по командам | Админ / участник |
| [Customer Usage](admin-usage-customer.md) | Расходы по клиентским аккаунтам | Админ |
| [Tag Usage](admin-usage-tag.md) | Расходы по тегам | Админ |
| [Agent Usage](admin-usage-agent.md) | Расходы по AI-агентам (A2A) | Админ |
| [User Usage](admin-usage-user.md) | Расходы по отдельным пользователям | Админ |
| [User Agent Activity](admin-usage-user-agent.md) | Логи активности user-agent | Админ |

## Какие эндпоинты питают подразделы

| Подраздел | Эндпоинты |
| --------- | ---------- |
| Global Usage | `/global/activity`, `/global/activity/model`, `/global/activity/cache_hits`, `/global/spend/models`, `/global/spend/keys`, `/global/spend/end_users`, `/global/spend/provider`, `/global/spend/logs`, `/global/spend/teams`, `/global/spend/tags`, `/global/spend/all_tag_names` |
| Your Usage | `/tag/user-agent/per-user-analytics`, `/user/daily/activity`, `/user/daily/activity/aggregated` |
| Organization | `/organization/daily/activity` |
| Team | `/team/daily/activity` |
| Customer | `/customer/daily/activity` |
| Tag | `/tag/daily/activity`, `/tag/dau`, `/tag/wau`, `/tag/mau`, `/tag/distinct`, `/tag/list` |
| Agent | `/agent/daily/activity` |
| User | `/user/daily/activity`, `/user/daily/activity/aggregated`, `/user/list` |
| User Agent | `/tag/summary`, `/tag/user-agent/per-user-analytics` |

Экспорт (на всех вкладках): `/export_data`, а также интеграция с CloudZero (`/cloudzero/export`, `/cloudzero/init`, `/cloudzero/settings`).

## Архитектура данных

Два слоя таблиц:

**Слой 1 — сырые логи** (`LiteLLM_SpendLogs`): каждая строка = один API-запрос. Используется старыми эндпоинтами (`/global/activity`, `/global/spend/*`).

**Слой 2 — агрегация по дням**:
- `LiteLLM_DailyUserSpend` — по user + date + api_key + model
- `LiteLLM_DailyTeamSpend` — по team
- `LiteLLM_DailyOrganizationSpend` — по organization
- `LiteLLM_DailyEndUserSpend` — по end_user
- `LiteLLM_DailyAgentSpend` — по agent
- `LiteLLM_DailyTagSpend` — по tag

Новые вкладки (User/Team/Org/Customer/Agent/Tag) используют агрегированные daily-таблицы через единую функцию `get_daily_activity()`. Вкладка Global использует сырые логи + materialized views (см. [Global Usage](admin-usage-global.md)).

## Метрики (SpendMetrics)

Единый набор метрик возвращается всеми daily-эндпоинтами:

| Метрика | Описание |
| ------- | -------- |
| `spend` | Стоимость в долларах |
| `prompt_tokens` | Input токены |
| `completion_tokens` | Output токены |
| `total_tokens` | Суммарные токены |
| `cache_read_input_tokens` | Токены из кэша |
| `cache_creation_input_tokens` | Токены создания кэша |
| `api_requests` | Общее число запросов |
| `successful_requests` | Успешные |
| `failed_requests` | Неудачные |

## Breakdown по измерениям

Данные разбиваются по 7 измерениям (`BreakdownMetrics`):

| Измерение | Ключ | Описание |
| --------- | ---- | -------- |
| Модель | `models` | По моделям |
| Model Group | `model_groups` | По группам моделей |
| Provider | `providers` | По провайдерам |
| MCP Server | `mcp_servers` | По MCP-серверам |
| Endpoint | `endpoints` | По эндпоинтам |
| API Key | `api_keys` | По API-ключам |
| Entity | `entities` | По сущностям (user/team/tag) |

Каждое измерение (кроме `api_keys`) содержит `api_key_breakdown` — подразбивку по ключам с метаданными (`key_alias`, `team_id`).

## Разграничение ролей

- `PROXY_ADMIN` видит все вкладки и все данные.
- `INTERNAL_USER` / `INTERNAL_USER_VIEW_ONLY` видят в основном свою собственную статистику (на вкладке Global для них заголовок «Your Usage»), а часть вкладок для них `adminOnly`.

## AI Usage Chat

Встроенный AI-ассистент на странице Usage, отвечает на вопросы о расходах.

**Эндпоинт:** `POST /usage/ai/chat` (SSE streaming)

**Tools:**
- `get_usage_data` — глобальные данные (все роли)
- `get_team_usage_data` — по командам (только админ)
- `get_tag_usage_data` — по тегам (только админ)

**Поток:** `status: "Thinking…"` → `tool_call: "running"` → `tool_call: "complete"` → `status: "Analyzing results…"` → `chunk: "…"` → `done`. Non-admin видят только свои данные.