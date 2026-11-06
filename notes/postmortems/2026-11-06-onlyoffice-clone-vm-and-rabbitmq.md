# Postmortem: ONLYOFFICE Docs - отказ после переноса ВМ и восстановление контейнера

**Статус:** Resolved
**Компонент:** ONLYOFFICE Docs (Docker) → Nextcloud (коннектор ONLYOFFICE) → Nginx Proxy Manager
**Среда:** домашняя лаборатория, Hyper-V, Ubuntu Server 24.04, Nextcloud, Docker, Nginx Proxy Manager
**Версия ONLYOFFICE Docs (определена постфактум):** 9.1.0, Build 168
**Характер:** постмортем по реальному инциденту, без поиска виноватых
**Публикация:** все домены, IP, имена ВМ и учётные записи заменены на плейсхолдеры.

---

## 1. Summary

В работающем Nextcloud перестал открываться онлайн-офис на базе ONLYOFFICE Docs. При попытке открыть документ появлялась ошибка:

> Приложение ONLYOFFICE недоступно. Пожалуйста, свяжитесь с администратором.

Сам Nextcloud оставался стабильным: файлы, календарь, магазин приложений работали. Проблема была локальной, но критичной - онлайн-офис был недоступен полностью.

Диагностика показала, что контейнер ONLYOFFICE запущен, но завис на этапе инициализации: внутренний RabbitMQ не стартовал, и весь сервис оставался в полузапущенном состоянии. Параллельно был выявлен сетевой конфликт: в той же сети был запущен клон ВМ с идентичными MAC/IP/hostname, из-за чего оригинал терял сеть.

**Итоговое решение:** устранить сетевой конфликт (выключить клон) и пересоздать контейнер ONLYOFFICE с чистого образа. После этого онлайн-офис заработал полностью.

---

## 2. Impact

- Онлайн-офис в Nextcloud был полностью недоступен.
- Документы `.docx` не открывались - ошибка «Приложение ONLYOFFICE недоступно».
- Остальные сервисы Nextcloud работали штатно.
- Для домашней лаборатории, используемой как личная self-hosted инфраструктура, это был критичный функциональный сбой: онлайн-офис был нужен как замена настольному пакету.

---

## 3. Trigger

Причинно-следственная связь подтверждена диагностикой. Инцидент был вызван **совокупностью двух факторов**:

1. **Сетевой конфликт из-за клона ВМ.** В той же сети была запущена перенесённая копия ВМ `<VM_NAME>` с идентичными MAC-адресом, IP (`<INTERNAL_IP>`) и hostname. Возник конфликт ARP/IP: пакеты, адресованные `<INTERNAL_IP>`, попадали то на оригинал, то на копию. Оригинал «терял» сеть, NPM не мог достучаться до ONLYOFFICE.
2. **Повреждение данных RabbitMQ в контейнере.** Контейнер ONLYOFFICE был запущен без именованных volume (`pedantic_carson` - авто-имя), всё его состояние - включая базу RabbitMQ - лежало внутри контейнера. После жёсткого выключения ВМ (перенос, остановка хоста) база RabbitMQ повредилась. При следующем старте `rabbitmq-server` не поднимался, и контейнер бесконечно висел на строке `Waiting for connection to the localhost host on port 5672`.

Оба фактора наложились друг на друга и маскировали симптоматику: казалось, что «сломался ONLYOFFICE», хотя на самом деле сломались и сеть, и состояние контейнера.

---

## 4. Detection

Проблема обнаружена при попытке открыть `.docx` в Nextcloud. Симптомы:

- ошибка «Приложение ONLYOFFICE недоступно. Пожалуйста, свяжитесь с администратором»;
- `curl -I http://localhost:8888` → `Recv failure: Connection reset by peer` (TCP-соединение устанавливается, но HTTP-ответ не приходит);
- `curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN>` → таймаут или `502 Bad Gateway`;
- в логах контейнера - бесконечный цикл `Waiting for connection to the localhost host on port 5672`.

---

## 5. Timeline

**T0. Исходное состояние.**
Nextcloud работал на `<VM_NAME>` (Ubuntu Server 24.04, Hyper-V). ONLYOFFICE Docs был развёрнут в Docker, опубликован через Nginx Proxy Manager по адресу `https://<OFFICE_SUBDOMAIN>.<DOMAIN>`. JWT отключён (`JWT_ENABLED=false`). Всё работало.

**T1. Перенос ВМ и запуск клона в той же сети.**
Была создана и запущена копия ВМ с идентичными MAC/IP/hostname. Возник сетевой конфликт. Оригинал начал проявлять нестабильность: сервисы то отвечали, то нет.

**T2. Первичная диагностика.**
Обнаружено, что контейнер ONLYOFFICE запущен (`docker ps` - `Up`), но приложение внутри не работает:
- `curl localhost:8888` → `Connection reset by peer`;
- `docker logs` → бесконечное ожидание `5672`.

**T3. Проверка JWT.**
`docker exec <container_name> documentserver-jwt-status.sh` → `JWT enabled - false`. Гипотеза «сломался JWT» отпала.

