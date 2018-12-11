# ozyab09_microservices
ozyab09 microservices repository

### Homework 14 (docker-3)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-3)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices )

Работа в папке src:
- `post-py` - сервис отвечающий за написание постов
- `comment` - сервис отвечающий за написание комментариев
- `ui` - веб-интерфейс, работающий с другими сервисами
* Сборка образов:
```
docker build -t ozyab/post:1.0 ./post-py
docker build -t ozyab/comment:1.0 ./comment
docker build -t ozyab/ui:1.0 ./ui
```
* Отдельная bridge-сеть для контейнеров, так как сетевые алиасы не работают в сети по умолчанию:
`docker network create reddit`

* Запуск контейнеров в этой сети с сетевыми алиасами контейнеров:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
* Остановка всех контейнеров:
`docker kill $(docker ps -q)`
* Создание Docker volume:
`docker volume create reddit_db`
* Запуск контейнеров с docker volume:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:2.0
```
* После перезапуска информация остается в базе

### Homework 13 (docker-2)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices )

* Работа с `docker-machine`:

`docker-machine create <имя>` - создание docker-хоста

`eval $(docker-machine env <имя>)` - перемеключение на docker-хост

`eval $(docker-machine env --unset)` - переключение на локальный docker

`docker-machine rm <имя>` - удаление docker-хоста

* Создание `docker-host`:
```
docker-machine create --driver google \
  --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
  --google-machine-type n1-standard-1 \
  --google-zone europe-west1-b \
  docker-host
```
* После этого увидеть созданный `docker-host` можно, выполнив: `docker-machine ls`
* Запустим `htop` в `docker`'е:
`docker run --rm -ti tehbilly/htop`

Будет виден только процесс `htop`.

Если выполнить `docker run --rm --pid host -ti tehbilly/htop`, то видны будут все процессы на хостовой машине

* Добавлены: 

`Dockerfile` - текстовое описание нашего образа

`mongod.conf` - подготовленный конфиг для `mongodb`

`db_config` - переменная окружения со ссылкой на `mongodb`

`start.sh` - скрипт запуска приложения

* Сборка образа: `docker build -t reddit:latest .`
* Запуск контейнера: `docker run --name reddit -d --network=host reddit:latest`
* Создание правило на входящий порт 9292:
```
gcloud compute firewall-rules create reddit-app \
  --allow tcp:9292 \
  --target-tags=docker-machine \
  --description="Allow PUMA connections" \
  --direction=INGRESS 
```
* Команды по работе с образом:

`docker tag reddit:latest <login>/otus-reddit:1.0` - добавить тэг образу `reddit`

`docker push <login>/otus-reddit:1.0` - отправка образа в registry

`docker logs reddit -f` - просмотр логов

`docker inspect <login>/otus-reddit:1.0` - просмотр информации об образе

`docker inspect <login>/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'` - просмотр только определенной информации о контейнере

`docker diff reddit` - просмотр изменений, произошедних в файловой системе запущенного контейнера



### Homework 12 (docker-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices )
* Добавлен файл шаблона PR `.github/PULL_REQUEST_TEMPLATE`
* Интеграция со slack выполняется командой `/github subscribe Otus-DevOps-2018-09/ozyab09_microservices`
* Запуск контейнера: `docker run hello-world`
* Список запущенных контейнеров: `docker ps`
* Список всех контейнеров: `docker ps -a`
* Создание и запуск контейнера: `docker run -it ubuntu:16.04 /bin/bash`
* Команда `run` создает и запускает контейнер из `image`
* `start` запускает остановленный  созданный ранее контейнер
* `attach` подсоединяет терминал к созданному контейнеру
* `docker system df` отображает сколько дискового пространства занято образами, контейнерами и `volume`'ами
* Создание образа из контейнера: `docker commit <container_id> username/ubuntu-tmp-file`
