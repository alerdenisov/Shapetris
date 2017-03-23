# Компоненты-команды в Rentitas

Если вы пропустили прошлые части ознакомьтесь с ними перед продолжением чтения:
- [Подготовка проекта](readme.md)
- [Создание игрового состояния](game-state.md) 
- [Игровые данные](game-data.md)
- [Базовые аспекты реализации игровой логики](game-data.md)

## Компоненты-команды или непрямое изменение объектов 

В большинстве случаев, мы не хотим, чтобы один объект знал логику изменения состояний другого объекта. В нашем случае, логика обработки пользовательского ввода не должна знать как поворачивать фигуры, но должна запускать цепочку событий, приводящую к повороту фигуры. 

В Rentitas для этого используются компоненты-команды. На самом деле они ничем не отличаются от обычных компонентов, но логика взаимодействия с ними уничтожает их сразу после завершения тика в ожидании следующей команды.

Для поворота будем использовать компонент `RotateCommand`, добавим его описание в `Gameplay\Commands.cs`:

```cs
public class RotateCommand : IGameplayComponent
{
    public Axis Direction;
}
```

Тип данных Axis опишем в `DataTypes\Axis.cs`:
```cs
namespace Shapetris
{
    public enum Axis
    {
        YawPositive,
        YawNegative,

        PitchPositive,
        PitchNegative
    }
}
```

Мы уже успели познакомиться с реактивными системами в прошлых частях статьи ими же будем отслеживать появление на объектах `RotateCommand`:

```cs 
using System;
using System.Collections.Generic;
using System.Text.RegularExpressions;
using Rentitas;

namespace Shapetris
{
    public class ListenPieceRotateCommand :
        IReactiveSystem<IGameplayComponent>,
        ISetPool<IGameplayComponent>,
        ICleanupSystem,
        IEnsureComponents
    {
        private Pool<IGameplayComponent> pool;
        private Group<IGameplayComponent> commands;

        public TriggerOnEvent Trigger => Matcher.AllOf(typeof(RotateCommand)).OnEntityAdded();

        public IMatcher EnsureComponents => Matcher.AllOf(typeof(Piece));

        public void SetPool(Pool<IGameplayComponent> typedPool)
        {
            pool = typedPool;
            commands = typedPool.GetGroup(Matcher.AllOf(typeof(RotateCommand)));
        }

        public void Execute(List<Entity<IGameplayComponent>> entities)
        {
            throw new NotImplementedException("Not yet");
        }

        public void Cleanup()
        {
            foreach (var entity in commands.GetEntities())
            {
                entity.Remove<RotateCommand>();
            }
        }
    }
}
```

В отличие от прошлых систем, тут мы использовали два новых интерфейса: `IEnsureComponents` и `ICleanupSystem`. Поверхностно с сleanup системами мы познакомились ранее, но теперь посмотрим на практике. 

1. При инициализации системы мы создаем группу, собирающую в себя все объекты пула с компонентом `RotateCommand`.
2. После завершения выполнения сценария (в конце тика), проходим по группе и удаляем компоненты `RotateCommand` со всех объектов.

Такой подход обеспечит 100% очищение объектов от комманд, но так же мы будем уверены, что все системы сценария получат информацию о появлении команды и её экземпляр.

`IEnsureComponents` — надстройка над реактивными системами, обеспечивающая предварительную проверку объектов на соответствие `Matcher`'у. В нашем случае под триггер попадают **все** объекты на которые добавлена команда, но в метод `Execute` попадут лишь те, что имеют `Piece`.

Так как у нас очень частный случай 3д поворота — поворот вдоль одной оси и на 90 градусов, мы воспользуемся "хаком" и будем переворачивать координаты:

```cs
  public void Execute(List<Entity<IGameplayComponent>> entities)
  {
      foreach (var piece in entities)
      {
          var axis = piece.Get<RotateCommand>().Direction;
          foreach (var containerDot in pool.GetContainerDots(piece))
          {
              var position = containerDot.Get<Position>();
              position.Value = position.Value.RightAngleRotate(axis);
              containerDot.ReplaceInstance(position);
          }
      }
  }
```

