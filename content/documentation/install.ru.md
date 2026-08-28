---
title: Установка
weight: 20
menuTitle: Deckhouse Delivery Kit
---

{{< alert level="warning" >}}
Функциональность Deckhouse Delivery Kit доступна только если у вас есть лицензия на любую коммерческую версию Deckhouse Kubernetes Platform.
{{< /alert >}}

## Установка Deckhouse CLI

Убедитесь, что в системе установлены следующие зависимости:

- Bash;
- Git версии 2.18.0 или выше;
- GPG;
- Docker Engine (инструкции по установке для [РЕД ОС](https://redos.red-soft.ru/base/server-configuring/container/docker-install/), [Astra Linux](https://wiki.astralinux.ru/pages/viewpage.action?pageId=158601444), [ALT Linux](https://www.altlinux.org/Docker)).

Установить Deckhouse CLI возможно двумя способами:

* Начиная с версии 0.10 доступна установка с помощью [trdl](https://ru.trdl.dev/). Такой способ позволяет непрерывно получать свежие версии утилиты со всеми доработками и исправлениями.

  {{< alert level="info" >}}
  Для установки через trdl необходим доступ в Интернет к tuf-репозиторию с утилитой. В кластере с закрытым окружением такой способ работать не будет.
  {{< /alert >}}

* Скачав исполняемый файл и установив его вручную.

### Установка с помощью trdl

Начиная с версии 0.10 Deckhouse CLI можно установить с помощью [trdl](https://ru.trdl.dev/).

{{< alert level="warning" >}}
Если у вас установлена версия ниже 0.10, то её необходимо предварительно удалить.

Если вам нужно установить одну из версий ниже 0.10, воспользуйтесь [устаревшим способом установки](#установка-исполняемого-файла).
{{< /alert >}}

1. Установите [клиент trdl](https://ru.trdl.dev/quickstart.html#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D0%BA%D0%BB%D0%B8%D0%B5%D0%BD%D1%82%D0%B0).

1. Добавьте репозиторий Deckhouse CLI в trdl:

   ```bash
   URL=https://deckhouse.ru/downloads/deckhouse-cli-trdl
   ROOT_VERSION=1
   ROOT_SHA512=343bd5f0d8811254e5f0b6fe292372a7b7eda08d276ff255229200f84e58a8151ab2729df3515cb11372dc3899c70df172a4e54c8a596a73d67ae790466a0491
   REPO=d8

   trdl add $REPO $URL $ROOT_VERSION $ROOT_SHA512
   ```

1. Установите последний стабильный выпуск утилиты `d8` и проверьте ее работоспособность:

   ```bash
   . $(trdl use d8 0 stable) && d8 --version
   ```

Если вы не хотите вызывать `. $(trdl use d8 0 stable)` перед каждым использованием Deckhouse CLI, добавьте строку `alias d8='trdl exec d8 0 stable -- "$@"'` в RC-файл вашей командной оболочки.

Готово, вы установили Deckhouse CLI.

### Установка исполняемого файла

{{< tabs name="d8_cli_install" >}}
{{% tab name="Linux x86-64" %}}
{{% d8-cli-install os="Linux" arch="x86-64" %}}
{{% /tab %}}
{{% tab name="macOS x86-64" %}}
{{% d8-cli-install os="macOS" arch="x86-64" %}}
{{% /tab %}}
{{% tab name="macOS ARM64" %}}
{{% d8-cli-install os="macOS" arch="ARM64" %}}
{{% /tab %}}
{{% tab name="Windows" %}}
{{% d8-cli-install os="Windows" arch="x86-64" %}}
{{% /tab %}}
{{< /tabs >}}
