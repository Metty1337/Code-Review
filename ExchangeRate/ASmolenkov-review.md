# Review на реализацию от [@ASmolenkov](https://github.com/ASmolenkov) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/ASmolenkov/CurrencyExchange)

## Реализация REST API

### Плюсы

- Имеется некоторая валидация на добавление новой валюты.
- По большой части правильная форма ответа у ендпоинтов в соответствии ТЗ.

### Замечания

- Можно добавить валюту без символа - нарушение ТЗ. В идеале надо показывать пользователю сообщение, что отсутствует
  нужно поле формы с кодом 400.

## По коду

### package model

- Неиспользуемая аннотация _@With_ в классе _ExchangeRate_. Но даже по смыслу эту аннотацию нет смысла использовать над
  всем классом, ведь у нас поменяться может только _rate_:

```
@Value
@With
@Builder
public class ExchangeRate {
    int id;
    Currency baseCurrency;
    Currency targetCurrency;
    BigDecimal rate;
}

// лучше 
@Value
@Builder
public class ExchangeRate {
    int id;
    Currency baseCurrency;
    Currency targetCurrency;
    @With
    BigDecimal rate;
}
```

- Неиспользуемый класс _Exchange_ - стоит убрать.
- Стоит рассмотреть использование _Long_ вместо _int_ для _id_.

### package dao

- (optional) В данный момент _DAO_ слой реализован для удобства использования сервисом. Как пример - метод
  `Currency save(Currency currency)`, который возвращает объект, хотя в канонической реализации _DAO_ с _JDBC_ метод
  _save_ ничего не должен возвращать. В чистой архитектуре слой _DAO_ ничего не должен знать каким образом его
  собираются использовать.
- Выбрасываешь исключение _ModelNotFoundException_, которое относится к бизнес-логике. Как вариант решения - в методах
  по типу `Currency findByCode` стоит возвращать объект через _Optional_ и обрабатывать случай ненахождения модели в
  слое сервиса. Плюс это сделает методы `boolean existsByCode(String code)` и
  `boolean existsByCode(String baseCode, String targetCode)` ненужными.
- Как вариант улучшения _SQL_ запросов: вместо `SELECT *` прописывай конкретно, что тебе нужно. Таким образом при
  изменении _schema_ твое приложение будет работать корректно.

```
private static final String SQL_FIND_ALL = """
            SELECT * FROM currencies
            """;
            
// лучше

private static final String SQL_FIND_ALL = """
            SELECT id, code, full_name, sign FROM currencies
            """;
```

- Для меня осталось неясным зачем ты усложняешь запрос подзапросами, когда можно напрямую искать по _id_.

```
private static final String SQL_INSERT = """
            INSERT INTO exchange_rates(base_currency_id, target_currency_id, rate)
            VALUES ((SELECT (currencies.id)
                     FROM currencies
                     WHERE code = ?), (SELECT (currencies.id)
                                           FROM currencies
                                           WHERE code = ?), ?)
            """;
// лучше
private static final String SQL_INSERT = """
            INSERT INTO exchange_rates(base_currency_id, target_currency_id, rate)
            VALUES (?, ?, ?)
            """;
```

- Точно также здесь:

```
private static final String SQL_UPDATE = """
            UPDATE exchange_rates
            SET rate = ?
            WHERE base_currency_id = (SELECT (id) FROM currencies WHERE code = ?)
              AND target_currency_id = (SELECT (id) FROM currencies WHERE code = ?);
            """;
//лучше
private static final String SQL_UPDATE = """
            UPDATE exchange_rates
            SET rate = ?
            WHERE base_currency_id = (?)
              AND target_currency_id = (?);
            """;
```

- Хорошее использование _JOIN_ для решения _N+1_ проблемы.

### package service

- Добавление unchecked исключения в сигнатуру метода - обычно лишнее.
- Считается хорошей практикой при использовании _Stream API_ разбивать на несколько строк по операциям (такое
  форматирование также можно настроить по-умолчанию в _idea_):

```
return jdbsCurrencyDao.findAll().stream().map(CurrencyMapper::toResponse).collect(Collectors.toList());

//лучше
return jdbsCurrencyDao.findAll()
                      .stream()
                      .map(CurrencyMapper::toResponse)
                      .collect(Collectors.toList());
```

Относится не только к этому примеру.

- Поля класса _ExchangeRatesService_ вполне могут быть _final_ - не игнорируй подсказки _idea_.
- Сбивающее с толку название метода
  `ExchangeRatesResponseDto getExchangeRatesByCode(String baseCode, String targetCode)` - почему-то _ExchangeRates_ в
  множественном числе, хотя возвращаем лишь один объект. То же относиться и к _createExchangeRates_ и соответствующим
  _DTO_. Множественное число обычно означает какую-то коллекцию.