**T4. Проверка NPM и Nextcloud.**
Прокси-хост в NPM указывал на корректный IP. Коннектор в Nextcloud - на корректный адрес, ключ пустой (так как JWT отключён). Проблема была не в NPM и не в коннекторе.

**T5. Выявление сетевого конфликта.**
Сопоставление фактов: клон ВМ запущен в той же сети, у него тот же MAC/IP. Это объяснило «плавающую» нестабильность оригинала.

**T6. Устранение сетевого конфликта.**
Клон ВМ выключен вручную через Hyper-V. Автозапуск клона отключён (`Set-VM -AutomaticStartAction Nothing`), чтобы после ребута хоста он не поднялся снова.

**T7. Пересоздание контейнера ONLYOFFICE.**
Старый контейнер `pedantic_carson` остановлен и удалён. Запущен новый контейнер с именем `onlyoffice`, фиксированными параметрами и отключённым JWT:

```bash
sudo docker run -i -t -d --name onlyoffice \
  -p 8888:80 --restart=always \
  -e JWT_ENABLED=false \
  onlyoffice/documentserver
```

> **Техдолг, зафиксированный по итогам.** В этой команде образ указан **без тега** (`onlyoffice/documentserver` = `latest`). Это осознанно оставлено как есть: на момент восстановления приоритетом было поднять сервис, а не фиксировать версию. Фактическая версия определена **постфактум** через логи: `9.1.0, Build 168`. В Lessons Learned и Prevention это отмечено как то, что нужно закрыть: при следующем пересоздании контейнера зафиксировать тег `9.1.0`, чтобы `docker pull` не привёз другую версию.

**T8. Проверка запуска.**
В логах нового контейнера:
- `Starting PostgreSQL 16 database server [ OK ]`;
- `Starting RabbitMQ Messaging Server rabbitmq-server [ OK ]`;
- `Connection to localhost (::1) 5672 port [tcp/amqp] succeeded!`;
- `ds:docservice: started`, `ds:converter: started`.

Сервис полностью поднялся.

**T9. Финальная проверка цепочки.**

```bash
curl -I http://localhost:8888
# HTTP/1.1 308 Permanent Redirect → /welcome/  (штатное поведение)

curl -s http://localhost:8888/healthcheck
# true

curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN>
# HTTP/2 308 → /welcome/  (NPM проксирует корректно)
```

**T10. Проверка в Nextcloud.**
Пересохранение настроек коннектора (Администрирование → ONLYOFFICE), открытие `.docx` - редактор загружается, копирование-вставка и форматирование работают.

---

## 6. Root Cause

Подтверждённая причина - **две независимые проблемы, наложившиеся друг на друга**:

1. **Сетевой конфликт (ARP/IP collision) из-за клона ВМ.** Клон с идентичными MAC/IP/hostname в той же сети создавал «мигающий» сетевой доступ к оригиналу. Это объясняет, почему оригинал «сломался», хотя его не трогали.
2. **Повреждение базы RabbitMQ внутри контейнера.** Контейнер был запущен без именованных volume. После жёсткого выключения ВМ данные RabbitMQ (`/var/lib/rabbitmq/mnesia`) повредились, и `rabbitmq-server` перестал стартовать. Контейнер висел на `Waiting for 5672`, frontend-nginx отвечал `Connection reset by peer`, NPM отдавал `502`, Nextcloud показывал «Приложение недоступно».

Оба фактора устранены: клон выключен, контейнер пересоздан с чистого образа.

---

## 7. What Didn’t Work

| Действие | Результат |
|---|---|
| Проверка JWT (`documentserver-jwt-status.sh`) | JWT уже был отключён, гипотеза отпала |
| Проверка прокси-хоста в NPM | Настройки корректны, проблема не в NPM |
| Пересохранение коннектора в Nextcloud | Не помогло, так как сам ONLYOFFICE не отвечал |
| `docker start` старого контейнера `pedantic_carson` | Контейнер запускался, но висел на `Waiting for 5672` |
| Проверка логов старого контейнера | Показала застревание на RabbitMQ - стало понятно, что нужно пересоздание |

---

## 8. Resolution

Рабочее решение:

1. Выключить клон ВМ `<VM_NAME>` в Hyper-V и отключить его автозапуск:
   ```powershell
   Stop-VM -Name "<VM_NAME_CLONE>" -Force
   Set-VM -Name "<VM_NAME_CLONE>" -AutomaticStartAction Nothing
   ```
2. Остановить и удалить повреждённый контейнер:
   ```bash
   sudo docker stop pedantic_carson
   sudo docker rm pedantic_carson
   ```
3. Запустить новый контейнер ONLYOFFICE с фиксированным именем и параметрами:
   ```bash
   sudo docker run -i -t -d --name onlyoffice \
     -p 8888:80 --restart=always \
     -e JWT_ENABLED=false \
     onlyoffice/documentserver:9.1.0
   ```
