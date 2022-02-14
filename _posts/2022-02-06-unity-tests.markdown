---
layout: post
title:  "Добавляем тесты в Unity проект"
date:   2022-02-06 00:00:00 +0000
categories: jekyll update
---
<small>По материалам курса K-Syndicate: Advanced Unit Testing In Unity [https://lms.k-syndicate.school/unittesting/](https://lms.k-syndicate.school/unittesting/)</small>

В [прошлом посте]({% post_url 2022-01-23-game-objects-validation %}) было показано, как добавить проверки в Unity, которые можно вызвать в ручном режиме. 

Для работы с юнит тестами в Unity уже есть готовое окно. Оно открывается через: Window → General → Test Runner. 

Unity позволяет работать с двумя типами тестов: Play Mode и Edit Mode. 

Собственно, из названия должно быть понятно, какие тесты какими являются:

- EditMode тесты запускаются, не запуская саму игру, и имеют доступ к Editor коду.
- PlayMode тесты запускают игру и работают в ней (но на отдельной сцене). Тесты могут являться корутинами, что позволяет пропускать кадры.

Создадим папку с тестами через кнопку “Create Edit Mode Test Assembly Folder”. Это создаст папку, внутри которой уже сразу будет лежать asmdef файл для проекта с тестами. 

Теперь создадим скрипт с тестами, для этого нажмем кнопку “Create Test Script in current folder”.

Для начала попробуем просто вызывать наш валидатор напрямую из тестов. У нас этого не выйдет, поскольку наша сборка с тестами не видит сборку с кодом. Если посмотреть, на какие сборки ссылаются тесты, то можно увидеть:

- UnityEngine.TestRunner
- UnityEditor.TestRunner
- nunit.framework.dll

Нужно создать сборку для наших скриптов и добавить на нее ссылку в тесты. 

Переместим наш валидатор в папку Tools. Внутри нее создадим новый Assembly Definition файл, который назовем Tools. Теперь сошлемся на него в asmdef файле тестов: Assembly Definition References → “+” → выбираем наш созданный Tools файл.

Теперь в коде тестов можно ссылаться на скрипты из папки Tools.

Теперь можем написать свой первый юнит тест:

```csharp
[Test]
public void MissingComponentsValidation()
{
   Validator.FindMissingComponents();
}
```

На самом деле валидатор нам не нужен. Он нужен был только в рамках данного поста, чтобы показать, как сослаться на код из скриптов. Все, что он умеет можно делать сразу в тестах. Перенесем валидатор в тесты. Вот что получится:

```csharp
public class GameObjectValidationTests
{
   [Test]
   public static void FindMissingPrefabs() =>
      ValidateGameObjectsOnScenes(
         PrefabUtility.IsPrefabAssetMissing,
         (scene, gameObject) => Debug.unityLogger.LogError("🧱", $"On scene {scene.name} game object {gameObject.name} has missing prefab"));

   [Test]
   public static void FindMissingComponents() =>
      ValidateGameObjectsOnScenes(
         go => GameObjectUtility.GetMonoBehavioursWithMissingScriptCount(go) > 0,
         (scene, gameObject) => Debug.unityLogger.LogError("🔗", $"On scene {scene.name} game object {gameObject.name} has missing component"));

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
}

```

Теперь мы можем автоматически запускать эти тесты. В принципе, сейчас можно было бы остановиться. Но у текущих тестов есть несколько недостатков: неудобный вывод ошибок, тест остановится, если найдется хотя бы один невалидный GameObject. Стоит переписать их, чтобы повысить удобство пользования:

```csharp
[Test]
public void MissingComponentsValidation()
{
    var errors =
        from scene in GetAllProjectScenesOpened()
        from gameObject in GetAllGameObjects(scene)
        where GameObjectUtility.GetMonoBehavioursWithMissingScriptCount(gameObject) > 0
        select $"On scene {scene.name} game object {gameObject.name} has missing component";

   Assert.That(errors, Is.Empty);
}

[Test]
public void MissingPrefabsValidation()
{
    var errors =
        from scene in GetAllProjectScenesOpened()
        from gameObject in GetAllGameObjects(scene)
        where PrefabUtility.IsPrefabAssetMissing(gameObject)
        select $"On scene {scene.name} game object {gameObject.name} has missing prefab";

   Assert.That(errors, Is.Empty);
}
```

Плюсы такого подхода:
* Тесты можно добавить в CI и настроить так, чтобы мерж не проходил, если тесты не зеленые
* Если есть тяжелые тесты, то можно их ставить на ночные билды и не тратить время на то, чтобы дождаться их результатов
* Просто покрытые фичами тесты не так страшно рефакторить

Минусы
* Тесты тоже надо поддерживать
* При плохо написанных тестах они скорее раздражают, чем помогают

Дополнительные ссылки:
- [Прошлый пост с ручной валидацией]({% post_url 2022-01-23-game-objects-validation %})
- [NUnit](https://nunit.org/)
- [Unity Test Framework](https://docs.unity3d.com/Packages/com.unity.test-framework@1.1/manual/index.html)
- [Assembley Definitions](https://docs.unity3d.com/Manual/ScriptCompilationAssemblyDefinitionFiles.html)

<small>Проверено в Unity 2020.3.12f1</small>