- Отсутствует модификатор доступа поля у _ExchangeService_.
    - _ExchangeService_:
        - _CurrencyDao_ - сервис не должен напрямую использовать _DAO_ другого сервиса, если уже
          есть существующий сервис, который инкапсулирует работу с этим _DAO_.
        - _buildExchangeDto_ - метод, который больше относиться к ответственности маппера, а не сервиса.
        - Главная проблема - использование _try-catch_ как _if-else_:
            1. Это дорого по производительности.
            2. Исключения - это ошибки, а не механизм ветвления логики.
            3. К тому же ты игнорируешь исключения, из-за чего теряется контекст этих самих исключений.
            4. Как вариант улучшения - считаю необходимым воспользоваться вспомогательными методами, которые будут
               возвращать нам _Optional_ (еще из _DAO_), что кратно улучшит читабельность нашего кода.
          ```
          public ExchangeDto getExchange(String baseCurrency, String targetCurrency, BigDecimal amount) {
              try {
                  ExchangeRate direct = exchangeRatesDao.findByCode(baseCurrency, targetCurrency);
                  return buildExchangeDto(direct.getBaseCurrency(), direct.getTargetCurrency(), direct.getRate(), amount);
    
              } catch (ModelNotFoundException ignored) {
    
              }
              try {
                ...
          // лучше 
          public ExchangeDto getExchange(String base, String target, BigDecimal amount) {
              return findDirect(base, target).or(() -> findReverse(base, target))
                                             .or(() -> findCross(base, target))
                                             .orElseThrow(() -> ModelNotFoundException(...));}
          ```

### package servlet

- (optional) В большинстве Java-проектов для констант используется формат: _NOUN_QUALIFIER_. То есть предпочтительнее
  называть
  константы - _NAME_PARAM_, а не _PARAM_NAME_.
- Часто встречается _ServletException_ в сигнатурах методов, через которые никогда не пройдет данное исключение - стоит
  убирать.
- Неиспользуемая аннотация _@Slf4j_ в множестве сервлетов.
- В целом неплохой пакет сервлетов.

### package config

- В пакете все классы являются фильтрами - почему бы также не назвать пакет?
- В целом стоит избегать комментариев - особенно, когда они _ИИ_-шные.
- Метод `void handleException(Exception e, HttpServletResponse response)` имеет ряд недостатков:
    1. Она велика, а при добавлении новых ошибок она будет разрастаться.
    2. Она нарушает принцип _SRP_, т.к. у нее существует несколько возможных причин изменения.
    3. Она нарушает принцип _OCP_.
- Как вариант улучшения:
    1. Ввести объект ошибки.
    2. Добавить фабрику ошибок.

У нас получиться более хороший вариант:

```
    private void handleException(Exception e, HttpServletResponse response) throws IOException {
        int status;
        String message;
        if (e instanceof ValidationException) {
            status = HttpServletResponse.SC_BAD_REQUEST;
            message = e.getMessage();
        } else if (e instanceof ModelNotFoundException) {
            status = HttpServletResponse.SC_NOT_FOUND;
        ...
    
// лучше
private void handleException(Exception e, HttpServletResponse response) throws IOException {
    ErrorResponse error = errorResponseFactory.from(e);

    log.error(e.getMessage(), e);
    JsonUtil.sendError(error.message(), error.status(), response);
}
```

Дополнительно прочитать про такие случаи - _Роберт Мартин, "Чистый код", Гл. 3 Функции, Команды switch_

### package mapper

- Неиспользуемый метод `CurrencyResponseDto resultSetToResponse(ResultSet resultSet)`.

### package utils

- Текущее решение в виде _ApplicationConfig_ не лучший вариант. Если есть желание использовать _Service Locator_
  паттерн, то лучше будет использовать встроенный _ServletContextListener_.
- В проекте с десяток кастомных исключений, но в _DatabaseManager_ все равно встречается _RuntimeException_. И в отличие
  от других мест - сообщение об ошибке захардкожены.
- В _executeSqlScript_ _InputStream_ используется без _try-with-resources_.
- Неоднородно используешь @UtilityClass - где-то есть, где-то нет - луче привести к единому формату.

## Общее

- Хорошо, что используется логгирование.
- Используй чаще форматирование (ctrl+alt+l в idea) - есть неаккуратные места.
- Не игнорируй замечания от idea - чаще всего они полезны.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.
- Магические числа/строки стоит заменять константами.
- Неплохо было бы оформлять коммиты в соответствии
  с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Хорошо было бы узнать про правильное оформление _pom.xml_, в том числе как настроить аннотации _lombok_ в нем. Для
  локальной сборки пришлось самому настраивать.
- groupId сейчас - by.smolenok, что не
  соответствует [конвенции](https://maven.apache.org/guides/mini/guide-naming-conventions.html).

# Итог

- Считаю очень достойный проект, замечаний меньше, чем обычно - после исправления незначительных мест можно смело идти дальше. Успехов!