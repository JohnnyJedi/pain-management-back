# 🔄 Автоматический Пересчет EMR - Полная Реализация

**Дата реализации:** 22.10.2025  
**Статус:** ✅ Полностью реализовано  
**Версия:** 1.0.0

---

## 📋 ОБЗОР

Реализована **полная система автоматического пересчета рекомендаций** при критических изменениях EMR (Electronic Medical Record).

### ✅ Что реализовано:

1. **Обнаружение критических изменений EMR**
   - Автоматическое сравнение старых и новых значений
   - Определение критических порогов
   - Генерация алертов

2. **Автоматический пересчет рекомендаций**
   - Маркировка старых рекомендаций как требующих пересмотра
   - Генерация новых рекомендаций с учетом обновленных данных
   - Сохранение истории изменений

3. **Уведомления врачам**
   - WebSocket real-time уведомления
   - Детальная информация об изменениях
   - Рекомендации по действиям

4. **Интеграция с синхронизацией EMR**
   - Автоматический запуск при синхронизации из FHIR
   - Обработка ручных обновлений EMR
   - Транзакционность и надежность

---

## 🏗️ АРХИТЕКТУРА

```
emr_recalculation/
└── service/
    └── EmrRecalculationService.java         # Основной сервис пересчета

external_emr_integration_service/service/
├── EmrSyncScheduler.java                    # Интегрирован с пересчетом
└── EmrChangeDetectionService.java           # Обнаружение изменений

common/patients/entity/
└── Recommendation.java                       # Добавлены поля для пересмотра

enums/
└── RecommendationStatus.java                 # Добавлен статус REQUIRES_REVIEW
```

---

## 🔍 КРИТИЧЕСКИЕ ПОРОГИ EMR

### Параметры, которые отслеживаются:

| Параметр | Критический порог | Описание |
|----------|-------------------|----------|
| **GFR** | < 30 или падение > 20 | Тяжелая почечная недостаточность |
| **PLT** | < 50 | Критическая тромбоцитопения (риск кровотечения) |
| **WBC** | < 2.0 | Тяжелая лейкопения (иммунодефицит) |
| **SAT** | < 90 | Критическая гипоксия |
| **Натрий** | < 125 или > 155 | Опасный электролитный дисбаланс |
| **Вес** | Изменение > 10% | Значительное изменение веса |
| **Child-Pugh** | Ухудшение класса | Прогрессирование печеночной недостаточности |

---

## ⚙️ АЛГОРИТМ РАБОТЫ

### Шаг 1: Обнаружение изменений

```java
// В EmrSyncScheduler после обновления EMR
boolean hasChanges = emrChangeDetectionService.detectChanges(oldEmr, newEmr);

if (hasChanges) {
    List<EmrChangeAlertDTO> criticalAlerts = 
        emrChangeDetectionService.checkCriticalChanges(oldEmr, newEmr, mrn);
}
```

### Шаг 2: Запуск пересчета

```java
// Если есть критические изменения
if (!criticalAlerts.isEmpty()) {
    emrRecalculationService.handleEmrChange(patient, oldEmr, newEmr);
}
```

### Шаг 3: Маркировка старых рекомендаций

```java
// Найти все активные рекомендации
List<Recommendation> activeRecommendations = 
    recommendationRepository.findByPatientMrnAndStatus(mrn, RecommendationStatus.APPROVED);

// Пометить как требующие пересмотра
for (Recommendation rec : activeRecommendations) {
    rec.setStatus(RecommendationStatus.REQUIRES_REVIEW);
    rec.setReviewReason("GFR: 45 → 25 (Тяжелая почечная недостаточность)");
    rec.setReviewRequestedAt(LocalDateTime.now());
    recommendationRepository.save(rec);
}
```

### Шаг 4: Генерация новых рекомендаций

```java
// Получить последний VAS
Integer lastVasLevel = patient.getVas().getLast().getPainLevel();

// Сгенерировать новую рекомендацию с учетом обновленных данных EMR
Recommendation newRecommendation = 
    treatmentProtocolService.generateRecommendation(patient, lastVasLevel);

// Добавить примечание о причине пересчета
String note = "Рекомендация пересчитана автоматически из-за критических изменений EMR: " +
              "GFR: 45 → 25";
newRecommendation.setJustification(newRecommendation.getJustification() + "\n\n" + note);
newRecommendation.setStatus(RecommendationStatus.PENDING);

recommendationRepository.save(newRecommendation);
```

### Шаг 5: Отправка уведомлений

