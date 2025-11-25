---
title: Дистрибуция
weight: 60
menuTitle: Deckhouse Delivery Kit
---

{{< alert level="warning" >}}
Функциональность Deckhouse Delivery Kit доступна только если у вас есть лицензия на любую коммерческую версию Deckhouse Kubernetes Platform.
{{< /alert >}}

## Основы

Обычный цикл доставки приложений с Deckhouse Delivery Kit выглядит как сборка образов, их публикация и последующее развертывание чартов, для чего бывает достаточно одного вызова команды `d8 dk converge`. Но иногда возникает необходимость разделить *дистрибуцию* артефактов (образы, чарты) и их *развертывание*, либо даже реализовать развертывание артефактов вовсе без Deckhouse Delivery Kit, а с использованием стороннего ПО.

В этом разделе рассматриваются способы дистрибуции образов и бандлов (чартов и связанных с ними образов) для их дальнейшего развертывания с Deckhouse Delivery Kit или без него. Инструкции развертывания опубликованных артефактов можно найти в разделе «Развертывание».

### Пример дистрибуции образа

Для дистрибуции единственного образа, собираемого через Dockerfile, который потом будет развернут сторонним ПО, достаточно двух файлов и одной команды `d8 dk export`, запущенной в Git-репозитории приложения:

```yaml
# werf.yaml:
project: myproject
configVersion: 1
---
image: myapp
dockerfile: Dockerfile
```

```dockerfile
# Dockerfile:
FROM node

WORKDIR /app
COPY . .
RUN npm ci

CMD ["node", "server.js"]
```

```shell
d8 dk export myapp --repo example.org/myproject --tag other.example.org/myproject/myapp:latest
```

Результат: опубликован образ приложения `other.example.org/myproject/myapp:latest`, готовый для развертывания сторонним ПО.

### Пример дистрибуции бандла

Для дистрибуции бандла для дальнейшего его развертывания с Deckhouse Delivery Kit, подключения его как зависимого чарта или развертывания бандла как чарта сторонним ПО в простейшем случае достаточно трёх файлов и одной команды `d8 dk bundle publish`, запущенной в Git-репозитории приложения:

```yaml
# werf.yaml:
project: mybundle
configVersion: 1
---
image: myapp
dockerfile: Dockerfile
```

```dockerfile
# Dockerfile:
FROM node

WORKDIR /app
COPY . .
RUN npm ci

CMD ["node", "server.js"]
```

```shell
# .helm/templates/myapp.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: {{ $.Values.werf.image.myapp }}
```

```shell
d8 dk bundle publish --repo example.org/bundles/mybundle
```

Результат: опубликован чарт `example.org/bundles/mybundle:latest` и связанный с ним собранный образ.

## Образы

### О дистрибуции образов

Дистрибуция собираемых Deckhouse Delivery Kit образов для использования сторонними пользователями и/или ПО осуществляется командой `d8 dk export`. Эта команда соберёт и опубликует образы в container registry, при этом убрав все ненужные для стороннего ПО метаданные, чем полностью выведет образы из-под контроля Deckhouse Delivery Kit, позволив организовать их дальнейший жизненный цикл сторонними средствами.

> Опубликованные командой `d8 dk export` образы *никогда* не будут удаляться командой `d8 dk cleanup`, в отличие от образов, опубликованных обычным способом. Очистка экспортированных образов должна быть реализована сторонними средствами.

### Дистрибуция образа

```shell
d8 dk export \
    --repo example.org/myproject \
    --tag other.example.org/myproject/myapp:latest
```

Результат: образ собран и сначала опубликован с content-based тегом в container registry `example.org/myproject`, а затем опубликован в другой container registry `other.example.org/myproject` как целевой экспортированный образ `other.example.org/myproject/myapp:latest`.

В параметре `--tag` можно указать тот же репозиторий, что и в `--repo`, таким образом используя один и тот же container registry и для сборки, и для экспортированного образа.

### Дистрибуция нескольких образов

