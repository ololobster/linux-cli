Docker и ко
===========

1. [Введение в Docker](#введение-в-docker)
1. [Docker CLI](#docker-cli)
1. [Внутреннее устройство реестра Docker](#внутреннее-устройство-реестра-docker)
1. [Docker Compose CLI](#docker-compose-cli)

# Введение в Docker

Docker — это ПО для автоматизации развёртывания и управления приложениями.
Позволяет «упаковать» приложение со всем его окружением и зависимостями в контейнер.
Приложение везде ведёт себя одинаково,
т.к. окружение и зависимости поставляются вместе с ним в контейнере.
Контейнеры менее прожорливы по сравнению с виртуальными машинами,
т.к. используют ядро ОС, на которой запущены.

Контейнер создаётся из образа (image), который содержит все исполняемые файлы, библиотеки,
конфиги и т.д., необходимые для запуска приложения.
Из одного образа можно сделать несколько контейнеров.
Образ состоит из неизменяемых слоёв для того,
чтобы один и тот же слой может использоваться несколькими образами.

![](img/docker/image_layers.png)

`Dockerfile` — это инструкция для создания образа.
Реестр Docker (Docker registry) — это сервер, на котором хранятся образы Docker.
Docker Hub (hub.docker.com) — это главный реестр.

Для каждого контейнера создаётся доп. изменеяемый слой (container layer) поверх слоёв образа,
чтобы приложение могло изменять свою ФС.

Том Docker (volume) — это хранилище информации, жизненный цикл которого не зависит от контейнеров.

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
Стандартный манифест (тип `application/vnd.oci.image.manifest.v1+json`) описывает слои образа и конфигурационный блоб.
Пример манифеста образа `keycloak/keycloak` под архитектуру amd64:
```
$ MANIFEST_ID=sha256:3e58dc82a9b72f364f37dac17f99fcaf53d0052a3c1ad7f00b069a1ed43201c0
$ curl -H "Authorization: Bearer ${TOKEN}" \
    "https://registry-1.docker.io/v2/keycloak/keycloak/manifests/${MANIFEST_ID}"
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "config": {
        "mediaType": "application/vnd.oci.image.config.v1+json",
        "digest": "sha256:9b0455f766d5032eafc3626dc956869b35cecf6db539bbe00086c0f6b15c04d8",
        "size": 10147
    },
    "layers": [
        {
            "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
            "digest": "sha256:0041f61dcf11a311c2ecd8ab448db7104c6b7c8d8eecd1f1224ebc36f4e6e673",
            "size": 7256675
        },
        {
            "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
            "digest": "sha256:498b6355eb8859e8d894611ef477987472bd66c8135d5dc0af198065b837904d",
            "size": 84367352
        },
        ...
    ]
}
```

Ид блоб — это его контрольная сумма sha256, ид манифеста — аналогично.
Манифесты и блоб являются неизменяемыми, т.к. правки ведут к изменению контрольной суммы.

Один тег может содержать образы для разных архитектур.
Такому тегу соответсвует манифест типа `application/vnd.oci.image.index.v1+json` ака fat manifest.
Пример манифеста тега `26.6.2-2` проекта `keycloak/keycloak`:
```
$ curl -H "Authorization: Bearer ${TOKEN}" \
    https://registry-1.docker.io/v2/keycloak/keycloak/manifests/26.6.2-2
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.index.v1+json",
    "manifests": [
        {
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "digest": "sha256:3e58dc82a9b72f364f37dac17f99fcaf53d0052a3c1ad7f00b069a1ed43201c0",
            "size": 1057,
            "platform": {"architecture": "amd64", "os": "linux"}
        },
        {
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "digest": "sha256:4f4f4d52e12845cd142322285ca311aa6edc75cef7284b3382c97a9e97faa360",
            "size": 1057,
            "platform": {"architecture": "arm64", "os": "linux"}
        },
        ...
    ]
}
```

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