И метод расширения в `Gameplay\Extensions\PointExtensions.cs`

```cs
public static Vector3i RightAngleRotate(this Vector3i p, Axis axis) {
    switch (axis)
    {
        case Axis.YawPositive:
            p = new Vector3i(p.z, p.y, -p.x);
            break;
        case Axis.YawNegative:
            p = new Vector3i(-p.z, p.y, p.x);
            break;
        case Axis.PitchPositive:
            p = new Vector3i(p.x, -p.z, p.y);
            break;
        case Axis.PitchNegative:
            p = new Vector3i(p.x, p.z, -p.y);
            break;
        default:
            throw new ArgumentException("Unknown axis!");
    }

    return p;
}
```

Можно конечно было написать алгоритм без switch\case, но он бы потерял легкость прочтения, а выигрыш был бы сомнительным. Допишем тест и убедимся, что сделали все верно: 

```cs
[Test]
public void Should_rotate_dots_inside_piece()
{
    var piece = pool.CreatePiece(3, 3, 3);
    var dot = pool.CreateDot(DotTypes.Blue, new Vector3i(2, 0, 1)).OwnedBy(piece);

    piece.Add<RotateCommand>(r => r.Direction = Axis.YawPositive);
    scenario.Execute();
    scenario.Cleanup();

    ComparePosition(dot.Get<WorldPosition>().Value, 1, 0, -2);

    piece.Add<RotateCommand>(r => r.Direction = Axis.YawNegative);
    scenario.Execute();
    scenario.Cleanup();

    ComparePosition(dot.Get<WorldPosition>().Value, 2, 0, 1);

    piece.Add<RotateCommand>(r => r.Direction = Axis.PitchPositive);
    scenario.Execute();
    scenario.Cleanup();

    ComparePosition(dot.Get<WorldPosition>().Value, 2, -1, 0);

    piece.Add<RotateCommand>(r => r.Direction = Axis.PitchNegative);
    scenario.Execute();
    scenario.Cleanup();

    ComparePosition(dot.Get<WorldPosition>().Value, 2, 0, 1);
}
```

Запускаем тест и понимаем, что забыли добавить в сценарий систему: 

```cs
[SetUp]
public void Setup()
{
    pool = new GameplayPool();
    scenario = new BaseScenario("Test gameplay scenario");
    scenario
        .Add(pool.CreateSystem(new ListenPieceRotateCommand()))
        .Add(pool.CreateSystem(new CalculateWorldPositionSystem()))
        .Add(pool.CreateSystem(new UpdateDotsWorldPositionInsideContainerSystem()))
    ;
    scenario.Initialize();
}
```

Тест не проходит. Почему? Помните, в прошлой части мы обсуждали, что системы выполняются в порядке добавления в сценарий. Так как система `ListenPieceRotateCommand` добавлена в конец к моменту проверки мировой позиции система `CalculateWorldPositionSystem` не успела узнать об изменениях. 

**Это очень важный момент!** Инцидент спродюсирован, но очень часто встречается в реальных проектах (написанных на любом фреймворке). Проблема в том, что если бы мы проверяли код запуском игры в Unity (без unit-тестов как это делают в 99% проектах на юнити), мы бы этого даже не заметили, так как система бы получила изменения на следующем кадре. 

И что? Получила бы на следующем кадре, обновила и все довольны. Но на практике наличие таких "ошибок" приводит к тому, что появляются нежелательные артефакты в представлении (в лучшем случае), а в худшем: между обновлением запускается другая логика и оперирует некорректными данными, что приводит к настоящим ошибкам.

> По опыту, большинство `NullReferenceException` в unity-проектах появляется как раз из-за подобных ошибок порядка вызова `Update` методов в компонентах.

Переставим системы в правильный порядок и убедимся, что тест проходит за один тик приложения.

