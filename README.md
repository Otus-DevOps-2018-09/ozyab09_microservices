# ozyab09_microservices
ozyab09 microservices repository



### Homework 17 (gitlab-ci-2)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=gitlab-ci-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Создание нового проекта <i>example2</i>

* Добавление проекта в <i>username_microservices</i>
```
git checkout -b gitlab-ci-2
git remote add gitlab2 http://vm-ip/homework/example2.git
git push gitlab2 gitlab-ci-2
```

* <b>Dev-окружение</b>: Изменение пайплайна таким образом, чтобы <i>job deploy</i> стал определением окружения <i>dev</i>, на которое условно будет выкатываться каждое изменение в коде проекта:
1. Переименуем <i>deploy stage</i> в <i>review</i>
2. <i>deploy_job</i> заменим на <i>deploy_dev_job</i>
3. Добавим <i>environment</i>
```yaml
name: dev
url: http://dev.example.com
```
В разделе <i>Operations</i> - <i>Environment</i> появится окружение <i>dev</i>

* Два новых этапа: <i>stage</i> и <i>production</i>. <i>Stage</i> будет содержать <i>job</i>, имитирующий выкатку на <i>staging</i> окружение, <i>production</i> - на <i>production</i> окружение. <i>Job</i> будут запускаться с кнопки

* Директива <i>only</i> описывает список условий, которые должны быть истинны, чтобы <i>job</i> мог запуститься. Регулярное выражение  `/^\d+\.\d+\.\d+/` означает, что должен стоять <i>semver</i> тэг в <i>git<i>, например, <i>2.4.10</i>

* Пометка текущего коммита тэгом:
```git tag 2.4.10```





### Homework 16 (gitlab-ci-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=gitlab-ci-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Установка <i>Docker</i>:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install docker-ce docker-compose
```

* Подготовка окружения:
```
mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
cd /srv/gitlab/
touch docker-compose.yml
```
<i>docker-compose.yml</i>:
```
web:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  hostname: 'gitlab.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://<VM-IP>'
  ports:
    - '80:80'
    - '443:443'
    - '2222:22'
  volumes:
    - '/srv/gitlab/config:/etc/gitlab'
    - '/srv/gitlab/logs:/var/log/gitlab'
    - '/srv/gitlab/data:/var/opt/gitlab'
```
* Запуск <i>Gitlab CI</i>: `docker-compose up -d`

* <i>GUI GitLab</i>: Отключение регистрации, создание группы проектов <i>homework</i>, создание проекта <i>example</i>

* Добавление <i>remote</i> в проект <i>microservices</i>:
`git remote add gitlab http://<ip>/homework/example.git`

* <i>Push</i> в репозиторий:
`http://35.204.52.154/homework/example`

* Определение <i>CI/CD Pipeline</i> проекта производится в файле <i>.gitlab-ci.yml</i>:
```
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo 'Building'

test_unit_job:
  stage: test
  script:
    - echo 'Testing 1'

test_integration_job:
  stage: test
  script:
    - echo 'Testing 2'

deploy_job:
  stage: deploy
  script:
    - echo 'Deploy'
```

* Установка GitLab Runner:
```
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest
```

* Запуск <i>runner</i>'а:
`docker exec -it gitlab-runner gitlab-runner register`

* Добавление исходного кода в репозиторий:
```
git clone https://github.com/express42/reddit.git && rm -rf ./reddit/.git
git add reddit/
git commit -m 'Add reddit app'
git push gitlab gitlab-ci-1
```

* Изменение описания пайплайна в <i>.gitlab-ci.yml</i>:
```
image: ruby:2.4.2
stages:
...
variables:
  DATABASE_URL: 'mongodb://mongo/user_posts'
before_script:
  - cd reddit
  - bundle install
...
test_unit_job:
  stage: test
  services:
    - mongo:latest
  script:
    - ruby simpletest.rb
...
```

* В пайплайне выше добавлен вызов <i>reddit/simpletest.rb</i>:
```
require_relative './app'
require 'test/unit'
require 'rack/test'

set :environment, :test

class MyAppTest < Test::Unit::TestCase
  include Rack::Test::Methods

  def app
    Sinatra::Application
  end

  def test_get_request
    get '/'
    assert last_response.ok?
  end
end
```

* Добавление библиотеки для тестирования в <i>reddit/Gemfile</i>: `gem 'rack-test'`

### Homework 15 (docker-4)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-4)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Подключение к `docker-host`:
`eval $(docker-machine env docker-host)`

* Запуск контейнера `joffotron/docker-net-tools` с набором сетевых утилит:
`docker run -ti --rm --network none joffotron/docker-net-tools -c ifconfig`

