---
layout: post
title: Ковариантность и контравариантность
date:   2023-03-29 00:00:00 +0000
categories: jekyll update
---
<small>
💬 В этом посте я хочу рассмотреть вариантность - термин, который я долго понимал максимально поверхностно. Здесь много допущений, и я игнорирую существование теории категорий, поэтому возможны неточности или не совсем корректные сравнения. Если вы обнаружите фактические ошибки - напишите мне, я исправлю.
</small>

Можно определить термины ковариантности, контравариантности и инвариантности под общим словом "вариантность". Вариантность - это свойство. Свойство функций в более широком смысле, чем в программировании.

Можно определить их как

- Ковариантность: свойство, при котором сохраняется иерархия у аргументов
- Контравариантность: свойство, при котором инвертируется иерархия аргументов.
- Инвариантность: отсутствие ковариантности и контравариантности.

## Пример с целыми числами

Если функция возрастает (для нее выполняется условие `(x ≤ y) → (f(x) ≤ f(y))`) на всей области определения, то она ковариантная. Если функция убывает (для нее выполняется условие `(x ≤ y) → (f(y) ≤ f(x))`), то она контравариантная. В любом другом случае можно сказать, что она инвариантная. 

Вот примеры функций на каждый вариант
- `D(x) = x + x`
    - Это ковариантная функция
- `N(x) = 0 - x`
    - Это контравариантная функция
- `S(x) = x * x`
    - Эта функция убывает при х ≤ 0 и растет при x ≥ 0. Она инвариантная.

## Вариантность в C#

Чтобы соотнести эти понятия с программированием, нужно уметь сравнивать типы. Для этого мы можем использовать следующее утверждение: если экземпляр типа X можно безопасно присвоить переменной типа Y, то X ≤ Y, и мы говорим, что X совместим по назначению с Y.

### Ковариантность массивов

Предположим, у нас есть следующая иерархия классов:

```csharp
public class Enemy { }
public class Undead : Enemy { }
public class Zombie : Undead { }
public class Orc : Enemy { }
```

Мы можем создать массив из типа и положить в него массив из подтипа.

```csharp
Enemy[] enemies = new Undead[5];
```

Можно спросить, почему это вообще обладает какой-либо вариантностью, тут же нет никакой функции? Но функция тут есть в том самом более широком смысле. Можем ее определить как `f(T) → T[]`, функция создания нового типа: создания массивов конкретных типов. `Enemy` и `Enemy[]` - это два разных типа! При этом мы нигде не описывали `Enemy[]` и сам класс `Array` ничего не знает непосредственно про `Enemy`. 

Можем проверить ковариантность с помощью нотации из примера про числа.

```
Наша новая функция должна удовлетворять (x ≤ y) → (f(x) ≤ f(y))
Функцию мы уже определили: f(T) → T[].

Тогда:
x = Undead
y = Enemy

f(x) = Undead[]
f(y) = Enemy[]

x ≤ y
Undead ≤ Enemy // Приведение типов
TRUE

f(x) ≤ f(y)
Undead[] ≤ Enemy[] // Ковариантность массивов
TRUE

Вывод f(T) → T[] - ковариантна по типу T
```

Стоит обратить внимание, что `Undead[]` не наследуется от `Enemy[]`, здесь реализуется именно ковариантность массивов, а не приведение типов, в то время как в аналогичном примере, но без массивов

```csharp
Enemy enemy = new Undead();
```

нет ковариантности, просто приведение типов.

Из-за ковариантности массивов при их использовании возможна ошибка:

```csharp
Enemy[] enemies = new Undead[5];
enemies[0] = new Orc(); // Ошибка рантайма System.ArrayTypeMismatchException
```

Именно поэтому класс List инвариантен:

```csharp
List<Enemy> enemies = new List<Undead>();
// Ошибка компиляции:
// Cannot convert initializer type 'List<Undead>' to target type 'List<Enemy>'
```

### Вариантность при конвертации группы методов в делегаты.

Конвертация группы методов ковариантна по возвращаемому типу и контравариантна по их аргументам. 

Сначала разберемся с контравариантностью аргументов.

Создадим делегат и 2 метода:

```csharp
public delegate void Delegate(Undead undead);

public static void UndeadProcessor(Undead undead) {}
public static void ZombieProcessor(Zombie zombie) {}
```

Теперь проверим делегаты по той же схеме, по которой проверяли массивы:

