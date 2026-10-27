# Spec Delta

## Purpose

Предоставляет программным клиентам документированный HTTP JSON API с теми же правилами контактов и поиска, которые доступны пользователю через браузер.

## ADDED Requirements

### Requirement: Versioned contact endpoints
Система SHALL предоставлять `POST /api/v1/contacts`, `GET /api/v1/contacts/{id}`, `GET /api/v1/contacts/by-name?name=...` и `GET /api/v1/contacts/by-phone?phone=...`. Все ответы этих маршрутов SHALL использовать JSON в UTF-8 и `Content-Type: application/json`. Поисковые маршруты SHALL принимать `limit` и `after_id` из contact-lookup.

#### Scenario: Programmatic round trip
- **WHEN** клиент отправляет `POST /api/v1/contacts` с `Content-Type: application/json` и телом `{"name":"Анна","phone":"+7 (999) 123-45-67"}`
- **THEN** сервер отвечает `201` с объектом `{"id":"<id>","name":"Анна","phone":"+7 (999) 123-45-67"}`
- **AND** `Location` указывает на `/api/v1/contacts/<id>`
- **WHEN** клиент выполняет GET по этому адресу
- **THEN** сервер отвечает `200` с тем же объектом

#### Scenario: Unknown and malformed identifiers
- **WHEN** клиент запрашивает корректный положительный `id`, которого нет в базе
- **THEN** сервер отвечает `404` с кодом `not_found`
- **WHEN** `id` не является положительным десятичным целым в диапазоне uint64
- **THEN** сервер отвечает `400` с кодом `validation_error`

### Requirement: Search response contract
Успешный поиск SHALL возвращать `200` и объект `{"items":[...],"next_after_id":null}`; элементы SHALL иметь поля `id`, `name`, `phone`. При наличии продолжения `next_after_id` SHALL быть строкой идентификатора согласно contact-lookup. Имя и телефон в query SHALL декодироваться как обычные URL-параметры; клиент SHALL кодировать литеральный `+` как `%2B`.

#### Scenario: API lookup and next page
- **WHEN** клиент запрашивает `/api/v1/contacts/by-phone?phone=%2B7%20%28999%29%20123-45-67&limit=1`, и номер связан с двумя контактами
- **THEN** ответ `200` содержит один контакт и строковый `next_after_id`
- **WHEN** клиент передаёт этот `next_after_id` как `after_id` с тем же номером
- **THEN** ответ содержит второй контакт и `next_after_id: null`

#### Scenario: Empty successful search
- **WHEN** корректный поисковый запрос не имеет совпадений
- **THEN** сервер отвечает `200` с `{"items":[],"next_after_id":null}`

### Requirement: Bounded and validated requests
POST SHALL принимать ровно один JSON-объект с обязательными строками `name` и `phone`, без неизвестных полей, повторных ключей и данных после объекта. Тело SHALL ограничиваться 4 KiB. Неверный JSON, типы, обязательные параметры или значения SHALL давать `400`; неподдерживаемый Content-Type — `415`; превышение размера — `413`. Ни одна такая ошибка SHALL не создавать контакт.

#### Scenario: Reject malformed payloads
- **WHEN** отправлен пустой объект, `null` вместо имени, неизвестное поле, повторный ключ, два JSON-объекта, некорректный UTF-8 либо невалидное имя или телефон
- **THEN** сервер отвечает `400` с кодом `validation_error`, контакт не создаётся

#### Scenario: Reject unsupported or large bodies
- **WHEN** POST передан как `text/plain`
- **THEN** сервер отвечает `415` с кодом `unsupported_media_type`
- **WHEN** тело JSON превышает 4 KiB
- **THEN** сервер отвечает `413` с кодом `request_too_large`

### Requirement: Structured errors and duplicate handling
Ошибки API SHALL иметь форму `{"error":{"code":"...","message":"..."}}`; ошибки поля SHALL дополнительно содержать `field`. Дубликат SHALL давать `409`, код `duplicate_contact` и `existing_id` внутри `error`. Отказ хранилища SHALL давать `503/storage_unavailable`, неожиданная внутренняя ошибка — `500/internal_error`, неподдерживаемый метод известного маршрута — `405/method_not_allowed` с `Allow`.

#### Scenario: Duplicate response
- **WHEN** клиент повторяет добавление уже сохранённой пары с эквивалентным форматированием телефона
- **THEN** он получает `409` с `error.code=duplicate_contact` и `error.existing_id`, равным строковому `id` ранее сохранённого контакта

#### Scenario: Distinguish field and storage errors
- **WHEN** поле `phone` невалидно
- **THEN** клиент получает `400`, `error.code=validation_error` и `error.field=phone`
- **WHEN** невозможно выполнить запись или синхронизацию данных
- **THEN** клиент получает `503` с кодом `storage_unavailable`, а не ответ об успешном добавлении

#### Scenario: Unsupported method
- **WHEN** клиент вызывает `DELETE /api/v1/contacts/1`
- **THEN** сервер отвечает `405`, JSON-ошибкой `method_not_allowed` и `Allow: GET`

### Requirement: Local service boundary and documentation
При обычном запуске сервер SHALL слушать `127.0.0.1` и обслуживать UI и API с одного origin без необходимости внешнего сервиса. Документация SHALL описывать запуск, каталог данных, все маршруты, форматы, коды ошибок, точность поиска и пагинацию, включая примеры запросов по имени и телефону.

#### Scenario: First programmatic use
- **WHEN** разработчик следует инструкции на машине без другой СУБД, собирает сервер и запускает его с пустым каталогом
- **THEN** он может добавить контакт и найти его обоими способами по документированным примерам
- **AND** сервер по умолчанию не принимает подключения через сетевые интерфейсы, отличные от loopback