```java
// WebSocket уведомления о критических изменениях EMR
UnifiedNotificationDTO emrAlert = UnifiedNotificationDTO.builder()
    .type(NotificationType.EMR_ALERT)
    .priority(NotificationPriority.CRITICAL)
    .title("Критическое изменение EMR: GFR")
    .message("GFR изменился с 45 на 25")
    .build();

unifiedNotificationService.sendCriticalNotification(emrAlert);

// Уведомление о новой рекомендации
UnifiedNotificationDTO recAlert = UnifiedNotificationDTO.builder()
    .type(NotificationType.RECOMMENDATION_UPDATE)
    .priority(NotificationPriority.HIGH)
    .title("Новая рекомендация после изменения EMR")
    .message("Сгенерирована новая рекомендация #" + newRecommendation.getId())
    .build();

unifiedNotificationService.sendNotification(recAlert);
```

---

## 📦 НОВЫЕ ПОЛЯ В RECOMMENDATION

### Добавленные поля для поддержки пересмотра:

```java
@Entity
@Table(name = "recommendation")
public class Recommendation {
    // ... существующие поля ...
    
    // ========== EMR RECALCULATION FIELDS ========== //
    
    @Column(name = "review_reason", length = 2000)
    private String reviewReason;  // Причина необходимости пересмотра
    
    @Column(name = "review_requested_at")
    private LocalDateTime reviewRequestedAt;  // Когда запрошен пересмотр
    
    @Column(name = "reviewed_by", length = 50)
    private String reviewedBy;  // Кто пересмотрел
    
    @Column(name = "reviewed_at")
    private LocalDateTime reviewedAt;  // Когда пересмотрено
    
    @Column(name = "description", length = 5000, columnDefinition = "TEXT")
    private String description;  // Описание рекомендации
    
    @Column(name = "justification", length = 5000, columnDefinition = "TEXT")
    private String justification;  // Обоснование рекомендации
}
```

### Новые статусы рекомендаций:

```java
public enum RecommendationStatus {
    PENDING,                          // Ожидает одобрения врача
    APPROVED_BY_DOCTOR,              // Одобрено врачом
    REJECTED_BY_DOCTOR,              // Отклонено врачом
    ESCALATED_TO_ANESTHESIOLOGIST,   // Эскалировано анестезиологу
    APPROVED_BY_ANESTHESIOLOGIST,    // Одобрено анестезиологом
    REJECTED_BY_ANESTHESIOLOGIST,    // Отклонено анестезиологом
    FINAL_APPROVED,                  // Финальное одобрение
    CANCELLED,                       // Отменено
    REQUIRES_REVIEW,                 // 🆕 Требует пересмотра из-за изменений EMR
    APPROVED                         // 🆕 Одобрено (общий статус)
}
```

---

## 🔗 ИНТЕГРАЦИЯ

### 1. Автоматическая синхронизация EMR (каждые 6 часов)

```java
// В EmrSyncScheduler.syncSinglePatient()
@Scheduled(cron = "0 0 */6 * * *")
public EmrSyncResultDTO syncAllFhirPatients() {
    // ... синхронизация EMR из FHIR ...
    
    // Сохраняем обновленный EMR
    emrRepository.save(updatedEmr);
    
    // АВТОМАТИЧЕСКИЙ ПЕРЕСЧЕТ
    emrRecalculationService.handleEmrChange(
        currentEmr.getPatient(), 
        currentEmr, 
        updatedEmr
    );
}
```

**Результат:**
- Каждые 6 часов синхронизируются данные из FHIR
- При критических изменениях автоматически пересчитываются рекомендации
- Врачи получают WebSocket уведомления

### 2. Ручное обновление EMR через API

```java
// В NurseServiceImpl.updateEmr()
@Transactional
public EmrDTO updateEmr(String mrn, EmrDTO emrDto) {
    Patient patient = patientRepository.findByMrn(mrn)
        .orElseThrow(() -> new NotFoundException("Patient not found"));
    
    Emr oldEmr = patient.getEmr().getLast();
    
    // Обновляем EMR
    Emr updatedEmr = updateEmrFromDto(oldEmr, emrDto);
    emrRepository.save(updatedEmr);
    
    // АВТОМАТИЧЕСКИЙ ПЕРЕСЧЕТ
    emrRecalculationService.handleEmrChange(patient, oldEmr, updatedEmr);
    
    return modelMapper.map(updatedEmr, EmrDTO.class);
}
```

**Результат:**
- При ручном обновлении EMR медсестрой
- Автоматически проверяются критические изменения
- При необходимости пересчитываются рекомендации

---

## 📊 ПРИМЕРЫ СЦЕНАРИЕВ

### Сценарий 1: Критическое падение GFR

**Исходная ситуация:**
- Пациент: Иван Иванов (MRN: EMR-12345)
- Текущий GFR: 45
- Активная рекомендация: Tramadol 100mg PO q6h PRN

**Событие:**
- Синхронизация EMR из FHIR
- Новый GFR: 25 (падение на 20 единиц)

