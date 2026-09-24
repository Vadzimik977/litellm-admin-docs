# Organization Usage

Раздел «Organization Usage» — аналитика расходов на уровне организаций. Доступен только администратору (`adminOnly`).

- **Админ:** «View usage across all organizations»
- **Не-админ:** раздел скрыт

## Что показывает

- Расход по организациям за период
- Breakdown по моделям, провайдерам, ключам внутри каждой организации
- Токены и число запросов

## Источник данных

Таблица `LiteLLM_DailyOrganizationSpend` через `get_daily_activity()` — агрегируется по `organization_id + date + api_key + model`.

## Фильтрация

- Админ видит все организации
- Org admin видит данные своих организаций

## Эндпоинт

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| GET | `/organization/daily/activity` | Daily-активность по организациям |