```cs
scenario
    .Add(pool.CreateSystem(new ListenPieceRotateCommand()))
    .Add(pool.CreateSystem(new CalculateWorldPositionSystem()))
    .Add(pool.CreateSystem(new UpdateDotsWorldPositionInsideContainerSystem()))
;
```

Перед тем как перейти к взаимодействию с игровым состоянием, немного допишем игровую логику: коллизии и ограничения (поворота, движения). Я не буду подробно описывать процесс, так как он ничем не отличается от предшествующего. Просто добавим необходимые компоненты и системы:

### Команда перерасчета ограничений

Компонент размеров игрового поля `LevelBounds`:
```cs
public class LevelBounds : IGameplayComponent
{
    public int Width;
    public int Height;
}
```

Команда на пересчет ограничений `CalculateContraintsCommand`:
```cs
public class CalculateContraintsCommand : IGameplayComponent, IFlag {}
```

Флаги ограничения движений: `LeftMovementConstraint`, `RightMovementConstraint`:
```cs
public class LeftMovementConstraint : IGameplayComponent, IFlag {}
public class RightMovementConstraint : IGameplayComponent, IFlag {}
```

Флаги ограничения поворотов: `YawPositiveContraint`, `YawNegativeContraint`, `PitchPositiveContraint`, `PitchNegativeContraint`:
```cs
public class YawPositiveContraint : IGameplayComponent, IFlag {}
public class YawNegativeContraint : IGameplayComponent, IFlag {}
public class PitchPositiveContraint : IGameplayComponent, IFlag {}
public class PitchNegativeContraint : IGameplayComponent, IFlag {}
```

Системы просчета ограничений `ListenPieceContraintsLevelBoundariesCommand`, `LinstenPieceConstraintCommand`:
```cs
public class LinstenPieceConstraintCommand :
    IReactiveSystem<IGameplayComponent>,
    ISetPool<IGameplayComponent>,
    ICleanupSystem,
    IEnsureComponents
{
    private Pool<IGameplayComponent> pool;
    private Group<IGameplayComponent> commands;

    public TriggerOnEvent Trigger => Matcher.AllOf(typeof(CalculateContraintsCommand)).OnEntityAdded();

    public IMatcher EnsureComponents => Matcher.AllOf(typeof(Piece), typeof(Container));

    public void SetPool(Pool<IGameplayComponent> typedPool)
    {
        pool = typedPool;
        commands = pool.GetGroup(Matcher.AllOf(typeof(CalculateContraintsCommand)));
    }

    public void Execute(List<Entity<IGameplayComponent>> entities)
    {
        foreach (var piece in entities)
        {
            var piecePosition = piece.Need<Position>().Value;
            
            var dots = pool.GetContainerDots(piece);

            if(!piece.Is<RightMovementConstraint>())
                piece.Toggle<RightMovementConstraint>(!pool.CheckMovement(dots,  1));

            if(!piece.Is<LeftMovementConstraint>())
                piece.Toggle<LeftMovementConstraint>(!pool.CheckMovement(dots, -1));

            if(!piece.Is<YawPositiveContraint>())
                piece.Toggle<YawPositiveContraint>(!pool.CheckRotation(piecePosition, dots, Axis.YawPositive));

            if(!piece.Is<YawNegativeContraint>())
                piece.Toggle<YawNegativeContraint>(!pool.CheckRotation(piecePosition, dots, Axis.YawNegative));

            if(!piece.Is<PitchPositiveContraint>())
                piece.Toggle<PitchPositiveContraint>(!pool.CheckRotation(piecePosition, dots, Axis.PitchPositive));

            if(!piece.Is<PitchNegativeContraint>())
                piece.Toggle<PitchNegativeContraint>(!pool.CheckRotation(piecePosition, dots, Axis.PitchNegative));
        }
    }

    public void Cleanup()
    {
        foreach (var entity in commands.GetEntities())
        {
            entity.Remove<CalculateContraintsCommand>();
        }
    }
}
```

