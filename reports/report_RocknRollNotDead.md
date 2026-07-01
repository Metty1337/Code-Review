# Отчёт аудита API

**База:** http://193.168.46.216:8080/currency-exchange/

**Время:** 2026-07-01 09:56:39

**Всего:** 98 | **Прошло:** 91 | **Упало:** 7

## Упавшие кейсы

### TC-041 элемент массива — объект

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-042 элемент: id number

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-043 элемент: baseCurrency объект

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-044 baseCurrency: id/name/code/sign типы ок

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-045 элемент: targetCurrency объект

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-046 targetCurrency: id/name/code/sign типы ок

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

### TC-047 элемент: rate number

- Причина: Массив пуст
- Request: `GET /exchangeRates`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: []`

## Прошедшие кейсы

- TC-001 GET /currencies → статус 200
- TC-002 /currencies без редиректа
- TC-003 /currencies Content-Type начинается с application/json
- TC-004 /currencies тело парсится как JSON
- TC-005 /currencies корень — массив
- TC-006 /currencies каждый элемент массива — объект
- TC-007 элемент: есть id и это число
- TC-008 элемент: есть name и это строка
- TC-009 элемент: есть code и это строка
- TC-010 элемент: есть sign и это строка
- TC-011 POST /currencies валидный → статус 201
- TC-012 ответ Content-Type json
- TC-013 ответ парсится как JSON объект
- TC-014 ответ: id number
- TC-015 ответ: name string == отправленному
- TC-016 ответ: code string == отправленному
- TC-017 ответ: sign string == отправленному
- TC-018 POST /currencies повтор с тем же code → 409
- TC-019 на 409: json-ошибка содержит message
- TC-020 нет name → 400 + {message}
- TC-021 name= пусто/пробелы → 400 + {message}
- TC-022 нет code → 400 + {message}
- TC-023 code= пусто/пробелы → 400 + {message}
- TC-024 нет sign → 400 + {message}
- TC-025 sign= пусто/пробелы → 400 + {message}
- TC-025a sign длиннее 3 символов → 400 + {message}
- TC-026 GET /currency/{C1} → 200
- TC-027 /currency/{C1} Content-Type json
- TC-028 /currency/{C1} тело — JSON объект
- TC-029 объект: id number
- TC-030 объект: name string
- TC-031 объект: code string == C1
- TC-032 объект: sign string
- TC-033 GET /currency/{NOPE} → 404
- TC-034 на 404: json-ошибка содержит message
- TC-035 GET /currency/ → 400
- TC-036 на 400: json-ошибка содержит message
- TC-037 GET /exchangeRates → 200
- TC-038 /exchangeRates без редиректа
- TC-039 /exchangeRates Content-Type json
- TC-040 /exchangeRates корень — массив
- TC-048 POST /exchangeRates валидный → 201
- TC-049 ответ: json + объект
- TC-050 ответ: id number
- TC-051 ответ: baseCurrency.code == A
- TC-052 ответ: targetCurrency.code == B
- TC-053 ответ: rate number == отправленному
- TC-054 повтор POST той же пары A/B → 409
- TC-055 на 409: {message}
- TC-056 нет baseCurrencyCode → 400 + {message}
- TC-057 baseCurrencyCode= пусто/пробелы → 400 + {message}
- TC-058 нет targetCurrencyCode → 400 + {message}
- TC-059 targetCurrencyCode= пусто/пробелы → 400 + {message}
- TC-060 нет rate → 400 + {message}
- TC-061 rate= пусто/пробелы → 400 + {message}
- TC-062 base не существует → 404 + {message}
- TC-063 target не существует → 404 + {message}
- TC-064 base и target не существуют → 404 + {message}
- TC-065 GET /exchangeRate/{AB} → 200
- TC-066 ответ: json объект
- TC-067 ответ: id number
- TC-068 ответ: baseCurrency.code == A
- TC-069 ответ: targetCurrency.code == B
- TC-070 ответ: rate number == ожидаемому
- TC-071 GET /exchangeRate/{NOPEPAIR} → 404 + {message}
- TC-072 GET /exchangeRate/ → 400 + {message}
- TC-073 PATCH /exchangeRate/{AB} валидный rate → 200
- TC-074 ответ: rate обновился (±1e-3)
- TC-075 ответ: baseCurrency/targetCurrency валидны по структуре
- TC-076 PATCH без rate → 400 + {message}
- TC-077 PATCH rate= пусто/пробелы → 400 + {message}
- TC-078 PATCH на несуществующую пару → 404 + {message}
- TC-079 GET /exchange?from=A1&to=B1&amount=10 → 200
- TC-080 ответ: baseCurrency.code == A1
- TC-081 ответ: targetCurrency.code == B1
- TC-082 ответ: rate ≈ r1 (±1e-3)
- TC-083 ответ: amount == 10 (как число)
- TC-084 ответ: convertedAmount ≈ amount*rate (±0.01)
- TC-085 GET /exchange?from=A2&to=B2&amount=10 → 200
- TC-086 rate ≈ 1/rBA (±1e-3)
- TC-087 convertedAmount ≈ amount*rate (±0.01)
- TC-088 GET /exchange?from=A3&to=B3&amount=10 → 200
- TC-089 rate ≈ rUB / rUA (±1e-3)
- TC-090 convertedAmount ≈ amount*rate (±0.01)
- TC-091 нет from → 400 + {message}
- TC-092 нет to → 400 + {message}
- TC-093 нет amount → 400 + {message}
- TC-094 amount=lol (не число) → 400 + {message}
- TC-095 from=XXX (валюты нет) → 404 + {message}
- TC-096 to=YYY (валюты нет) → 404 + {message}
- TC-097 обе валюты есть, но курс по 3 сценариям не получается → 404 + {message}

