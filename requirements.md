# Описание системы:

Онлайн-маркетплейс товаров. Необходимо реализовать отправку пуш-уведомлений в мобильное приложение об изменении статуса заказа. Данные об изменении статуса получать из API внешнего сервиса.

### Ключевые требования системы:

1. Получение статуса заказа из API внешнего сервиса.
2. Формирование контента уведомления в соответствии с полученными данными.
3. Отправка push-уведомления.
4. Фиксация результатов в БД. 

### Границы требований процесса:

* **Начало**: получение данных об изменении статуса заказа из API внешнего сервиса.

* **Конец**: фиксация финального статуса уведомления (`DELIVERED`, `CANCELLED` или `FAILED`).

* **Участники**:

    * внешний сервис доставки;

    * система маркетплейса;

    * мобильное приложение пользователя.



## Функциональные требования

Функциональные требования описывают основные функции системы, необходимые для реализации push-уведомлений в мобильное приложение об изменении статуса заказа

1. Получение данных о статусе заказа
2. Анализ изменений статуса
3. Формирование контента уведомления
4. Отправка push-уведомления
5. Обработка результатов отправки
6. Управление статусами уведомлений

<details> <summary>Подробности</summary>

1. **Получение данных о статусе заказа из внешнего API**

    * Автоматический опрос API внешнего сервиса доставки с настраиваемым интервалом (<mark>не даны точные сведения о сервисе и его ограничениях</mark>)

    * Запрос статуса по конкретному заказу через эндпоинт: `GET /orders/status/{order_id}`

    * Обработка JSON‑ответов с обязательными полями:

        * `order_id` — идентификатор заказа

        * `status` — текущий статус заказа

        * `timestamp` — время изменения статуса

        * `details` — дополнительные детали (перевозчик, трек‑номер и т.д.)
        * Обработка ошибок API (4xx, 5xx) с последующим завершением процесса

2. **Анализ изменений статуса**
   
    * Сравнение нового статуса с предыдущим (хранится в БД маркетплейса)

    * Фиксация факта изменения статуса для инициирования отправки уведомления

    * Проверка актуальности заказа (не отменён, не завершён)

3. **Формирование контента уведомления**
    
    * Подбор шаблона текста под конкретный статус заказа

    * Подстановка динамических данных в шаблон (например):

        * номер заказа

        * текущий статус

        * детали доставки из поля details

        * расчётное время доставки

    * Генерация deep link для перехода в раздел заказа в мобильном приложении

    * Формирование payload для push‑сервисов (FCM/APNs) со структурой:
    ```json
    {
    "title": "Заказ в пути",
    "body": "Ваш заказ № ORD-12345 передан курьеру",
    "deep_link": "app://orders/ORD-12345"
    }
    ```

4. **Отправка push‑уведомлений**
    
    * Интеграция с внешними push‑сервисами:

        * FCM (Firebase Cloud Messaging) для Android

        * APNs (Apple Push Notification Service) для iOS

5. **Обработка результатов отправки**

    * Проверка HTTP‑статуса от push‑сервиса:

        * 200/201 — успех (DELIVERED)

        * 4xx/5xx — ошибка

    * Логирование результатов отправки:

        * успех: фиксация `delivered_at`;

        * ошибка: запись `error_code`

    
6. **Управление статусами уведомлений**

    * Поддержка статусов:

        `CREATED` — уведомление создано, отправка не начата

        `SENDING` — идёт отправка в push‑сервис

        `DELIVERED` — push‑сервис подтвердил приём

        `FAILED` — попытка отправки завершилась ошибкой

        `CANCELLED` — отправка отменена до начала

    * Автоматическое обновление статусов на каждом этапе процесса

</details>   

## Нефункциональные требования 

Основные требования системы определяют характеристики системы, такие как производительность и время отклика

1. Производительность
2. Надежность
3. Отказоустойчивость
4. Логирование

<details> 

<summary>Подробности</summary>

1. **Производительность**

    * время обработки одного уведомления ≤ N секунд

    * пиковая нагрузка: до N уведомлений в минуту

    * задержка доставки ≤ N минут / секунд

