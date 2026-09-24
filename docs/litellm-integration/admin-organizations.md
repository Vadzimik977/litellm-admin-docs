# Страница Organizations: управление организациями

Страница `/ui/organizations` — верхний уровень иерархии: владеет командами и пользователями.

## Структура

Запись в `LiteLLM_OrganizationTable`:

| Поле | Описание |
| ---- | -------- |
| `organization_name` | Название организации |
| `budget_id` | Привязка к бюджету |
| `max_budget` | Лимит расхода |
| `allowed_models` | Список допустимых моделей |
| `model_aliases` | Алиасы моделей |
| `object_permission` | Ограничения доступа |

Участники организации имеют роли: `org_admin`, `internal_user`.

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| POST | `/organization/new` | Создание |
| PATCH | `/organization/update` | Обновление |
| DELETE | `/organization/delete` | Удаление |
| GET | `/organization/list` | Список |
| GET | `/organization/info` | Информация |
| POST | `/organization/member_add` | Добавление участника |
| POST | `/organization/member_update` | Обновление участника |
| DELETE | `/organization/member_delete` | Удаление участника |
| GET | `/organization/daily/activity` | Дневная активность |
