# Review на реализацию от [@Sibiryaq](https://github.com/Sibiryaq) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/Sibiryaq/currency-exchange)

## Реализация

- Плюсы:
    - Ответы эндпоинтов полностью соответствуют ТЗ и подходят для фронта.
    - Есть валидация длины символа валюты.
- Замечания:
    - Нет валидации длины кода валюты.
    - Нет валидации длины имени валюты.

## По коду

### package model

- Не используется пустой конструктор, не стоит оставлять в коде.
- (Архитектурное предпочтение) Советую не использовать сеттер для `id`, чтобы защититься от случайного неправильного
  использования. Лучше использовать специальный конструктор или паттерн _With_, оставляя класс иммутабельным.
- Сеттеры для других полей не используются, если следовать предыщему комментарию, то можно вовсе конвертировать _POJO_ в
  _record_.
- `Long` для `id` - хорошо.
- `BigDecimal` для `rate` - хорошо.

### package dao

- Стоит давать более информативные имена локальным переменным:

  ```java
  String sql = "SELECT id, code, full_name, sign FROM currencies";
         
  // лучше 
  String findAllSql = "SELECT id, code, full_name, sign FROM currencies";
  // etc
  ```

- Правильнее будет оборачивать `ResultSet rs = ps.executeQuery();` в `try-with-resources`:

  ```java
  
      try (Connection conn = ConnectionManager.getConnection();
           PreparedStatement ps = conn.prepareStatement(sql)) {

          ps.setString(1, code);
          ResultSet rs = ps.executeQuery();
  
  // лучше
      try (Connection conn = ConnectionManager.getConnection();
           PreparedStatement ps = conn.prepareStatement(sql)) {

          ps.setString(1, code);
          try (ResultSet rs = ps.executeQuery()) {
  ...
  ```

- Никогда не возвращай `null` - это повышает шанс `NPE`. Для таких случаев идеально подходит `Optional`:

  ```java
    public Currency findByCode(String code) {
          ...
        return null;

  // лучше 
  
    public Optional<Currency> findByCode(String code) {
      ...
        return Optional.empty();
  ```
- Не оборачивай в голый `RuntimeException`. Создай кастомный `DatabaseException` - это будет понятное бизнес-логике исключение.
```
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
        
        // лучше
        } catch (SQLException e) {
            throw new DatabaseException("Failed to update exchange rate", e);
        }
```
- Не создавай `ExchangeRateDao` внутри класса, вместо используй `this`:
```java
            ExchangeRateDao dao = new ExchangeRateDao();
            return dao.findByCurrencies(baseCurrencyId, targetCurrencyId);
            
            // лучше
            return this.findByCurrencies(baseCurrencyId, targetCurrencyId);
```

-  Метод `ExchangeRate update(Long baseCurrencyId, Long targetCurrencyId, BigDecimal newRate)` - выполняет 2 _SQL_, в таком случае надо сделать этот метод атомарным:
```java
    public ExchangeRate update(Long baseCurrencyId, Long targetCurrencyId, BigDecimal newRate) {
        String sql = "UPDATE exchange_rates SET rate = ? WHERE base_currency_id = ? AND target_currency_id = ?";

        try (Connection conn = ConnectionManager.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setBigDecimal(1, newRate);
            ps.setLong(2, baseCurrencyId);
            ps.setLong(3, targetCurrencyId);

            int updatedRows = ps.executeUpdate();

            if (updatedRows == 0) {
                return null;
            }

            ExchangeRateDao dao = new ExchangeRateDao();
            return dao.findByCurrencies(baseCurrencyId, targetCurrencyId);

        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
    
    // лучше
    
public ExchangeRate update(Long baseCurrencyId, Long targetCurrencyId, BigDecimal newRate) {
    String updateSql = """
        UPDATE exchange_rates SET rate = ?
        WHERE base_currency_id = ? AND target_currency_id = ?
        """;

// также можем вынести этот запрос в консанты и использовать его также в findByCurrencies
    String selectSql = """
        SELECT er.id, er.rate,
               bc.id AS base_id, bc.code AS base_code, bc.full_name AS base_name, bc.sign AS base_sign,
               tc.id AS target_id, tc.code AS target_code, tc.full_name AS target_name, tc.sign AS target_sign
        FROM exchange_rates er
        JOIN currencies bc ON er.base_currency_id = bc.id
        JOIN currencies tc ON er.target_currency_id = tc.id
        WHERE er.base_currency_id = ? AND er.target_currency_id = ?
        """;

    try (Connection conn = ConnectionManager.getConnection()) {
        conn.setAutoCommit(false); // вручную менеджим транзакцию

        try {
            try (PreparedStatement ps = conn.prepareStatement(updateSql)) {
                ps.setBigDecimal(1, newRate);
                ps.setLong(2, baseCurrencyId);
                ps.setLong(3, targetCurrencyId);

                if (ps.executeUpdate() == 0) {
                    throw new ExchangeRateNotFoundException(baseCurrencyId, targetCurrencyId); // не возвращаем null, вместо этого пробрасываем понятное бизнес-логике исключение
                }
            }

            try (PreparedStatement ps = conn.prepareStatement(selectSql)) {
                ps.setLong(1, baseCurrencyId);
                ps.setLong(2, targetCurrencyId);

                ResultSet rs = ps.executeQuery();
                rs.next();
                conn.commit();
                return mapRow(rs);
            }

        } catch (SQLException e) {
            conn.rollback(); // rollback всей транзакции при исключении
            throw e;
        }

    } catch (SQLException e) {
        throw new DatabaseException("Failed to update exchange rate", e);
    }
}
```
### package service

