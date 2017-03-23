# Игровые данные Shapetris

Если вы пропустили прошлые части с [подготовкой проекта](readme.md) и [игровым состоянием](game-state.md) ознакомьтесь с ними перед продолжением чтения.

## Основные компоненты
Основой нашего 3д тетриса является: уровень, фигуры и точки. На самом деле уровень от фигуры мало чем отличается: и первый, и второй по сути просто контейнеры для точек, но для обеспечения игрового процесса нам их нужно различать.

Создадим в файле `Pools.cs` новый интерфейс компонентов пула: `IGameplayComponent` по аналогии с `ICoreComponent`.

Описывать компоненты будем в общем файле: `Gameplay\Components.cs`:
```cs
using System;
using Rentitas;

namespace Shapetris
{
    public class Level : IGameplayComponent, ISingleton, IFlag {}
    public class Piece : IGameplayComponent, IFlag {}

    public class CurrentPiece : IGameplayComponent, ISingleton, IFlag {}
    public class NextPiece : IGameplayComponent, ISingleton, IFlag {}

    public class Container : IGameplayComponent
    {
        public Guid Id;
    }

    public class Dot : IGameplayComponent
    {
        public DotTypes Type;
    }

    public class Owner : IGameplayComponent
    {
        public Guid OwnerId;
    }

    public class Position : IGameplayComponent
    {
        public Vector3i Value;

        public int X => Value.x;
        public int Y => Value.y;
        public int Z => Value.z;
    }

    public class WorldPosition : IGameplayComponent
    {
        public Vector3i Value;

        public int X => Value.x;
        public int Y => Value.y;
        public int Z => Value.z;
    }
}
```

Не ожиданные данные, да? После классического ООП мыслить компонентами очень сложно, но давайте разберем, что за компоненты мы создали и зачем:

- `Dot` — добавляет объекту особенности игровой точки и хранит информацию о ее типе
- `Container` — компонент, добавляющий объекту особенности контейнера `Dot`ов и определяет хранит его уникальный Id
- `Level` и `Piece` — флаги объекта определяющие его тип в пуле
- `Owner` — хранит идентификатор контейнера владеющего точкой 
- `Position`, `WorldPosition` — вполне ожидаемые типы данных

Вы наверно спросите, а где массив точек? Действительно, можно было бы в контейнере создать массив структур `Dot`, а не делать из этого компонент для объектов пула. И конечно же мы бы выиграли в производительности. 

Но мы поступим хитрее, потеряем незначительные доли процента производительности при получении из коллекции (храниться связи будут в индексе построенном на `Dictionary`), но получим реактивность при изменении точек: добавлять на них бонусы, разбивать фигуру, выкидывать из фигуры точки.. Да, что душе угодно с ними делать - они же полноценные объекты! А самое главное быстрое и удобный расчет коллизий. Но давайте по порядку. 

Давайте для начала, по аналогии с `ApplicationState`, опишем методы взаимодействия с игровыми данными. Создаем файл `Gameplay\Extensions\GameplayPoolExtensions.cs`:

<details>
<summary style="background-color: red"><b>GameplayPoolExtensions.cs</b></summary>
<p>

```cs
using System;
using System.Collections.Generic;
using Rentitas;

namespace Shapetris
{
    public static class GameplayPoolExtensions
    {
        #region getters
        public static Entity<IGameplayComponent> GetActivePiece(this Pool<IGameplayComponent> pool)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetLevel(this Pool<IGameplayComponent> pool)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> container)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
            this Pool<IGameplayComponent> pool,
            Guid owner)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetOwner(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetOwner(
            this Pool<IGameplayComponent> pool,
            Guid ownerId)
        {
            throw new NotImplementedException("Not implemented yet...");
        }

        public static bool IsLevelDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static bool IsPieceDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetDotAtPosition(
            this Pool<IGameplayComponent> pool,
            int x, int y, int z)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetDotAtPosition(
            this Pool<IGameplayComponent> pool,
            int x, int y)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        #endregion

        #region constructors
        public static Entity<IGameplayComponent> CreatePiece(
            this Pool<IGameplayComponent> pool)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> CreateLevel(
            this Pool<IGameplayComponent> pool,
            int width, int height)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> CreateDot(
            this Pool<IGameplayComponent> pool,
            DotTypes type,
            int x, int y, int z)
        {
            throw new NotImplementedException("Not implemented yet..");
        }
        #endregion
    }
}
```
</p></details>


