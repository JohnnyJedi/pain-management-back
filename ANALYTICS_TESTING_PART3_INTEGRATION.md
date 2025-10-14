# 📊 ТЕСТИРОВАНИЕ МОДУЛЯ АНАЛИТИКИ - ЧАСТЬ 3: ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ

## 🎯 ЦЕЛЬ
Полное сквозное тестирование модуля аналитики в реальных сценариях использования системы.

---

## СЦЕНАРИЙ 1: Полный цикл работы медсестры

### Шаг 1: Вход медсестры
**POSTMAN:**
```http
POST http://localhost:8080/api/persons/login
{
  "login": "nurse001",
  "password": "password123"
}
```

**ПРОВЕРКА В MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "USER_LOGIN_SUCCESS",
  userId: "nurse001"
}).count()
// Должно увеличиться на 1
```

---

### Шаг 2: Медсестра записывает VAS для пациента
**POSTMAN:**
```http
POST http://localhost:8080/api/nurse/vas?nurseId=nurse001
{
  "patientMrn": "MRN001",
  "vasLevel": 8,
  "painLocation": "Abdomen"
}
```

**ПРОВЕРКА В MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "VAS_RECORDED",
  userId: "nurse001",
  patientMrn: "MRN001"
}).sort({timestamp: -1}).limit(1)
```

**ПРОВЕРКА:**
- ✓ vasLevel = 8
- ✓ priority = "HIGH" (т.к. >= 8)
- ✓ metadata.isCritical = true

---

### Шаг 3: Проверка активности медсестры
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/nurse001/activity
```

**ОЖИДАЕМЫЙ РЕЗУЛЬТАТ:**
```json
{
  "userId": "nurse001",
  "userRole": "NURSE",
  "totalActions": 2,
  "loginCount": 1
}
```

**ГАЛОЧКА ✓:** Полный цикл медсестры отслеживается

---

## СЦЕНАРИЙ 2: Полный цикл работы доктора с рекомендацией

### Шаг 1: Вход доктора
**POSTMAN:**
```http
POST http://localhost:8080/api/persons/login
{
  "login": "doctor001",
  "password": "password123"
}
```

---

### Шаг 2: Доктор создает рекомендацию
**POSTMAN:**
```http
POST http://localhost:8080/api/doctor/recommendations?doctorId=doctor001
{
  "patientMrn": "MRN001",
  "description": "Increase opioid dosage",
  "justification": "VAS level 8"
}
```

**ЗАПОМНИТЕ:** recommendationId (например, 5)

---

### Шаг 3: Доктор одобряет рекомендацию
**POSTMAN:**
```http
PUT http://localhost:8080/api/doctor/recommendations/5/approve?doctorId=doctor001
{
  "comment": "Approved"
}
```

---

### Шаг 4: Проверка событий
**MONGODB:**
```javascript
// Должно быть 3 события от doctor001:
// 1. USER_LOGIN_SUCCESS
// 2. (создание рекомендации - если есть событие)
// 3. RECOMMENDATION_APPROVED

db.analytics_events.find({
  userId: "doctor001"
}).sort({timestamp: 1})
```

---

### Шаг 5: Проверка статистики производительности
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/performance
```

**ПРОВЕРКА:**
- ✓ totalRecommendations увеличилось
- ✓ approvedRecommendations увеличилось
- ✓ averageProcessingTimeMs рассчитано

**ГАЛОЧКА ✓:** Полный цикл доктора с одобрением отслеживается

---

## СЦЕНАРИЙ 3: Полный цикл с эскалацией

### Шаг 1: Доктор создает рекомендацию
**POSTMAN:**
```http
POST http://localhost:8080/api/doctor/recommendations?doctorId=doctor001
{
  "patientMrn": "MRN002",
  "description": "High-risk medication",
  "justification": "Severe pain"
}
```

**ЗАПОМНИТЕ:** recommendationId (например, 6)

---

### Шаг 2: Доктор отклоняет рекомендацию (создается эскалация)
**POSTMAN:**
```http
PUT http://localhost:8080/api/doctor/recommendations/6/reject?doctorId=doctor001
{
  "rejectedReason": "Requires specialist approval",
  "comment": "High risk"
}
```

---

### Шаг 3: Проверка создания эскалации
**MONGODB:**
```javascript
// Должно быть 2 события:
db.analytics_events.find({
  recommendationId: 6
}).sort({timestamp: 1})

// 1. RECOMMENDATION_REJECTED
// 2. ESCALATION_CREATED
```

