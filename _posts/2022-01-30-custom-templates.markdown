---
layout: post
title:  "Создание своего шаблона скрипта в Unity"
date:   2022-01-30 00:00:00 +0000
categories: jekyll update
---
# Как создать свой шаблон скрипта для Unity

В стандартном скрипте, который создает юнити, есть лишние комментарии. Хотелось бы избавиться от них сразу, а не удалять руками каждый раз. Кроме того, заранее прописаны Start и Update, которые, быть может, и не нужны вовсе, или даже в данном конкретном проекте вообще не используются MonoBehaviour, а какой-нибудь другой базовый класс.

Шаблоны хранятся в двух папках:

- `\UnityProject\Assets\ScriptTemplates` - ее нужно создать руками
- `\Unity\Hub\Editor\2020.3.12f1\Editor\Data\Resources\ScriptTemplates` - всегда есть с установленной версией юнити. Очевидно что вместо 2020.3.12f1 может стоять ваша версия юнити.

Туда нужно положить текстовый файл, названный по определенному шаблону:

`nn-TemplateGroup__TemplateName-TemplateFileName.TemplateExtension.txt`

Теперь разберем по частям мною написанное:

- nn - номер, по которому будет сортироваться ваш шаблон в меню создания ассетов
- TemplateGroup - меню, в котором будет храниться пункт создания нового скрипта
- TemplateName - имя пункта в меню, по которому будет создаваться скрипт
- TemplateFileName - имя нового файла по умолчанию
- TemplateExtension - расширение нового файла

Например: `01-C# Templates__ClearMonoBehaviour-NewMono.cs.txt`

Это создаст пункт ClearMonoBehaviour в меню C# Templates, который будет отображаться по возможности первым. При выборе этого пункта создастся файл с названием NewMono.cs. Стоит обратить внимание, что меню и пункт меню разделяются двойным символом _.

Внутрь текстового файла нужно положить, собственно, шаблон. Например:

```
using UnityEngine;

public class #SCRIPTNAME# : MonoBehaviour
{

    private void Start()
    {
#NOTRIM#
    }

}
```

Теперь в новых файлах содержание будет такое:

```csharp
using UnityEngine;

public class NewMono: MonoBehaviour
{

    private void Start()
    {

    }

}
```

В шаблоне так же используются 2 дополнительных параметра:

- #SCRIPTNAME# - вместо этого подставится имя файла скрипта
- #NOTRIM# - нужно что бы убедиться что IDE не удалить пробелы и оставит пустую строчку

Дополнительные ссылки:

- [https://support.unity.com/hc/en-us/articles/210223733-How-to-customize-Unity-script-templates](https://support.unity.com/hc/en-us/articles/210223733-How-to-customize-Unity-script-templates)
- [https://4experience.co/how-to-create-useful-unity-script-templates/](https://4experience.co/how-to-create-useful-unity-script-templates/)

<small>Проверено в Unity 2020.3.12f1</small>