# Hardening: включение JWT в ONLYOFFICE Docs

- **Parent document:** [`2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md`](./2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md)
- **Evidence:** [`2026-11-06-onlyoffice-clone-vm-and-rabbitmq.evidence.md`](./2026-11-06-onlyoffice-clone-vm-and-rabbitmq.evidence.md)
- **Related:** [`2026-11-06-onlyoffice-uptime-kuma.md`](./2026-11-06-onlyoffice-uptime-kuma.md)
- **Статус:** Applied
- **Компонент:** ONLYOFFICE Docs (Docker) → Nextcloud (коннектор ONLYOFFICE) → Nginx Proxy Manager
- **Среда:** домашняя лаборатория, Hyper-V, Ubuntu Server 24.04, Nextcloud, Docker, Nginx Proxy Manager

## 1. Контекст

После восстановления ONLYOFFICE Docs (см. родительский постмортем) сервис работал с отключённым JWT (`-e JWT_ENABLED=false`). Это было оправдано на этапе восстановления - нужно было быстро вернуть работоспособность.

Однако сервис опубликован в интернет через публичный поддомен `https://<OFFICE_SUBDOMAIN>.<DOMAIN>` (NPM + Let's Encrypt). Отключённый JWT на публично доступном сервисе - риск: любой, кто узнает внутренний идентификатор документа (`docId`/`key`), мог бы обратиться к ONLYOFFICE напрямую, минуя Nextcloud.

Внешний ревью обратил на это внимание. Решение: включить JWT и настроить обмен секретом между ONLYOFFICE и Nextcloud. Это **упреждающее** изменение - не реакция на инцидент, а закрытие известной дыры до того, как она сработала.

## 2. Что такое JWT и что он даёт

JWT (JSON Web Token) - механизм подписи запросов между Nextcloud и ONLYOFFICE Docs. Когда JWT включён:

- Каждый запрос на открытие/сохранение документа подписывается секретом (`JWT_SECRET`).
- ONLYOFFICE отвергает запросы без валидной подписи.
- Прямое обращение к `https://<OFFICE_SUBDOMAIN>.<DOMAIN>` без подписи не позволяет открыть документ.

Когда JWT выключен - ONLYOFFICE принимает запросы на открытие документа по `docId`/`key`, если знает адрес. Идентификаторы случайны и живут ограниченное время, поэтому риск низкий, но не нулевой. Для публично доступного сервиса JWT - базовый уровень гигиены.

**Ключевое:** JWT - это не «пароль для пользователя» (пользователи входят в Nextcloud как обычно). Это **подпись между сервисами**, которая работает «под капотом» и не мешает работе. Именно поэтому её нужно включать, а не закрывать NPM паролем (Basic Auth сломал бы коннектор Nextcloud).

## 3. Решение

### 3.1. Сгенерировать секрет

```bash
openssl rand -hex 32
```

Вывод - 64 hex-символа. Это `JWT_SECRET`. Сохранить в менеджере секретов. Не публиковать.

### 3.2. Пересоздать контейнер с явным секретом

> **Перед запуском:** замените `<JWT_SECRET>` на значение из шага 3.1. В публичном репозитории секрет **никогда** не приводится в открытом виде - только плейсхолдер. Значение должно быть **непустым**: `-e JWT_SECRET=` (пустая строка) формально включает JWT, но не защищает и ломает подпись со стороны Nextcloud - документы не откроются.

```bash
sudo docker stop onlyoffice
sudo docker rm onlyoffice
sudo docker run -i -t -d --name onlyoffice \
  -p 8888:80 --restart=always \
  -e JWT_ENABLED=true \
  -e JWT_SECRET=<JWT_SECRET> \
  onlyoffice/documentserver:9.1.0
```

Дождаться полного старта (1–3 минуты):

```bash
sudo docker logs -f onlyoffice
```

Признаки успеха: `RabbitMQ ... [ OK ]`, `ds:docservice: started`, `ds:converter: started`.

### 3.3. Проверить статус JWT

```bash
sudo docker exec onlyoffice documentserver-jwt-status.sh
```

Ожидаемо:

```text
JWT enabled -  true
JWT secret  -  <ваш секрет>
JWT header  -  Authorization
```

Если `JWT secret` пуст - секрет не применился. Проверить фактическое значение в `local.json`:

```bash
sudo docker exec onlyoffice bash -c 'grep -A2 -i "secret" /etc/onlyoffice/documentserver/local.json | sed "s/\"string\": \"[^\"]*\"/\"string\": \"***\"/g"'
```

Три поля `inbox`, `outbox`, `session` должны содержать одинаковое непустое значение (в выводе замаскировано как `***`).

### 3.4. Прописать секрет в Nextcloud

**Администрирование → ONLYOFFICE:**

- **Адрес ONLYOFFICE Docs:** `https://<OFFICE_SUBDOMAIN>.<DOMAIN>`
- **Секретный ключ:** `<JWT_SECRET>` (тот же, что в контейнере)

**Сохранить.** Коннектор перепроверит соединение, индикатор должен стать зелёным.

### 3.5. Проверить открытие документа

Открыть любой `.docx` в Nextcloud. Если редактор загрузился и копирование-вставка работают - JWT настроен корректно.

Дополнительно:

```bash
curl -s http://localhost:8888/healthcheck
# ожидаемо: true
```

Healthcheck может не требовать JWT, поэтому главный тест - именно открытие документа.

## 4. Доказательства

### 4.1. JWT-статус

```text
JWT enabled -  true
JWT secret  -  <секрет отображается (замаскирован при публикации)>
JWT header  -  Authorization
```

### 4.2. Конфиг внутри контейнера

```bash
sudo docker exec onlyoffice bash -c 'grep -A2 -i "secret" /etc/onlyoffice/documentserver/local.json | sed "s/\"string\": \"[^\"]*\"/\"string\": \"***\"/g"'
```

Ожидаемо - три поля `***`:

```json
"secret": {
  "inbox": { "string": "***" },
  "outbox": { "string": "***" },
  "session": { "string": "***" }
}
```

### 4.3. Логи docservice

В логах `docservice` присутствует строка, подтверждающая, что токены включены:

```text
Set services.CoAuthoring.token.enable.browser, services.CoAuthoring.token.enable.request.inbox, services.CoAuthoring.token.enable.request.outbox in the Document Server config to prevent an unauthorized access to your documents and the substitution of important parameters in Document Server requests.
```

Эта строка выводится всегда, но теперь она соответствует фактическому состоянию - токены действительно проверяются.

### 4.4. Версия

```text
Version: 9.1.0. Build: 168
```

## 5. Уроки

- **Всегда задавать явный `JWT_SECRET`.** Пустое значение (`-e JWT_SECRET=`) формально включает JWT, но не защищает и ломает подпись со стороны Nextcloud - документы не откроются.
- **Не доверять строке в логах.** `JWT is enabled by default. A random secret is generated automatically` выводится скриптом инициализации **всегда**, независимо от фактического состояния. Источник правды - `documentserver-jwt-status.sh` и `local.json`.
- **Секрет должен совпадать на обеих сторонах.** Если в контейнере один секрет, а в Nextcloud другой (или пусто) - документы не откроются с ошибкой «Приложение ONLYOFFICE недоступно». Самая частая причина сбоя после включения JWT.
- **Хранить секрет в менеджере секретов.** Не в Git, не в DokuWiki, не в заметках. Секрет от `openssl rand -hex 32` восстановить нельзя - только пересоздать контейнер с новым.
- **JWT + публичный поддомен + Let's Encrypt - базовый уровень для self-hosted.** Отключённый JWT оправдан только на этапе восстановления или в полностью изолированной сети.
- **Не закрывать NPM паролем (Basic Auth) вместо JWT.** Basic Auth защищает вход браузера, но ломает серверную часть коннектора Nextcloud - он не умеет вводить пароль в форму. JWT защищает «под капотом» и не мешает пользователям.

## 6. Профилактика

- При каждом пересоздании контейнера ONLYOFFICE явно передавать `-e JWT_ENABLED=true -e JWT_SECRET=<JWT_SECRET>`.
- Секрет хранить в менеджере секретов.
- После любого пересоздания контейнера проверять `documentserver-jwt-status.sh` и открытие `.docx`.
- Не публиковать секрет в GitHub / DokuWiki / чатах. В постмортеме и логах - только плейсхолдер `<JWT_SECRET>`.
- При ротации секрета: пересоздать контейнер с новым `JWT_SECRET`, обновить поле «Секретный ключ» в Nextcloud, сохранить, проверить открытие документа.
- Добавить healthcheck в Uptime Kuma - см. [`2026-11-06-onlyoffice-uptime-kuma.md`](./2026-11-06-onlyoffice-uptime-kuma.md).

## 7. Чек-лист включения JWT

- [ ] Сгенерирован секрет: `openssl rand -hex 32`
- [ ] Секрет сохранён в менеджере секретов
- [ ] Контейнер пересоздан с `-e JWT_ENABLED=true -e JWT_SECRET=<JWT_SECRET>` (значение **непустое**)
- [ ] `documentserver-jwt-status.sh` показывает `JWT enabled - true` и непустой `JWT secret`
- [ ] `local.json` содержит одинаковый непустой секрет в `inbox`, `outbox`, `session`
- [ ] В Nextcloud → ONLYOFFICE поле «Секретный ключ» = тот же секрет
- [ ] Настройки сохранены, индикатор соединения зелёный
- [ ] `.docx` открывается, копирование-вставка и форматирование работают
- [ ] `curl -s http://localhost:8888/healthcheck` → `true`
- [ ] Checkpoint/экспорт ВМ сделан

## 8. Ссылки

- Родительский постмортем: [`2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md`](./2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md)
- Evidence: [`2026-11-06-onlyoffice-clone-vm-and-rabbitmq.evidence.md`](./2026-11-06-onlyoffice-clone-vm-and-rabbitmq.evidence.md)
- Мониторинг: [`2026-11-06-onlyoffice-uptime-kuma.md`](./2026-11-06-onlyoffice-uptime-kuma.md)
- Официальная документация ONLYOFFICE по JWT: https://api.onlyoffice.com/editors/signature/
