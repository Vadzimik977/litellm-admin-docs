# Team Usage

Раздел «Team Usage» — аналитика расходов по командам.

## Что показывает

- Расход по командам за период
- Breakdown по моделям, провайдерам, ключам внутри каждой команды
- Токены и число запросов

## Источник данных

Таблица `LiteLLM_DailyTeamSpend` через `get_daily_activity()` — агрегируется по `team_id + date + api_key + model`.

## Фильтрация

- Админ видит все команды
- Team admin или обладатель permission `/team/daily/activity` — полный доступ к своей команде
- Team member — только данные по своим API-ключам

## Эндпоинт

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| GET | `/team/daily/activity` | Daily-активность по командам |

Формат ответа — `SpendAnalyticsPaginatedResponse` (см. [метрики](admin-usage.md)).