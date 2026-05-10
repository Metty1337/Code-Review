# Отчёт аудита API

**База:** http://185.68.246.189:8080/CurrencyExchange/

**Время:** 2026-05-01 14:17:46

**Всего:** 98 | **Прошло:** 81 | **Упало:** 17

## Упавшие кейсы

### TC-011 POST /currencies валидный → статус 201

- Причина: Ожидали HTTP 201, получили 200
- Request: `POST /currencies | Form: name=CurrencyCXCE&code=XCE&sign=¤`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "code" : "XCE",
  "id" : 4,
  "name" : "CurrencyCXCE",
  "sign" : "¤"
}`

### TC-021 name= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /currencies | Form: name= &code=OPH&sign=#`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "code" : "OPH",
  "id" : 5,
  "name" : " ",
  "sign" : "#"
}`

### TC-023 code= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /currencies | Form: name=BlankCode&code= &sign=#`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "code" : " ",
  "id" : 6,
  "name" : "BlankCode",
  "sign" : "#"
}`

### TC-025 sign= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /currencies | Form: name=BlankSign&code=XKH&sign= `
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "code" : "XKH",
  "id" : 8,
  "name" : "BlankSign",
  "sign" : " "
}`

### TC-025a sign длиннее 3 символов → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /currencies | Form: name=LongSign&code=FTM&sign=ABCD`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "code" : "FTM",
  "id" : 9,
  "name" : "LongSign",
  "sign" : "ABCD"
}`

### TC-048 POST /exchangeRates валидный → 201

- Причина: Ожидали HTTP 201, получили 200
- Request: `POST /exchangeRates | Form: baseCurrencyCode=CZC&targetCurrencyCode=HSU&rate=1.5659`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "baseCurrency" : {
    "code" : "CZC",
    "id" : 10,
    "name" : "CurrencyACZC",
    "sign" : "¤"
  },
  "id" : 3,
  "rate" : 1.5659,
  "targetCurrency" : {
    "code" : "HSU",
    "id" : 11,
    "name" : "CurrencyBHSU",
    "sign" : "¤"
  }
}`

### TC-057 baseCurrencyCode= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /exchangeRates | Form: baseCurrencyCode= &targetCurrencyCode=HSU&rate=1`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "baseCurrency" : {
    "code" : " ",
    "id" : 6,
    "name" : "BlankCode",
    "sign" : "#"
  },
  "id" : 4,
  "rate" : 1,
  "targetCurrency" : {
    "code" : "HSU",
    "id" : 11,
    "name" : "CurrencyBHSU",
    "sign" : "¤"
  }
}`

### TC-059 targetCurrencyCode= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `POST /exchangeRates | Form: baseCurrencyCode=CZC&targetCurrencyCode= &rate=1`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "baseCurrency" : {
    "code" : "CZC",
    "id" : 10,
    "name" : "CurrencyACZC",
    "sign" : "¤"
  },
  "id" : 5,
  "rate" : 1,
  "targetCurrency" : {
    "code" : " ",
    "id" : 6,
    "name" : "BlankCode",
    "sign" : "#"
  }
}`

### TC-061 rate= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 409
- Request: `POST /exchangeRates | Form: baseCurrencyCode=CZC&targetCurrencyCode=HSU&rate= `
- Response: `HTTP 409 | Content-Type: application/json;charset=UTF-8 | Body: {"error": "HTTP Error 409 Conflict", "message": "Валютная пара с таким кодом уже существует"}`

### TC-071 GET /exchangeRate/{NOPEPAIR} → 404 + {message}

- Причина: Ожидали HTTP 404, получили 500
- Request: `GET /exchangeRate/ZZZYYY`
- Response: `HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"error": "Internal Server Error", "message": "Произошла ошибка при обработке запроса. Код ZZZ не найден"}`

### TC-077 PATCH rate= пусто/пробелы → 400 + {message}

- Причина: Ожидали HTTP 400, получили 200
- Request: `PATCH /exchangeRate/CZCHSU | Form: rate= `
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "baseCurrency" : {
    "code" : "CZC",
    "id" : 10,
    "name" : "CurrencyACZC",
    "sign" : "¤"
  },
  "id" : 3,
  "rate" : 20,
  "targetCurrency" : {
    "code" : "HSU",
    "id" : 11,
    "name" : "CurrencyBHSU",
    "sign" : "¤"
  }
}`

### TC-078 PATCH на несуществующую пару → 404 + {message}

- Причина: Ожидали HTTP 404, получили 500
- Request: `PATCH /exchangeRate/ZZZYYY | Form: rate=1`
- Response: `HTTP 500 | Content-Type: application/json;charset=UTF-8 | Body: {"error": "Internal Server Error", "message": "Произошла ошибка при обработке запроса. Код ZZZ не найден"}`

### TC-086 rate ≈ 1/rBA (±1e-3)

- Причина: Ожидали 0.5263157894736842, получили 1.9
- Request: `GET /exchange?from=ZFM&to=KFB&amount=10`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "amount" : 10,
  "baseCurrency" : {
    "code" : "ZFM",
    "id" : 14,
    "name" : "CurrencyAZFM",
    "sign" : "¤"
  },
  "convertedAmount" : 19.0,
  "rate" : 1.9,
  "targetCurrency" : {
    "code" : "KFB",
    "id" : 15,
    "name" : "CurrencyBKFB",
    "sign" : "¤"
  }
}`

### TC-087 convertedAmount ≈ amount*rate (±0.01)

