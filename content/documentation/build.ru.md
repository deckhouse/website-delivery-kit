---
title: Сборка
weight: 40
menuTitle: Deckhouse Delivery Kit
---

{{< alert level="warning" >}}
Функциональность Deckhouse Delivery Kit доступна только если у вас есть лицензия на любую коммерческую версию Deckhouse Kubernetes Platform.
{{< /alert >}}

## Основы

Доставка приложения в Kubernetes предполагает его контейнеризацию (сборку одного или нескольких образов) для последующего развёртывания в кластере.

Для сборки пользователю необходимо описать сборочные инструкции в виде Dockerfile'а, а остальное Deckhouse Delivery Kit возьмёт на себя:

* оркестрация одновременной/параллельной сборки образов приложения;
* кроссплатформенная и мультиплатформенная сборка образов;
* общий кеш промежуточных слоёв и образов в container registry, доступный с любых раннеров;
* оптимальная схема тегирования, основанная на содержимом образа, предотвращающая лишние пересборки и время простоя приложения при выкате;
* система обеспечения воспроизводимости и неизменности образов для коммита: однажды собранные образы для коммита более не будут пересобраны.

Парадигма сборки и публикации образов в Deckhouse Delivery Kit отличается от парадигмы, предлагаемой сборщиком Docker, в котором есть несколько команд: `build`, `tag` и `push`. Deckhouse Delivery Kit собирает, тегирует и публикует образы в один шаг. Данная особенность связана с тем, что Deckhouse Delivery Kit по сути не собирает образы, а синхронизирует текущее состояние приложения (для текущего коммита) с container registry, дособирая недостающие слои образов и синхронизируя работу параллельных сборщиков.

В общем случае сборка образов с Deckhouse Delivery Kit предполагает наличие container registry (`--repo`), поскольку образ не только собирается, но и сразу публикуется. При этом ручной вызов команды `d8 dk build` не требуется, так как сборка выполняется автоматически при запуске всех верхнеуровневых команд Deckhouse Delivery Kit, в которых требуются образы (например, `d8 dk converge`).

Однако ручной запуск команды `d8 dk build` может быть полезным во время локальной разработки. В этом случае сборку можно запускать отдельно и без участия container registry.

## Образы и зависимости

### Добавление образов

Для сборки c Deckhouse Delivery Kit необходимо добавить описание образов в `werf.yaml` проекта. Каждый образ добавляется директивой `image` с указанием имени образа:

```yaml
project: example
configVersion: 1
---
image: frontend
# ...
---
image: backend
# ...
---
image: database
# ...
```

> Имя образа — это уникальный внутренний идентификатор образа, который позволяет ссылаться на него при конфигурации и при вызове команд Deckhouse Delivery Kit.

Далее для каждого образа в `werf.yaml` необходимо определить сборочные инструкции [с помощью Dockerfile](#dockerfile).

#### Dockerfile

<!-- прим. для перевода: на основе https://werf.io/documentation/v1.2/reference/werf_yaml.html#dockerfile-builder -->

##### Написание Dockerfile-инструкций

Для описания сборочных инструкций образа поддерживается стандартный Dockerfile. Следующие ресурсы помогут в его написании:

* [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/).
* [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).

##### Использование Dockerfile

Конфигурация сборки Dockerfile может выглядеть следующим образом:

```Dockerfile
# Dockerfile
FROM node
WORKDIR /app
COPY package*.json /app/
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: backend
dockerfile: Dockerfile
```

##### Использование определённой Dockerfile-стадии

Также вы можете описывать несколько целевых образов из разных стадий одного и того же Dockerfile:

```Dockerfile
# Dockerfile

FROM node as backend
WORKDIR /app
COPY package*.json /app/
RUN npm ci
COPY . .
CMD ["node", "server.js"]

FROM python as frontend
WORKDIR /app
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-"]
```

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: backend
dockerfile: Dockerfile
target: backend
---
image: frontend
dockerfile: Dockerfile
target: frontend
```

И конечно вы можете описывать образы, основанные на разных Dockerfile:

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: backend
dockerfile: dockerfiles/Dockerfile.backend
---
image: frontend
dockerfile: dockerfiles/Dockerfile.frontend
```

##### Выбор директории сборочного контекста

Чтобы указать сборочный контекст используется директива `context`. **Важно:** в этом случае путь до Dockerfile указывается относительно директории контекста:

```yaml
project: example
configVersion: 1
---
image: docs
context: docs
dockerfile: Dockerfile
---
image: service
context: service
dockerfile: Dockerfile
```

Для образа `docs` будет использоваться Dockerfile по пути `docs/Dockerfile`, а для `service` — `service/Dockerfile`.

##### Добавление произвольных файлов в сборочный контекст

По умолчанию контекст сборки Dockerfile-образа включает только файлы из текущего коммита репозитория проекта. Файлы, не добавленные в Git, или некоммитнутые изменения не попадают в сборочный контекст. Такая логика действует в соответствии ) по умолчанию.

