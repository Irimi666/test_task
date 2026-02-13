# Order Status Update and Notification Flow

```mermaid
sequenceDiagram

    title Потоки данных
    autoNumber nested

    // Actor definitions with icons and colors for related services
    Сервис доставки [icon: globe, color: green]
    API Gateway [icon: aws-api-gateway, color: blue]
    Система маркетплейса [icon: server, color: blue]
    Database [icon: database, color: orange]
    Push Notification Service [icon: send, color: purple]
    FCM (Android) [icon: android, color: purple]
    APNs (iOS) [icon: apple, color: purple]
    Logging Service [icon: file-text, color: gray]
    // 1. External service sends order status via REST API or webhook
    Сервис доставки > API Gateway:  Отправка статуса [color: green]

    // 2. API Gateway authenticates and authorizes the request
    activate API Gateway
    API Gateway > API Gateway: Аутентификация

    // 3. API Gateway forwards request to Status Service
    API Gateway > Система маркетплейса: Отправка информации

    // 4. Status Service updates status in DB (transactional)
    activate Система маркетплейса
    Система маркетплейса > Database: Сравнение статуса
    Система маркетплейса > Database: Обновление статуса 
    Система маркетплейса > Database: Формирование контента уведомления

    // 5. Status Service forms and sends notification to Push Notification Service
    Система маркетплейса > Push Notification Service: Отправка уведомления

    // 6. Push Notification Service sends push to Android and iOS in parallel
par [label: Send push notifications, icon: send] {
    Push Notification Service > FCM (Android): Отправка push-уведомления
    Push Notification Service > APNs (iOS): Отправка push-уведомления
    }

    // 7. Confirm delivery and handle failures
    alt [label: Delivery confirmation, icon: check] {
    FCM (Android) > Push Notification Service: Уведомление доставлено
    APNs (iOS) > Push Notification Service: Уведомление доставлено
    }
    else [label: Delivery failed, icon: x] {
    FCM (Android) > Push Notification Service: Уведомление не доставлено
    APNs (iOS) > Push Notification Service: Уведомление не доставлено
    Push Notification Service > Система маркетплейса: Логирование
    }

    // 8. Logging and metrics collection (in parallel)
    par [label: Logging and metrics, icon: bar-chart] {
    API Gateway > Logging Service: Лог API запроса
    Система маркетплейса > Logging Service: Лог обновления статуса
    Push Notification Service > Logging Service: Лог события

    }

    deactivate Система маркетплейса
    deactivate API Gateway
```