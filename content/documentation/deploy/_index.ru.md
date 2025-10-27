---
title: Развертывание
weight: 50
menuTitle: Deckhouse Delivery Kit
---

{{< alert level="warning" >}}
Функциональность Deckhouse Delivery Kit доступна только если у вас есть лицензия на любую коммерческую версию Deckhouse Kubernetes Platform.
{{< /alert >}}

## Основы

При организации доставки приложения в Kubernetes необходимо определиться с тем, какой формат выбрать для управления конфигурацией развёртывания (параметризации, управления зависимостями, конфигурации под различные окружения и т.д.), а также способом применения этой конфигурации – непосредственно механизмом развёртывания.

В Deckhouse Delivery Kit встроен Helm, и именно он используется для решения перечисленных задач. Разработка и сопровождение конфигурации реализуется с помощью Helm-чарта, а для процесса развёртывания предлагается Helm c дополнительными возможностями:

- отслеживание состояния выкатываемых ресурсов (с возможностью изменения поведения для каждого ресурса):
  - умное ожидание готовности ресурсов;
  - мгновенное завершение проблемного развертывания без необходимости ожидания таймаута;
  - прогресс развёртывания, логи, системные события и ошибки приложения.
- использование произвольного порядка развертывания для любых ресурсов, а не только для хуков;
- ожидание создания и готовности ресурсов, не принадлежащих релизу;
- интеграция сборки и развертывания и многое другое.

Deckhouse Delivery Kit стремится сделать работу с Helm более простой, удобной и гибкой, при этом не ломая обратную совместимость с Helm-чартами, Helm-шаблонами и Helm-релизами.

### Простой пример развертывания

Для развертывания простого приложения достаточно двух файлов и команды `d8 dk converge`, запущенной в Git-репозитории приложения:

```yaml
# .helm/templates/hello.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - image: nginxdemos/hello:plain-text
```

```yaml
# werf.yaml:
configVersion: 1
project: hello
```

```shell
d8 dk converge --repo registry.example.org/repo --env production
```

Результат: Deployment `hello` развёрнут в Namespace'е `hello-production`.

### Расширенный пример развертывания

Более сложный пример развертывания со сборкой образов и внешними Helm-чартами:

```yaml
# werf.yaml:
configVersion: 1
project: myapp
---
image: backend
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

```yaml
# .helm/Chart.yaml:
dependencies:
- name: postgresql
  version: "~12.1.9"
  repository: https://charts.bitnami.com/bitnami
```

```yaml
# .helm/values.yaml:
backend:
  replicas: 1
```

```yaml
# .helm/templates/backend.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: {{ $.Values.backend.replicas }}
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - image: {{ $.Values.werf.image.backend }}
```

```shell
d8 dk converge --repo registry.example.org/repo --env production
```

Результат: собран образ `backend`, а затем Deployment `backend` и ресурсы чарта `postgresql` развёрнуты в Namespace'е `myapp-production`.

## Чарты и зависимости

### О чартах

Чарты в Deckhouse Delivery Kit – это Helm-чарты с некоторыми дополнительными возможностями. А Helm-чарты – это распространяемые пакеты с Helm-шаблонами, values-файлами и некоторыми метаданными. Из чартов формируются конечные Kubernetes-манифесты для дальнейшего развертывания.

### Создание нового чарта

При развертывании c `d8 dk converge` или публикации c `d8 dk bundle publish` используется чарт, лежащий в директории `.helm` в корне Git-репозитория. Этот чарт называется *основным*. Директорию основного чарта можно изменить директивой `deploy.helmChartDir` файла `werf.yaml`.

Для обычного чарта требуется создание файла `Chart.yaml` и указание в нём имени и версии чарта:

```yaml
# Chart.yaml:
apiVersion: v2
name: mychart
version: 1.2.3
```

А вот для основного чарта это не обязательно, т. к. при отсутствии файла `Chart.yaml` или отсутствии в нём имени или версии чарта будет использована следующая конфигурация:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
name: <имя проекта Deckhouse Delivery Kit>
version: 1.0.0
```

Если такая конфигурация основного чарта вас не устраивает, то создайте файл `.helm/Chart.yaml` самостоятельно и переопределите вышеупомянутые директивы.

В случае, если ваш чарт будет содержать только именованные шаблоны для использования в других чартах, добавьте в `Chart.yaml` директиву `type: library`:

```yaml
# Chart.yaml:
type: library
```

Если ваш чарт совместим только с частью версий Kubernetes, то ограничьте версии кластера Kubernetes, в которые чарт можно развернуть, директивой `kubeVersion`, например:

```yaml
# Chart.yaml:
kubeVersion: "~1.20.3"
```

При желании можно добавить следующие информационные директивы:

```yaml
# Chart.yaml:
appVersion: "1.0"
deprecated: false
icon: https://example.org/mychart-icon.svg
description: This is My Chart
home: https://example.org
sources:
  - https://github.com/my/chart
keywords:
  - apps
annotations:
  anyAdditionalInfo: here
maintainters:
  - name: John Doe
    email: john@example.org
    url: https://john.example.org
```

Полученный чарт уже можно развернуть или опубликовать, хотя в таком виде от него мало пользы. Теперь вам понадобится, по меньшей мере, либо добавить шаблоны в директорию `templates`, либо подключить зависимые чарты.

### Добавление файлов в чарт

По мере необходимости добавьте в чарт шаблоны, параметры, зависимые чарты и прочее. Содержимое основного чарта может выглядеть так:

```
.helm/
  charts/
    dependent-chart/
      # ...
  templates/
    deployment.yaml
    _helpers.tpl
    NOTES.txt
  crds/
    crd.yaml
  secret/                   # Только в Deckhouse Delivery Kit
    some-secret-file
  values.yaml
  values.schema.json
  secret-values.yaml        # Только в Deckhouse Delivery Kit
  Chart.yaml
  Chart.lock
  README.md
  LICENSE
  .helmignore
```

Подробнее:

- `charts/*` — зависимые чарты, чьи Helm-шаблоны/values-файлы используются для формирования манифестов наравне с Helm-шаблонами/values-файлами родительского чарта;

- `templates/*.yaml` — Helm-шаблоны, из которых формируются Kubernetes-манифесты;

- `templates/*.tpl` — файлы с Helm-шаблонами для использования в других Helm-шаблонах. Результат шаблонизации этих файлов игнорируется;

- `templates/NOTES.txt` — Deckhouse Delivery Kit отображает содержимое этого файла в терминале в конце каждого удачного развертывания;

- `crds/*.yaml` — [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions), которые развертываются до развертывания манифестов в `templates`;

- `secret/*` — (только в Deckhouse Delivery Kit) зашифрованные файлы, расшифрованное содержимое которых можно подставлять в Helm-шаблоны;

- `values.yaml` — файлы с декларативной конфигурацией для использования в Helm-шаблонах. Конфигурация в них может переопределяться переменными окружения, аргументами командной строки или другими values-файлами;

- `values.schema.json` — JSON-схема для валидации `values.yaml`;

- `secret-values.yaml` — (только в Deckhouse Delivery Kit) зашифрованный файл с декларативной конфигурацией, аналогичный `values.yaml`. Его расшифрованное содержимое объединяется с `values.yaml` во время формирования манифестов;

- `Chart.yaml` — основная конфигурация и метаданные чарта;

- `Chart.lock` — lock-файл, защищающий от нежелательного изменения/обновления зависимых чартов;

- `README.md` — документация чарта;

- `LICENSE` — лицензия чарта;

- `.helmignore` — список файлов в директории чарта, которые не нужно включать в чарт при его публикации.

### Подключение дополнительных чартов

#### Подключение зависимых локальных чартов

Подключить зависимые локальные чарты можно, положив их в директорию `charts` родительского чарта. В таком случае манифесты сформируются и для родительского, и для зависимых чартов, после чего объединятся вместе для дальнейшего развертывания. Дополнительная конфигурации не обязательна.

Пример содержимого основного чарта, имеющего локальные зависимые чарты:

```
.helm/
  charts/
    postgresql/
      templates/
        postgresql.yaml
      Chart.yaml
    redis/
      templates/
        redis.yaml
      Chart.yaml
  templates/
    backend.yaml
```

> Обратите внимание, что у локальных зависимых чартов имя должно *обязательно* совпадать с именем их директории.

Если локальному зависимому чарту требуется дополнительная конфигурация, то в файле `Chart.yaml` родительского чарта укажите имя зависимого чарта без указания `dependencies[].repository` и добавьте интересующие директивы таким образом:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: redis
  condition: redis.enabled
```

При необходимости подключить локальный чарт не из директории `charts`, а из другого места, используйте директиву `dependencies[].repository` так:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: redis
  repository: file://../redis
```

#### Подключение зависимых чартов из репозитория

Подключить дополнительные чарты из OCI/HTTP-репозитория можно, указав их как зависимые в директиве `dependencies` файла `Chart.yaml` родительского чарта. В таком случае манифесты сформируются и для родительского, и для зависимых чартов, после чего объединятся вместе для дальнейшего развертывания.

Пример конфигурации чарта из репозитория, зависимого от основного чарта:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: database
  version: "~1.2.3"
  repository: https://example.com/charts
