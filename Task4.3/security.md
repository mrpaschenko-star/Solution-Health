# Безопасная интеграция мобильного приложения

## Аутентификация и авторизация

[Схема PNG](auth-sequence.png) · [PlantUML](auth-sequence.puml). Используется **OpenID Connect поверх OAuth 2.0 Authorization Code с PKCE S256**. Native-приложение — публичный client: client secret в APK/IPA не размещается. Авторизация открывается в системном браузере через проверенный redirect URI/App Link; `state` защищает привязку ответа, `nonce` проверяется в ID token, а PKCE связывает code exchange с начавшим его клиентом.

Access token передаётся только заголовком `Authorization: Bearer <token>` по HTTPS. ID token служит клиенту для результата аутентификации и не принимается пациентским API вместо access token. Gateway проверяет signature, allowlist алгоритмов, issuer, audience, exp/nbf и scope; patient API повторно обеспечивает объектные права. Верификация телефона/логина не даёт автоматически доступ к одноимённой карточке: привязка IAM subject → CRM patient проходит контролируемую процедуру.

Стартовая политика: access token **10 минут**, rotating refresh token **30 дней** максимальной жизни с ограничением неактивности по политике IAM. Refresh-token reuse отзывает семейство сессии, logout отзывает refresh family; токены хранятся в Keychain/Keystore, не в логах/URL/общих preferences. Self-contained access token не исчезает мгновенно при logout: для медицинских чтений/изменений API проверяет актуальность сессии через IAM/introspection либо согласованную revocation policy. При недоступности проверки и невозможности доказать действительность доступ закрывается; это зависимость availability, требующая HA IAM.

Доступ «мои записи/анализы» определяется субъектом на сервере, без доверия query-параметру patient ID. Врач имеет отдельный workforce client, MFA и разрешение по роли/клинике/отношению лечения; пациентский токен не принимается как врачебный. Для детей и представителей нужны явные делегированные права и audit; до их реализации MVP не показывает чужие карточки. Риск SD-007 расследуется отдельно: ротация сессий, проверка session fixation, authz и аудита; причина не объявляется доказанной подменой токена.

## Rate limiting

Лимиты — стартовые проектные значения, не результат измерений. Алгоритм — token bucket: скорость пополнения и максимальный burst указаны явно. Счётчики общие для gateway replicas, ключи нормализуются, Redis/limiter защищён и отказоустойчив. Это ограничение одного клиента; глобальное admission control задаётся по нагрузочным испытаниям.

| Тип запроса | Лимит | Действие при превышении |
|---|---|---|
| Публичный `GET /api/v1/doctors` | **60 запросов/мин на IP**, refill 1/с, burst **10**; дополнительная abuse-оценка устройства/ASN без доверия подделываемому client ID | `429 Too Many Requests`, `Retry-After` до доступного токена, компактный problem response. Shared NAT может давать ложные ограничения: наблюдать и корректировать; бот-защита не заменяет общий backend budget |
| Авторизованный `POST /api/v1/booking-operations` («создать запись») | **6 новых намерений/мин на subject**, refill 1/10 с, burst **2**. Дополнительный предохранитель **60 запросов/мин на IP**, burst **10** | `429`, `Retry-After`, не создавать SAGA/hold/платёж. Одинаковый idempotency key не расходует квоту новых намерений повторно, но все HTTP-попытки проходят anti-abuse лимит |
| Чтение своей операции/записей | Стартово **60/мин на subject**, refill 1/с, burst **10**, в том числе polling | `429`, `Retry-After`; клиент использует backoff с jitter и не опрашивает несколько экранов независимо |

При отказе общего limiter бронирование fail-closed (`503`), не выдаётся каждой реплике отдельный полный бюджет. Публичный поиск допускает только заранее распределённую локальную квоту на разрешённый кеш или `503`; обращение к backend без ограничений запрещено. Превышение лимита не отменяет уже существующую SAGA. Gateway idempotency lookup — ускорение; durable request/business dedup и объектные права остаются в сервисах.

## Обязательная защита каналов и данных

- TLS 1.2+ с предпочтением TLS 1.3 на внешних HTTPS, строгая проверка certificate chain/hostname, запрет cleartext и отключения проверки сертификата. Certificate pinning допустим только с поддерживаемой ротацией/backup pins; иначе может заблокировать легитимных пациентов при замене сертификата.
- TLS между Gateway, API и внутренними системами; mTLS/workload identity там, где поддерживается. Для legacy SOAP/HTTP — защищённый ingress adapter в изолированном сегменте, дальше шифрованный tunnel/TLS terminator с документированной остаточной plaintext-границей. Старый FTP изолировать, заменить SFTP/FTPS при поддержке оборудования; до этого network segmentation/VPN и минимальные права, без заявления end-to-end шифрования незащищённого участка.
- Least privilege сетей и сервисных ролей: приложение не видит Oracle/1С/LIS/RabbitMQ, backend не принимает произвольный upstream URL. В брокере отдельные users/vhosts/routing permissions, AMQPS; credentials в managed secret store с ротацией. Примеры документации не содержат секретов.
- Объектная авторизация и минимизация ответов на каждом API, ownership проверки результатов и appointments, `Cache-Control: no-store` для медицинских/пациентских ответов. CDN/общий кеш допустим для публичного каталога врачей; результаты не смешиваются по URL между пациентами.
- Parameterized SQL и allowlist фильтров/сортировки в API **и старом веб-портале**, least-privilege DB account, negative injection tests. WAF/валидация полезны, но не заменяют исправление INT-006. Для XML — запрет DTD/external entities, лимиты глубины/размера; JSON/XML schema проверяется на adapter boundary.
- PSP callback: подпись/HMAC или mTLS согласно договору, проверка timestamp/replay window и event ID, сопоставление merchant/account, суммы, валюты и payment reference; lookup provider state для спорного исхода. Нельзя доверять `success=true` из mobile redirect. PAN/CVV не проходят backend клиники: используется hosted provider flow/токенизация; границы платёжного compliance подтверждаются с провайдером.
- Push/SMS/email без диагноза, типа исследования, полиса и иных медицинских подробностей; переход ведёт к повторно авторизованному API. Никаких bearer tokens в deep links. Отдельное согласие на маркетинг и возможность отписки; транзакционные уведомления не смешиваются с рекламными.
- Шифрование sensitive stores/backups, ключи отдельно от данных, аудит чтения/изменения с ограниченным доступом. В trace/log/метриках нет токенов, полных тел результатов и паспортных данных; trace ID не становится идентификатором пациента. Retention, удаление/архив, локализация и медтайна согласуются с ИБ/юристами в применимом российском регулировании; техническая схема сама по себе не доказывает compliance.

## Проверки до запуска

INT-006 и SD-007 — блокирующие safety issues до раскрытия медицинских данных новому каналу. Проверить IDOR/BOLA (подмена appointment/result ID), разные audience/expired/ID tokens, revoked session, refresh reuse, XML XXE, SQL injection, replay PSP callback, смену amount/client quote, обход rate limit через реплики/IP headers, утечку PII в push/log/cache. Для входа через пароль/OTP нужны собственные anti-bruteforce лимиты и MFA/recovery policy IAM, помимо лимитов бизнес-API.
