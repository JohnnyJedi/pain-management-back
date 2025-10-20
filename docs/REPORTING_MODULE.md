# Модуль Отчетности (Reporting Module)

## 📋 Содержание
1. [Обзор](#обзор)
2. [Архитектура](#архитектура)
3. [Поток данных](#поток-данных)
4. [Автоматические задачи](#автоматические-задачи)
5. [REST API эндпоинты](#rest-api-эндпоинты)
6. [Экспорт отчетов](#экспорт-отчетов)
7. [Email рассылка](#email-рассылка)

---

## 🎯 Обзор

Модуль отчетности предоставляет комплексные возможности аналитики и отчетности для системы управления болью. Модуль агрегирует данные из MongoDB (AnalyticsEvent) в PostgreSQL для долгосрочного хранения и генерирует отчеты в различных форматах.

### Ключевые возможности
- ✅ Автоматическая ежедневная агрегация данных (MongoDB → PostgreSQL)
- ✅ REST API для получения отчетов
- ✅ Экспорт в Excel (.xlsx) и PDF форматы
- ✅ Email уведомления с вложениями
- ✅ Запланированная очистка старых событий MongoDB (30-дневное хранение)

---

## 🏗️ Архитектура

### Структура компонентов

```
reporting/
├── entity/
│   ├── DailyReportAggregate.java      # Ежедневные агрегированные метрики
│   ├── WeeklyReportAggregate.java     # Еженедельные метрики (будущее)
│   └── MonthlyReportAggregate.java    # Ежемесячные метрики (будущее)
├── repository/
│   ├── DailyReportRepository.java
│   ├── WeeklyReportRepository.java
│   └── MonthlyReportRepository.java
├── service/
│   ├── DataAggregationService.java    # Основная логика агрегации
│   ├── ExcelExportService.java        # Генерация Excel
│   ├── PdfExportService.java          # Генерация PDF
│   └── EmailReportService.java        # Отправка email
└── controller/
    └── ReportController.java          # REST API эндпоинты
```

### Схема базы данных

**PostgreSQL (H2 для разработки):**
- `daily_report_aggregates` - Хранит ежедневные агрегированные метрики

**MongoDB:**
- `analytics_events` - Сырые данные событий (30-дневное хранение)

---

## 🔄 Поток данных

### 1. Сбор событий (Real-time)
```
Действие пользователя → AnalyticsEvent → MongoDB
```

**Типы событий:**
- `PATIENT_REGISTERED` - Регистрация пациента
- `VAS_RECORDED` - Запись уровня боли по шкале VAS
- `RECOMMENDATION_GENERATED` - Генерация рекомендации
- `RECOMMENDATION_APPROVED` - Одобрение рекомендации
- `RECOMMENDATION_REJECTED` - Отклонение рекомендации
- `ESCALATION_CREATED` - Создание эскалации
- `ESCALATION_RESOLVED` - Разрешение эскалации
- `USER_LOGIN_SUCCESS` - Успешный вход
- `USER_LOGIN_FAILED` - Неудачная попытка входа

### 2. Ежедневная агрегация (Запланированная)
```
MongoDB (AnalyticsEvent) → DataAggregationService → PostgreSQL (DailyReportAggregate)
```

**Расписание:** Каждый день в 00:30 (cron: `0 30 0 * * *`)

**Процесс:**
1. Извлечение событий из MongoDB за предыдущий день
2. Агрегация метрик по типам
3. Расчет статистики (средние значения, подсчеты, проценты)
4. Сохранение в PostgreSQL
5. Очистка старых событий MongoDB (>30 дней)

### 3. Генерация отчетов (По запросу)
```
API запрос → DailyReportRepository → Export Service → Ответ (JSON/Excel/PDF)
```

### 4. Доставка по Email (По запросу или по расписанию)
```
API запрос → EmailReportService → SMTP сервер → Получатель
```

---

## ⏰ Автоматические задачи

### Задача ежедневной агрегации
**Класс:** `DataAggregationService.aggregateDailyData()`  
**Расписание:** `@Scheduled(cron = "0 30 0 * * *")` - Каждый день в 00:30  
**Функция:**
- Агрегирует данные за вчерашний день
- Создает запись `DailyReportAggregate`
- Очищает события MongoDB старше 30 дней

**Конфигурация:**
```java
@EnableScheduling  // В AsyncConfig.java
```

---

## 🌐 REST API эндпоинты

### 1. Получение отчетов

#### Получить ежедневные отчеты за период
```http
GET /api/reports/daily?startDate=2025-10-01&endDate=2025-10-19
```

**Ответ:**
```json
[
  {
    "id": 1,
    "reportDate": "2025-10-18",
    "totalPatientsRegistered": 15,
    "totalVasRecords": 120,
    "averageVasLevel": 4.5,
    "criticalVasCount": 8,
    "totalRecommendations": 45,
    "approvedRecommendations": 38,
    "rejectedRecommendations": 7,
    "approvalRate": 84.44,
    "totalEscalations": 5,
    "resolvedEscalations": 3,
    "pendingEscalations": 2,
    "averageResolutionTimeHours": 2.5,
    "totalLogins": 67,
    "uniqueActiveUsers": 23,
    "createdAt": "2025-10-19T00:35:00"
  }
]
```

#### Получить отчет за конкретную дату
```http
GET /api/reports/daily/2025-10-18
```

#### Получить последние отчеты
```http
GET /api/reports/daily/recent?limit=7
```

#### Получить сводную статистику
```http
GET /api/reports/summary?startDate=2025-10-01&endDate=2025-10-19
```

**Ответ:**
```json
{
  "period": {
    "startDate": "2025-10-01",
    "endDate": "2025-10-19",
    "daysCount": 19
  },
  "patients": {
    "totalRegistered": 285,
    "totalVasRecords": 2280,
    "averageVasLevel": 4.32
  },
  "recommendations": {
    "total": 855,
    "approved": 722,
    "rejected": 133,
    "approvalRate": 84.44
  },
  "escalations": {
    "total": 95,
    "resolved": 57,
    "pending": 38
  },
  "users": {
    "totalLogins": 1273,
    "uniqueActiveUsers": 45
  }
}
```

### 2. Экспорт отчетов

#### Экспорт ежедневного отчета в Excel
```http
GET /api/reports/daily/2025-10-18/export/excel
```
**Ответ:** Файл `daily_report_2025-10-18.xlsx`

#### Экспорт ежедневного отчета в PDF
```http
GET /api/reports/daily/2025-10-18/export/pdf
```
**Ответ:** Файл `daily_report_2025-10-18.pdf`

#### Экспорт сводки за период в Excel
```http
GET /api/reports/export/excel?startDate=2025-10-01&endDate=2025-10-19
```
**Ответ:** Файл `reports_summary_2025-10-01_to_2025-10-19.xlsx`

#### Экспорт сводки за период в PDF
```http
GET /api/reports/export/pdf?startDate=2025-10-01&endDate=2025-10-19
```
**Ответ:** Файл `reports_summary_2025-10-01_to_2025-10-19.pdf`

### 3. Email рассылка

#### Отправить ежедневный отчет по email
```http
POST /api/reports/daily/2025-10-18/email?email=admin@example.com&attachPdf=true&attachExcel=true
```

**Ответ:**
```json
{
  "status": "success",
  "message": "Email is being sent to admin@example.com",
  "date": "2025-10-18"
}
```

#### Отправить сводный отчет по email
```http
POST /api/reports/email/summary?startDate=2025-10-01&endDate=2025-10-19&email=admin@example.com
```

**Ответ:**
```json
{
  "status": "success",
  "message": "Summary email is being sent to admin@example.com",
  "startDate": "2025-10-01",
  "endDate": "2025-10-19",
  "reportsCount": "19"
}
```

### 4. Ручная генерация отчетов

#### Сгенерировать отчет за конкретную дату
```http
POST /api/reports/daily/generate?date=2025-10-18&regenerate=false
```

**Параметры:**
- `date` - Дата для генерации отчета
- `regenerate` - Перегенерировать если уже существует (по умолчанию false)

### 5. Health Check

#### Проверка состояния модуля
```http
GET /api/reports/health
```

**Ответ:**
```json
{
  "status": "UP",
  "module": "Reporting",
  "totalReports": 150,
  "latestReportDate": "2025-10-18",
  "timestamp": "2025-10-19"
}
```

---

## 📊 Экспорт отчетов

### Excel Export (Apache POI 5.2.3)

**Возможности:**
- Автоматическое форматирование таблиц
- Стилизация заголовков и данных
- Автоматическая ширина колонок
- Поддержка нескольких листов

**Формат файла:** `.xlsx` (Excel 2007+)

**Структура отчета:**
1. Заголовок с датой
2. Секция "PATIENT STATISTICS"
3. Секция "RECOMMENDATION STATISTICS"
4. Секция "ESCALATION STATISTICS"
5. Секция "SYSTEM PERFORMANCE"
6. Секция "USER ACTIVITY"

### PDF Export (Apache PDFBox 3.0.0)

**Возможности:**
- Профессиональное форматирование
- Автоматическое создание новых страниц
- Футер с датой генерации
- Поддержка различных шрифтов

**Формат файла:** `.pdf`

**Особенности:**
- Формат A4
- Шрифт: Helvetica
- Автоматический перенос на новую страницу при нехватке места

---

## 📧 Email рассылка

### Конфигурация SMTP

**Файл:** `application.properties`

```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your-email@gmail.com
spring.mail.password=your-app-password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

### Возможности Email сервиса

**Асинхронная отправка:**
- Использует `@Async` для неблокирующей отправки
- Не замедляет API ответы

**HTML шаблоны:**
- Профессиональный дизайн писем
- Адаптивная верстка
- Цветовое кодирование секций

**Вложения:**
- Автоматическое прикрепление PDF
- Автоматическое прикрепление Excel
- Настраиваемые параметры вложений

**Типы писем:**
1. **Daily Report Email** - Ежедневный отчет с детальной статистикой
2. **Summary Report Email** - Сводный отчет за период

---

## 🔧 Технические детали

### Используемые технологии

**Backend:**
- Spring Boot 3.5.5
- Spring Data JPA (PostgreSQL/H2)
- Spring Data MongoDB
- Spring Mail
- Spring Scheduling

**Библиотеки:**
- Apache POI 5.2.3 (Excel)
- Apache PDFBox 3.0.0 (PDF)
- Lombok (упрощение кода)
- Jackson (JSON сериализация)

### Конфигурация

**AsyncConfig.java:**
```java
@Configuration
@EnableAsync
@EnableScheduling
public class AsyncConfig {
    // Включает асинхронные операции и планировщик задач
}
```

### Retention Policy

**MongoDB события:**
- Хранятся 30 дней
- Автоматически удаляются после агрегации
- Метод: `DataAggregationService.cleanupOldEvents(30)`

**PostgreSQL отчеты:**
- Хранятся бессрочно
- Могут быть удалены вручную через API (будущая функция)

---

## 📈 Метрики в отчетах

### Статистика пациентов
- **Total Patients Registered** - Зарегистрировано пациентов
- **Total VAS Records** - Записей измерений боли
- **Average VAS Level** - Средний уровень боли (0-10)
- **Critical Cases** - Критические случаи (VAS >= 7)

### Статистика рекомендаций
- **Total Recommendations** - Всего рекомендаций
- **Approved** - Одобрено
- **Rejected** - Отклонено
- **Approval Rate** - Процент одобрения

### Статистика эскалаций
- **Total Escalations** - Всего эскалаций
- **Resolved** - Разрешено
- **Pending** - В ожидании
- **Avg Resolution Time** - Среднее время разрешения (часы)

### Производительность системы
- **Avg Processing Time** - Среднее время обработки (мс)
- **Total Operations** - Всего операций
- **Failed Operations** - Неудачных операций

### Активность пользователей
- **Total Logins** - Всего входов
- **Unique Active Users** - Уникальных активных пользователей
- **Failed Login Attempts** - Неудачных попыток входа

---

## 🚀 Быстрый старт

### 1. Настройка Email (опционально)

Отредактируйте `application.properties`:
```properties
spring.mail.host=smtp.gmail.com
spring.mail.username=your-email@gmail.com
spring.mail.password=your-app-password
```

### 2. Запуск приложения

```bash
mvn spring-boot:run
```

### 3. Проверка работы модуля

```bash
curl http://localhost:8080/api/reports/health
```

### 4. Ручная генерация тестового отчета

```bash
curl -X POST "http://localhost:8080/api/reports/daily/generate?date=2025-10-18"
```

### 5. Получение отчета

```bash
curl http://localhost:8080/api/reports/daily/2025-10-18
```

---

## 🐛 Troubleshooting

### Проблема: Отчеты не генерируются автоматически

**Решение:**
- Проверьте, что `@EnableScheduling` присутствует в `AsyncConfig.java`
- Проверьте логи на наличие ошибок в 00:30

### Проблема: Email не отправляются

**Решение:**
- Проверьте SMTP настройки в `application.properties`
- Для Gmail: используйте App Password, а не обычный пароль
- Проверьте логи на наличие `MessagingException`

### Проблема: Ошибка при экспорте в PDF/Excel

**Решение:**
- Убедитесь, что зависимости Apache POI и PDFBox присутствуют в `pom.xml`
- Проверьте, что отчет существует в БД перед экспортом

---

## 📝 Примечания

- Все даты в формате `yyyy-MM-dd`
- Все времена в UTC
- Email отправка асинхронная (не блокирует API)
- Экспорт файлов синхронный (блокирует до генерации)
- MongoDB события удаляются автоматически через 30 дней