```

После каждого добавления/обновления удалённых зависимых чартов или изменения их конфигурации требуется:

1. (Если используется приватный OCI или HTTP-репозиторий c зависимым чартом) Добавить OCI или HTTP-репозиторий вручную с `d8 dk helm repo add`, указав нужные опции для доступа к репозиторию.

2. Вызвать `d8 dk helm dependency update`, который обновит `Chart.lock`.

3. Закоммитить обновлённые `Chart.yaml` и `Chart.lock` в Git.

Также при использовании чартов из репозитория рекомендуется добавить `.helm/charts/**.tgz` в `.gitignore`.

#### Указание имени подключаемого чарта

В директиве `dependencies[].name` родительского чарта указывается оригинальное имя зависимого чарта, установленное его разработчиком, например:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: backend
```

Если нужно подключить несколько зависимых чартов с одинаковым именем или подключить один и тот же зависимый чарт несколько раз, то используйте директиву `dependencies[].alias` родительского чарта, чтобы поменять имена подключаемых чартов, например:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: backend
  alias: main-backend
- name: backend
  alias: secondary-backend
```

#### Указание версии подключаемого чарта

Ограничить подходящие версии зависимого чарта, из которых будет выбрана наиболее свежая, можно директивой `dependencies[].version` родительского чарта, например:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: backend
  version: "~1.2.3"
```

Результат: будет использована самая свежая версия 1.2.x, но как минимум 1.2.3.

#### Указание источника подключаемого чарта

Указать путь к источнику чартов, в котором можно найти указанный зависимый чарт, можно директивой `dependencies[].repository` родительского чарта, например:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: mychart
  repository: oci://example.org/myrepo
```

Результат: будет использован чарт `mychart` из OCI-репозитория `example.org/myrepo`.

#### Включение/отключение зависимых чартов

По умолчанию все зависимые чарты включены. Для произвольного включения/отключения зависимых чартов можно использовать директиву `dependencies[].condition` родительского чарта, например:

```yaml
# .helm/Chart.yaml:
apiVersion: v2
dependencies:
- name: backend
  condition: backend.enabled
```

Результат: зависимый чарт `backend` будет включен, только если параметр `$.Values.backend.enabled` имеет значение `true` (по умолчанию — `true`).

Также можно использовать директиву `dependencies[].tags` родительского чарта для включения/отключения целых групп зависимых чартов сразу, например:

```yaml
# .helm/Chart.yaml:
dependencies:
- name: backend
  tags: ["app"]
- name: frontend
  tags: ["app"]
```

Результат: зависимые чарты `backend` и `frontend` будут включены, только если параметр `$.Values.tags.app` имеет значение `true` (по умолчанию — `true`).

## Шаблоны

### Шаблонизация

Механизм шаблонизации в Deckhouse Delivery Kit ничем не отличается от Helm. Используется движок шаблонов [Go text/template](https://pkg.go.dev/text/template), расширенный готовым набором функций [Sprig](https://masterminds.github.io/sprig/) и Helm.

### Файлы шаблонов

В директории `templates` чарта находятся файлы шаблонов.

Файлы шаблонов `templates/*.yaml` формируют конечные Kubernetes-манифесты для развертывания. Каждый из этих файлов может формировать сразу несколько манифестов Kubernetes-ресурсов. Для этого манифесты должны быть разделены строкой `---`.

Файлы шаблонов `templates/_*.tpl` содержат только именованные шаблоны для использования в других файлах. Файлы `*.tpl` не формируют Kubernetes-манифесты сами по себе.

### Действия

Главный элемент шаблонизации — действие. Действие может возвращать только строки. Действие заключается в двойные фигурные скобки:

```
{{ print "hello" }}
```

Результат:

```
hello
```

### Переменные

Переменные используются для хранения или указания на данные любого типа.

Объявление и присваивание переменной:

```
{{ $myvar := "hello" }}
```

Присваивание нового значения существующей переменной:

```
{{ $myvar = "helloworld" }}
```

Использование переменной:

```
{{ $myvar }}
```

Результат:

```
helloworld
```

Использование предопределенных переменных:

```
{{ $.Values.werf.env }}
```

Данные можно подставлять и без объявления переменных:

```
labels:
  app: {{ "myapp" }}
```

Результат:

```yaml
labels:
  app: myapp
```

Также в переменные можно сохранять результат выполнения функций или конвейеров:

```
{{ $myvar := 1 | add 1 1 }}
{{ $myvar }} 
```

Результат:

```
3
```

### Области видимости переменных

Область видимости ограничивает видимость переменных. По умолчанию область видимости переменных ограничена файлом-шаблоном.

Область видимости может меняться при использовании некоторых блоков и функций. К примеру, блок `if` создаёт новую область видимости, а переменные, объявленные в блоке `if`, будут недоступны снаружи:

```
{{ if true }}
  {{ $myvar := "hello" }}
{{ end }}

{{ $myvar }}
```

Результат:

```
Error: ... undefined variable "$myvar"
```

Чтобы обойти это ограничение, объявите переменную за пределами блока, а значение присвойте ей внутри блока:

```
{{ $myvar := "" }}
{{ if true }}
  {{ $myvar = "hello" }}
{{ end }}

{{ $myvar }}
```

Результат:

```
hello
```

### Типы данных

Доступные типы данных:

| Тип данных                                                           | Пример                                              |
| -------------------------------------------------------------------- | --------------------------------------------------- |
| Логический                                                           | `{{ true }}`                   |
| Строка                                                               | `{{ "hello" }}`                |
| Целое число                                                          | `{{ 1 }}`                      |
| Число с плавающей точкой                                             | `{{ 1.1 }}`                    |
| Список с элементами любого типа, упорядоченный                       | `{{ list 1 2 3 }}`             |
| Словарь с ключами-строками и значениями любого типа, неупорядоченный | `{{ dict "key1" 1 "key2" 2 }}` |
| Специальные объекты                                                  | `{{ $.Files }}`                |
| Нуль                                                                 | `{{ nil }}`                    |

### Функции

В Deckhouse Delivery Kit встроена обширная библиотека функций для использования в шаблонах. Основная их часть — функции Helm.

Функции можно использовать только в действиях. Функции *могут* иметь аргументы и *могут* возвращать данные любого типа. Например, приведенная ниже функция сложения принимает три аргумента-числа и возвращает число:

```
{{ add 3 2 1 }}
```

Результат:

```
6
```

Обратите внимание, что **результат выполнения действия всегда конвертируется в строку** независимо от возвращаемого функцией типа данных.

Аргументами функций могут быть:

- простые значения: `1`;

- вызовы других функций: `add 1 1`;

- конвейеры: `1 | add 1`;

- комбинации вышеперечисленных типов: `1 | add (add 1 1)`.

Если аргумент — не простое значение, а вызов другой функции или конвейер, заключите его в круглые скобки `()`:

```
{{ add 3 (add 1 1) (1 | add 1) }}
```

Чтобы игнорировать возвращаемый функцией результат, просто сохраните его в переменную `$_`:

```
{{ $_ := set $myDict "mykey" "myvalue"}}
```

### Конвейеры

Конвейеры позволяют передать результат выполнения первой функции как последний аргумент во вторую функцию, а результат второй функции — как последний аргумент в третью и так далее:

```
{{ now | unixEpoch | quote }}
```

Здесь результат выполнения функции `now` (получить текущую дату) передаётся как аргумент в функцию `unixEpoch` (преобразует дату в Unix time), после чего полученное значение передаётся в функцию `quote` (оборачивает в кавычки).

Итоговый результат:

```
"1671466310"
```

Использование конвейеров не обязательно, и при желании их можно переписать следующим образом:

```
{{ quote (unixEpoch (now)) }}
```

... однако рекомендуется использовать именно конвейеры.

### Логические операции и сравнения

Логические операции реализуются следующими функциями:

| Операция | Функция                        | Пример                                     |
| -------- | ------------------------------ | ------------------------------------------ |
| НЕ       | `not <arg>`                    | `{{ not false }}`     |
| И        | `and <arg> <arg> [<arg>, ...]` | `{{ and true true }}` |
| ИЛИ      | `or <arg> <arg> [<arg>, ...]`  | `{{ or false true }}` |

Сравнения реализуются следующими функциями:

| Сравнение               | Функция                        | Пример                                           |
| ----------------------- | ------------------------------ | ------------------------------------------------ |
| Эквивалентно            | `eq <arg> <arg> [<arg>, ...]`  | `{{ eq "hello" "hello" }}`  |
| Не эквивалентно         | `neq <arg> <arg> [<arg>, ...]` | `{{ neq "hello" "world" }}` |
| Меньше                  | `lt <arg> <arg>`               | `{{ lt 1 2 }}`              |
| Больше                  | `gt <arg> <arg>`               | `{{ gt 2 1 }}`              |
| Меньше или эквивалентно | `le <arg> <arg>`               | `{{ le 1 2 }}`              |
| Больше или эквивалентно | `ge <arg> <arg>`               | `{{ ge 2 1 }}`              |

Пример комбинирования:

```
{{ and (eq true true) (neq true false) (not (empty "hello")) }}
```

### Ветвления

Ветвления `if/else` позволяют выполнять шаблонизацию только при выполнении/невыполнении определенных условий. Пример:

```
{{ if $.Values.app.enabled }}
# ...
{{ end }}
```

Условие считается *невыполненным*, если результатом его вычисления является:

* логическое `false`;

* число `0`;

* пустая строка `""`;

* пустой список `[]`;

* пустой словарь `{}`;

* нуль: `nil`.

В остальных случаях условие считается выполненным. Условием могут быть данные, переменная, функция или конвейер.

Полный пример:

```
{{ if eq $appName "backend" }}
app: mybackend
{{ else if eq $appName "frontend" }}
app: myfrontend
{{ else }}
app: {{ $appName }}
{{ end }}
```

Простые ветвления можно реализовывать не только с `if/else`, но и с функцией `ternary`. Например, следующее выражение с `ternary`:

```
{{ ternary "mybackend" $appName (eq $appName "backend") }}
```

... аналогично приведенной ниже конструкции `if/else`:

```
{{ if eq $appName "backend" }}
app: mybackend
{{ else }}
app: {{ $appName }}
{{ end }}
```

### Циклы

#### Циклы по спискам

Циклы `range` позволяют перебирать элементы списка и выполнять нужную шаблонизацию на каждой итерации:

```
{{ range $urls }}
{{ . }}
{{ end }}
```

Результат:

```
https://example.org
https://sub.example.org
```

Относительный контекст `.` всегда указывает на элемент списка, соответствующий текущей итерации, хотя указатель можно сохранить и в произвольную переменную:

```
{{ range $elem := $urls }}
{{ $elem }}
{{ end }}
```

Результат будет таким же:

```
https://example.org
https://sub.example.org
```

Получить индекс элемента в списке можно следующим образом:

```
{{ range $i, $elem := $urls }}
{{ $elem }} имеет индекс {{ $i }}
{{ end }}
```

Результат:

```
https://example.org имеет индекс 0
https://sub.example.org имеет индекс 1
```

#### Циклы по словарям

Циклы `range` позволяют перебирать ключи и значения словарей и выполнять нужную шаблонизацию на каждой итерации:

```yaml
# values.yaml:
apps:
  backend:
    image: openjdk
  frontend:
    image: node
```

```
# templates/app.yaml:
{{ range $.Values.apps }}
{{ .image }}
{{ end }}
```

Результат:

```
openjdk
node
```

Относительный контекст `.` всегда указывает на значение элемента словаря, соответствующего текущей итерации, при этом указатель можно сохранить и в произвольную переменную:

```
{{ range $app := $.Values.apps }}
{{ $app.image }}
{{ end }}
```

Результат будет таким же:

```
openjdk
node
```

Получить ключ элемента словаря можно так:

```
{{ range $appName, $app := $.Values.apps }}
{{ $appName }}: {{ $app.image }}
{{ end }}
```

Результат:

```yaml
backend: openjdk
frontend: node
```

#### Контроль выполнения цикла

Специальное действие `continue` позволяет пропустить текущую итерацию цикла. В качестве примера пропустим итерацию для элемента `https://example.org`:

```
{{ range $url := $urls }}
{{ if eq $url "https://example.org" }}{{ continue }}{{ end }}
{{ $url }}
{{ end }}
```

Специальное действие `break` позволяет не только пропустить текущую итерацию, но и прервать весь цикл:

```
{{ range $url := $urls }}
{{ if eq $url "https://example.org" }}{{ break }}{{ end }}
{{ $url }}
{{ end }}
```

### Контекст

#### Корневой контекст ($)

Корневой контекст — словарь, на который ссылается переменная `$`. Через него доступны values и некоторые специальные объекты. Корневой контекст имеет глобальную видимость в пределах файла-шаблона (исключение — блок `define` и некоторые функции).

Пример использования:

```
{{ $.Values.mykey }}
```

Результат:

```
myvalue
```

К корневому контексту можно добавлять произвольные ключи/значения, которые также станут доступны из любого места файла-шаблона:

```
{{ $_ := set $ "mykey" "myvalue"}}
{{ $.mykey }}
```

Результат:

```
myvalue
```

Корневой контекст остаётся неизменным даже в блоках, изменяющих относительный контекст (исключение — `define`):

```
{{ with $.Values.backend }}
- command: {{ .command }}
  image: {{ $.Values.werf.image.backend }}
{{ end }}
```

Некоторые функции вроде `tpl` или `include` могут терять корневой контекст. Для сохранения доступа к корневому контексту многим из них можно передать корневой контекст аргументом:

```
{{ tpl "{{ .Values.mykey }}" $ }}
```

Результат:

```
myvalue
```

#### Относительный контекст (.)

Относительный контекст — данные любого типа, на которые ссылается переменная `.`. По умолчанию относительный контекст указывает на корневой контекст.

Некоторые блоки и функции могут менять относительный контекст. В примере ниже в первой строке относительный контекст указывает на корневой контекст `$`, а во второй строке — уже на `$.Values.containers`:

```
{{ range .Values.containers }}
{{ . }}
{{ end }}
```

Для смены относительного контекста можно использовать блок `with`:

```
{{ with $.Values.app }}
image: {{ .image }}
{{ end }}
```

### Переиспользование шаблонов

#### Именованные шаблоны

Для переиспользования шаблонизации объявите *именованные шаблоны* в блоках `define` в файлах `templates/_*.tpl`:

```
# templates/_helpers.tpl:
{{ define "labels" }}
app: myapp
team: alpha
{{ end }}
```

Далее подставляйте именованные шаблоны в файлы `templates/*.(yaml|tpl)` функцией `include`:

```
# templates/deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels: {{ include "labels" nil | nindent 6 }}
  template:
    metadata:
      labels: {{ include "labels" nil | nindent 8 }}
```

Результат:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
      team: alpha
  template:
    metadata:
      labels:
        app: myapp
        team: alpha
```

Имя именованного шаблона для функции `include` может быть динамическим:

```
{{ include (printf "%s.labels" $prefix) nil }}
```

**Именованные шаблоны обладают глобальной видимостью** — единожды объявленный в родительском или любом дочернем чарте именованный шаблон становится доступен сразу во всех чартах — и в родительском, и в дочерних. Убедитесь, что в подключенных родительском и дочерних чартах нет именованных шаблонов с одинаковыми именами.

#### Параметризация именованных шаблонов

Функция `include`, подставляющая именованные шаблоны, принимает один произвольный аргумент. Этот аргумент можно использовать для параметризации именованного шаблона, где этот аргумент станет относительным контекстом `.`:

```
{{ include "labels" "myapp" }}
```

```
{{ define "labels" }}
app: {{ . }}
{{ end }}
```

Результат:

```yaml
app: myapp
```

Для передачи сразу нескольких аргументов используйте список с несколькими аргументами:

```
{{ include "labels" (list "myapp" "alpha") }}
```

```
{{ define "labels" }}
app: {{ index . 0 }}
team: {{ index . 1 }}
{{ end }}
```

... или словарь:

```
{{ include "labels" (dict "app" "myapp" "team" "alpha") }}
```

```
{{ define "labels" }}
app: {{ .app }}
team: {{ .team }}
{{ end }}
```

Необязательные позиционные аргументы можно реализовать так:

```
{{ include "labels" (list "myapp") }}
{{ include "labels" (list "myapp" "alpha") }}
```

```
{{ define "labels" }}
app: {{ index . 0 }}
{{ if gt (len .) 1 }}
team: {{ index . 1 }}
{{ end }}
{{ end }}
```

А необязательные непозиционные аргументы — так:

```
{{ include "labels" (dict "app" "myapp") }}
{{ include "labels" (dict "team" "alpha" "app" "myapp") }}
```

```
{{ define "labels" }}
app: {{ .app }}
{{ if hasKey . "team" }}
team: {{ .team }}
{{ end }}
{{ end }}
```

Именованному шаблону, не требующему параметризации, просто передайте `nil`:

```
{{ include "labels" nil }}
```

#### Результат выполнения include

Функция `include`, подставляющая именованный шаблон, **всегда возвращает только текст**. Для возврата структурированных данных нужно *десериализовать* результат выполнения `include` с помощью функции `fromYaml`:

```
{{ define "commonLabels" }}
app: myapp
{{ end }}
```

```
{{ $labels := include "commonLabels" nil | fromYaml }}
{{ $labels.app }}
```

Результат:

```
myapp
```

> Обратите внимание, что `fromYaml` не работает для списков. Специально для них (и только для них) предназначена функция `fromYamlArray`.

Для явной сериализации данных можно воспользоваться функциями `toYaml` и `toJson`, для десериализации — функциями `fromYaml/fromYamlArray` и `fromJson/fromJsonArray`.

#### Контекст именованных шаблонов

Объявленные в `templates/_*.tpl` именованные шаблоны теряют доступ к корневому и относительному контекстам файла, в который они включаются функцией `include`. Исправить это можно, передав корневой и/или относительный контекст в виде аргументов `include`:

```
{{ include "labels" $ }}
{{ include "labels" . }}
{{ include "labels" (list $ .) }}
{{ include "labels" (list $ . "myapp") }}
```

#### include в include

В блоках `define` тоже можно использовать функцию `include` для включения именованных шаблонов:

```
{{ define "doSomething" }}
{{ include "doSomethingElse" . }}
{{ end }}
```

Через `include` можно вызвать даже тот именованный шаблон, из которого и происходит вызов, т. е. вызвать его рекурсивно:

```
{{ define "doRecursively" }}
{{ if ... }}
{{ include "doRecursively" . }}
{{ end }}
{{ end }}
```

### Шаблонизация с tpl

Функция `tpl` позволяет выполнить шаблонизацию любой строки и тут же получить результат. Она принимает один аргумент, который должен быть корневым контекстом.

Пример шаблонизации values:

```yaml
# values.yaml:
appName: "myapp"
deploymentName: "{{ .Values.appName }}-deployment"
```

```
# templates/app.yaml:
{{ tpl $.Values.deploymentName $ }}
```

Результат:

```
myapp-deployment
```

Пример шаблонизации произвольных файлов, которые сами по себе не поддерживают Helm-шаблонизацию:

```
{{ tpl ($.Files.Get "nginx.conf") $ }}
```

Для передачи дополнительных аргументов в функцию `tpl` можно добавить аргументы как новые ключи корневого контекста:

```
{{ $_ := set $ "myarg" "myvalue"}}
{{ tpl "{{ $.myarg }}" $ }}
```

### Контроль отступов

Используйте функцию `nindent` для выставления отступов:

```
       containers: {{ .Values.app.containers | nindent 6 }}
```

Результат:

```yaml
      containers:
      - name: backend
        image: openjdk
```

Пример комбинации с другими данными:

```
       containers:
       {{ .Values.app.containers | nindent 6 }}
       - name: frontend
         image: node
```

Результат:

```yaml
      containers:
      - name: backend
        image: openjdk
      - name: frontend
        image: node
```

Используйте `-` после `{{` и/или до `}}` для удаления лишних пробелов до и/или после результата выполнения действия, например:

```
  {{- "hello" -}} {{ "world" }}
```

Результат:

```
helloworld
```

### Комментарии

Поддерживаются два типа комментариев — комментарии шаблонизации `{{ /* */ }}` и комментарии манифестов `#`.