2. **Надежность**
    
    * доступность сервиса: N дней в месяц / N% времени

    * устойчивость к сбоям внешнего API

3. **Отказоустойчивость**

    * автоматическое восстановление после сбоев

    * резервное копирование логов и данных о статусах

4. **Безопасность**

    * шифрование данных при передаче (HTTPS)

    * защита токенов устройств (не хранить в открытом виде)

    * аутентификация запросов к API внешнего сервиса

5. **Логирование**

    * запись всех этапов процесса в лог (`timestamp`, `order_id`, `status`, `result`, `error_code`)

    * хранение логов N времени

    * доступ к данным о статусах заказов 

</details>


## UML диаграмма состояния

<!--[if IE]><meta http-equiv="X-UA-Compatible" content="IE=5,IE=9" ><![endif]-->
<!DOCTYPE html>
<html>
<head>
<title>Диаграмма без названия</title>
<meta charset="utf-8"/>
</head>
<body><div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#000000&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;dark-mode&quot;:&quot;light&quot;,&quot;toolbar&quot;:&quot;zoom layers tags lightbox&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; agent=\&quot;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 YaBrowser/25.12.0.0 Safari/537.36\&quot; version=\&quot;29.3.8\&quot;&gt;\n  &lt;diagram name=\&quot;Страница-1\&quot; id=\&quot;4miKt8JBv4ejTgPMT3zc\&quot;&gt;\n    &lt;mxGraphModel dx=\&quot;1042\&quot; dy=\&quot;568\&quot; grid=\&quot;1\&quot; gridSize=\&quot;10\&quot; guides=\&quot;1\&quot; tooltips=\&quot;1\&quot; connect=\&quot;1\&quot; arrows=\&quot;1\&quot; fold=\&quot;1\&quot; page=\&quot;1\&quot; pageScale=\&quot;1\&quot; pageWidth=\&quot;827\&quot; pageHeight=\&quot;1169\&quot; math=\&quot;0\&quot; shadow=\&quot;0\&quot;&gt;\n      &lt;root&gt;\n        &lt;mxCell id=\&quot;0\&quot; /&gt;\n        &lt;mxCell id=\&quot;1\&quot; parent=\&quot;0\&quot; /&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-1\&quot; parent=\&quot;1\&quot; style=\&quot;ellipse;html=1;shape=startState;fillColor=#000000;strokeColor=#ff0000;\&quot; value=\&quot;\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;30\&quot; x=\&quot;399\&quot; y=\&quot;30\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-2\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-1\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;html=1;verticalAlign=bottom;endArrow=open;endSize=8;strokeColor=#ff0000;rounded=0;\&quot; value=\&quot;\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint x=\&quot;414\&quot; y=\&quot;120\&quot; as=\&quot;targetPoint\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-3\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-2\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Создание уведомления\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.0667\&quot; y=\&quot;-2\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint x=\&quot;2\&quot; y=\&quot;-8\&quot; as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-9\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-4\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;entryX=0.5;entryY=0;entryDx=0;entryDy=0;exitX=0.25;exitY=1;exitDx=0;exitDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-5\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint x=\&quot;360\&quot; y=\&quot;130\&quot; as=\&quot;sourcePoint\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-23\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-9\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Начать отправку\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.193\&quot; y=\&quot;-2\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint x=\&quot;-10\&quot; as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-15\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-4\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=1;exitY=1;exitDx=0;exitDy=0;entryX=0.25;entryY=0;entryDx=0;entryDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-8\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-22\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-15\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Отмена до отправки\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;0.0042\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-4\&quot; parent=\&quot;1\&quot; style=\&quot;html=1;align=center;verticalAlign=top;rounded=1;absoluteArcSize=1;arcSize=10;dashed=0;whiteSpace=wrap;\&quot; value=\&quot;CREATED\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;81\&quot; x=\&quot;379\&quot; y=\&quot;120\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-10\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-5\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=0.25;exitY=1;exitDx=0;exitDy=0;entryX=0.5;entryY=0;entryDx=0;entryDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-6\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-25\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-10\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Код 200 от FCM/APNs\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; y=\&quot;3\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-5\&quot; parent=\&quot;1\&quot; style=\&quot;html=1;align=center;verticalAlign=top;rounded=1;absoluteArcSize=1;arcSize=10;dashed=0;whiteSpace=wrap;\&quot; value=\&quot;SENDING\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;81\&quot; x=\&quot;300\&quot; y=\&quot;200\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-21\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-6\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=0.5;exitY=1;exitDx=0;exitDy=0;entryX=0;entryY=0.5;entryDx=0;entryDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-18\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-26\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-21\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Завершение\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.0765\&quot; y=\&quot;21\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-6\&quot; parent=\&quot;1\&quot; style=\&quot;html=1;align=center;verticalAlign=top;rounded=1;absoluteArcSize=1;arcSize=10;dashed=0;whiteSpace=wrap;\&quot; value=\&quot;DELIVERED\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;81\&quot; x=\&quot;250\&quot; y=\&quot;280\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-20\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-7\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=0.5;exitY=1;exitDx=0;exitDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-18\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-27\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-20\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Завершение\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.084\&quot; y=\&quot;2\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-7\&quot; parent=\&quot;1\&quot; style=\&quot;html=1;align=center;verticalAlign=top;rounded=1;absoluteArcSize=1;arcSize=10;dashed=0;whiteSpace=wrap;\&quot; value=\&quot;FAILED\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;81\&quot; x=\&quot;409\&quot; y=\&quot;250\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-19\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-8\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=0.5;exitY=1;exitDx=0;exitDy=0;entryX=1;entryY=0.5;entryDx=0;entryDy=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-18\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-28\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-19\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Завершение\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.3631\&quot; y=\&quot;-3\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint x=\&quot;3\&quot; y=\&quot;-30\&quot; as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-8\&quot; parent=\&quot;1\&quot; style=\&quot;html=1;align=center;verticalAlign=top;rounded=1;absoluteArcSize=1;arcSize=10;dashed=0;whiteSpace=wrap;\&quot; value=\&quot;CANCELLED&amp;lt;div&amp;gt;&amp;lt;br&amp;gt;&amp;lt;/div&amp;gt;\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;81\&quot; x=\&quot;520\&quot; y=\&quot;190\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-16\&quot; edge=\&quot;1\&quot; parent=\&quot;1\&quot; source=\&quot;g4ZNIi5u_4i4k3F48QBV-5\&quot; style=\&quot;edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;exitX=1;exitY=0.5;exitDx=0;exitDy=0;entryX=0.605;entryY=0.027;entryDx=0;entryDy=0;entryPerimeter=0;curved=1;\&quot; target=\&quot;g4ZNIi5u_4i4k3F48QBV-7\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-24\&quot; connectable=\&quot;0\&quot; parent=\&quot;g4ZNIi5u_4i4k3F48QBV-16\&quot; style=\&quot;edgeLabel;html=1;align=center;verticalAlign=middle;resizable=0;points=[];\&quot; value=\&quot;Ошибка\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry relative=\&quot;1\&quot; x=\&quot;-0.1596\&quot; y=\&quot;-3\&quot; as=\&quot;geometry\&quot;&gt;\n            &lt;mxPoint as=\&quot;offset\&quot; /&gt;\n          &lt;/mxGeometry&gt;\n        &lt;/mxCell&gt;\n        &lt;mxCell id=\&quot;g4ZNIi5u_4i4k3F48QBV-18\&quot; parent=\&quot;1\&quot; style=\&quot;ellipse;html=1;shape=endState;fillColor=#000000;strokeColor=#ff0000;\&quot; value=\&quot;\&quot; vertex=\&quot;1\&quot;&gt;\n          &lt;mxGeometry height=\&quot;30\&quot; width=\&quot;30\&quot; x=\&quot;399\&quot; y=\&quot;340\&quot; as=\&quot;geometry\&quot; /&gt;\n        &lt;/mxCell&gt;\n      &lt;/root&gt;\n    &lt;/mxGraphModel&gt;\n  &lt;/diagram&gt;\n&lt;/mxfile&gt;\n&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>
</body>
</html>

```

