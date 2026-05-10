# Review на реализацию от [@VictorGotsenko](https://github.com/VictorGotsenko) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/VictorGotsenko/CurrencyExchange-API)

## Реализация (судил по отсчету бота от 01.05)

- Хорошо:
    - Сделанный деплой REST API

- Замечания:
    - Жалко без фронта, у третьего проекта есть готовый фронт.
    - Некоторые запросы возвращают не валидный ответ (часто 200 вместо 201, 400 вместо 200 и тд),
      прикладываю [отсчет](../reports/report_VictorGotsenko.md)

## По коду

### package model

- Не понял зачем нужно поле `LocalDateTime createdAt` для моделей, если оно скорей всего никак не используется.
- Очень странное наследование моделей от BaseModel - нарушает `is a` наследования. Наследование ради общего паттерна для
  даты через чур. Нигде не используется полиморфно.
- `Long` для `id` семантически чуть лучше подходит.
- Все поля в `ExchangeRate` имеют стандартный модификатор доступа.
- Зачем принимать в конструктор rate как String вместо BigDecimal? Получается очень странный контракт для клиента:

```
    public ExchangeRate(int baseCurrencyId, int targetCurrencyId, String rate) {
        this.baseCurrencyId = baseCurrencyId;
        this.targetCurrencyId = targetCurrencyId;
        this.rate = new BigDecimal(rate);
    }
```

### package repository

- Выделенные интерфейсы для репозиториев - хорошо.
- Не стоит пробрасывать `throws SQLException` БД исключение в верхние слои, надо обернуть в кастомное исключение,
  например `DatabaseException`.
- `Optional` для поиска - хорошо.
- Вместо `*` стоит конкретно прописывать аргументы:

```
"SELECT * FROM currencies"

// лучше
"SELECT code, fullname, sign FROM currencies"
```

- `Statement`, `PreparedStatement` и `ResultSet` - стоит использовать через try-with-resources, не везде это соблюдено.
- В некоторых местах одновременно есть обработка SQLException в RuntimeException и в сигнатуре метода стоит
  `throws SQLException`, что вызывает путаницу.
- Стоит выделять повторяющиеся участки кода - их подсвечивает IDEA.
- В `getCurrencies` странный вечный цикл с try внутри, стоит упростить.
- Обычно не используют напрямую Connection, лучше получать его через DataSource / connection pool.
- Один репозиторий зависит от другого - точно так делать не стоит. Лучше использовать JOIN или в сервис вынести вызовы
  другого репозитория.

```java
public final class ExchangeRatesRepositoryImpl implements ExchangeRatesRepository {
    private final Connection connection;
    private final CurrenciesRepository currenciesRepository;

    public ExchangeRatesRepositoryImpl(Connection connection) {
        this.connection = connection;
        currenciesRepository = new CurrenciesRepositoryImpl(connection); // ? лучше принимать в конструктор
    }
```

### package service

- Метод называется crossRateCalculation - во первых это существительное, а не глагол, во вторых он возвращает прямой
  курс, если он существует.
- В конструктор должен приниматься сам exchangeRatesRepository, а не Connection.
- Возвращаешь хемшапу вместо DTO, которые уже есть в проекте - ошибка. Сервис должен возвращать дто для контроллера.
- У `ExchangeRatesRepository exchangeRatesRepository;` нет модификатора доступа.
- Вместо `System.err.println` стоит выбрасывать исключение.

```java
        } catch (SQLException e) {
            System.err.println("Ошибка при работе с БД: " + e.getMessage());
        }
```

- По всему проекту бесконечные комментарии, что затрудняет чтение кода. Вроде ты хотел сделать их в виде Java doc, тогда
  стоит сделать это в интерфейсе.
- Хотелось бы сделать метод короче засчет вспомогательных методов, сделать код читабельнее и тогда не будет нужды в
  комментариях.
- Очень жаль, что весь сервисный слой содержит лишь один метод, что означает использование репозитория в самом
  контроллере - контроллер не должен ничего знать про модель.

### package servlets

- Нет модификатор полей в сервлетах.
- Чуть ли каждый сервлет супер пере раздутый, в одном аж 338 строк - антипаттерн "Fat controller". Стоит стремиться к
  тонкому контроллеру - буквально вызов сервиса и возращение результата, все остальное - валидация, обработка ошибок, не
  дай бог какая логика - не должны присутствовать в контроллере. Их стоит вынести либо в фильтры или в сервис.
- После `response.sendError` - обязательно возвращать `return`. Сам по себе он не останавливает выполнение метода.
- В `CurrencyServlet` где надо вернуть конкретную валюты ты вызываешь getCurrencies() и фильтруешь по нужному тебе - не
  эффективно.
- Каждый раз записываешь ошибку в сессию `request.getSession().setAttribute("jsonError", jsonError);`, когда REST API
  наш stateless.
- Много дубликатов кода - вынести в утилитные классы.
- Также рекомендую ознакомиться с паттерном `composition root`, дабы не создавать через new в init зависимости.
- Также обязательно ознакомится с мапперами.

### package dto

- Правильно называть `CurrencyDto` вместо `CurrencyDTO`.
- Неиспользуемый класс `CurrenciesDTO`.
- Все дто должны быть иммутабельны, что не соответствует ни одному классу из пакета.

### package utils

- Вроде бы утилитарные классы. но содержат в себе нестатические методы.

## Общее

- Не игнорируй замечания от idea - чаще всего они полезны.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.
- Неплохо было бы оформлять коммиты в соответствии
  с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Много комментариев - можешь отдельно почитать почему это плохо - _Мартин_, _"Чистый Код"_, гл.4, _Комментарии_
- Можно добавить готовый фронт.
- Советую посмотреть [лекцию Сергея про MVC паттерн](https://www.youtube.com/live/syjOb_jPJWE?si=CLXIVWWNdujQQMwC)
- Советую ознакомиться с дополнительными материалами по 3-ему проекту доступные через подписку.

## Итог

- Проект с нюансами, которые важно проработать во время рефакторинга, после можно идти дальше. Удачи!