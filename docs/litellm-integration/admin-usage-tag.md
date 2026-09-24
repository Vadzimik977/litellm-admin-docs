# Tag Usage

Раздел «Tag Usage» — аналитика расходов, сгруппированных по тегам. Доступен администратору (`adminOnly`).

## Что показывает

- Расход по тегам за период
- МАУ / WAU / DAU (активные пользователи)
- Colle количества тегов
- Breakdown по моделям, провайдерам, ключам внутри каждого тега

## Источник данных

Таблица `LiteLLM_DailyTagSpend` через `get_daily_activity()` — агрегируется по `tag + date`.

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| GET | `/tag/daily/activity` | Daily-активность по тегам |
| GET | `/tag/dau` | Daily active users |
| GET | `/tag/wau` | Weekly active users |
| GET | `/tag/mau` | Monthly active users |
| GET | `/tag/distinct` | Количество уникальных тегов |
| GET | `/tag/list` | Список тегов |