```
Наша новая функция должна удовлетворять (x ≤ y) → (f(y) ≤ f(x))
Определим нашу функцию как: f(T) → Delegate(Method(T)).

Тогда:
x = Zombie
y = Undead

f(x) = Delegate(Method(Zombie))
f(y) = Delegate(Method(Undead))

x ≤ y
Zombie ≤ Undead // Приведение типов
TRUE

f(y) ≤ f(x)
Delegate(Method(Undead)) ≤ Delegate(Method(Zombie))
Мы не можем напрямую сравнить делегаты.
Но мы можем сравнить их косвенно через конструктор делегата, принимающего в себя Method(T):
new Delegate(Method(Undead)); // OK
new Delegate(Method(Zombie)); // Compile Error
⇒
Delegate(Method(Undead)) ≤ constructor
Delegate(Method(Zombie)) ≥ constructor
⇒
Delegate(Method(Undead)) ≤ Delegate(Method(Zombie))

Вывод: f(T) → Delegate(Method(T)) - контравариантна по типу T
```

Теперь ковариантность возвращаемого типа. Сделаем примерно то же самое. 

```csharp
public delegate Zombie Delegate();

public static Zombie ZombieCreator() { return new Zombie(); }
public static Undead UndeadCreator() { return new Undead(); }
```

Проведем проверку

```
Наша новая функция должна удовлетворять (x ≤ y) → (f(x) ≤ f(y))
Определим нашу функцию как: f(T) → Delegate(Method(): T).

Тогда:
x = Zombie
y = Undead

f(x) = Delegate(Method(): Zombie)
f(y) = Delegate(Method(): Undead)

x ≤ y
Zombie ≤ Undead // Приведение типов
TRUE

f(y) ≤ f(x)
new Delegate(Method(): Zombie); // OK
new Delegate(Method(): Undead); // Compile Error
⇒
Delegate(Method(): Zombie) ≤ Delegate(Method(): Undead)

Вывод: f(T) → Delegate(Method(): T) - ковариантна по типу T
```

Один из возникающих вопросов после прочтения этой части это:

#### Зачем нужна контравариантность в аргументах делегата?

Добавим возможность зомби заражать тех, кого он ударил

```csharp
public class Zombie : Undead
{
    public void Infect(Object target)
    {
        Console.WriteLine($"{target} is now infected!");
    }
}
```

Напишем такой хэндлер на атаку от зомби:

```csharp
private void ZombieHitHandler(Zombie zombie)
{
    zombie.Infect(this);
}
```

Теперь попробуем подписаться на хэндлер 

```csharp
UndeadHitEvent undeadHitEvent = new UndeadHitEvent(ZombieHitHandler); // Ошибка!
```

И закономерно получим ошибку. Потому что никто не запретит вызвать этот делегат **не** с зомби:

```csharp
undeadHitEvent.Invoke(new Undead());
```

Стоит обратить внимание, что контравариантность здесь работает только при конверсии метода (или группы методов) в делегат! Для аргументов делегата все так же работает приведение типов.

### Вариантность в дженериках.

Этот вид вариантности можно использовать в интерфейсах и делегатах.

Когда мы создаем параметр типа, мы можем явно указать, хотим ли мы, чтобы он был ковариантным (**`out`**), контравариантным(**`in`**) или, если ничего не указывать, инвариантным.

```csharp
public interface IVariantInterface<in TInput, out TOut>
{
    TOut SomeAction(TInput input);
}

public interface IInvariantInterface<TInput, TOut>
{
    TOut SomeAction(TInput input);
}
```

Тогда при использовании:

```csharp
// OK
IVariantInterface<Undead, Undead> variant = new Implementation<Enemy, Zombie>(); 

// Ошибка!
IInvariantInterface<Undead, Undead> invariant = new Implementation<Enemy, Zombie>();
```

Стоит обратить внимание, что если параметр используется как аргумент типа, то компилятор не даст объявить его ковариантным, и наоборот, если он используется как возвращаемый тип, то он не может быть контравариантным.

#### Почему возвращаемый тип может быть ковариантным, но не наоборот?

```csharp
// OK
IEnumerable<Zombie> zombies = new List<Zombie>();
IEnumerable<Enemy> enemies = zombies;

// Ошибка!
IList<Zombie> zombiesList = new List<Zombie>();
IList<Enemy> enemiesList = zombies;
```

Первые две строчки компилятор пропускает, они ковариантные. Но IList<T> - инвариантный. Вспоминаем ковариантность массивов и к какой ошибке это может привести. 

В тоже время перечислениям нормально быть ковариантными, поскольку в перечисления не предоставляют интерфейса для добавления новых элементов. Конкретные имплементации перечислений (например, список) - да, позволяют, но они скорее всего будут в свою очередь инвариантными, что обеспечит нам безопасность. Таким образом ковариантность безопасна для возвращаемого типа. 