#### Комментарии шаблонизации

Комментарии шаблонизации скрываются при формировании манифестов:

```
{{ /* Этот комментарий пропадёт */ }}
app: myApp
```

Комментарии могут быть многострочными:

```
{{ /*
Hello
World
/* }}
```

Шаблоны в них игнорируются:

```
{{ /*
{{ print "Эта шаблонизация игнорируется" }}
/* }}
```

#### Комментарии манифестов

Комментарии манифестов сохраняются при формировании манифестов:

```yaml
# Этот комментарий сохранится
app: myApp
```

Комментарии могут быть только однострочнными:

```yaml
# Для многострочных комментариев используйте
# несколько однострочных комментариев подряд
```

Шаблоны в них выполняются:

```
# {{ print "Эта шаблонизация выполняется" }}
```

### Отладка

Используйте `d8 dk render`, чтобы полностью сформировать и отобразить конечные Kubernetes-манифесты. Укажите опцию `--debug`, чтобы увидеть манифесты, даже если они не являются корректным YAML.

Отобразить содержимое переменной:

```
output: {{ $appName | toYaml }}
```

Отобразить содержимое переменной-списка или словаря:

```
output: {{ $dictOrList | toYaml | nindent 2 }}
```

Отобразить тип данных у переменной:

```
output: {{ kindOf $myvar }}
```

