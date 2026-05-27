# Проектирование API

## Цели

API описывает целевой контракт web-клиента и будущих мобильных клиентов. В MVP реализуется подмножество ресурсов редактора, промежуточной валидации, snapshot-версий и одного формата экспорта; импорт, согласование и STOMP-контракт вводятся в post-MVP релизах.

Внешний REST prefix: `/api/v1`.

## Принципы

- URI содержат ресурсы во множественном числе, а не названия операций;
- идентификаторы передаются в path, фильтры и курсоры - в query;
- массовые операции редактирования не входят в MVP;
- создание ресурса возвращает `201 Created` и `Location`;
- асинхронная операция возвращает `202 Accepted` и ссылку на job-ресурс;
- коллекции используют cursor pagination с `limit <= 100`;
- момент времени передается в ISO 8601 UTC, например `2026-06-30T12:15:00Z`;
- поля API используют `snake_case`;
- неизвестные поля ответа клиенты должны игнорировать.

## Аутентификация и авторизация

REST и STOMP CONNECT используют access token внешнего identity provider. Spring Security на входе в backend проверяет подпись, issuer, audience и срок действия токена. Прикладные модули дополнительно проверяют роль и доступ пользователя к образовательной программе.

Для REST используется `Authorization: Bearer <token>`. Для STOMP токен передается в заголовке CONNECT, а не в query string.

Роли не кодируются в URI. Недостаток прав возвращает `403`, а отсутствие ресурса за пределами организационной области - `404`, чтобы не раскрывать его существование.

Авторизация использует RBAC+ABAC:

1. Security filter chain проверяет внешний токен и не принимает присланные клиентом внутренние identity headers.
2. Identity связывает внешний `sub` с локальным `user_id`, ролями и организационными scopes.
3. Backend формирует доверенный authorization context текущего запроса с `user_id`, `organization_id`, roles, scopes и `authz_version`.
4. Прикладной модуль проверяет разрешение роли на действие и принадлежность запрошенного ресурса разрешенной организации или образовательной программе.
5. Решение о доступе принимается до чтения или изменения бизнес-данных; одного знания UUID недостаточно.

Контекст ограничен временем запроса или WebSocket-сессии и не сериализуется для внутрипроцессных вызовов. Изменения критических прав инвалидируют кеш авторизации. При выделении модуля в сервис контекст передается как короткоживущий подписанный token с отдельными `issuer` и `audience`.

При STOMP CONNECT Collaboration использует ту же проверку внешнего токена. Каждая SUBSCRIBE/SEND операция дополнительно проверяет destination и доступ пользователя к указанному `curriculum_id`.

## Конкурентное редактирование

Ответы с изменяемыми сущностями содержат `ETag`, построенный из `row_version` или ревизии плана. `PATCH` и `DELETE` требуют `If-Match`.

`GET /curricula/{curriculum_id}` принимает `If-None-Match` для эффективного повторного чтения и возвращает `304 Not Modified`, если ревизия не изменилась.

Если клиент прислал устаревшую версию, API возвращает `409 Conflict` с кодом `revision_conflict`, текущим ETag и безопасным представлением актуальной сущности в `details`. Клиент показывает конфликт или повторяет изменение после явного объединения данных.

UTC не зависит от региона развертывания и однозначно представляет момент времени при обмене между компонентами. Web-клиент отображает его в зоне пользователя, для ВШЭ по умолчанию `Europe/Moscow`. Локальные календарные значения без времени, например год набора или дата действия норматива, передаются без преобразования в UTC.

Во всех релизах `Idempotency-Key` не используется в обычных `PATCH`, `PUT`, `DELETE` и CRUD-созданиях. Он обязателен для доступных в соответствующем релизе операций, случайный повтор которых создает значимый побочный эффект: сохранение версии, запуск согласования, решение согласующего, создание и применение импорта, экспорт. Один ключ в области пользователя и endpoint возвращает тот же результат повторного запроса и хранится не менее 24 часов.

## Основные REST-ресурсы

### Учебные планы

| Метод | URI | Результат |
|---|---|---|
| `GET` | `/curricula` | список планов с фильтрами по программе, направлению и году набора |
| `POST` | `/curricula` | создать план |
| `GET` | `/curricula/{curriculum_id}` | получить агрегат редактора и ETag |
| `PATCH` | `/curricula/{curriculum_id}` | изменить метаданные плана |
| `DELETE` | `/curricula/{curriculum_id}` | архивировать план |

### Структура плана

