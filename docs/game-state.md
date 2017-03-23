# Состояние игры Shapetris

Если вы пропустили прошлую часть с [подготовкой проекта](readme.md) ознакомьтесь с ней перед продолжением чтения.

## Общая структура
Перед описание игровых данных, я планирую и продумываю возможные состояния приложения, например, меню, игра, пауза и т.д. Тщательность и внимание к данному аспекту с лихвой окупится (или аукнется) в будущем. У нас не сложная игра и состояний у нее не много:

- Меню
  - Стартовый экран
  - Выбор типа игры: аркада \ миссии
    - Выбор миссии
  - Экран лидеров
  - Экран покупок
- Игра
  - Активная 
    - Аркада
    - Миссия
  - Пауза
    - Обучение
    - Пользовательская
  - Завершение
- Конец игры
  - Реклама
  - Экран очков\переигровки

Внимательный читатель уже обратил внимание на подводный камень: наложение состояний. Например, игра может быть миссионная, но на паузе обучения или пользовательская пауза аркадной игры. 

Правда в том, что я слукавил. Я не делаю дерево состояний: я рисую себе огромную матрицу:

|                 | СЭ | ВТ | ЭЛ | ЭП | ЭО | ИА | ИМ | КИ | ОБ | ПА | РЕ |
| --------------- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Стартовый экран |    |    |    |    |    |    |    |    |    | ️️   |    |
| Выбор типа игры | .  |    |    |    |    |    |    |    |    |    |    |
| Экран лидеров   | .  | .  |    |    |    |    |    |    |    |    |    |
| Экран покупок   | .  | .  | .  |    |    |    |    |    |    |    |    |
| Экран очков     | .  | .  | .  | .  |    |    |    |    |    |    |    |
| Игра аркада     | .  | .  | .  | .  | .  |    |    | ️   | ️   |    |    |
| Игра миссия     | .  | .  | .  | .  | .  | .  |    |    |    |    |    |
| Конец игры      | .  | .  | .  | .  | .  | .  | .  |    |    |    |    |
| Обучение        | .  | .  | .  | .  | .  | ▄  | ▄  | .  |    |    |    |
| Пауза           | .  | .  | .  | .  | ▄  | ▄  | ▄  | .  | ▄  |    |    |
| Реклама         | ▄  | ▄  | ▄  | ▄  | ▄  | ▄  | ▄  | ▄  | ▄  | ▄  |    |

```
▄ - допускается наложение состояний
```

Конечно, в таком небольшом проекте это может показаться излишним, но я не умею держать много информации в голове и даже представить эту матрицу без визуального подкрепления — сложно.

А теперь самое интересное: анализ матрицы. Что мы на ней видим? У нас есть 7 уникальных состояний:
- Стартовый экран
- Выбор типа игры
- Экран лидеров
- Экран покупок
- Экран очков
- Игра аркада
- Игра миссия
- Конец игры

Их характерная особенность, что они никогда не пересекаются, а значит это `enum`:
```cs
public enum GameState {
  StartScreen,
  ChoiceGameScreen,
  LeadersScreen,
  ShopScreen,
  ArcadeGame,
  MissionsGame,
  EndGameScreen,
  ScoresScreen
}
```

Так же заметно, что они делятся на две с половинной категории: до игры, во время и после игры. А значит у нас есть группы состояний: `InMenu`, `InGame`, `AfterGame`. 

Кроме групп и уникальных состояний потребуются флаги: `InPause`, `InAdvertisment`, `InTutorial`.

Давайте опишем объект состояния: 

```cs
public enum Screens
{
    // Main Menu
    StartScreen,
    ChoiceGameScreen,
    LeadersScreen,   
    ShopScreen,
    
    // After game
    EndGameScreen,
    ScoresScreen
}

public enum GameTypes
{
    Arcade,
    Missions
}

public class ApplicationState
{
    private const int U_OFF = 8;
    private const int F_OFF = 4;

    public enum States {
        _InMenu             = 0,
        _InGame             = 1,
        _AfterGame          = 2,

        InPause             = 1 << (F_OFF + 0),
        InAdvertisment      = 1 << (F_OFF + 1),
        InTutorial          = 1 << (F_OFF + 2),

        StartScreen         = (Screens.StartScreen      << U_OFF) | _InMenu,
        ChoiceGameScreen    = (Screens.ChoiceGameScreen << U_OFF) | _InMenu,
        LeadersScreen       = (Screens.LeadersScreen    << U_OFF) | _InMenu,
        ShopScreen          = (Screens.ShopScreen       << U_OFF) | _InMenu,

        ArcadeGame          = (GameTypes.Arcade         << U_OFF) | _InGame,
        MissionsGame        = (GameTypes.Missions       << U_OFF) | _InGame,

        EndGameScreen       = (Screens.EndGameScreen    << U_OFF) | _AfterGame,
        ScoresScreen        = (Screens.ScoresScreen     << U_OFF) | _AfterGame
    }

    public States Current;
}
```

