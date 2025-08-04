---
title: Установка
weight: 20
menuTitle: Deckhouse Delivery Kit
---

{{< alert level="warning" >}}
Функциональность Deckhouse Delivery Kit доступна только если у вас есть лицензия на любую коммерческую версию Deckhouse Kubernetes Platform.
{{< /alert >}}

## Установка Deckhouse CLI

### Linux

Требования:
* Bash
* Git версии 2.18.0 или выше
* GPG
* Docker Engine (инструкции по установке для [РЕД ОС](https://redos.red-soft.ru/base/server-configuring/container/docker-install/), [Astra Linux](https://wiki.astralinux.ru/pages/viewpage.action?pageId=158601444), [ALT Linux](https://www.altlinux.org/Docker))

Скачайте и установите Deckhouse CLI:
```shell
curl -L "https://deckhouse.ru/downloads/deckhouse-cli/v0.9.1/d8-v0.9.1-linux-amd64.tar.gz" | tar xvfz -
sudo install -t /usr/local/bin/ linux-amd64/d8
```

Если нужны сборки образов для платформ, отличных от Linux x64, то выполните:
```shell
docker run --restart=always --name=qemu-user-static -d --privileged --entrypoint=/bin/sh multiarch/qemu-user-static -c "/register --reset -p yes && tail -f /dev/null"
```

Убедитесь, что установка прошла удачно:
```shell
d8 dk --help
```

### macOS

Требования:
* Bash
* Git версии 2.18.0 или выше
* GPG
* Docker Engine

Скачайте и установите Deckhouse CLI:
```shell
arch=$([ "$(uname -p)" = "x86_64" ] && echo amd64 || echo arm64)
curl -L "https://deckhouse.ru/downloads/deckhouse-cli/v0.9.1/d8-v0.9.1-darwin-$arch.tar.gz" | tar xvfz -
sudo install -t /usr/local/bin/ darwin-$arch/d8
```

Убедитесь, что установка прошла удачно:
```shell
d8 dk --help
```