Отобразить произвольную строку, остановив дальнейшее формирование шаблонов:

```
{{ fail (printf "Тип данных: %s" (kindOf $myvar)) }}
```

## Параметризация шаблонов

### Основы параметризации

Содержимое словаря `$.Values` можно использовать для параметризации шаблонов. Каждый чарт имеет свой словарь `$.Values`. Словарь формируется слиянием параметров, полученных из файлов параметров, опций командной строки и других источников.

Простой пример параметризации через `values.yaml`:

```yaml
# values.yaml:
myparam: myvalue
```

```
# templates/example.yaml:
{{ $.Values.myparam }}
```

Результат:

```yaml
myvalue
```

Более сложный пример:

```yaml
# values.yaml:
myparams:
- value: original
```

```
# templates/example.yaml:
{{ (index $.Values.myparams 0).value }}
```

```
d8 dk render --set myparams[0].value=overriden
```

Результат:

```yaml
overriden
```

### Источники параметров и их приоритет

Словарь `$.Values` формируется объединением параметров из источников параметров в указанном порядке:

1. `values.yaml` текущего чарта.
2. `secret-values.yaml` текущего чарта (только в Deckhouse Delivery Kit).
3. Словарь в `values.yaml` родительского чарта, у которого ключ — алиас или имя текущего чарта.
4. Словарь в `secret-values.yaml` родительского чарта (только в Deckhouse Delivery Kit), у которого ключ — алиас или имя текущего чарта.
5. Файлы параметров из переменной `WERF_VALUES_*`.
6. Файлы параметров из опции `--values`.
7. Файлы секретных параметров из переменной `WERF_SECRET_VALUES_*`.
8. Файлы секретных параметров из опции `--secret-values`.
9. Параметры в set-файлах из переменной `WERF_SET_FILE_*`.
10. Параметры в set-файлах из опции `--set-file`.
11. Параметры из переменной `WERF_SET_STRING_*`.
12. Параметры из опции `--set-string`.
13. Параметры из переменной `WERF_SET_*`.
14. Параметры из опции `--set`.
15. Служебные параметры Deckhouse Delivery Kit.
16. Параметры из директивы `export-values` родительского чарта (только в Deckhouse Delivery Kit).
17. Параметры из директивы `import-values` дочерних чартов.

Правила объединения параметров:

* простые типы данных перезаписываются;

* списки перезаписываются;

* словари объединяются;

* при конфликтах параметры из источников выше по списку перезаписываются параметрами из источников ниже по списку.

### Параметризация чарта

Чарт можно параметризовать через его файл параметров:

```yaml
# values.yaml:
myparam: myvalue
```

```
# templates/example.yaml:
{{ $.Values.myparam }}
```

Результат:

```
myvalue
```

Также добавить/переопределить параметры чарта можно и аргументами командной строки:

```shell
d8 dk render --set myparam=overriden  # или WERF_SET_MYPARAM=myparam=overriden d8 dk render
```

```shell
d8 dk render --set-string myparam=overriden  # или WERF_SET_STRING_MYPARAM=myparam=overriden d8 dk render
```

... или дополнительными файлами параметров:

```yaml
# .helm/values-production.yaml:
myparam: overriden
```

```shell
d8 dk render --values .helm/values-production.yaml  # или WERF_VALUES_PROD=.helm/values-production.yaml d8 dk render
```

... или файлом секретных параметров основного чарта (только в Deckhouse Delivery Kit):

```yaml
# .helm/secret-values.yaml:
myparam: <encrypted>
```

```shell
d8 dk render
```

... или дополнительными файлами секретных параметров основного чарта (только в Deckhouse Delivery Kit):

```yaml
# .helm/secret-values-production.yaml:
myparam: <encrypted>
```

```shell
d8 dk render --secret-values .helm/secret-values-production.yaml  # или WERF_SECRET_VALUES_PROD=.helm/secret-values-production.yaml d8 dk render
```

... или set-файлами:

```
# myparam.txt:
overriden
```

```shell
d8 dk render --set-file myparam=myparam.txt  # или WERF_SET_FILE_PROD=myparam=myparam.txt d8 dk render
```

Результат везде тот же:

```
overriden
```

### Параметризация зависимых чартов

Зависимый чарт можно параметризовать как через его собственный файл параметров, так и через файл параметров родительского чарта.

К примеру, здесь параметры из словаря `mychild` в файле `values.yaml` чарта `myparent` перезаписывают параметры в файле `values.yaml` чарта `mychild`:

```yaml
# Chart.yaml:
name: myparent
dependencies:
- name: mychild
```

```yaml
# values.yaml:
mychild:
  myparam: overriden
```

```yaml
# charts/mychild/values.yaml:
myparam: original
```

```
# charts/mychild/templates/example.yaml:
{{ $.Values.myparam }}
```

Результат:

```
overriden
```

Обратите внимание, что словарь, находящийся в `values.yaml` родительского чарта и содержащий параметры для зависимого чарта, должен иметь в качестве имени `alias` (если есть) или `name` зависимого чарта.

Также добавить/переопределить параметры зависимого чарта можно и аргументами командной строки:

```shell
d8 dk render --set mychild.myparam=overriden  # или WERF_SET_MYPARAM=mychild.myparam=overriden d8 dk render
```

```shell
d8 dk render --set-string mychild.myparam=overriden  # или WERF_SET_STRING_MYPARAM=mychild.myparam=overriden d8 dk render
```

... или дополнительными файлами параметров:

```yaml
# .helm/values-production.yaml:
mychild:
  myparam: overriden
```

```shell
d8 dk render --values .helm/values-production.yaml  # или WERF_VALUES_PROD=.helm/values-production.yaml d8 dk render
```

... или файлом секретных параметров основного чарта (только в Deckhouse Delivery Kit):

```yaml
# .helm/secret-values.yaml:
mychild:
  myparam: <encrypted>
```

```shell
d8 dk render
```

... или дополнительными файлами секретных параметров основного чарта (только в Deckhouse Delivery Kit):

```yaml
# .helm/secret-values-production.yaml:
mychild:
  myparam: <encrypted>
```

```shell
d8 dk render --secret-values .helm/secret-values-production.yaml  # или WERF_SECRET_VALUES_PROD=.helm/secret-values-production.yaml d8 dk render
```

... или set-файлами:

```
# mychild-myparam.txt:
overriden
```

```shell
d8 dk render --set-file mychild.myparam=mychild-myparam.txt  # или WERF_SET_FILE_PROD=mychild.myparam=mychild-myparam.txt d8 dk render
```

... или директивой `export-values` (только в Deckhouse Delivery Kit):

```yaml
# Chart.yaml:
name: myparent
dependencies:
- name: mychild
  export-values:
  - parent: myparam
    child: myparam
```

```yaml
# values.yaml:
myparam: overriden
```

```shell
d8 dk render
```

Результат везде тот же:

```
overriden
```

### Использование параметров зависимого чарта в родительском

Для передачи параметров зависимого чарта в родительский можно использовать  директиву `import-values` в родительском чарте:

```yaml
# Chart.yaml:
name: myparent
dependencies:
- name: mychild
  import-values:
  - child: myparam
    parent: myparam
```

```yaml
# values.yaml:
myparam: original
```

```yaml
# charts/mychild/values.yaml:
myparam: overriden
```

```
# templates/example.yaml:
{{ $.Values.myparam }}
```

Результат:

```
overriden
```

### Глобальные параметры

Параметры чарта доступны только в этом же чарте (и ограниченно доступны в зависимых от него). Один из простых способов получить доступ к параметрам одного чарта в других подключенных чартах — использование глобальных параметров.

**Глобальный параметр имеет глобальную область видимости** — параметр, объявленный в родительском, дочернем или другом подключенном чарте становится доступен *во всех подключенных чартах* по одному и тому же пути:

```yaml
# Chart.yaml:
name: myparent
dependencies:
- name: mychild1
- name: mychild2
```

```yaml
# charts/mychild1/values.yaml:
global:
  myparam: myvalue
```

```
# templates/example.yaml:
myparent: {{ $.Values.global.myparam }}
```

```
# charts/mychild1/templates/example.yaml:
mychild1: {{ $.Values.global.myparam }}
```

```
# charts/mychild2/templates/example.yaml:
mychild2: {{ $.Values.global.myparam }}
```

Результат:

```yaml
myparent: myvalue
---
mychild1: myvalue
---
mychild2: myvalue
```

### Секретные параметры (только в d8 dk)

Для хранения секретных параметров можно использовать файлы секретных параметров, хранящиеся в зашифрованном виде в Git-репозитории.

По умолчанию Deckhouse Delivery Kit пытается найти файл `.helm/secret-values.yaml`, содержащий зашифрованные параметры, и при нахождении файла расшифровывает его и объединяет расшифрованные параметры с остальными:

```yaml
# .helm/values.yaml:
plainParam: plainValue
```

```yaml
# .helm/secret-values.yaml:
secretParam: 1000625c4f1d874f0ab853bf1db4e438ad6f054526e5dcf4fc8c10e551174904e6d0
```

```
{{ $.Values.plainParam }}
{{ $.Values.secretParam }}
```

Результат:

```
plainValue
secretValue
```

#### Работа с файлами секретных параметров

Порядок работы с файлами секретных параметров:

