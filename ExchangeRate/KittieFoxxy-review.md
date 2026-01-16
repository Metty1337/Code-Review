# Review на реализацию от [@KittieFoxxy](https://github.com/KittieFoxxy) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/KittieFoxxy/currency-exchange)

## Реализация REST API

- Можно добавить более информативное сообщение при ошибке валидации по длине (например минимальное и/или максимальное
  кол-во символов)

```
// сейчас
{
    "message": "Name is very long."
}
// например так
{
  "message": "Name is very long. Max 20 symbols."
}
```

- Сейчас можно создать валюту с одно буквенным именем, неплохо было бы добавить минимальную длину для имени.
- Правильная форма ответа у ендпоинтов в соответствии ТЗ, что хорошо.

## По коду

### package entity

- Пакет называется _entity_, хотя хранится в нем скорее _model/domain_ классы. Распространенно, что в _Java_-экосистеме
  _entity_ - это в
  первую очередь _JPA Entity_, с которым познакомитесь в четвертом проекте. Рекомендация, а не ошибка.
- Класс _Currency_:
    - Неиспользуемый метод `void setCode(String code)`, стоит убрать, если не нужен.
    - В целом класс не предполагает какие-либо мутации полей, тогда было бы неплохо добавить к полям модификатор _final_
      и быть может даже рассмотреть использование _record_. Опциональное улучшение.
    - Контракты _equals()_ и _hashCode()_ рекомендуется делать вокруг иммутабельных полей. Если же мы хотим оставить
      поля мутабельными, то в таком случае логичным было бы оставить в контракте поле _code_.
    - `@JsonProperty("name")` - эта аннотация явный признак, что модель (или _entity_) используют как _DTO_ - нарушение
      _SRP_. Решение - использовать _DTO_ для передачи данных между сервисом и контроллером.
- Класс _ExchangeRate_:
    - В целом класс не предполагает какие-либо мутации полей, кроме поля _rate_, тогда было бы неплохо добавить к полям
      модификатор _final_. Опциональное улучшение.
    - Контракты _equals()_ и _hashCode()_ рекомендуется делать вокруг иммутабельных полей. Если же мы хотим оставить
      поля мутабельными, то в таком случае логичным было бы оставить в контракте только поле _code_.

### package repository

- Пакет носит название репозитория, но на самом деле является _DAO_ классом, т.к. работает напрямую с _JDBC_, что
  является слишком низким уровнем абстракции для репозитория.
- Хорошее использование интерфейсов.
- Класс _JdbcCurrencyRepository_:
    - Заменяй магические строки на константы.
    - Текст запроса удобнее читать, когда он логично разбит на строки, для этого можно использовать текстовые блоки.
    - Метод `Optional<Currency> save(Currency currency)` возвращает _Optional_, что некорректно, ведь случай
      несохранения данных - исключительная ситуация и должна обрабатываться в слое работы с данными, а не где-то в
      сервисе, ведь это не является бизнес-логикой.
    - В случае исключения в базе данных выбрасывается обычный _RuntimeException_ вместо кастомного, стоит сделать также
      кастомное исключение, как и в других местах.
  ```
  // вместо
  throw new RuntimeException("Database error: ", e);

  // лучше
  throw new DatabaseException(DATABASE_EXCEPTION_MESSAGE, e);
  ```
    - Повторяющаяся часть кода, стоит вынести во вспомогательные методы - сделает методы меньше, улучшит
      читабельность:
    ```
    new Currency(
                        resultSet.getLong("id"),
                        resultSet.getString("code"),
                        resultSet.getString("full_name"),
                        resultSet.getString("sign"))
                )
    ```
- Класс _JdbcExchangeRateRepository_:
    - Неплохие _SQL_ запросы, решающие проблему _N+1_.
    - Текст запроса удобнее читать, когда он логично разбит на строки, для этого можно использовать текстовые блоки.

    - Метод `Optional<ExchangeRate> save(ExchangeRate exchangeRate)` возвращает _Optional_, что некорректно, ведь
      случай
      несохранения данных - исключительная ситуация и должна обрабатываться в слое работы с данными, а не где-то в
      сервисе, ведь это не является бизнес-логикой.

### package service

