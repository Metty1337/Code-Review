
# Review на реализацию от [@RocknRollNotDead](https://github.com/RocknRollNotDead) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/RocknRollNotDead/currency-exchange)

## Реализация

- Сделал [отсчет](../reports/report_RocknRollNotDead.md) через бота, есть пару падающих тестов, но в целом хорошо.
- Работающий прикрученный фронт - хорошо.

## По коду

### package models
- Из того, что видел обычно пакет с моделями называют в единственном числе.
- Более семантически верным, как мне кажется, является использование `Long` для id.
- BigDecimal для rate идеально.

### package db

- Бывает, что всякие инфраструктурные штуки выносят в отдельный пакет `config`, а сам пакет с дао называют `dao` или `repository`.
- sql запрос в целом можно вынести хотя бы в локальные переменные, ResultSet лучше использовать только через try с ресурсами, а названия колонок можно вынести в константы, ты ведь их в каждом методе используешь, можно переиспользовать.
```java
        try (PreparedStatement stmt = conn.prepareStatement(
                "SELECT id, code, full_name, sign FROM currencies")){
            List<Currency> currencies = new ArrayList<>();
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                Currency currency = new Currency(rs.getInt("id"),
                        rs.getString("code"),
                        rs.getString("full_name"),
                        rs.getString("sign") );
                currencies.add(currency);
            }
            ...
// лучше
    public static final String ID_COL = "id";
    public static final String CODE_COL = "code";
    public static final String FULL_NAME_COL = "full_name";
    public static final String SIGN_COL = "sign";
        ...
        
        String getAllQuery = "SELECT id, code, full_name, sign FROM currencies";
        try (PreparedStatement stmt = conn.prepareStatement(
                getAllQuery)){
            List<Currency> currencies = new ArrayList<>();
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    Currency currency = new Currency(rs.getInt(ID_COL),
                            rs.getString(CODE_COL),
                            rs.getString(FULL_NAME_COL),
                            rs.getString(SIGN_COL));
                    currencies.add(currency);
                }
            }
            ...
            
```
- Сигнатура метода `public int addCurrency(String code, String fullName, String sign)` не типична, если я правильно понимаю, то если у нас не вставится значение, то пробросится исключение, соответственно дополнительно проверять кол-во возвращенных строк, как мне кажется, излишне.
- Никогда не возвращай null, в java есть специальный класс Optional<T>, который реализует безопасную работу с null.
```java
    public Currency findByCode(String code){
            ...
            if (rs.next()) {
                return new Currency(rs.getInt("id"),
                        rs.getString("code"),
                        rs.getString("full_name"),
                        rs.getString("sign")

                );
            }
            return null;
    }
    
// лучше
    public Optional<Currency> findByCode(String code){
            ...
            if (rs.next()) {
                return Optional.ofNullable(new Currency(rs.getInt("id"),
                        rs.getString("code"),
                        rs.getString("full_name"),
                        rs.getString("sign")

                ));
            }
            return Optional.empty();
    }
```
- Нашел комментарий:
```java
    public int deleteRate(int id){ // DAO не должен знать, что делает Service. DAO должен только давать методы для
        try (PreparedStatement stmt = conn.prepareStatement( // CRUD. Он не должен знать, используется там delete или нет.
```
В этом конечно есть частичка правды, но на работе ты также команде будешь рассказывать про 100 строк неиспользуемого кода?) Все в разумных пределах.

- Можно также добавить интерфейсы, соблюсти принцип инверсии зависимости.

### package service

- Если мы стремимся к соблюдению принципа разделения ответственности, то DataSource и Connection к БД хотелось бы оставить в нашем слое `DAO`. Иначе роль dao слоя как слоя обращения к бд будет нарушена.
- `SQLException` хочется ловить в слое дао, а не в сервисе.
- Нашел комментарий про валидацию, которая находится в сервисе:
```java
// Принципиально не буду делать Validator классы ради 40 строчек. 1. Это задача сервиса. 2. Будет менее читаемо

    private void validateCode(String code){
        if (!ADMISSION_CODE.matcher(code).matches()){       //  code.matches("^[A-Z0-9]{3}$")
            throw new ValidationException("Code must be latin and length 3 symbols");
        }
    }
```
Во первых валидация формата ДТО это ответственность контроллера, во вторых почему будет менее читабельно? Почему тебе жалко класс ради 40 строчек используемого кода (тем самым, соблюдая разделенную ответственность), но не жаль около 100 строк мертвого код ради разделенной ответственности? Какой-то карго-культ.

