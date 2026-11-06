# ONLYOFFICE 2026-11-06 - необработанные журналы и диагностический след

**Parent document:** [`2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md`](./2026-11-06-onlyoffice-clone-vm-and-rabbitmq.md)
**Назначение:** сырые логи и последовательность команд, подтверждающие root cause. Приведены «как есть», с минимальной правкой переносов строк.
**Среда:** домашняя лаборатория, Hyper-V, Ubuntu Server 24.04, Nextcloud, Docker, Nginx Proxy Manager.
**Период инцидента:** 2026-11-06.
**Публикация:** все домены, IP, MAC и ID контейнеров заменены на плейсхолдеры. Имя `pedantic_carson` оставлено как есть - это автогенерированное имя Docker, оно не чувствительно и иллюстрирует техдолг (запуск без `--name`).

---

## A.1. Контейнер запущен, но приложение не работает

**Команда:**
```bash
sudo docker ps | grep onlyoffice
```

**Вывод:**
```text
<CONTAINER_ID>   onlyoffice/documentserver   "/app/ds/run-documen…"   12 months ago   Up 14 minutes   443/tcp, 0.0.0.0:8888->80/tcp, [::]:8888->80/tcp   pedantic_carson
```

**Интерпретация:** контейнер живёт (`Up 14 minutes`), порт `8888` проброшен. Первое впечатление - «всё запущено». На деле контейнер не был работоспособен (см. A.2).

**Замечание:** имя `pedantic_carson` - авто-сгенерированное (запуск без `--name`). Параметры запуска не зафиксированы, при пересоздании их легко потерять.

---

## A.2. Ключевой симптом: RabbitMQ не стартует (бесконечное ожидание 5672)

**Команда:**
```bash
sudo docker logs --tail 80 pedantic_carson
```

**Вывод (фрагмент, ~80 строк одного и того же):**
```text
Waiting for connection to the localhost host on port 5672
Waiting for connection to the localhost host on port 5672
Waiting for connection to the localhost host on port 5672
... (повторяется)
```

**Интерпретация:** контейнер завис на этапе инициализации. `rabbitmq-server` не поднялся, сервис остался в полузапущенном состоянии. Frontend-nginx принимает TCP-соединения (порт слушается), но проксировать запросы некуда - backend не готов. Отсюда `Connection reset by peer` (см. A.4).

**Это корневой технический симптом инцидента.** Наиболее вероятная причина - повреждение базы RabbitMQ (`/var/lib/rabbitmq/mnesia`) внутри контейнера как следствие жёсткого выключения ВМ при отсутствии именованных volume. Точная формулировка ошибки `rabbitmq-server` не сохранена: контейнер удалён до сохранения полного лога (см. урок в родительском постмортеме, раздел 10).

---

## A.3. Проверка конфигурации старого контейнера (ключевые переменные)

**Команда:**
```bash
sudo docker inspect pedantic_carson --format '{{json .Config.Env}}'
```

**Вывод (ключевые переменные, полный список опущен):**
```text
JWT_ENABLED=false
PG_VERSION=16
BASE_VERSION=24.04
```

**Интерпретация:** `JWT_ENABLED=false` - JWT был отключён изначально. Гипотеза «сломался JWT» отпадает. Проблема не в аутентификации.

---

## A.4. Проверка доступности «изнутри» - соединение рвётся

**Команда:**
```bash
curl -I http://localhost:8888
```

**Вывод:**
```text
curl: (56) Recv failure: Connection reset by peer
```

**Интерпретация:** TCP-соединение устанавливается (nginx слушает порт), но HTTP-ответ не приходит - соединение сбрасывается. Ровно то поведение, которое даёт frontend-nginx без готового backend. Подтверждает диагноз из A.2.

---

## A.5. Сетевая проверка: клон ВМ в той же сети

**Команды:**
```bash
hostname -I
ip a
```

**Вывод (ключевые строки):**
```text
<INTERNAL_IP> 172.17.0.1

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    link/ether <MAC_ADDRESS> brd ff:ff:ff:ff:ff:ff
    inet <INTERNAL_IP>/24 ...
```

**Интерпретация:** оригинал на `<INTERNAL_IP>`, MAC `<MAC_ADDRESS>` (Hyper-V OUI). Параллельно в той же сети был запущен клон ВМ с идентичными MAC/IP/hostname - источник «мигающего» сетевого доступа к оригиналу. Клон выключен вручную; автозапуск отключён на уровне Hyper-V.

