
# Review на реализацию от [@Ephirious](https://github.com/Ephirious) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/Ephirious/currencies-exchanger)

## Реализация
- Все автотесты прошли успешно - молодец
- Прикручен фронт сообщества - супер

## По коду

### package entities

- Аннотация @Data как будто излишне, туда ведь входит куча аннотаций и ты точно использовать будешь меньшинство.
- `BigDecimal` для `rate` отлично.

### package dao 

- В таблицах принято называть колонки в lower_case, то есть не `fullname`, а `full_name`.
- Интересный родитель, классный дизайн методов. Круто, что нашел место использовать табличный выражения, вопросов к пакету никаких, все отлично, но можно еще выделить интерфейсы.

### package services

- В целом стандарт разработки писать сообщение об исключении на английском:
```java
.orElseThrow(() -> new NotFoundException("Валюта с кодом %s не была найдена".formatted(code)));

// лучше
.orElseThrow(() -> new NotFoundException("Failed to find currency %s".formatted(code)));
```
- Как вариант некого упрощения можно создать специальный конструктор под нужное число параметров, чтобы не оперировать напрямую с null:
```java
    Currency currency = new Currency(null, code, name, sign);

// лучше

    public Currency(String code, String name, String sign) {
        this.code = code;
        this.name = name;
        this.sign = sign;
    }

    Currency currency = new Currency(code, name, sign);
```
- Доставать объект из Optional все таки стоит либо через `ifPresent` или через тот же `orElseThrow()`:
```java
        return currencyDao.insert(currency)
                .map(CurrencyDTO::fromCurrency)
                .get();
// лучше
        return currencyDao.insert(currency)
                .map(CurrencyDTO::fromCurrency)
                .orElseThrow(() -> new NotFoundException("Failed to add currency %s".formatted(code)));

                
```
- В каждом методе принимается ДТО, отдается ДТО - все как надо.
- Важные бизнес-значения вынесены в константы - супер:
```java
private static final int AMOUNT_PRECISION = 2;
```

### package controller

- Получаешь зависимости через DI контейнер, а не создаешь - супер:
```java
    @Override
    public void init() throws ServletException {
        ApplicationContainer container = (ApplicationContainer) getServletContext()
                .getAttribute(ApplicationContext.APPLICATION_ATTRIBUTE);

        currencyService = container.get(CurrencyService.class);
        mapper = container.get(ObjectMapper.class);
    }
```
- ServletOutputStream нет смысла использовать в try с ресурсами, поскольку он не реализует интерфейс Autoclosable или Closable, тем более в апишке написано, что закрытием его занимается контейнер.
```java
        try (ServletOutputStream outputStream = response.getOutputStream()) {
            mapper.writeValue(outputStream, currencies);
        }
```
- Очень хорошие контроллеры получились, все методы получились тонкими благодаря утилитным классам:
```java
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String codeFrom = ServletUtils.getParamOrThrow(request, "from");
        String codeTo = ServletUtils.getParamOrThrow(request, "to");
        String amount = ServletUtils.getParamOrThrow(request, "amount");

        CurrencyValidator.ensureCode(codeFrom);
        CurrencyValidator.ensureCode(codeTo);
        ensureAmount(amount);

        mapper.writeValue(response.getOutputStream(), exchangeService.exchange(codeFrom, codeTo, amount));
    }
```

- Единственное нет необходимости писать свои enum для кодов ошибок, они все есть у `HttpServletResponse`.


### package utils

- Все утилитные классы final модификатор и private конструктор - все как полагается.


## Общее
- По поводу структуры проекта, я бы обернул src/main в пакет backend, поскольку по соседству у нас есть frontend, то есть сохранил бы некое равенство уровня пакета, надеюсь донес мысль.

## Итог
- Наверное один из самых лучших третьих проектов, что я видел. Учитывая, что ты способен сделать проект такого уровня качества за 10 дней, то посоветовал бы тебе следующие проекты делать не усложняя себе задачу, чисто базовое ТЗ и пошли дальше, если уже имеется опыт за плечами. Я имею в виду, например, вместо усложнения с общим родительским классом через генерики и ФИ можно сделать 2 простых дао класса и тд, вместо 113 коммитов у тебя выйдет гораздо меньше и тем раньше ты поступишь в менторство, если конечно такие мотивы есть. Смело иди дальше, удачи!
