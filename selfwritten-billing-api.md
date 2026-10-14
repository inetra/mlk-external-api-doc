# API самописного биллинга для МЛК

> [!warning]
> Документация соответствует МЛК версии 26.10.2

## Назначение

Документ описывает HTTP API, которое **биллинг оператора** должен реализовать, чтобы мобильное приложение
МЛК могло с ним работать. Бэкенд МЛК выступает клиентом: он вызывает перечисленные ниже методы биллинга
от имени абонента и преобразует ответы для мобильного приложения.

Описание составлено по фактическому поведению бэкенда МЛК. Поля, которые МЛК не читает, можно не передавать;
обязательные поля помечены в таблицах.

---

## Общие положения

### Базовый адрес

Адрес биллинга задаётся на стороне МЛК переменной `BILLING_URL`. Все пути ниже указаны **относительно** него.

```
BILLING_URL=https://billing.example.com/api/v1/
```

Адрес должен заканчиваться на `/`, иначе последний сегмент пути будет отброшен.
Пример: метод `subscriber/accounts` будет вызван как `https://billing.example.com/api/v1/subscriber/accounts`.

Таймаут запроса со стороны МЛК — 50 секунд.

### Формат запросов

- Параметры в теле `POST`/`PATCH` передаются как **`application/x-www-form-urlencoded`** (не JSON).
- Каждый запрос содержит заголовок `Accept: application/json`.
- Даты в запросах передаются в формате `Y-m-d H:i:s` (например, `2026-10-01 00:00:00`).
- Пути передаются ровно так, как указано в документе, включая завершающий `/` там, где он есть.

### Формат ответов

Все ответы — JSON в стандартной обёртке:

```json
{
  "success": true,
  "data": {},
  "message": "success",
  "code": 200
}
```

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `success` | boolean | да | Признак успешного выполнения |
| `data` | any | да | Полезная нагрузка, описана для каждого метода |
| `message` | string | нет | Текстовое сообщение (при ошибке показывается в логах МЛК) |
| `code` | integer | см. метод | Код результата. Для части методов МЛК проверяет `code == 200` (отмечено в описании) |

Ответ с ошибкой:

```json
{
  "success": false,
  "data": null,
  "message": "incorrect username or password",
  "code": 401
}
```

Ошибки следует возвращать с соответствующим HTTP-статусом. Если биллинг ответил статусом 4xx, МЛК передаёт
приложению тот же статус и однострочный `message` из ответа — его увидит абонент. Статусы 5xx и сетевые
ошибки приложение получает как `502 Billing request failed`.

Даты в ответах — строки, которые разбираются PHP `DateTime`. Рекомендуемый формат — ISO 8601 со смещением:
`2026-10-20T14:34:10+0300`. Формат `2026-10-20 14:34:10` также принимается.

