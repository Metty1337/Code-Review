# Review на реализацию от [@sutyaginev](https://github.com/sutyaginev) проекта [Виселица](https://zhukovsd.github.io/java-backend-learning-course/projects/hangman/)

[Сама реализация](https://github.com/sutyaginev/currency-exchange)

## Реализация REST API

- Хорошо:
    - Ответы эндпоинтов соответствуют ТЗ и подходят для фронта.
    - Есть валидация кода валюты.
- Замечания:
    - Нет ограничения на длину имени из-за чего легко сломать верстку.
    - Нет ограничения на длину символа из-за чего легко сломать верстку.
    - На некоторые реквесты, где по ТЗ ожидается ответ код 400 приходит код 500.
    - Прикреплю упавшие кейсы от бота (@currency_exchange_api_bot) для рефакторинга:
        1) TC-025a sign длиннее 3 символов → 400 + {message}
           Причина: Ожидали HTTP 400, получили 201
           Request: POST /currencies | Form: name=LongSign&code=GTS&sign=ABCD
           Response: HTTP 201 | Content-Type: application/json;charset=UTF-8 | Body: {"id":19,"code":"GTS","name":"
           LongSign","sign":"ABCD"}

        2) TC-060 нет rate → 400 + {message}
           Причина: Ожидали HTTP 400, получили 500
           Request: POST /exchangeRates | Form: baseCurrencyCode=KTN&targetCurrencyCode=MBR
           Response: HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"message":"Внутренняя ошибка
           сервера"}

        3) TC-080 ответ: baseCurrency.code == A1
           Причина: Поле baseCurrency.code не совпадает с ожидаемым
           Request: GET /exchange?from=DWP&to=FBN&amount=10
           Response: HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {"base":{"id":22,"code":"DWP","
           name":"CurrencyADWP","sign":"¤"},"target":{"id":23,"code":"FBN","name":"CurrencyBFBN","sign":"¤"},"rate":
           1.25,"amount":10.00,"convertedAmount":12.50}

        4) TC-081 ответ: targetCurrency.code == B1
           Причина: Поле targetCurrency.code не совпадает с ожидаемым
           Request: GET /exchange?from=DWP&to=FBN&amount=10
           Response: HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {"base":{"id":22,"code":"DWP","
           name":"CurrencyADWP","sign":"¤"},"target":{"id":23,"code":"FBN","name":"CurrencyBFBN","sign":"¤"},"rate":
           1.25,"amount":10.00,"convertedAmount":12.50}

        5) TC-091 нет from → 400 + {message}
           Причина: Ожидали HTTP 400, получили 500
           Request: GET /exchange?to=USD&amount=10
           Response: HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"message":"Внутренняя ошибка
           сервера"}

        6) TC-092 нет to → 400 + {message}
           Причина: Ожидали HTTP 400, получили 500
           Request: GET /exchange?from=USD&amount=10
           Response: HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"message":"Внутренняя ошибка
           сервера"}

        7) TC-093 нет amount → 400 + {message}
           Причина: Ожидали HTTP 400, получили 500
           Request: GET /exchange?from=USD&to=EUR
           Response: HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"message":"Внутренняя ошибка
           сервера"}

## По коду

### package entity

- Для `id` можно вместо `Integer` использовать `Long`.
- Подходящий конструктор для случаев, когда нет `id` - хорошо.
- `BigDecimal` для `rate` - хорошо.
- Оба класса могут быть без последствий конвертированы в _record_.

### package dao

- синглтон анти-паттерн: нарушает SRP, скрытые зависимости, сложность в тестировании. Стоит явно выражать зависимости
  через конструктор и давать другим их подменять.

```
// лучше переделать
public class CurrencyDao {

    private static final CurrencyDao INSTANCE = new CurrencyDao();
    private static final DataSource DATA_SOURCE = DataSourceProvider.getDataSource();
    
        private CurrencyDao() {
    }
```

- `Optional` для поиска - хорошо.
- Дублирующийся код можно вынести в вспомогательный метод:

```
public Optional<Currency> findByCode(String code) {
            if (resultSet.next()) {
                return Optional.of(new Currency(
                        resultSet.getInt("id"),
                        resultSet.getString("code"),
                        resultSet.getString("name"),
                        resultSet.getString("sign")
                ));
            }
            ...
            
            public List<Currency> findAll() {
                        while (resultSet.next()) {
                currencies.add(new Currency(
                        resultSet.getInt("id"),
                        resultSet.getString("code"),
                        resultSet.getString("name"),
                        resultSet.getString("sign")
                ));
            }
            
// лучше
    public List<Currency> findAll() {
                while (resultSet.next()) {
                currencies.add(mapRowToCurrency(resultSet));
            }
            
    public Optional<Currency> findByCode(String code) {
                if (resultSet.next()) {
                return Optional.of(mapRowToCurrency(resultSet));
            }
            
    private static Currency mapRowToCurrency(ResultSet resultSet) throws SQLException {
        return new Currency(
                resultSet.getInt("id"),
                resultSet.getString("code"),
                resultSet.getString("name"),
                resultSet.getString("sign")
        );
    }
```

- синглтон анти-паттерн: нарушает SRP, скрытые зависимости, сложность в тестировании. Стоит явно выражать зависимости
  через конструктор и давать другим их подменять.

```
// лучше переделать
public class ExchangeRateDao {

    private static final ExchangeRateDao INSTANCE = new ExchangeRateDao();
    private static final DataSource DATA_SOURCE = DataSourceProvider.getDataSource();
   
        private ExchangeRateDao() {
    }
```

- Качественные _SQL_ запросы - хорошо.

- Из-за отсутствия интерфейсов нарушает _DIP_.

### package service

CurrencyService:

- синглтон анти-паттерн: нарушает SRP, скрытые зависимости, сложность в тестировании. Стоит явно выражать зависимости
  через конструктор и давать другим их подменять.

```
// лучше переделать
public class CurrencyService {

    private static final CurrencyService INSTANCE = new CurrencyService();
    private static final CurrencyDao CURRENCY_DAO = CurrencyDao.getInstance();
    private static final CurrencyMapper CURRENCY_MAPPER = CurrencyMapper.getInstance();

    private CurrencyService() {
    }
```

- Можно упросить названия методов: `getAllCurrencies` -> `getAll`, `getCurrencyByCode` -> `getByCode`.
- Также для сервиса название `create` подходит больше, чем `save`.
- Скрывая детали реализации, делай код более удобочитабельным, пользуясь вспомогательными методами:

```
    public CurrencyResponseDto getCurrencyByCode(String code) {
        String normalizedCode = code.trim().toUpperCase();
        Currency currency = CURRENCY_DAO.findByCode(normalizedCode).
                orElseThrow(() -> new CurrencyNotFoundException(normalizedCode));

        return CURRENCY_MAPPER.toResponseDto(currency);
    }
        public CurrencyResponseDto save(CurrencyRequestDto requestDto) {
        String code = requestDto.code();
        String name = requestDto.name();
        String sign = requestDto.sign();

        CurrencyRequestDto normalized = new CurrencyRequestDto(
                code.trim().toUpperCase(),
                name.trim(),
                sign.trim()
        );
        Currency currency = CURRENCY_MAPPER.toCurrency(normalized);
        Currency saved = CURRENCY_DAO.save(currency);

        return CURRENCY_MAPPER.toResponseDto(saved);
    }
    // лучше
    
    public CurrencyResponseDto getCurrencyByCode(String code) {
        String normalizedCode = normalizeCode(code);
        Currency currency = CURRENCY_DAO.findByCode(normalizedCode).
                orElseThrow(() -> new CurrencyNotFoundException(normalizedCode));

        return CURRENCY_MAPPER.toResponseDto(currency);
    }

    public CurrencyResponseDto save(CurrencyRequestDto requestDto) {
        String code = requestDto.code();
        String name = requestDto.name();
        String sign = requestDto.sign();

        CurrencyRequestDto normalized = normalizeCurrencyRequest(code, name, sign);
        Currency currency = CURRENCY_MAPPER.toCurrency(normalized);
        Currency saved = CURRENCY_DAO.save(currency);

        return CURRENCY_MAPPER.toResponseDto(saved);
    }

    private static String normalizeCode(String code) {
        return code.trim().toUpperCase();
    }

    private static CurrencyRequestDto normalizeCurrencyRequest(String code, String name, String sign) {
        return new CurrencyRequestDto(
                code.trim().toUpperCase(),
                name.trim(),
                sign.trim()
        );
    }
```

ExchangeRateService:

- синглтон анти-паттерн: нарушает SRP, скрытые зависимости, сложность в тестировании. Стоит явно выражать зависимости
  через конструктор и давать другим их подменять.