Чтобы добавить в сборочный контекст файлы, которые не хранятся в Git, нужна директива `contextAddFiles` в `werf.yaml`, а также нужно разрешить использование директивы `contextAddFiles` в `werf-giterminism.yaml`:

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: app
context: app
contextAddFiles:
- file1
- dir1/
- dir2/file2.out
```

```yaml
# werf-giterminism.yaml
giterminismConfigVersion: 1
config:
  dockerfile:
    allowContextAddFiles:
    - app/file1
    - app/dir1/
    - app/dir2/file2.out
```

В данной конфигурации контекст сборки будет состоять из следующих файлов:

- `app/**/*` из текущего коммита репозитория проекта;
- файлы `app/file1`, `app/dir2/file2.out` и директория `dir1`, которые находятся в директории проекта.

### Взаимодействие между образами

#### Наследование и импортирование файлов

При написании одного Dockerfile в нашем распоряжении имеется механизм multi-stage. Он позволяет объявить в Dockerfile отдельный образ-стадию и использовать её в качестве базового для другого образа, либо скопировать из неё отдельные файлы.

Deckhouse Delivery Kit позволяет реализовать это не только в рамках одного Dockerfile, но и между произвольными образами, определяемыми в `werf.yaml`, в том числе собираемыми из разных Dockerfile'ов. Всю оркестрацию и выстраивание зависимостей Deckhouse Delivery Kit возьмёт на себя и произведёт сборку за один шаг (вызов `d8 dk build`).

Пример использования образа собранного из `base.Dockerfile` в качестве базового для образа из `Dockerfile`:

```Dockerfile
# base.Dockerfile
FROM ubuntu:22.04
RUN apt update -q && apt install -y gcc g++ build-essential make curl python3
```

```Dockerfile
# Dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}
WORKDIR /app
COPY . .
CMD [ "/app/server", "start" ]
```

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: base
dockerfile: base.Dockerfile
---
image: app
dockerfile: Dockerfile
dependencies:
- image: base
  imports:
  - type: ImageName
    targetBuildArg: BASE_IMAGE
```

#### Передача информации о собранном образе в другой образ

Deckhouse Delivery Kit позволяет получить информацию о собранном образе при сборке другого образа. Например, если в сборочных инструкциях образа `app` требуются имена и digest'ы образов `auth` и `controlplane`, опубликованных в container registry, то конфигурация могла бы выглядеть так:

```Dockerfile
# modules/auth/Dockerfile
FROM alpine
WORKDIR /app
COPY . .
RUN ./build.sh
```

```Dockerfile
# modules/controlplane/Dockerfile
FROM alpine
WORKDIR /app
COPY . .
RUN ./build.sh
```

```Dockerfile
# Dockerfile
FROM alpine
WORKDIR /app
COPY . .

ARG AUTH_IMAGE_NAME
ARG AUTH_IMAGE_DIGEST
ARG CONTROLPLANE_IMAGE_NAME
ARG CONTROLPLANE_IMAGE_DIGEST

RUN echo AUTH_IMAGE_NAME=${AUTH_IMAGE_NAME}                     >> modules_images.env
RUN echo AUTH_IMAGE_DIGEST=${AUTH_IMAGE_DIGEST}                 >> modules_images.env
RUN echo CONTROLPLANE_IMAGE_NAME=${CONTROLPLANE_IMAGE_NAME}     >> modules_images.env
RUN echo CONTROLPLANE_IMAGE_DIGEST=${CONTROLPLANE_IMAGE_DIGEST} >> modules_images.env
```

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: auth
dockerfile: Dockerfile
context: modules/auth/
---
image: controlplane
dockerfile: Dockerfile
context: modules/controlplane/
---
image: app
dockerfile: Dockerfile
dependencies:
- image: auth
  imports:
  - type: ImageName
    targetBuildArg: AUTH_IMAGE_NAME
  - type: ImageDigest
    targetBuildArg: AUTH_IMAGE_DIGEST
