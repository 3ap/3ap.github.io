---
layout: post
title: "Поддержка папок в Jenkins Job Builder"
description: "Как привязать job к определённой папке?"
---

Для того, чтобы в [Jenkins Job Builder (jjb)](http://docs.openstack.org/infra/jenkins-job-builder/)
создать и обновлять job в определённой папке, достаточно прописать в
названии job'а полный путь а-ля:

```yaml
- job:
    name: "developer/zar/buildroot"
```

Таким образом job с названием `buildroot` попадёт в директорию
`developer/zar`.

**Важно**: на данный момент `jjb` не умеет
самостоятельно создавать директории: поддержку этой фичи внедряют уже
несколько лет, а проследить за прогрессом можно в официальном
Gerrit'е создателей `jjb` [вот
тут](https://review.openstack.org/#/c/134307/).
