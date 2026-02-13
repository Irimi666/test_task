# Структура взаимодействия API

Описание взаимодействия API онлайн‑маркетплейса с внешним сервисом доставки. Цель интеграции — получение актуальных статусов заказов для отправки push‑уведомлений пользователям.

**1. Базовый URL стороннего сервиса**:
```
https://api.delivery-service.com/v1
```

**2. Аутентификация**:

Для доступа к API используется Bearer‑токен в заголовке запроса:
```
Authorization: Bearer <your_access_token>
```
Токен выдаётся после регистрации приложения в личном кабинете внешнего сервиса.

**3. Взаимодействие с API внешнего сервиса:**

* **3.1. Сценарий 1 (основной) : Webhook**

    * Зарегистрировать вебхук, сообщить сервису доставки URL вашего эндпоинта и тип событий, которые нужно отслеживать

        Пример:

        ```http
        POST /webhooks HTTP/1.1
        Host: api.delivery-service.ru
        Authorization: Bearer your_access_token
        Content-Type: application/json
        ```
        ```json
        {
        "callback_url": "https://marketplace.ru/api/webhooks/order-statuses",
        "event_type": "order.status.updated",
        "secret": "secret_token_1234",
        "active": true
        }
        ```
        `event_type` - событие, которое будет отслеживаться через вебхук

        `secret` - подпись (секретный ключ маркетплейса)

        `callback_url` - адрес эндпоинта маркетплейса куда будут отправляться запросы от сервиса доставки (POST)

        `active`- статус подписки на сервис (активна / не активна)

    * Когда происходит событие (изменение статуса заказа), сервис формирует тело JSON-payload уведомления и шифрует подпись 

    * Сервис отправляет POST-запрос на эндпоинт `callback_url` включая заголовок и подпись

* **3.2. Сценарий 2 (резервный) : Получение списка обновлений статусов за период** 

    * При запуске сервиса или по расписанию (например, каждые 5 минут) выполняется запрос к `/delivery/order-statuses` для получения всех обновлений за указанный период с обязательными`from_date` и `to_date`, пример:

    ```http
    GET /delivery/order-statuses?from_date=2026-02-12T00:00:00Z&to_date=2026-02-13T00:00:00Z HTTP/1.1
    Host: api.delivery-service.ru
    Authorization: Bearer your_access_token
    ```

**4. Получение уведомления от стороннего сервиса**

* На стороне маркетплейса расшифровывается и проверяется подлинность подписи используя локально сохраненный `secret`

* Если подписи совпадают - данные можно обрабатывать

* Если нет - запрос отклоняется

* Пример запроса от стороннего сервиса

    ```http
    POST https://api.marketplace.ru/webhooks/order-statuses
    Content-Type: application/json
    ```

    ```json
    {
    "event_id": "EVT-78901",
    "order_id": "ORD-12345",
    "status": "CREATED",
    "timestamp": "2026-02-12T14:30:00Z",
    "details": {
        "tracking_number": "TRK-987654321",
        "estimated_delivery": "2026-02-12T18:00:00Z"
    },
    "signature": "123456hash"
    }
    ```
* Обязательные поля

    `event_id`- уникальный идентификатор события

    `order_id`- уникальный идентификатор заказа

    `status` - статус заказа

    `timestamp` - время изменения статуса

    `signature` - подпись

    `details` - детали доставки

**5. Логика обработки payload (полезной нагрузки) от сервиса доставки**

* Проверка подлинности по подписи `signature` и возврат `HTTP 200 OK`, если успешно

* Валидация обязательных данных `order_id`, `status`

* Поиск заказа `order_id` в БД маркетплейса

* Сравнение статусов: если новый `status` или `details` отличается от текущего - продолжить обработку

* Логирование

**6. Формирование payload для push-уведомления и отправка**

* Формирование контента уведомления в зависимости от статуса

* Определение платформы устройста (Android / IOS)

**7. Отправка уведомления**

* Отправка payload в мобильное приложение через сервис уведомлений
    
    Пример для Android Firebase Cloud Messaging (FCM):
    ```bash
    POST https://fcm.googleapis.com/fcm/send HTTP/1.1
    Authorization: key=YOUR_SERVER_KEY # серверный ключ FCM
    Content-Type: application/json
    ```
    ```json
        {
        "to": "USER_FCM_TOKEN", //токен устройства
        "notification": {
            "title": "Курьер забрал заказ",
            "body": "Заказ № 12345 передан курьеру"
            },
        "data": {
            "order_id": "ORD-12345",
            "status": "OUT_FOR_DELIVERY"
            },
        "priority": "high",
        "time_to_live": 3600 // сохранение уведомления на 1 час если устройство оффлайн
        }
    ```
    Пример для Apple Push Notification Service (APNs) для iOS:
    
    ```bash
    POST https://api.push.apple.com:443/3/device/{device_token} HTTP/2 # токен устройства
    authorization: bearer {JWT} # токен для аутентификации (генерируется на сервере)
    apns-topic: com.example.myapp
    apns-push-type: alert
    apns-expiration: 1 # сохранение уведомления если устройство оффлайн
    apns-priority: 10
    ```
    ```json
    {
    "aps": {
        "alert": {
        "title": "Доставлен в пункт выдачи",
        "body": "Заказ № 12345 ждёт вас в пункте выдачи"
        },
        "badge": 1,
        "sound": "default",
        "category": "order_notification"
        },
    "order_id": "ORD-12345",
    "status": "PICKUP_READY"
    }
    ```
**8. Получение ответа**

* `HTTP 200 OK` от APNs/FCM - подтверждение доставки уведомления на устройство

* Отсутствие ответ - если устройство оффлайн, сервис push-уведомлений сохраняет его у себя на заданное время

* Ошибка 

**9. Логирование ответа**

* Запись ответа в лог и завершение процесса

## BPMN-диаграмма процесса:

![BPMN: Бизнес процесс отправки уведомлений](/diagrams/business-process-pic.png)

[.bpmn-файл](/diagrams/business-process.bpmn)

## UML-диаграмма статусов:

![uml-state-diagram](/diagrams/condition-status-diagram.png)

*  Создан — заказ создан, но не оплачен

* Оплачен — заказ оплачен, готов к обработке

* Подтвержден - заказ обработан, готов к передаче в доставку

* Передан в доставку — заказ передан в службу доставки

* В пути — доставка началась, заказ движется к получателю

* Доставлен — заказ успешно доставлен пользователю

* Отменён — заказ отменён пользователем или системой

## UML-диаграмма состояний:

![UML: статусы уведомлений](/diagrams/notification-status-diagram.png)

[.html-файл](diagrams/status-diagram.html)

* CREATED - уведомление получено -> начать отправку

* SENDING - отправка уведомления -> успешно доставлено / не доставлено

* DELIVERED - уведомление доставлено -> завершение процесса

* FAILED - уведомление не доставлено -> завершение процесса

* CANCELLED - отправка отменена -> завершение процесса


## Диаграмма компонентов:

![components-diagram](/diagrams/components-diagram.svg)

## Диаграмма потока данных:

![data-flow-diagram](/diagrams/data-flow-diagram.png)

