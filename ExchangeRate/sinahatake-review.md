# Review на реализацию от [@sinahatake](https://github.com/sinahatake) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/sinahatake/currency-exchange-api)

## Реализация REST API

- Плюсы:
    - Ответы эндпоинтов полностью соответствуют ТЗ и подходят для фронта.
    - Есть валидация на длину кода валюты.
- Минусы:
    - Хотелось бы добавить ограничение на длину _Sign_, например не больше трех символов.
    - То же самое относится к длине имени валюты.
    - Допускается хранение значения _Exchange Rate_ только до 2 знаков после запятой, хотя по ТЗ необходимо 6.

## По коду

### package entity

- Для _id_ можно использовать _Long_ вместо _int_.
- Плюс, что используется _BigDecimal_ для _rate_.
- На твоем месте я бы сделал _ExchangeRate_ более похожим на schema, то есть:

```
// сейчас
public class ExchangeRate {
    private int id;
    private Currency baseCurrency;
    private Currency targetCurrency;
    private BigDecimal rate;
}

// лучше
public class ExchangeRate {
    private Long id;
    private Long baseCurrencyId;
    private Long targetCurrencyId;
    private BigDecimal rate;
}
```

Твой вариант больше подходит для _ORM_, который встретится в следующих проектах, но для простого JDBC это избыточная
сложность.

### package dao

- В целом принято, что константы пишутся как `static final`, а не `final static`.
- Неиспользуемые методы _delete_.
- Не стоит пробрасывать _SQLException_ в верхние слои, они не должны знать про _JDBC_. Лучше самому ловить исключения и
  выбрасывать понятные для бизнес-логики исключения. Например:

```
// сейчас
    public Currency save(Currency currency) throws SQLException {
        try (var connection = ConnectionManager.getConnection();
             var statement = connection.prepareStatement(SAVE_SQL, Statement.RETURN_GENERATED_KEYS)) {
            statement.setString(1, currency.getCode());
            statement.setString(2, currency.getFullName());
            statement.setString(3, currency.getSign());

            statement.executeUpdate();
            ResultSet keys = statement.getGeneratedKeys();
            if (keys.next()) {
                currency.setId(keys.getInt(1));
            }
            return currency;
        }
    }
    
// лучше 
    public Currency save(Currency currency) {
        try (var connection = ConnectionManager.getConnection();
             var statement = connection.prepareStatement(SAVE_SQL, Statement.RETURN_GENERATED_KEYS)) {
            statement.setString(1, currency.getCode());
            statement.setString(2, currency.getFullName());
            statement.setString(3, currency.getSign());

            statement.executeUpdate();
            ResultSet keys = statement.getGeneratedKeys();
            if (keys.next()) {
                currency.setId(keys.getInt(1));
            }
            return currency;
        } catch (SQLException e) {
            throw new DatabaseException(DATABASE_EXCEPTION_MESSAGE, e); // или другое нужное тебе исключение
        }
    }
```

- Неиспользуемый метод `boolean update(Currency currency)`.
- Рекомендую использовать более содержательные имена для локальных переменных.

```
// сейчас
List<Currency> list = new ArrayList<>();

// лучше
List<Currency> currencies = new ArrayList<>();
```

- Добавить чуть форматирования для длинного _SQL_ будет неплохо:

```
// сейчас
    private final static String FIND_BY_CURRENCY_CODES_SQL = """
            SELECT er.ID, er.Rate,
                                           c1.ID as BaseCurrencyId, c1.Code as BaseCode, c1.FullName as BaseName, c1.Sign as BaseSign,
                                           c2.ID as TargetCurrencyId, c2.Code as TargetCode, c2.FullName as TargetName, c2.Sign as TargetSign
                                    FROM ExchangeRates er
                                    JOIN Currencies c1 ON er.BaseCurrencyId = c1.ID
                                    JOIN Currencies c2 ON er.TargetCurrencyId = c2.ID
                                    WHERE c1.Code = ? AND c2.Code = ?
            """;

// лучше 
    private static final String FIND_BY_CURRENCY_CODES_SQL = """
             SELECT
                 er.ID, er.Rate,
                \s
                 c1.ID as BaseCurrencyId,
                 c1.Code as BaseCode,
                 c1.FullName as BaseName,
                 c1.Sign as BaseSign,
                \s
                 c2.ID as TargetCurrencyId,
                 c2.Code as TargetCode,
                 c2.FullName as TargetName,
                 c2.Sign as TargetSign
             FROM ExchangeRates er
             JOIN Currencies c1 ON er.BaseCurrencyId = c1.ID
             JOIN Currencies c2 ON er.TargetCurrencyId = c2.ID
             WHERE c1.Code = ? AND c2.Code = ?
            \s""";
```