Разработчикам не знакомым с бинарными операциями данный код может показаться незнакомым, поэтому объясню его подробнее. 

> Enum в памяти хранится по-стандарту как целочисленное значение, типа `int32`, то есть 32 бита (0 или 1). Если думать об `int32` как о 32-х лампочках принимающих два состояния, то 32 лампочки — это 4 294 967 296 уникальных комбинации (2 в степени 32) или 32 отдельных true\false значения. 

В нашем случае 4 млн состояний излишне как и 32 флага. Поэтому я представил, что часть выделенной памяти — флаги, а другая уникальные значения зависящие от них. Я использую сдвиги `<<`, чтобы определить место в памяти куда записать нужные биты, а так же бинарное ИЛИ `|`, чтобы склеить воедино группы и значения. 

```
                                                         F_OFF
                                                         <------------
                            U_OFF
                            <-----------------------------------------

[0000 0000 0000 0000]  [0000]  [0       0         0     0]      [0000] 
UNUSED                 Unique  UNUSED   Tutorial  Adv   Pause   States
                       state                                    group
```

> Данные о состоянии можно упаковать значииительно сильнее (в `byte`). Любознательные могут самостоятельно попробовать запихнуть все данные в 0000 0000.

## Хранение данных и получение
Нужно организовать доступ к данным о текущем состоянии приложения из любого места простым и удобным. Тут нам поможет Rentitas.

Основное правило Rentitas, что всё (вообще всё!) данные должны храниться в компонентах, а компоненты в объектах (entity), а entity в пулах (pool). С таким подходом мы можем запустить одновременно (в одном треде) сколько угодно экземпляров приложений и пока их данные в пулах, мы будем уверены на 100%, что у каждого приложения будет свой экземпляр состояния.

### ICoreComponent и Core пул
Для таких компонентов как `State` отведем специальный пул — `Core`. Разделение пулов в Rentitas реализуется через опеределение общего для всех его компонентов интерфейса, в случае с системным пулом — `ICoreComponent`.

Давайте создадим в корне проекта `Shapetris` файл: `Pools.cs`. 

> Это не по канонам C# задавать имена файлам отличные от названий классов\интерфейсов\структур описанных в них, но вы очень скоро поймете почему я иду на такую "жертву".

Все, что нам нужно добавить в `Pools.cs` на данный момент — пустой интерфейс `ICoreComponent`:

```cs
using Rentitas;

namespace Shapetris
{
    /// <summary>
    /// Interface as an identifier of components from core pool 
    /// <seealso cref="Shapetris.Core"/>
    /// </summary>
    public interface ICoreComponent : IComponent {}
}
```

В дальнейшем, в этом файле появится еще ряд других аналогичных интерфейсов разделяющих данные приложения. Но, подождите, а что такое `IComponent`? Если вы подключили Rentitas как проект, то обратившись к исходникам увидите, что это опять же пустой интерфейс. Этот интерфейс как и `ICorePool` просто говорит нам (или внутренностям Rentitas) о том, что мы имеем дело с компонентами.

### Первый компонент пула
Так как мы уже определились с описанием объекта `ApplicationState`, самое время сделать из него компонент `Core` пула: cоздадим в проекте `Shapetris` подпапку `Core` и добавим туда файл класса: `ApplicationState.cs`.

Код `ApplicationState` ~~почти~~ никак не будет отличаться от того, что мы обсуждали прежде. За исключением того, что класс `ApplicationState` будет "реализовывать" пустой интерфейс `ICoreComponent`, чтобы мы могли добавить его экземпляр в Core пул.

