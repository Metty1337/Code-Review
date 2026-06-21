# Review на реализацию от [@AmanbekAzizUulu](https://github.com/AmanbekAzizUulu) проекта [Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)

[Сама реализация](https://github.com/AmanbekAzizUulu/simple-currency-exchange-service)

## Реализация

- По тестам бота все пройдено - супер
- Тестовый фронт подключен.
- Добавил дополнительную страничку с информацией об эндпоинтах - тоже плюс.

## По коду

### package model

- На мой взгляд излишнее использование аннотации `@Data`, которая навешивает сразу тонну аннотаций, из которых тебе
  нужны только геттеры и сеттеры.
- Использования билдера для 4 полей обычно считается избыточным, почему бы не использовать конструктор.
- В остальном классика.

### package dao

- Выделены интерфейсы - плюс.
- Имхо - более идиоматичным названием для метода, который сохраняет сущность, является `save()`, а не `persist()`.
- Неожиданное проглатывание исключений, стоит оборачивать SQL исключение в исключение, понятное верхним слоям:

```
		} catch (SQLException e) {
			LOGGER.error("Error fetching all currencies", e); 
		}
		
// лучше
		} catch (SQLException e) {
			LOGGER.error("Error fetching all currencies", e);
			throw new  DataAccessException("Error fetching all currencies", e);
		}
```

- Заметил, что часто переиспользуешь сеттеры, там где можно обойтись лишь одним конструктором:

```
				ExchangeRate rate = new ExchangeRate();
				rate.setId(rs.getLong("id"));
				rate.setBaseCurrencyId(rs.getLong("base_currency_id"));
				rate.setTargetCurrencyId(rs.getLong("target_currency_id"));
				rate.setRate(rs.getBigDecimal("rate"));
				rates.add(rate);
				
// лучше
				rates.add(new ExchangeRate(
						rs.getLong("id"),
						rs.getLong("base_currency_id"),
						rs.getLong("target_currency_id"),
						rs.getBigDecimal("rate")
				));
```

- Эти самые строки "id", "base_currency_id" и тд каждый раз повторяются, можно вынести их в константы:
```
			while (rs.next()) {
				ExchangeRate rate = new ExchangeRate();
				rate.setId(rs.getLong("id"));
				rate.setBaseCurrencyId(rs.getLong("base_currency_id"));
				rate.setTargetCurrencyId(rs.getLong("target_currency_id"));
				rate.setRate(rs.getBigDecimal("rate"));
				rates.add(rate);
			}

// лучше
			while (rs.next()) {
				ExchangeRate rate = new ExchangeRate();
				rate.setId(rs.getLong(COL_ID));
				rate.setBaseCurrencyId(rs.getLong(COL_BASE_CURRENCY_ID));
				rate.setTargetCurrencyId(rs.getLong(COL_TARGET_CURRENCY_ID));
				rate.setRate(rs.getBigDecimal(COL_RATE));
				rates.add(rate);
			}
```
- Круто, что расшифровываешь исключения по SQLState, единственное я бы подумал может стоит вынести условия в отдельный
  метод, класс.

### package service

- За выделенные интерфейсы - плюс.
- Принимаешь DTO, отдаешь DTO - все правильно.
- Не проще ли в этой ситуации создать отдельный конструктор вместо билдера:

```
Currency currency = Currency.builder().code(code).fullName(name).sign(sign).build();

// лучше
Currency currency = new Currency(code, name, sign)
```

- Валидация входных данных - ответственность контроллера, а не сервиса:

```
try {
			if (!isValidCurrencyFullName(name)) {
				throw new InvalidCurrencyFullNameException("Invalid currency full name: " + request.name());
			}
			if (!isValidCurrencySign(sign)) {
				throw new InvalidCurrencySignException("Invalid currency sign: " + request.sign());
			}
			if (!isValidCurrencyCode(code)) {
				throw new InvalidCurrencyCodeException("Invalid currency code: " + request.code());
			}
// ответственность контроллера, не сервиса
```

- Неожиданные sout вместо логгера:

```
		} catch (DataAccessException e) {
			System.out.println("DataAccessException");
			throw new InternalServerErrorException("Database error", e);
		}
```

- Метод, достойный отдельного маппера, странно, что он в сервисе, , то же касается `ExchangeRateService`:

```
	private CurrencyResponseDto mapToDto (Currency currency) {
		return new CurrencyResponseDto(currency.getId(), currency.getFullName(), currency.getCode(), currency.getSign());
	}
```

- `isValidCurrencyCode`, `isValidCurrencySign`, `isValidCurrencyFullName` - методы достойные отдельного валидатора, то
  же касается `ExchangeRateService`.
- Часто встречаю неимоверно длинные строчки, где интуитивно необходим хотя бы один Enter:

```
Currency target = currencyDao.findByCode(request.targetCurrencyCode().toUpperCase()).orElseThrow( () -> new TargetCurrencyNotFoundException(request.targetCurrencyCode()));

// лучше
Currency target = currencyDao.findByCode(request.targetCurrencyCode().toUpperCase())
				.orElseThrow( () -> new TargetCurrencyNotFoundException(request.targetCurrencyCode()));
```

- Очень некрасивый огромный метод `exchange` по нахождению курса, тебе даже пришлось оставлять несколько комментариев,
  чтобы в этом коде можно было разобраться. В таких случаях стоит заняться декомпозицией - воспользоваться
  вспомогательными методами:

```
	public ExchangeResponseDto exchange (String from, String to, BigDecimal amount) {
		Currency base = currencyDao.findByCode(from).orElseThrow(() -> new BaseCurrencyNotFoundException(from));
		Currency target = currencyDao.findByCode(to).orElseThrow(() -> new TargetCurrencyNotFoundException(to));

		BigDecimal rate;

		// direct exchange
		Optional<ExchangeRate> direct = exchangeRateDao.findByPair(from, to);
		if (direct.isPresent()) {
			rate = direct.get().getRate();
			return toExchangeResponseDto(rate, amount, base, target);
		}

		// inverse rate
		Optional<ExchangeRate> inverse = exchangeRateDao.findByPair(to, from);
		if (inverse.isPresent()) {
			rate = BigDecimal.ONE.divide(inverse.get().getRate(),RATE_SCALE,RoundingMode.HALF_UP);
			return toExchangeResponseDto(rate, amount, base, target);
		}

		// cross rate
		Optional<ExchangeRate> crossCurrencyToFrom = exchangeRateDao.findByPair(CURRENCY_CODE_FOR_CROSS_RATE_EXCHANGE, from);
		Optional<ExchangeRate> crossCurrencyToTo = exchangeRateDao.findByPair(CURRENCY_CODE_FOR_CROSS_RATE_EXCHANGE, to);

		if (crossCurrencyToFrom.isPresent() && crossCurrencyToTo.isPresent()) {
			BigDecimal rateTo = crossCurrencyToTo.get().getRate();
			BigDecimal rateFrom = crossCurrencyToFrom.get().getRate();

			if (rateFrom.compareTo(BigDecimal.ZERO) == 0) {
				throw new InternalServerErrorException("USD→from rate is zero");
			}

			rate = rateTo.divide(rateFrom, RATE_SCALE, RoundingMode.HALF_UP);
			LOGGER.debug("Cross rate USD→{}={}, USD→{}={}, calculated rate={}", from, rateFrom, to, rateTo, rate);

			return toExchangeResponseDto(rate, amount, base, target);
		}

		// в случае если не найден курс обмена
		throw new ExchangeRateNotFoundException(from, to);
	}

// в идеале метод может выглядеть так, он не требует дополнительных комметариев, т.к. имена вспомогательных методов делают картину ясной и так:
          public ExchangeDto getExchange(String base, String target, BigDecimal amount) {
              return findDirect(base, target).or(() -> findReverse(base, target))
                                             .or(() -> findCross(base, target))
                                             .orElseThrow(() -> ModelNotFoundException(...));}
	
```
### package controller

- В данный момент зависимости для контроллера получаются путем создания их через конструктор, у этого способа есть очевидные минуса, поэтому советую ознакомится с паттерном Composition Root, в jakarta есть лаконичный способ его реализации.
- Регистрировал сервлеты как я понял через конфиг (в репе он побитый), но также есть возможность через аннотацию над контроллером, пример `@WebServlet("/currencies")`.
- В остальном вроде все ок.

### package dto
- Имхо лишние билдеры для рекордов.
- `RequestDto` звучит как масло масляное, request уже предплогает, что это DTO.

### package exception
- Пакеты в Java называют в нижнем регистре (lowercase), используя обратную нотацию доменного имени. То есть вместо `bad-request` - лучше `bad.request` и тд.
- Построил удобную иерархию исключений, только минус, что они все жестко привязаны к HTTP.

## Общее

- Заметил странный стиль - ставишь пробел после "@" в аннотациях, так никто не делает.
- Как вариант улучшения мапперов (и само их добавление) - можно ознакомиться с _MapStruct_.
- Не игнорируй замечания от idea - чаще всего они полезны.
  Магические числа/строки стоит заменять константами.
- Используй чаще форматирование (ctrl+alt+l в idea) - есть неаккуратные места.

## Итог
- Неплохой проект, критических замечаний не было, учти нюансы и смело иди дальше. Удачи!