- Неожиданно нарушение конвенции в _ExchangeRateDAO_ - конструктор стоит ниже методов, особенно странно, когда в проекте
  используется _Lombok_.
- Также классы принято называть в _PascalCase_ и это не исключение для _DAO_ классов, то есть не принято писать эту
  аббревиатуру заглавными буквами. Лучше _CurrencyDao_ и _ExchangeRateDao_.
- Было бы большим плюсом добавить интерфейсы для _DAO_ классов.
- Метод `List<ExchangeRate> findAll()` имеет проблемы с производительностью из-за _N + 1_ проблемы. Решение - оформить _SQL_ через _JOIN_ так же как и в _findByCurrencyCodes_.

### package service

- _CurrencyService_:
    - Заменил бы имя локальной переменной `currencyDAOAll` на чуть более содержательное `allCurrencies`.
    - `CurrencyDTO addNewCurrency(String code, String name, String sign)` - тернарная функция и передача такого кол-ва
      аргументов сильно затрудняет ее использование. Предлагаю упаковать аргументы в отдельную _DTO_.
- _ExchangeRateService_:

    - Заменил бы имя локальной переменной `exchangeRatesDao` на чуть более содержательное `allExchangeRates`.
    - `ExchangeRateDTO addExchangeRate(String baseCode, String targetCode, BigDecimal rate)` - тернарная функция и
      передача такого кол-ва
      аргументов сильно затрудняет ее использование. Предлагаю упаковать аргументы в отдельную _DTO_.
    - Стоит использовать более конкретные исключения:
    ```
    // сейчас
          if (!isUpdated) {
            throw new SQLException("Failed to update exchange rate in database");
        }
    // лучше
          if (!isUpdated) {
            throw new UpdateFailedException("Failed to update exchange rate in database");
        }
    ```
- _ExchangeService_:
    - `ExchangeResultDTO exchange(String baseCode, String targetCode, BigDecimal amount)` - тернарная функция и
      передача такого кол-ва
      аргументов сильно затрудняет ее использование. Предлагаю упаковать аргументы в отдельную _DTO_.
    - Метод `BigDecimal calculateRate(String baseCode, String targetCode)` не имеет модификатор доступа.
    - В _Java_ принято называть локальные переменные в _camelCase_, то есть не `UsdToBase`, а `usdToBase` и тд.


- _try-catch_ c _SQLException_ - то же нарушение SRP, DIP. Стоит оставить это в слое _DAO_.
- Валидацию можно вынести в отдельный класс, что сделает код более удобочитабельным.
- Многие методы пробрасывают в сигнатуре _SQLException_ - нарушение _SRP_, _DIP_. Стоит еще в _DAO_ слое "превращать" это исключение в более понятное для бизнес-логики исключение.
- Конструктор можно заменить аннотацией _@RequiredArgsConstructor_.
- Вместо подобного использования сеттеров сразу после инициализации можно создать отдельный конструктор (при условии,
  что поле с _ID_ у нас не примитив):

```
// сейчас
        Currency currency = new Currency();
        currency.setCode(code);
        currency.setFullName(name);
        currency.setSign(sign);
        
// лучше 
    public Currency(String code, String fullName, String sign) {
        this.code = code;
        this.fullName = fullName;
        this.sign = sign;
    }
    // service
    Currency currency = new Currency(code, name, sign);
```

- Также блоки тел _try-catch_ стоит выделять в отдельные функции, т.к. блоки _try-catch_ добавляют уровень шума и этим
  запутывают структуру кода.
