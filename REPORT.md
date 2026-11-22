# Отчёт по проверкам

Проверки на 4, 6 и 8 баллов выполнялись последовательно из одного скрипта в Ubuntu WSL 2. Docker Engine 29.5.2, Docker Compose v5.8.0.

## 4 балла. Один контейнер

```bash
docker build -t hw3-app .
docker run -d --rm --name hw3-check-single -p 127.0.0.1:18080:8000 \
  -e 'GREETING=Привет из Docker' hw3-app
curl -fsS --max-time 5 http://127.0.0.1:18080/
curl -fsS --max-time 5 -X POST http://127.0.0.1:18080/increment
curl -fsS --max-time 5 http://127.0.0.1:18080/
docker logs hw3-check-single
docker stop -t 2 hw3-check-single
```

Запросы вернули `count` со значениями `0`, `1`, `1` и приветствие `Привет из Docker`.

```text
2026-11-22 19:32:13,039 listening port=8000 storage=sqlite
2026-11-22 19:32:13,512 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-22 19:32:13,520 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-22 19:32:13,555 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-22 19:32:13,576 http 172.17.0.1 "GET / HTTP/1.1" 200 -
```

## 6 баллов. Compose и SQLite

```bash
export HOST_PORT=18081 GREETING='Привет из Compose'
docker compose --env-file .env.example -p hw3-check-sqlite -f compose.sqlite.yaml up -d --build
curl -fsS --max-time 5 http://127.0.0.1:18081/
curl -fsS --max-time 5 -X POST http://127.0.0.1:18081/increment
curl -fsS --max-time 5 -X POST http://127.0.0.1:18081/increment
docker compose --env-file .env.example -p hw3-check-sqlite -f compose.sqlite.yaml down --timeout 3
docker volume inspect hw3-check-sqlite_sqlite-data --format '{{.Name}}'
docker compose --env-file .env.example -p hw3-check-sqlite -f compose.sqlite.yaml up -d
curl -fsS --max-time 5 http://127.0.0.1:18081/
docker compose --env-file .env.example -p hw3-check-sqlite -f compose.sqlite.yaml down --timeout 3
```

Два POST подняли счётчик с `0` до `2`. После `down` том `hw3-check-sqlite_sqlite-data` остался. Ниже ответы перед удалением и после создания нового контейнера.

```json
{"greeting": "Привет из Compose", "count": 2, "hostname": "63f832d6702c", "storage": "sqlite"}
{"greeting": "Привет из Compose", "count": 2, "hostname": "f2003cea5e18", "storage": "sqlite"}
```

## 8 баллов. Compose и Redis

```bash
export HOST_PORT=18082
unset GREETING
docker compose --env-file .env.example -p hw3-check-redis up -d --build --wait --wait-timeout 45
docker compose --env-file .env.example -p hw3-check-redis ps
docker compose --env-file .env.example -p hw3-check-redis exec -T redis redis-cli CONFIG GET appendonly
curl -fsS --max-time 5 http://127.0.0.1:18082/
curl -fsS --max-time 5 -X POST http://127.0.0.1:18082/increment
curl -fsS --max-time 5 -X POST http://127.0.0.1:18082/increment
curl -fsS --max-time 5 -X POST http://127.0.0.1:18082/increment
docker compose --env-file .env.example -p hw3-check-redis down --timeout 3
docker compose --env-file .env.example -p hw3-check-redis up -d --wait --wait-timeout 45
curl -fsS --max-time 5 http://127.0.0.1:18082/
```

Оба сервиса вышли в `healthy`. У `app` опубликован `127.0.0.1:18082->8000/tcp`. У Redis `PortBindings` равен `{}`, `appendonly` равен `yes`. Три POST вернули `1`, `2` и `3`. После `down` том Redis сохранился и `GET /` вернул `3`.

```bash
docker compose --env-file .env.example -p hw3-check-redis stop --timeout 3 redis
curl -isS --max-time 5 http://127.0.0.1:18082/healthz
curl -isS --max-time 5 http://127.0.0.1:18082/readyz
curl -isS --max-time 5 -X POST http://127.0.0.1:18082/increment
docker compose --env-file .env.example -p hw3-check-redis start redis
```

| Запрос при остановленном Redis | Статус | Ответ |
| --- | --- | --- |
| `GET /healthz` | `200` | `{"status": "alive"}` |
| `GET /readyz` | `503` | `{"error": "storage_unavailable"}` |
| `POST /increment` | `503` | `{"error": "storage_unavailable"}` |

После запуска Redis `/readyz` снова вернул `200`.

```bash
curl -fsS --max-time 5 http://127.0.0.1:18082/
curl -fsS --max-time 5 -X POST http://127.0.0.1:18082/increment
```

```json
{"greeting": "Привет из Docker", "count": 3, "hostname": "4aad8be76024", "storage": "redis"}
{"greeting": "Привет из Docker", "count": 4, "hostname": "4aad8be76024", "storage": "redis"}
```

ID контейнера `app` и время его запуска совпали до остановки Redis и после восстановления. Оба сервиса снова стали `healthy`.