**ЗАПОМНИТЕ:** escalationId из события ESCALATION_CREATED

---

### Шаг 4: Анестезиолог разрешает эскалацию
**POSTMAN:**
```http
PUT http://localhost:8080/api/anesthesiologist/escalations/{escalationId}/resolve?anesthesiologistId=anesth001
{
  "resolution": "Approved with modifications",
  "approved": true
}
```

---

### Шаг 5: Проверка разрешения эскалации
**MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "ESCALATION_RESOLVED",
  escalationId: {escalationId}
})
```

**ПРОВЕРКА:**
- ✓ userId = "anesth001"
- ✓ userRole = "ANESTHESIOLOGIST"
- ✓ processingTimeMs > 0 (время от создания до разрешения)
- ✓ metadata.approved = true

---

### Шаг 6: Проверка статистики производительности
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/performance
```

**ПРОВЕРКА:**
- ✓ totalEscalations увеличилось
- ✓ resolvedEscalations увеличилось
- ✓ averageEscalationResolutionTimeMs рассчитано

**ГАЛОЧКА ✓:** Полный цикл с эскалацией отслеживается от начала до конца

---

## СЦЕНАРИЙ 4: Администратор управляет персоналом

### Шаг 1: Админ создает нового доктора
**POSTMAN:**
```http
POST http://localhost:8080/api/admin/persons
{
  "personId": "DOC999",
  "firstName": "Test",
  "lastName": "Doctor",
  "login": "testdoc",
  "password": "password123",
  "role": "DOCTOR"
}
```

---

### Шаг 2: Проверка события создания
**MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "PERSON_CREATED",
  "metadata.newPersonId": "DOC999"
})
```

**ПРОВЕРКА:**
- ✓ userId = "admin001" (кто создал)
- ✓ userRole = "ADMIN"
- ✓ metadata.newPersonRole = "DOCTOR"

---

### Шаг 3: Админ обновляет данные доктора
**POSTMAN:**
```http
PUT http://localhost:8080/api/admin/persons/DOC999?updatedBy=admin001
{
  "firstName": "Test",
  "lastName": "Doctor-Updated",
  "login": "testdoc",
  "role": "DOCTOR"
}
```

---

### Шаг 4: Проверка события обновления
**MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "PERSON_UPDATED",
  "metadata.updatedPersonId": "DOC999"
})
```

**ПРОВЕРКА:**
- ✓ metadata.changedFields содержит "lastName"

---

### Шаг 5: Админ удаляет доктора
**POSTMAN:**
```http
DELETE http://localhost:8080/api/admin/persons/DOC999?deletedBy=admin001&reason=Test completed
```

---

### Шаг 6: Проверка события удаления
**MONGODB:**
```javascript
db.analytics_events.find({
  eventType: "PERSON_DELETED",
  "metadata.deletedPersonId": "DOC999"
})
```

**ПРОВЕРКА:**
- ✓ metadata.reason = "Test completed"

---

### Шаг 7: Проверка активности админа
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/admin001/activity
```

**ПРОВЕРКА:**
- ✓ totalActions включает все 3 операции (create, update, delete)

**ГАЛОЧКА ✓:** Полный цикл управления персоналом отслеживается

---

## СЦЕНАРИЙ 5: Массовая нагрузка и проверка производительности

### Шаг 1: Создание 10 пациентов
**ЧТО ДЕЛАТЬ:**
Через Postman или скрипт создайте 10 пациентов с разными данными.

---

### Шаг 2: Запись 50 VAS записей
**ЧТО ДЕЛАТЬ:**
Создайте 50 VAS записей для разных пациентов с разными уровнями боли (1-10).

---

### Шаг 3: Проверка статистики пациентов
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/patients/stats
```

**ПРОВЕРКА:**
- ✓ totalPatients = 10
- ✓ totalVasRecords = 50
- ✓ patientsByGender распределение корректно
- ✓ patientsByAgeGroup распределение корректно
- ✓ averageVasLevel рассчитан правильно

---

### Шаг 4: Проверка производительности запросов
**POSTMAN:**
Выполните запрос и проверьте время:
```http
GET http://localhost:8080/api/analytics/events/stats
```

**ОЖИДАЕМОЕ ВРЕМЯ:** < 500ms

**ГАЛОЧКА ✓:** Система справляется с большим количеством данных

---

## СЦЕНАРИЙ 6: Проверка технических логов

