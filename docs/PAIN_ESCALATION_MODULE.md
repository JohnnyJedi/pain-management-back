# 🚨 Pain Escalation Tracking Module

**Дата создания:** 21.10.2025  
**Последнее обновление:** 22.10.2025  
**Статус:** ✅ Полностью реализовано  
**Версия:** 2.0.0

---

## 📋 ОПИСАНИЕ

Модуль **Pain Escalation Tracking** обеспечивает автоматическое отслеживание роста боли у пациентов и создание эскалаций при критических ситуациях. Система анализирует историю VAS (Visual Analog Scale) и введенных доз препаратов для принятия решений о необходимости вмешательства врача или анестезиолога.

**НОВОЕ В ВЕРСИИ 2.0:**
- ✅ WebSocket real-time уведомления врачам и анестезиологам
- ✅ Автоматический мониторинг пациентов каждые 15 минут
- ✅ Проверка просроченных доз каждый час
- ✅ Ежедневная сводка по эскалациям в 08:00
- ✅ Интеграция с аналитикой через события

---

## 🎯 ОСНОВНЫЕ ФУНКЦИИ

### 1. Автоматическое обнаружение роста боли
- Сравнение текущего и предыдущего уровня VAS
- Определение критических изменений (рост на 2+ балла)
- Учет времени с момента последней дозы

### 2. Проверка интервалов между дозами
- Минимальный интервал: **4 часа** (настраивается)
- Предотвращение преждевременного введения доз
- История всех введенных препаратов

### 3. Анализ тренда боли
- Статистика за последние **24 часа** (настраивается)
- Определение направления: INCREASING, DECREASING, STABLE
- Средний, максимальный и минимальный VAS за период

### 4. Автоматическое создание эскалаций
- Приоритеты: CRITICAL, HIGH, MEDIUM, LOW
- Публикация событий в аналитику
- Рекомендации для медперсонала

---

## 🏗️ АРХИТЕКТУРА

```
pain_escalation_tracking/
├── config/
│   └── PainEscalationConfig.java                    # Конфигурация пороговых значений
├── entity/
│   └── DoseAdministration.java                      # Сущность введенной дозы
├── repository/
│   └── DoseAdministrationRepository.java            # Репозиторий доз
├── dto/
│   ├── PainEscalationCheckResult.java               # Результат проверки эскалации
│   ├── PainTrendAnalysis.java                       # Анализ тренда боли
│   ├── PainEscalationNotificationDTO.java           # DTO для WebSocket уведомлений
│   ├── DoseAdministrationRequestDTO.java            # Запрос на регистрацию дозы
│   └── DoseAdministrationResponseDTO.java           # Ответ после регистрации дозы
├── service/
│   ├── PainEscalationService.java                   # Interface
│   ├── PainEscalationServiceImpl.java               # Реализация логики
│   └── PainEscalationNotificationService.java       # WebSocket уведомления
├── controller/
│   └── PainEscalationController.java                # REST API endpoints
└── scheduler/
    └── PainMonitoringScheduler.java                 # Автоматический мониторинг
```

---

## 📦 ОСНОВНЫЕ КОМПОНЕНТЫ

### 1. **PainEscalationConfig**
Конфигурация пороговых значений через `application.properties`:

```properties
pain.escalation.min-vas-increase=2                    # Минимальный рост VAS для эскалации
pain.escalation.min-dose-interval-hours=4             # Минимальный интервал между дозами
pain.escalation.critical-vas-level=8                  # Критический уровень VAS
pain.escalation.high-vas-level=6                      # Высокий уровень VAS
pain.escalation.trend-analysis-period-hours=24        # Период анализа тренда
pain.escalation.max-escalations-per-period=3          # Макс. эскалаций за период
```

### 2. **DoseAdministration Entity**
Отслеживание введенных доз:

