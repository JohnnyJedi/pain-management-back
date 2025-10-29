# 🔗 Полное руководство: Analytics, Performance SLA Monitoring и Reporting

**Дата:** 29.10.2025  
**Версия:** 1.0  
**Автор:** Nick  
**Цель:** Объяснить различия, взаимосвязи и взаимодействие трех модулей мониторинга

---

## 📋 СОДЕРЖАНИЕ

1. [Краткий обзор модулей](#краткий-обзор-модулей)
2. [Ключевые различия](#ключевые-различия)
3. [Архитектура взаимодействия](#архитектура-взаимодействия)
4. [Поток данных](#поток-данных)
5. [Когда использовать какой модуль](#когда-использовать-какой-модуль)
6. [Соответствие требованиям SRS](#соответствие-требованиям-srs)
7. [FAQ](#faq)

---

## 🎯 КРАТКИЙ ОБЗОР МОДУЛЕЙ

### 1️⃣ Analytics Module
**Назначение:** Бизнес-аналитика и аудит действий  
**Вопрос:** ЧТО произошло в системе?  
**Данные:** Бизнес-события (одобрения, эскалации, регистрации)  
**Хранилище:** MongoDB (analytics_events)  
**Retention:** 30 дней  
**Пользователи:** Admin, Management

### 2️⃣ Performance SLA Monitoring
**Назначение:** Технический мониторинг производительности  
**Вопрос:** КАК БЫСТРО работает система?  
**Данные:** Метрики времени выполнения операций  
**Хранилище:** MongoDB (performance_metrics)  
**Retention:** 30 дней  
**Пользователи:** Admin, DevOps, SRE

### 3️⃣ Reporting Module
**Назначение:** Долгосрочная отчетность и экспорт  
**Вопрос:** КАКИЕ ТРЕНДЫ за период?  
**Данные:** Агрегированные метрики из Analytics  
**Хранилище:** PostgreSQL (daily_report_aggregates)  
**Retention:** Бессрочно  
**Пользователи:** Admin, Management, Executives

---

## 🔍 КЛЮЧЕВЫЕ РАЗЛИЧИЯ

| Аспект | Analytics | Performance SLA | Reporting |
|--------|-----------|-----------------|-----------|
| **Тип данных** | Бизнес-события | Метрики производительности | Агрегированная статистика |
| **Детализация** | Каждое событие | Каждый вызов метода | Ежедневные суммы |
| **Хранилище** | MongoDB | MongoDB | PostgreSQL |
| **Retention** | 30 дней | 30 дней | Бессрочно |
| **Сбор данных** | Spring Events | AOP Aspect | Агрегация из Analytics |
| **Real-time** | ✅ Да | ✅ Да | ❌ Нет (ежедневно) |
| **Экспорт** | ❌ Нет | ❌ Нет | ✅ Да (Excel, PDF) |
| **Email** | ❌ Нет | ❌ Нет | ✅ Да |
| **SLA контроль** | ❌ Нет | ✅ Да | ❌ Нет |
| **Бизнес-метрики** | ✅ Да | ❌ Нет | ✅ Да (агрегированные) |

---

## 🏗️ АРХИТЕКТУРА ВЗАИМОДЕЙСТВИЯ

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PAIN MANAGEMENT SYSTEM                         │
│                    (Services & Controllers)                         │
└─────────────────────────────────────────────────────────────────────┘
                    ↓                              ↓
        ┌───────────────────────┐    ┌───────────────────────┐
        │   Spring Events       │    │   AOP Aspect          │
        │   (Business Actions)  │    │   (Method Calls)      │
        └───────────────────────┘    └───────────────────────┘
                    ↓                              ↓
        ┌───────────────────────┐    ┌───────────────────────┐
        │  ANALYTICS MODULE     │    │  PERFORMANCE SLA      │
        │  (Business Events)    │    │  (Time Metrics)       │
        └───────────────────────┘    └───────────────────────┘
                    ↓                              ↓
        ┌───────────────────────┐    ┌───────────────────────┐
        │  MongoDB              │    │  MongoDB              │
        │  analytics_events     │    │  performance_metrics  │
        │  (30 days)            │    │  (30 days)            │
        └───────────────────────┘    └───────────────────────┘
                    ↓                              
        ┌───────────────────────────────────────────────────┐
        │  REPORTING MODULE                                 │
        │  (Daily Aggregation @ 00:30)                      │
        └───────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────┐
        │  PostgreSQL           │
        │  daily_report_        │
        │  aggregates           │
        │  (Forever)            │
        └───────────────────────┘
                    ↓
        ┌───────────────────────────────────────────────────┐
        │  EXPORT & EMAIL                                   │
        │  - Excel (.xlsx)                                  │
        │  - PDF                                            │
        │  - Email with attachments                         │
        └───────────────────────────────────────────────────┘
```

---

## 🔄 ПОТОК ДАННЫХ

### Сценарий: Врач одобряет рекомендацию

#### Шаг 1: Бизнес-действие в DoctorService
```java
public void approveRecommendation(Long id, RecommendationApprovalDTO dto) {
    recommendation.setStatus(RecommendationStatus.APPROVED);
    recommendationRepository.save(recommendation);
    
    // Публикация события для Analytics
    eventPublisher.publishEvent(new RecommendationApprovedEvent(...));
}
```

#### Шаг 2: Analytics Module (асинхронно)
Сохраняет в MongoDB:
```json
{
  "eventType": "RECOMMENDATION_APPROVED",
  "userId": "doctor_123",
  "status": "APPROVED",
  "metadata": {"comment": "Good choice"}
}
```

#### Шаг 3: Performance SLA (автоматически через AOP)
Сохраняет в MongoDB:
```json
{
  "operationName": "recommendation.approve",
  "executionTimeMs": 850,
  "slaThresholdMs": 1000,
  "slaViolated": false
}
```

#### Шаг 4: Reporting (ежедневно в 00:30)
Агрегирует и сохраняет в PostgreSQL:
```json
{
  "reportDate": "2025-10-29",
  "totalRecommendations": 45,
  "approvedRecommendations": 38,
  "approvalRate": 84.44
}
```

---

## 🎯 КОГДА ИСПОЛЬЗОВАТЬ КАКОЙ МОДУЛЬ?

### Analytics Module — для real-time бизнес-аналитики

✅ **Используйте когда нужно:**
- Узнать количество событий за период
- Проанализировать действия конкретного пользователя
- Получить детали события (комментарии, причины)
- Real-time мониторинг системы

**Примеры вопросов:**
- "Сколько рекомендаций одобрено сегодня?"
- "Что делал врач doctor_123?"
- "Последние 50 событий в системе"

**API:**
```http
GET /api/analytics/events/stats
GET /api/analytics/users/{userId}/activity
GET /api/analytics/events/recent?limit=50
```

---

### Performance SLA Monitoring — для технического мониторинга

✅ **Используйте когда нужно:**
- Проверить соблюдение SLA
- Найти медленные операции
- Получить метрики производительности (p95, p99)
- Мониторить техническое здоровье системы

**Примеры вопросов:**
- "Соблюдается ли SLA 2000ms для рекомендаций?"
- "Какие операции превышают SLA?"
- "Каков 95-й перцентиль времени ответа?"

**API:**
```http
GET /api/performance/statistics
GET /api/performance/sla-violations/recent
GET /api/performance/slowest?limit=10
```

---

### Reporting Module — для долгосрочной отчетности

✅ **Используйте когда нужно:**
- Получить статистику за длительный период
- Сгенерировать отчеты для руководства
- Экспортировать данные в Excel/PDF
- Отправить отчет по email
- Проанализировать тренды

**Примеры вопросов:**
- "Динамика одобрений за последний квартал?"
- "Нужен PDF отчет за октябрь"
- "Отправить ежемесячный отчет на email директора"

**API:**
```http
GET /api/reports/daily?startDate=...&endDate=...
GET /api/reports/export/excel?startDate=...&endDate=...
POST /api/reports/email/summary?email=admin@example.com
```

---

## 📋 СООТВЕТСТВИЕ ТРЕБОВАНИЯМ SRS

### SRS Section 3.5: Reporting

> "The system shall generate reports on usage and outcomes"

| Требование | Analytics | Performance | Reporting |
|------------|-----------|-------------|-----------|
| Usage data | ✅ Real-time | ❌ | ✅ Агрегированные |
| Outcome data | ✅ Детальные | ❌ | ✅ Суммарные |
| Performance metrics | ❌ | ✅ Детальные | ❌ |

**Вывод:** Все три модуля нужны для полного покрытия.

---

### SRS Section 3.5: KPI Tracking

> "Track Key Performance Indicators to measure effectiveness and time savings"

| KPI | Analytics | Performance | Reporting |
|-----|-----------|-------------|-----------|
| Approval rate | ✅ | ❌ | ✅ |
| Escalation rate | ✅ | ❌ | ✅ |
| Avg response time | ❌ | ✅ | ❌ |
| SLA compliance | ❌ | ✅ | ❌ |

**Вывод:** Analytics и Performance покрывают разные типы KPI.

---

### SRS Section 4.2: Performance

> "The system shall generate recommendations within [specific time frame]"

**Только Performance SLA Monitoring реализует это требование:**
- ✅ Мониторинг времени выполнения
- ✅ SLA контроль (2000ms для рекомендаций)
- ✅ Алерты при нарушениях

---

## ❓ FAQ

### Q1: Почему нельзя объединить Analytics и Performance?

**A:** Разные механизмы сбора данных:
- **Analytics** — Spring Events (публикуются вручную)
- **Performance** — AOP Aspect (автоматический перехват)

Невозможно измерить время выполнения через события.

---

### Q2: Зачем Reporting, если есть Analytics?

**A:** Reporting решает задачи, которые Analytics не может:
1. Долгосрочное хранение (Analytics — 30 дней)
2. Экспорт в Excel/PDF
3. Email рассылка
4. Предвычисленные метрики (быстрее)
5. Исторический анализ

---

### Q3: Дублируются ли данные?

**A:** ❌ НЕТ. Каждый модуль хранит РАЗНЫЕ данные:

**Analytics:**
```json
{"eventType": "RECOMMENDATION_APPROVED", "userId": "doctor_123"}
```

**Performance:**
```json
{"operationName": "recommendation.approve", "executionTimeMs": 850}
```

**Reporting:**
```json
{"reportDate": "2025-10-29", "totalRecommendations": 45}
```

---

### Q4: Какой модуль для Dashboard?

**A:** **ВСЕ ТРИ:**
- **Analytics** — real-time метрики
- **Performance** — технические метрики (SLA)
- **Reporting** — тренды (графики за месяц)

---

### Q5: Можно ли удалить один из модулей?

**A:** ❌ НЕТ. Каждый покрывает разные требования SRS:
- **Analytics** → SRS 3.5 (Usage data), 3.6 (Audit)
- **Performance** → SRS 4.2 (Performance monitoring)
- **Reporting** → SRS 3.5 (Long-term reporting)

---

## 📊 СРАВНИТЕЛЬНАЯ ТАБЛИЦА ДАННЫХ

| Информация | Analytics | Performance | Reporting |
|------------|-----------|-------------|-----------|
| Timestamp | ✅ Точное | ✅ Точное | ✅ Дата |
| User ID | ✅ | ✅ | ❌ |
| Event Type | ✅ | ❌ | ❌ |
| Operation Name | ❌ | ✅ | ❌ |
| Execution Time | ❌ | ✅ | ❌ |
| SLA Info | ❌ | ✅ | ❌ |
| Status | ✅ | ✅ | ❌ |
| Metadata | ✅ | ❌ | ❌ |
| Aggregates | ❌ | ❌ | ✅ |

---

## ✅ ЗАКЛЮЧЕНИЕ

### Три модуля — три разные задачи

1. **Analytics** = ЧТО произошло (бизнес-события)
2. **Performance** = КАК БЫСТРО (технические метрики)
3. **Reporting** = ТРЕНДЫ (долгосрочная статистика)

### Все три модуля необходимы

- ✅ Покрывают разные требования SRS
- ✅ Используют разные механизмы сбора
- ✅ Хранят разные типы данных
- ✅ Решают разные задачи
- ✅ Имеют разных пользователей

### Нет дублирования

Каждый модуль уникален и критически важен для полноценной работы системы.

---

**Документ подготовлен:** 29.10.2025  
**Автор:** Nick  
**Статус:** ✅ Готов для использования