- image: controlplane
  imports:
  - type: ImageName
    targetBuildArg: CONTROLPLANE_IMAGE_NAME
  - type: ImageDigest
    targetBuildArg: CONTROLPLANE_IMAGE_DIGEST
```

В процессе сборки Deckhouse Delivery Kit автоматически подставит в указанные build-arguments соответствующие имена и идентификаторы. Всю оркестрацию и выстраивание зависимостей Deckhouse Delivery Kit возьмёт на себя и произведёт сборку за один шаг (вызов `d8 dk build`).

### Мультиплатформенная и кроссплатформенная сборка

Deckhouse Delivery Kit позволяет собирать образы как для родной архитектуры хоста, где запущен Deckhouse Delivery Kit, так и в кроссплатформенном режиме с помощью эмуляции целевой архитектуры, которая может быть отлична от архитектуры хоста. Также Deckhouse Delivery Kit позволяет собрать образ сразу для множества целевых платформ.

> **ЗАМЕЧАНИЕ:** Подготовка хост-системы для мультиплатформенной сборки рассмотрена ), а поддержка этого режима для различных синтаксисов инструкций и бекендов рассмотрены ).

#### Сборка образов под одну целевую платформу

По умолчанию в качестве целевой используется платформа хоста, где запущен Deckhouse Delivery Kit. Выбор другой целевой платформы для собираемых образов осуществляется с помощью параметра `--platform`:

```shell
d8 dk build --platform linux/arm64
```

— все конечные образы, указанные в werf.yaml, будут собраны для указанной платформы с использованием эмуляции.

Целевую платформу можно также указать директивой конфигурации `build.platform`:

```yaml
# werf.yaml
project: example
configVersion: 1
build:
  platform:
  - linux/arm64
---
image: frontend
dockerfile: frontend/Dockerfile
---
image: backend
dockerfile: backend/Dockerfile
```

В этом случае запуск `d8 dk build` без параметров вызовет сборку образов для указанной платформы (при этом явно указанный параметр `--platform` переопределяет значение из werf.yaml).

#### Сборка образов под множество целевых платформ

Поддерживается и сборка образов сразу для набора архитектур. В этом случае в container registry публикуется манифест включающий в себя собранные образы под каждую из указанных платформ (во время скачивания такого образа автоматически будет выбираться образ под требуемую архитектуру).

Можно определить общий список платформ для всех образов в werf.yaml с помощью конфигурации:

```yaml
# werf.yaml
project: example
configVersion: 1
build:
  platform:
  - linux/arm64
  - linux/amd64
  - linux/arm/v7
```

Можно определить список целевых платформ отдельно для каждого собираемого образа (такая настройка будет иметь приоритет над общим списком определённым в werf.yaml):

```yaml
# werf.yaml
project: example
configVersion: 1
---
image: mysql
dockerfile: ./Dockerfile.mysql
platform:
- linux/amd64
---
image: backend
dockerfile: ./Dockerfile.backend
platform:
- linux/amd64
- linux/arm64
```

Общий список можно также переопределить параметром `--platform` непосредственно в момент вызова сборки:

```shell
d8 dk build --platform=linux/amd64,linux/i386
```

— такой параметр переопределяет список целевых платформ указанных в werf.yaml (как общих, так и для отдельных образов).

## Сборочный процесс

### Аутентификация в container registry

Перед работой с образами необходимо аутентифицироваться в container registry. Сделать это можно командой `d8 dk cr login`:

```shell
d8 dk cr login <registry url>
```

Например:

```shell
# Login with username and password from command line
d8 dk cr login -u username -p password registry.example.com

# Login with token from command line
d8 dk cr login -p token registry.example.com

