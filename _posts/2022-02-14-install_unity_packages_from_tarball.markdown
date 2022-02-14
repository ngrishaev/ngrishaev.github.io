---
layout: post
title: Устанавливаем Unity packages из tarball (tar, tgz) файлов
date:   2022-02-14 00:00:00 +0000
categories: jekyll update
---

Некоторые пэкеджи нельзя скачать напрямую из Package Manager. Возможно, вам нужно установить не самую актуальную версию, ну или они просто отсутствуют в UPM (Unity Package Manager). При этом, если у вас достаточно старая версия Unity, у вас не будет опции установить tarball через графический интерфейс. Это можно сделать руками, в файле расположенном по пути `%project_path%/Packages/manifest.json`

После этого нужно добавить в раздел `dependencies` строчку вида 
`"package.name" : "path_to_file"`
Например:
```
{
  "dependencies": {
    "com.google.external-dependency-manager": "file:./GooglePackages/com.google.external-dependency-manager-1.2.15.tgz",
    [...]
  }
}
```
Можно указывать как абсолютный путь, так и относительный, относительно файла `manifest.json`

В некоторых случаях Юнити все равно будет ругаться что не находит файл `package.json`. Тогда нужно распаковать архив и указать путь к папке, внутри которой будет лежать `package.json`

Стоит обратить внимание, что если вы мигрируете с UPM то сначала нужно удалить их руками через графический интерфейс Package Manager.

Так же стоит поступить если вы мигрируете с локальных пекеджей на UPM.

Если устанавливаемые пекеджи много весят, то рекомендуется включить git lfs.

Дополнительные ссылки:
- [Install Google packages for Unity](https://developers.google.com/unity/instructions)
- [Unity's Package Manager](https://docs.unity3d.com/Manual/Packages.html)


<small>Проверено в Unity 2018.4.18f1</small>