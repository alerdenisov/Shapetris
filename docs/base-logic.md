# Игровая логика в Rentitas

Если вы пропустили прошлые части ознакомьтесь с ними перед продолжением чтения:
- [Подготовка проекта](readme.md)
- [Создание игрового состояния](game-state.md) 
- [Игровые данные](game-data.md)

## Системы Rentitas
Вся логика в Rentitas описывается в системах. Системы — это очень простые классы выполняющие одну задачу в определенный момент времени. В Rentitas предусмотрено пять видов систем:

- `IInitializeSystem` — выполняется при старте сценария (`scenario.Initialize()`)
- `IDeinitializeSystem` — выполняется при завершении сценария (`scenario.Deinitialize()`)
- `IExecuteSystem` — выполняется каждый тик сценария (`scenario.Execute()`)
- `IReactiveSystem<TPool>` — выполнятся при изменении отслеживаемой группы
- `ICleanupSystem` — выполняется после завершения тика сценария

Системы собираются в сценарии и выполняются в той очередности в которой добавлены в сценарий:

```cs
var scenario = new Scenario("Example scenario")
    .Add(pool.CreateSystem(new WillExecuteFirst()))
    .Add(pool.CreateSystem(new WillExecuteSecond()))
    .Add(pool.CreateSystem(new WillExecuteThird()))
    //...
    
    .Add(pool.CreateSystem(new WillExecuteAtTheEnd()));
```

Для вычисления `WorldPosition` мы будем использовать `IReactiveSystem`. `IReactiveSystem` — наиболее сложная в понимании система и когда вы разберетесь с ней — все остальные будут для вас очевидны.

Создадим систему в `Gameplay\Systems\CalculateWorldPositionSystem.cs`:

```cs
using System.Collections.Generic;
using Rentitas;

namespace Shapetris.Systems
{
    public class CalculateWorldPositionSystem : IReactiveSystem<IGameplayComponent>
    {
        public void Execute(List<Entity<IGameplayComponent>> entities)
        {
            throw new System.NotImplementedException();
        }

        public TriggerOnEvent Trigger { get; }
    }
}
```

Интерфейс `IReactiveSystem` требует реализовать метод `Execute` и свойство `Trigger`. Метод будет вызываться один раз за тик и принимать коллекцию измененных объектов. Триггер же определяет какие объекты нас интересуют:

```cs
public TriggerOnEvent Trigger => Matcher.AllOf(
        typeof(Owner),
        typeof(Position))
    .OnEntityAdded();
```

Мы указали, что нас интересуют все объекты пула с компонентами: `Position` и `Owner` в момент когда один из указанных компонентов изменится или добавится. 

Чтобы посчитать мировую позицию нам потребуется позиция контейнера. У нас есть метод `GetOwner` в расширениях пула, но в системе нету экземпляра пула.

Явно указать, что системе необходим экземпляр пула в котором она создана можно через специальный интерфейс: `ISetPool<TPool>`.

```cs
public class CalculateWorldPositionSystem : IReactiveSystem<IGameplayComponent>, ISetPool<IGameplayComponent>
{
    private Pool<IGameplayComponent> pool;

    public void SetPool(Pool<IGameplayComponent> typedPool)
    {
        pool = typedPool;
    }

    <IReactiveSystem Implementation>
}
```

Теперь, когда у нас есть экземпляр пула и триггер, мы можем написать простой метод вычисления позиции:

```cs
public void Execute(List<Entity<IGameplayComponent>> entities)
{
    foreach (var entity in entities)
    {
        var worldPosition = entity.Need<WorldPosition>();
        var owner = pool.GetOwner(entity);
        if (owner.Has<Position>())
        {
            worldPosition.Value = entity.Get<Position>().Value + owner.Get<Position>().Value;
        }
        else
        {
            worldPosition.Value = entity.Get<Position>().Value;
        }

        entity.ReplaceInstance(worldPosition);
    }
}
```