Cразу опишем тесты в `Shapetris.Tests/GameplayPoolTests.cs`:


<details>
<summary style="background-color: red"><b>GameplayPoolTests.cs</b></summary>
<p>

```cs
using System.Linq;
using NUnit.Framework;
using Rentitas;

namespace Shapetris.Tests
{
    [TestFixture]
    public class GameplayPoolTests
    {
        private Pool<IGameplayComponent> pool;

        [Test]
        public void Should_create_level()
        {
            var level = pool.CreateLevel(10, 10);
            Assert.NotNull(level);
            Assert.IsTrue(level.Is<Level>());
        }

        [Test]
        public void Level_should_have_a_container()
        {
            var level = pool.CreateLevel(10, 10);
            Assert.IsTrue(level.Has<Container>());
        }

        [Test]
        public void Should_create_piece()
        {
            var piece = pool.CreatePiece();
            Assert.NotNull(piece);
            Assert.IsTrue(piece.Is<Piece>());
        }

        [Test]
        public void Piece_should_have_a_container()
        {
            var piece = pool.CreatePiece();
            Assert.IsTrue(piece.Has<Container>());
        }

        [Test]
        public void Should_create_dot()
        {
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0));
            Assert.NotNull(dot);
            Assert.IsTrue(dot.Has<Dot>());
        }

        [Test]
        public void Should_return_current_piece()
        {
            var piece = pool.CreatePiece();
            pool.SetPieceAsCurrent(piece);
            
            Assert.AreSame(pool.GetActivePiece(), piece);
        }

        [Test]
        public void Should_return_current_level()
        {
            var level = pool.CreateLevel(10, 10);
            pool.SetLevelAsCurrent(level);
            Assert.AreSame(pool.GetLevel(), level);
        }

        [Test]
        public void Should_return_position()
        {
            var level = pool.CreateLevel(10, 10);
            var piece = pool.CreatePiece();
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0));

            Assert.IsTrue(level.Has<Position>());
            Assert.IsTrue(piece.Has<Position>());
            Assert.IsTrue(dot.Has<Position>());

            ComparePosition(level.Get<Position>().Value, 0, 0, 0);
            ComparePosition(piece.Get<Position>().Value, 0, 0, 0);
            ComparePosition(dot.Get<Position>().Value, 2, 1, 3);
        }

        private void ComparePosition(Vector3i position, int x, int y, int z)
        {
            Assert.AreEqual(x, position.x);
            Assert.AreEqual(y, position.y);
            Assert.AreEqual(z, position.z);
        }

        [Test]
        public void Should_return_dot_world_position()
        {
            var piece = pool.CreatePiece();
            piece.Replace<Position>(p => p.Value = new Vector3i(10, 10, 10));

            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);

            ComparePosition(pool.GetWorldPosition(dot), 15, 14, 13);
        }

        [Test]
        public void Sould_return_dot_in_world_position()
        {
            var piece = pool.CreatePiece();
            piece.Replace<Position>(p => p.Value = new Vector3i(10, 10, 10));
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);

            var dotAtPos = pool.GetDotAtPosition(new Vector3i(15, 14, 13));

            Assert.AreSame(dot, dotAtPos);
        }

        [Test]
        public void Should_return_dots_inside_container()
        {
            var piece = pool.CreatePiece();
            var beforeCreation = pool.GetContainerDots(piece);
            Assert.AreEqual(0, beforeCreation.Count());

            for (int index = 0; index < 3; index++)
            {
                pool.CreateDot(DotTypes.Blue, new Vector3i(index, index, 0)).OwnedBy(piece);
            }

            var afterCreation = pool.GetContainerDots(piece);
            Assert.AreEqual(3, afterCreation.Count());
        }

        [Test]
        public void Should_return_dot_owner()
        {
            var piece = pool.CreatePiece();
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);

            Assert.AreSame(piece, pool.GetOwner(dot));
        }

        [Test]
        public void Should_receive_correct_retaining_state()
        {
            var piece = pool.CreatePiece();
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0));
            Assert.IsFalse(pool.IsRetainDot(dot));
            dot.OwnedBy(piece);
            Assert.IsTrue(pool.IsRetainDot(dot));
        }

        [Test]
        public void Should_be_piece_dot()
        {
            var piece = pool.CreatePiece();
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);
            Assert.IsTrue(pool.IsPieceDot(dot));
        }

        [Test]
        public void Should_be_level_dot()
        {
            var level = pool.CreateLevel(3, 3);
            var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(level);
            Assert.IsTrue(pool.IsLevelDot(dot));
        }
    }
}
```
</p>
</details>