1. Возьмите существующий секретный ключ или создайте новый командой `d8 dk helm secret generate-secret-key`.

2. Сохраните секретный ключ в переменную окружения `WERF_SECRET_KEY`, либо в файлы `<корень Git-репозитория>/.werf_secret_key` или `<домашняя директория>/.werf/global_secret_key`.

3. Командой `d8 dk helm secret values edit .helm/secret-values.yaml` откройте файл секретных параметров и добавьте/измените в нём расшифрованные параметры.

4. Сохраните файл — файл зашифруется и сохранится в зашифрованном виде.

5. Закоммитите в Git добавленный/изменённый файл `.helm/secret-values.yaml`;

6. При дальнейших вызовах Deckhouse Delivery Kit секретный ключ должен быть установлен в вышеупомянутых переменной окружения или файлах, иначе файл секретных параметров не сможет быть расшифрован.

> Имеющий доступ к секретному ключу может расшифровать содержимое файла секретных параметров, поэтому **держите секретный ключ в безопасном месте**!

При использовании файла `<корень Git-репозитория>/.werf_secret_key` обязательно добавьте его в `.gitignore`, чтобы случайно не сохранить его в Git-репозитории.

Многие команды Deckhouse Delivery Kit можно запускать и без указания секретного ключа благодаря опции `--ignore-secret-key`, но в таком случае параметры будут доступны для использования не в расшифрованной форме, а в зашифрованной.

#### Дополнительные файлы секретных параметров

В дополнение к файлу `.helm/secret-values.yaml` можно создавать и использовать дополнительные секретные файлы:

```yaml
# .helm/secret-values-production.yaml:
secret: 1000625c4f1d874f0ab853bf1db4e438ad6f054526e5dcf4fc8c10e551174904e6d0
```

```shell
d8 dk --secret-values .helm/secret-values-production.yaml
```

### Информация о собранных образах (только в Deckhouse Delivery Kit)

Deckhouse Delivery Kit хранит информацию о собранных образах в параметрах `$.Values.werf` основного чарта:

```yaml
werf:
  image:
    # Полный путь к собранному Docker-образу для Deckhouse Delivery Kit-образа "backend":
    backend: example.org/apps/myapp:a243949601ddc3d4133c4d5269ba23ed58cb8b18bf2b64047f35abd2-1598024377816
  # Адрес container registry для собранных образов:
  repo: example.org/apps/myapp
  tag:
    # Тег собранного Docker-образа для Deckhouse Delivery Kit-образа "backend":
    backend: a243949601ddc3d4133c4d5269ba23ed58cb8b18bf2b64047f35abd2-1598024377816
```

Пример использования:

```
image: {{ $.Values.werf.image.backend }}
```

Результат:

```yaml
image: example.org/apps/myapp:a243949601ddc3d4133c4d5269ba23ed58cb8b18bf2b64047f35abd2-1598024377816
```

Для использования `$.Values.werf` в зависимых чартах воспользуйтесь директивой `export-values` (только в Deckhouse Delivery Kit):

```yaml
# .helm/Chart.yaml:
dependencies:
- name: backend
  export-values:
  - parent: werf
    child: werf
```

```
# .helm/charts/backend/templates/example.yaml:
image: {{ $.Values.werf.image.backend }}
```

Результат:

```yaml
image: example.org/apps/myapp:a243949601ddc3d4133c4d5269ba23ed58cb8b18bf2b64047f35abd2-1598024377816
```

### Информация о релизе

Deckhouse Delivery Kit хранит информацию о релизе в свойствах объекта `$.Release`:

```yaml
# Устанавливается ли релиз в первый раз:
IsInstall: true
# Обновляется ли уже существующий релиз:
IsUpgrade: false
# Имя релиза:
Name: myapp-production
# Имя Kubernetes Namespace:
Namespace: myapp-production
# Номер ревизии релиза:
Revision: 1
```

... и в параметрах `$.Values.werf` основного чарта (только в Deckhouse Delivery Kit):

```yaml
werf:
  # Имя Deckhouse Delivery Kit-проекта:
  name: myapp
  # Окружение:
  env: production
```

Пример использования:

```
{{ $.Release.Namespace }}
{{ $.Values.werf.env }}
```

Результат:

```
myapp-production
production
```

Для использования `$.Values.werf` в зависимых чартах воспользуйтесь директивой `export-values` (только в Deckhouse Delivery Kit):

```yaml
# .helm/Chart.yaml:
dependencies:
- name: backend
  export-values:
  - parent: werf
    child: werf
```

```
# .helm/charts/backend/templates/example.yaml:
{{ $.Values.werf.env }}
```

Результат:

```yaml
production
```

### Информация о чарте

Deckhouse Delivery Kit хранит информацию о текущем чарте в объекте `$.Chart`:

```yaml
# Является ли чарт основным:
IsRoot: true

# Содержимое Chart.yaml:
Name: mychart
Version: 1.0.0
Type: library
KubeVersion: "~1.20.3"
AppVersion: "1.0"
Deprecated: false
Icon: https://example.org/mychart-icon.svg
Description: This is My Chart
Home: https://example.org
Sources:
  - https://github.com/my/chart
Keywords:
  - apps
Annotations:
  anyAdditionalInfo: here
Dependencies:
- Name: redis
  Condition: redis.enabled
```

Пример использования:

```
{{ $.Chart.Name }}
```

Результат:

```
mychart
```

### Информация о шаблоне

Deckhouse Delivery Kit хранит информацию о текущем шаблоне в свойствах объекта `$.Template`:

```yaml
# Относительный путь к директории templates чарта:
BasePath: mychart/templates
# Относительный путь к текущему файлу шаблона:
Name: mychart/templates/example.yaml
```

Пример использования:

```
{{ $.Template.Name }}
```

Результат:

```
mychart/templates/example.yaml
```

### Информация о Git-коммите (только в Deckhouse Delivery Kit)

Deckhouse Delivery Kit хранит информацию о Git-коммите, на котором он был запущен, в параметрах `$.Values.werf.commit` основного чарта:

```yaml
werf:
  commit:
    date:
      # Дата Git-коммита, на котором был запущен Deckhouse Delivery Kit (человекочитаемая форма):
      human: 2022-01-21 18:51:39 +0300 +0300
      # Дата Git-коммита, на котором был запущен Deckhouse Delivery Kit (Unix time):
      unix: 1642780299
    # Хэш Git-коммита, на котором был запущен Deckhouse Delivery Kit:
    hash: 1b28e6843a963c5bdb3579f6fc93317cc028051c
```

Пример использования:

```
{{ $.Values.werf.commit.hash }}
```

Результат:

```
1b28e6843a963c5bdb3579f6fc93317cc028051c
```

Для использования `$.Values.werf.commit` в зависимых чартах воспользуйтесь директивой `export-values` (только в Deckhouse Delivery Kit):

```yaml
# .helm/Chart.yaml:
dependencies:
- name: backend
  export-values:
  - parent: werf
    child: werf
```

```
# .helm/charts/backend/templates/example.yaml:
{{ $.Values.werf.commit.hash }}
```

Результат:

```yaml
1b28e6843a963c5bdb3579f6fc93317cc028051c
```

### Информация о возможностях кластера Kubernetes

Deckhouse Delivery Kit предоставляет информацию о возможностях кластера Kubernetes, в который Deckhouse Delivery Kit стал бы применять Kubernetes-манифесты, через свойства объекта `$.Capabilities`:

```yaml
KubeVersion:
  # Полная версия кластера Kubernetes:
  Version: v1.20.0
  # Мажорная версия кластера Kubernetes:
  Major: "1"
  # Минорная версия кластера Kubernetes:
  Minor: "20"
# API, поддерживаемые кластером Kubernetes:
APIVersions:
- apps/v1
- batch/v1
- # ...
```

... и методы объекта `$.Capabilities`:

* `APIVersions.Has <arg>` — поддерживается ли кластером Kubernetes указанное аргументом API (например, `apps/v1`) или ресурс (например, `apps/v1/Deployment`).

Пример использования:

```
{{ $.Capabilities.KubeVersion.Version }}
{{ $.Capabilities.APIVersions.Has "apps/v1" }}
```

Результат:

```
v1.20.0
true
```

## Разные окружения

### Параметризация шаблонов в зависимости от окружения (только в Deckhouse Delivery Kit)

*Окружение* Deckhouse Delivery Kit указывается опцией `--env` (`$WERF_ENV`), либо автоматически выставляется командой `d8 dk ci-env`. Текущее окружение доступно в параметре `$.Values.werf.env` основного чарта.

Окружение Deckhouse Delivery Kit используется при формировании имени релиза и имени Namespace'а, а также может использоваться для параметризации шаблонов:

```yaml
# .helm/values.yaml:
memory:
  staging: 1G
  production: 2G
```

```
# .helm/templates/example.yaml:
memory: {{ index $.Values.memory $.Values.werf.env }}
```

```shell
d8 dk render --env production
```

Результат:

```yaml
memory: 2G
```

Для использования `$.Values.werf.env` в зависимых чартах воспользуйтесь директивой `export-values` (только в Deckhouse Delivery Kit):

```yaml
# .helm/Chart.yaml:
dependencies:
- name: child
  export-values:
  - parent: werf
    child: werf
```

```
# .helm/charts/child/templates/example.yaml:
{{ $.Values.werf.env }}
```

Результат:

```
production
```

### Развертывание в разные Kubernetes Namespace

Имя Kubernetes Namespace для развертываемых ресурсов формируется автоматически (только в Deckhouse Delivery Kit) по специальному шаблону `[[ project ]]-[[ env ]]`, где `[[ project ]]` — имя проекта Deckhouse Delivery Kit, а `[[ env ]]` — имя окружения.

Достаточно изменить окружение Deckhouse Delivery Kit и вместе с ним изменится и Namespace:

```yaml
# werf.yaml:
project: myapp
```

```shell
d8 dk converge --env staging
d8 dk converge --env production
```

Результат: один экземпляр приложения развёрнут в Namespace `myapp-staging`, а второй — в `myapp-production`.

