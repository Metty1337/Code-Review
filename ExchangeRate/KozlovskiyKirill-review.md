# Review на реализацию от [@KozlovskiyKirill](https://github.com/KozlovskiyKirill) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/KozlovskiyKirill/CurrencyExchange)


## Реализация
- Не успел ознакомиться с деплоем, но в сообществе есть бот с автотестами, обязательно воспользуйся.
- Советую попробовать подключить готовый фронт, это несложно, но полезно.

## По коду

### package model
- Нарушение java конвенции по именованию полей:
```java
    private final int _id;
    private final String _code;
    private final String _fullName;
    private final String _sign;
```
- То же касается методов:
```java
    public int get_id(){
        return _id;
    }

    public String get_code(){
        return _code;
    }

    public String get_fullName(){
        return _fullName;
    }
    public String get_sign(){
        return _sign;
    }
```
- Концептуально для `id` лучше подходит `Long` вместо `int` или `Integer`.
- Название метода обманывает:
```java
public Currency getBaseCurrencyID() {
        return _baseCurrency;
    }
```

### package DAO
- Нарушение конвенции именования пакетов - все должно быть с маленькой буквы, а также в классах аббревиатура не пишется заглавными буквами - `CurrencyDAO` -> `CurrencyDao`. 
- Класс Connect и AppInitListener хотелось бы видеть не в этом пакете, а в отдельном инфраструктурном.
- В классе AppInitListener не помешало бы реализовать метод для закрытия ресурсов.
- ResultSet тоже надо обрабатывать в try-с-ресурсами.
- Не стоит пробрасывать `SQLException` в верхние слои, они не должны знать про JDBC. Лучше ловить исключение в DAO и выбрасывать понятное для бизнес-логики исключение, например `DatabaseException`.
- Банальная ошибка - опечатка:
```java
               int _id = rs.getInt(1);
               String _name = rs.getString(2);
               String _code = rs.getString(3);
               String _sign = rs.getString(4);
               Currency cur = new Currency(_id, _name,code,_sign);
               return Optional.of(cur);
```
- Для блоков текста в java есть тройные кавычки:
```java
            ResultSet rs = statement.executeQuery("select\n" +

                    "er.id,\n" +

                    "bc.ID,\n" +
                    "bc.FullName,\n" +
                    "bc.code,\n" +
                    "bc.Sign,\n" +

                    "tc.ID,\n" +
                    "tc.FullName,\n" +
                    "tc.code,\n" +
                    "tc.Sign,\n" +

                    "er.rate\n" +
                    "\n" +
                    "from exchange_rates er\n" +
                    "join currencies bc on er.BaseCurrencyID=bc.ID\n" +
                    "join currencies tc on er.TargetCurrencyID=tc.ID;");
```
- Как будто проще возвращать Optional вместо int, более простой API `public int findExchangeRatesPair`.

### package service
- Инжект через конструктор предпочтительнее, чем создание прямо в поле:
```java
public class CurrencyService {
    private final CurrencyDAO _currencyDao = new CurrencyDAO();
```
- Нельзя пропускать SQLException в слой контроллера:
```java
    public List<Currency> getAllCurrencies() throws SQLException {
        return _currencyDao.getAllCurrencies();
    }
```
- В сервисе всегда - принимаем DTO, отдаем DTO, здесь же такого не происходит, а стоит к этому стремиться.
- В ООП языке мы никогда не возвращаем массив, когда нам нужна какая-то пара объектов, надеюсь намек понят:
```java
private Currency[] getCurrenciesPair
```
- В этом методе я бы подумал над блокировкой, но это уже более advanced тема, можно не заморачиваться, если нет желания:
```java
    public ExchangeRate addNewExchangeRate(String baseCurrencyCode,
                                           String targetCurrencyCode, BigDecimal rate) throws SQLException {
        Currency[] currencies = getCurrenciesPair(baseCurrencyCode, targetCurrencyCode);
        Currency baseCurrency = currencies[0];
        Currency targetCurrency = currencies[1];
        int baseCurrencyID = baseCurrency.get_id();
        int targetCurrencyID = targetCurrency.get_id();
        ensurePairDoesNotExist(baseCurrencyID, targetCurrencyID);

        int newRateID = _dao.addNewExchangeRate(baseCurrencyID, targetCurrencyID, rate);
        return new ExchangeRate(newRateID, baseCurrency, targetCurrency, rate);
    }
```
### package servlet
- Можно вынести "ловлю" исключений в отдельный `ExceptionHandlingFilter implements Filter`, тогда сервлеты станут тоньше и ближе к соблюдению SRP.
- Зависимости не стоит создавать через `new` прямо в сервлетах. Рекомендую реализовать паттерн Composition Root, например через `AppInitializer implements ServletContextListener`, который создаст все зависимости и положит их в `ServletContext`.
- Валидацию также можно вынести в отдельные классы/методы.
- Мапить сущность в dto стоит в сервисе, а не в контроллере.

### package dto
- Раз уж используем java 25, то можно использовать record.
- Как вариант улучшения мапперов - можно ознакомиться с MapStruct.
- ExchangeCurrency в целом класс мне кажется излишним, можно оставить один лишь дто.


## Общее
- Папку `.idea` и `target` стоит добавлять в `.gitignore`, поскольку это локальная конфигурация IDE.
- БД не принято добавлять в репу, только миграции.
- Неплохо было бы оформлять коммиты в соответствии с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- В большинстве случаев комментарии - признак трудно читаемого кода, который необходимо переписать. Стоит сделать код самодокументируемым за счет декомпозиции и хороших имен методов. Подробнее: Мартин, "Чистый Код", гл.4, Комментарии.
- Большая часть boilerplate-кода может быть заменена Lombok - рекомендую ознакомиться.
- Местами не хватает форматирования, настоятельно рекомендую использовать Reformat Code (Ctrl+Alt+L) в IntelliJ IDEA.
- Весь проект в целом не компилируемый, пушить коммиты стоит уже рабочей версии приложения.
- По всему проекту нарушения конвенции наименования классов, полей, методов. Не знаю где поля называют с _, но в java точно так не делают.

## Итог
- Проект выполнен на начальном уровне, есть некие грубости в реализации, комментарии специально оставлял без сильно разжеванных объяснений для как раз таки возможности самому разобраться. Удачи!