Ключевое отличие от `ApplicationState` в том, что мы не сможем реализовать методы взаимодействия без дополнительной работы на пулом.


## Кастомный пул и индексы

Чтобы реализовать методы `GetDotAtPosition` и `GetContainerDots` необходим инструмент получения объекта(ов) в соответствии со значениями их компонентов. Для этого в Rentitas существуют индексы. Давайте создадим класс кастомного пула `GameplayPool` в `Gameplay\GameplayPool.cs`:

```cs
using System;
using Rentitas;

namespace Shapetris
{
    /// <summary>
    /// Special pool for gameplay entities. Contains useful indexes to access fast as possible
    /// </summary>
    public class GameplayPool : Pool<IGameplayComponent>
    {
        /// <summary>
        /// Index of containers in pool
        /// </summary>
        public PrimaryEntityIndex<IGameplayComponent, Guid> ContainersIndex
        {
            get; private set;
        }

        /// <summary>
        /// Index of dots based on their world positions
        /// </summary>
        public PrimaryEntityIndex<IGameplayComponent, Vector3i> InWorldDots
        {
            get; private set;
        }

        public GameplayPool() : base(RentitasUtility.CollectComponents<IGameplayComponent>().Build())
        {
            ContainersIndex = new PrimaryEntityIndex<IGameplayComponent, Guid>(
                    GetGroup(Matcher.AllOf(typeof(Container))),
                    (entity, component) => (component as Container).Id);

            InWorldDots = new PrimaryEntityIndex<IGameplayComponent, Vector3i>(
                    GetGroup(Matcher.AllOf(typeof(WorldPosition))),
                    (entity, component) => (component as WorldPosition).Value);
        }
    }
}
```

Мы потихоньку подходим к самой интересной части Rentitas — реактивности компонентов. Давайте разберемся, что такое индексы в общем и `PrimaryEntityIndex<TPool, TKey>` в частности. Взглянем на методы взаимодействия с экземпляром класса (мне кажется они говорят сами за себя): 
- `GetEntity<TKey>(TKey key)` — возвращает объект из индекса с ключом `key` (или выбросывает исключение)
- `HasEntity<TKey>(TKey key)` — проверяет индекс на наличие объекта с ключом `key`
- `TryGetEntity<TKey>(TKey key)` — возвращает объект из индекса с ключом `key` или `null` если такой не найдем

Выглядит как key-value коллекция, чем по сути и является. Только содержанием коллекции зависит от изменений компонентов внутри пула. При инициализации индекса мы передаем первым аргументом группу\наблюдателя, а вторым лямба-функцию объясняющую индексу как извлечь из события изменения группы ключ. 