4. Дождаться полного старта (1–3 минуты), проверить логи:
   ```bash
   sudo docker logs -f onlyoffice
   ```
   Убедиться, что появились `RabbitMQ ... [ OK ]`, `ds:docservice: started`, `ds:converter: started`.
5. Проверить цепочку:
   ```bash
   curl -I http://localhost:8888              # 308 → /welcome/
   curl -s http://localhost:8888/healthcheck   # true
   curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN> # 308 → /welcome/
   ```
6. В Nextcloud → Администрирование → ONLYOFFICE: адрес `https://<OFFICE_SUBDOMAIN>.<DOMAIN>`, секретный ключ пустой (JWT отключён). Сохранить.
7. Открыть `.docx` - убедиться, что редактор работает.
8. Сделать checkpoint/экспорт ВМ.

---

## 9. Result

После восстановления:

- онлайн-офис работает полностью;
- `.docx` открывается корректно;
- копирование-вставка и форматирование работают;
- Nextcloud остался стабильным;
- сетевой конфликт устранён, автозапуск клона отключён;
- создан checkpoint ВМ, фиксирующий рабочее состояние;
- фактическая версия ONLYOFFICE Docs определена и зафиксирована в документации: **9.1.0, Build 168**.

---

## 10. Lessons Learned

- **Клон ВМ нельзя запускать в той же сети без смены MAC/IP/hostname.** Это классический источник «мистических» сбоев, при которых оригинал «ломается», хотя его не трогали. Если нужно запустить копию - либо в изолированной сети (Internal/Host-only), либо с предварительной сменой идентификаторов.
- **Контейнеры без именованных volume уязвимы к жёсткому выключению.** База RabbitMQ, PostgreSQL и внутреннее состояние контейнера живут внутри слоя контейнера. При резком выключении ВМ они повреждаются. Лечение - пересоздание контейнера; профилактика - volume или корректный `docker stop` перед выключением.
- **Безымянные контейнеры (`pedantic_carson`-подобные) - это техдолг.** `--name onlyoffice` делает управление предсказуемым, а `docker start onlyoffice` возвращает контейнер с ожидаемыми параметрами.
- **Фиксация версии образа обязательна.** `onlyoffice/documentserver:latest` при пересоздании может привезти другую версию. Фиксируйте тег (`onlyoffice/documentserver:<version>`). В этом инциденте тег не был зафиксирован, версия определена постфактум по логам: **9.1.0, Build 168**.
- **`JWT_ENABLED=false` приемлемо для домашней лаборатории, но не для публичного доступа.** Если сервис открыт в интернет - включайте JWT и храните секрет в менеджере секретов.

---

## 11. Prevention

- Перед запуском клона ВМ: сменить MAC, IP, hostname; либо запустить в изолированной сети.
- Отключить автозапуск клонов ВМ в Hyper-V (`AutomaticStartAction = Nothing`).
- Для ONLYOFFICE использовать именованные volume или как минимум корректно останавливать контейнер (`docker stop onlyoffice`) перед выключением ВМ.
- Зафиксировать версию образа ONLYOFFICE в команде запуска (`onlyoffice/documentserver:9.1.0`).
- Делать checkpoint Hyper-V перед изменениями ресурсов, обновлениями и переносом ВМ на тестовый сервер.
- После успешного восстановления - экспорт ВМ без контрольных точек.
- Включить JWT, если сервис станет публичным.

---

## 12. Appendix: команды диагностики и восстановления

```bash
# 1. Проверить, запущен ли контейнер
sudo docker ps | grep onlyoffice

# 2. Посмотреть логи (искать зависание на 5672)
sudo docker logs --tail 80 <container_name>

# 3. Проверить JWT
sudo docker exec <container_name> documentserver-jwt-status.sh

# 4. Определить фактическую версию и сборку (постфактум)
sudo docker logs onlyoffice 2>&1 | grep -iE "Version|Build"
# Пример вывода: Version: 9.1.0. Build: 168

# 5. Проверить доступность изнутри
curl -I http://localhost:8888
curl -s http://localhost:8888/healthcheck

# 6. Проверить доступность снаружи (через NPM)
curl -I https://<OFFICE_SUBDOMAIN>.<DOMAIN>

# 7. Пересоздать контейнер (с фиксированной версией)
sudo docker stop <container_name>
sudo docker rm <container_name>
sudo docker run -i -t -d --name onlyoffice \
  -p 8888:80 --restart=always \
  -e JWT_ENABLED=false \
  onlyoffice/documentserver:9.1.0

# 8. Дождаться старта и проверить логи
sudo docker logs -f onlyoffice
```

**NPM (proxy host):**

- Domain: `<OFFICE_SUBDOMAIN>.<DOMAIN>`
- Scheme: `http`
- Forward Hostname / IP: `<INTERNAL_IP>`
- Forward Port: `8888`
- Websockets Support: **ON**
- SSL: Let's Encrypt + Force SSL

**Nextcloud (Администрирование → ONLYOFFICE):**

- Адрес ONLYOFFICE Docs: `https://<OFFICE_SUBDOMAIN>.<DOMAIN>`
- Секретный ключ: пусто (при `JWT_ENABLED=false`)
