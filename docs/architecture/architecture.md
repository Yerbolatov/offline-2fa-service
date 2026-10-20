# Архитектура системы

## Назначение

Архитектура описывает MVP локального сервиса двухфакторной аутентификации для изолированной корпоративной сети. Сервис работает внутри LAN без внешнего интернета и использует внутренние NTP и CA. Основной второй фактор — TOTP; интеграция выполняется через REST API и forward-auth.

Редактируемый исходник всех трёх диаграмм находится в [offline-2fa-architecture.drawio](../../diagrams/src/offline-2fa-architecture.drawio). Файл содержит три страницы и открывается в diagrams.net/draw.io.

## Диаграмма контекста

На контекстной диаграмме показаны сотрудник, администратор, внутренние приложения/reverse proxy, сервис, внутренний NTP и внутренний CA. Красная граница и пунктирная связь явно показывают отсутствие выхода в интернет.

![Контекст системы](../../diagrams/export/system-context.svg)

## Диаграмма вариантов использования

Диаграмма показывает актёров и основные сценарии MVP. Идентификаторы в скобках ссылаются на требования из [requirements.md](../requirements/requirements.md): вход — FR-001–FR-005, подключение 2FA — FR-003, recovery — FR-006, forward-auth — API-003, сброс устройства — FR-009, аудит — SEC-006, управление пользователями и ролями — FR-008–FR-010.

![Варианты использования](../../diagrams/export/use-cases.svg)

## Черновик диаграммы компонентов

Сервис разделён на API/Web/forward-auth слой, authentication service, TOTP модуль, криптографический модуль, audit service и SQLite. TOTP-секреты хранятся только как AES-256-GCM ciphertext, recovery codes — как хеши, а ключ шифрования находится во внешнем key store и не попадает в БД или Git. Время берётся из внутреннего NTP, TLS доверяет внутреннему CA.

![Компоненты](../../diagrams/export/component-draft.svg)

## Трассировка требований, компонентов и сценариев

| Требования | Компоненты | Сценарии |
|---|---|---|
| FR-001, FR-002, FR-007 | API layer, Authentication service, Cryptography | Вход, смена пароля, отзыв сессии |
| FR-003, FR-004, FR-005 | TOTP module, Cryptography, Internal NTP, SQLite | QR enrollment, проверка TOTP, защита replay |
| FR-006 | TOTP module, Cryptography, SQLite | Использование recovery code |
| FR-008, FR-009, FR-010 | Web interfaces, API layer, Authentication service, Audit service | Управление пользователями, RBAC, reset устройства |
| API-001, API-002 | API layer, Authentication service | REST API и авторизация запросов |
| API-003, API-004 | Forward-auth, Authentication service | Проверка запроса reverse proxy |
| SEC-001, SEC-010 | API layer, Web interfaces, Internal CA | HTTPS и защита browser/API requests |
| SEC-002, SEC-003, SEC-007 | Cryptography, SQLite, External key store | Защита секретов и логов |
| SEC-004, SEC-006, SEC-008 | Authentication service, TOTP module, Audit service | Lockout, аудит, RBAC |
| NFR-001–NFR-006 | Все runtime-компоненты, Internal NTP/CA | Офлайн-развёртывание и backup |

## Архитектурные ограничения и границы

- Интернет, SMS, push, e-mail и облачные MFA-провайдеры не используются.
- WebAuthn, OIDC, LDAP/AD, RADIUS и SIEM-export остаются расширениями.
- SQLite выбран для single-node MVP; PostgreSQL может быть добавлен для multi-instance deployment.