В параметре `--tag` можно использовать шаблоны `%image%`, `%image_slug%` и `%image_safe_slug%` для подставления имени образа из `werf.yaml`, основанном на его содержимом, например:

```shell
d8 dk export \
    --repo example.org/mycompany/myproject \
    --tag example.org/mycompany/myproject/%image%:latest
```

### Дистрибуция произвольных образов

Используя позиционные аргументы и имена образов из `werf.yaml` можно выбрать произвольные образы, например:

```shell
d8 dk export backend frontend \
    --repo example.org/mycompany/myproject \
    --tag example.org/mycompany/myproject/%image%:latest
```

### Использование content-based-тега при формировании тега

В параметре `--tag` можно использовать шаблон `%image_content_based_tag%` для использования тега образа, основанном на его содержимом, например:

```shell
d8 dk export \
    --repo example.org/mycompany/myproject \
    --tag example.org/mycompany/myproject/myapp:%image_content_based_tag%
```

### Добавление произвольных лейблов

Используя параметр `--add-label` можно добавить произвольное количество дополнительных лейблов к экспортируемому образу(ам), например:

```shell
d8 dk export \
    --repo example.org/mycompany/myproject \
    --tag registry.werf.io/werf/werf:latest \
    --add-label io.artifacthub.package.readme-url=https://raw.githubusercontent.com/werf/werf/main/README.md \
    --add-label org.opencontainers.image.created=2023-03-13T11:55:24Z \
    --add-label org.opencontainers.image.description="Official image to run Deckhouse Delivery Kit in containers"
```

## Бандлы и чарты

### О бандлах и чартах

Бандл — это способ дистрибуции чарта и связанных с ним образов как единого целого.

Командой `d8 dk bundle publish` можно опубликовать чарт и связанные образы для дальнейшего развёртывания с Deckhouse Delivery Kit. При этом для развёртывания доступ к Git-репозиторию приложения больше не потребуется.

Эта же команда подходит для публикации чарта. Опубликованный в OCI-репозиторий чарт может использоваться в качестве основного или зависимого чарта с Deckhouse Delivery Kit, Helm, Argo CD, Flux и другими решениями.

При упаковке Deckhouse Delivery Kit автоматически добавляет следующие данные в чарт:

* имена собираемых образов и их динамических тегов в Values чарта;
* значения, переданных через параметры командной строки или переменные окружения, в Values чарта;
* глобальные пользовательские и служебные аннотации и лейблы для добавления в ресурсы чарта при развёртывании командой `d8 dk bundle apply`.

Опубликованный бандл (чарт и связанные с ним образы) можно копировать в другой репозиторий container registry или выгружать в/из архива с помощью одной команды `d8 dk bundle copy`.

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

> В случае использования команды `d8 dk ci-env` с поддерживаемыми CI/CD-системами аутентификация во встроенные container registry выполняется в рамках команды, поэтому использование команды d8 dk cr login в этом случае не требуется.

### Публикация бандла

Опубликовать бандл в OCI-репозиторий можно следующим способом:

1. Создайте `werf.yaml`, если его ещё нет:

   ```yaml
   # werf.yaml:
   project: mybundle
   configVersion: 1
   ```

2. Разместите файлы в директории основного чарта (по умолчанию `.helm` в корне Git-репозитория). Заметьте, что при публикации в чарт будут включены *только* следующие файлы и директории:

   ```shell
   .helm/
     charts/
     templates/
     crds/
     files/
     Chart.yaml
     values.yaml
     values.schema.json
     LICENSE
     README.md
   ```

   Для публикации дополнительных файлов/директорий выставьте переменную окружения `WERF_BUNDLE_SCHEMA_NONSTRICT=1`, после чего будут публиковаться *все* файлы и директории в директории основного чарта, а не только вышеуказанные.

3. Следующей командой опубликуйте бандл. Соберите и опубликуйте описанные в `werf.yaml` образы (если таковые есть), затем опубликуйте содержимое `.helm` в виде OCI-образа `example.org/bundles/mybundle:latest`:

   ```shell
   d8 dk bundle publish --repo example.org/bundles/mybundle
   ```