```cs
public class ListenPieceContraintsLevelBoundariesCommand :
    IReactiveSystem<IGameplayComponent>,
    ISetPool<IGameplayComponent>,
    ICleanupSystem,
    IEnsureComponents
{
    private Pool<IGameplayComponent> pool;
    private Group<IGameplayComponent> commands;

    public TriggerOnEvent Trigger => Matcher.AllOf(typeof(CalculateContraintsCommand)).OnEntityAdded();

    public IMatcher EnsureComponents => Matcher.AllOf(typeof(Piece), typeof(Container));

    public void SetPool(Pool<IGameplayComponent> typedPool)
    {
        pool = typedPool;
        commands = pool.GetGroup(Matcher.AllOf(typeof(CalculateContraintsCommand)));
    }

    public void Execute(List<Entity<IGameplayComponent>> entities)
    {
        var level = pool.GetLevel();
        var bounds = level.Get<LevelBounds>();

        var minX = level.Get<Position>().X;
        var maxX = minX + bounds.Width;

        foreach (var piece in entities)
        {
            var dots = pool.GetContainerDots(piece);
            if (!piece.Is<RightMovementConstraint>())
                piece.Toggle<RightMovementConstraint>(!dots.CheckBounds(minX, maxX, 1));

            if(!piece.Is<LeftMovementConstraint>())
                piece.Toggle<LeftMovementConstraint>(!dots.CheckBounds(minX, maxX, -1));
        }
    }

    public void Cleanup()
    {
        foreach (var entity in commands.GetEntities())
        {
            entity.Remove<CalculateContraintsCommand>();
        }
    }
}
```

Системы запуска перерасчета ограничений `RecalculateConstraintWhenMovedSystem`, `RecalculateConstraintWhenAddDotSystem`:
```cs
public class RecalculateConstraintWhenMovedSystem : 
    IReactiveSystem<IGameplayComponent>, 
    IEnsureComponents
{
    public void Execute(List<Entity<IGameplayComponent>> entities)
    {
        foreach (var piece in entities)
        {
            piece.Toggle<CalculateContraintsCommand>(true);
        }
    }

    public TriggerOnEvent Trigger => Matcher.AllOf(typeof(Position)).OnEntityAdded();
    public IMatcher EnsureComponents => Matcher.AllOf(typeof(Piece), typeof(Container));
}
```
```cs
public class RecalculateConstraintWhenAddDotSystem :
    IReactiveSystem<IGameplayComponent>,
    IEnsureComponents,
    ISetPool<IGameplayComponent>
{
    private Pool<IGameplayComponent> pool;

    public TriggerOnEvent Trigger => Matcher.AllOf(typeof(Owner)).OnEntityAdded();

    public IMatcher EnsureComponents => Matcher.AllOf(typeof(Dot));

    public void SetPool(Pool<IGameplayComponent> typedPool)
    {
        pool = typedPool;
    }

    public void Execute(List<Entity<IGameplayComponent>> entities)
    {
        foreach (var dot in entities)
        {
            pool.GetOwner(dot).Toggle<CalculateContraintsCommand>(true);
        }
    }
}
```

Процесс "запекания" фигуры в уровень будет запускаться как альтернатива падения. То есть, добавим еще одно ограничение: `FallConstraint` и систему `LinstenPieceConstraintsFallCommand`.

