# 📊 ТЕСТИРОВАНИЕ МОДУЛЯ АНАЛИТИКИ - ЧАСТЬ 2A: API СТАТИСТИКИ

## 🎯 ЦЕЛЬ
Тестирование REST API эндпоинтов для получения статистики. Проверка правильности расчетов и агрегации данных.

---

## API ENDPOINT 1: GET /api/analytics/events/stats

### Описание
Получить общую статистику по всем событиям с группировкой по типу, роли и статусу.

---

### ТЕСТ 1.1: Получение общей статистики (без фильтров)

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/events/stats
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "totalEvents": 21,
  "eventsByType": {
    "VAS_RECORDED": 7,
    "USER_LOGIN_SUCCESS": 3,
    "PATIENT_REGISTERED": 2,
    "PERSON_CREATED": 2,
    "RECOMMENDATION_APPROVED": 1,
    "RECOMMENDATION_REJECTED": 1,
    "ESCALATION_CREATED": 1,
    "ESCALATION_RESOLVED": 1,
    "PERSON_UPDATED": 1,
    "PERSON_DELETED": 1,
    "USER_LOGIN_FAILED": 1
  },
  "eventsByRole": {
    "NURSE": 7,
    "ADMIN": 5,
    "DOCTOR": 6,
    "ANESTHESIOLOGIST": 1
  },
  "eventsByStatus": {
    "SUCCESS": 3,
    "FAILED": 1,
    "APPROVED": 1,
    "REJECTED": 1,
    "RESOLVED": 1
  }
}
```

**ПРОВЕРКИ:**
- ✓ totalEvents = сумма всех событий из Part 1
- ✓ eventsByType содержит все 10+ типов событий
- ✓ eventsByRole содержит NURSE, DOCTOR, ADMIN, ANESTHESIOLOGIST
- ✓ eventsByStatus содержит различные статусы
- ✓ Суммы в группах совпадают с totalEvents

**ЧТО ПРОВЕРИТЬ В MONGODB:**
```javascript
// Подсчет вручную
db.analytics_events.count()
// Должно совпадать с totalEvents

// Группировка по типу
db.analytics_events.aggregate([
  { $group: { _id: "$eventType", count: { $sum: 1 } } }
])
// Должно совпадать с eventsByType
```

**ГАЛОЧКА ✓:** Общая статистика корректна, числа совпадают с MongoDB

---

### ТЕСТ 1.2: Статистика за период (с фильтрами)

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/events/stats?startDate=2025-10-13T00:00:00&endDate=2025-10-13T23:59:59
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "totalEvents": 21,
  "eventsByType": { ... },
  "eventsByRole": { ... },
  "eventsByStatus": { ... }
}
```

**ПРОВЕРКИ:**
- ✓ Возвращаются только события за сегодня
- ✓ totalEvents <= общего количества событий

**ЧТО ПРОВЕРИТЬ В MONGODB:**
```javascript
db.analytics_events.find({
  timestamp: {
    $gte: ISODate("2025-10-13T00:00:00Z"),
    $lte: ISODate("2025-10-13T23:59:59Z")
  }
}).count()
```

**ГАЛОЧКА ✓:** Фильтрация по датам работает корректно

---

### ТЕСТ 1.3: Статистика за прошлый день (пустой результат)

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/events/stats?startDate=2025-10-12T00:00:00&endDate=2025-10-12T23:59:59
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "totalEvents": 0,
  "eventsByType": {},
  "eventsByRole": {},
  "eventsByStatus": {}
}
```

**ГАЛОЧКА ✓:** Пустой период возвращает нулевую статистику

---

## API ENDPOINT 2: GET /api/analytics/users/{userId}/activity

### Описание
Получить активность конкретного пользователя: количество действий, последний вход, статистику логинов.

---

### ТЕСТ 2.1: Активность администратора

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/admin001/activity
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "userId": "admin001",
  "userRole": "ADMIN",
  "totalActions": 5,
  "lastActivity": "2025-10-13T12:40:00",
  "loginCount": 2,
  "failedLoginCount": 1
}
```

