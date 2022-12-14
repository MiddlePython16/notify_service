# Сервис Notifications

## Описание сервиса

![notification_service](https://www.plantuml.com/plantuml/png/hLPBRzj64BxpLso476GWCWzjBqLIn9PaLwWbEPPT7ucLk9O8aro2ktHW10KQ6saGj4Y0145pokCGvAeODkBMiVKNbl-eCyk7gIVSehfmkJExyyqttmnrNqWQoiXp3SEuvVWzDx8P6KWPYEzDCwGFb_kE74JTIB2mntt9VBdSuvuPkDJ7KnKXRTVfcjLlAvkujTFSC3qg90jXowh22UhuB5mET8HRDjk3LrAh_zoejbkT6mIIKK3jy9lhW0vOAtlWKH51w4WcidWsEq2RoAEijLlRAxlrJhjP_ry3wJtwa0dkiSft1Tgoj4oRIjgbWYLfJzk3GpdW_ZnhRn32pVEi7lVxtEvEcE-wlC-5kK_tzkaFrzi52anCMAI5a8vq6L4Vr6lrNhrKRzG7qcflDzOQrPNj1aGnZ4rmvdwPsfxZto6b5Nmwe1N9mnAyjxXqzfmUbGxtYNOgBHb6Nk-oNx1Rscu5r6TkTXNUiHPg-crS0umpDLGNwbnzKSFuDtKUlo1g15kNO81j0ijJzHNibs1x71_5luBVaRh29ucFrGdudQehSBc05JYfl-3YsqdjxfKIKX5avn2gw5chFF9j6XwEGmMEG4LiCiDqkxH47Xa4VdCI5Rq1ioENXN2-awqnlIgagjDyfR6TQV1faaDrWda0SCpdF7uAviVgRrWzGGRGznpCfnCcM5ofp_7lU0PINLg4YmOE4h_PqBI3YjEbUgSlFrDV9wuZGBZU-G9nXvhDinkGdEXw4akjEuucwmua2UH4-monD57NjLKDwoFoeok_A78ESXGVWnv04y3aSVmyVfPmbo2_X9oU546c3PIYJDJm868H6qUvHRjTHpOVt2bbguGnNH34CeHo31C0oDop0YpO7C7cXTRg4GeuVhwuGWBwGCjcbuVxYMOs4p8nw5kuPAGvEO7g3_zp9Q2TQI0JUQQA7W_q3b43Xqzpwh7pC3qC_XcwxKW3A79Bi6V9JpXv4ky9iEg497AR1lVBT3ToET3uslZC-axaZPUubsaa1wXeU8pLiHprUx2_iruTt_RuWSE4bKspfdubA-dSoc4aCj135Bpg1_1j8AiXJmkYKopceHslHqgPUTHn2oEaZkyaOnhyZ2tcPYDrmOJS4P0nFFL72yx1AQksOAJEwFKPLymO2VFQKlq9rHlfEVl7pGxdtJBhiBKrTqpEW9wUsPdT87EZVjADfaNxBsue9bsumPn88YcS5bscZXaSQV4ji8-hSBuZ8CyRDVUQpc_ZscisZTE5opYOL2esAEgpysDxrrhRQCE_WDQwjL5VhxTgPJ3SXtFmn-4V "notification_service")

Сервис позволяет создавать email-рассылки. Сервис спроектирован с учётом потенциального расширения количества способов
связи (SMS, Push и др.). Источниками уведомлений являются:

- панель администратора,
- генератор автоматических событий,
- другие сервисы онлайн кинотеатра (Auth, UGC, Movies Admin).

Все события по созданию уведомлений поступают в API. Модуль API обеспечивает CRUD для шаблонов уведомлений и самих
уведомлений в хранилище уведомлений (MongoDB).

События поступающие на **endpoint** `add_notification` передаются через очередь сообщений (RabbitMQ) процессу, который
отправляет уведомление (воркер).

Очередь поддерживает до 10 уровней приоритета соощебний, что позволяет отправлять мгновенные сообщения с максимальной
задержкой в несколько минут.

События из других сервисов собираются из брокера событий (Kafka) при помощи модуля Kafka-worker.

Систему можно масштабировать путем увеличения количества воркеров.

## Сбор данных для темплейта

Сбор данных прописан в template, пример:

```JSON
{
  "subject": "pass",
  "body": "{{password}}",
  "template_type": "email",
  "fields": [
    {
      "name": "user",
      "url": "http://nginx_auth:80/api/v1/users/,start,user_id,end,",
      "body": {},
      "headers": {},
      "fetch_pattern": "model:dotwiz",
      "http_type": "get"
    },
    {
      "name": "password",
      "url": "http://nginx_auth:80/api/v1/users/,start,user_id,end,/password",
      "body": {},
      "headers": {},
      "fetch_pattern": "key:password",
      "http_type": "get"
    }
  ]
}
```

При сборе данных есть определенный дополнительный json, который изначально выглядит как {"user_id": "..."}, чтобы было
от чего оттолкнуться в сборе данных.

Данные из этого JSONа можно подставлять в любые места, за исключением изначальных ключей fieldsItem. Для подставления
нужно вставить в строку данных паттерн:  ,start,{key:str},end, .

Как видно в примере выше, таким образом подставляется user_id для получения полной информации по пользователю(в полях
url).

Данные, что будут собраны, будут пополнять дополнительный JSON, и могут быть использованы в последующих полях для
подставления

Для получения результата из http ответа, существует поле fetch_pattern в данный момент оно поддерживает 3 паттерна

- key:{key_name:str}
- index:{index_value:int}
- model:{choices=[dotwiz]}

Паттерны можно применять последовательно, если перечислить их через ", "

Пример: "fetch_pattern":"key:user, model:dotwiz"

В таком случае, к ответу сначала будет применена выборка по ключу user, и после чего json, что лежал под данным ключем
будет передан в dotwiz(dict, что поддерживает обращения через точку к ключам)

В случае, если вы использовали dotwiz fetch_pattern и хотите использовать его данные в подстановке, то вы можете
обращаться к ключам через . , а к индексам через [{value:int}]. Например:

Вы сверху собрали поле user, и использовали fetch_pattern='...model:dotwiz',предположим что ваш user выглядел так:

```JSON
{
  "id": "1",
  "email": "user@mail.ru",
  "login": "first_user",
  "friends": [
    {
      "id": "2",
      "email": "friend@mail.ru",
      "login": "second_user"
    }
  ]
}
```

Тогда, в полях ниже, вы можете подставить почту друга: ,start,user.friends[0].email,end,

## Стек технологий

- Docker
- FastAPI
- Filebeat
- Kafka
- MongoDB
- Nginx
- Python
- RabbitMQ
- Zookeeper

## Установка

1. Склонировать репозиторий

```sh
git clone git@github.com:Wiped-Out/yandex_16_team_work.git
```

2. Перед запуском необходимо создать .env файлы по примеру .env.sample файлов. Для этого можно воспользоваться
   скриптом [rename_script.py](../rename_script.py).

## Запуск

1. Перейти в дирректорию `notify_compose`

```sh
cd notify_compose
```

2. Создать сеть

```sh
docker network create services_network
```

3. Файл [docker-compose.yml](/notify_compose/docker-compose.yaml) обеспечивает сборку окружения для работы сервиса
   авторизации

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:5.4.9
    env_file:
      - ./zookeeper/.env
    #    ports:
    #      - 22181:2181
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_logs:/var/lib/zookeeper/log
    networks:
      - backend

  notify_kafka_pipeline:
    image: confluentinc/cp-kafka:5.4.9
    depends_on:
      - zookeeper
    volumes:
      - kafka_data:/var/lib/kafka/data
      - kafka_logs:/var/lib/kafka/logs
    #    ports:
    #      - 29092:29092
    env_file:
      - ./kafka/.env
    networks:
      - backend
      - services_network

  kafka_worker:
    build:
      context: ./kafka_worker
    depends_on:
      - mongodb
      - rabbitmq
    networks:
      - backend
      - internal_bridge

  rabbitmq:
    image: rabbitmq:3.10.7-management
    env_file:
      - ./rabbitmq/.env
    #    ports:
    #      - 5672:5672
    #      - 15672:15672
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq/
    networks:
      - backend

  fastapi_notify:
    build:
      context: ./fastapi
    volumes:
      - fastapi_static:/app/src/static
    depends_on:
      - mongodb
      - rabbitmq
    networks:
      - internal_bridge
      - backend
      - services_network

  nginx_notify:
    image: nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - fastapi_static:/data/fastapi_static
      - nginx_logs:/var/log/nginx/
    depends_on:
      - fastapi_notify
    ports:
      - 83:83
    restart: always
    networks:
      - internal_bridge
      - services_network

  filebeat_notify:
    build:
      context: ./filebeat
    volumes:
      - nginx_logs:/var/log/nginx:ro
    env_file:
      - ./filebeat/.env
    depends_on:
      - fastapi_notify
      - nginx_notify
    networks:
      - services_network

  mongodb:
    image: mongo:6.0.1
    restart: unless-stopped
    volumes:
      # optional to preserve database after container is deleted.
      - mongodb_data:/data/db
    env_file:
      - ./mongodb/.env
    #    ports:
    #      - 27017:27017
    networks:
      - backend

  worker1:
    build:
      context: ./worker
    env_file:
      - ./worker/.env
    depends_on:
      - mongodb
      - rabbitmq
      - fastapi_notify
    networks:
      - backend
      - services_network


# Указываем Docker, какие именованные тома потребуются сервисам
volumes:
  fastapi_static:
  nginx_logs:
  kafka_data:
  kafka_logs:
  zookeeper_data:
  zookeeper_logs:
  mongodb_data:
  rabbitmq_data:


networks:
  backend:
  internal_bridge:
  services_network:
    external: true
```