| Метод | URI | Результат |
|---|---|---|
| `POST` | `/curricula/{curriculum_id}/blocks` | добавить блок |
| `PATCH` | `/curricula/{curriculum_id}/blocks/{block_id}` | изменить блок |
| `DELETE` | `/curricula/{curriculum_id}/blocks/{block_id}` | удалить пустой блок |
| `POST` | `/curricula/{curriculum_id}/disciplines` | включить каталожную дисциплину в план |
| `PATCH` | `/curricula/{curriculum_id}/disciplines/{discipline_id}` | изменить параметры дисциплины |
| `DELETE` | `/curricula/{curriculum_id}/disciplines/{discipline_id}` | удалить дисциплину из плана |
| `PUT` | `/curricula/{curriculum_id}/disciplines/{discipline_id}/workloads/{period_id}` | создать или полностью заменить нагрузку модуля |
| `DELETE` | `/curricula/{curriculum_id}/disciplines/{discipline_id}/workloads/{period_id}` | удалить нагрузку модуля |
| `POST` | `/curricula/{curriculum_id}/prerequisites` | создать связь пререквизита |
| `DELETE` | `/curricula/{curriculum_id}/prerequisites/{prerequisite_id}` | удалить связь |

Каждый запрос изменяет одну сущность. Перенос набора дисциплин, копирование модулей и пакетная замена часов не поддерживаются в MVP.

### Версии и сравнение

| Метод | URI | Результат |
|---|---|---|
| `GET` | `/curricula/{curriculum_id}/versions` | список сохраненных версий |
| `POST` | `/curricula/{curriculum_id}/versions` | сохранить immutable snapshot и вернуть статус его проверки |
| `GET` | `/curricula/{curriculum_id}/versions/{version_id}` | получить метаданные и snapshot версии |
| `GET` | `/curricula/{curriculum_id}/version-comparisons?from_version_id=...&to_version_id=...` | структурированное сравнение двух версий |
| `POST` | `/curricula/{curriculum_id}/restorations` | восстановить snapshot в рабочее состояние как новую ревизию |

Diff формируется по стабильным UUID сущностей и содержит:

- `added` - сущности, появившиеся в целевой версии;
- `removed` - сущности, отсутствующие в целевой версии;
- `modified` - сущности с массивом измененных полей и значениями `before`/`after`;
- `summary` - количество изменений по типам `BLOCK`, `DISCIPLINE`, `WORKLOAD`, `CONTROL_FORM`, `PREREQUISITE`.

Порядок элементов не считается изменением сам по себе, если `sort_order` не изменился. Сравнивать можно только версии одного плана.

В MVP snapshot проверяется синхронно фиксированным набором правил и обычно возвращается со статусом `COMPLETED`. В Post-MVP 1 Versioning одной транзакцией сохраняет версию со статусом `PENDING` и job в DB-backed очереди. Worker выполняет проверку асинхронно, а клиент получает обновления через `GET` версии и персональные STOMP-уведомления.

### Валидация

`POST /curricula/{curriculum_id}/validation-runs` синхронно создает запуск проверки текущей ревизии или указанной версии и возвращает `201 Created` с результатами. Проверка должна укладываться в server timeout; базовый целевой предел - 5 секунд.

Если `version_id` не передан, выполняется промежуточная проверка текущей рабочей ревизии без создания snapshot. Результат содержит `source_revision` и считается устаревшим после следующего изменения плана. Промежуточная проверка носит рекомендательный характер и не может использоваться как основание для запуска согласования.

Запуски и результаты доступны через:

- `GET /curricula/{curriculum_id}/validation-runs`;
- `GET /curricula/{curriculum_id}/validation-runs/{run_id}`.

Если фактическое время стабильно превышает предел, контракт расширяется асинхронным режимом с `202`, но это не входит в MVP.

Технический status запуска: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `UNAVAILABLE`. Результат правила и агрегированный outcome: `ACCEPT`, `WARNING`, `ERROR`. `INFO` в MVP не используется.

Ручной запуск через REST остается синхронным. Асинхронный запуск для сохраненной версии создается worker DB-backed очереди, но доступен через те же GET-ресурсы validation runs.

### Согласование (Post-MVP 2)

| Метод | URI | Результат |
|---|---|---|
| `POST` | `/approval-instances` | начать согласование сохраненной версии |
| `GET` | `/approval-instances/{approval_id}` | получить маршрут, этапы и замечания |
| `POST` | `/approval-instances/{approval_id}/decisions` | согласовать, отклонить или вернуть на доработку |
| `POST` | `/approval-instances/{approval_id}/comments` | добавить замечание к версии или сущности snapshot |

Decision и comment являются отдельными неизменяемыми ресурсами. `Idempotency-Key` требуется для decision, но не для обычного комментария.

Создание approval instance запрещено до `validation_status = COMPLETED` и при outcome `ERROR`. Для outcome `WARNING` request должен содержать непустое `warning_acknowledgement`.

### Импорт (Post-MVP 1) и экспорт