Но что, если бы возвращаемый тип был бы контравариантным? 

Тогда следующие строчки были бы валидными:

```csharp
IEnumerable<Enemy> enemies = new List<Enemy>();
IEnumerable<Zombie> zombies = enemies;
```

Это уже выглядит некорректно. Не каждый враг может быть зомби. Кроме этого, если мы захотим пройтись по по всем зомби что бы заразить кого-нибудь ими, то можем наткнуться на орка, который не может заражать противников.

#### Почему тип аргумента может быть контравариантным, но не наоборот?

Представим следующий интерфейс и имплементацию:

```csharp
public interface IBlackHole<in T>
{
    void Consume(T obj);
}

public class BlackHole<T> : IBlackHole<T>
{
    private List<T> _consumedObjs = new List<T>();

    public void Consume(T obj)
    {
        _consumedObjs.Add(obj);
    }
}
```

И использование:

```csharp
// OK!
IBlackHole<Enemy> blackHoleWithEnemies = new BlackHole<Enemy>();
IBlackHole<Undead> blackHoleWithUndeads = blackHoleWithEnemies;
blackHoleWithUndeads.Consume(new Undead());

// Ошибка
IBlackHole<Zombie> blackHoleWithZombies = new BlackHole<Zombie>();
IBlackHole<Undead> blackHoleWithUndeads = blackHoleWithZombies;
blackHoleWithUndeads.Consume(new Undead());
```

Сейчас не очень очевидно, в чем может быть ошибка, но если “очистить” получившийся код от интерфейсов и имплементаций то получим

```csharp
// OK!
List<Enemy> _consumedObjs = new List<Enemy>();
_consumedObjs.Add(new Undead());

// Ошибка
List<Zombie> _consumedObjs = new List<Zombie>();
_consumedObjs.Add(new Undead());
```

Таким образом, становится очевидным ошибка. В список супертипа можно добавлять элементы-подтипы, но не наоборот.

Еще один пример:

```csharp
public interface IEquityComparer<in T>
{
    bool Equals(T first, T second);
}

public class ZombieEquityComparer : IEquityComparer<Zombie>
{
    public bool Equals(Zombie first, Zombie second)
    {
        return first.ZeroPatientId == second.ZeroPatientId;
    }
}
```

Опустим вопрос, насколько разумно делать в принципе такие проверки на равенство, нас интересует безопасность с точки зрения типов.

```csharp
IEquityComparer<Zombie> zombiesComparer = new ZombieEquityComparer();
IEquityComparer<Undead> undeadComparer = zombiesComparer;

undeadComparer.Equals(new Undead(), new Undead());
```

Если бы компилятор не выдавал ошибку на второй строке, то мы бы пытались сравнить двух андедов по их ZeroPatientId, который присутствует только у зомби.

### Return ковариантность

C# 9 позволяет использовать ковариантность в возвращаемом типе перегруженной функции:

```csharp
public class Enemy
{
    public virtual Enemy CreateNewEnemy() => 
        new Enemy();
}

public class Undead: Enemy
{
    public override Undead CreateNewEnemy() => 
        new Undead();
}
```

## Резюме

- В C# вариантность в основном представлена в следующих местах:
    - В массивах
    - В конвертации групп методов в делегаты
    - В дженериках
        - В делегатах
        - В интерфейсах
    - В return типах
- Вариантность используется там где сам тип используется как аргумент. Если в качестве аргумента используются экземпляры типа то там, скорей всего, применяется приведение типов.
- В возвращаемом типе можно использовать ковариантность.
- В типе аргумента можно использовать контравариантность.
- Если тип используется и в аргументе и возвращается то придется делать его инвариантным.
- Напоминание: значимые типы всегда являются неявно запечатанными, то есть не могут иметь иерархии. Значит, вариантность имеет смысл только в контексте ссылочных типов.

---

## Ссылки:

- [https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/) - MSDN
- [https://learn.microsoft.com/en-us/archive/blogs/ericlippert/whats-the-difference-between-covariance-and-assignment-compatibility](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/whats-the-difference-between-covariance-and-assignment-compatibility) - отсюда взял сравнение с целыми числами
- [https://learn.microsoft.com/en-us/archive/blogs/ericlippert/covariance-and-contravariance-in-c-part-one](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/covariance-and-contravariance-in-c-part-one) - серия постов одного из разработчиков C# по теме
- [https://tomasp.net/blog/variance-explained.aspx/](https://tomasp.net/blog/variance-explained.aspx/) - про вариантность с точки зрения теории категорий
- C# in Depth, 4-е издание, главы 2.3.3 и 4.3. Введение в вариантность от Джона Скита