- Большой плюс, что есть интерфейсы к сервисам.
- Во всех методах сервиса, кроме `ConvertResponse exchange(String from, String to, BigDecimal amount)` возвращается
  модель. Лучше возвращать предназначенные для такого рода задач объекты - _DTO_.
- Класс _DefaultExchangeService_:
    - В большинстве случаев комментарии в коде - признак трудного читаемого кода, который необходимо переписать. На
      примере метода `Optional<ExchangeRate> findExchangeRate(String from, String to)`, где у нас встречаются
      комментарии - мы можем от них избавится достаточно несложной декомпозицией.
  ```
   private Optional<ExchangeRate> findExchangeRate(String from, String to) {
        // Прямой курс
        Optional<ExchangeRate> direct = repository.findByCodePair(from, to);
        if (direct.isPresent()) return direct;

        // Обратный курс
        Optional<ExchangeRate> reverse = repository.findByCodePair(to, from);
        if (reverse.isPresent()) {
            BigDecimal rate = BigDecimal.ONE.divide(reverse.get().getRate(), 6, RoundingMode.HALF_EVEN);
            return Optional.of(new ExchangeRate(null, reverse.get().getTargetCurrency(), reverse.get().getBaseCurrency(), rate));
        }

        // Кросс-курс через USD
        return findCrossRate(from, to);
    }
  
  // лучше - нет необходимости в комментариях, код стал самодокументируемым.
   private Optional<ExchangeRate> findExchangeRate(String from, String to) {
        Optional<ExchangeRate> direct = findDirectExchange(from, to);
        if (direct.isPresent()) return direct;

        Optional<ExchangeRate> reverse = findReverseExchange(from, to);
        if (reverse.isPresent()) {
            BigDecimal rate = BigDecimal.ONE.divide(reverse.get().getRate(), 6, RoundingMode.HALF_EVEN);
            return Optional.of(new ExchangeRate(null, reverse.get().getTargetCurrency(), reverse.get().getBaseCurrency(), rate));
        }

        return findCrossRate(from, to);
    }
  ```
    - Чтобы дополнить мысль о важности декомпозиции прикреплю практические идеальный метод, который мог бы быть для
      класса _ExchangeService_:
  ```
  public ExchangeDto getExchange(String base, String target, BigDecimal amount) {
              return findDirect(base, target).or(() -> findReverse(base, target))
                                             .or(() -> findCross(base, target))
                                             .orElseThrow(() -> ModelNotFoundException(...));}
  ```

### package servlet

- В сервлетах из сервисов везде получаешь модель, а лучше сразу же получать _DTO_. Таким образом наши сервлеты не будут
  ничего знать о моделях, а методы станут тоньше.
- В сервлетах обрабатываются исключения - нарушение _SRP_, стоит вынести в отдельный фильтр, который будет отвечать за
  обработку исключений.
- Те немногие фильтры, которые есть в классе сервлетов, было бы хорошо вынести в отдельный пакет. Как и
  _ApplicationContextListener_ в отдельный пакет.
- Метод `void doPatch(HttpServletRequest req, HttpServletResponse resp)` получился довольно трудно читаемым, было бы
  неплохо его улучшить с помощью вспомогательных методов.

### package response

- Пакет выполняет роль _DTO_ - в целом можно переименовать _response_ в _DTO_ для ясности содержимого.
- _Record_ идеально подходит для подобного рода объектов.

### package util

- Для всех утилитарных классов стоит добавлять модификатор _final_ в сигнатуру класса, так же как и закрытый пустой конструктор.
- В ошибках валидации пробрасывать _IllegalArgumentException_, на мой взгляд, слишком обобщенно. Задумался бы о кастомных исключениях для ошибок валидации.

## Общее

- Опциональное улучшение - большая часть **boilerplate*-кода* может быть заменена *Lombok*'ом.
- Магические числа/строки стоит заменять константами.
- _ApplicationContextListener_ в качестве реализации паттерна _Composition Root_ вполне корректна.
- Неплохо было бы оформлять коммиты в соответствии
    с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Хорошо было бы узнать про правильное оформление _pom.xml_.
- В качестве улучшение проекта можно попробовать добавить логирование.
- Дополнительно про комментарии можешь почитать здесь - _Мартин_, _"Чистый Код"_, гл.4, _Комментарии_.

### Итог
- Считаю очень достойный проект, замечаний по минимуму. Сгладив углы рефакторингом - надо идти дальше. Успехов!

