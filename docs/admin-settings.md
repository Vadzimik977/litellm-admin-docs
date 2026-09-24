# Страница Settings: настройки админки

Страница `/ui/settings` — общие настройки UI и прокси.

## Разделы настроек

### UI Settings

| Параметр | Описание |
| -------- | -------- |
| `disable_model_add_for_internal_users` | Запрет добавления моделей для internal users |
| `disable_team_admin_delete_team_user` | Запрет team admin удалять участников |
| `enabled_ui_pages_internal_users` | Список страниц, доступных internal users |
| `require_auth_for_public_ai_hub` | Требовать авторизацию для публичного AI Hub |
| `scope_user_search_to_org` | Ограничить поиск пользователей организацией |
| `disable_custom_api_keys` | Запрет пользовательские ключи |
| `disable_key_generate_for_org_admin` | Запрет org admin генерировать ключи |
| `forward_client_headers_to_llm_api` | Пробрасывать клиентские заголовки |
| `forward_llm_provider_auth_headers` | Пробрасывать auth-заголовки провайдера |
| `disable_agents_for_internal_users` | Запрет агентов для internal users |
| `disable_vector_stores_for_internal_users` | Запрет векторных хранилищ |

### Theme Settings

Настройки внешнего вида: лого, favicon, цвета, тёмный/светлый режим.

### SSO Settings

Настройки единого входа: Google, Microsoft, Generic OIDC.

### Internal User Settings

Дефолтные параметры для новых internal users: `max_budget`, `budget_duration`, `models`.

### Default Team Settings

Дефолтные параметры для новых команд.

### Allowed IPs

Белый список IP-адресов для доступа к прокси.

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| GET | `/get/ui_settings` | Общие UI-настройки |
| PATCH | `/update/ui_settings` | Обновление |
| GET | `/get/ui_theme_settings` | Тема |
| PATCH | `/update/ui_theme_settings` | Обновление темы |
| POST | `/upload/logo` | Загрузка логотипа |
| GET | `/get/allowed_ips` | Белый список IP |
| POST | `/add/allowed_ip` | Добавление IP |
| DELETE | `/delete/allowed_ip` | Удаление IP |
| GET | `/get/internal_user_settings` | Настройки users |
| PATCH | `/update/internal_user_settings` | Обновление |
| GET | `/get/default_team_settings` | Настройки команд |
| PATCH | `/update/default_team_settings` | Обновление |
| GET | `/get/sso_settings` | Настройки SSO |
| PATCH | `/update/sso_settings` | Обновление SSO |