**Автоматические действия:**

1. **Обнаружение критического изменения**
   ```
   WARN - CRITICAL EMR changes detected for patient EMR-12345: 1 alerts
   WARN - GFR: 45 → 25 (Kidney function critically decreased)
   ```

2. **Маркировка старой рекомендации**
   ```
   INFO - Recommendation #456 marked for review: GFR: 45 → 25 (Тяжелая почечная недостаточность)
   ```

3. **Генерация новой рекомендации**
   ```
   INFO - Generating updated recommendations for patient EMR-12345 with new EMR data
   INFO - New recommendation #789 generated for patient EMR-12345 due to EMR changes
   ```
   
   **Новая рекомендация:**
   - Tramadol 50mg PO q6h PRN (снижено из-за GFR < 30)
   - Justification: "Дозировка скорректирована с учетом тяжелой почечной недостаточности (GFR 25)"

4. **Отправка уведомлений**
   - EMR Alert (CRITICAL): "GFR изменился с 45 на 25"
   - Recommendation Update (HIGH): "Новая рекомендация #789 после изменения EMR"

**Результат:**
- Врач получает real-time уведомление
- Старая рекомендация помечена как REQUIRES_REVIEW
- Новая рекомендация со скорректированной дозировкой в статусе PENDING
- Врач может одобрить новую рекомендацию

### Сценарий 2: Критическая тромбоцитопения

**Исходная ситуация:**
- Пациент: Мария Петрова (MRN: EMR-67890)
- Текущий PLT: 120
- Активная рекомендация: Ketorolac 30mg IV q6h PRN

**Событие:**
- Ручное обновление EMR медсестрой
- Новый PLT: 45 (критическая тромбоцитопения)

**Автоматические действия:**

1. **Обнаружение критического изменения**
   ```
   WARN - CRITICAL EMR changes detected for patient EMR-67890: 1 alerts
   WARN - PLT: 120 → 45 (Critically low platelets - bleeding risk)
   ```

2. **Маркировка старой рекомендации**
   ```
   INFO - Recommendation #234 marked for review: PLT: 120 → 45 (Критическая тромбоцитопения - риск кровотечения)
   ```

3. **Генерация новой рекомендации**
   ```
   INFO - New recommendation #567 generated for patient EMR-67890 due to EMR changes
   ```
   
   **Новая рекомендация:**
   - Tramadol 50mg PO q6h PRN (Ketorolac противопоказан из-за PLT < 50)
   - Justification: "НПВС противопоказаны при тромбоцитопении < 50. Рекомендован опиоидный анальгетик."

4. **Отправка уведомлений**
   - EMR Alert (CRITICAL): "PLT изменился с 120 на 45 - риск кровотечения"
   - Recommendation Update (HIGH): "Новая рекомендация #567 - НПВС противопоказаны"

**Результат:**
- Врач немедленно узнает о критическом изменении
- Опасный препарат (Ketorolac) автоматически заменен на безопасный
- Снижен риск кровотечения

### Сценарий 3: Множественные критические изменения

**Исходная ситуация:**
- Пациент: Петр Сидоров (MRN: EMR-11111)
- Текущий GFR: 50, PLT: 100, WBC: 5.0
- Активные рекомендации: Morphine 10mg IV q4h PRN, Ketorolac 30mg IV q6h PRN

**Событие:**
- Синхронизация EMR из FHIR
- Новый GFR: 20, PLT: 40, WBC: 1.5

**Автоматические действия:**

1. **Обнаружение множественных критических изменений**
   ```
   WARN - CRITICAL EMR changes detected for patient EMR-11111: 3 alerts
   WARN - GFR: 50 → 20 (Kidney function critically decreased)
   WARN - PLT: 100 → 40 (Critically low platelets - bleeding risk)
   WARN - WBC: 5.0 → 1.5 (Critically low white blood cells - immunodeficiency)
   ```

2. **Маркировка всех активных рекомендаций**
   ```
   INFO - Recommendation #111 marked for review: GFR: 50 → 20; PLT: 100 → 40; WBC: 5.0 → 1.5
   INFO - Recommendation #222 marked for review: GFR: 50 → 20; PLT: 100 → 40; WBC: 5.0 → 1.5
   ```

3. **Генерация новых рекомендаций**
   ```
   INFO - New recommendation #333 generated for patient EMR-11111 due to EMR changes
   ```
   
   **Новая рекомендация:**
   - Tramadol 25mg PO q8h PRN (снижено из-за GFR < 30)
   - Justification: "Дозировка скорректирована с учетом тяжелой почечной недостаточности (GFR 20). НПВС противопоказаны из-за PLT < 50. Требуется осторожность из-за иммунодефицита (WBC 1.5)."