`POST /imports` принимает `multipart/form-data`, синхронно проверяет и разбирает файл и возвращает `201 Created` с нормализованным staging JSON и issues. Временный файл удаляется после обработки. `GET /imports/{import_id}` повторно возвращает сохраненный staging, а `POST /imports/{import_id}/applications` синхронно и атомарно применяет подтвержденные данные.

`ERROR` блокирует применение; `WARNING` требует явного подтверждения. Исходный бинарный файл не хранится, поэтому после технического сбоя до создания staging пользователь загружает его повторно.

`GET /curricula/{curriculum_id}/versions/{version_id}/export?format=XLSX` синхронно формирует экспорт и передает его потоковым ответом с корректными `Content-Type` и `Content-Disposition`. PDF, JSON и XML добавляются в Post-MVP 1. Артефакт не сохраняется; повторный запрос генерирует его заново.

## STOMP-контракт (Post-MVP 1)

WebSocket endpoint: `/ws`.

| Направление | Destination | Payload |
|---|---|---|
| subscribe | `/topic/curricula/{curriculum_id}/changes` | подтвержденное изменение и новая ревизия |
| subscribe | `/topic/curricula/{curriculum_id}/presence` | список активных пользователей |
| subscribe | `/user/queue/notifications` | персональные workflow/job-уведомления |
| subscribe | `/user/queue/errors` | ошибка с `request_id` и признаком resync |
| send | `/app/curricula/{curriculum_id}/presence` | вход, heartbeat или выход из редактора |

Web-клиент открывает одно STOMP/WebSocket-соединение к `/ws` и создает на нем все необходимые подписки. Live-изменения, presence, персональные ошибки и уведомления используют разные destinations, но не разные сетевые соединения.

Notification формирует адресное событие, а Collaboration отправляет его в `/user/queue/notifications`. Инициировавший REST-команду пользователь получает основной результат из HTTP response; STOMP нужен для фоновых результатов и изменений, возникших вне текущего запроса.

Изменение плана не отправляется в `/app`: клиент выполняет REST-команду, Curriculum фиксирует ее в PostgreSQL, после чего Collaboration получает событие after-commit и публикует подтвержденное изменение в topic. Outbox добавляется, если доставка пересекает границу процесса.

Каждое сообщение содержит `event_id`, `event_type`, `occurred_at`, `curriculum_id`, `revision` и `correlation_id`. Клиент игнорирует уже обработанные `event_id`; при пропуске ревизии повторно получает план через REST.

## Ошибки

Единый формат:

```json
{
  "error": {
    "code": "revision_conflict",
    "message": "The resource was changed by another user",
    "details": [
      {
        "field": "row_version",
        "reason": "stale_value",
        "current_etag": "\"17\""
      }
    ],
    "request_id": "req_01J..."
  }
}
```

Основные коды:

| HTTP | `code` | Сценарий |
|---|---|---|
| `400` | `malformed_request` | некорректный JSON или параметр |
| `401` | `authentication_required` | отсутствует или недействителен токен |
| `403` | `access_denied` | недостаточно прав |
| `404` | `resource_not_found` | ресурс отсутствует или недоступен в области пользователя |
| `409` | `revision_conflict` | устаревший ETag |
| `409` | `business_conflict` | нарушено состояние workflow |
| `422` | `validation_error` | команда синтаксически корректна, но нарушает инвариант |
| `429` | `rate_limited` | превышен лимит запросов |
| `503` | `dependency_unavailable` | недоступна обязательная внешняя система |

## Ограничения и лимиты

- максимальный `limit` коллекции: 100;
- максимальный размер импортируемого файла задается конфигурацией, начальное значение 50 MiB;
- синхронные импорт и экспорт ограничены server timeout, размером файла и числом одновременных операций;
- backend применяет отдельные rate limits к чтению, экспортам и загрузкам;
- `request_id` и W3C `traceparent` проходят через reverse proxy, backend и STOMP events; при принятии Kafka context передается также в message headers.

## OpenAPI

Точкой входа в машиночитаемый контракт остается `docs/api/openapi.yaml`. Спецификация разделена по доменным областям:

- `docs/api/paths` - curricula, versions, validation, approvals и files;
- `docs/api/components/schemas` - схемы тех же доменных областей и общие схемы;
- `docs/api/components/parameters.yaml` - повторно используемые параметры;
- `docs/api/components/headers.yaml` - общие заголовки;
- `docs/api/components/responses.yaml` - единые ответы с ошибками.

В CI корневой файл проверяется командой `redocly lint docs/api/openapi.yaml`. Для генераторов клиентов и публикации документации создается единый артефакт командой `redocly bundle docs/api/openapi.yaml -o build/openapi.yaml`.
