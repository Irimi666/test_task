```mermaid
graph TD
    A[Внешний сервис доставки] -->|Статус заказа|B[API Gateway]
    B -->|Запрос статуса|C[Сервис обработки статусов]
    C -->|Обновление статуса|D[База данных заказов]
    C -->|Уведомление|E[Сервис отправки пушей]
    E -->|FCM|F[Android-устройство]
    E -->|APNs|G[iOS-устройство]
    C -->|Логи|H[Сервис логирования и мониторинга]
  

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#fff3e0
    style D fill:#e8f5e8
    style E fill:#fff9c4
    style F fill:#b2dfdb
    style G fill:#b2dfdb
    style H fill:#ffecb3
```