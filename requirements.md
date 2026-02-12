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


## BPMN диаграмма процесса

```bpmn
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:dc="http://www.omg.org/spec/DD/20100524/DC" xmlns:di="http://www.omg.org/spec/DD/20100524/DI" id="Definitions_02l9qnq" targetNamespace="http://bpmn.io/schema/bpmn" exporter="bpmn-js (https://demo.bpmn.io)" exporterVersion="18.12.0">
  <bpmn:collaboration id="Collaboration_16tndhq">
    <bpmn:participant id="Participant_163frdf" name="Отправка пуш-уведомлений о доставке" processRef="Process_1v5ihpy" />
  </bpmn:collaboration>
  <bpmn:process id="Process_1v5ihpy">
    <bpmn:laneSet id="LaneSet_0uz0cos">
      <bpmn:lane id="Lane_0jmicoq" name="Система маркетплейса">
        <bpmn:flowNodeRef>Activity_0otjrlt</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_0rjjaof</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Gateway_0un5lsg</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_1196mqc</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_1hi8iqt</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Event_1p80eug</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Event_1ajchaw</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_000hvqe</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_1spgumw</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Event_0lfeczd</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_0ykyrme</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_0rul86p</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Gateway_0tei6kf</bpmn:flowNodeRef>
      </bpmn:lane>
      <bpmn:lane id="Lane_0gajj4m" name="Мобильное приложение пользователя">
        <bpmn:flowNodeRef>Event_1ac33rz</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Activity_120u8o2</bpmn:flowNodeRef>
        <bpmn:flowNodeRef>Event_0g1wmhs</bpmn:flowNodeRef>
      </bpmn:lane>
      <bpmn:lane id="Lane_0qzn06w" name="Внешний сервис доставки">
        <bpmn:flowNodeRef>Event_0efb55i</bpmn:flowNodeRef>
      </bpmn:lane>
    </bpmn:laneSet>
    <bpmn:task id="Activity_0otjrlt" name="Получить статус через API">
      <bpmn:incoming>Flow_0jqs1bd</bpmn:incoming>
      <bpmn:outgoing>Flow_1r5aog6</bpmn:outgoing>
    </bpmn:task>
    <bpmn:intermediateCatchEvent id="Event_1ac33rz" name="Пуш-уведомление получено устройством">
      <bpmn:outgoing>Flow_0iyyna0</bpmn:outgoing>
      <bpmn:messageEventDefinition id="MessageEventDefinition_0wzwtpd" />
    </bpmn:intermediateCatchEvent>
    <bpmn:task id="Activity_120u8o2" name="Отобразить уведомление">
      <bpmn:incoming>Flow_0iyyna0</bpmn:incoming>
      <bpmn:outgoing>Flow_0h1trxo</bpmn:outgoing>
    </bpmn:task>
    <bpmn:sendTask id="Activity_0rjjaof" name="Отправить уведомление">
      <bpmn:incoming>Flow_1811y2r</bpmn:incoming>
      <bpmn:outgoing>Flow_0ect44b</bpmn:outgoing>
    </bpmn:sendTask>
    <bpmn:exclusiveGateway id="Gateway_0un5lsg" name="Отправка успешна?">
      <bpmn:incoming>Flow_0ect44b</bpmn:incoming>
      <bpmn:outgoing>Flow_048zvso</bpmn:outgoing>
      <bpmn:outgoing>Flow_0006i37</bpmn:outgoing>
    </bpmn:exclusiveGateway>
    <bpmn:task id="Activity_1196mqc" name="Записать ошибку в лог">
      <bpmn:incoming>Flow_0006i37</bpmn:incoming>
      <bpmn:outgoing>Flow_1bvxqpv</bpmn:outgoing>
    </bpmn:task>
    <bpmn:task id="Activity_1hi8iqt" name="Записать успех в лог">
      <bpmn:incoming>Flow_048zvso</bpmn:incoming>
      <bpmn:outgoing>Flow_1sl214g</bpmn:outgoing>
    </bpmn:task>
    <bpmn:endEvent id="Event_1p80eug" name="Уведомление доставлено">
      <bpmn:incoming>Flow_1sl214g</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:endEvent id="Event_1ajchaw" name="Процесс завершен с ошибкой">
      <bpmn:incoming>Flow_1bvxqpv</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:endEvent id="Event_0g1wmhs" name="Уведомление доставлено">
      <bpmn:incoming>Flow_0h1trxo</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:task id="Activity_000hvqe" name="Сформировать уведомление">
      <bpmn:incoming>Flow_1s2e09b</bpmn:incoming>
      <bpmn:outgoing>Flow_1811y2r</bpmn:outgoing>
    </bpmn:task>
    <bpmn:task id="Activity_1spgumw" name="Записать в лог: не требуется">
      <bpmn:incoming>Flow_0qxgerl</bpmn:incoming>
      <bpmn:outgoing>Flow_1uzqo6r</bpmn:outgoing>
    </bpmn:task>
    <bpmn:endEvent id="Event_0lfeczd" name="Уведомление не требовалось">
      <bpmn:incoming>Flow_1uzqo6r</bpmn:incoming>
    </bpmn:endEvent>
    <bpmn:sequenceFlow id="Flow_0jqs1bd" sourceRef="Event_0efb55i" targetRef="Activity_0otjrlt" />
    <bpmn:sequenceFlow id="Flow_0h00d8r" sourceRef="Activity_0ykyrme" targetRef="Gateway_0tei6kf" />
    <bpmn:sequenceFlow id="Flow_0iyyna0" sourceRef="Event_1ac33rz" targetRef="Activity_120u8o2" />
    <bpmn:sequenceFlow id="Flow_0h1trxo" sourceRef="Activity_120u8o2" targetRef="Event_0g1wmhs" />
    <bpmn:sequenceFlow id="Flow_1s2e09b" name="Да" sourceRef="Gateway_0tei6kf" targetRef="Activity_000hvqe" />
    <bpmn:sequenceFlow id="Flow_0qxgerl" name="Нет" sourceRef="Gateway_0tei6kf" targetRef="Activity_1spgumw" />
    <bpmn:sequenceFlow id="Flow_1811y2r" sourceRef="Activity_000hvqe" targetRef="Activity_0rjjaof" />
    <bpmn:sequenceFlow id="Flow_0ect44b" sourceRef="Activity_0rjjaof" targetRef="Gateway_0un5lsg" />
    <bpmn:sequenceFlow id="Flow_048zvso" name="Да" sourceRef="Gateway_0un5lsg" targetRef="Activity_1hi8iqt" />
    <bpmn:sequenceFlow id="Flow_0006i37" name="Нет" sourceRef="Gateway_0un5lsg" targetRef="Activity_1196mqc" />
    <bpmn:sequenceFlow id="Flow_1bvxqpv" sourceRef="Activity_1196mqc" targetRef="Event_1ajchaw" />
    <bpmn:sequenceFlow id="Flow_1sl214g" sourceRef="Activity_1hi8iqt" targetRef="Event_1p80eug" />
    <bpmn:sequenceFlow id="Flow_1uzqo6r" sourceRef="Activity_1spgumw" targetRef="Event_0lfeczd" />
    <bpmn:startEvent id="Event_0efb55i" name="Изменение статуса">
      <bpmn:outgoing>Flow_0jqs1bd</bpmn:outgoing>
    </bpmn:startEvent>
    <bpmn:task id="Activity_0ykyrme" name="Анализ статуса">
      <bpmn:incoming>Flow_0htyjxn</bpmn:incoming>
      <bpmn:outgoing>Flow_0h00d8r</bpmn:outgoing>
    </bpmn:task>
    <bpmn:task id="Activity_0rul86p" name="Сохранить статус в БД">
      <bpmn:incoming>Flow_1r5aog6</bpmn:incoming>
      <bpmn:outgoing>Flow_0htyjxn</bpmn:outgoing>
    </bpmn:task>
    <bpmn:sequenceFlow id="Flow_1r5aog6" sourceRef="Activity_0otjrlt" targetRef="Activity_0rul86p" />
    <bpmn:sequenceFlow id="Flow_0htyjxn" sourceRef="Activity_0rul86p" targetRef="Activity_0ykyrme" />
    <bpmn:exclusiveGateway id="Gateway_0tei6kf" name="Нужно отправлять уведомления?">
      <bpmn:incoming>Flow_0h00d8r</bpmn:incoming>
      <bpmn:outgoing>Flow_1s2e09b</bpmn:outgoing>
      <bpmn:outgoing>Flow_0qxgerl</bpmn:outgoing>
    </bpmn:exclusiveGateway>
  </bpmn:process>
  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="Collaboration_16tndhq">
      <bpmndi:BPMNShape id="Participant_02ybbep_di" bpmnElement="Participant_163frdf" isHorizontal="true">
        <dc:Bounds x="160" y="80" width="1260" height="660" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Lane_0qzn06w_di" bpmnElement="Lane_0qzn06w" isHorizontal="true">
        <dc:Bounds x="190" y="80" width="1230" height="170" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Lane_0gajj4m_di" bpmnElement="Lane_0gajj4m" isHorizontal="true">
        <dc:Bounds x="190" y="490" width="1230" height="250" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Lane_0jmicoq_di" bpmnElement="Lane_0jmicoq" isHorizontal="true">
        <dc:Bounds x="190" y="250" width="1230" height="240" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_0otjrlt_di" bpmnElement="Activity_0otjrlt">
        <dc:Bounds x="230" y="340" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_0m9sn73_di" bpmnElement="Event_1ac33rz">
        <dc:Bounds x="942" y="592" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="926" y="635" width="68" height="53" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_120u8o2_di" bpmnElement="Activity_120u8o2">
        <dc:Bounds x="1050" y="570" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_05hlxuz_di" bpmnElement="Activity_0rjjaof">
        <dc:Bounds x="930" y="340" width="100" height="80" />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Gateway_0un5lsg_di" bpmnElement="Gateway_0un5lsg" isMarkerVisible="true">
        <dc:Bounds x="1085" y="355" width="50" height="50" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1086" y="412" width="50" height="27" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_1196mqc_di" bpmnElement="Activity_1196mqc">
        <dc:Bounds x="1190" y="380" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_1hi8iqt_di" bpmnElement="Activity_1hi8iqt">
        <dc:Bounds x="1190" y="280" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_1p80eug_di" bpmnElement="Event_1p80eug">
        <dc:Bounds x="1352" y="302" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1336" y="345" width="69" height="27" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_1ajchaw_di" bpmnElement="Event_1ajchaw">
        <dc:Bounds x="1352" y="402" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1341" y="445" width="60" height="40" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_0g1wmhs_di" bpmnElement="Event_0g1wmhs">
        <dc:Bounds x="1222" y="592" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1206" y="635" width="69" height="27" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_000hvqe_di" bpmnElement="Activity_000hvqe">
        <dc:Bounds x="740" y="380" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_1spgumw_di" bpmnElement="Activity_1spgumw">
        <dc:Bounds x="740" y="280" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_0lfeczd_di" bpmnElement="Event_0lfeczd">
        <dc:Bounds x="882" y="302" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="858" y="266" width="84" height="27" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Event_1psahy4_di" bpmnElement="Event_0efb55i">
        <dc:Bounds x="262" y="162" width="36" height="36" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="251" y="125" width="58" height="27" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_0ykyrme_di" bpmnElement="Activity_0ykyrme">
        <dc:Bounds x="480" y="340" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Activity_0rul86p_di" bpmnElement="Activity_0rul86p">
        <dc:Bounds x="350" y="260" width="100" height="80" />
        <bpmndi:BPMNLabel />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Gateway_0tei6kf_di" bpmnElement="Gateway_0tei6kf" isMarkerVisible="true">
        <dc:Bounds x="615" y="355" width="50" height="50" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="603" y="415" width="74" height="40" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge id="Flow_0jqs1bd_di" bpmnElement="Flow_0jqs1bd">
        <di:waypoint x="280" y="198" />
        <di:waypoint x="280" y="340" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0h00d8r_di" bpmnElement="Flow_0h00d8r">
        <di:waypoint x="580" y="380" />
        <di:waypoint x="615" y="380" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0iyyna0_di" bpmnElement="Flow_0iyyna0">
        <di:waypoint x="978" y="610" />
        <di:waypoint x="1050" y="610" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0h1trxo_di" bpmnElement="Flow_0h1trxo">
        <di:waypoint x="1150" y="610" />
        <di:waypoint x="1222" y="610" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1s2e09b_di" bpmnElement="Flow_1s2e09b">
        <di:waypoint x="665" y="380" />
        <di:waypoint x="693" y="380" />
        <di:waypoint x="693" y="420" />
        <di:waypoint x="740" y="420" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="707" y="402" width="14" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0qxgerl_di" bpmnElement="Flow_0qxgerl">
        <di:waypoint x="640" y="355" />
        <di:waypoint x="640" y="320" />
        <di:waypoint x="740" y="320" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="645" y="335" width="20" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1811y2r_di" bpmnElement="Flow_1811y2r">
        <di:waypoint x="840" y="420" />
        <di:waypoint x="885" y="420" />
        <di:waypoint x="885" y="380" />
        <di:waypoint x="930" y="380" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0ect44b_di" bpmnElement="Flow_0ect44b">
        <di:waypoint x="1030" y="380" />
        <di:waypoint x="1085" y="380" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_048zvso_di" bpmnElement="Flow_048zvso">
        <di:waypoint x="1110" y="355" />
        <di:waypoint x="1110" y="320" />
        <di:waypoint x="1190" y="320" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1118" y="335" width="14" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0006i37_di" bpmnElement="Flow_0006i37">
        <di:waypoint x="1135" y="380" />
        <di:waypoint x="1163" y="380" />
        <di:waypoint x="1163" y="420" />
        <di:waypoint x="1190" y="420" />
        <bpmndi:BPMNLabel>
          <dc:Bounds x="1168" y="397" width="20" height="14" />
        </bpmndi:BPMNLabel>
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1bvxqpv_di" bpmnElement="Flow_1bvxqpv">
        <di:waypoint x="1290" y="420" />
        <di:waypoint x="1352" y="420" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1sl214g_di" bpmnElement="Flow_1sl214g">
        <di:waypoint x="1290" y="320" />
        <di:waypoint x="1352" y="320" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1uzqo6r_di" bpmnElement="Flow_1uzqo6r">
        <di:waypoint x="840" y="320" />
        <di:waypoint x="882" y="320" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_1r5aog6_di" bpmnElement="Flow_1r5aog6">
        <di:waypoint x="330" y="380" />
        <di:waypoint x="380" y="380" />
        <di:waypoint x="380" y="340" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_0htyjxn_di" bpmnElement="Flow_0htyjxn">
        <di:waypoint x="410" y="340" />
        <di:waypoint x="410" y="380" />
        <di:waypoint x="480" y="380" />
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>

```