**ПРОВЕРКИ:**
- ✓ userId = "admin001"
- ✓ userRole = "ADMIN"
- ✓ totalActions = количество событий где userId = admin001
- ✓ lastActivity = timestamp последнего события
- ✓ loginCount = количество USER_LOGIN_SUCCESS
- ✓ failedLoginCount = количество USER_LOGIN_FAILED

**ЧТО ПРОВЕРИТЬ В MONGODB:**
```javascript
// Все события пользователя
db.analytics_events.find({ userId: "admin001" }).count()
// Должно = totalActions

// Последняя активность
db.analytics_events.find({ userId: "admin001" })
  .sort({ timestamp: -1 })
  .limit(1)
// timestamp должен = lastActivity

// Успешные логины
db.analytics_events.find({
  userId: "admin001",
  eventType: "USER_LOGIN_SUCCESS"
}).count()
// Должно = loginCount
```

**ГАЛОЧКА ✓:** Активность пользователя рассчитана правильно

---

### ТЕСТ 2.2: Активность медсестры

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/NURSE001/activity
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "userId": "NURSE001",
  "userRole": "NURSE",
  "totalActions": 7,
  "lastActivity": "2025-10-13T12:35:00",
  "loginCount": 0,
  "failedLoginCount": 0
}
```

**ПРОВЕРКИ:**
- ✓ totalActions = 7 (все VAS записи)
- ✓ userRole = "NURSE"
- ✓ loginCount = 0 (медсестра не логинилась в тестах)

**ГАЛОЧКА ✓:** Активность медсестры корректна

---

### ТЕСТ 2.3: Активность доктора за период

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/DOC001/activity?startDate=2025-10-13T12:00:00&endDate=2025-10-13T12:30:00
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "userId": "DOC001",
  "userRole": "DOCTOR",
  "totalActions": 4,
  "lastActivity": "2025-10-13T12:28:00",
  "loginCount": 0,
  "failedLoginCount": 0
}
```

**ПРОВЕРКИ:**
- ✓ Учитываются только события в указанном периоде
- ✓ totalActions меньше общего количества действий доктора

**ГАЛОЧКА ✓:** Фильтрация по датам для активности работает

---

### ТЕСТ 2.4: Несуществующий пользователь

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/users/NONEXISTENT/activity
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "userId": "NONEXISTENT",
  "userRole": "UNKNOWN",
  "totalActions": 0,
  "lastActivity": null,
  "loginCount": 0,
  "failedLoginCount": 0
}
```

**ГАЛОЧКА ✓:** Несуществующий пользователь возвращает пустую статистику

---

## API ENDPOINT 3: GET /api/analytics/performance

### Описание
Статистика производительности: рекомендации, эскалации, время обработки.

---

### ТЕСТ 3.1: Общая статистика производительности

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/performance
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "averageProcessingTimeMs": 1234.5,
  "totalRecommendations": 2,
  "approvedRecommendations": 1,
  "rejectedRecommendations": 1,
  "totalEscalations": 1,
  "resolvedEscalations": 1,
  "averageEscalationResolutionTimeMs": 5678.0
}
```

**ПРОВЕРКИ:**
- ✓ totalRecommendations = approvedRecommendations + rejectedRecommendations
- ✓ averageProcessingTimeMs > 0
- ✓ totalEscalations >= resolvedEscalations
- ✓ averageEscalationResolutionTimeMs > 0

