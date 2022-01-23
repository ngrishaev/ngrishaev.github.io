---
layout: post
title:  "Валидация GameObject'ов из меню Unity"
date:   2022-01-15 00:00:00 +0300
categories: jekyll update
---
<small>По материалам курса K-Syndicate: Advanced Unit Testing In Unity [https://lms.k-syndicate.school/unittesting/](https://lms.k-syndicate.school/unittesting/)</small>

В прошлом посте было показано как добавить вызов кастомных методов прямо в Unity. Кроме расширения функционала редактора в такие методы можно добавить работу с непосредственно сценой и объектами на них, или ассетами. В этом посте будет показано как добавить в вызов такого метода валидацию префабов.

Сделаем, для примера, проверку отсутствия пропавших компонентов на объектах. Так же, постараемся сделать его расширяемым. 

Создадим статический класс Validator и запишем в нем первую версию метода:

(Кстати, в названиях пунктов меню можно использовать эмодзи. Так можно выделять часто используемые пункты — глаз быстрее обратит внимание на изображение, чем на текст)

```csharp
public static class Validator
{
	[MenuItem("Tools/🔗 Missing Components")]
	public static void FindMissingComponents()
	{
		Debug.Log("Searching for missing components");

		var scene = SceneManager.GetActiveScene();
		var gameObjectQueue = new Queue<GameObject>(scene.GetRootGameObjects());

		while (gameObjectQueue.Count > 0)
		{
			var gameObject = gameObjectQueue.Dequeue();
			var hasMissingScripts = GameObjectUtility.GetMonoBehavioursWithMissingScriptCount(gameObject) > 0;
			if (hasMissingScripts)
				Debug.LogWarning("🔗", $"Name: {gameObject.name}");

			foreach (Transform child in gameObject.transform)
				gameObjectQueue.Enqueue(child.gameObject);
		}
	}
}
```

Все объекты на сцене находятся в иерархической структуре, которую можно представить в виде дерева с самой сценой в корне. Мы проходимся по всем объектам на сцене и проверяем есть ли на них пропавшие компоненты. И, если есть, выкидываем сообщение об этом в лог.

В целом, цель достигнута, но можно попробовать отрефакторить этот код, что бы его было легче расширять. Описанный алгоритм можно разбить на 2 части: 

- Валидацию объекта на сцене
- Сбор всех объектов

Попробуем переписать этот метод так, что бы его можно было расширить и добавить другие проверки, например не является ли проверяемый объект префабом, который отсутствует в ассетах. Для этого напишем один обобщенный метод проверки объектов и 2 конкретных метода, которые его будут использовать:

```csharp
[MenuItem("Tools/🧱 Missing Prefab")]
public static void FindMissingPrefabs() =>
    ValidateGameObjectsOnScenes(
        PrefabUtility.IsPrefabAssetMissing,
        (scene, gameObject) => Debug.LogError("🧱", $"On scene {scene.name} game object {gameObject.name} has missing prefab"));

[MenuItem("Tools/🔗 Missing Components")]
public static void FindMissingComponents() =>
    ValidateGameObjectsOnScenes(
        go => GameObjectUtility.GetMonoBehavioursWithMissingScriptCount(go) > 0,
        (scene, gameObject) => Debug.LogError("🔗", $"On scene {scene.name} game object {gameObject.name} has missing component"));

private static void ValidateGameObjectsOnScenes(Func<GameObject, bool> validator, Action<Scene, GameObject> callback)
{
    foreach (var scene in GetAllProjectScenesOpened())
    foreach (var gameObject in GetAllGameObjects(scene))
        if (validator(gameObject))
            callback(scene, gameObject);
}

private static IEnumerable<Scene> GetAllProjectScenesOpened()
{
    var scenePaths = AssetDatabase
        .FindAssets("t:Scene", new[] {"Assets"})
        .Select(AssetDatabase.GUIDToAssetPath);

    foreach (var scenePath in scenePaths)
    {
        var currentScene = SceneManager.GetSceneByPath(scenePath);
        if (currentScene.isLoaded)
        {
            yield return currentScene;
        }
        else
        {
            var openedScene = EditorSceneManager.OpenScene(scenePath, OpenSceneMode.Additive);
            yield return openedScene;
            EditorSceneManager.CloseScene(openedScene, true);
        }
    }
}

private static IEnumerable<GameObject> GetAllGameObjects(Scene scene)
{
    var gameObjectQueue = new Queue<GameObject>(scene.GetRootGameObjects());

    while (gameObjectQueue.Count > 0)
    {
        var gameObject = gameObjectQueue.Dequeue();

        yield return gameObject;

        foreach (Transform child in gameObject.transform)
            gameObjectQueue.Enqueue(child.gameObject);
    }
}
```

Основной метод: `ValidateGameObjectsOnScenes`

Он принимает два аргумента — способ валидации GameObject’a и коллбэк, который вызывается если объект невалидный, в коллбэке мы пишем сообщение для метода валидации, у каждого — свое.

Так же мы расширили тест, что бы он проходился по всем сценам в игре, а не только в активной. Стоит обратить внимание, что при переборе сцен мы проверяем, не возвращаем ли мы сейчас активную сцену, что бы не закрыть ее случайно после. 

Теперь мы можем писать сколько угодно проверок для объектов на сценах и нам не нужно думать о сопутствующем коде, только о самих проверках.

Такой способ валидации имеет несколько преимуществ:
* Он прост и его быстро написать, не нужно добавлять проект с тестами. Можно просто добавить static метод и уже писать проверку
* Каждой такой проверке можно дать уникальный хоткей
* Их можно запускать прямо во время запущенной игры

Но одновременно есть и минусы, конечно:
* Не поддерживаемо при написании большого количества проверок — меню становится помойкой, каждый тест надо поддерживать отдельно
* Эти проверки необязательные, ничто не остановит вас запушить изменения в репозиторий, даже если на сцене присутствуют невалидные объекты

Дополнительные ссылки:
- [Прошлый пост по MenuItem]({% post_url 2022-01-15-lock-inspector-in-unity %})
- [Unity - Scripting API: MenuItem](https://docs.unity3d.com/ScriptReference/MenuItem.html)
- [Unity - Scripting API: AssetDatabase](https://docs.unity3d.com/ScriptReference/AssetDatabase.html)
- [Unity - Scripting API: PrefabUtility](https://docs.unity3d.com/ScriptReference/PrefabUtility.html)
- [Unity - Scripting API: GameObjectUtility](https://docs.unity3d.com/ScriptReference/GameObjectUtility.html)


<small>Проверено в Unity 2020.3.12f1</small>