### Публикация нескольких бандлов из одного Git-репозитория

Разместите `.helm` с содержимым чарта и соответствующий ему `werf.yaml` в отдельную директорию для каждого бандла:

```shell
bundle1/
  .helm/
    templates/
    # ...
  werf.yaml
bundle2/
  .helm/
    templates/
    # ...
  werf.yaml
```

Теперь опубликуйте каждый бандл по отдельности:

```shell
cd bundle1
d8 dk bundle publish --repo example.org/bundles/bundle1

cd ../bundle2
d8 dk bundle publish --repo example.org/bundles/bundle2
```

### Исключение файлов или директорий из публикуемого чарта

Файл `.helmignore`, находящийся в корне чарта, может содержать фильтры по именам файлов, при соответствии которым файлы или директории не будут добавляться в чарт при публикации. Формат правил такой же, как и [в .gitignore](https://git-scm.com/docs/gitignore), за исключением:

- `**` не поддерживается;

- `!` в начале строки не поддерживается;

- `.helmignore` не исключает сам себя по умолчанию.

Также опция `--disable-default-values` для команды `d8 dk bundle publish` позволяет исключить из публикуемого чарта файл `values.yaml`.

### Указание версии чарта при публикации

По умолчанию чарт публикуется с тегом `latest`. Указать иной тег, например, семантическую версию для публикуемого чарта, можно опцией `--tag`:

```shell
d8 dk bundle publish --repo example.org/bundles/mybundle --tag v1.0.0
```

Результат: опубликован чарт `example.org/bundles/mybundle:v1.0.0`.

Если при публикации будет обнаружено, что в OCI-репозитории уже существует чарт с таким тегом, то чарт в репозитории будет перезаписан.

### Изменение версии опубликованного чарта

Для изменения тега уже опубликованного чарта скопируйте бандл с новым тегом с помощью команды `d8 dk bundle copy`, например:

```shell
d8 dk bundle copy --from example.org/bundles/mybundle:v1.0.0 --to example.org/bundles/renamedbundle:v2.0.0
```

### Копирование бандла в другой репозиторий

Для удобного копирования бандла в другой репозиторий имеется команда `d8 dk bundle copy`. Кроме непосредственного копирования чарта и связанных с ним образов эта команда также обновит сохранённые в чарте Values, указывающие на путь к образам.

Пример:

```shell
d8 dk bundle copy --from example.org/bundles/mybundle:v1.0.0 --to other.example.org/bundles/mybundle:v1.0.0
```

### Экспорт бандла из container registry в архив

После публикации бандл может быть экспортирован из репозитория в локальный архив для дальнейшей дистрибуции иными способами с помощью команды `d8 dk bundle copy`, например:

```shell
d8 dk bundle copy --from example.org/bundles/mybundle:v1.0.0 --to archive:archive.tar.gz
```

### Импорт бандла из архива в репозиторий

Экспортированный в архив бандл можно снова импортировать в тот же или другой OCI-репозиторий командой `d8 dk bundle copy`, например:

```shell
d8 dk bundle copy --from archive:archive.tar.gz --to other.example.org/bundles/mybundle:v1.0.0
```

После этого вновь опубликованный бандл (чарт и его образы) снова можно использовать привычными способами.

### Container registries, поддерживающие публикацию бандлов

Для публикации бандлов требуется container registry, поддерживающий спецификацию OCI ([Open Container Initiative](https://github.com/opencontainers/image-spec)). Список наиболее популярных container registries, совместимость с которыми была проверена:

| Container registry        | Поддерживает публикацию бандлов |
|---------------------------|:-------------------------------:|
| AWS ECR                   |                +                |
| Azure CR                  |                +                |
| Docker Hub                |                +                |
| GCR                       |                +                |
| GitHub Packages           |                +                |
| GitLab Registry           |                +                |
| Harbor                    |                +                |
| JFrog Artifactory         |                +                |
| Yandex container registry |                +                |
| Nexus                     |                +                |
| Quay                      |                -                |
