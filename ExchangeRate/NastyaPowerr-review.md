# Review на реализацию от [@NastyaPowerr](https://github.com/NastyaPowerr) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/NastyaPowerr/CurrencyExchanger)

## Реализация REST API

- Судя по отсчету от @currency_exchange_api_bot особых проблем не возникло, молодец.
- Единственное - у тебя фронтенд обращается к беку через двойной слеш:

```
"POST /CurrencyExchanger//currencies HTTP/1.1"
```

## По коду

### package model

- Четкое разделение _DTO_ на _request_ и _response_ - большой плюс.
- С точки зрения архитектуры: _entity_ - это persistence model, а _dto_ - transport model, потому они не принадлежат к
  одному уровню. На твоем месте я бы вынес _DTO_ в отдельный пакет, и после этого можно было бы убрать абстрактный
  _model_ и получили бы:

```
// сейчас
model
 ├─ dto
 │   ├─ request
 │   └─ response
 └─ entity

// лучше
entity
dto
```

- К классам _entity_ обычно не добавляют суффикс `Entity`, а оставляют просто _Currency_.
- В пакете _Entity_ только _CurrencyEntity_ и
  _ExchangeRateEntity_ и являются _Entity_, а остальные же _CurrencyCodePair_ и _ExchangeRateUpdateEntity_ больше похожи
  на _Value Object_
  или _DTO_, нежели _Entity_.
- _Record_ для _DTO_ - идеальный вариант, но для _Entity_, на мой вкус, не очень удобно.
- Опциональное улучшение: на твоем месте я бы сделал _ExchangeRate_ более похожим на schema, то есть:

```java
// сейчас
public record ExchangeRateEntity(
        Long id,
        CurrencyEntity baseCurrencyEntity,
        CurrencyEntity targetCurrencyEntity,
        BigDecimal rate
) {
}

// лучше
public record ExchangeRate (
    Long id,
    Long baseCurrencyId,
    Long targetCurrencyId,
    BigDecimal rate
){
}
```

Твой вариант больше подходит для _ORM_, который встретится в следующих проектах, но для простого JDBC это избыточная
сложность.

### package dao

- Выделены интерфейсы для _DAO_ - большой плюс.
- Текст запроса удобнее читать, когда он логично разбит на строки, для этого можно использовать текстовые блоки:

```java
// сейчас 
    private static final String GET_BY_CODE_QUERY = "SELECT * FROM currencies WHERE code = ?";
// лучше 
    private static final String GET_BY_CODE_QUERY = """
        SELECT *
        FROM currencies
        WHERE code = ?
        """;
```

- Как вариант улучшения _SQL_ запросов: вместо `SELECT *` прописывай конкретно, что тебе нужно. Таким образом при
  изменении _schema_ твое приложение будет работать корректно.

```java
// сейчас 
    private static final String GET_BY_CODE_QUERY = "SELECT * FROM currencies WHERE code = ?";
// лучше
    private static final String GET_BY_CODE_QUERY = """
        SELECT id, code, full_name, sign
        FROM currencies
        WHERE code = ?
        """;
```

- _JdbcExchangeRateDao_:
    - Очень странный метод `ExchangeRateEntity saveFromCodes(ExchangeRateUpdateEntity exchangeRate)`:
        - Сначала ты сохраняешь _ExchangeRateEntity_ по их кодам, а потом зачем-то делаешь еще один запрос на поиск
          ExchangeRateEntity - нарушение SRP. Лучше было бы получить _ExchangeRateEntity_ через _getGeneratedKeys_ или
          же вызывать метод _findByCodes_ уже в сервисе.
        - После "ловли" _SQLException_ ты делаешь еще 2 _SQL_ запроса, пытаясь расшифровать ошибку - антипаттерн.
        - В аргументах _ExchangeRateUpdateEntity_ - класс, который не является _Entity_, почему бы тогда не использовать
          сам _ExchangeRateEntity_, у которого null вместо _id_.
- Правильное "превращение" `SQLException` в понятные бизнес-логике исключения - хорошо.

### package service

