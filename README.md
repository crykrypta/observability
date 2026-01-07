# Observability Stack (Grafana + Loki + Promtail)

Общий observability-стек для сервера:
**логи всех Docker-контейнеров → Loki → Grafana**.

Стек разворачивается **один раз на сервере** и используется всеми сервисами, которые пишут логи в `stdout`.

---

## Состав

- **Grafana** — UI для просмотра логов
- **Loki** — хранилище логов
- **Promtail** — сбор логов из Docker контейнеров

---

## Архитектура

Docker container stdout
↓
Promtail
↓
Loki
↓
Grafana

Приложения:
- ничего не знают про Loki / Grafana
- просто пишут логи в stdout (JSON / structlog — идеально)

---

## Требования

- Linux server
- Docker
- Docker Compose v2
- Доступ к `/var/lib/docker/containers` и `docker.sock`

---

## Быстрый старт

```bash
git clone <repo-url>
cd observability
docker compose up -d
```

⸻

## Структура репозитория

```
observability/
├── docker-compose.yml
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── grafana/
├── .env
└── README.md
```

⸻

## Как собираются логи

### Promtail:
	•	использует Docker service discovery
	•	читает *-json.log контейнеров
	•	парсит JSON-логи приложений
	•	добавляет labels с низкой кардинальностью
	•	отправляет данные в Loki

⸻

### Используемые labels:
	•	container — имя контейнера
	•	service — имя сервиса из docker compose
	•	level — уровень логирования
	•	status — статус операции (если есть)
	•	model — модель LLM (если есть)


Для контейнеров, запущенных через docker run, используйте:

```
{container="container-name"}
```

⸻

## Timestamp и старые логи

В Loki разрешён приём старых логов (например, при первом запуске):

reject_old_samples_max_age: 8760h

Retention при этом всё равно ограничивает хранение логов.

⸻

Retention
	•	Loki хранит логи 7 дней
	•	старые данные автоматически удаляются compactor’ом

⸻

## Безопасный доступ к Grafana (ОБЯЗАТЕЛЬНО)

Grafana не открыта в интернет.
Она слушает только 127.0.0.1 на сервере.

Доступ осуществляется через SSH tunnel.

⸻

### Подключение к Grafana

Вариант 1. Обычный способ (одноразовая команда)

На локальной машине:
```zsh
ssh -L 3000:127.0.0.1:3000 avet@<SERVER_IP>
```
Затем открыть в браузере:
```zsh
http://localhost:3000
```
Туннель живёт, пока открыта SSH-сессия.

⸻

Вариант 2. Удобный способ (через SSH config) ⭐ РЕКОМЕНДУЕТСЯ

На локальной машине добавить в `~/.ssh/config`:
```
Host observability
  HostName <SERVER_IP>
  User avet
  LocalForward 3000 127.0.0.1:3000
```
После этого достаточно выполнить:

```
ssh observability
```

И Grafana будет доступна по адресу:
```
http://localhost:3000
```
