# 🛡️ Ops & Security Homelab (IaC & Observability)

Воспроизводимая инженерная платформа управления, мониторинга и безопасности
домашней инфраструктуры на базе **Debian 13 (Trixie)**, **Docker Compose**
и **Ansible**.

Проект развивает архитектурную концепцию, начатую в
[Office-infra](https://github.com/Iso-Hallow/Office-infra).
Если Office-infra - это прикладной изолированный контур для пользователей
(User Plane), то данный репозиторий реализует **контур управления и
наблюдаемости (Management & SecOps Plane)**.

---

## 🚀 Ключевые архитектурные решения

- **Периметральная защита (Zero Exposed Backend Ports).** Ни один сервисный
  порт не смотрит в локальную сеть напрямую. Вся маршрутизация замкнута на
  Reverse Proxy (Nginx Proxy Manager), а сервисы изолированы во внутренней
  bridge-сети Docker.
- **Локальный центр сертификации (PKI).** Выпуск собственного
  Wildcard-сертификата `*.lab` через OpenSSL для безопасной работы менеджера
  паролей по HTTPS (соответствие требованиям Web Crypto API).
- **Наблюдаемость (Observability).** Сбор метрик операционной системы и
  контейнеров в реальном времени через связку
  **Node Exporter + Prometheus + Grafana**.
- **Инфраструктура как код (IaC).** Первоначальный харденинг ОС (UFW,
  Fail2Ban, NTP-синхронизация) и доставка конфигураций автоматизированы
  через **Ansible**.
- **Контроль безопасности в CI/CD.** Автоматический аудит в GitHub Actions:
  сканирование на утечки секретов (**Gitleaks**) и статический анализ
  уязвимостей конфигураций (**Trivy**).

---

## 📦 Состав инженерного стека

| Сервис | Назначение | Внутренний порт | Домен в сети |
| :--- | :--- | :--- | :--- |
| **Nginx Proxy Manager** | Обратный прокси, TLS-терминация | 80, 443, 81 | `http://<SERVER_IP>:81` |
| **Grafana** | Визуализация метрик и дашборды | 3000 | `http://grafana.<HOMELAB_DOMAIN>` |
| **Prometheus** | TSDB, сбор временных рядов | 9090 | *Изолирован в Docker-сети* |
| **Node Exporter** | Телеметрия ОС хоста (CPU, RAM, диск) | 9100 | *Изолирован в Docker-сети* |
| **Vaultwarden** | Менеджер учётных данных (HTTPS) | 80 | `https://vault.<HOMELAB_DOMAIN>` |
| **Dashy** | Инженерный дашборд быстрого доступа | 8080 | `http://dashy.<HOMELAB_DOMAIN>` |
| **DokuWiki** | Внутренняя база знаний и регламентов | 80 | `http://wiki.<HOMELAB_DOMAIN>` |
| **IT-Tools** | Набор утилит системного инженера | 80 | `http://tools.<HOMELAB_DOMAIN>` |
| **CyberChef** | Анализ данных, декодирование, криптография | 8000 | `http://chef.<HOMELAB_DOMAIN>` |

> `<HOMELAB_DOMAIN>` по умолчанию - `lab` (например, `grafana.lab`).
> Для работы доменов нужен локальный DNS (Pi-hole, dnsmasq) или записи
> в `hosts` на клиентских машинах.

---

## 🛠️ Развёртывание через Ansible (IaC)

1. Укажите IP-адрес целевого сервера в инвентаре:

   ```bash
   cp ansible/inventory/hosts.example ansible/inventory/hosts
   ```
2. Запустите автоматический сценарий харденинга и настройки:
   ```bash 
   ansible-playbook -i ansible/inventory/hosts ansible/playbooks/deploy_homelab.yml
   ```
---

## 🏃 Быстрый ручной запуск (Docker Compose)

1. Склонируйте репозиторий:
   ```bash 
   git clone https://github.com/Iso-Hallow/Hub-Leo-Homelab.git
   cd Hub-Leo-Homelab
   ``` 
2. Подготовьте переменные окружения:
   ```bash 
   cp .env.example .env
   ```
3. Запустите стек:
   ```bash 
   docker compose up -d
   ```
   
 ---
   
## 🔐 Безопасность
Модель угроз: домашняя лаборатория в изолированном сегменте сети.
Публичного доступа из интернета нет. Все сервисы доступны только через
Reverse Proxy с TLS-терминацией.

Backend-порты не публикуются в локальную сеть.

Wildcard-сертификат ```*.lab``` выпущен локальным CA (OpenSSL).

Секреты хранятся в ```.env``` (файл не коммитится в Git, в репозитории
только .env.example).

Версии Docker-образов зафиксированы.

CI/CD проверяет утечки секретов (Gitleaks) и уязвимости (Trivy).

При необходимости можно перейти на менеджер секретов (Vaultwarden уже
в стеке; для CI/CD - GitHub Secrets или HashiCorp Vault).

---

## 📚 Документация и регламенты

* [docs/runbooks.md](docs/runbooks.md) - **Инженерная аптечка:** решение проблем рассинхронизации времени (Time Drift в Hyper-V), обхода ограничений Web Crypto API и устранения ошибок 502 в реверс-прокси.

## 📄 Лицензия
MIT License
