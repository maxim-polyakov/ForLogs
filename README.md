# Сбор логов и метрик

Стек запускает:

- Grafana — интерфейс и дашборды;
- Loki — хранение логов;
- Grafana Alloy — автоматический сбор логов Docker-контейнеров;
- Prometheus — сбор и хранение метрик.

## Запуск

Нужен Docker Desktop с Linux-контейнерами.

```powershell
Copy-Item .env.example .env
```

Замените `GRAFANA_ADMIN_PASSWORD` в `.env`, затем запустите:

```powershell
docker compose up -d
```

Интерфейсы:

- Grafana: http://localhost:3020
- Prometheus: http://localhost:9090
- Loki: http://localhost:3100/ready
- Alloy: http://localhost:12345

Источники Loki и Prometheus создаются в Grafana автоматически. Для просмотра
логов откройте **Explore → Loki** и выполните запрос:

```logql
{compose_service=~".+"}
```

Можно отфильтровать конкретный сервис:

```logql
{compose_project="my-project", compose_service="api"}
```

## Метрики проектов

Добавьте endpoint приложения в `prometheus/prometheus.yml`:

```yaml
  - job_name: my-api
    static_configs:
      - targets: ["host.docker.internal:8080"]
```

После изменения примените конфигурацию:

```powershell
docker compose restart prometheus
```

## Деплой через GitHub Actions

Workflow `.github/workflows/deploy.yml` запускается при push в `main` или вручную
через **Actions → Deploy monitoring stack → Run workflow**.

Создайте GitHub Environment с именем `production`, затем добавьте в него
секреты:

- `SSH_HOST` — адрес сервера;
- `SSH_USER` — пользователь с доступом к Docker;
- `SSH_PRIVATE_KEY` — приватный SSH-ключ;
- `DEPLOY_PATH` — каталог на сервере, например `/home/baxic/forlogs`;
- `GRAFANA_ADMIN_PASSWORD` — пароль администратора Grafana.

Доступные GitHub Variables и их значения по умолчанию:

- `SSH_PORT`: `22`;
- `GRAFANA_ADMIN_USER`: `admin`;
- `PUBLIC_BIND_ADDRESS`: `0.0.0.0`;
- `GRAFANA_PORT`: `3020`;
- `LOKI_PORT`: `3100`;
- `ALLOY_PORT`: `12345`;
- `PROMETHEUS_PORT`: `9090`;
- `PROMETHEUS_RETENTION`: `15d`;
- `GRAFANA_VERSION`, `LOKI_VERSION`, `ALLOY_VERSION`,
  `PROMETHEUS_VERSION`: `latest`.

На сервере должны быть установлены Docker и Docker Compose, а SSH-пользователь
должен иметь право выполнять `docker compose` без `sudo`.

## Важно

Loki и Prometheus в этой конфигурации не имеют авторизации. Не открывайте порты
`3100`, `9090` и `12345` в интернет. Для сбора данных с других серверов следует
поставить Alloy на каждом сервере и закрыть центральные сервисы через HTTPS и
авторизацию в reverse proxy.

Остановка:

```powershell
docker compose down
```

Удаление вместе со всеми сохранёнными логами, метриками и настройками Grafana:

```powershell
docker compose down -v
```