```java
@Entity
@Table(name = "dose_administrations")
public class DoseAdministration {
    private Long id;
    private Patient patient;
    private Recommendation recommendation;
    private String drugName;              // Название препарата
    private String dosage;                // Дозировка
    private String route;                 // Путь введения
    private LocalDateTime administeredAt; // Время введения
    private String administeredBy;        // Кто ввел
    private Integer vasBefore;            // VAS до введения
    private Integer vasAfter;             // VAS после введения
    private String notes;                 // Примечания
}
```

### 3. **PainEscalationCheckResult DTO**
Результат проверки эскалации:

```java
public class PainEscalationCheckResult {
    private String patientMrn;
    private boolean escalationRequired;        // Требуется ли эскалация
    private String escalationReason;           // Причина эскалации
    private String escalationPriority;         // Приоритет: CRITICAL/HIGH/MEDIUM/LOW
    private Integer currentVas;                // Текущий VAS
    private Integer previousVas;               // Предыдущий VAS
    private Integer vasChange;                 // Изменение VAS
    private boolean canAdministerNextDose;     // Можно ли дать следующую дозу
    private LocalDateTime lastDoseTime;        // Время последней дозы
    private Long hoursSinceLastDose;           // Часов с последней дозы
    private Integer requiredIntervalHours;     // Требуемый интервал
    private String recommendations;            // Рекомендации
    private PainTrendAnalysis painTrendAnalysis; // Анализ тренда
}
```

### 4. **PainTrendAnalysis DTO**
Анализ тренда боли:

```java
public class PainTrendAnalysis {
    private String patientMrn;
    private Integer currentVas;
    private Integer previousVas;
    private Integer vasChange;
    private LocalDateTime lastVasRecordedAt;
    private LocalDateTime previousVasRecordedAt;
    private Long hoursSinceLastVas;
    private String painTrend;              // INCREASING/DECREASING/STABLE
    private List<Integer> vasHistory;      // История VAS за период
    private Double averageVas;             // Средний VAS
    private Integer maxVas;                // Максимальный VAS
    private Integer minVas;                // Минимальный VAS
    private Integer vasRecordCount;        // Количество записей VAS
}
```

---

## 🔧 ОСНОВНЫЕ МЕТОДЫ

### 1. `checkPainEscalation(String mrn)`
Проверяет необходимость эскалации боли для пациента.

**Логика принятия решения:**

#### Сценарий 1: Критический уровень боли (VAS >= 8)
```java
if (currentVas >= 8) {
    escalationPriority = "CRITICAL";
    escalationReason = "Critical pain level: VAS 8+";
    recommendations = "URGENT: Immediate intervention required. Consider IV analgesics or anesthesiologist consultation.";
}
```

#### Сценарий 2: Значительный рост боли слишком рано после дозы
```java
if (vasChange >= 2 && hoursSinceLastDose < 4) {
    escalationPriority = currentVas >= 6 ? "HIGH" : "MEDIUM";
    escalationReason = "Pain increased by 2+ points only X hours after last dose";
    recommendations = "Current pain management protocol may be insufficient. Consider dose adjustment.";
}
```

#### Сценарий 3: Высокий уровень боли с растущим трендом
```java
if (currentVas >= 6 && painTrend == "INCREASING") {
    escalationPriority = "MEDIUM";
    escalationReason = "High pain level with increasing trend";
    recommendations = "Monitor closely. Consider proactive pain management adjustment.";
}
```

### 2. `canAdministerNextDose(String mrn)`
Проверяет, можно ли ввести следующую дозу (прошло ли 4+ часа).

### 3. `registerDoseAdministration(DoseAdministration dose)`
Регистрирует введение дозы препарата для последующего анализа.

### 4. `analyzePainTrend(String mrn)`
Анализирует тренд боли за последние 24 часа.

### 5. `handleNewVasRecord(String mrn, Integer vasLevel)`
**Автоматически вызывается** при создании нового VAS:
- Проверяет необходимость эскалации
- Создает эскалацию при необходимости
- Публикует событие `EscalationCreatedEvent`

---

## 🔗 ИНТЕГРАЦИЯ