**ЧТО ПРОВЕРИТЬ В MONGODB:**
```javascript
// Подсчет рекомендаций
db.analytics_events.find({
  eventType: { $in: ["RECOMMENDATION_APPROVED", "RECOMMENDATION_REJECTED"] }
}).count()
// Должно = totalRecommendations

// Одобренные рекомендации
db.analytics_events.find({
  eventType: "RECOMMENDATION_APPROVED"
}).count()
// Должно = approvedRecommendations

// Среднее время обработки
db.analytics_events.aggregate([
  {
    $match: {
      eventType: { $in: ["RECOMMENDATION_APPROVED", "RECOMMENDATION_REJECTED"] },
      processingTimeMs: { $exists: true }
    }
  },
  {
    $group: {
      _id: null,
      avgTime: { $avg: "$processingTimeMs" }
    }
  }
])
// avgTime должно = averageProcessingTimeMs
```

**ГАЛОЧКА ✓:** Статистика производительности рассчитана верно

---

### ТЕСТ 3.2: Производительность за период

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/performance?startDate=2025-10-13T00:00:00&endDate=2025-10-13T23:59:59
```

**ПРОВЕРКИ:**
- ✓ Учитываются только события за указанный период
- ✓ Все метрики >= 0

**ГАЛОЧКА ✓:** Фильтрация по датам работает для производительности

---

## API ENDPOINT 4: GET /api/analytics/patients/stats

### Описание
Статистика по пациентам: количество, распределение по полу и возрасту, VAS записи.

---

### ТЕСТ 4.1: Общая статистика по пациентам

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/patients/stats
```

**ЧТО ДОЛЖНО ВЕРНУТЬСЯ:**
```json
{
  "totalPatients": 2,
  "patientsByGender": {
    "FEMALE": 1,
    "MALE": 1
  },
  "patientsByAgeGroup": {
    "30-44": 1,
    "45-59": 1
  },
  "totalVasRecords": 7,
  "criticalVasRecords": 2,
  "averageVasLevel": 6.14
}
```

**ПРОВЕРКИ:**
- ✓ totalPatients = количество PATIENT_REGISTERED событий
- ✓ patientsByGender: сумма = totalPatients
- ✓ patientsByAgeGroup: сумма = totalPatients
- ✓ totalVasRecords = количество VAS_RECORDED событий
- ✓ criticalVasRecords = количество VAS с priority="HIGH"
- ✓ averageVasLevel = среднее значение vasLevel

**ЧТО ПРОВЕРИТЬ В MONGODB:**
```javascript
// Пациенты
db.analytics_events.find({
  eventType: "PATIENT_REGISTERED"
}).count()
// Должно = totalPatients

// Пациенты по полу
db.analytics_events.aggregate([
  { $match: { eventType: "PATIENT_REGISTERED" } },
  { $group: { _id: "$metadata.gender", count: { $sum: 1 } } }
])
// Должно совпадать с patientsByGender

// VAS записи
db.analytics_events.find({
  eventType: "VAS_RECORDED"
}).count()
// Должно = totalVasRecords

// Критические VAS
db.analytics_events.find({
  eventType: "VAS_RECORDED",
  priority: "HIGH"
}).count()
// Должно = criticalVasRecords

// Средний VAS
db.analytics_events.aggregate([
  { $match: { eventType: "VAS_RECORDED" } },
  { $group: { _id: null, avg: { $avg: "$vasLevel" } } }
])
// avg должно = averageVasLevel
```

**ГАЛОЧКА ✓:** Статистика по пациентам корректна

---

### ТЕСТ 4.2: Проверка возрастных групп

**ОЖИДАЕМЫЕ ГРУППЫ:**
- "0-17" - дети
- "18-29" - молодые
- "30-44" - средний возраст
- "45-59" - зрелый возраст
- "60-74" - пожилые
- "75+" - очень пожилые

**ПРОВЕРКА:**
Создайте тестовых пациентов разных возрастов и убедитесь, что они попадают в правильные группы.

**ГАЛОЧКА ✓:** Возрастные группы рассчитываются правильно

---

### ТЕСТ 4.3: Статистика пациентов за период

**ЧТО ДЕЛАТЬ В POSTMAN:**
```http
GET http://localhost:8080/api/analytics/patients/stats?startDate=2025-10-13T12:00:00&endDate=2025-10-13T12:20:00
```