# Login into insecure registry (over http)
d8 dk cr login --insecure-registry registry.example.com
```

> В случае использования команды `d8 dk ci-env` с поддерживаемыми CI/CD-системами аутентификация во встроенные container registry выполняется в рамках команды, поэтому использование команды `d8 dk cr login` в этом случае не требуется.

### Тегирование образов

<!-- прим. для перевода: на основе https://werf.io/documentation/v1.2/internals/stages_and_storage.html#stage-naming -->

Тегирование образов с Deckhouse Delivery Kit выполняется автоматически в рамках сборочного процесса. Используется оптимальная схема тегирования, основанная на содержимом образа, которая предотвращает лишние пересборки и время простоя приложения при выкате.

#### Получение тегов

Для получения тегов образов может использоваться опция `--save-build-report` для команд `d8 dk build`, `d8 dk converge` и пр.:

```shell
# По умолчанию формат JSON.
d8 dk build --save-build-report --repo REPO

# Поддерживается формат envfile.
d8 dk converge --save-build-report --build-report-path .werf-build-report.env --repo REPO

# В команде рендера финальные теги будут доступны только с параметром --repo.
d8 dk render --save-build-report --repo REPO
```

> **ЗАМЕЧАНИЕ:** Получить теги заранее, не вызывая сборочный процесс, на данный момент невозможно, можно получить лишь теги уже собранных ранее образов.

#### Добавление произвольных тегов

Пользователь может добавить произвольное количество дополнительных тегов с опцией `--add-custom-tag`:

```shell
d8 dk build --repo REPO --add-custom-tag main

# Можно добавить несколько тегов-алиасов.
d8 dk build --repo REPO --add-custom-tag main --add-custom-tag latest --add-custom-tag prerelease
```

Шаблон тега может включать следующие параметры:

- `%image%`, `%image_slug%` или `%image_safe_slug%` для использования имени образа из `werf.yaml` (обязательно при сборке нескольких образов);
- `%image_content_based_tag%` для использования content-based тега.

```shell
d8 dk build --repo REPO --add-custom-tag "%image%-latest"
```

> **ЗАМЕЧАНИЕ:** При использовании опций создаются **дополнительные теги-алиасы**, ссылающиеся на автоматические теги-хэши. Полное отключение создания автоматических тегов не предусматривается.

### Послойное кэширование образов

Послойное кэширование образов является неотъемлемой частью сборочного процесса Deckhouse Delivery Kit. Deckhouse Delivery Kit сохраняет и переиспользует сборочный кэш в container registry, а также синхронизирует работу параллельных сборщиков.

> **ЗАМЕЧАНИЕ:** Предполагается, что репозиторий образов для проекта не будет удален или очищен сторонними средствами без негативных последствий для пользователей CI/CD, построенного на основе `d8 dk cleanup`.

#### Dockerfile

По умолчанию Dockerfile-образы кешируются одним образом в container registry.
Для включения послойного кеширования Dockerfile-инструкций в container registry необходимо использовать директиву `staged` в werf.yaml:

```yaml
# werf.yaml
image: example
dockerfile: ./Dockerfile
staged: true
```

### Параллельность и порядок сборки образов

<!-- прим. для перевода: на основе https://werf.io/documentation/v1.2/internals/build_process.html#parallel-build -->

Все образы, описанные в `werf.yaml`, собираются параллельно на одном сборочном хосте. При наличии зависимостей между образами сборка разбивается на этапы, где каждый этап содержит набор независимых образов и может собираться параллельно.

> При использовании Dockerfile-стадий параллельность их сборки также определяется на основе дерева зависимостей. Также, если Dockerfile-стадия используется разными образами, объявленными в `werf.yaml`, Deckhouse Delivery Kit обеспечит однократную сборку этой общей стадии без лишних пересборок

Параллельная сборка в Deckhouse Delivery Kit регулируется двумя параметрами `--parallel` и `--parallel-tasks-limit`. По умолчанию параллельная сборка включена и собирается не более 5 образов одновременно.

Рассмотрим следующий пример:

```Dockerfile
# backend/Dockerfile
FROM node as backend
WORKDIR /app
COPY package*.json /app/
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