Так же у приложения не может быть несколько состояний одновременно и по логике объект `ApplicationState` должен реализовывать паттерн Singleton. Для таких задач в Rentitas предусмотрен специальный интерфейс `ISingleton` (угадаете какой? ~~пустой~~). Добавим к `ApplicationState` и его тоже, а как он нам поможет увидем чуть позже когда будем реализовывать взаимодействие с состоянием.

```cs
using Rentitas;

namespace Shapetris
{
    public enum Screens
    {
        // Main Menu
        StartScreen,
        ChoiceGameScreen,
        LeadersScreen,
        ShopScreen,

        // After game
        EndGameScreen,
        ScoresScreen
    }

    public enum GameTypes
    {
        Arcade,
        Missions
    }

    public class ApplicationState : ICoreComponent, ISingleton
    {
        internal const int U_OFF = 8;
        internal const int F_OFF = 4;

        internal enum States {
            _InMenu             = 0,
            _InGame             = 1,
            _AfterGame          = 2,

            InPause             = 1 << (F_OFF + 0),
            InAdvertisment      = 1 << (F_OFF + 1),
            InTutorial          = 1 << (F_OFF + 2),

            StartScreen         = (Screens.StartScreen      << U_OFF) | _InMenu,
            ChoiceGameScreen    = (Screens.ChoiceGameScreen << U_OFF) | _InMenu,
            LeadersScreen       = (Screens.LeadersScreen    << U_OFF) | _InMenu,
            ShopScreen          = (Screens.ShopScreen       << U_OFF) | _InMenu,

            ArcadeGame          = (GameTypes.Arcade         << U_OFF) | _InGame,
            MissionsGame        = (GameTypes.Missions       << U_OFF) | _InGame,

            EndGameScreen       = (Screens.EndGameScreen    << U_OFF) | _AfterGame,
            ScoresScreen        = (Screens.ScoresScreen     << U_OFF) | _AfterGame
        }

        internal States Current;

        public ApplicationState()
        {
            Current = States.StartScreen;
        }
    }
}
```


#### Методы взаимодействие с состоянием
Дизайн объекта `ApplicationState` не располагает к взаимодействию с ним. Плюс, я специально отметил его поле и enum `ApplicationState.States` как `internal`, чтобы они были доступны исключительно текущему проекту (в случае с большой модульной системой это удобно, а в текущем проекте просто хорошая привычка). 

Реализовать изменения подобных компонентов в Rentitas можно тремя способами:
1. "В лоб" — используя "низкоуровневые" методы объекта `Entity`.
2. Через компоненты команды, например, `SetPause`.
3. Через методы расширения.

Мы выберем третий способ, так как он самый простой и не требует от нас создания многочисленных компонентов.

Создаем в папке `Core` еще одну папку: `Extensions`. (Если у вас стоит ReSharper или вы пользуетесь Rider, отключите Namespace в свойствах папки). В папке создадим файл: `CorePoolExtensions.cs`:

```cs
using Rentitas;

namespace Shapetris
{
    public static class CorePoolExtensions
    {
        #region Application state getters
        public static bool InMenu(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool InGame(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool InAfterGame(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool InPause(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool InAdvertisment(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool InTutorial(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsStartScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsChoiceGameScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsLeadersScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsShopScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsArcadeGame(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsMissionsGame(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsEndGameScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static bool IsScoresScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static Screens GetCurrentScreen(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }
        #endregion

        #region Application state setter
        public static void OpenScreen(this Pool<ICoreComponent> pool, Screens screen)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static void StartGame(this Pool<ICoreComponent> pool, GameTypes type)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static void TogglePause(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static void ToggleAdvertisment(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }

        public static void ToggleTutorial(this Pool<ICoreComponent> pool)
        {
            throw new System.NotImplementedException("Not implemented yet");
        }
        #endregion
    }
}
```
Ух, ну и длинный файл, да? А это еще даже без реализации. На самом деле если присмотреться к названиям методов становится понятно, что и как они будут делать даже без документации.

Реализацию методов отложим на будущее. Сейчас напишем наши первые тесты в соответствии с матрицей.

## Тестирование данных

Помните, что у нас есть `Shapetris.Tests`? Создадим там новый файл тестов: `ApplicationTests.cs`: 

