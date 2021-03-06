# Взаимодействие с игровым состоянием или огранизация игрового цикла

Если вы пропустили прошлые части ознакомьтесь с ними перед продолжением чтения:
- [Подготовка проекта](readme.md)
- [Создание игрового состояния](game-state.md) 
- [Игровые данные](game-data.md)
- [Базовые аспекты реализации игровой логики](game-data.md)
- [Компоненты-команды и развитие игровой логики](game-data.md)

## Игровой цикл
В прошлых частях мы подготовили костяк нашей игры: описали игровое состояние и основные игровые данные. Пришло время объединить это в игровой цикл.

### Вложенность сценариев
До этого момента мы описывали создавали сценарии только в рамках теста и очень простыми. На деле же сценарии намного более функциональные, чем могло показаться. Ключевая их возможность: вложенность и отлючаемость.

Давайте вспомним нашу матрицу игрового состояния?

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

Я советую добавлять сценарий под каждое уникальное состояние (даже если вам в него пока нечего добавить):

- Main scenario
  - Start screen scenario
  - Choice game screen scenario
  - Leader screen scenario
  - Shop screen scenario
  - Score screen scenario
  - Arcade game scenario
  - Missions game scenario
  - Game over screen scenario

Выглядит удобно, но возникают вопросы, например, просчет ограничений движения фигур необходим и в аркадном режиме, и в миссиях, а дублировать код не хочется. Программист знакомый с ООП сразу подумает о наследовании: абстрактном `GameScenario` и его дочках: `ArcadeGameScenario` и `MissonsGameScenario`. Но это не мой подход. ООП конечно хорош, но на протяжении всей статьи мы использовали подход композиции вместо наследования и сценарии не исключение.

Просто добавим к сценариям еще `Game scenario` и `Menu scenario`. В ходе разработки к ним присоединятся еще `Logging scenario`, `User input scenario`, `View scenerio` и т.д.

> Основное правило: если вы можете по какой-то характеристике выделить сценарий — сделайте это. Сценарии удобны тем, что их можно в любой момент отключить и посмотреть на то как это повлияет на приложение в целом.

### Управление сценариями
