---
layout: post
title:  "Блокирование окна инспектора хоткеем"
date:   2022-01-15 00:00:00 +0000
categories: jekyll update
---
Иногда хочется закрепить окно инспектора, чтобы заполнить сразу несколько полей в компонентах, например. Но кнопка, отвечающая за это, слишком маленькая, ее неудобно искать, а потом нужно ее еще раз найти чтобы разблокировать окно. Гораздо удобнее это было бы сделать с помощью хоткея. 

В Unity можно добавить любую статическую функцию в пункт меню, делается это с помощью атрибута `[MenuItem("Menu name/Submeny/Menu point %l")]`

Одноуровневые меню (`[MenuItem("Menu point %l")]`) не допускаются.

Также к этим пунктам можно привязать хоткеи в формате специальный символ + буква. %l означает ctrl + l.

Список символов:

- % ctrl
- \# shift
- & alt
- _ отсутствие спецсимвола

Также можно привязать к специальным кнопкам с помощью ключевых слов: LEFT, RIGHT, UP, DOWN, F1 … F12, HOME, END, PGUP, PGDN

# Стандартный Hello World

Создадим в любом удобном месте скрипт. Например, в Assets\Scripts\EditorTools\Hello.cs

```csharp
public static class Hello
{
    [MenuItem("Hello/World %&h")]
    private static void HelloWorld()
    {
        Debug.Log("Hello World!");
    }
}
```

Теперь при выборе соответствующего пункта в меню или при нажатии Ctrl + Alt + H в консоль будет выводиться "Hello World!".

# Блок окна инспектора

Сразу весь код:

```csharp
[MenuItem("UI Helpers/Block Inspector %l")]
private static void ToggleInspectorLock()
{
    Type inspectorWindowType = Assembly.GetAssembly(typeof(Editor)).GetType("UnityEditor.InspectorWindow");

    var foundWindows = Resources.FindObjectsOfTypeAll(inspectorWindowType);
    if(foundWindows.Length == 0)
    {
        Debug.LogError("Cant find inspector window. Please check that inspector window is open");
        return;
    }

    var editorWindow = (EditorWindow)foundWindows.FirstOrDefault(window => ((EditorWindow)window).titleContent.text.Equals("Inspector"));
    if (editorWindow == null)
    {
        Debug.LogError("Cant find inspector window. Please check that inspector window is open");
        return;
    }

    PropertyInfo propertyInfo = inspectorWindowType.GetProperty("isLocked");
    bool isWindowCurrentlyLocked = (bool) propertyInfo.GetValue(editorWindow);
    propertyInfo.SetValue(editorWindow, !isWindowCurrentlyLocked, null);
    editorWindow.Repaint();
}
```

Теперь разберем его по частям. 

```csharp
Type inspectorWindowType = Assembly.GetAssembly(typeof(Editor)).GetType("UnityEditor.InspectorWindow");
```

Для того чтобы получить окно инспектора мы будем использовать рефлексию. А для использования рефлексии нам нужен класс окна инспектора. В саму рефлексию я углубляться не буду, это тема для другого поста (или даже блога).

Чтобы получить классы других распространенных типов окна, можно использовать следующие строки (взял [отсюда](https://forum.unity.com/threads/opening-the-built-in-windows-inspector-scene-etc-without-hard-coding-a-menu-path.546617/#post-3607555)):

+ Project: UnityEditor.ProjectBrowser
+ Hierarchy: UnityEditor.SceneHierarchyWindow
+ Scene: UnityEditor.SceneView
+ Game: UnityEditor.GameView
+ Console: UnityEditor.ConsoleWindow


```csharp
var foundWindows = Resources.FindObjectsOfTypeAll(inspectorWindowType);
if(foundWindows.Length == 0)
{
    Debug.LogError("Cant find inspector window. Please check that inspector window is open");
    return;
}
```

Теперь, когда у нас есть класс инспектора, мы можем попробовать найти нужное окно. В этом коде мы получаем все открытые на данный момент окна инспектора. Их может быть несколько, если просто открыть два окна инспектора в юнити, или сделать свое кастомное окно инспектора.

```csharp
var editorWindow = (EditorWindow)foundWindows.FirstOrDefault(window => ((EditorWindow)window).titleContent.text.Equals("Inspector"));
if (editorWindow == null)
{
    Debug.LogError("Cant find inspector window. Please check that inspector window is open");
    return;
}
```

Здесь мы получаем непосредственно окно инспектора. Я добавил дополнительную проверку, что тайтл окна соответствует стандартному, но, в целом, это необязательно и можно использовать просто `(EditorWindow)foundWindows.FirstOrDefault();`. Проверка добавлена на случай, если Resources.FindObjectsOfTypeAll() вернет больше одного окна, или если будет открыто какое-нибудь кастомное окно инспектора, а стандартное будет закрыто.

```csharp
PropertyInfo propertyInfo = inspectorWindowType.GetProperty("isLocked");
bool isWindowCurrentlyLocked = (bool) propertyInfo.GetValue(editorWindow);
propertyInfo.SetValue(editorWindow, !isWindowCurrentlyLocked, null);
```

Смотрим, в каком сейчас состоянии окно и, если заблокировано, то снимаем блок, и наоборот.

```csharp
editorWindow.Repaint();
```

Заново рендерим окно. Нужно, чтобы изменения на нем отобразились, без этого кнопка замка, отображающая статус блокировки, не будет менять свой статус.

Дополнительные ссылки:
- [Unity - Scripting API: MenuItem](https://docs.unity3d.com/ScriptReference/MenuItem.html)
- [Unity - Scripting API: MenuItem Constructor](https://docs.unity3d.com/ScriptReference/MenuItem-ctor.html)

<small>Проверено на Unity 2020.3.12.f1.</small>