- Для зависимостей класса стоит использовать _Constructor Injection_, а не инициализировать их в прямо в полях, чтобы было ясно от чего зависит класс и было легче подменять зависимости при тестировании:
```java
public class CurrencyService {
    private final CurrencyDao currencyDao = new CurrencyDao();
    
// лучше
public class CurrencyService {
    private final CurrencyDao currencyDao;

    public CurrencyService(CurrencyDao currencyDao) {
        this.currencyDao = currencyDao;
    }
```
- `ExchangeRateService` использует `CurrencyDao` для поиска валюты, тем самым нарушая абстракцию и инкапсуляцию, ведь есть `CurrencyService`, который инкапсулирует работу с валютами:
```java
public class ExchangeRateService {
    private final CurrencyDao currencyDao;
    
// лучше
public class ExchangeRateService {
    private final CurrencyService currencyService;
```
- Лживый комментарий:
```java

        if (base == null || target == null) {
            throw new IllegalArgumentException("Currency not found"); // ← не RuntimeException!
        }
        ...
        public class IllegalArgumentException extends RuntimeException {} // <- RuntimeException

```

- Каждый метод слоя сервиса возвращает модель, а не `DTO`, из-за чего контроллер начинает знать детали внутренней структуре данных.
```java
    public List<ExchangeRate> getAllRates() {
        return exchangeRateDao.findAll();
    }
    
// лучше

    public List<ExchangeRateDto> getAllRates() {
        return exchangeRateDao.findAll().stream()
                .map(ExchangeRateMapper::toDto)
                .toList();
    }
```
- Не стоит возвращать `null` из-за повышения шанса _NPE_, лучше вернуть исключение:
```java
        if (baseCode == null || targetCode == null) return null;

// лучше
        if (baseCode == null || targetCode == null){
            throw new ValidationException("Validation failed.");
        }
```
- Нарушение _DRY_, выноси повторяющиеся блоки во вспомогательные методы:
```java
        if (base == null || target == null) {
            throw new IllegalArgumentException("Currency not found");
        }
// etc

// лучше
        if (isValid(base, target)) {
            throw new IllegalArgumentException("Currency not found");
        } // etc
        
    private static boolean isValid(Currency base, Currency target) {
        return base == null || target == null;
    }
```
### package controller

- Сервису лучше давать контроллеру сразу _DTO_, чтобы не раскрывать детали структуры данных.
- Зависимости инициализированы сразу в поле. Более предпочтительным было бы использовать встроенный `init()` или реализовать паттерн _composition root_ через `AppInitializer implements ServletContextListener`:
```java
public class CurrenciesServlet extends HttpServlet {
    private final CurrencyService currencyService = new CurrencyService();

// пример как можно сделать, но уже со своими зависимостями
@WebListener
public class AppInitializer implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        DataSource ds = createDataSource();
        UserRepository userRepo = new JdbcUserRepository(ds);
        UserService userService = new UserServiceImpl(userRepo);

        ServletContext ctx = sce.getServletContext();
        ctx.setAttribute("userService", userService);
    }
}

@WebServlet("/users")
public class UserServlet extends HttpServlet {

    private UserService userService;

    @Override
    public void init() {
        userService = (UserService) getServletContext().getAttribute("userService");
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // используешь userService
    }
}
```
- Для исключений можно выделить отдельный `ExceptionFilter`, тем самым сделав сервлеты тоньше.
- Валидацию можно вынести в отдельный класс, тем самым скрыв детали реализации и сделав код более читабельным.
- Вместо `e.printStackTrace();` стоит использовать `java.util.logging.Logger` или _SLF4J_.
### package dto

- _DTO_ - должен быть иммутабельным классом, а значит иметь модификатор `private` на полях и иметь один полный конструктор вместе с геттерами:
```java
public class CurrencyDto {
    public Long id;
    public String code;
    public String name;
    public String sign;
}

// лучше

public class CurrencyDto {
    private Long id;
    private String code;
    private String name;
    private String sign;
    // конструктор, геттеры
}
```
- `DTO` курса состоит из `DTO` валют - хорошо.

### package util

- Утилитарные классы всегда должны иметь модификатор `final` и приватный конструктор.
- Классам, связанным с инициализацией _DB_, можно добавить кастомные исключения.
- `EncodingFilter implements Filter` - не утилитарный класс, стоит отнести в другой пакет.
- `DatabaseInitializer implements ServletContextListener` - не утилитарный класс, стоит отнести в другой пакет.

## Общее

- С этого проекта можно попробовать _Lombok_, который дает возможность уменьшить _boilerplate_ код засчет
  аннотаций. 
- Можно добавить пакет с кастомными исключениями.
- В _readme_ для сборки написан самостоятельный деплой артефакта в _Tomcat_, хотя в проекте содержится _Dockerfile_.
- Не стоит игнорировать замечания от *IDEA* - почти всегда это по делу, если ты специально не хочешь сделать по другому.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.

## Итог
- Работающий проект, после рефакторинга недостающих моментов можно смело идти дальше. Удачи!