4. **Отправка множественных уведомлений**
   - 3 EMR Alerts (CRITICAL) для каждого параметра
   - 1 Recommendation Update (HIGH) о новой рекомендации

**Результат:**
- Врач получает полную картину критических изменений
- Все опасные препараты заменены на безопасные
- Дозировки скорректированы с учетом всех изменений

---

## 🧪 ТЕСТИРОВАНИЕ

### Тест 1: Ручное обновление EMR с критическим изменением

```bash
# 1. Получить текущий EMR пациента
GET /api/nurse/patients/EMR-12345/emr

# Текущий GFR: 45

# 2. Обновить EMR с критическим GFR
PUT /api/nurse/patients/EMR-12345/emr
Content-Type: application/json

{
  "gfr": "25",
  "plt": "120",
  "wbc": "5.0",
  "sodium": "140",
  "sat": "95",
  "weight": "70",
  "height": "175",
  "childPughScore": "A"
}

# ✅ Ожидаемый результат:
# - EMR обновлен
# - Обнаружено критическое изменение GFR: 45 → 25
# - Старые рекомендации помечены как REQUIRES_REVIEW
# - Сгенерирована новая рекомендация со скорректированной дозировкой
# - Отправлены WebSocket уведомления
```

### Тест 2: Автоматическая синхронизация EMR

```bash
# Запустить синхронизацию вручную (или дождаться автоматической каждые 6 часов)
POST /api/emr-integration/sync

# ✅ Ожидаемый результат:
# - Синхронизированы данные из FHIR для всех пациентов
# - Обнаружены критические изменения
# - Пересчитаны рекомендации
# - Отправлены уведомления
```

### Тест 3: Проверка статуса рекомендаций

```bash
# Получить все рекомендации пациента
GET /api/doctor/patients/EMR-12345/recommendations

# Проверить статусы:
# - Старые рекомендации: status = "REQUIRES_REVIEW"
# - Новые рекомендации: status = "PENDING"
# - reviewReason заполнен для старых рекомендаций
```

---

## 📊 ЛОГИРОВАНИЕ

### Примеры логов

```
INFO  - Handling EMR change for patient: EMR-12345
INFO  - EMR changes detected for patient EMR-12345
WARN  - CRITICAL EMR changes detected for patient EMR-12345: 1 alerts
INFO  - Processing 1 critical EMR changes for patient EMR-12345
INFO  - Found 2 active recommendations for patient EMR-12345
INFO  - Recommendation 456 marked for review: GFR: 45 → 25 (Тяжелая почечная недостаточность)
INFO  - Recommendation 457 marked for review: GFR: 45 → 25 (Тяжелая почечная недостаточность)
INFO  - Generating updated recommendations for patient EMR-12345 with new EMR data
INFO  - New recommendation 789 generated for patient EMR-12345 due to EMR changes
INFO  - EMR change notifications sent for patient EMR-12345
INFO  - New recommendation notification sent for patient EMR-12345
INFO  - Critical EMR changes processed successfully for patient EMR-12345
```

---

## ✅ ЧЕКЛИСТ РЕАЛИЗАЦИИ

- [x] EmrRecalculationService создан
- [x] Обнаружение критических изменений EMR
- [x] Маркировка старых рекомендаций как REQUIRES_REVIEW
- [x] Генерация новых рекомендаций с учетом обновленных данных
- [x] Отправка WebSocket уведомлений
- [x] Интеграция с EmrSyncScheduler
- [x] Поддержка ручного обновления EMR
- [x] Новые поля в Recommendation entity
- [x] Новые статусы рекомендаций
- [x] Транзакционность и обработка ошибок
- [x] Логирование всех операций
- [x] Документация

---

## 🚀 ГОТОВО К ИСПОЛЬЗОВАНИЮ

Система автоматического пересчета EMR полностью реализована и готова к использованию:

1. ✅ Автоматически работает при синхронизации EMR (каждые 6 часов)
2. ✅ Автоматически работает при ручном обновлении EMR
3. ✅ Отправляет real-time уведомления врачам
4. ✅ Генерирует безопасные рекомендации с учетом новых данных
5. ✅ Сохраняет историю изменений

---

## 📚 СВЯЗАННАЯ ДОКУМЕНТАЦИЯ

- [WebSocket Real-Time Notifications](WEBSOCKET_REALTIME_NOTIFICATIONS.md)
- [Pain Escalation Module](PAIN_ESCALATION_MODULE.md)
- [EMR Sync Scheduler](../src/main/java/pain_helper_back/external_emr_integration_service/service/EmrSyncScheduler.java)

---

**Автор:** Pain Management Team  
**Дата:** 22.10.2025  
**Статус:** ✅ Полностью реализовано