Думаю, все понятно? Если у контейнера есть позиция — добавляем ее к позиции точки, а если нету считаем локальную - глобальной.

Давайте допишем `Setup` метод теста и подкорректируем тесты, чтобы проверить работоспособность кода?

```cs
[SetUp]
public void Setup()
{
    pool = new GameplayPool();
    scenario = new BaseScenario("Test gameplay scenario");
    scenario.Add(pool.CreateSystem(new CalculateWorldPositionSystem()));
    scenario.Initialize();
}
```

```cs
[Test]
public void Should_return_dot_world_position()
{
    var piece = pool.CreatePiece(3, 3, 3);
    piece.Replace<Position>(p => p.Value = new Vector3i(10, 10, 10));
    var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);

    scenario.Execute();
    scenario.Cleanup();

    ComparePosition(pool.GetWorldPosition(dot), 12, 11, 10);
}

[Test]
public void Sould_return_dot_in_world_position()
{
    var piece = pool.CreatePiece(3, 3, 3);
    piece.Replace<Position>(p => p.Value = new Vector3i(10, 10, 10));
    var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 1, 0)).OwnedBy(piece);

    scenario.Execute();
    scenario.Cleanup();

    var dotAtPos = pool.GetDotAtPosition(new Vector3i(12, 11, 10));

    Assert.AreSame(dot, dotAtPos);
}
```

Проверяем.. 15 из 15! Что и требовалось! Но подождите, а где тест на движение контейнера? Допишем еще один тест:

```cs 
[Test]
public void When_contaners_moves_it_should_update_world_position_of_chilren()
{
    var piece = pool.CreatePiece(3, 3, 3);
    var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);

    var x = 0;
    while (x++ < 5)
    {
        piece.Replace<Position>(p => p.Value = new Vector3i(x, 0, 0));
        scenario.Execute();
        scenario.Cleanup();

        ComparePosition(dot.Get<WorldPosition>().Value, x, 0, 0);
    }
}
```

Как и ожидалось система тест не проходит. Нам надо обновлять `WorldPosition` каждый раз когда у контейнера меняется `Position`. Сделать это можно двумя путями: добавлять компонент-команду точкам об обновлении или считать для них позицию и выставлять. 

Чтобы не усложнять систему командами (мы еще успеем), давайте напишем реактивную систему считающую корректную позицию точек в контейнере: `Gameplay\Systems\UpdateDotsWorldPositionInsideContainerSystem.cs`

```cs
using System.Collections.Generic;
using Rentitas;

namespace Shapetris
{
    public class UpdateDotsWorldPositionInsideContainerSystem : IReactiveSystem<IGameplayComponent>, ISetPool<IGameplayComponent>
    {
        private Pool<IGameplayComponent> pool;

        public void SetPool(Pool<IGameplayComponent> typedPool)
        {
            pool = typedPool;
        }

        public void Execute(List<Entity<IGameplayComponent>> entities)
        {
            foreach (var container in entities)
            {
                var containerPosition = container.Get<Position>().Value;
                foreach (var containerDot in pool.GetContainerDots(container))
                {
                    var dotWorldPosition = containerDot.Need<WorldPosition>();
                    var dotPosition = containerDot.Need<Position>().Value;

                    dotWorldPosition.Value = containerPosition + dotPosition;

                    containerDot.ReplaceInstance(dotWorldPosition);
                }
            }
        }

        public TriggerOnEvent Trigger => Matcher.AllOf(typeof(Container), typeof(Position)).OnEntityAdded();
    }
}
```

Не забываем добавить систему в тестовый сценарий, запускаем тест и наслаждаемся: 31 тест успешно пройден!

## Компоненты-команды и поворот фигур

Далее по списку модификация фигур — поворот в двух осях. Реализовано это будет на компонентах-командах, подробнее в [следующей главе](game-commands.md).