### Шаг 1: Выполнение различных операций
**ЧТО ДЕЛАТЬ:**
Выполните 10-15 различных операций в системе (создание, обновление, удаление).

---

### Шаг 2: Проверка логов в MongoDB
**MONGODB:**
```javascript
db.log_entries.find().sort({timestamp: -1}).limit(20)
```

**ПРОВЕРКА:**
- ✓ Для каждой операции есть лог
- ✓ className и methodName корректны
- ✓ durationMs > 0
- ✓ module определен правильно

---

### Шаг 3: Проверка логов через API
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/logs/recent?limit=20
```

**ПРОВЕРКА:**
- ✓ Логи совпадают с MongoDB
- ✓ Сортировка правильная

---

### Шаг 4: Создание ошибки
**ЧТО ДЕЛАТЬ:**
Попытайтесь создать пациента с невалидными данными (например, пустое имя).

---

### Шаг 5: Проверка ERROR логов
**POSTMAN:**
```http
GET http://localhost:8080/api/analytics/logs/level/ERROR
```

**ПРОВЕРКА:**
- ✓ ERROR лог создан
- ✓ errorMessage содержит описание ошибки
- ✓ errorStackTrace присутствует
- ✓ success = false

**ГАЛОЧКА ✓:** Технические логи фиксируют все операции и ошибки

---

## СЦЕНАРИЙ 7: Фронтенд - Полная проверка дашборда

### Шаг 1: Открытие дашборда
**ЧТО ДЕЛАТЬ:**
1. Войдите как администратор
2. Откройте страницу аналитики/дашборда

---

### Шаг 2: Проверка главной страницы
**ЧТО ПРОВЕРИТЬ:**
- [ ] Общая статистика загружается
- [ ] Числа отображаются корректно
- [ ] Графики рендерятся без ошибок
- [ ] Нет ошибок в консоли браузера (F12)

---

### Шаг 3: Проверка фильтров по датам
**ЧТО ДЕЛАТЬ:**
1. Выберите "Today"
2. Проверьте, что данные обновились
3. Выберите "Last 7 days"
4. Проверьте, что данные изменились

**ПРОВЕРКА:**
- ✓ Все секции обновляются при изменении фильтра
- ✓ Числа меняются логично

---

### Шаг 4: Проверка Recent Activity
**ЧТО ПРОВЕРИТЬ:**
- [ ] Последние события отображаются
- [ ] Клик на событие открывает детали
- [ ] Время обновляется (если real-time)

---

### Шаг 5: Проверка User Activity
**ЧТО ДЕЛАТЬ:**
1. Кликните на пользователя в списке
2. Проверьте детальную информацию

**ПРОВЕРКА:**
- ✓ Статистика пользователя корректна
- ✓ График активности отображается

---

### Шаг 6: Проверка Performance Metrics
**ЧТО ПРОВЕРИТЬ:**
- [ ] Метрики производительности отображаются
- [ ] Графики трендов работают
- [ ] Можно навести на точку графика для деталей

---

### Шаг 7: Проверка Patient Analytics
**ЧТО ПРОВЕРИТЬ:**
- [ ] Pie chart для пола отображается
- [ ] Bar chart для возрастов отображается
- [ ] VAS статистика корректна
- [ ] Критические случаи выделены

---

### Шаг 8: Проверка System Logs
**ЧТО ДЕЛАТЬ:**
1. Откройте раздел логов
2. Примените фильтр "ERROR"
3. Кликните на ERROR лог

**ПРОВЕРКА:**
- ✓ Таблица логов загружается
- ✓ Фильтр работает
- ✓ Stack trace отображается при клике
- ✓ Можно скопировать stack trace

---

### Шаг 9: Проверка экспорта данных
**ЧТО ДЕЛАТЬ:**
1. Нажмите кнопку "Export" (если есть)
2. Выберите формат (CSV/Excel)
3. Скачайте файл

**ПРОВЕРКА:**
- ✓ Файл скачивается
- ✓ Данные в файле корректны
- ✓ Формат читаемый

**ГАЛОЧКА ✓:** Фронтенд полностью функционален

---

## СЦЕНАРИЙ 8: Проверка real-time обновлений (если реализовано)

### Шаг 1: Открыть дашборд в браузере
**ЧТО ДЕЛАТЬ:**
Откройте дашборд и оставьте его открытым.

---

### Шаг 2: Выполнить действие через Postman
**ЧТО ДЕЛАТЬ:**
Создайте VAS запись через Postman.

---

### Шаг 3: Проверка обновления на фронтенде
**ЧТО ПРОВЕРИТЬ:**
- ✓ Дашборд обновился автоматически (если есть WebSocket/SSE)
- ✓ Новое событие появилось в Recent Activity
- ✓ Счетчики обновились

**ГАЛОЧКА ✓:** Real-time обновления работают (если реализованы)

---

## ФИНАЛЬНАЯ ПРОВЕРКА: Целостность данных

### Проверка 1: Соответствие событий и логов
**MONGODB:**
```javascript
// Подсчет событий
var eventsCount = db.analytics_events.count()