- `rate.setScale(2, RoundingMode.HALF_UP)` - нарушение _ТЗ_, необходимо округлять хотя бы до 6 знаков после запятой.
- Было бы плюсом добавить интерфейсы для слоя сервиса.

### package servlet

- Методы void `writeError(HttpServletResponse response, int status, String message)` и
  `void writeJson(HttpServletResponse response, int status, Object data)` дублируют друг друга:

```
// сейчас
    protected void writeError(HttpServletResponse response, int status, String message) throws IOException {
        response.setStatus(status);
        response.setContentType("application/json;charset=UTF-8");

        ErrorResponseDTO error = new ErrorResponseDTO(message);
        objectMapper.writeValue(response.getWriter(), error);
    }

    protected void writeJson(HttpServletResponse response, int status, Object data) throws IOException {
        response.setStatus(status);
        response.setContentType("application/json;charset=UTF-8");
        objectMapper.writeValue(response.getWriter(), data);
    }
    
// лучше
    protected void writeJson(HttpServletResponse response, int status, Object data) throws IOException {
        response.setStatus(status);
        response.setContentType("application/json;charset=UTF-8");
        objectMapper.writeValue(response.getWriter(), data);
    }

    protected void writeError(HttpServletResponse response, int status, String message) throws IOException {
        ErrorResponseDTO error = new ErrorResponseDTO(message);
        writeJson(response, status, error);
    }    
```

- Несмотря на то, что в методе _writeJson_ уже уставливается _ContentType_ все равно в начале каждого сервлета
  устанавливается еще раз _ContentType_.

```
@Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setContentType("application/json;charset=UTF-8"); // лишнее

        logger.info("GET request for all exchange rates");
```

- Можно вынести "ловлю" исключения в отдельный _ExceptionsFilter_, тогда сервлеты станут тоньше и ближе к соблюдению
  _SRP_.
- Валидацию можно вынести в отдельный класс, что сделает код более удобочитабельным.
- Большой плюс, что имеется логирование, которое облегчает дебаг.
- _EncodingFilter_ я бы вынес в отдельный пакет.
- Также блоки тел _try-catch_ стоит выделять в отдельные функции, т.к. блоки _try-catch_ добавляют уровень шума и этим
  запутывают структуру кода.
- Большой плюс, что contoller передает _DTO_ во _view_, а не _Entity_.

### package config

- Если есть желание использовать _Service Locator_ паттерн, то лучше будет использовать встроенный
  `ApplicationContextListener implements ServletContextListener`.

### package dto

- Обычно аббревиатуру _DTO_ не пишут заглавными, и придерживаются нужного стиля, например _PascalCase_ для названий классов. То есть вместо _CurrencyDTO_ лучше _CurrencyDto_.

### package mapperDto

- Имена пакетом обычно пишутся в виде _com.example.project_. Из названия _mapperDto_ не ясно что там лежит, я бы оставил
  просто _mapper_.
- Как вариант улучшения - познакомится с _MapStruct_.

### package util

- Обычно для утилитарных классов ставят модификатор _final_ и закрытый конструктор.

```
// сейчас
public class ConnectionManager {
    private static final HikariDataSource ds;
// скрытый пустой конструктор

// лучше
public final class ConnectionManager {
    private static final HikariDataSource ds;
    private ConnectionManager() {}
```
### package test
- Большой плюс наличия интеграционного тестирования как для эндпоинтов, так и для сервисов.


## Общее

- Магические числа/строки стоит заменять константами.
- Неплохо было бы оформлять коммиты в соответствии
  с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Логику обработки исключений и конвертацию ошибок рекомендую вынести в отдельный *ExceptionHandlingFilter*.
- Хорошо, что используется логирование.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.
- Советую посмотреть [лекцию](https://www.youtube.com/live/syjOb_jPJWE?si=JMTHby_6-FoInmBr) Сергея про MVC, там как раз есть часть про обработку ошибок.

# Итог
- Считаю достойный проект, после рефакторинга можно смело идти дальше. Удачи!