```
// лучше переделать
public class ExchangeRateService {

    private static final ExchangeRateService INSTANCE = new ExchangeRateService();
    private static final CurrencyDao CURRENCY_DAO = CurrencyDao.getInstance();
    private static final ExchangeRateDao EXCHANGE_RATE_DAO = ExchangeRateDao.getInstance();
    private static final ExchangeRateMapper EXCHANGE_RATE_MAPPER = ExchangeRateMapper.getInstance();

    private ExchangeRateService() {
    }
```

- Нарушение инкапсуляции - стоит использовать сервис, который уже инкапсулирует работу с ДАО:

```
public class ExchangeRateService {
    private static final CurrencyDao CURRENCY_DAO = CurrencyDao.getInstance();

// лучше
public class ExchangeRateService {
    private static final CurrencyService CURRENCY_SERVICE = CurrencyService.getInstance();
```

- Интересная реализация с целью не дублировать курсы, но есть места, которые можно облегчить:

```
    public ExchangeRateResponseDto findByCodes(String baseCode, String targetCode) {
        CurrencyPair pair = getPair(baseCode, targetCode);
        CanonicalContext context = getCanonical(pair.base, pair.target);
        BigDecimal rate = context.inverted ? invert(context.stored().getRate()) : context.stored().getRate();
        ExchangeRate response = new ExchangeRate(
                context.stored.getId(),
                pair.base,
                pair.target,
                rate);

        return EXCHANGE_RATE_MAPPER.toResponseDto(response);
    }
    
    public ExchangeRateResponseDto save(ExchangeRateRequestDto requestDto) {
        CurrencyPair pair = getPair(requestDto.baseCode(), requestDto.targetCode());
        BigDecimal normalRate = requestDto.rate().setScale(6, RoundingMode.HALF_UP);
        ExchangeRate saved = canonicalize(pair.base, pair.target, normalRate);

        return EXCHANGE_RATE_MAPPER.toResponseDto(EXCHANGE_RATE_DAO.save(saved));
    }
// лучше
    public ExchangeRateResponseDto findByCodes(String baseCode, String targetCode) {
        CurrencyPair pair = getPair(baseCode, targetCode);
        CanonicalContext context = getCanonical(pair.base, pair.target);
        BigDecimal rate = resolveRate(context, context.stored.getRate());
        ExchangeRate response = toResponseView(context, pair, rate);

        return EXCHANGE_RATE_MAPPER.toResponseDto(response);
    }

    public ExchangeRateResponseDto save(ExchangeRateRequestDto requestDto) {
        CurrencyPair pair = getPair(requestDto.baseCode(), requestDto.targetCode());
        BigDecimal normalRate = normalizeRate(requestDto);
        ExchangeRate saved = canonicalize(pair.base, pair.target, normalRate);

        return EXCHANGE_RATE_MAPPER.toResponseDto(EXCHANGE_RATE_DAO.save(saved));
    }

    public ExchangeRateResponseDto update(ExchangeRateRequestDto requestDto) {
        CurrencyPair pair = getPair(requestDto.baseCode(), requestDto.targetCode());
        BigDecimal normalRate = normalizeRate(requestDto);
        CanonicalContext context = getCanonical(pair.base, pair.target);
        BigDecimal rate = resolveRate(context, normalRate);

        ExchangeRate updated = toCanonicalView(context, rate);

        return EXCHANGE_RATE_MAPPER.toResponseDto(EXCHANGE_RATE_DAO.update(updated));
    }

    private static ExchangeRate toCanonicalView(CanonicalContext context, BigDecimal rate) {
        return new ExchangeRate(
                context.stored.getId(),
                context.stored.getBaseCurrency(),
                context.stored.getTargetCurrency(),
                rate
        );
    }

    private static ExchangeRate toResponseView(CanonicalContext context, CurrencyPair pair, BigDecimal rate) {
        return new ExchangeRate(
                context.stored.getId(),
                pair.base,
                pair.target,
                rate);
    }

    private BigDecimal resolveRate(CanonicalContext context, BigDecimal rate) {
        return context.inverted ? invert(rate) : rate;
    }

    private static BigDecimal normalizeRate(ExchangeRateRequestDto requestDto) {
        return requestDto.rate().setScale(6, RoundingMode.HALF_UP);
    }
```

Все-таки сложную логику стоит как можно аккуратнее преподносить для обозревателя.