---

## A.6. Пересоздание контейнера

**Команды:**
```bash
sudo docker stop pedantic_carson
sudo docker rm pedantic_carson
sudo docker run -d --name onlyoffice \
  -p 8888:80 --restart=always \
  -e JWT_ENABLED=false \
  onlyoffice/documentserver
```

**Вывод:**
```text
pedantic_carson
pedantic_carson
<CONTAINER_ID>
```

**Интерпретация:** старый повреждённый контейнер удалён, запущен новый с фиксированным именем `onlyoffice`, отключённым JWT и автоперезапуском.

> **Примечание:** в команде выше сохранён исторический вид - без `-i -t` (они для демона бесполезны) и **без тега**. В родительском постмортеме, раздел 8 (Resolution), приведена **финальная** версия с зафиксированной версией `onlyoffice/documentserver:9.1.0`. Разница объяснена в разделе 5 (T7) постмортема.

---

## A.7. Успешный старт: RabbitMQ поднимается [ OK ]

**Команда:**
```bash
sudo docker logs -f onlyoffice
```

**Вывод (ключевые строки):**
```text
 * Starting PostgreSQL 16 database server                               [ OK ]
 * Starting RabbitMQ Messaging Server rabbitmq-server                   [ OK ]
Connection to localhost (::1) 5432 port [tcp/postgresql] succeeded!
Connection to localhost (::1) 5672 port [tcp/amqp] succeeded!
ds:docservice: stopped
ds:docservice: started
ds:converter: stopped
ds:converter: started
```

**Ключевой контраст с A.2:**

| Старый контейнер (`pedantic_carson`) | Новый контейнер (`onlyoffice`) |
|---|---|
| `Waiting for connection to the localhost host on port 5672` (бесконечно) | `Connection to localhost (::1) 5672 port [tcp/amqp] succeeded!` |
| `rabbitmq-server` не стартовал | `Starting RabbitMQ Messaging Server rabbitmq-server [ OK ]` |
| `ds:docservice` не запускался | `ds:docservice: started`, `ds:converter: started` |

**Это и есть доказательство:** причина была во внутреннем состоянии старого контейнера (RabbitMQ), а не в конфигурации Nextcloud, NPM или сети. Пересоздание с чистого образа устранило проблему.

---

## A.8. Логи docservice: версия и режим работы

**Вывод (фрагмент логов нового контейнера):**
```text
[2026-11-06T14:14:40.814] [WARN] [localhost] [docId] [userId] nodeJS - Express server listening on port 8000 in production-linux mode. Version: 9.1.0. Build: 168
```

**Интерпретация:** docservice поднялся, версия **9.1.0, build 168**. Предупреждения `notifyLicenseExpiration(): expiration date is not defined` - нормальные для Community Edition.

---

## A.9. Подтверждение статуса JWT

**Команда:**
```bash
sudo docker exec onlyoffice documentserver-jwt-status.sh
```

**Вывод:**
```text
Your JWT settings:
JWT enabled -  false
```

**Интерпретация:** JWT фактически **выключен**. Предупреждение в логах «JWT is enabled by default» - стандартное сообщение скрипта инициализации, оно выводится всегда. При диагностике проверять статус через `documentserver-jwt-status.sh`, а не по строке в логах.

---

## A.10. Проверка доступности «изнутри» - успех

**Команды:**
```bash
curl -I http://localhost:8888
curl -s http://localhost:8888/healthcheck
```

**Вывод:**
```text
HTTP/1.1 308 Permanent Redirect
Server: nginx
Location: http://localhost:8888/welcome/

true
```

**Интерпретация:**
- `308 Permanent Redirect` на `/welcome/` - штатное поведение ONLYOFFICE. Сравните с A.4: было `Recv failure: Connection reset by peer`, стало `308` - сервис ожил.
- `healthcheck → true` - все внутренние компоненты (PostgreSQL, RabbitMQ, docservice, converter) в норме.

---

## A.11. Проверка доступности «снаружи» через NPM - успех

**Команда:**
```bash
curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN>
```

