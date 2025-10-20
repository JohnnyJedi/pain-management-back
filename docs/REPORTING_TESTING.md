# Руководство по тестированию модуля отчетности

## 📋 Содержание
1. [Подготовка](#подготовка)
2. [Тестирование агрегации](#тестирование-агрегации)
3. [Тестирование API](#тестирование-api)
4. [Тестирование экспорта](#тестирование-экспорта)
5. [Тестирование Email](#тестирование-email)

---

## 🔧 Подготовка

### Запуск приложения
```bash
mvn spring-boot:run
```

### Проверка Health
```bash
curl http://localhost:8080/api/reports/health
```

### Создание тестовых данных MongoDB
```javascript
db.analytics_events.insertMany([
  {
    eventType: "PATIENT_REGISTERED",
    timestamp: ISODate("2025-10-18T10:30:00Z"),
    userId: "user123",
    metadata: {}
  },
  {
    eventType: "VAS_RECORDED",
    timestamp: ISODate("2025-10-18T11:00:00Z"),
    userId: "user123",
    metadata: { vasLevel: "7" }
  }
])
```

---

## 📊 Тестирование агрегации

### Тест 1: Ручная генерация
```bash
curl -X POST "http://localhost:8080/api/reports/daily/generate?date=2025-10-18"
```
**Ожидается:** HTTP 201, создан отчет

### Тест 2: Защита от дубликатов
```bash
curl -X POST "http://localhost:8080/api/reports/daily/generate?date=2025-10-18"
```
**Ожидается:** HTTP 409 Conflict

### Тест 3: Перегенерация
```bash
curl -X POST "http://localhost:8080/api/reports/daily/generate?date=2025-10-18&regenerate=true"
```
**Ожидается:** HTTP 201, отчет обновлен

---

## 🌐 Тестирование API

### Тест 4: Получение отчета
```bash
curl http://localhost:8080/api/reports/daily/2025-10-18
```
**Ожидается:** HTTP 200, JSON с данными

### Тест 5: Отчеты за период
```bash
curl "http://localhost:8080/api/reports/daily?startDate=2025-10-01&endDate=2025-10-19"
```
**Ожидается:** HTTP 200, массив отчетов

### Тест 6: Сводная статистика
```bash
curl "http://localhost:8080/api/reports/summary?startDate=2025-10-01&endDate=2025-10-19"
```
**Ожидается:** HTTP 200, агрегированная статистика

---

## 📄 Тестирование экспорта

### Тест 7: Excel экспорт
```bash
curl -o report.xlsx "http://localhost:8080/api/reports/daily/2025-10-18/export/excel"
```
**Проверка:** Откройте файл в Excel, проверьте форматирование

### Тест 8: PDF экспорт
```bash
curl -o report.pdf "http://localhost:8080/api/reports/daily/2025-10-18/export/pdf"
```
**Проверка:** Откройте PDF, проверьте читаемость

---

## 📧 Тестирование Email

### Настройка Mailtrap
```properties
spring.mail.host=sandbox.smtp.mailtrap.io
spring.mail.port=2525
spring.mail.username=your-username
spring.mail.password=your-password
```

### Тест 9: Отправка отчета
```bash
curl -X POST "http://localhost:8080/api/reports/daily/2025-10-18/email?email=test@example.com"
```
**Проверка:** Проверьте Mailtrap inbox, письмо должно прийти с вложениями

---

## ✅ Чеклист тестирования

- [ ] Health check работает
- [ ] Ручная генерация отчета
- [ ] Защита от дубликатов
- [ ] Получение отчета за дату
- [ ] Получение отчетов за период
- [ ] Сводная статистика
- [ ] Excel экспорт работает
- [ ] PDF экспорт работает
- [ ] Email отправляется
- [ ] Вложения прикрепляются
- [ ] Cleanup старых событий
- [ ] @Scheduled задача работает