**ПРОВЕРКИ:**
- ✓ Учитываются только пациенты, зарегистрированные в указанный период
- ✓ VAS записи также фильтруются по периоду

**ГАЛОЧКА ✓:** Фильтрация по датам работает для пациентов

---

## 🎨 ТЕСТИРОВАНИЕ С ФРОНТЕНДА (Dashboard)

### Что проверить на дашборде администратора:

#### 1. Общая статистика (EventStatsDTO)
**ГДЕ СМОТРЕТЬ:** Главная панель дашборда

**ЧТО ДОЛЖНО ОТОБРАЖАТЬСЯ:**
- Общее количество событий (число)
- График/диаграмма событий по типам
- Распределение по ролям (pie chart)
- Распределение по статусам

**ПРОВЕРКА:**
- ✓ Числа совпадают с API ответом
- ✓ Графики обновляются при изменении периода
- ✓ Цвета различаются для разных типов событий

---

#### 2. Активность пользователей
**ГДЕ СМОТРЕТЬ:** Раздел "User Activity"

**ЧТО ДОЛЖНО ОТОБРАЖАТЬСЯ:**
- Список пользователей с их активностью
- Последняя активность каждого пользователя
- Количество логинов
- Количество неудачных попыток входа

**ПРОВЕРКА:**
- ✓ Данные совпадают с API
- ✓ Время отображается в правильном формате
- ✓ Можно кликнуть на пользователя для детальной информации

---

#### 3. Производительность системы
**ГДЕ СМОТРЕТЬ:** Раздел "Performance Metrics"

**ЧТО ДОЛЖНО ОТОБРАЖАТЬСЯ:**
- Среднее время обработки рекомендаций
- Соотношение одобренных/отклоненных рекомендаций
- Статистика эскалаций
- Среднее время разрешения эскалаций

**ПРОВЕРКА:**
- ✓ Метрики обновляются в реальном времени
- ✓ Графики показывают тренды
- ✓ Время отображается в удобном формате (секунды/минуты)

---

#### 4. Статистика пациентов
**ГДЕ СМОТРЕТЬ:** Раздел "Patient Analytics"

**ЧТО ДОЛЖНО ОТОБРАЖАТЬСЯ:**
- Общее количество пациентов
- Распределение по полу (pie chart)
- Распределение по возрастным группам (bar chart)
- Статистика VAS записей
- Средний уровень боли
- Количество критических случаев

**ПРОВЕРКА:**
- ✓ Pie chart для пола показывает правильные пропорции
- ✓ Bar chart для возрастов читаемый
- ✓ Критические случаи выделены красным
- ✓ Средний VAS отображается с 2 знаками после запятой

---

## ✅ ЧЕКЛИСТ ЧАСТЬ 2A: API СТАТИСТИКИ

- [ ] GET /api/analytics/events/stats возвращает корректную статистику
- [ ] Фильтрация по датам работает для events/stats
- [ ] GET /api/analytics/users/{userId}/activity показывает активность
- [ ] Активность пользователя рассчитывается правильно
- [ ] Несуществующий пользователь возвращает UNKNOWN
- [ ] GET /api/analytics/performance показывает метрики производительности
- [ ] Время обработки рассчитывается корректно
- [ ] GET /api/analytics/patients/stats показывает статистику пациентов
- [ ] Возрастные группы рассчитываются правильно
- [ ] Критические VAS помечены как HIGH priority
- [ ] Средний VAS рассчитан верно
- [ ] Все эндпоинты поддерживают фильтрацию по датам
- [ ] Дашборд отображает все метрики корректно
- [ ] Графики на фронтенде обновляются
- [ ] Числа на фронтенде совпадают с API

**ЕСЛИ ВСЕ ГАЛОЧКИ ПРОСТАВЛЕНЫ** ✅ - переходите к PART 2B: События и логи