- Методы _save_ я бы переименовал в _create_, _save_ больше подходит для _dao_, а не бизнес-логике.
- `ExchangeRateResponseDto save(ExchangeRateRequestDto exchangeRate)`:
    - На мой взгляд, было бы правильнее сначала найти валюту в сервисе, а потом отправить _Entity_ в _DAO_, а не делать
      эти операции в
      _DAO_ в обратном порядке:
  ```java
  // сейчас
      public ExchangeRateResponseDto save(ExchangeRateRequestDto exchangeRate) {
        ExchangeRateUpdateEntity entity = new ExchangeRateUpdateEntity(
                exchangeRate.baseCurrencyCode(),
                exchangeRate.targetCurrencyCode(),
                exchangeRate.rate()
        );
        ExchangeRateEntity savedEntity = exchangeRateDao.saveFromCodes(entity);
        return ExchangeRateMapper.INSTANCE.toResponseDto(savedEntity);
    }
  
  // лучше
      public ExchangeRateResponseDto save(ExchangeRateRequestDto exchangeRate) {
        CurrencyEntity baseCurrency = currencyService.findByCode(exchangeRate.baseCurrencyCode())
                .orElseThrow(...);
        CurrencyEntity targetCurrency = currencyService.findByCode(exchangeRate.targetCurrencyCode())
                .orElseThrow(...);
        ExchangeRateEntity exchangeRateEntity = new ExchangeRateEntity(null, baseCurrency, targetCurrency, exchangeRate.rate());
        ExchangeRateEntity savedEntity = exchangeRateDao.save(exchangeRateEntity);
        return ExchangeRateMapper.INSTANCE.toResponseDto(savedEntity);
    }
  ``` 

- Старайся не кодировать в имени то, что уже видно из типа:

```java
// сейчас
Optional<ExchangeRateEntity> rateEntityOpt = exchangeRateDao.findByCodes(codePair);

// лучше
Optional<ExchangeRateEntity> exchangeRate = exchangeRateDao.findByCodes(codePair);
```

- Так же как и `ExchangeRateResponseDto save(ExchangeRateRequestDto exchangeRate)` можно сначала попробовать найти
  валюты и курс, а потом отправить запрос в _DAO_, оставляя бизнес-логику сервису.

```java
    // лучше
    public ExchangeRateResponseDto update(ExchangeRateRequestDto exchangeRate) {
        CurrencyCodePair codePair = new CurrencyCodePair(exchangeRate.baseCurrencyCode(), exchangeRate.targetCurrencyCode());
        ExchangeRateUpdateEntity entity = new ExchangeRateUpdateEntity(
                exchangeRate.baseCurrencyCode(),
                exchangeRate.targetCurrencyCode(),
                exchangeRate.rate()
        );
        exchangeRateDao.update(entity);
        return getByCode(codePair);
    }
    
  // лучше
      public ExchangeRateResponseDto update(ExchangeRateRequestDto exchangeRate) {
        CurrencyEntity baseCurrency = currencyService.findByCode(exchangeRate.baseCurrencyCode())
                .orElseThrow(...);
        CurrencyEntity targetCurrency = currencyService.findByCode(exchangeRate.targetCurrencyCode())
                .orElseThrow(...);
        ExchangeRateEntity exchangeRateEntity = exchangeRateService.find(baseCurrency, targetCurrency)
                .orElseThrow(...);
        exchangeRateDao.update(exchangeRateEntity);
        return ...
    }
```

- _ExchangeService_ неплохой класс, хоть и можно дополнительно декомпозировать с помощью вспомогательных методов.

### package servlet

- Можно вынести "ловлю" исключения в отдельный _ExceptionsFilter_, тогда сервлеты станут тоньше и ближе к соблюдению
  _SRP_.
- Получаешь зависимости через _ServletContext_ - большой плюс.
- В целом неплохой пакет сервлетов.

### package exception

- Я бы рассмотрел возможность добавления класса констант для сообщений для исключений, чтобы не повторятся каждый раз и
  избавится от магических строк.

### package mapper

- Использование _mapstruct_ для мапперов - хорошо.

### package util

- Все утилитарные классы с модификатором _final_ и закрытым конструктором - плюс.
- _ServletResponseUtil_ странная инициализация с _init_, потому что это не потокобезопасно и нет гарантии инициализации.
  Это антипаттерин Hidden Global Dependency. Самым простым было бы:

```java
// сейчас
public final class ServletResponseUtil {
    private static ObjectMapper objectMapper;

    private ServletResponseUtil() {
    }

    public static void init(ObjectMapper mapper) {
        if (objectMapper == null) {
            objectMapper = mapper;
        } else {
            throw new IllegalStateException("ObjectMapper already initialized.");
        }
    }
    
// лучше    
public final class ServletResponseUtil {
    private static final ObjectMapper objectMapper =  new ObjectMapper();
    ...
    }
```

## Общее

- Правильное оформление коммитов
- Повсеместно используются константы вместо магических строк/чисел.
- Как вариант улучшения - можно ознакомится с _Lombok_, точно пригодится в будущем.

### Итог

- Считаю очень достойный проект, после рефакторинга некоторых моментов можно смело переходить к следующему проекту.
  Удачи!