```cs
using NUnit.Framework;
using Rentitas;

namespace Shapetris.Tests
{
    [TestFixture]
    public class ApplicationTests
    {
        private Pool<ICoreComponent> pool;

        [SetUp]
        public void Setup()
        {
            pool = new Pool<ICoreComponent>(new ApplicationState());
        }
    }
}
```

С помощью `new Pool<T>()` мы создаем локальный\тестовый (оторванный от контекста приложений Rentitas) экземпляр пула. Но создание экземпляра компонента может вас запутать. Передавая в конструктор пула экземпляр компонента, мы говорим пулу с какими компонентами он будет работать.

> Будьте внимательны, это не создаст `Entity<ICoreComponent>` с `ApplicationState` компонентом, а определит, что данный экземпляр пула работает с данным компонентом. А созданный экземпляр попадет в кеш компонентов пула ожидая дальнейшего использования. Это очень важно!

Чтобы убедиться в этом, напишем первый тест: 
```cs 
[Test]
public void Core_pool_should_have_a_state()
{
    Assert.IsNotNull(pool.Get<ApplicationState>());
}
```

Метод `Get<T>()` пула позволяет получить экземпляр компонента, реализовывающий `ISingleton`. 

Запустите тест и убедитесь, что метод вернул `null`. Добавим в метод настройки контекста теста создание Entity с необходимым компонентом: 

```cs
pool.CreateEntity().Add<ApplicationState>();
```

Теперь тест успешно проходится. Я не буду объяснять каждый тест, думаю они говорят сами за себя, просто ознакомьтесь с ними и приступим к реализации методов расширения, чтобы они проходились. 

Полный листинг текстов `ApplicationState`:
```cs
using System;
using NUnit.Framework;
using Rentitas;

namespace Shapetris.Tests
{
    [TestFixture]
    public class ApplicationTests
    {
        private Pool<ICoreComponent> pool;

        [SetUp]
        public void Setup()
        {
            pool = new Pool<ICoreComponent>(new ApplicationState());
            pool.CreateEntity().Add<ApplicationState>();
        }

        [Test]
        public void Core_pool_should_have_a_state()
        {
            Assert.IsNotNull(pool.Get<ApplicationState>());
        }

        [Test]
        public void Should_start_on_menu()
        {
            Assert.IsTrue(pool.InMenu());
        }

        [Test]
        public void Should_start_on_start_screen()
        {
            Assert.IsTrue(pool.GetCurrentScreen() == Screens.StartScreen);
        }

        [TestCase(Screens.LeadersScreen,    TestName = "Open leaders game")]
        [TestCase(Screens.ChoiceGameScreen, TestName = "Open choice game")]
        [TestCase(Screens.ShopScreen,       TestName = "Open shop game")]
        public void Should_can_change_screen(Screens screen)
        {
            pool.OpenScreen(screen);
            Assert.IsTrue(pool.GetCurrentScreen() == screen);
        }

        [Test]
        public void Should_change_group_after_screen()
        {
            Assert.IsTrue(pool.InMenu());
            Assert.IsTrue(pool.GetCurrentScreen() == Screens.StartScreen);
            pool.OpenScreen(Screens.EndGameScreen);
            Assert.IsTrue(pool.GetCurrentScreen() == Screens.EndGameScreen);
            Assert.IsTrue(pool.InAfterGame());
        }

        [Test]
        public void Exception_on_change_screen_to_same()
        {
            Assert.IsTrue(pool.GetCurrentScreen() == Screens.StartScreen);
            Assert.Throws<ArgumentException>(() => pool.OpenScreen(Screens.StartScreen));
        }

        [Test]
        public void Should_start_game()
        {
            Assert.IsFalse(pool.InGame());
            pool.StartGame(GameTypes.Arcade);
            Assert.IsTrue(pool.InGame());
        }

        [TestCase(GameTypes.Arcade, TestName = "Start arcade game")]
        [TestCase(GameTypes.Missions, TestName = "Start missions game")]
        public void Should_start_corresponding_game(GameTypes type)
        {
            Assert.IsFalse(pool.InGame());
            pool.StartGame(type);
            Assert.IsTrue(pool.InGame());
            Assert.IsTrue(pool.GetCurrentGameType() == type);
        }

        [Test]
        public void Exception_on_getting_current_game_type_outside_game_state()
        {
            Assert.IsFalse(pool.InGame());
            Assert.Throws<ArgumentException>(() => pool.GetCurrentGameType());
        }

        public void Exception_then_trying_start_game_when_already_in_game()
        {
            Assert.IsFalse(pool.InGame());
            pool.StartGame(GameTypes.Arcade);
            Assert.Throws<ArgumentException>(() => pool.StartGame(GameTypes.Missions));
        }

        [Test]
        public void Should_can_pause()
        {
            pool.StartGame(GameTypes.Arcade);
            Assert.IsFalse(pool.InPause());
            pool.TogglePause();
            Assert.IsTrue(pool.InPause());
        }

        [Test]
        public void Exception_on_pause_outside_game()
        {
            Assert.IsFalse(pool.InGame());
            Assert.IsFalse(pool.InPause());
            Assert.Throws<ArgumentException>(() => pool.TogglePause());
        }

        [Test]
        public void Should_can_toggle_adv()
        {
            pool.StartGame(GameTypes.Arcade);
            Assert.IsFalse(pool.InAdvertisment());
            pool.ToggleAdvertisment();
            Assert.IsTrue(pool.InAdvertisment());
        }
    }
}
```

