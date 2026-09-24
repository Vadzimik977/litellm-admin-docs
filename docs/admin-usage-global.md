# Global Usage

Раздел «Global Usage» — первый (дефолтный) пункт страницы Usage. Показывает общую аналитику по всем ресурсам прокси.

**Админ:** «View usage across all resources». **Не-админ:** этот раздел называется «Your Usage» и показывает только его собственную статистику.

## Сводные карточки

- **Total Spend** — общий расход (USD)
- **Total Requests** — общее число запросов
- **Total Successful Requests** — успешные запросы
- **Failed Requests** — «Requests that failed to route to a provider» (ошибки 429)
- **Requests / Spend (USD)** — соотношение

## Графики

- **Daily Spend / Spend per day** — расход по дням
- **Requests per day** — запросы по дням
- **Total Requests Over Time** — накопительные запросы
- **Request / Token Consumption** — потребление токенов
- **Success vs Failed Requests** — разбивка успех/ошибка
- **Success vs Failed Requests by Endpoint** — по эндпоинтам
- **Daily metrics split by model** — метрики по моделям
- **Endpoint Usage Trends** — тренды по эндпоинтам

## Кэш

- **Cache Read Tokens** — токены из кэша
- **Cache Creation** — токены создания кэша
- **Cache Write Tokens** — токены записи в кэш

Питается эндпоинтом `/global/activity/cache_hits`.

## Топы

- **Top Models** (Top Litellm Models) — топ моделей
- **Top Virtual Keys by Spend** — топ ключей, по умолчанию «Showing Top 5 by Spend»
- **Spend by Provider** — расходы по провайдерам

## Фильтры

- Filter by team
- Filter by user
- Filter by User Agents
- Show Zero Spend
- выбор дат

## Источник данных

Сырые логи `LiteLLM_SpendLogs` + materialized views:

| View | Назначение |
| ---- | ---------- |
| `MonthlyGlobalSpend` | Глобальный расход за 30 дней |
| `MonthlyGlobalSpendPerKey` | По ключам |
| `Last30dKeysBySpend` | Топ ключей |
| `Last30dModelsBySpend` | Топ моделей |

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| GET | `/global/activity` | Дневные запросы и токены |
| GET | `/global/activity/model` | Активность по моделям |
| GET | `/global/activity/cache_hits` | Кэш hits/misses |
| GET | `/global/spend/models` | Топ моделей по расходу |
| GET | `/global/spend/keys` | Топ ключей по расходу |
| GET | `/global/spend/end_users` | Топ end users |
| GET | `/global/spend/provider` | Расходы по провайдерам |
| GET | `/global/spend/logs` | Глобальные логи |
| GET | `/global/spend/teams` | Расходы по командам |
| GET | `/global/spend/tags` | Расходы по тегам |
| GET | `/global/spend/all_tag_names` | Все теги |