### 1. **NurseServiceImpl.createVAS()**
```java
@Transactional
public VasDTO createVAS(String mrn, VasDTO vasDto) {
    // ... сохранение VAS ...
    
    // 🔥 АВТОМАТИЧЕСКАЯ ПРОВЕРКА ЭСКАЛАЦИИ
    painEscalationService.handleNewVasRecord(mrn, vas.getPainLevel());
    
    return modelMapper.map(vas, VasDTO.class);
}
```

### 2. **ExternalVasIntegrationService.processExternalVasRecord()**
```java
@Transactional
public Long processExternalVasRecord(ExternalVasRecordRequest externalVas) {
    // ... сохранение VAS ...
    
    // 🔥 АВТОМАТИЧЕСКАЯ ПРОВЕРКА ЭСКАЛАЦИИ
    painEscalationService.handleNewVasRecord(patient.getMrn(), externalVas.getVasLevel());
    
    // ... генерация рекомендации ...
    return savedVas.getId();
}
```

---

## 📊 СОБЫТИЯ АНАЛИТИКИ

При создании эскалации публикуется событие:

```java
EscalationCreatedEvent(
    source = PainEscalationServiceImpl,
    escalationId = 123,
    patientMrn = "EMR-A1B2C3D4",
    priority = "CRITICAL",
    reason = "Critical pain level: VAS 9",
    vasLevel = 9,
    createdBy = "PAIN_ESCALATION_SERVICE",
    createdAt = LocalDateTime.now()
)
```

Событие сохраняется в MongoDB для аналитики.

---

## 🧪 ТЕСТИРОВАНИЕ

### Тест 1: Критический уровень боли
```bash
# 1. Создать пациента
POST /api/nurse/patients
{
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "1980-01-01",
  "gender": "MALE"
}

# 2. Создать EMR
POST /api/nurse/patients/{mrn}/emr
{
  "gfr": 90,
  "weight": 70,
  "height": 175
}

# 3. Создать VAS = 9 (критический)
POST /api/nurse/patients/{mrn}/vas
{
  "painLevel": 9,
  "painPlace": "Lower back"
}

# ✅ Ожидаемый результат: Автоматически создана эскалация с приоритетом CRITICAL
```

### Тест 2: Рост боли после дозы
```bash
# 1. Зарегистрировать дозу
POST /api/pain-escalation/doses
{
  "patientMrn": "000001",
  "drugName": "Morphine",
  "dosage": "10mg IV",
  "route": "INTRAVENOUS",
  "administeredBy": "nurse_id",
  "vasBefore": 7
}

# 2. Через 2 часа создать VAS = 9
POST /api/nurse/patients/000001/vas
{
  "painLevel": 9,
  "painPlace": "Lower back"
}

# ✅ Ожидаемый результат: Эскалация с приоритетом HIGH
# Причина: "Pain increased by 2 points only 2 hours after last dose (minimum interval: 4 hours)"
```

### Тест 3: Проверка интервала между дозами
```bash
# Проверить, можно ли дать следующую дозу
GET /api/pain-escalation/can-administer-dose?mrn=000001

# Ответ:
{
  "canAdminister": false,
  "hoursSinceLastDose": 2,
  "requiredInterval": 4,
  "message": "Next dose can be administered in 2 hours"
}
```

### Тест 4: Анализ тренда боли
```bash
# Получить анализ тренда боли за последние 24 часа
GET /api/pain-escalation/pain-trend?mrn=000001

# Ответ:
{
  "patientMrn": "000001",
  "currentVas": 7,
  "previousVas": 5,
  "vasChange": 2,
  "painTrend": "INCREASING",
  "averageVas": 6.2,
  "maxVas": 8,
  "minVas": 4,
  "vasRecordCount": 5,
  "vasHistory": [7, 6, 8, 5, 4]
}
```

---

## ⚙️ НАСТРОЙКА

### Изменение пороговых значений

Отредактируйте `application.properties`:

```properties
# Более строгие критерии эскалации
pain.escalation.min-vas-increase=1              # Эскалация при росте на 1 балл
pain.escalation.min-dose-interval-hours=6       # Интервал 6 часов
pain.escalation.critical-vas-level=7            # Критический VAS = 7
pain.escalation.high-vas-level=5                # Высокий VAS = 5

# Более длительный период анализа
pain.escalation.trend-analysis-period-hours=48  # Анализ за 48 часов
```

---

## 📈 МЕТРИКИ И МОНИТОРИНГ

### Логирование
Все операции логируются с уровнем INFO/WARN:

```
INFO  - Checking pain escalation for patient: EMR-A1B2C3D4
WARN  - Escalation required for patient EMR-A1B2C3D4: Critical pain level: VAS 9
INFO  - Escalation created: id=123, priority=CRITICAL, reason=Critical pain level: VAS 9
INFO  - Notification should be sent to doctor about escalation for patient EMR-A1B2C3D4
```

### Аналитика в MongoDB
Все эскалации сохраняются в коллекцию `analytics_events`:

```json
{
  "eventType": "ESCALATION_CREATED",
  "escalationId": 123,
  "patientMrn": "EMR-A1B2C3D4",
  "priority": "CRITICAL",
  "reason": "Critical pain level: VAS 9",
  "vasLevel": 9,
  "createdBy": "PAIN_ESCALATION_SERVICE",
  "createdAt": "2025-10-21T18:30:00"
}
```

---

## 🔒 БЕЗОПАСНОСТЬ

- **Транзакционность:** Все операции выполняются в транзакциях
- **Валидация:** Проверка существования пациента перед операциями
- **Обработка ошибок:** Graceful handling с логированием
- **Асинхронность:** События публикуются асинхронно (@Async)

---

## 🔔 WEBSOCKET УВЕДОМЛЕНИЯ

### Доступные топики

**1. `/topic/escalations/doctors`** - все эскалации для врачей
**2. `/topic/escalations/anesthesiologists`** - эскалации для анестезиологов
**3. `/topic/escalations/dashboard`** - мониторинг для dashboard
**4. `/topic/escalations/critical`** - только критические эскалации (VAS >= 8)
**5. `/topic/escalations/status-updates`** - обновления статусов эскалаций
**6. `/queue/escalations`** - персональные уведомления врачу

### Подключение к WebSocket

```javascript
const socket = new SockJS('http://localhost:8080/ws-notifications');
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
    // Подписка на эскалации для врачей
    stompClient.subscribe('/topic/escalations/doctors', (message) => {
        const escalation = JSON.parse(message.body);
        console.log('New escalation:', escalation);
        showNotification(escalation);
    });
    
    // Подписка на критические эскалации
    stompClient.subscribe('/topic/escalations/critical', (message) => {
        const critical = JSON.parse(message.body);
        showCriticalAlert(critical);
    });
});
```

### Формат уведомления

```json
{
  "escalationId": 123,
  "recommendationId": 456,
  "patientMrn": "EMR-A1B2C3D4",
  "patientName": "John Doe",
  "currentVas": 9,
  "previousVas": 6,
  "vasChange": 3,
  "escalationReason": "Critical pain level: VAS 9",
  "priority": "CRITICAL",
  "recommendations": "URGENT: Immediate intervention required",
  "createdAt": "2025-10-22T15:30:00",
  "latestDiagnoses": ["M54.5 - Low back pain"]
}
```

---

## ⏰ АВТОМАТИЧЕСКИЙ МОНИТОРИНГ

### PainMonitoringScheduler

**1. Мониторинг пациентов с высоким уровнем боли**
- Частота: каждые 15 минут
- Проверяет пациентов с VAS >= 6
- Анализирует только недавние записи (последние 2 часа)
- Автоматически создает эскалации при необходимости

**2. Проверка просроченных доз**
- Частота: каждый час
- Находит пациентов с VAS >= 5 и записью старше 6 часов
- Проверяет возможность введения следующей дозы
- Логирует пациентов, нуждающихся во внимании