- Нарушаешь принцип разделенной ответственности, создавай другой сервис в контроллере, тем самым получается ситуация, в которой другой сервис управляет жизненным циклом другого сервиса. Можно просто инжектить в контроллере:
```java
    public ExchangeRateService(DataSource dataSource) {
        currencyService = new CurrencyService(dataSource);
        this.dataSource = dataSource;
    }
    
// лучше
    public ExchangeRateService(CurrencyService currencyService, DataSource dataSource) {
        this.currencyService = currencyService;
        this.dataSource = dataSource;
    }
```
- Карго-культ:
```
        int baseCurrencyId = currencyService.getIdFromCode(baseCurrencyCode);
        int targetCurrencyId = currencyService.getIdFromCode(targetCurrencyCode);
        // в идеале не делать эти лишние запросы в sql, а добавлять в RatesDao не по ID, а сразу по коду,
        // но тогда это будет не чистый mvc - DAO не должен знать про code, его задача здесь - просто добавить в бд строчку
```
- Метод `calculateRate` получился не самым легко читабельным, я бы стремился к чему-то такому по простоте:
```java
    public ExchangeDto calculateRate(String base, String target, BigDecimal amount) {
        return findDirect(base, target)
                .or(() -> findReverse(base, target))
                .or(() -> findCross(base, target))
                .orElseThrow(() -> ModelNotFoundException(...));
    }
```

### package dto

- В целом класс из рекордов, все как надо.
- Нашел комментарий:
```java
// Это повторение класса Currency (это модель). Оно нужно только для соблюдение тз. На работу программы не влияет никак.

public record CurrencyDto(int id, String code, String name, String sign) {

}
```
Если появляются такие вопросы, то стоит еще раз обратиться к определению DTO.

### package mapper
- Прикол MapStruct в том, что он сам генерирует реализацию, поэтому нет смысла использовать синглтон:
```java
@Mapper
public interface CurrencyMapper {
    CurrencyMapper INSTANCE = Mappers.getMapper(CurrencyMapper.class);

    @Mapping(source = "fullName", target = "name")
    CurrencyDto toDto(Currency currency);


    List<CurrencyDto> toDtoList(List<Currency> currencies);

    Currency toModel(CurrencyDto dto);
}
```
### package servlets

- В сервлете `CurrenciesServlet` объединенно несколько GET эндпоинтов, что конечно нарушает знакомый тебе принцип разделенной ответственности. Не стоит жалеть классов. Тоже касается остальных сервлетов:
```java
@WebServlet(urlPatterns = {"/currency/*", "/currencies"})
public class CurrenciesServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String servletPath = req.getServletPath();
        String path = req.getPathInfo();

        String json;
        String code;
        if (servletPath.equals("/currencies")) {
            json = gson.toJson(currencyService.getAllCurrencies());
        } else {
            try{
                code = path.substring(1);
            } catch (NullPointerException e){
                throw new UncorrectRequestException("request must be not null");
            }

            if (code.length() == 3){

                json = gson.toJson(currencyService.getCurrency(code));
```
- Нашел комментарий:
```java
        if (servletPath.equals("/exchangeRates")) { // знаю про очистку от rateService.getRate("/USD/give/one/response"),
            json = gson.toJson(exchangeRateService.getAllExchangeRates()); // но в spring boot оно само это делается, а тут только код засорит
```
Действительно, в Spring MVC есть встроенный Jackson для сериализации. Раз ты на него опираешься, то как раз-то в спринге ты не сможешь в методе, который предполагает один эндпоинт, расположить сразу 2. А сам Spring MVC построен поверх Servlet API. Это еще одна причина почему не стоит 2 эндпоинта размещать в одном методе.

- Отсутствующий модификатор доступа:
```java
Gson gson = new Gson();
```
- В целом мне сервлеты показались довольно толстыми, что является анти-паттерном fat controller. Можно попробовать что-то вынести в другие классы. Ты конечно не наплодил "мусорные", по твоим словам, классы, но и контроллеры вышли объемными.


## Общее
- Сам писал readme, такое редко встречается - круто, а может я уже не различаю ИИ творения.
- Папка frontend-main как будто лишняя, поскольку внутри бека уже в webapps скопирован фронт. Так же если у нас есть папка фронта, то должна быть и бека (а не сразу src/main). То есть хочется соблюдать какую-то равную иерархическую структуру. Надеюсь передал мысль.
- Папки .idea и .smarttomcat обычно добавляют в .gitignore, поскольку это больше про локальные настройки.
- Ctrl+Alt+L обязателен.

## Итог

- Проект рабочий, стоит приглядеться к принципу S из SOLID. Учти нюансы и можешь смело идти дальше. Удачи!