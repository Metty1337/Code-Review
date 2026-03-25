# Review на реализацию от [@NikitaPopovsky](https://github.com/NikitaPopovsky) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/NikitaPopovsky/CurrencyExchange)

## Реализация

Смотрел по деплою:

- Хорошо:
    - Форма ответов эндпоинтов в большей степени соответствуют ТЗ и подходят для фронта.
- Замечания:
    - Исходя из отчета бота [@currency_exchange_api_bot](https://t.me/currency_exchange_api_bot) некоторые _HTTP_
      response status codes не сходятся с _ТЗ_.
    - Неправильно считается кросс-курс.

## По коду

### package model

- Концептуально для `id` лучше подходит `Long`.
- Класс `Exchange` относится скорее к `DTO`, ведь у класса нет `id`. Полагаю ты решил также, ведь в проекте есть свой
  `ExchangeDTO`, а сам `Exchange` нигде не используется

### package db

- Классы в _Java_ называют в стиле _UpperCamelCase_, это касается, в том числе сокращений.
- Также лучше не создавать свои зависимости внутри конструктора, это жесткая связность - невозможно подменить
  зависимости. Стоит обратиться к _Dependency Injection_ подходу, благодаря которому ясно видно какие зависимости нужны
  классу. Упомяну, что это происходит в каждом классе, где есть зависимости, чтобы в дальнейшем не повторятся.
- Пробрасываешь кастомные исключения понятные бизнес-логике - хорошо. Но по конвенции желательно, чтобы все исключения
  имели суффикс `Exception`. Примеры из стандартной библиотеки:
    - `IOException`
    - `SQLException`
    - `IllegalArgumentException`
    - `NullPointerException`
- Также обязательно при возможности надо пробрасывать _cause_ в исключение, а не съедать его:

```java
        } catch (SQLException e) {
            throw new DataBaseUnavailable(ExceptionMessage.DB_NOT_UNAVAILABLE.getMessage());
        }
        
// лучше
        } catch (SQLException e) {
            throw new DataBaseUnavailable(ExceptionMessage.DB_NOT_UNAVAILABLE.getMessage(), e);
        }
```

- Также лучше не пробрасывать пустое исключение, нужно добавлять контекст:

```java
            } else {
                throw new SQLException();
            }
            
// лучше
            } else {
                throw new SQLException("Insert failed: no rows affected");
            }
```

- _SQL_ запрос удобнее читать, когда он удобно разбит на строки. Для этого можно использовать текстовые блоки:

```java
"SELECT id, base_currency_id, target_currency_id, rate FROM exchange_rates"

// лучше
"""
SELECT id, base_currency_id, target_currency_id, rate
FROM exchange_rates
"""
```

- Название `UtilDAO` не очень хорошо описывает, что происходит в классе, ведь обычно `DAO` это работа с конкретной
  сущностью, а на самом деле здесь лежит инфраструктура. Более подходящее название `DataSourceProvider`. А так как класс
  утилитарный, то стоит также поставить модификатор `final` для класса и сделать закрытый конструктор.

- `Optional` для поиска - хорошо.

- Все сообщения об исключениях писать надо только по-английски:

```java
            throw new DataBaseUnavailable("Ошибка драйвера подключения к БД");

// лучше
            throw new DataBaseUnavailable("Database connection driver error");

```

- Захардкожены настройки для конфигурации, лучше конечно получать их из вне:

```java
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        config.setIdleTimeout(300000);
        config.setConnectionTimeout(30000);
        config.setLeakDetectionThreshold(60000);
```

- В _DAO_ активно используется `ExchangeRateDAOMapper`, который самостоятельно ходит в _БД_ за каждой валютой -
  нарушение _SRP_, а также _N+1_ проблема. Решение - использовать `JOIN` в _SQL_ запросах, чтобы сразу вытягивать данные
  валют.
- В методах `save` `ResultSet` никогда не закрывается - утечка ресурсов.
- Отсутствуют интерфейсы для _DAO_ из-за чего не представляется возможным замена реализации и использования
  полиморфизма.

### service

- Для создания валюты ты ходишь в базу с запросом на поиск, чтобы проверить не создана ли валюта уже, но из минусов -
  приходиться делать лишний запрос. В данной ситуации можно положится на _unique constraint_ твоей _БД_ похожим образом,
  что исключит возможные _race condition_ проблемы:

```java
    public CurrencyDTO create (CurrencyDTO currencyDTO) {
        Optional <Currency>  currencyOptional = dao.findByCode(currencyDTO.code());

        if (currencyOptional.isPresent()) {
            throw new ObjectAlreadyExists("Валюта с такими полями уже существет");
        }

        Currency currency = dao.save(mapper.toModel(currencyDTO));
        return mapper.toDTO(currency);
    }

// лучше
// в ExceptionHandlerFilter ловишь например исключение CurrencyAlreadyExistException, которое выбросит _DAO_
public CurrencyDTO create(CurrencyDTO currencyDTO) {
        Currency currency = dao.save(mapper.toModel(currencyDTO));
        return mapper.toDTO(currency);
}
```

- На мой взгляд логика обмена валюты достойна отдельного `ExchangeService`.
- Стоит использовать интерфейс вместо конкретной реализации:

```java
HashMap<String, String> codes = splitCodes(pairCode);
// лучше
Map<String, String> codes = splitCodes(pairCode);
```

- Для кодов валют используется `HashMap` с магическими `from` и `to`. Это очень хрупко. Стоит сделать простой `record`:

```java
    private record CurrencyPair(String from, String to) {
    }
```

- Метод обмена валюты использует исключения как _flow control_ - анти-паттерн. Тем более не стоит проглатывать
  исключения. Лучше использовать `Optional` и проверки.
- Также бессмысленный try-catch, который ничего не делает:

```java
        } catch (RuntimeException e) {
            throw e;
        }
```

- Этот же метод `findForExchange` получился довольно громоздким и плохо читаемым. Стоит воспользоваться вспомогательными
  методами.
- Здесь же и ошибка почему неправильно работает кросс-курс, неправильный порядок валют:

```java
            return new ExchangeRate(0, exchangeRateUSDToBase.targetCurrency(),
                    exchangeRateUSDToTarget.targetCurrency(),  rate);
```

- По _ТЗ_ курс должен быть десятичным числом с шестью знаками:

```java
            BigDecimal rate = exchangeRateUSDToBase.rate()
                    .divide(exchangeRateUSDToTarget.rate(), 3, RoundingMode.HALF_UP); // всего 3
```

### servlet

- Слой контроллера общается с сервисом только через _DTO_ - хорошо.
- _POST_ эндпоинты в случае успеха возвращают 200 вместо 201.
- Не хватает `null` проверок для параметров.
- Сейчас происходит дублирование экземпляров сервисов, каждый сервелет создает свой сервис, а тот создает свой
  маппер/дао и т.д. Решением видится реализация паттерна _composition root_, например таким образом:

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

### package filters

- Название `HandleExceptionFilter` класса должно быть существительным, то есть `ExceptionHandlerFilter`.
- По _ТЗ_ ответ в случае ошибки:

```json
{
  "message": "Валюта не найдена"
}
``` 

- Фильтр ловит только `CurrencyExchangeException`, а при любом другом исключении выдаст клиенту _HTML_ страничку вместо
  _JSON_.

а не исключение с внутренностями.

- Также не хватает конкретной установки `Content-Type`.

### package util

- Утилитарным классам не хватает приватного конструктора.
- При любой ошибке валидации пробрасывается один и тот же `CodeIsMissing`, даже если это касалось имени валюты.

### package mapper

- Как и упомянул ранее - маппер не может сам делать запросы в БД.
- Не совсем понял почему некоторые мапперы используются как полноценные объекты, а другие как утилитарные классы.
  Хотелось бы, чтобы все были полноценными объектами.
- Можно чуть упростить названия:

```
ExchangeRateServiceMapper -> ExchangeRateMapper
CurrencyServiceMapper -> CurrencyMapper
CurrencyDAOMapper -> CurrencyRowMapper
ExchangeRateDAOMapper -> ExchangeRateRowMapper
```

## Общее

- Неплохо было бы оформлять коммиты в соответствии
  с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- Местами не хватает форматирования, настоятельно рекомендую использовать *Reformat Code (Ctrl + alt + L)*
- Лучше не оставлять неиспользуемые импорты.
- В таблице валют нет уникального индекса для поля `Code` как того требует ТЗ.
- Не используемый нигде `PATH_CONFIG_PROPERTIES("src/main/resources/config.properties")` в `Config`.
- В pom.xml находится JUnit, который не используется.
- `.idea` папку лучше добавлять в `.gitignore`.
- С этого проекта можно попробовать _Lombok_, который дает возможность уменьшить _boilerplate_ код засчет
  аннотаций.
- Не стоит игнорировать замечания от *IDEA* - почти всегда это по делу, если ты специально не хочешь сделать по другому.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.

## Итог

- Работающий проект, после рефакторинга недостающих мест можно смело идти дальше. Удачи!