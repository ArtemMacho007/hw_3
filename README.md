# hw_3

Запуск:

```sh
cp .env.example .env
docker compose up -d --build --wait
```

В `.env` можно поменять `APP_PORT` (8081) и `GREETING` (`PrivetCompose`). Приложение подключается к `redis://redis:6379/0`. Данные Redis лежат в томе `redis_data`.

Проверка счётчика:

```sh
url=http://127.0.0.1:8081
curl -fsS -w '\n' -X POST "$url/increment"
curl -fsS -w '\n' -X POST "$url/increment"
docker compose down
docker compose up -d --wait
curl -fsS -w '\n' "$url/"
```

Два `POST` дали `1` и `2`. После пересоздания контейнеров осталось `2`.

Остановка и запуск Redis:

```sh
docker compose stop redis
curl -sS -w '\nHTTP %{http_code}\n' "$url/healthz"
curl -sS -w '\nHTTP %{http_code}\n' "$url/readyz"
curl -sS -w '\nHTTP %{http_code}\n' -X POST "$url/increment"
docker compose start redis
curl -fsS --retry 8 --retry-delay 1 -w '\n' "$url/readyz"
curl -fsS -w '\n' "$url/"
curl -fsS -w '\n' -X POST "$url/increment"
```

При остановленном Redis ответы были `200`, `503`, `503`. После запуска Redis `GET` вернул `2`, а `POST` — `3`. Приложение не перезапускал.

Отдельный контейнер: счётчик `0 → 1`. В Compose с SQLite после пересоздания счётчик остался `1`. [Логи](hw_3.txt).
