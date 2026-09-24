# Страница Teams: управление командами

Страница `/ui/teams` — группировка пользователей, ключей и бюджетов.

## Что показывает

- Список команд с участниками, расходом, моделями
- Управление участниками (добавление/удаление)
- Настройки моделей и бюджетов для команды

## Структура команды

Запись в `LiteLLM_TeamTable`:

| Поле | Описание |
| ---- | -------- |
| `team_name` | Название команды |
| `members_with_roles` | JSON-массив участников с ролями (admin/user) |
| `budget_id` | Привязка к бюджету |
| `max_budget` | Лимит расхода команды |
| `models` | Список допустимых моделей |
| `model_aliases` | Алиасы моделей |
| `object_permission` | Ограничения доступа к объектам |
| `model_rpm_limit` / `model_tpm_limit` | Лимиты RPM/TPM по моделям |
| `max_parallel_requests` | Лимит параллельных запросов |
| `blocked` | Блокировка команды |

## Проверки прав

- `_is_user_team_admin` — является ли пользователь админом команды
- `_is_user_org_admin_for_team` — является ли админом организации команды
- После записи: `_refresh_cached_team` обновляет in-memory кэш

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| POST | `/team/new` | Создание |
| POST | `/team/update` | Обновление |
| POST | `/team/delete` | Удаление |
| POST | `/team/member_add` | Добавление участника |
| POST | `/team/member_delete` | Удаление участника |
| POST | `/team/member_update` | Обновление участника |
| POST | `/team/bulk_member_add` | Массовое добавление |
| POST | `/team/block` | Блокировка |
| POST | `/team/unblock` | Разблокировка |
| GET | `/team/list` | Список |
| GET | `/team/info` | Информация |
| POST | `/team/model/add` | Добавление модели |
| POST | `/team/model/delete` | Удаление модели |
| GET | `/team/permissions_list` | Список прав |
| POST | `/team/permissions_update` | Обновление прав |
| GET | `/team/daily/activity` | Дневная активность |
