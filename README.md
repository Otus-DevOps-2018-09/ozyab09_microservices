# ozyab09_microservices
ozyab09 microservices repository


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