## Изменение и получение компонентов системы
Для получения компонентов и их изменения Rentitas предоставляет ряд методов. Давайте для начала разберемся с получением: реализуем метод `GetCurrentGameType`:
```cs
public static GameTypes GetCurrentGameType(this Pool<ICoreComponent> pool)
{
    if(!pool.InGame())
        throw new ArgumentException("Can't get game type when application outside the game");

    var state = pool.Get<ApplicationState>().Current;
    return (GameTypes)((int)state >> ApplicationState.U_OFF);
}
```

В коде выше нас интересует метод `Get<TComponent>()` объекта `pool`. Так как мы пометили `ApplicationState` как `ISingleton`, мы можем получить его экземпляр сразу из пула без поиска объекта, содержащего его. С поиском мы разберемся позднее.

Кроме получения экземпляра компонента, мы можем запросить и объект содержащий его через `GetSingle<TSingletonComponent>()`. Этот метод нам потребуется, чтобы реализовывать функционал изменения состояния. Давайте взглянем на реализацию `ToggleAdvertisment`:
```cs
public static void ToggleAdvertisment(this Pool<ICoreComponent> pool)
{
    var stateEntity = pool.GetSingle<ApplicationState>();
    var state = stateEntity.Get<ApplicationState>();

    if (pool.InAdvertisment())
    {
        state.Current &= ~ApplicationState.States.InAdvertisment;
    }
    else
    {
        state.Current |= ApplicationState.States.InAdvertisment;
    }
    stateEntity.ReplaceInstance(state);
}
```

1. Мы получаем ссылку `stateEntity` на объект в Core пуле, содержащий `ApplicationState`
2. Извлекаем экземпляр из объекта через метод `Get<TComponent>()`
3. Вносим изменения в значение поля `Current`
4. Сохраняем изменения в stateEntity: `ReplaceInstance`. Так как мы редактировали существующий экземпляр, корректнее сказать: уведомляем систему об изменениях.