Обратите внимание, что если в манифесте Kubernetes-ресурса явно указан Namespace, то для этого ресурса будет использован именно указанный в нём Namespace.

#### Изменение шаблона имени Namespace (только в Deckhouse Delivery Kit)

Если вас не устраивает специальный шаблон, из которого формируется имя Namespace, вы можете его изменить:

```yaml
# werf.yaml:
project: myapp
deploy:
  namespace: "backend-[[ env ]]"
```

```shell
d8 dk converge --env production
```

Результат: приложение развёрнуто в Namespace `backend-production`.

#### Прямое указание имени Namespace

Вместо формирования имени Namespace по специальному шаблону можно указывать Namespace явно для каждой команды (рекомендуется также изменять и имя релиза):

```shell
d8 dk converge --namespace backend-production --release backend-production
```

Результат: приложение развёрнуто в Namespace `backend-production`.

#### Форматирование имени Namespace

Namespace, сформированный по специальному шаблону или указанный опцией `--namespace`, приводится к формату [RFC 1123 Label Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) автоматически. Отключить автоматическое форматирование можно директивой `deploy.namespaceSlug` файла `werf.yaml`.

Вручную отформатировать любую строку согласно формату RFC 1123 Label Names можно командой `d8 dk slugify -f kubernetes-namespace`.

### Развертывание в разные кластеры Kubernetes

По умолчанию Deckhouse Delivery Kit развертывает Kubernetes-ресурсы в кластер, на который настроена команда `d8 k`. Для развертывания в разные кластеры можно использовать разные kube-контексты единого kube-config файла (по умолчанию — `$HOME/.kube/config`):

```shell
d8 dk converge --kube-context staging  # или $WERF_KUBE_CONTEXT=...
d8 dk converge --kube-context production
```

... или использовать разные kube-config файлы:

```shell
d8 dk converge --kube-config "$HOME/.kube/staging.config"  # или $WERF_KUBE_CONFIG=...
d8 dk converge --kube-config-base64 "$KUBE_PRODUCTION_CONFIG_IN_BASE64"  # или $WERF_KUBE_CONFIG_BASE64=...
```

### Развертывание из-под разных пользователей Kubernetes

По умолчанию Deckhouse Delivery Kit для развертывания использует пользователя Kubernetes, через которого работает команда `d8 k`. Для развертывания из-под разных пользователей используйте разные kube-контексты:

```shell
d8 dk converge --kube-context admin  # или $WERF_KUBE_CONTEXT=...
d8 dk converge --kube-context regular-user
```

## Порядок развертывания

### Стадии развертывания

Развертывание Kubernetes-ресурсов происходит в следующей последовательности:

1. Развертывание `CustomResourceDefinitions` из директорий `crds` подключенных чартов.

2. Развертывание хуков `pre-install`, `pre-upgrade` или `pre-rollback` по одному хуку за раз, от хуков с меньшим весом к большему. Если хук имеет зависимость от внешнего ресурса, то он развернётся только после его готовности.

3. Развертывание основных ресурсов: объединение ресурсов с одинаковым весом в группы (ресурсы без указанного веса имеют вес 0) и развертывание по одной группе за раз, от групп с ресурсами меньшего веса к группам с ресурсами большего веса. Если ресурс в группе имеет зависимость от внешнего ресурса, то она начнёт развертывание только после его готовности.

4. Развертывание хуков `post-install`, `post-upgrade` или `post-rollback` по одному хуку за раз, от хуков с меньшим весом к большему. Если хук имеет зависимость от внешнего ресурса, то он развернётся только после его готовности.

### Развертывание CustomResourceDefinitions

Для развертывания CustomResourceDefinitions поместите CRD-манифесты в нешаблонизируемые файлы `crds/*.yaml` в любом из подключенных чартов. При следующем развертывании эти CRD будут развернуты первыми, а хуки и основные ресурсы будут развернуты только после них.

Пример:

```yaml
# .helm/crds/crontab.yaml:
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
# ...
spec:
  names:
    kind: CronTab
```

```
# .helm/templates/crontab.yaml:
apiVersion: example.org/v1
kind: CronTab
# ...
```

```shell
d8 dk converge
```

Результат: сначала развернут CRD для CronTab-ресурса, а затем развернут сам CronTab-ресурс.

### Изменение порядка развертывания ресурсов (только в Deckhouse Delivery Kit)

По умолчанию Deckhouse Delivery Kit объединяет все основные ресурсы (основные — не являющиеся хуками или CRDs из `crds/*.yaml`) в одну группу, создаёт ресурсы этой группы, а затем отслеживает их готовность.

Создание ресурсов группы происходит в следующем порядке:

- Namespace;
- NetworkPolicy;
- ResourceQuota;
- LimitRange;
- PodSecurityPolicy;
- PodDisruptionBudget;
- ServiceAccount;
- Secret;
- SecretList;
- ConfigMap;
- StorageClass;
- PersistentVolume;
- PersistentVolumeClaim;
- CustomResourceDefinition;
- ClusterRole;
- ClusterRoleList;
- ClusterRoleBinding;
- ClusterRoleBindingList;
- Role;
- RoleList;
- RoleBinding;
- RoleBindingList;
- Service;
- DaemonSet;
- Pod;
- ReplicationController;
- ReplicaSet;
- Deployment;
- HorizontalPodAutoscaler;
- StatefulSet;
- Job;
- CronJob;
- Ingress;
- APIService.

Отслеживание готовности включается для всех ресурсов группы одновременно сразу после создания *всех* ресурсов группы.

Для изменения порядка развертывания ресурсов можно создать *новые группы ресурсов* через задание ресурсам *веса*, отличного от веса по умолчанию `0`. Все ресурсы с одинаковым весом объединяются в группы, а затем группы ресурсов развертываются по очереди, от группы с меньшим весом к большему, например:

```
# .helm/templates/example.yaml:
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  annotations:
    werf.io/weight: "-1"
# ...
---
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migrations
# ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  annotations:
    werf.io/weight: "1"
# ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  annotations:
    werf.io/weight: "1"
# ...
```

```shell
d8 dk converge
```

Результат: сначала был развернут ресурс `database`, затем — `database-migrations`, а затем параллельно развернулись `app1` и `app2`.

### Запуск задач перед/после установки, обновления, отката или удаления релиза

Для развертывания определенных ресурсов только перед или после установки, обновления, отката или удаления релиза преобразуйте ресурс в *хук* аннотацией `helm.sh/hook`, например:

```
# .helm/templates/example.yaml:
apiVersion: batch/v1
kind: Job
metadata:
  name: database-initialization
  annotations:
    helm.sh/hook: pre-install
# ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
```

```shell
d8 dk converge
```

Результат: ресурс `database-initialization` будет развернут только при первой *установке* релиза, а ресурс `myapp` будет развертываться и при установке, и при обновлении, и при откате релиза.

Аннотация `helm.sh/hook` объявляет ресурс-хуком и указывает, при каких условиях этот ресурс должен развертываться (можно указать несколько условий через запятую). Возможные условия для развертывания хука:

* `pre-install` — при установке релиза до установки основных ресурсов;

* `pre-upgrade` — при обновлении релиза до обновления основных ресурсов;

* `pre-rollback` — при откате релиза до отката основных ресурсов;

* `pre-delete` — при удалении релиза до удаления основных ресурсов;

* `post-install` — при установке релиза после установки основных ресурсов;

* `post-upgrade` — при обновлении релиза после обновления основных ресурсов;

* `post-rollback` — при откате релиза после отката основных ресурсов;

* `post-delete` — при удалении релиза после удаления основных ресурсов.

Для задания хукам порядка развертывания присвойте им разные *веса* (по умолчанию — `0`), чтобы хуки развертывались по очереди, от хука с меньшим весом к большему, например:

```
# .helm/templates/example.yaml:
apiVersion: batch/v1
kind: Job
metadata:
  name: first
  annotations:
    helm.sh/hook: pre-install
    helm.sh/hook-weight: "-1"
# ...
---
apiVersion: batch/v1
kind: Job
metadata:
  name: second
  annotations:
    helm.sh/hook: pre-install
# ...
---
apiVersion: batch/v1
kind: Job
metadata:
  name: third
  annotations:
    helm.sh/hook: pre-install
    helm.sh/hook-weight: "1"
# ...
```

```shell
d8 dk converge
```

Результат: сначала будет развернут хук `first`, затем хук `second`, затем хук `third`.

**По умолчанию при повторных развертываниях того же самого хука старый хук в кластере удаляется прямо перед развертыванием нового хука.** Этап удаления старого хука можно изменить аннотацией `helm.sh/hook-delete-policy`, которая принимает следующие значения:

- `hook-succeeded` — удалять новый хук сразу после его удачного развертывания, при неудачном развертывании не удалять совсем;

- `hook-failed` — удалять новый хук сразу после его неудачного развертывания, при удачном развертывании не удалять совсем;

- `before-hook-creation` — (по умолчанию) удалять старый хук сразу перед созданием нового.

### Ожидание готовности ресурсов, не принадлежащих релизу (только в Deckhouse Delivery Kit)

Развертываемым в текущем релизе ресурсам могут требоваться ресурсы, которые не принадлежат текущему релизу. Deckhouse Delivery Kit может дожидаться готовности этих внешних ресурсов благодаря аннотации `<name>.external-dependency.werf.io/resource`, например:

```
# .helm/templates/example.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    secret.external-dependency.werf.io/resource: secret/my-dynamic-vault-secret
# ...
```

```shell
d8 dk converge
```

Результат: Deployment `myapp` начнёт развертывание только после того, как Secret `my-dynamic-vault-secret`, создаваемый автоматически оператором в кластере, будет создан и готов.

А так можно ожидать готовности сразу нескольких внешних ресурсов:

```
# .helm/templates/example.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    secret.external-dependency.werf.io/resource: secret/my-dynamic-vault-secret
    database.external-dependency.werf.io/resource: statefulset/my-database
# ...
```

