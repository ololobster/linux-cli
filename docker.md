Docker и ко
===========

1. [Введение в Docker](#введение-в-docker)
1. [Docker CLI](#docker-cli)
1. [Внутреннее устройство реестра Docker](#внутреннее-устройство-реестра-docker)

# Введение в Docker

Docker — это ПО для автоматизации развёртывания и управления приложениями.
Позволяет «упаковать» приложение со всем его окружением и зависимостями в контейнер.
Приложение везде ведёт себя одинаково,
т.к. окружение и зависимости поставляются вместе с ним в контейнере.
Контейнеры менее прожорливы по сравнению с виртульными машинами,
т.к. используют ядро ОС, на которой запущены.

Контейнер создаётся из образа (image), который содержит все исполняемые файлы, библиотеки,
конфиги и т.д., необходимые для запуска приложения.
Из одного образа можно сделать несколько контейнеров.
Образ состоит из неизменяемых слоёв для того,
чтобы один и тот же слой может использоваться несколькими образами.

![](img/docker/image_layers.png)

`Dockerfile` — это инструкция для создания образа.
Реестр Docker (Docker registry) — это сервер, на котором хранятся образы Docker.

Для каждого контейнера создаётся доп. изменеяемый слой (container layer) поверх слоёв образа,
чтобы приложение могло изменять свою ФС.

Том Docker (volume) — это хранилище информации, которое можно расшарить с окружающим миром.

Демон Docker (`dockerd`) занимается высокоуровневым управленимем контейнерами.
Утилита `docker` — это клиент, отсылающий ему команды.
Среда выполения (container runtime) работает на более низком уровне,
непосредственно занимается изоляцией процессов, запускаемых в контейнерах.
Примеры: `containerd`, `runc`.

OCI (Open Container Initiative) — это проект, определяющий открытые стандарты для образов и сред выполнения.
CRI (Container Runtime Interface) — это интерфейс, определяющий взаимодействие Kubernetes и среды выполнения.

# Docker CLI

## Манипуляции с образами

Вывести список локально доступных образов:
```
$ docker images
```

Скачать образ из реестра:
```
$ docker pull ⟨image⟩
```

Собрать свой образ (в каталоге должен быть файл `Dockerfile`):
```
$ docker build --tag=⟨new image's name & tag⟩ ⟨directory⟩
```

Удалить образ:
```
$ docker rmi ⟨image⟩
```

## Манипуляции с контейнерами

Создать, но не запускать контейнер:
```
$ docker create --name ⟨name⟩ ⟨image⟩
```

Создать и запустить контейнер:
```
$ docker run -d --name ⟨name⟩ ⟨image⟩
```

По умолчанию сетевой стек контейнера изолирован.
Создать и запустить контейнер, пробросив порт контейнера 8081 на локальный порт 9000:
```
$ docker run -d -p 9000:8081 ⟨image⟩
```

Создать и запустить контейнер, передав ему переменную окружения `VAR`:
```
$ docker run -d -e VAR=VALUE ⟨image⟩
```
Примечания:
1. Аргументов `-e` может быть несколько.
1. Можно передать уже существующую переменную окружения: `-e VAR`.

Создать и запустить контейнер, передав ему переменные окружения из файла `in.env`:
```
$ docker run -d --env-file in.env ⟨image⟩
```

Остановить/запустить/удалить контейнер:
```
$ docker stop ⟨id or name⟩
$ docker start ⟨id or name⟩
$ docker rm ⟨id or name⟩
```

Запустить консоль контейнера:
```
# docker exec -it ⟨container id⟩ bash
```

Способы задать runtime-переменные для Docker-контейнера:
- ключ `-e`:
  ```
  docker run -e VARIABLE=VALUE ...
  ```
- ключ `--env-file`:
  ```
  docker run --env-file=FILE ...
  ```
- секция `environment` в `docker-compose.yml`.

Манипуляции томами:
```
$ docker volume ls
$ docker volume rm ⟨volume⟩
```

Манипуляции сетями:
```
$ docker network ls
$ docker network rm traefic_network
```

# Внутреннее устройство реестра Docker

Манифест — это JSON-документ, описывающий образ.
Стандартный манифест (тип `application/vnd.docker.distribution.manifest.v2+json`) описывает слои и конфигурационный блоб.
Пример:
```
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "config": {
       "mediaType": "application/vnd.docker.container.image.v1+json",
       "size": 1402,
       "digest": "sha256:a7d85984bee3d822fd3b3c25454575645b3a512240130a7c03b87528f8e42fce"
    },
    "layers": [
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 3799689,
            "digest": "sha256:9824c27679d3b27c5e1cb00a73adb6f4f8d556994111c12db3c5d61a0c843df8"
        },
        ...
    ]
}
```

Ид блоб — это его контрольная сумма sha256, ид манифеста — аналогично.
Манифесты и блоб являются неизменяемыми, т.к. правки ведут к изменению контрольной суммы.

Один тег может содержать образы для разных архитектур.
Такому тегу соответсвует манифест типа `application/vnd.docker.distribution.manifest.list.v2+json` ака fat manifest.
Пример:
```
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
    "manifests": [
        {
            "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
            "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
            "size": 7143,
            "platform": {"architecture": "ppc64le", "os": "linux"}
        },
        ...
    ]
}
```

Манифест можно скачать как по тегу, так и по хешу манифеста:
```
GET /v2/⟨img⟩/manifests/⟨tag or hash⟩"
```

Для заливки именованного манифеста:
```
PUT /v2/⟨img⟩/manifests/⟨tag⟩
```
Этот вызов выполняется после того, как загружены все слои и конфигурационные blobы.

# Docker Compose CLI

Вывести конфиг, подставив все переменные:
```
$ docker-compose config
```

Собрать образы для сервисов, использующих кастомные образы:
```
$ docker compose build
```
Примечания:
1. `--build-arg ⟨var⟩=⟨value⟩` позволяет задать build-time переменную.
   Аргументов `--build-arg` может быть много.
   Альтернатива — секция `services.⟨service⟩.build.args` в `docker-compose.yml`.

Собрать образы и запустить контейнеры:
```
$ docker compose up
```
Примечания:
1. `--no-build` повелевает не собирать уже имеющиеся кастомные образы.

Запустить контейнеры в фоновом режиме:
```
$ docker compose up -d
```

Остановить и удалить контейнеры, удалить сети:
```
$ docker compose down
```

Остановить и удалить контейнеры, удалить сети, удалить тома:
```
$ docker compose down -v
```

Остановить и удалить контейнеры, удалить сети, удалить тома и образы:
```
$ docker compose down -v --rmi all --remove-orphans
```
