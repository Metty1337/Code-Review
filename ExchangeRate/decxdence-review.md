# Review на реализацию от [@decxdence](https://github.com/decxdence) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/decxdence/currency-exchange)

## Реализация
- К сожалению, не успел ознакомиться с деплоем, но надеюсь ты прогонял автотесты от бота сообщества.
- Советую попробовать подключить готовый фронт, это несложно, но полезно.
- По ТЗ валюта должна быть уникально по коду, не хватает соответствующего индекса в таблице.
## По коду

### package model

- BigDecimal хорош для rate.
- Хороший пакет, только я б удалил неиспользуемые конструкторы.

### package dao

- Чет не понял зачем update и delete для валюты
- Увидел, что по всему проекту используются синглтоны и это является антипаттерном (почитай почему), советую использовать DI подход через конструктор, точно также будешь делать на Spring.
- Еще приметил, что ты используешь буквально ВЕЗДЕ VAR. Это конечно приятная штука для длинных параметрических типов, но java это все таки язык со статической типизацией.
- private методы должны идти ниже публичных.
- resultset лучше юзать в try-c ресурсами
- в остальном ок

### package service

- то же самое private методы должны идти ниже публичных.
- вынеси лучше валидацию в контроллер, а в сервисе только бизнес-логика.
- вынеси маппинг в мапперы тот же MapStruct, зачем плодить код.
- я пропустил это в дао, но ты совершил ошибку и наткнулся на n+1 проблему, лучше джойнить прямо в запросе, чем потом совершать n запросов для каждой под-сущности:
```java
        List<ExchangeRateResponseDto> exchangeRates = new ArrayList<>();
        List<ExchangeRate> exchangeRatesList = exchangeRateDao.findAll();

        for (ExchangeRate exchangeRate : exchangeRatesList) {
            var baseCurrency = currencyDao.findById(exchangeRate.getBaseCurrencyId())
                    .orElseThrow(() -> new CurrencyNotFoundException());
            var targetCurrency = currencyDao.findById(exchangeRate.getTargetCurrencyId())
                    .orElseThrow(() -> new CurrencyNotFoundException());

            var baseCurrencyDto = buildCurrencyDto(baseCurrency);
            var targetCurrencyDto = buildCurrencyDto(targetCurrency);

            exchangeRates.add(new ExchangeRateResponseDto(baseCurrencyDto, targetCurrencyDto, exchangeRate.getRate()));
        }
```

- получать объект из Optional после проверки или через .orElseThrow
- обычно метод validate значит вернуть void или эксепшен, не вводи в заблуждение других

## package controller
- обработку исключений можно вынести в отдельный фильтр, "утончив" контроллеры
- Вот кстати вместо синглтона рекомендую реализовать паттерн Composition Root, например через `AppInitializer implements ServletContextListener`, который создаст все зависимости и положит их в `ServletContext` и в контроллере ты сможешь инжектить эти зависимости через конструктор.
- Есть ощущение, что много кода повторяется от сервлета к сервлету, можно вынести в утилиту либо через наследование чуть исправить ситуацию

## package exception
- dto это неизменяемый объект лучше сеттеры, либо использовать record

## Общее
- Папку `.idea` стоит добавлять в `.gitignore`, поскольку это локальная конфигурация IDE.
- Неплохо было бы оформлять коммиты в соответствии с [конвенцией](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13).
- БД не принято добавлять в репу, только миграции.
- Большая часть boilerplate-кода может быть заменена Lombok - рекомендую ознакомиться.

## Итог
- В целом норм, но про синглтон пересмотри подход, и var кмк не так много используют. Удачи!