Особенность `PrimaryEntityIndex` в том, что под одним ключом в пуле может находиться исключительно один объект. Для коллекций объектов (например, все синии фигуры) мы будем использовать `EntityIndex`.

Далее мы перейдем к работе с индексами и расчетам `WorldPosition` в первой системе (единственное место в Rentitas где по соглашению можно вызывать изменения компонентов). Но перед этим, я реализую все методы пула, использующие уже знакомый нам функционал Rentitas:

```cs
using System;
using System.Collections.Generic;
using Rentitas;

namespace Shapetris
{
    public static class GameplayPoolExtensions
    {
        #region getters
        public static Entity<IGameplayComponent> GetActivePiece(this Pool<IGameplayComponent> pool)
        {
            return pool.GetSingle<CurrentPiece>();
        }

        public static Entity<IGameplayComponent> GetLevel(this Pool<IGameplayComponent> pool)
        {
            return pool.GetSingle<Level>();
        }

        public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> container)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
            this Pool<IGameplayComponent> pool,
            Guid owner)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetOwner(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetOwner(
            this Pool<IGameplayComponent> pool,
            Guid ownerId)
        {
            throw new NotImplementedException("Not implemented yet...");
        }

        public static bool IsRetainDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
        {
            return dot.Has<Owner>();
        }

        public static bool IsLevelDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static bool IsPieceDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetDotAtPosition(
            this Pool<IGameplayComponent> pool,
            Vector3i position)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Entity<IGameplayComponent> GetDotAtPosition(
            this Pool<IGameplayComponent> pool,
            int x, int y)
        {
            throw new NotImplementedException("Not implemented yet..");
        }

        public static Vector3i GetWorldPosition(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> dot)
        {
            return dot.Get<WorldPosition>().Value;
        }

        #endregion

        #region constructors
        public static Entity<IGameplayComponent> CreatePiece(
            this Pool<IGameplayComponent> pool,
            int width, int height, int depth)
        {
            var pieceEntity = pool.CreateContainer();
            pieceEntity.Toggle<Piece>(true);
            return pieceEntity;
        }

        public static Entity<IGameplayComponent> CreateLevel(
            this Pool<IGameplayComponent> pool,
            int width, int height)
        {
            // Get current level or create new one
            var levelEntity = pool.Is<Level>()
                ? pool.GetSingle<Level>()
                : pool.CreateContainer();

            levelEntity.Toggle<Level>(true);

            return levelEntity;
        }

        private static Entity<IGameplayComponent> CreateContainer(
            this Pool<IGameplayComponent> pool)
        {
            var containerEntity = pool.CreateEntity();
            var containerComponent = containerEntity.CreateComponent<Container>();
            var containerPosition = containerEntity.CreateComponent<Position>();

            containerComponent.Id = Guid.NewGuid();

            containerEntity.AddInstance(containerComponent).AddInstance(containerPosition);

            return containerEntity;
        }

        public static Entity<IGameplayComponent> CreateDot(
            this Pool<IGameplayComponent> pool,
            DotTypes type, Vector3i position)
        {
            var dotEntity = pool.CreateEntity();
            var dotComponent = dotEntity.CreateComponent<Dot>();
            var dotPosition = dotEntity.CreateComponent<Position>();

            dotComponent.Type = type;
            dotPosition.Value = position;

            return dotEntity.AddInstance(dotComponent).AddInstance(dotPosition);
        }
        #endregion

        #region setters
        public static Entity<IGameplayComponent> SetPieceAsCurrent(
            this Pool<IGameplayComponent> pool,
            Entity<IGameplayComponent> piece)
        {
            if (pool.Is<CurrentPiece>())
            {
                pool.GetSingle<CurrentPiece>().Toggle<CurrentPiece>(false);
            }

            piece.Toggle<CurrentPiece>(true);
            return piece;
        }

        public static Entity<IGameplayComponent> AttachDot(
            this Entity<IGameplayComponent> container,
            Entity<IGameplayComponent> dot)
        {
            var ownerComponent = dot.Need<Owner>();
            ownerComponent.OwnerId = container.Get<Container>().Id;

            dot.ReplaceInstance(ownerComponent);

            return container;
        }

        public static Entity<IGameplayComponent> OwnedBy(
            this Entity<IGameplayComponent> dot,
            Entity<IGameplayComponent> container)
        {
            var ownerComponent = dot.Need<Owner>();
            ownerComponent.OwnerId = container.Get<Container>().Id;
            return dot.ReplaceInstance(ownerComponent);
        }
        #endregion
    }
}
```