Листинг всех методов:
```cs
using System;
using System.Diagnostics.CodeAnalysis;
using Rentitas;

namespace Shapetris
{
    [SuppressMessage("ReSharper", "BitwiseOperatorOnEnumWithoutFlags")]
    public static class CorePoolExtensions
    {
        private static bool HasStateFlag(ApplicationState.States current, ApplicationState.States flag)
        {
            return (current & flag) == flag;
        }

        #region Application state getters
        public static bool InMenu(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States._InMenu);
        }

        public static bool InGame(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States._InGame);
        }

        public static bool InAfterGame(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States._AfterGame);
        }

        public static bool InPause(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States.InPause);
        }

        public static bool InAdvertisment(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States.InAdvertisment);
        }

        public static bool InTutorial(this Pool<ICoreComponent> pool)
        {
            return HasStateFlag(pool.Get<ApplicationState>().Current, ApplicationState.States.InTutorial);
        }

        public static bool IsArcadeGame(this Pool<ICoreComponent> pool)
        {
            return pool.GetCurrentGameType() == GameTypes.Arcade;
        }

        public static bool IsMissionsGame(this Pool<ICoreComponent> pool)
        {
            return pool.GetCurrentGameType() == GameTypes.Missions;
        }

        public static GameTypes GetCurrentGameType(this Pool<ICoreComponent> pool)
        {
            if(!pool.InGame())
                throw new ArgumentException("Can't get game type when application outside the game");

            var state = pool.Get<ApplicationState>().Current;
            return (GameTypes)((int)state >> ApplicationState.U_OFF);
        }

        public static Screens GetCurrentScreen(this Pool<ICoreComponent> pool)
        {
            if(pool.InGame())
                throw new Exception("Can't get current screen while application in the game");

            var state = pool.Get<ApplicationState>().Current;
            return (Screens)((int)state >> ApplicationState.U_OFF);
        }

        #endregion

        #region Application state setter
        public static void OpenScreen(this Pool<ICoreComponent> pool, Screens screen)
        {
            if(pool.InPause())
                throw new ArgumentException("Can't open screens in pause");

            if(pool.GetCurrentScreen() == screen)
                throw new ArgumentException("Can't open same screens");

            // receive entity which contains ApplicationState component
            var stateEntity = pool.GetSingle<ApplicationState>();
            var state = stateEntity.Get<ApplicationState>();

            switch (screen)
            {
                case Screens.StartScreen:
                    state.Current = ApplicationState.States.StartScreen;
                    break;

                case Screens.ChoiceGameScreen:
                    state.Current = ApplicationState.States.ChoiceGameScreen;
                    break;

                case Screens.LeadersScreen:
                    state.Current = ApplicationState.States.LeadersScreen;
                    break;

                case Screens.ShopScreen:
                    state.Current = ApplicationState.States.ShopScreen;
                    break;

                case Screens.EndGameScreen:
                    state.Current = ApplicationState.States.EndGameScreen;
                    break;

                case Screens.ScoresScreen:
                    state.Current = ApplicationState.States.ScoresScreen;
                    break;

                default:
                    throw new ArgumentException("Unknown screen type");
            }

            stateEntity.ReplaceInstance(state);
        }

        public static void StartGame(this Pool<ICoreComponent> pool, GameTypes type)
        {
            if(pool.InGame())
                throw new ArgumentException("Can't start game twice!");

            var stateEntity = pool.GetSingle<ApplicationState>();
            var state = stateEntity.Get<ApplicationState>();

            switch (type)
            {
                case GameTypes.Arcade:
                    state.Current = ApplicationState.States.ArcadeGame;
                    break;

                case GameTypes.Missions:
                    state.Current = ApplicationState.States.MissionsGame;
                    break;

                default:
                    throw new ArgumentException("Unknown game type");
            }

            stateEntity.ReplaceInstance(state);
        }

        public static void TogglePause(this Pool<ICoreComponent> pool)
        {
            if(!pool.InGame())
                throw new ArgumentException("Can't pause outside the game");

            var stateEntity = pool.GetSingle<ApplicationState>();
            var state = stateEntity.Get<ApplicationState>();

            if (pool.InPause())
            {
                state.Current &= ~ApplicationState.States.InPause;
            }
            else
            {
                state.Current |= ApplicationState.States.InPause;
            }
            stateEntity.ReplaceInstance(state);
        }

        public static void ToggleAdvertisment(this Pool<ICoreComponent> pool)
        {
            var stateEntity = pool.GetSingle<ApplicationState>();
            var state = stateEntity.Get<ApplicationState>();

            if (pool.InAdvertisment())
            {
                state.Current &= ~ApplicationState.States.InAdvertisment;
            }
            else
            {
                state.Current |= ApplicationState.States.InAdvertisment;
            }
            stateEntity.ReplaceInstance(state);
        }

        public static void ToggleTutorial(this Pool<ICoreComponent> pool)
        {
            if(!pool.InGame())
                throw new ArgumentException("Can't show tutorial outside the game");

            var stateEntity = pool.GetSingle<ApplicationState>();
            var state = stateEntity.Get<ApplicationState>();

            if (pool.InTutorial())
            {
                state.Current &= ~ApplicationState.States.InTutorial;
            }
            else
            {
                state.Current |= ApplicationState.States.InTutorial;
            }
            stateEntity.ReplaceInstance(state);
        }
        #endregion
    }
}
```

Вьюх! Дальше больше. Теперь попробуем описать [игровые данные](game-data.md).