- Магическую цифру "6" стоит заменить константой с содержательным названием.

ExchangeService:

- синглтон анти-паттерн: нарушает SRP, скрытые зависимости, сложность в тестировании. Стоит явно выражать зависимости
  через конструктор и давать другим их подменять.

```
// лучше переделать
public class ExchangeService {

    private static final ExchangeService INSTANCE = new ExchangeService();
    private static final ExchangeRateService EXCHANGE_RATE_SERVICE = ExchangeRateService.getInstance();
    private static final CurrencyService CURRENCY_SERVICE = CurrencyService.getInstance();
    private static final String USD = "USD";

    private ExchangeService() {
    }
``` 

- Метод `BigDecimal resolveRate(String baseCode, String targetCode)` содержит анти-паттерн "exceptions as control flow",
  стоит переделать через вспомогательные методы, которые будут возвращать `Optional`:

```
    private BigDecimal resolveRate(String baseCode, String targetCode) {
        try {
            return EXCHANGE_RATE_SERVICE.findByCodes(baseCode, targetCode).rate();
        } catch (ExchangeRateNotFoundException ignored) {
        }

        try {
            BigDecimal usdToBase = EXCHANGE_RATE_SERVICE.findByCodes(USD, baseCode).rate();
            BigDecimal usdToTarget = EXCHANGE_RATE_SERVICE.findByCodes(USD, targetCode).rate();

            return usdToTarget.divide(usdToBase, 6, RoundingMode.HALF_UP);
        } catch (ExchangeRateNotFoundException ignored) {
        }

        throw new ExchangeRateNotFoundException(baseCode, targetCode);
    }

// лучше

private BigDecimal resolveRate(String baseCode, String targetCode) {
    return findDirectRate(baseCode, targetCode)
            .orElseGet(() -> findCrossRate(baseCode, targetCode)
                    .orElseThrow(() -> new ExchangeRateNotFoundException(baseCode, targetCode)));
}

private Optional<BigDecimal> findDirectRate(String baseCode, String targetCode){
    ...
}

private Optional<BigDecimal> findCrossRate(String baseCode, String targetCode){
    ...
}
```

- Скрывай детали реализации, делай код более удобочитабельным:

```
        BigDecimal convertedAmount = amount.multiply(rate).setScale(2, RoundingMode.HALF_UP);

// лучше
        BigDecimal convertedAmount = getConvertedAmount(amount, rate);
```

- Странное округление валюты в методе `exchange`. Как я понял, по _ТЗ_ надо хранить и проводить все операции с
  десятичным числом с шестью знаками после запятой.

### package controller

- Вынесена обработка ошибок в отдельный фильтр - хорошо.
- В сервлетах используется _DTO_, а не напрямую _Entity_ - хорошо.
- Вынесена отправка ответа в утилитарный класс - хорошо.

### package dto

- Хорошее разделение на response/request, стоит также разнести классы по разным пакетам `dto.request` и `dto.response`.
- _DTO_ классы если они разделены на response/request, то называются без суффикса `Dto`. То есть `CurrencyRequestDto` ->
  `CurrencyRequest`.

### package exception

- Все сообщения об исключения должны быть написаны на английском:

```
public class CurrencyNotFoundException extends RuntimeException {
    public CurrencyNotFoundException(String code) {
        super("Валюта с кодом %s не найдена".formatted(code));
    }
}
// лучше
public class CurrencyNotFoundException extends RuntimeException {
    public CurrencyNotFoundException(String code) {
        super("Currency with code %s not found".formatted(code));
    }
}
```

### package util

- Все классы имеют закрытый конструктор и `final` модификатор в сигнатуре - хорошо.

## Общее

- В проекте нестандартное расположение фронта. Стандарт:

```
src/main/webapp/
├── css/
├── js/
├── index.html
└── WEB-INF/
    └── web.xml
```

- Магические числа/строки стоит заменять константами.
- Как вариант улучшения мапперов - можно ознакомиться с _MapStruct_.
- Опциональное улучшение - большая часть **boilerplate*-кода* может быть заменена *Lombok*'ом.
- Используй чаще форматирование (ctrl+alt+l в idea) - есть неаккуратные места.
- Не игнорируй замечания от idea - чаще всего они полезны.

### Итог

- Достойный проект, после рефакторинга некоторые моментов можно смело идти дальше. Удачи!