// Подсчет логов
var logsCount = db.log_entries.count()

// Логов должно быть >= событий (т.к. логируются все методы)
print("Events: " + eventsCount)
print("Logs: " + logsCount)
```

**ГАЛОЧКА ✓:** Логов >= событий

---

### Проверка 2: Нет дублирования событий
**MONGODB:**
```javascript
// Проверка на дубликаты (одинаковые события в одно время)
db.analytics_events.aggregate([
  {
    $group: {
      _id: {
        eventType: "$eventType",
        userId: "$userId",
        timestamp: "$timestamp"
      },
      count: { $sum: 1 }
    }
  },
  {
    $match: { count: { $gt: 1 } }
  }
])
// Должно вернуть пустой массив
```

**ГАЛОЧКА ✓:** Нет дублирования событий

---

### Проверка 3: Все события имеют обязательные поля
**MONGODB:**
```javascript
// Проверка обязательных полей
db.analytics_events.find({
  $or: [
    { timestamp: { $exists: false } },
    { eventType: { $exists: false } },
    { userId: { $exists: false } }
  ]
}).count()
// Должно быть 0
```

**ГАЛОЧКА ✓:** Все события валидны

---

### Проверка 4: Расчеты статистики корректны
**ЧТО ДЕЛАТЬ:**
1. Получите статистику через API
2. Вручную подсчитайте в MongoDB
3. Сравните результаты

**ПРИМЕР:**
```javascript
// API говорит: totalVasRecords = 57
// MongoDB:
db.analytics_events.find({ eventType: "VAS_RECORDED" }).count()
// Должно быть 57
```

**ГАЛОЧКА ✓:** Расчеты статистики точны

---

## ✅ ФИНАЛЬНЫЙ ЧЕКЛИСТ: МОДУЛЬ АНАЛИТИКИ

### События (Part 1)
- [ ] Все 10 типов событий создаются
- [ ] События содержат все обязательные поля
- [ ] Metadata содержит дополнительную информацию
- [ ] Асинхронная обработка работает

### API Статистики (Part 2A)
- [ ] GET /api/analytics/events/stats работает
- [ ] GET /api/analytics/users/{userId}/activity работает
- [ ] GET /api/analytics/performance работает
- [ ] GET /api/analytics/patients/stats работает
- [ ] Фильтрация по датам работает везде

### API Событий и Логов (Part 2B)
- [ ] GET /api/analytics/events/recent работает
- [ ] GET /api/analytics/events/type/{eventType} работает
- [ ] GET /api/analytics/logs/recent работает
- [ ] GET /api/analytics/logs/level/{level} работает
- [ ] Технические логи создаются автоматически

### Интеграция (Part 3)
- [ ] Полный цикл медсестры отслеживается
- [ ] Полный цикл доктора отслеживается
- [ ] Полный цикл с эскалацией работает
- [ ] Управление персоналом отслеживается
- [ ] Система справляется с нагрузкой
- [ ] ERROR логи фиксируют ошибки

### Фронтенд
- [ ] Дашборд загружается без ошибок
- [ ] Все секции отображаются корректно
- [ ] Фильтры работают
- [ ] Графики рендерятся
- [ ] Экспорт данных работает

### MongoDB
- [ ] Индексы созданы автоматически
- [ ] Нет дублирования данных
- [ ] Все документы валидны
- [ ] Запросы выполняются быстро

### Производительность
- [ ] API отвечает < 500ms
- [ ] Асинхронность не блокирует основной поток
- [ ] Система справляется с большим объемом данных

---

## 🎉 ПОЗДРАВЛЯЕМ!

Если все галочки проставлены ✅, ваш модуль аналитики:
- ✅ Полностью функционален
- ✅ Корректно отслеживает все события
- ✅ Предоставляет точную статистику
- ✅ Работает быстро и стабильно
- ✅ Интегрирован с фронтендом
- ✅ Готов к продакшену

**Модуль аналитики протестирован полностью!** 🚀
