# hw_3

## Один контейнер

```sh
docker build -t hw3-counter .
docker run -d --name hw_3 -p 127.0.0.1:8080:8000 -e GREETING=Privet hw3-counter
```

Проверка:

```sh
curl -fsS -w '\n' http://127.0.0.1:8080/
curl -fsS -w '\n' -X POST http://127.0.0.1:8080/increment
docker logs hw_3 > hw_3.txt 2>&1
```

Проверено: приветствие `Privet`, счётчик вырос с `0` до `1`. Логи в [hw_3.txt](hw_3.txt).

## Docker Compose с Redis

```sh
cp .env.example .env
docker compose up -d --build --wait
```

В `.env` меняются `APP_PORT` и `GREETING`. Приложение слушает порт `8000` внутри контейнера и подключается к `redis://redis:6379/0` по имени сервиса. На компьютере открыт только порт приложения. Redis пишет журнал AOF в том `redis_data`, подключённый к `/data`.

Redis проверяется через `redis-cli ping`, приложение — через `GET /readyz`. Перед запуском приложения Compose ждёт готовности Redis.

В командах ниже используется порт `8081` из `.env.example`.

### Сохранение счётчика

```sh
curl -fsS -w '\n' http://127.0.0.1:8081/
curl -fsS -w '\n' -X POST http://127.0.0.1:8081/increment
curl -fsS -w '\n' -X POST http://127.0.0.1:8081/increment
docker compose logs --no-color app redis >> hw_3.txt 2>&1
docker compose down
docker compose up -d --wait
curl -fsS -w '\n' http://127.0.0.1:8081/
```

Проверено: `app` и `redis` стали `Healthy`. После двух `POST` счётчик вырос с `0` до `2`. После `down` и `up -d --wait` он остался `2`, а `hostname` сменился с `b5165a64f958` на `a5166e548f26`. Команда `down` без `-v` сохранила том.

### Отключение и восстановление Redis

```sh
docker compose stop redis
curl -sS -w '\nHTTP %{http_code}\n' http://127.0.0.1:8081/healthz
curl -sS -w '\nHTTP %{http_code}\n' http://127.0.0.1:8081/readyz
curl -sS -w '\nHTTP %{http_code}\n' -X POST http://127.0.0.1:8081/increment
docker compose start redis
curl -fsS --retry 8 --retry-delay 1 -w '\n' http://127.0.0.1:8081/readyz
curl -fsS -w '\n' http://127.0.0.1:8081/
curl -fsS -w '\n' -X POST http://127.0.0.1:8081/increment
docker compose logs --no-color app redis >> hw_3.txt 2>&1
```

При остановленном Redis `/healthz` вернул `200`, а `/readyz` и `POST /increment` вернули `503`. После запуска Redis `/readyz` снова вернул `ready`, `GET /` вернул `2`, а следующий `POST /increment` — `3`. `hostname` приложения остался `a5166e548f26`.

На этапе с SQLite уже проверено, что после пересоздания контейнера счётчик остался `1`, а `hostname` сменился.
