---
layout: post
title: "Деплоим с докером без простоя"
description: "Как деплоить docker контейнеры без даунтайма"
category: blog
blog: true
tags: ["docker", "deploy", "downtime", "rails"]
lang: ru
---

## Деплоим в прод с докером

В классическом случае сервер приложений имеет возможность перезапускаться без простоя, например это умеет unicorn, puma. Но что делать если у вас используется докер?
Допустим мы подняли наш контейнер с приложением версии 1.0 и теперь хотим выкатить версию 2.0. Старый контейнер остановится и поднимется новый, но пока новая версия приложения будет запускаться сервер будет отдавать какой-то из 50x статусов, например 502 если наш
сервер приложений стоит за nginx.

### Решение

Предположим что мы используем docker-compose с таким конфигом:

    version: '2'

    services:

      nginx:
        image: jwilder/nginx-proxy
        links:
          - app
        ports:
          - 80:80
          - 443:443
        volumes:
          - /var/run/docker.sock:/tmp/docker.sock:ro
          - /opt/docker/my_app/nginx/proxy.conf:/etc/nginx/proxy.conf
          - /opt/docker/my_app/nginx/vhost.d:/etc/nginx/vhost.d
        volumes_from:
          - app:ro
        restart: always

      app:
        image: my_app:1.0
        env_file:
          - .env
        environment:
          - VIRTUAL_HOST=api.my-app.com,admin.my-app.com
          - VIRTUAL_PORT=3000
        restart: always
        expose:
          - '3000'
        command: bundle exec rails s -p 3000 -b 0.0.0.0


Основная особенность конфига это использование образа [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy),
он позволяет поднять контейнер с nginx который будет автоматически обновлять конфиг nginx'а при запуске/остановке ссылающихся контейнеров.
Вот так будет выглядеть автоматически сгенеренный конфиг nginx:

    upstream admin.my-app.com {
        server 172.18.0.6:3000;
    }
    server {
        server_name admin.my-app.com;
        listen 80 ;
        access_log /var/log/nginx/access.log vhost;
        include /etc/nginx/vhost.d/admin.my-app.com;
        location / {
            proxy_pass http://admin.my-app.com;
        }
    }

#### Как теперь поднимать новые версии контейнеров?

Ок. Балансировщик у нас есть. Теперь мы хотим выкатить новую версию нашего аппа. В конфиге docker-compose обновится тег используемого образа нашего приложения:

    app:
      image: my_app:2.0

Но docker-compose в данный момент не предоставляет готовой команды для деплоя
контейнера. В issues можно найти обсуждение где предлагается подход с использованием `docker-compose up --scale` командой которая позволяет поднять несколько копий контейнера.
В коменте [https://github.com/docker/compose/issues/1786#issuecomment-294434786](https://github.com/docker/compose/issues/1786#issuecomment-294434786)
можно увидеть описание workflow как это все может работать.  Воспользуемся этим подходом.

Приведем изначальный вывод команды `docker ps`:

    CONTAINER ID IMAGE               COMMAND                 CREATED         STATUS        PORTS               NAMES
    fed0f242012d my_app:1.0          "docker-entrypoint.s…"  10 seconds ago  Up 10 seconds 3000/tcp            my_app_app_1
    d61ec99796b3 jwilder/nginx-proxy "/app/docker-entrypo…"  10 seconds ago  Up 10 seconds 0.0.0.0:80->80/tcp  my_app_nginx_1

Теперь наши шаги:

1. `docker-compose up --scale app=2 -d app` - поднимаем еще один контейнер app.

Как теперь выглядит вывод `docker ps`:

    CONTAINER ID  IMAGE               COMMAND                 CREATED         STATUS         PORTS               NAMES
    fed0f242012d  my_app:1.0          "docker-entrypoint.s…"  20 seconds ago  Up 20 seconds  3000/tcp            my_app_app_1
    16088324c0d2  my_app:2.0          "docker-entrypoint.s…"  10 seconds ago  Up 10 seconds  3000/tcp            my_app_app_2
    d61ec99796b3  jwilder/nginx-proxy "/app/docker-entrypo…"  20 seconds ago  Up 20 seconds  0.0.0.0:80->80/tcp  my_app_nginx_1

При этом в конфиге nginx в upstream автоматом добавился наш новый контейнер и запросы обрабатываются двумя версиями приложения:

    upstream admin.my-app.com {
        server 172.18.0.6:3000;
        server 172.18.0.7:3000;
    }

2. Ждем пока контейнер `my_app_app_2` поднимется. Можно поставить тупой `sleep` или пинговать curl'ом.

3. Останавливаем наш старый контейнер с версией 1.0 и убиваем его:

```
docker stop my_app_app_1
docker rm my_app_app_1
```

Остается контейнер с версией 2.0, upstream в nginx опять же обновляется.
Вывод `docker ps`:

    CONTAINER ID  IMAGE               COMMAND                 CREATED        STATUS        PORTS               NAMES
    16088324c0d2  my_app:2.0          "docker-entrypoint.s…"  20 seconds ago Up 20 seconds 3000/tcp            my_app_app_2
    d61ec99796b3  jwilder/nginx-proxy "/app/docker-entrypo…"  30 seconds ago Up 30 seconds 0.0.0.0:80->80/tcp  my_app_nginx_1


Данный процесс удобно оформить в bash скрипт:

    #!/usr/bin/env bash

    CONTAINER_NAME="my_app_"
    # номер в конце названия контейнера инкрементится при запуске нового и надо запомнить номер первого контейнера (со старой версией нашего аппа)
    FIRST_NUM=`docker ps | awk '{print $NF}' | grep "$CONTAINER_NAME[0-9]*$" | awk -F  "_" '{print $NF}' | sort | head -1`
    NUM_OF_CONTAINERS=1
    MAX_NUM_OF_CONTAINERS=2

    # если контейнера с аппом еще нет, то просто запустим его
    if [[ $FIRST_NUM = "" ]]; then
      docker-compose up --build -d app
      exit 0
    fi

    # скейлим
    docker-compose up --scale app=$MAX_NUM_OF_CONTAINERS --no-recreate -d app

    sleep 5

    {% raw %}
    # найдем номер нашего нового контейнера и его ip адрес
    NEW_CONTAINER_NUM=`sudo docker ps | awk '{print $NF}' | grep "$CONTAINER_NAME[0-9]*$" | awk -F  "_" '{print $NF}' | sort | tail -1`
    NEW_CONTAINER_IP=`sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $CONTAINER_NAME$NEW_CONTAINER_NUM`
    {% endraw %}

    while true
    do
      sleep 1
      STATUS_CODE=`curl -s -o /dev/null -w "%{http_code}" http://$NEW_CONTAINER_IP:3000/hc`
      echo $STATUS_CODE
      # ждем пока поднимется новый контейнер
      if [[ $STATUS_CODE = 200 ]]; then
        # убиваем старый(е)
        for ((i=$FIRST_NUM;i<$NUM_OF_CONTAINERS+$FIRST_NUM;i++))
        do
          docker stop $CONTAINER_NAME$i
          docker rm $CONTAINER_NAME$i
          exit 0
        done
      fi
    done


#### Итог

В итоге имеем деплой нашего приложения без простоя. Проверено, работает стабильно. В целом следует придерживаться правила что обе версии приложения должны функционировать с вашей базой данных или сервисом очередей. Ну это как везде.