- Причина: Ожидали 5.263157894736842, получили 19.0
- Request: `GET /exchange?from=ZFM&to=KFB&amount=10`
- Response: `HTTP 200 | Content-Type: application/json;charset=UTF-8 | Body: {
  "amount" : 10,
  "baseCurrency" : {
    "code" : "ZFM",
    "id" : 14,
    "name" : "CurrencyAZFM",
    "sign" : "¤"
  },
  "convertedAmount" : 19.0,
  "rate" : 1.9,
  "targetCurrency" : {
    "code" : "KFB",
    "id" : 15,
    "name" : "CurrencyBKFB",
    "sign" : "¤"
  }
}`

### TC-091 нет from → 400 + {message}

- Причина: Ожидали HTTP 400, получили 500
- Request: `GET /exchange?to=USD&amount=10`
- Response: `HTTP 500 | Content-Type: text/html;charset=utf-8 | Body: <!doctype html><html lang="en"><head><title>HTTP Status 500 – Internal Server Error</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 500 – Internal Server Error</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Cannot invoke &quot;String.toUpperCase()&quot; because the return value of &quot;jakarta.servlet.http.HttpServletRequest.getParameter(String)&quot; is null</p><p><b>Description</b> The server encountered an unexpected condition that prevented it from fulfilling the request.</p><p><b>Excepti...`

### TC-092 нет to → 400 + {message}

- Причина: Ожидали HTTP 400, получили 500
- Request: `GET /exchange?from=USD&amount=10`
- Response: `HTTP 500 | Content-Type: text/html;charset=utf-8 | Body: <!doctype html><html lang="en"><head><title>HTTP Status 500 – Internal Server Error</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 500 – Internal Server Error</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Cannot invoke &quot;String.toUpperCase()&quot; because the return value of &quot;jakarta.servlet.http.HttpServletRequest.getParameter(String)&quot; is null</p><p><b>Description</b> The server encountered an unexpected condition that prevented it from fulfilling the request.</p><p><b>Excepti...`

### TC-093 нет amount → 400 + {message}

- Причина: Ожидали HTTP 400, получили 500
- Request: `GET /exchange?from=USD&to=EUR`
- Response: `HTTP 500 | Content-Type: text/html;charset=utf-8 | Body: <!doctype html><html lang="en"><head><title>HTTP Status 500 – Internal Server Error</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 500 – Internal Server Error</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Cannot invoke &quot;java.lang.CharSequence.length()&quot; because &quot;this.text&quot; is null</p><p><b>Description</b> The server encountered an unexpected condition that prevented it from fulfilling the request.</p><p><b>Exception</b></p><pre>java.lang.NullPointerException: Cannot invoke...`

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
- TC-012 ответ Content-Type json
- TC-013 ответ парсится как JSON объект
- TC-014 ответ: id number
- TC-015 ответ: name string == отправленному
- TC-016 ответ: code string == отправленному
- TC-017 ответ: sign string == отправленному
- TC-018 POST /currencies повтор с тем же code → 409
- TC-019 на 409: json-ошибка содержит message
- TC-020 нет name → 400 + {message}
- TC-022 нет code → 400 + {message}
- TC-024 нет sign → 400 + {message}
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
- TC-041 элемент массива — объект
- TC-042 элемент: id number
- TC-043 элемент: baseCurrency объект
- TC-044 baseCurrency: id/name/code/sign типы ок
- TC-045 элемент: targetCurrency объект
- TC-046 targetCurrency: id/name/code/sign типы ок
- TC-047 элемент: rate number
- TC-049 ответ: json + объект
- TC-050 ответ: id number
- TC-051 ответ: baseCurrency.code == A
- TC-052 ответ: targetCurrency.code == B
- TC-053 ответ: rate number == отправленному
- TC-054 повтор POST той же пары A/B → 409
- TC-055 на 409: {message}
- TC-056 нет baseCurrencyCode → 400 + {message}
- TC-058 нет targetCurrencyCode → 400 + {message}
- TC-060 нет rate → 400 + {message}
- TC-062 base не существует → 404 + {message}
- TC-063 target не существует → 404 + {message}
- TC-064 base и target не существуют → 404 + {message}
- TC-065 GET /exchangeRate/{AB} → 200
- TC-066 ответ: json объект
- TC-067 ответ: id number
- TC-068 ответ: baseCurrency.code == A
- TC-069 ответ: targetCurrency.code == B
- TC-070 ответ: rate number == ожидаемому
- TC-072 GET /exchangeRate/ → 400 + {message}
- TC-073 PATCH /exchangeRate/{AB} валидный rate → 200
- TC-074 ответ: rate обновился (±1e-3)
- TC-075 ответ: baseCurrency/targetCurrency валидны по структуре
- TC-076 PATCH без rate → 400 + {message}
- TC-079 GET /exchange?from=A1&to=B1&amount=10 → 200
- TC-080 ответ: baseCurrency.code == A1
- TC-081 ответ: targetCurrency.code == B1
- TC-082 ответ: rate ≈ r1 (±1e-3)
- TC-083 ответ: amount == 10 (как число)
- TC-084 ответ: convertedAmount ≈ amount*rate (±0.01)
- TC-085 GET /exchange?from=A2&to=B2&amount=10 → 200
- TC-088 GET /exchange?from=A3&to=B3&amount=10 → 200
- TC-089 rate ≈ rUB / rUA (±1e-3)
- TC-090 convertedAmount ≈ amount*rate (±0.01)
- TC-094 amount=lol (не число) → 400 + {message}
- TC-095 from=XXX (валюты нет) → 404 + {message}
- TC-096 to=YYY (валюты нет) → 404 + {message}
- TC-097 обе валюты есть, но курс по 3 сценариям не получается → 404 + {message}