По умолчанию Deckhouse Delivery Kit ищет внешний ресурс в Namespace релиза (если, конечно, ресурс не кластерный). Namespace внешнего ресурса можно изменить аннотацией `<name>.external-dependency.werf.io/namespace`:

```
# .helm/templates/example.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    secret.external-dependency.werf.io/resource: secret/my-dynamic-vault-secret
    secret.external-dependency.werf.io/namespace: my-namespace 
```

*Обратите внимание, что ожидать готовность внешнего ресурса будут и все другие ресурсы релиза с тем же весом, так как ресурсы объединяются по весу в группы и развертываются именно группами.*

## Сценарии развертывания

### Обычное развертывание

Обычно развертывание осуществляется командой `d8 dk converge`, которая собирает образы и развертывает приложение, но требует запуска из Git-репозитория приложения. Пример:

```shell
d8 dk converge --repo example.org/mycompany/myapp
```

Если требуется разделить шаги сборки и развертывания, то это можно сделать так:

```shell
d8 dk build --repo example.org/mycompany/myapp
```

```shell
d8 dk converge --require-built-images --repo example.org/mycompany/myapp
```

### Развертывание с использованием произвольных тегов образов

По умолчанию собранные образы получают тег на основе их содержимого, который становится доступен в Values для их дальнейшего использования в шаблонах при развертывании. Но если возникает необходимость тегировать образы иным тегом, то можно использовать параметр `--use-custom-tag`, например:

```shell
d8 dk converge --use-custom-tag '%image%-v1.0.0' --repo example.org/mycompany/myapp
```

Результат: образы были собраны и опубликованы с тегами `<имя image>-v1.0.0`, после чего теги этих образов стали доступны в Values, на основе которых были сформированы и применены конечные манифесты Kubernetes.

В имени тега, указываемом в параметре `--use-custom-tag`, можно использовать шаблоны `%image%`, `%image_slug%` и `%image_safe_slug%` для подставления имени образа и `%image_content_based_tag%` для подставления оригинального тега на основе содержимого.

> Обратите внимание, что при указании произвольного тега публикуется также и образ с тегом на основе содержимого. В дальнейшем при вызове `d8 dk cleanup` образ с тегом на основе содержимого и образы с произвольными тегами удаляются вместе.

Если требуется разделить шаги сборки и развертывания, то это можно сделать так:

```shell
d8 dk build --add-custom-tag '%image%-v1.0.0' --repo example.org/mycompany/myapp
```

```shell
d8 dk converge --require-built-images --use-custom-tag '%image%-v1.0.0' --repo example.org/mycompany/myapp
```

### Развертывание без доступа к Git-репозиторию приложения

Если нужно развернуть приложение без доступа к Git-репозиторию приложения, то необходимо выполнить три шага:

1. Сборка образов и их публикация в container registry.

2. Добавление переданных параметров и публикация основного чарта в OCI-репозиторий. Чарт содержит указатели на опубликованные в первом шаге образы.

3. Применение опубликованного бандла в кластер.

Первые два шага выполняются командой `d8 dk bundle publish`, находясь в Git-репозитории приложения, например:

```shell
d8 dk bundle publish --tag latest --repo example.org/mycompany/myapp
```

А третий шаг выполняется командой `d8 dk bundle apply` уже без необходимости находиться в Git-репозитории приложения, например:

```shell
d8 dk bundle apply --tag latest --release myapp --namespace myapp-production --repo example.org/mycompany/myapp
```

Конечный результат будет тот же самый, что и при использовании `d8 dk converge`.

Если требуется разделить первый и второй шаг, то это можно сделать так:

```shell
d8 dk build --repo example.org/mycompany/myapp
```

```
d8 dk bundle publish --require-built-images --tag latest --repo example.org/mycompany/myapp
```

### Развертывание без доступа к Git-репозиторию и container registry приложения

Если нужно развернуть приложение без доступа к Git-репозиторию приложения и без доступа к container registry приложения, то необходимо выполнить пять шагов:

1. Сборка образов и их публикация в container registry приложения.

2. Добавление переданных параметров и публикация основного чарта в OCI-репозиторий. Чарт содержит указатели на опубликованные в первом шаге образы.

3. Экспорт бандла и связанных с ним образов в локальный архив.

4. Импорт заархивированного бандла и его образов в container registry, доступный из Kubernetes-кластера, используемого для развертывания.

5. Применение в кластер бандла, опубликованного в новом container registry.

Первые два шага выполняются командой `d8 dk bundle publish`, находясь в Git-репозитории приложения, например:

```shell
d8 dk bundle publish --tag latest --repo example.org/mycompany/myapp
```

Третий шаг выполняется командой `d8 dk bundle copy` уже без необходимости находиться в Git-репозитории приложения, например:

```shell
d8 dk bundle copy --from example.org/mycompany/myapp:latest --to archive:myapp-latest.tar.gz
```

Теперь полученный локальный архив `myapp-latest.tar.gz` переносится удобным способом туда, откуда имеется доступ в container registry, используемый для развертывания в Kubernetes-кластер, и снова выполняется команда `d8 dk bundle copy`, например:

```shell
d8 dk bundle copy --from archive:myapp-latest.tar.gz --to registry.internal/mycompany/myapp:latest
```

В результате чарт и связанные с ним образы опубликуются в новый container registry, к которому из Kubernetes-кластера уже есть доступ. Осталось только развернуть опубликованный бандл в кластер командой `d8 dk bundle apply`, например:

```shell
d8 dk bundle apply --tag latest --release myapp --namespace myapp-production --repo registry.internal/mycompany/myapp
```

На этом шаге уже не требуется доступ ни в Git-репозиторий приложения, ни в его первоначальный container registry. Конечный результат развертывания бандла будет тот же самый, что и при использовании `d8 dk converge`.

Если требуется разделить первый и второй шаг, то это можно сделать так:

```shell
d8 dk build --repo example.org/mycompany/myapp
```

```
d8 dk bundle publish --require-built-images --tag latest --repo example.org/mycompany/myapp
```

### Развертывание сторонним инструментом

Если нужно выполнить применение конечных манифестов приложения не с Deckhouse Delivery Kit, а с использованием другого инструмента (kubectl, Helm, ...), то необходимо выполнить три шага:

1. Сборка образов и их публикация в container registry.

2. Формирование конечных манифестов.

3. Развертывание получившихся манифестов в кластер, используя сторонний инструмент.

Первые два шага выполняются командой `d8 dk render`, находясь в Git-репозитории приложения:

```shell
d8 dk render --output manifests.yaml --repo example.org/mycompany/myapp
```

Теперь полученные манифесты можно передать в сторонний инструмент для дальнейшего развертывания, например:

```shell
kubectl apply -f manifests.yaml
```

> Обратите внимание, что некоторые специальные возможности Deckhouse Delivery Kit вроде возможности изменения порядка развертывания ресурсов на основании их веса (аннотация `werf.io/weight`) скорее всего не будут поддерживаться при применении манифестов сторонним инструментом.

Если требуется разделить первый и второй шаг, то это можно сделать так:

```shell
d8 dk build --repo example.org/mycompany/myapp
```

```
d8 dk render --require-built-images --output manifests.yaml --repo example.org/mycompany/myapp
```

### Развертывание сторонним инструментом без доступа к Git-репозиторию приложения

Если нужно выполнить применение конечных манифестов приложения не с Deckhouse Delivery Kit, а с использованием другого инструмента (kubectl, Helm, ...), при этом не имея доступа к Git-репозиторию приложения, то необходимо выполнить три шага:

1. Сборка образов и их публикация в container registry.

2. Добавление переданных параметров и публикация основного чарта в OCI-репозиторий. Чарт содержит указатели на опубликованные в первом шаге образы.

3. Формирование из бандла конечных манифестов.

4. Развертывание получившихся манифестов в кластер используя сторонний инструмент.

Первые два шага выполняются командой `d8 dk bundle publish`, находясь в Git-репозитории приложения:

```shell
d8 dk bundle publish --tag latest --repo example.org/mycompany/myapp
```

А третий шаг выполняется командой `d8 dk bundle render` уже без необходимости находиться в Git-репозитории приложения, например:

```shell
d8 dk bundle render --output manifests.yaml --tag latest --release myapp --namespace myapp-production --repo example.org/mycompany/myapp
```

Теперь полученные манифесты можно передать в сторонний инструмент для дальнейшего развертывания, например:

```shell
kubectl apply -f manifests.yaml
```

> Обратите внимание, что некоторые специальные возможности Deckhouse Delivery Kit, вроде возможности изменения порядка развертывания ресурсов на основании их веса (аннотация `werf.io/weight`), скорее всего не будут поддерживаться при применении манифестов сторонним инструментом.

Если требуется разделить первый и второй шаг, то это можно сделать так:

```shell
d8 dk build --repo example.org/mycompany/myapp
```

```
d8 dk bundle publish --require-built-images --tag latest --repo example.org/mycompany/myapp
```

### Сохранение отчета о развертывании

Команды `d8 dk converge` и `d8 dk bundle apply` имеют параметр `--save-deploy-report`, который позволяет сохранить отчёт о последнем развертывании в файл. Отчёт содержит имя релиза, Namespace, статус развертывания и ряд других данных. Пример:

```shell
d8 dk converge --save-deploy-report
```

Результат: после развертывания появится файл `.werf-deploy-report.json`, содержащий информацию о последнем релизе.

Путь к отчёту о развертывании можно изменить параметром `--deploy-report-path`.

### Удаление развернутого приложения

Удалить развернутое приложение можно командой `d8 dk dismiss`, запущенной из Git-репозитория приложения, например:

```shell
d8 dk dismiss --env staging
```

При отсутствии доступа к Git-репозиторию приложения можно явно указать имя релиза и Namespace:

```shell
d8 dk dismiss --release myapp-staging --namespace myapp-staging
```

... или использовать отчёт о предыдущем развертывании, включаемый опцией `--save-deploy-report` у `d8 dk converge` и `d8 dk bundle apply`, который содержит имя релиза и Namespace:

```shell
d8 dk converge --save-deploy-report
cp .werf-deploy-report.json /anywhere
cd /anywhere
d8 dk dismiss --use-deploy-report
```

Путь к отчёту о развертывании можно изменить параметром `--deploy-report-path`.

## Отслеживание ресурсов

### Отслеживание состояния ресурсов

Развертывание ресурсов делится на две стадии: применение ресурсов в кластер и *отслеживание состояния* этих ресурсов. Deckhouse Delivery Kit реализует продвинутое отслеживание состояния ресурсов (только в Deckhouse Delivery Kit) благодаря библиотеке [kubedog](https://github.com/werf/kubedog).

Отслеживание ресурсов включено по умолчанию для всех поддерживаемых ресурсов, а именно для:

* всех ресурсов релиза;

* некоторых ресурсов, опосредованно создаваемых ресурсами релиза;

* ресурсов вне релиза, указанных в аннотациях `<name>.external-dependency.werf.io/resource`.

Для ресурсов Deployment, StatefulSet, DaemonSet, Job и Flagger Canary задействуются *специальные* отслеживатели состояния, которые не только точно определяют, удачно или неудачно ресурс был развернут, но и отслеживают состояние дочерних ресурсов, таких как Pod'ы, создаваемые Deployment'ом.

Для остальных ресурсов, не имеющих *специальных* отслеживателей состояния, задействуется *универсальный* отслеживатель, который *предполагает* удачность развертывания ресурса на основании доступной в кластере информации о ресурсе. В редких случаях, если универсальный отслеживатель ошибается в своих предположениях, отслеживание для этого ресурса можно отключить.

#### Изменение критериев неудачного развертывания ресурса (только в Deckhouse Delivery Kit)

По умолчанию Deckhouse Delivery Kit прерывает развертывание и помечает его как неудачное, если произошло более двух ошибок при развертывании одного из ресурсов.

Изменить максимальное количество ошибок развертывания для ресурса можно аннотацией `werf.io/failures-allowed-per-replica`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/failures-allowed-per-replica: "5"
```

Если ресурс имеет аннотацию `werf.io/fail-mode: HopeUntilEndOfDeployProcess`, то ошибки его развертывания будут учитываться только после того, как все остальные ресурсы удачно развернутся.

А помеченный аннотацией `werf.io/track-termination-mode: NonBlocking` ресурс будет отслеживаться только пока все остальные ресурсы не будут развернуты, после чего этот ресурс автоматически посчитается развернутым, даже если это не так.

Универсальный отслеживатель *отсутствие активности* у ресурса в течение 4 минут считает ошибкой развертывания. Изменить этот период можно аннотацией `werf.io/no-activity-timeout`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/no-activity-timeout: 10m
```

#### Отключение отслеживания состояния и игнорирование ошибок ресурса (только в Deckhouse Delivery Kit)

Для отключения отслеживания состояния ресурса и игнорирования ошибок его развертывания пометьте ресурс аннотациями `werf.io/fail-mode: IgnoreAndContinueDeployProcess` и `werf.io/track-termination-mode: NonBlocking`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/fail-mode: IgnoreAndContinueDeployProcess
    werf.io/track-termination-mode: NonBlocking
```

### Отображение логов контейнеров (только в Deckhouse Delivery Kit)

Благодаря библиотеке [kubedog](https://github.com/werf/kubedog) werf автоматически отображает логи контейнеров, создаваемых при развертывании Deployment, StatefulSet, DaemonSet и Job.

Выключить отображение логов для ресурса можно аннотацией `werf.io/skip-logs: "true"`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/skip-logs: "true"
```

А в аннотации `werf.io/show-logs-only-for-containers` можно явно перечислить контейнеры, логи которых следует отображать, в то же время скрыв логи всех остальных контейнеров, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/show-logs-only-for-containers: "backend,frontend"
```

... или наоборот — в аннотации `werf.io/skip-logs-for-containers` перечислить контейнеры, логи которых *не* следует отображать, в то же время отображая логи всех остальных контейнеров, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/skip-logs-for-containers: "sidecar"
```

Для отображения только тех строк лога, которые соответствуют регулярному выражению, используйте аннотацию `werf.io/log-regex`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/log-regex: ".*ERROR.*"
```

Возможно отфильтровать строки лога согласно регулярному выражению не для всех контейнеров сразу, а только для определённого контейнера, если использовать аннотацию `werf.io/log-regex-for-<имя контейнера>`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/log-regex-for-backend: ".*ERROR.*"
```

### Отображение Events ресурсов (только в Deckhouse Delivery Kit)

Благодаря библиотеке [kubedog](https://github.com/werf/kubedog) werf может отображать Events отслеживаемых ресурсов, если ресурс имеет аннотацию `werf.io/show-service-messages: "true"`, например:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    werf.io/show-service-messages: "true"
```

## Управление релизами

### О релизах

Результатом развертывания является *релиз* — совокупность развернутых в кластере ресурсов и служебной информации.

Технически релизы Deckhouse Delivery Kit являются релизами Helm 3 и полностью с ними совместимы. Служебная информация по умолчанию хранится в специальном Secret-ресурсе.

### Автоматическое формирование имени релиза (только в Deckhouse Delivery Kit)

По умолчанию имя релиза формируется автоматически по специальному шаблону `[[ project ]]-[[ env ]]`, где `project` — имя проекта Deckhouse Delivery Kit, а `env` — имя окружения, например:

```yaml
# werf.yaml:
project: myapp
```

```shell
d8 dk converge --env staging
d8 dk converge --env production
```

Результат: созданы релизы `myapp-staging` и `myapp-production`.

### Изменение шаблона имени релиза (только в Deckhouse Delivery Kit)

Если вас не устраивает специальный шаблон, из которого формируется имя релиза, вы можете его изменить:

```yaml
# werf.yaml:
project: myapp
deploy:
  helmRelease: "backend-[[ env ]]"
```

```shell
d8 dk converge --env production
```

Результат: создан релиз `backend-production`.

### Прямое указание имени релиза

Вместо формирования имени релиза по специальному шаблону можно указывать имя релиза явно для каждой команды:

```shell
d8 dk converge --release backend-production  # или $WERF_RELEASE=...
```

Результат: создан релиз `backend-production`.

### Форматирование имени релиза

Имя релиза, сформированное по специальному шаблону или указанное опцией `--release`, приводится к формату [RFC 1123 Label Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) автоматически. Отключить автоматическое форматирование можно директивой `deploy.helmReleaseSlug` файла `werf.yaml`.

Вручную отформатировать любую строку согласно формату RFC 1123 Label Names можно командой `d8 dk slugify -f helm-release`.

### Добавление в релиз уже существующих в кластере ресурсов

Deckhouse Delivery Kit не позволяет развернуть новый ресурс релиза поверх уже существующего в кластере ресурса, если ресурс в кластере *не является частью текущего релиза*. Такое поведение предотвращает случайные обновления ресурсов, принадлежащих другому релизу или развернутых без Deckhouse Delivery Kit. Если все же попытаться это сделать, то отобразится следующая ошибка:

```
Error: helm upgrade have failed: UPGRADE FAILED: rendered manifests contain a resource that already exists...
```

Чтобы добавить ресурс в кластере в текущий релиз и разрешить его обновление, выставьте ресурсу в кластере аннотации `meta.helm.sh/release-name: <имя текущего релиза>`, `meta.helm.sh/release-namespace: <Namespace текущего релиза>` и лейбл `app.kubernetes.io/managed-by: Helm`, например:

```shell
kubectl annotate deploy/myapp meta.helm.sh/release-name=myapp-production
kubectl annotate deploy/myapp meta.helm.sh/release-namespace=myapp-production
kubectl label deploy/myapp app.kubernetes.io/managed-by=Helm
```

... после чего перезапустите развертывание:

```shell
d8 dk converge
```

Результат: ресурс релиза `myapp` успешно обновил ресурс `myapp` в кластере и теперь ресурс в кластере является частью текущего релиза.

### Автоматическое аннотирование выкатываемых ресурсов релиза

Deckhouse Delivery Kit автоматически выставляет следующие аннотации всем ресурсам чарта в процессе развёртывания:

* `"werf.io/version": FULL_WERF_VERSION` — версия Deckhouse Delivery Kit, использованная в процессе запуска команды `d8 dk converge`;
* `"project.werf.io/name": PROJECT_NAME` — имя проекта, указанное в файле конфигурации `werf.yaml`;
* `"project.werf.io/env": ENV` — имя окружения, указанное с помощью параметра `--env` или переменной окружения `WERF_ENV` (аннотация не устанавливается, если окружение не было указано при запуске).

При использовании команды `d8 dk ci-env` с поддерживаемыми CI/CD системами добавляются аннотации, которые позволяют пользователю перейти в связанный пайплайн, задание и коммит при необходимости.

### Добавление произвольных аннотаций и лейблов для выкатываемых ресурсов релиза

Пользователь может устанавливать произвольные аннотации и лейблы используя CLI-параметры при развёртывании `--add-annotation annoName=annoValue` (может быть указан несколько раз) и `--add-label labelName=labelValue` (может быть указан несколько раз). Аннотации и лейблы так же могут быть заданы с помощью соответствующих переменных `WERF_ADD_LABEL*` и `WERF_ADD_ANNOTATION*` (к примеру, `WERF_ADD_ANNOTATION_1=annoName1=annoValue1` и `WERF_ADD_LABEL_1=labelName1=labelValue1`).

Например, для установки аннотаций и лейблов `commit-sha=9aeee03d607c1eed133166159fbea3bad5365c57`, `gitlab-user-email=vasya@myproject.com` всем ресурсам Kubernetes в чарте, можно использовать следующий вызов команды деплоя:

```shell
d8 dk converge \
  --add-annotation "commit-sha=9aeee03d607c1eed133166159fbea3bad5365c57" \
  --add-label "commit-sha=9aeee03d607c1eed133166159fbea3bad5365c57" \
  --add-annotation "gitlab-user-email=vasya@myproject.com" \
  --add-label "gitlab-user-email=vasya@myproject.com" \
  --env dev \
  --repo REPO
```