```Dockerfile
# frontend/Dockerfile

FROM ruby as application
WORKDIR /app
COPY Gemfile* /app
RUN bundle install
COPY . .
RUN bundle exec rake assets:precompile
CMD ["rails", "server", "-b", "0.0.0.0"]

FROM nginx as assets
WORKDIR /usr/share/nginx/html
COPY configs/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=application /app/public/assets .
COPY --from=application /app/vendor .
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

```yaml
image: backend
dockerfile: Dockerfile
context: backend
---
image: frontend
dockerfile: Dockerfile
context: frontend
target: application
---
image: frontend-assets
dockerfile: Dockerfile
context: frontend
target: assets
```

Имеется 3 образа `backend`, `frontend` и `frontend-assets`. Образ `frontend-assets` зависит от `frontend`, потому что он импортирует скомпилированные ассеты из `frontend`.

Формируются следующие наборы для сборки:

```shell
┌ Concurrent builds plan (no more than 5 images at the same time)
│ Set #0:
│ - ⛵ image backend
│ - ⛵ image frontend
│
│ Set #1:
│ - ⛵ frontend-assets
└ Concurrent builds plan (no more than 5 images at the same time)
```

### Использование container registry

При использовании Deckhouse Delivery Kit container registry используется не только для хранения конечных образов, но также для сборочного кэша и служебных данных, необходимых для работы Deckhouse Delivery Kit (например, метаданные для очистки container registry на основе истории Git). Репозиторий container registry задаётся параметром `--repo`:

```shell
d8 dk converge --repo registry.mycompany.org/project
```

В дополнение к основному репозиторию существует ряд дополнительных:

- `--final-repo` для сохранения конечных образов в отдельном репозитории;
- `--secondary-repo` для использования репозитория в режиме `read-only` (например, для использования container registry CI, в который нельзя пушить, но можно переиспользовать сборочный кэш);
- `--cache-repo` для поднятия репозитория со сборочным кэшом рядом со сборщиками.

> **ВАЖНО.** Для корректной работы Deckhouse Delivery Kit container registry должен быть надёжным (persistent), а очистка должна выполняться только с помощью специальной команды `d8 dk cleanup`

#### Дополнительный репозиторий для конечных образов

При необходимости в дополнение к основному репозиторию могут использоваться т.н. **финальные** репозитории для непосредственного хранения конечных образов.

```shell
d8 dk build --repo registry.mycompany.org/project --final-repo final-registry.mycompany.org/project-final
```

Финальные репозитории позволяют сократить время загрузки образов и снизить нагрузку на сеть за счёт поднятия container registry ближе к кластеру Kubernetes, на котором происходит развёртывание приложения. Также при необходимости финальные репозитории могут использоваться в том же container registry, что и основной репозиторий (`--repo`).

#### Дополнительный репозиторий для быстрого доступа к сборочному кэшу

С помощью параметра `--cache-repo` можно указать один или несколько т.н. **кеширующих** репозиториев.

```shell
# Дополнительный кэширующий репозиторий в локальной сети.
d8 dk build --repo registry.mycompany.org/project --cache-repo localhost:5000/project
```

Кеширующий репозиторий может помочь сократить время загрузки сборочного кэша, но для этого скорость загрузки из него должна быть значительно выше по сравнению с основным репозиторием — как правило, это достигается за счёт поднятия container registry в локальной сети, но это необязательно.

При загрузке сборочного кэша кеширующие репозитории имеют больший приоритет, чем основной репозиторий. При использовании кеширующих репозиториев сборочный кэш продолжает сохраняться и в основном репозитории.

Очистка кeширующего репозитория может осуществляться путём его полного удаления без каких-либо рисков.

### Синхронизация сборщиков

<!-- прим. для перевода: на основе https://werf.io/documentation/v1.2/advanced/synchronization.html -->

Для обеспечения согласованности в работе параллельных сборщиков, а также гарантии воспроизводимости образов и промежуточных слоёв, Deckhouse Delivery Kit берёт на себя ответственность за синхронизацию сборщиков.

Сервис синхронизации — это компонент Deckhouse Delivery Kit, который предназначен для координации нескольких процессов Deckhouse Delivery Kit и выполняет роль _менеджера блокировок_. Блокировки требуются для корректной публикации новых образов в container registry и реализации алгоритма сборки, описанного в разделе [«Послойное кэширование образов»](#послойное-кэширование-образов).

На сервис синхронизации отправляются только обезличенные данные в виде хэш-сумм тегов, публикуемых в container registry.

В качестве сервиса синхронизации может выступать:
1. HTTP-сервер синхронизации, реализованный в команде `d8 dk synchronization`.
2. Ресурс ConfigMap в кластере Kubernetes. В качестве механизма используется библиотека [lockgate](https://github.com/werf/lockgate), реализующая распределённые блокировки через хранение аннотаций в выбранном ресурсе.
3. Локальные файловые блокировки, предоставляемые операционной системой.

#### Использование собственного сервиса синхронизации

##### HTTP-сервер

Сервер синхронизации можно запустить командой `d8 dk synchronization`, например для использования порта 55581 (по умолчанию):

```shell
d8 dk synchronization --host 0.0.0.0 --port 55581
```

— данный сервер поддерживает только работу в режиме HTTP, для использования HTTPS необходима настройка дополнительной SSL-терминации сторонними средствами (например через Ingress в Kubernetes).

Далее во всех командах Deckhouse Delivery Kit, которые используют параметр `--repo` дополнительно указывается параметр `--synchronization=http[s]://DOMAIN`, например:

```shell
d8 dk build --repo registry.mydomain.org/repo --synchronization https://synchronization.domain.org
d8 dk converge --repo registry.mydomain.org/repo --synchronization https://synchronization.domain.org
```

##### Специальный ресурс в Kubernetes

Требуется лишь предоставить рабочий кластер Kubernetes, и выбрать namespace, в котором будет хранится сервисный ConfigMap/werf, через аннотации которого будет происходить распределённая блокировка.

Далее во всех командах Deckhouse Delivery Kit, которые используют параметр `--repo` дополнительно указывается параметр `--synchronization=kubernetes://NAMESPACE[:CONTEXT][@(base64:CONFIG_DATA)|CONFIG_PATH]`, например:

```shell
# Используем стандартный ~/.kube/config или KUBECONFIG.
d8 dk build --repo registry.mydomain.org/repo --synchronization kubernetes://mynamespace
d8 dk converge --repo registry.mydomain.org/repo --synchronization kubernetes://mynamespace

# Явно указываем содержимое kubeconfig через base64.
d8 dk build --repo registry.mydomain.org/repo --synchronization kubernetes://mynamespace@base64:YXBpVmVyc2lvbjogdjEKa2luZDogQ29uZmlnCnByZWZlcmVuY2VzOiB7fQoKY2x1c3RlcnM6Ci0gY2x1c3RlcjoKICBuYW1lOiBkZXZlbG9wbWVudAotIGNsdXN0ZXI6CiAgbmFtZTogc2NyYXRjaAoKdXNlcnM6Ci0gbmFtZTogZGV2ZWxvcGVyCi0gbmFtZTogZXhwZXJpbWVudGVyCgpjb250ZXh0czoKLSBjb250ZXh0OgogIG5hbWU6IGRldi1mcm9udGVuZAotIGNvbnRleHQ6CiAgbmFtZTogZGV2LXN0b3JhZ2UKLSBjb250ZXh0OgogIG5hbWU6IGV4cC1zY3JhdGNoCg==

# Используем контекст mycontext в конфиге /etc/kubeconfig.
d8 dk build --repo registry.mydomain.org/repo --synchronization kubernetes://mynamespace:mycontext@/etc/kubeconfig
```

> **ЗАМЕЧАНИЕ:** Данный способ неудобен при доставке проекта в разные кластера Kubernetes из одного Git-репозитория из-за сложности корректной настройки. В этом случае для всех команд Deckhouse Delivery Kit требуется указывать один и тот же адрес кластера и ресурс, даже если деплой происходит в разные контура, чтобы обеспечить консистивность данных в container registry. Поэтому для такого случая рекомендуется запустить отдельный общий сервис синхронизации, чтобы исключить вероятность некорректной конфигурации.

##### Локальная синхронизация

Включается опцией `--synchronization=:local`. Локальный _менеджер блокировок_ использует файловые блокировки, предоставляемые операционной системой.

```shell
d8 dk build --repo registry.mydomain.org/repo --synchronization :local
d8 dk converge --repo registry.mydomain.org/repo --synchronization :local
```

> **ЗАМЕЧАНИЕ:** Данный способ подходит лишь в том случае, если в вашей CI/CD системе все запуски Deckhouse Delivery Kit происходят с одного и того же раннера.
