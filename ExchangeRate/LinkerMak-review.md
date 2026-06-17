# Review на реализацию в рамках учебной подписки от [@LinkerMak](https://github.com/LinkerMak) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/LinkerMak/CurrencyEx)

## Реализация

- Я так понимаю задеплоил ты необычным способом, в этом нет ничего плохого, но рекомендую в будущем попробовать классику, если еще не пробовал - через аренду VPS.
- Проверил твой REST API через бота сообщества - все супер!


## По коду

### package model

- В данном проекте model по смыслу ближе к Entity, то есть это представление объекта в таблице. И по этому смыслу подходят классы `Currency` и `ExchangeRate`, ведь у них свои таблицы, а класс `Exchange` выпадает. Он выполняет скорее роль некого DTO, но DTO в проекте уже есть, поэтому я бы назвал этот класс излишним, вскоре мы еще вернемся к его роли в сервисном/контроллер слое.
```java
public class Exchange {
    private Currency baseCurrency;
    private Currency targetCurrency;
    private BigDecimal rate;
    private BigDecimal amount;
    private BigDecimal convertedAmount;
}
```
- В остальном к пакету вопросов нет.

### package repository

- Выделены интерфейсы для dao - супер
- ResultSet реализует интерфейс AutoClosable, что значит, что лучше его использовать через try-with-resources:
```java
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return Optional.of(mapRow(rs));
            }

// лучше
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapRow(rs));
                }
            }            
            
```
- В блоке catch, если метод `throwIfUniquesConstraintViolated` выбросит исключение, то и `throw new DatabaseOperationException` тоже выполнится. На сколько я понимаю - твое первое исключение будет помечено как suppressed, но на первом месте будет стоять другое исключение, и это явно не то что мы хотим донести до клиента. Проще просто обернуть нашу логику в if-else, избегая затирания исключения:
```java
        } catch (SQLException e) {
            SqliteExceptionTranslator.throwIfUniquesConstraintViolated(e, "Валюта с таким кодом уже существует");
            throw new DatabaseOperationException("Не удалось сохранить валюту", e);
        }
        
// лучше
        } catch (SQLException e) {
            if (SqliteExceptionTranslator.isUniquesConstraintViolated(e)) {
                throw new EntityAlreadyExistsException((e, "Валюта с таким кодом уже существует"));
            } else {
                throw new DatabaseOperationException("Не удалось сохранить валюту", e);
            }
        }
```
- Обычно методы, которые сохраняют в БД сущность, называют `save` вместо `insert`.
- Удобнее вместо бесконечных кавычек использовать блоки текста:
```java
        String sql = "SELECT" +
                " er.ID AS id, " +
                " bc.ID AS base_id," +
                " bc.FullName AS base_name," +
                " bc.Sign AS base_sign," +
                " bc.Code AS base_code," +
                " tc.ID AS target_id," +
                " tc.FullName AS target_name," +
                " tc.Sign AS target_sign," +
                " tc.Code AS target_code," +
                " er.Rate AS rate" +
                " from ExchangeRates er" +
                " join Currencies tc ON er.TargetCurrencyId = tc.id" +
                " join Currencies bc ON er.BaseCurrencyId = bc.id" +
                " WHERE (bc.id = (SELECT c1.id FROM Currencies c1 WHERE c1.code = ?) AND" +
                " tc.id = (SELECT c2.id FROM Currencies c2 WHERE c2.code = ?))";
                
// лучше
String sql = """
        SELECT
            er.ID AS id,
            bc.ID AS base_id,
            bc.FullName AS base_name,
            bc.Sign AS base_sign,
            bc.Code AS base_code,
            tc.ID AS target_id,
            tc.FullName AS target_name,
            tc.Sign AS target_sign,
            tc.Code AS target_code,
            er.Rate AS rate
        FROM ExchangeRates er
        JOIN Currencies tc ON er.TargetCurrencyId = tc.id
        JOIN Currencies bc ON er.BaseCurrencyId = bc.id
        WHERE bc.id = (SELECT c1.id FROM Currencies c1 WHERE c1.code = ?)
          AND tc.id = (SELECT c2.id FROM Currencies c2 WHERE c2.code = ?)
        """;
```

### package service

- Хорошей практикой в сервисе является возвращать сами DTO, тем самым скрывая настоящее устройство модели от контроллера, то есть сохраняя границы слоев.
```java
    public Currency findByCode(String code) {
        return currencyDao.findByCode(code)
                .orElseThrow(() -> new ResourceNotFoundException("Валюта " + code + " не найдена"));
    }
    
// лучше
    public CurrencyResponse findByCode(String code) {
        return currencyDao.findByCode(code)
                .map(currencyMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Валюта " + code + " не найдена"));
    }
```

- Также лишние методы лучше убирать из готового кода.
- Можно чуть упростить названия: `getAllCurrencies` -> `getAll`, `createCurrency` -> `create`.
- Сервис по расчету курса выглядит довольно солидно - четко и кратко.


### package servlet

- Молодец, что создаешь зависимости не сама, а получаешь из DI.
- Выводы в консоль через sout лучше не оставлять в готовом коде, можно заменить на логирование:
```java
        String code = CurrencyCodeNormalizer.normalize(pathInfo.substring(1));
        System.out.println(code);
        DtoValidator.validateCurrencyCode(code);
        
// лучше 
        String code = CurrencyCodeNormalizer.normalize(pathInfo.substring(1));
        log.info(code);
        DtoValidator.validateCurrencyCode(code);
```

- Как и писал ранее, лучше в контроллере не вызывать маппер, все это должно быть либо в сервисе или в фасаде.
- Также имхо ни разу не видел, чтобы MapStruct мапперы вызывались через синглтон, лучше получать его через тот же DI.
- Сервлеты достаточно чистые с точки зрения повторяемого кода, все валидации и прочее вынесены в утилитарные классы - молодец.
- Все утилитарные классы с верным модификатор и конструктором - супер.

### package mapper

- В некоторых мапперах у тебя прописана логика округления и тут на самом деле спорный момент насколько там вообще должна быть эта логика. С одной стороны принято, что маппер не должен содержать никакой логики, с другой стороны это достаточно удобно, я предпочитаю тонкие мапперы, а ты смотри сам.

### package dto

- У дто никогда не должно быть сеттеров, поскольку это должен быть неизменяемый объект. Советую использовать record для них, чтобы понять смысл дто.

## Общее
- Пользуйся [commit convention](https://habr.com/ru/articles/867012/) для именования коммитов.
- В проекте есть зависимость JUnit, но тестов нет - лишняя зависимость.
- Пакеты в Java называют в нижнем регистре (lowercase), используя обратную нотацию доменного имени. То есть вместо `webbackend` - лучше `web.backend` и тд.


## Итог
- Достаточно солидный третий проект, учти пару нюансов и можешь смело идти дальше, удачи! 