Суммы — числа (`float`), валюта — объект [Currency](#currency).

### Авторизация абонента

1. Абонент вводит в приложении логин и пароль от биллинга. МЛК вызывает [`POST auth/token`](#post-authtoken)
   и получает JWT.
2. Все запросы от имени абонента МЛК отправляет с заголовком `Authorization: Bearer <jwt>`.
3. МЛК кэширует JWT. Если в payload токена есть поле `exp` (Unix time), МЛК обновляет токен заранее, за 60 секунд
   до истечения. **Рекомендуется всегда указывать `exp`.**
4. Если биллинг вернул ошибку (HTTP 4xx/5xx), в тексте которой есть строка `Expired token`, МЛК заново вызывает
   `POST auth/token` с сохранёнными логином и паролем абонента и повторяет исходный запрос один раз.

Подпись JWT проверяет только биллинг, МЛК читает лишь `exp`. Содержимое токена биллинг определяет сам.
**Не помещайте в JWT логин и пароль абонента** — payload токена не шифруется.

Методы из раздела [Без авторизации абонента](#без-авторизации-абонента) вызываются без заголовка `Authorization`.
Их рекомендуется ограничить на стороне биллинга, например по IP-адресу сервера МЛК.

---

## Сводка методов

| Метод | Путь | Авторизация | Назначение |
|-------|------|-------------|------------|
| POST | `auth/token` | — | [Вход абонента](#post-authtoken) |
| POST | `auth/logout` | JWT | [Выход абонента](#post-authlogout) |
| GET | `subscriber` | JWT | [Данные абонента](#get-subscriber) |
| GET | `subscriber/accounts` | JWT | [Список лицевых счетов](#get-subscriberaccounts) |
| GET | `subscriber/accounts/{accountId}/` | JWT | [Лицевой счёт](#get-subscriberaccountsaccountid) |
| PATCH | `subscriber/accounts/{accountId}/` | JWT | [Блокировка, разблокировка, обещанный платёж](#patch-subscriberaccountsaccountid) |
| GET или POST | `subscriber/accounts/{accountId}/transactions` | JWT | [История операций](#subscriberaccountsaccountidtransactions) |
| GET | `subscriber/accounts/{accountNumber}/pay-link/?amount={amount}` | JWT | [Ссылка на оплату](#get-subscriberaccountsaccountnumberpay-link) |
| GET | `subscriber/accounts/{accountNumber}/auto-payment-link/?amount={amount}` | JWT | [Подключение автоплатежа](#get-subscriberaccountsaccountnumberauto-payment-link) |
| GET | `subscriber/accounts/{accountNumber}/auto-payment-off` | JWT | [Отключение автоплатежа](#get-subscriberaccountsaccountnumberauto-payment-off) |
| GET | `subscriber/accounts/{accountId}/services` | JWT | [Услуги](#get-subscriberaccountsaccountidservices) |
| GET | `subscriber/accounts/{accountId}/services/{serviceId}/tariffs` | JWT | [Тарифы для смены](#get-subscriberaccountsaccountidservicesserviceidtariffs) |
| PATCH | `subscriber/accounts/{accountId}/services/{serviceId}/` | JWT | [Смена тарифа, блокировка услуги](#patch-subscriberaccountsaccountidservicesserviceid) |
| PATCH | `subscriber/accounts/{accountId}/services/{serviceId}` | JWT | [Разблокировка услуги](#patch-subscriberaccountsaccountidservicesserviceid) |
| PATCH | `subscriber/accounts/{accountId}/services/{serviceId}/suspension-service` | JWT | [Услуга приостановки](#patch-subscriberaccountsaccountidservicesserviceidsuspension-service) |
| GET | `subscriber/accounts/{accountId}/services/{serviceId}/additional-services` | JWT | [Доступные доп. услуги](#get-subscriberaccountsaccountidservicesserviceidadditional-services) |
| PATCH | `subscriber/accounts/{accountId}/services/{serviceId}/additional-services/{additionalServiceId}/` | JWT | [Подключение и отключение доп. услуги](#patch-subscriberaccountsaccountidservicesserviceidadditional-servicesadditionalserviceid) |
| GET | `resources/promised-pay-terms` | JWT | [Условия обещанного платежа](#get-resourcespromised-pay-terms) |
| GET | `restore-password/find-mobile/{phone}` | — | [Поиск абонента по телефону](#get-restore-passwordfind-mobilephone) |
| POST | `restore-password/` | — | [Реквизиты для восстановления доступа](#post-restore-password) |
| POST | `support/send-email` | — | [Обращение в техподдержку](#post-supportsend-email) |

---

## Авторизация

### POST auth/token

Вход абонента. Вызывается при входе в приложение и при обновлении истёкшего JWT.

**Тело запроса:**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `username` | string | да | Логин абонента |
| `password` | string | да | Пароль абонента |

**Успешный ответ:**

```json
{
  "success": true,
  "data": {
    "id": "100001",
    "jwt": "<JWT>"
  },
  "message": "success",
  "code": 200
}
```

| Поле `data` | Тип | Обяз. | Описание |
|-------------|-----|-------|----------|
| `id` | string | да | Идентификатор абонента в биллинге |
| `jwt` | string | да | JWT для последующих запросов, желательно с `exp` |

Вход считается успешным только при `success: true` и наличии `data.id` и `data.jwt`.
При неверных учётных данных верните HTTP 401:

```json
{
  "success": false,
  "message": "incorrect username or password",
  "code": 401
}
```

### POST auth/logout

Выход абонента из приложения. Тело запроса пустое.

**Успешный ответ:**

```json
{ "success": true, "message": "success", "code": 200 }
```

Ошибка этого метода не мешает выходу из приложения.

---

## Абонент

### GET subscriber

Данные абонента.

**Успешный ответ:**

```json
{
  "success": true,
  "data": {
    "id": "100001",
    "first_name": "Иван",
    "last_name": "Иванов",
    "second_name": "Иванович"
  },
  "message": "success",
  "code": 200
}
```

| Поле `data` | Тип | Обяз. | Описание |
|-------------|-----|-------|----------|
| `first_name` | string | да | Имя |
| `last_name` | string | да | Фамилия |
| `second_name` | string | нет | Отчество |

Основным лицевым счётом абонента МЛК считает **первый** счёт из [`GET subscriber/accounts`](#get-subscriberaccounts).

---

## Лицевые счета

### GET subscriber/accounts

Список лицевых счетов абонента. МЛК берёт из ответа только `id` и для каждого счёта отдельно вызывает
[`GET subscriber/accounts/{accountId}/`](#get-subscriberaccountsaccountid).

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    { "id": "100001", "number": "100001" },
    { "id": "100002", "number": "100002" }
  ],
  "message": "success",
  "code": 200
}
```

| Поле элемента | Тип | Обяз. | Описание |
|---------------|-----|-------|----------|
| `id` | string | да | Идентификатор лицевого счёта |

### GET subscriber/accounts/{accountId}/

Лицевой счёт.

**Успешный ответ:**

```json
{
  "success": true,
  "data": {
    "id": "100001",
    "number": "100001",
    "suspend_allow": true,
    "auto_payment": false,
    "balance": {
      "total": -350.5,
      "recommended_pay": 650,
      "pay_until": "2026-10-20T00:00:00+0300",
      "promised_pay": {
        "sum": 600,
        "promised_until": "2026-10-23T00:00:00+0300",
        "status": "taken"
      },
      "currency": { "id": 1, "code": "RUB", "title": "Руб" }
    },
    "suspend_info": {
      "suspend_from": "2026-11-01 00:00:00",
      "suspend_to": "2026-11-15 00:00:00"
    }
  },
  "message": "success",
  "code": 200
}
```

**Account (`data`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `id` | string | да | Идентификатор лицевого счёта, используется как `{accountId}` |
| `number` | string | да | Номер лицевого счёта, показывается абоненту и используется как `{accountNumber}` в методах оплаты |
| `suspend_allow` | boolean | да | Разрешена ли добровольная блокировка счёта |
| `auto_payment` | boolean | нет | Подключён ли автоплатёж |
| `balance` | [Balance](#balance) | да | Баланс |
| `suspend_info` | object или пустое значение | нет | Текущая добровольная блокировка. Если блокировки нет — `[]`, `null` или поле отсутствует |
| `suspend_info.suspend_from` | string (дата) | да, если блокировка есть | Начало блокировки |
| `suspend_info.suspend_to` | string (дата) | да, если блокировка есть | Окончание блокировки |

#### Balance

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `total` | number | да | Баланс лицевого счёта |
| `pay_until` | string (дата) | да | Дата, до которой нужно оплатить |
| `recommended_pay` | number | нет | Рекомендуемая сумма к оплате |
| `promised_pay` | [PromisedPay](#promisedpay) | нет | Состояние обещанного платежа |
| `currency` | [Currency](#currency) | нет | Валюта |

#### PromisedPay

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `sum` | number | да | Сумма обещанного платежа |
| `status` | string | да | `available` — можно взять, `taken` — взят, `unavailable` — недоступен, `expired` — истёк |
| `promised_until` | string (дата) или `null` | да | Дата окончания действия обещанного платежа |
| `reject_reason` | string | нет | Причина, по которой обещанный платёж недоступен (показывается абоненту) |

#### Currency

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | integer | Идентификатор валюты |
| `code` | string | Код валюты, например `RUB` |
| `title` | string | Отображаемое название |

### PATCH subscriber/accounts/{accountId}/

Действия над лицевым счётом. Действие задаётся параметром `action` в теле запроса.

**Добровольная блокировка счёта (`action=suspend`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `suspend` |
| `date_start` | string (`Y-m-d H:i:s`) | да | Начало блокировки |
| `date_end` | string (`Y-m-d H:i:s`) | да | Окончание блокировки |

**Снятие блокировки (`action=unsuspend`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `unsuspend` |

**Обещанный платёж (`action=promised-pay`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `promised-pay` |

Сумму и срок обещанного платежа определяет биллинг.

**Успешный ответ:**

```json
{ "success": true, "data": null, "message": "success", "code": 200 }
```

Для `suspend` и `unsuspend` МЛК проверяет `code == 200`: любой другой код считается ошибкой.

### subscriber/accounts/{accountId}/transactions

История операций по лицевому счёту за период.

> МЛК вызывает этот метод как `GET` для всех биллингов, кроме инсталляции Starlink, где используется `POST`.
> Рекомендуется поддерживать оба метода.

**Параметры (тело запроса):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `start_date` | string (`Y-m-d H:i:s`) | да | Начало периода |
| `end_date` | string (`Y-m-d H:i:s`) | да | Конец периода |

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    {
      "date": "2026-10-01T10:15:00+0300",
      "debit": 1000,
      "type": "payment",
      "comment": "Пополнение счёта",
      "currency": { "id": 1, "code": "RUB", "title": "Руб" }
    },
    {
      "date": "2026-10-01T00:00:00+0300",
      "debit": -650,
      "comment": "Абонентская плата",
      "currency": { "id": 1, "code": "RUB", "title": "Руб" }
    }
  ],
  "message": "success",
  "code": 200
}
```

`data` — массив операций (без дополнительной вложенности):

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `date` | string (дата) | да | Дата операции |
| `debit` | number | нет | Сумма со знаком: **положительная — зачисление**, **отрицательная — списание** |
| `credit` | number | нет | Сумма зачисления (альтернатива положительному `debit`) |
| `comment` | string | нет | Описание операции |
| `type` | string | нет | Тип операции |
| `currency` | [Currency](#currency) | нет | Валюта |

---

## Оплата

Оплата в приложении выполняется во встроенном браузере (WebView): биллинг возвращает ссылку на платёжную
страницу, приложение её открывает и отслеживает переход на `success_redirect_url` или `failure_redirect_url`.

В методах оплаты вместо идентификатора счёта передаётся его **номер** — поле `number` из
[Account](#get-subscriberaccountsaccountid).

### GET subscriber/accounts/{accountNumber}/pay-link/

Ссылка на оплату.

**Query:**

| Параметр | Тип | Обяз. | Описание |
|----------|-----|-------|----------|
| `amount` | number | да | Сумма оплаты |

**Успешный ответ:**

```json
{
  "success": true,
  "data": {
    "pay_link": "https://pay.example.com/form?order=0f8e6b2c",
    "success_redirect_url": "https://billing.example.com/payment/success",
    "failure_redirect_url": "https://billing.example.com/payment/failure"
  },
  "message": "success",
  "code": 200
}
```

| Поле `data` | Тип | Обяз. | Описание |
|-------------|-----|-------|----------|
| `pay_link` | string | да | Адрес платёжной страницы |
| `success_redirect_url` | string | да | Адрес, на который платёжная страница перенаправляет после успешной оплаты |
| `failure_redirect_url` | string | да | Адрес, на который платёжная страница перенаправляет после неуспешной оплаты |

### GET subscriber/accounts/{accountNumber}/auto-payment-link/

Ссылка на страницу привязки карты для автоплатежа.

**Query:**

| Параметр | Тип | Обяз. | Описание |
|----------|-----|-------|----------|
| `amount` | number | да | Сумма автоплатежа |

**Успешный ответ** — в том же формате, что и у [`pay-link`](#get-subscriberaccountsaccountnumberpay-link),
ссылка передаётся в поле **`pay_link`**:

```json
{
  "success": true,
  "data": {
    "pay_link": "https://pay.example.com/bind-card?order=4a91c7d0",
    "success_redirect_url": "https://billing.example.com/payment/success",
    "failure_redirect_url": "https://billing.example.com/payment/failure"
  },
  "message": "success",
  "code": 200
}
```

### GET subscriber/accounts/{accountNumber}/auto-payment-off

Отключение автоплатежа. `data` передаётся в приложение без изменений.

**Успешный ответ:**

```json
{ "success": true, "data": "Auto-payment is turned off", "message": "success", "code": 200 }
```

---

## Услуги

### GET subscriber/accounts/{accountId}/services

Услуги на лицевом счёте вместе с подключёнными дополнительными услугами.

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    {
      "id": 28,
      "title": "Интернет 100 Мбит/с",
      "description": "",
      "type": "type_inet",
      "info": { "type": "speed", "title": "Скорость", "description": "100000 / 100000" },
      "pay_until": "2026-10-20T00:00:00+0300",
      "started_at": "2026-01-01T00:00:00+0300",
      "price": {
        "total": 650,
        "currency": { "id": 1, "code": "RUB", "title": "Руб" },
        "frequency": "month"
      },
      "state": { "id": 1, "code": "unlock", "title": "Услуга оказывается", "paused": false },
      "address": "г. Новосибирск, ул. Примерная, д. 1",
      "mutations": ["changeTariff", "suspend", "addServs"],
      "next": {
        "id": 64,
        "title": "Интернет 500 Мбит/с",
        "description": "",
        "info": { "type": "speed", "title": "Скорость", "description": "500000 / 500000" },
        "started_at": "2026-11-01T00:00:00+0300",
        "price": {
          "total": 900,
          "currency": { "id": 1, "code": "RUB", "title": "Руб" },
          "frequency": "month"
        },
        "state": { "id": 1, "code": "unlock", "title": "Услуга оказывается", "paused": false },
        "address": "г. Новосибирск, ул. Примерная, д. 1",
        "mutations": []
      },
      "additional_services": [
        {
          "id": 101,
          "title": "Статический IP-адрес",
          "description": "",
          "pay_until": "2026-10-20T00:00:00+0300",
          "ended_at": "2026-10-20T00:00:00+0300",
          "price": {
            "total": 150,
            "currency": { "id": 1, "code": "RUB", "title": "Руб" },
            "frequency": "month"
          },
          "state": { "id": 1, "code": "unlock", "title": "Услуга оказывается", "paused": false },
          "mutations": []
        }
      ]
    }
  ],
  "message": "success",
  "code": 200
}
```

**Service:**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `id` | integer или string | да | Идентификатор услуги, используется как `{serviceId}` |
| `title` | string | да | Название |
| `description` | string | да | Описание (может быть пустой строкой) |
| `pay_until` | string (дата) | да | Дата оплаты услуги |
| `price` | [Price](#price) | да | Стоимость |
| `state` | [State](#state) | да | Состояние |
| `mutations` | string[] | да | Доступные абоненту действия (см. ниже), может быть пустым массивом |
| `type` | string | нет | Тип услуги, по нему выбирается иконка (см. ниже) |
| `info` | object или массив из одного object | нет | Скорость: `{ "type": "speed", "title": "Скорость", "description": "<входящая> / <исходящая>" }` |
| `started_at` | string (дата) | нет | Дата начала действия |
| `address` | string | нет | Адрес оказания услуги |
| `object_id` | string | нет | Идентификатор объекта подключения |
| `next` | Service или `null` | нет | Запланированная услуга, например тариф со следующего месяца. Если передаётся, обязательны поля `id`, `title`, `description`, `info`, `started_at`, `price`, `state`, `address`, `mutations` |
| `additional_services` | Service[] | нет | Подключённые дополнительные услуги. Для каждой обязательны `id`, `title`, `description`, `pay_until`, `ended_at`, `price`, `state`, `mutations` |

**Значения `mutations`:**

| Значение | Действие |
|----------|----------|
| `changeTariff` | Доступна смена тарифа |
| `suspend` | Доступна добровольная блокировка услуги |
| `addServs` | Доступно подключение и отключение дополнительных услуг |

**Значения `type`:** `type_inet` — интернет, `type_tv` — телевидение, `type_iptv` — IP-телевидение,
`type_phone` — телефония, `type_cctv` — видеонаблюдение, `type_intercom` — домофон, `type_discount` — скидка,
`type_loan` — рассрочка, `type_serv` — сервисные услуги, `type_misc` — прочее.

#### Price

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `total` | number | да | Стоимость |
| `frequency` | string | да | Период списания, например `month` или `30 day` |
| `currency` | [Currency](#currency) | нет | Валюта |

#### State

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `id` | integer | да | Идентификатор состояния |
| `code` | string | да | Код состояния, например `lock` / `unlock` |
| `title` | string | да | Отображаемое название |
| `paused` | boolean | да | `true` — услуга приостановлена или заблокирована |

### GET subscriber/accounts/{accountId}/services/{serviceId}/tariffs

Тарифы, на которые можно перейти с услуги.

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    {
      "id": "64",
      "title": "Интернет 500 Мбит/с",
      "started_at": "2026-11-01T00:00:00+0300",
      "price": {
        "total": 900,
        "currency": { "id": 1, "code": "RUB", "title": "Руб" },
        "frequency": "month"
      }
    }
  ],
  "message": "success",
  "code": 200
}
```

| Поле элемента | Тип | Обяз. | Описание |
|---------------|-----|-------|----------|
| `id` | string или integer | да | Идентификатор тарифа |
| `title` | string | да | Название |
| `started_at` | string (дата) | да | Дата, с которой начнёт действовать тариф |
| `price` | [Price](#price) | да | Стоимость |

### PATCH subscriber/accounts/{accountId}/services/{serviceId}

Действия над услугой. Действие задаётся параметром `action` в теле запроса.

> Обратите внимание на завершающий `/`: смена тарифа и блокировка отправляются на
> `.../services/{serviceId}/`, а разблокировка — на `.../services/{serviceId}` (без `/`).
> Биллинг должен принимать оба варианта.

**Смена тарифа (`action=change-tariff`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `change-tariff` |
| `tariffId` | string | да | Идентификатор тарифа из [`tariffs`](#get-subscriberaccountsaccountidservicesserviceidtariffs) |

**Блокировка услуги (`action=suspend`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `suspend` |

**Разблокировка услуги (`action=unsuspend`):**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `unsuspend` |

**Успешный ответ:**

```json
{ "success": true, "data": null, "message": "success", "code": 200 }
```

### PATCH subscriber/accounts/{accountId}/services/{serviceId}/suspension-service

Услуга приостановки, связанная с основной услугой: её название и стоимость показываются абоненту перед
блокировкой. Тело запроса пустое.

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    {
      "id": 501,
      "title": "Добровольная блокировка",
      "price": {
        "total": 100,
        "currency": { "id": 1, "code": "RUB", "title": "Руб" },
        "frequency": "month"
      }
    }
  ],
  "message": "success",
  "code": 200
}
```

`data` — массив, МЛК использует **первый** элемент: обязательны `id`, `title`, `price.total`, `price.frequency`.

### GET subscriber/accounts/{accountId}/services/{serviceId}/additional-services

Дополнительные услуги, которые можно подключить к услуге.

**Успешный ответ:**

```json
{
  "success": true,
  "data": [
    {
      "id": 15741,
      "title": "Статический IP-адрес",
      "description": "Интернет с публичным IP",
      "pay_until": "2026-10-20T00:00:00+0300",
      "ended_at": "2026-10-20T00:00:00+0300",
      "price": {
        "total": 150,
        "currency": { "id": 1, "code": "RUB", "title": "Руб" },
        "frequency": "30 day"
      },
      "state": { "id": 2, "code": "lock", "title": "Не подключена", "paused": true },
      "mutations": []
    }
  ],
  "message": "success",
  "code": 200
}
```

| Поле элемента | Тип | Обяз. | Описание |
|---------------|-----|-------|----------|
| `id` | integer | да | Идентификатор доп. услуги, используется как `{additionalServiceId}` |
| `title` | string | да | Название |
| `description` | string | нет | Описание |
| `pay_until` | string (дата) | нет | Дата оплаты |
| `ended_at` | string (дата) | нет | Дата окончания |
| `price` | [Price](#price) | нет | Стоимость |
| `state` | [State](#state) | нет | Состояние: `paused: false` — подключена |
| `mutations` | string[] | нет | Доступные действия |

### PATCH subscriber/accounts/{accountId}/services/{serviceId}/additional-services/{additionalServiceId}/

Подключение или отключение дополнительной услуги.

**Тело запроса:**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `1` — подключить, `0` — отключить |

**Успешный ответ:**

```json
{ "success": true, "data": null, "message": "success", "code": 200 }
```

МЛК проверяет `code == 200`: любой другой код считается ошибкой.

---

## Справочная информация

### GET resources/promised-pay-terms

Условия обещанного платежа. `data` — HTML-фрагмент, который приложение показывает абоненту как есть.

**Успешный ответ:**

```json
{
  "success": true,
  "data": "<div><p>Обещанный платёж предоставляется на 3 дня на сумму рекомендуемого платежа.</p><p>Воспользоваться услугой можно раз в 25 дней.</p></div>",
  "message": "success",
  "code": 200
}
```

---

## Без авторизации абонента

Эти методы вызываются без заголовка `Authorization`.

### GET restore-password/find-mobile/{phone}

Поиск абонента по номеру телефона. Первый шаг восстановления доступа.

| Параметр пути | Описание |
|---------------|----------|
| `phone` | Номер телефона в формате `79XXXXXXXXX` |

**Успешный ответ:** `data` — **непустой массив** найденных лицевых счетов абонента
(элементы в формате [Account](#get-subscriberaccountsaccountid), МЛК использует только их количество).

```json
{
  "success": true,
  "data": [
    { "id": "100001", "number": "100001" }
  ],
  "message": "success",
  "code": 200
}
```

Если абонент не найден, верните HTTP 404:

```json
{ "success": false, "data": "fail", "message": "Not a subscriber", "code": 404 }
```

### POST restore-password/

Получение реквизитов для входа по номеру телефона. МЛК сам отправляет абоненту SMS с логином и паролем
из ответа и сам ограничивает частоту запросов (одна отправка в 60 секунд).

**Тело запроса:**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `action` | string | да | `phone` |
| `phone_number` | string | да | Номер телефона в формате `79XXXXXXXXX` |

**Успешный ответ:**

```json
{
  "success": true,
  "data": {
    "login": "100001",
    "password": "<пароль абонента>"
  },
  "message": "success",
  "code": 200
}
```

| Поле `data` | Тип | Обяз. | Описание |
|-------------|-----|-------|----------|
| `login` | string | да | Логин для входа |
| `password` | string | да | Пароль для входа (новый или текущий — на усмотрение биллинга) |

Если абонент не найден, верните HTTP 404:

```json
{ "success": false, "data": "fail", "message": "Not a subscriber", "code": 404 }
```

### POST support/send-email

Обращение абонента в техподдержку. Доставку обращения (email, тикет-система) выполняет биллинг.

**Тело запроса:**

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| `account` | string | нет | Номер лицевого счёта |
| `phone_number` | string | да | Контактный телефон |
| `email` | string | нет | Контактный email |
| `message` | string | да | Текст обращения |

**Успешный ответ:**

```json
{ "success": true, "data": true, "message": "success", "code": 200 }
```