Добавить `SetUp` метод в тесты, чтобы создать экземпляр `GameplayPool`а перед стартом тестов и проверим их работоспособность. Успешно пройдены — 9 из 15! Отлично. Переходим к работе с индексами. 

Сначала разберемся с функционалом связанным с владением контейнерами точек, начнем с первого метода расширения `GetContainerDots`.

```cs
public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
    this Pool<IGameplayComponent> pool,
    Entity<IGameplayComponent> container)
{
    return pool.GetContainerDots(container.Get<Container>().Id);
}

public static IEnumerable<Entity<IGameplayComponent>> GetContainerDots(
    this Pool<IGameplayComponent> pool,
    Guid owner)
{
    return ((GameplayPool)pool).ContainerDotsIndex.GetEntities(owner);
}
```

Запускаем тест `Should_return_dots_inside_container` и убеждаемся, что наш индекс работает. С индексами вообще все очень просто: изменили компонент, изменилась коллекция.  

> Почему в методе создания индекса использую: `(component as Owner)`, а не `entity.Get<Owner>()`? Дело в том, что данный метод вызывается в двух случаях: при изменении\создании компонента `Owner` и при удалении его. В первом случае `entity.Get<Owner>()` вернет экземпляр компонента, но во втором выбросит исключение: `EntityDoesNotHaveComponentException`. 

По аналогии допишем оставшиеся методы:

```cs
public static Entity<IGameplayComponent> GetOwner(
    this Pool<IGameplayComponent> pool,
    Entity<IGameplayComponent> dot)
{
    return pool.GetOwner(dot.Get<Owner>().OwnerId);
}

public static Entity<IGameplayComponent> GetOwner(
    this Pool<IGameplayComponent> pool,
    Guid ownerId)
{
    return ((GameplayPool) pool).ContainersIndex.GetEntity(ownerId);
}

public static bool IsLevelDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
{
    return pool.GetOwner(dot).Is<Level>();
}

public static bool IsPieceDot(this Pool<IGameplayComponent> pool, Entity<IGameplayComponent> dot)
{
    return pool.GetOwner(dot).Is<Piece>();
}

public static Entity<IGameplayComponent> GetDotAtPosition(
    this Pool<IGameplayComponent> pool,
    Vector3i position)
{
    return ((GameplayPool) pool).InWorldDotsIndex.TryGetEntity(position);
}

public static Entity<IGameplayComponent> GetDotAtPosition(
    this Pool<IGameplayComponent> pool,
    int x, int y)
{
    return pool.GetDotAtPosition(new Vector3i(x, y, 0));
}

public static Vector3i GetWorldPosition(
    this Pool<IGameplayComponent> pool,
    Entity<IGameplayComponent> dot)
{
    return dot.Get<WorldPosition>().Value;
}
```

Запускаем тесты и как ожидалось — 13 из 15. Осталось только с `WorldPosition` разобраться.

## Реактивные системы Rentitas и первая игровая логика

Давайте разберемся с подходом к написанию игровой логики в фреймворке Rentitas. Добро пожаловать во главу [Базовая игровая логика](base-logic.md)