```cs
public class FallConstraint : IGameplayComponent, IFlag {}

public class LinstenPieceConstraintsFallCommand :
    IReactiveSystem<IGameplayComponent>,
    ISetPool<IGameplayComponent>,
    ICleanupSystem,
    IEnsureComponents
{
    private Pool<IGameplayComponent> pool;
    private Group<IGameplayComponent> commands;

    public TriggerOnEvent Trigger => Matcher.AllOf(typeof(CalculateContraintsCommand)).OnEntityAdded();

    public IMatcher EnsureComponents => Matcher.AllOf(typeof(Piece), typeof(Container));

    public void SetPool(Pool<IGameplayComponent> typedPool)
    {
        pool = typedPool;
        commands = pool.GetGroup(Matcher.AllOf(typeof(CalculateContraintsCommand)));
    }

    public void Execute(List<Entity<IGameplayComponent>> entities)
    {
        foreach (var piece in entities)
        {
            var dots = pool.GetContainerDots(piece);

            if(!piece.Is<FallConstraint>())
                piece.Toggle<FallConstraint>(!pool.CheckFalling(dots,  1));
        }
    }

    public void Cleanup()
    {
        foreach (var entity in commands.GetEntities())
        {
            entity.Remove<CalculateContraintsCommand>();
        }
    }
}
```

Покроем системы тестами (не забыв добавить в сценарий):

```cs
[Test]
public void Should_constrain_left_movement_by_bounds()
{
    var level = pool.CreateLevel(10,10);
    var piece = pool.CreatePiece();

    pool.CreateDot(DotTypes.Blue, new Vector3i(-1, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(1, 0, 0)).OwnedBy(piece);

    piece.SetPosition(1, 4, 0);
    scenario.Execute();
    scenario.Cleanup();

    Assert.IsTrue(piece.Has<LeftMovementConstraint>());
    Assert.IsFalse(piece.Has<RightMovementConstraint>());
}

[Test]
public void Should_constrain_falling_when_achive_floor()
{
    var level = pool.CreateLevel(10,10);
    var piece = pool.CreatePiece();

    pool.CreateDot(DotTypes.Blue, new Vector3i(-1, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(1, 0, 0)).OwnedBy(piece);

    piece.SetPosition(3, 0, 0);
    scenario.Execute();
    scenario.Cleanup();

    Assert.IsTrue(piece.Has<FallConstraint>());
}

[Test]
public void Should_constrain_falling_when_collide_level_dots()
{
    var level = pool.CreateLevel(10,10);

    for (int x = 1; x < 10; x++)
    {
        pool.CreateDot(DotTypes.Red, new Vector3i(x, 0, 0)).OwnedBy(level);
    }

    var piece = pool.CreatePiece();

    pool.CreateDot(DotTypes.Blue, new Vector3i(-1, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(1, 0, 0)).OwnedBy(piece);

    piece.SetPosition(3, 1, 0);
    scenario.Execute();
    scenario.Cleanup();

    Assert.IsTrue(piece.Has<FallConstraint>());
}


[Test]
public void Should_constrain_right_movement_by_bounds()
{
    var level = pool.CreateLevel(10,10);
    var piece = pool.CreatePiece();

    pool.CreateDot(DotTypes.Blue, new Vector3i(-1, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(1, 0, 0)).OwnedBy(piece);

    piece.SetPosition(9, 4, 0);
    scenario.Execute();
    scenario.Cleanup();

    Assert.IsTrue(piece.Has<RightMovementConstraint>());
    Assert.IsFalse(piece.Has<LeftMovementConstraint>());
}

[Test]
public void Should_constrain_movement()
{
    var level = pool.CreateLevel(10,10);

    for (int y = 0; y < 10; y++)
    {
        pool.CreateDot(DotTypes.Red, new Vector3i(5, y, 0)).OwnedBy(level);
    }

    var piece = pool.CreatePiece();
    pool.CreateDot(DotTypes.Blue, new Vector3i(-1, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(0, 0, 0)).OwnedBy(piece);
    pool.CreateDot(DotTypes.Blue, new Vector3i(1, 0, 0)).OwnedBy(piece);
    piece.SetPosition(3, 4, 0);

    scenario.Execute();
    scenario.Cleanup();

    Assert.IsTrue(piece.Has<RightMovementConstraint>());
    Assert.IsFalse(piece.Has<LeftMovementConstraint>());
}
```

Данных наработок достаточно, чтобы перейти к взаимодействию с игровым состоянием — огранизовать [игровой цикл](game-cycle.md).