**3. Ежедневная сводка**
- Частота: каждый день в 08:00
- Статистика эскалаций за последние 24 часа
- Количество критических эскалаций
- Количество нерешенных эскалаций

### Пример логов

```
INFO  - Starting automatic pain monitoring check...
DEBUG - Checking patient EMR-A1B2C3D4 with VAS 7
WARN  - Scheduled check found escalation needed for patient EMR-A1B2C3D4: High pain level with increasing trend
INFO  - Pain monitoring check completed. Checked: 15, Escalations created: 3

INFO  - === DAILY ESCALATION SUMMARY ===
INFO  - Escalations in last 24h: 12
INFO  - Critical escalations: 3
INFO  - Currently pending: 5
INFO  - ================================
```

---

## 🎯 REST API ENDPOINTS

### Регистрация дозы
```http
POST /api/pain-escalation/patients/{mrn}/doses
Content-Type: application/json

{
  "drugName": "Morphine",
  "dosage": "10mg",
  "route": "IV",
  "administeredBy": "nurse_123",
  "vasBefore": 8,
  "vasAfter": 4,
  "recommendationId": 456,
  "notes": "Patient responded well"
}
```

### Проверка доступности дозы
```http
GET /api/pain-escalation/patients/{mrn}/can-administer-next-dose

Response:
{
  "patientMrn": "EMR-A1B2C3D4",
  "canAdminister": true,
  "hoursSinceLastDose": 5,
  "requiredInterval": 4,
  "message": "Can administer next dose. 5 hours passed since last dose."
}
```

### Анализ тренда боли
```http
GET /api/pain-escalation/patients/{mrn}/trend

Response:
{
  "patientMrn": "EMR-A1B2C3D4",
  "currentVas": 7,
  "previousVas": 5,
  "vasChange": 2,
  "painTrend": "INCREASING",
  "averageVas": 6.2,
  "maxVas": 8,
  "minVas": 4,
  "vasRecordCount": 5,
  "vasHistory": [7, 6, 8, 5, 4]
}
```

### Принудительная проверка эскалации
```http
POST /api/pain-escalation/patients/{mrn}/check
Content-Type: application/json

{
  "vasLevelOverride": 8
}
```

### Получить последние эскалации
```http
GET /api/pain-escalation/escalations/recent?limit=20
```

### Получить эскалацию по ID
```http
GET /api/pain-escalation/escalations/{id}
```

---

## 🚀 БУДУЩИЕ УЛУЧШЕНИЯ

1. ~~**REST API контроллер** для ручного управления дозами~~ ✅ Реализовано
2. ~~**WebSocket уведомления** врачам о критических эскалациях~~ ✅ Реализовано
3. ~~**Автоматический мониторинг** пациентов~~ ✅ Реализовано
4. **Machine Learning** для предсказания роста боли
5. **Интеграция с системой назначений** для автоматического учета доз
6. **Dashboard** для визуализации трендов боли
7. **Email уведомления** при критических эскалациях

---

## 📚 СВЯЗАННАЯ ДОКУМЕНТАЦИЯ

- [Модуль Аналитики](ANALYTICS_MODULE_README.md)
- [VAS Внешняя Интеграция](VAS_EXTERNAL_INTEGRATION_README.md)
- [Workflow README](../WORKFLOW_README.md)
- [Нереализованные функции](UNIMPLEMENTED_FEATURES.md)

---

## 🔧 КОНФИГУРАЦИЯ

### application.properties

```properties
# Pain Escalation Configuration
pain.escalation.min-vas-increase=2
pain.escalation.min-dose-interval-hours=4
pain.escalation.critical-vas-level=8
pain.escalation.high-vas-level=6
pain.escalation.trend-analysis-period-hours=24
pain.escalation.max-escalations-per-period=3

# Scheduler Configuration (опционально)
spring.task.scheduling.pool.size=5
spring.task.scheduling.thread-name-prefix=pain-scheduler-
```

---

**Автор:** Pain Management Team  
**Дата последнего обновления:** 22.10.2025
