# ДЗ по Docker

Команды запускаются из корня проекта в Bash. На Windows подойдёт терминал Ubuntu в WSL 2 с доступом к Docker.

## 4 балла. Один контейнер

```bash
docker build -t hw3-app .
docker run -d --rm --name hw3-single \
  -p 127.0.0.1:8080:8000 -e 'GREETING=Привет из Docker' hw3-app

url=http://127.0.0.1:8080
curl --retry 5 --retry-connrefused -fsS "$url/"
curl -fsS -X POST "$url/increment"
curl -fsS "$url/"
docker logs hw3-single
docker stop hw3-single
```

В новом контейнере счётчик начинается с `0`. После `POST /increment` следующий `GET /` показывает `1`.

## 6 баллов. Compose и SQLite

При первой настройке скопируй `.env.example` в `.env`. В `.env` можно задать `GREETING` и `HOST_PORT`. Порт приложения внутри контейнера — `8000`.

```bash
cp .env.example .env
url=http://127.0.0.1:8080
```

Если меняешь `HOST_PORT`, укажи тот же порт в `url`. Контейнер хранит SQLite в `/data/counter.db`. Каталог `/data` подключён к именованному тому `sqlite-data`.

```bash
docker compose -f compose.sqlite.yaml up -d --build
curl --retry 5 --retry-connrefused -fsS "$url/"
curl -fsS -X POST "$url/increment"
docker compose -f compose.sqlite.yaml down
docker compose -f compose.sqlite.yaml up -d
curl --retry 5 --retry-connrefused -fsS "$url/"
docker compose -f compose.sqlite.yaml down
```

Последний `GET /` возвращает тот же `count`, что и `POST /increment` до удаления контейнера.

## 8 баллов. Compose и Redis

```bash
docker compose up -d --build --wait
docker compose ps
curl -fsS "$url/"
curl -fsS -X POST "$url/increment"
curl -fsS -X POST "$url/increment"
```

`app` подключается к `redis:6379` через сеть Compose. Наружу опубликован порт `app`. Redis сохраняет данные в AOF в томе `redis-data`. Оба сервиса имеют healthcheck.

Запиши текущее значение счётчика `N`. Обычный `down` сохраняет том. Флаг `-v` удаляет его.

```bash
docker compose down
docker compose up -d --wait
curl -fsS "$url/"

docker compose stop redis
curl -i "$url/healthz"
curl -i "$url/readyz"
curl -i -X POST "$url/increment"

docker compose start redis
docker compose ps
```

После пересоздания `GET /` показывает `N`. Пока Redis остановлен, статусы трёх запросов по порядку — `200`, `503`, `503`.

Дождись состояния `healthy` у обоих сервисов после запуска Redis. Затем проверь счётчик.

```bash
curl -fsS "$url/"
curl -fsS -X POST "$url/increment"
docker compose down
```

После восстановления `GET /` показывает `N`, а следующий `POST /increment` возвращает `N + 1`.

Результаты проверок и логи есть в [отчёте](REPORT.md).