Использован `none-driver`, вывод работы контейнера:
```
lo Link encap:Local Loopback
   inet addr:127.0.0.1  Mask:255.0.0.0
   UP LOOPBACK RUNNING  MTU:65536  Metric:1
   RX packets:0 errors:0 dropped:0 overruns:0 frame:0
   TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
   collisions:0 txqueuelen:1000
   RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
* Запуск контейнера в сетевом пространстве `docker`-хоста:
`docker run -ti --rm --network host joffotron/docker-net-tools -c ifconfig`

Запуск `ipconfig` на `docker-host`'е приведет к аналогичному выводу:
`docker-machine ssh docker-host ifconfig`

* Запуст nginx в фоне в сетевом пространстве docker-host:
`docker run --network host -d nginx`

При повторном выполнении команды получим ошибку:
```
2018/11/30 19:50:53 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```
По причине того, что порт 80 уже занят.

* Для просмотра существующих net-namespaces необходимо выполнить на docker-host:
`sudo ln -s /var/run/docker/netns /var/run/netns`
Просмотр:
`sudo ip netns`
- При запуске контейнера с сетью `host` net-namespace один - default.
- При запуске контейнера с сетью `none` в спсике добавится id net-namespace.
Вывод списка `net-namespac`'ов:
```
user@docker-host:~$ sudo ip net
88f8a9be77ca
default
```
Можно выполнить команду в выбранном `net-namespace`:
```
user@docker-host:~$ sudo ip netns exec 88f8a9be77ca ifconfig
lo  Link encap:Local Loopback  
  inet addr:127.0.0.1  Mask:255.0.0.0
  UP LOOPBACK RUNNING  MTU:65536  Metric:1
  RX packets:0 errors:0 dropped:0 overruns:0 frame:0
  TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
  collisions:0 txqueuelen:1000 
  RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

* Создание `bridge`-сети:
`docker network create reddit --driver bridge`

* Запуск проекта `reddit` с использованием `bridge`-сети:
```
docker run -d --network=reddit mongo:latest
docker run -d --network=reddit ozyab/post:1.0
docker run -d --network=reddit ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
В данной конфигурации `web`-сервис `puma` не сможет подключиться к БД `mongodb`.

Сервисы ссылаются друг на друга по `dns`-именам, прописанным в `ENV`-переменных `Dockerfil`'а. Встроенный `DNS docker`'а ничего не знает об этих именах.

Присвоение контейнерам имен или сетевых алиасов при старте:
```
--name <name> (max 1 имя)
--network-alias <alias-name> (1 или более)
```

* Запуск контейнеров с сетевыми алиасами:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
* Запуск проекта в 2-х `bridge` сетях. Сервис `ui` не имеет доступа к базе данных.

Создание `docker`-сетей:
```
docker network create back_net --subnet=10.0.2.0/24
docker network create front_net --subnet=10.0.1.0/24
```
Запуск контейнеров:
```
docker run -d --network=front_net -p 9292:9292 --name ui ozyab/ui:1.0
docker run -d --network=back_net --name comment ozyab/comment:1.0
docker run -d --network=back_net --name post ozyab/post:1.0
docker run -d --network=back_net --name mongo_db  --network-alias=post_db --network-alias=comment_db mongo:latest 
```
Docker при инициализации контейнера может подключить к нему только 1 сеть, поэтому контейнеры `comment` и `post` не видят контейнер `ui` из соседних сетей. 

Нужно поместить контейнеры post и comment в обе сети. Дополнительные сети подключаются командой: `docker network connect <network> <container>`:
```
docker network connect front_net post
docker network connect front_net comment 
```
Установка пакета `bridge-utils`:
```
docker-machine ssh docker-host
sudo apt-get update && sudo apt-get install bridge-utils
```
Выполнив `docker network ls` можно увидеть список виртуальных сетей `docker`'а.
`ifconfig | grep br` покажет список `bridge`-интерфейсов:
```
br-45935d0f2bbf Link encap:Ethernet  HWaddr 02:42:6d:5a:8b:7e
br-45bbc0c70de1 Link encap:Ethernet  HWaddr 02:42:94:69:ab:35
br-b6342f9c65f2 Link encap:Ethernet  HWaddr 02:42:9a:b1:73:d9
```
Можно просмотреть информацию о каждом `bridge`-интерфейсе командой `brctl show <interface>`:
```
docker-user@docker-host:~$brctl show br-45935d0f2bbf
bridge name      bridge id          STP enabled   interfaces
br-45935d0f2bbf  8000.02426d5a8b7e  no            veth05b2946
                                                  veth2f50985
                                                  vetha882d28
```
_veth-интерфейс_ - часть виртуальной пары интерфейсов, которая лежат в сетевом пространстве хоста и
также отображаются в ifconfig. Вторые часть виртуального интерфейса находится внутри контейнеров.

Просмотр `iptables`: `sudo iptables -nL -t nat`
```
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  10.0.1.0/24          0.0.0.0/0
MASQUERADE  all  --  10.0.2.0/24          0.0.0.0/0
MASQUERADE  tcp  --  10.0.1.2             10.0.1.2             tcp dpt:9292
```
Первые правила отвечают за выпуск трафика из контейнеров наружу.
```
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.1.2:9292
```
Последняя строка прокидывает порт 9292 внутрь контейнера.

## Docker-compose
* Файлу `./src/docker-compose.yml` требуется переменная окружения `USERNAME`: `export USERNAME=ozyab`

Можно выполнить: `docker-compose up -d`

* Изменить `docker-compose` под кейс с множеством сетей, сетевых алиасов 
* Файл `.env` - переменные для `docker-compose.yml`
* Базовое имя создается по имени папки, в которой происходит запуск `docker-compose`.

Для задания базового имени проекта необходимо добавить переменную `COMPOSE_PROJECT_NAME=dockermicroservices`

### Homework 14 (docker-3)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-3)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

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
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

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
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)
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