**Вывод:**
```text
HTTP/2 308
server: openresty
location: https://<OFFICE_SUBDOMAIN>.<DOMAIN>/welcome/
x-served-by: <OFFICE_SUBDOMAIN>.<DOMAIN>
```

**Интерпретация:** `HTTP/2 308` - NPM проксирует запрос к контейнеру, тот отвечает редиректом. SSL работает (HTTP/2, Let's Encrypt). Заголовок `x-served-by` от NPM подтверждает, что запрос прошёл через прокси-хост. До восстановления `curl` на тот же адрес отдавал таймаут / 502.

---

## A.12. Извлечение версии (финальная проверка)

**Команда:**
```bash
sudo docker logs onlyoffice 2>&1 | grep -iE "Version|Build"
```

**Вывод:**
```text
[2026-11-06T14:14:40.814] ... Version: 9.1.0. Build: 168
[2026-11-06T14:15:40.881] ... Version: 9.1.0. Build: 168
```

**Интерпретация:** подтверждена версия **9.1.0, build 168**. Две строки - два старта `docservice` (при инициализации контейнера он перезапускается один раз, это нормально). Версия зафиксирована для команды запуска: `onlyoffice/documentserver:9.1.0`.

---

## A.13. Сводка доказательств

| # | Что проверялось | Команда | «До» | «После» |
|---|---|---|---|---|
| 1 | Контейнер запущен | `docker ps` | `Up`, но неработоспособен | `Up`, работоспособен |
| 2 | RabbitMQ | `docker logs` | `Waiting for 5672` (вечно) | `[ OK ]`, `5672 succeeded!` |
| 3 | docservice | `docker logs` | `stopped` | `started` |
| 4 | converter | `docker logs` | `stopped` | `started` |
| 5 | Доступ изнутри | `curl localhost:8888` | `Connection reset by peer` | `308 → /welcome/` |
| 6 | Healthcheck | `curl /healthcheck` | - | `true` |
| 7 | Доступ снаружи | `curl https://...` | таймаут / 502 | `308 → /welcome/` |
| 8 | JWT | `documentserver-jwt-status.sh` | `false` | `false` (не менялся) |
| 9 | Версия | docker logs \| grep Version | не определена (контейнер удалён до снятия версии) | 9.1.0 / 168 |

**Вывод:** зафиксирован переход из состояния «контейнер запущен, но приложение не работает» в состояние «все сервисы подняты, доступ изнутри и снаружи подтверждён, healthcheck = true». Причина поломки и метод восстановления - в родительском постмортеме.

**Почему «до» для версии - «не определена».** Старый контейнер `pedantic_carson` жил ~12 месяцев (см. A.1) и был запущен из образа без тега (`onlyoffice/documentserver` = `latest`). Его фактическая версия на момент инцидента **неизвестна**: контейнер удалён до того, как версия была снята из логов. Это не пробел в диагностике, а **следствие** отсутствия правила «сохраняй логи до удаления». Именно поэтому в A.14 и в родительском постмортеме (раздел 10) это правило зафиксировано отдельно.

---

## A.14. Чек-лист для следующего инцидента

```bash
# 1. Запущен ли контейнер
sudo docker ps | grep onlyoffice

# 2. Если запущен - здоров ли он (главный признак)
curl -s http://localhost:8888/healthcheck
# ожидаемо: true

# 3. Не завис ли RabbitMQ
sudo docker logs --tail 20 onlyoffice | grep -iE "5672|rabbitmq"
# хорошо: "5672 port [tcp/amqp] succeeded!" или "RabbitMQ Messaging Server [ OK ]"
# плохо: "Waiting for connection to the localhost host on port 5672"

# 4. Доступность снаружи
curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN>
# ожидаемо: HTTP/2 308, Location: .../welcome/

# 5. Версия
sudo docker logs onlyoffice 2>&1 | grep -oE "Version: [0-9.]+\. Build: [0-9]+" | tail -1
```
**Правило на будущее (из ревью постмортема):** перед удалением упавшего контейнера - **сначала сохранить логи**:
```bash
sudo docker logs <container_name> > incident-$(date +%F).log 2>&1
```
Только потом `docker rm`. Иначе доказательная база теряется, и root cause остаётся гипотезой.

---

*Конец файла. Ссылка из родительского постмортема: раздел 6 (Root Cause) и раздел 12 (Appendix).*
