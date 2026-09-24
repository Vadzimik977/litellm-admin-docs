# Страница Users: управление пользователями

Страница `/ui/users` — CRUD внутренних пользователей (Internal Users).

## Что показывает

- Список пользователей с ролями, расходом, числом ключей
- Создание / обновление / удаление пользователей
- Invite flow (приглашение по email)
- Массовое обновление

## Роли

| Роль | Значение | Права |
| ---- | -------- | ----- |
| `proxy_admin` | PROXY_ADMIN | Полный доступ ко всему |
| `proxy_admin_viewer` | PROXY_ADMIN_VIEW_ONLY | Просмотр без изменений |
| `internal_user` | INTERNAL_USER | CRUD собственных ключей, просмотр расхода |
| `internal_user_viewer` | INTERNAL_USER_VIEW_ONLY | Только просмотр |

## Как создаётся пользователь

1. Проверка дубликатов (`user_id`, `email`)
2. Проверка лимита лицензии
3. Применение `default_internal_user_params`
4. Пароль хэшируется через `scrypt`
5. Если `auto_create_key = true` — создаётся API-ключ автоматически
6. Пользователь добавляется в команду (`team_id`)
7. Вызывается хук `async_user_created_hook`

## Budget для пользователя

- При создании `INTERNAL_USER` автоматически берутся `litellm.max_internal_user_budget` и `litellm.internal_user_budget_duration`
- Поля: `max_budget`, `soft_budget`, `budget_duration`, `tpm_limit`, `rpm_limit`, `max_parallel_requests`
- Non-admin не могут менять свои `max_budget`, `soft_budget`, `spend`

## Массовое обновление

Два режима:
1. Конкретные пользователи: `users: [UpdateUserRequest, ...]`
2. Все пользователи: `all_users: true` + `user_updates: {...}` (макс. 500)

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| POST | `/user/new` | Создание |
| POST | `/user/update` | Обновление |
| POST | `/user/bulk_update` | Массовое обновление |
| GET | `/user/list` | Список |
| GET | `/user/info` | Информация |
| POST | `/user/delete` | Удаление |
| GET | `/user/available_roles` | Доступные роли |
| GET | `/user/filter/ui` | Поиск для UI |
| GET | `/user/daily/activity` | Дневная активность |
