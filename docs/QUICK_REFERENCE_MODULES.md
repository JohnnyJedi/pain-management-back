# 🚀 Быстрая справка: Какой модуль использовать?

**Дата:** 29.10.2025  
**Версия:** 1.0  

---

## 🎯 ВЫБОР МОДУЛЯ ПО ВОПРОСУ

### "Сколько?" / "Кто?" / "Что?" → Analytics Module

```
✅ Сколько рекомендаций одобрено?
✅ Кто одобрил рекомендацию?
✅ Что делал пользователь?
✅ Какой комментарий оставил врач?
✅ Сколько эскалаций создано?
✅ Последние события в системе?

API: GET /api/analytics/...
```

---

### "Как быстро?" / "Соблюдается ли SLA?" → Performance SLA

```
✅ Как быстро генерируются рекомендации?
✅ Соблюдается ли SLA 2000ms?
✅ Какие операции медленные?
✅ Каков 95-й перцентиль?
✅ Есть ли нарушения SLA?
✅ Какие методы требуют оптимизации?

API: GET /api/performance/...
```

---

### "Тренды?" / "За период?" / "Экспорт?" → Reporting

```
✅ Динамика за месяц?
✅ Сравнение с прошлым кварталом?
✅ Нужен Excel/PDF отчет?
✅ Отправить отчет на email?
✅ Статистика за год?
✅ Презентация для руководства?

API: GET /api/reports/...
```

---

## 📊 ВЫБОР ПО ТИПУ ДАННЫХ

| Нужны | Модуль |
|-------|--------|
| Детальные события | Analytics |
| Метрики времени | Performance |
| Агрегаты за период | Reporting |
| Real-time данные | Analytics или Performance |
| Исторические данные | Reporting |
| Бизнес-статусы | Analytics |
| Технические метрики | Performance |

---

## 🎭 ВЫБОР ПО РОЛИ ПОЛЬЗОВАТЕЛЯ

### Администратор больницы
- **Analytics** — активность персонала, события
- **Performance** — здоровье системы, SLA
- **Reporting** — отчеты для руководства

### DevOps / SRE
- **Performance** — мониторинг производительности
- **Analytics** — контекст ошибок

### Руководитель / Директор
- **Reporting** — долгосрочная статистика, экспорт

### Врач / Медсестра
- **Analytics** — своя активность (опционально)

---

## ⚡ БЫСТРЫЕ КОМАНДЫ

### Analytics
```bash
# Статистика событий
curl "http://localhost:8080/api/analytics/events/stats"

# Активность пользователя
curl "http://localhost:8080/api/analytics/users/doctor_123/activity"

# Последние события
curl "http://localhost:8080/api/analytics/events/recent?limit=50"
```

### Performance SLA
```bash
# Общая статистика
curl "http://localhost:8080/api/performance/statistics/recent?hours=24"

# Нарушения SLA
curl "http://localhost:8080/api/performance/sla-violations/recent?hours=24"

# Медленные операции
curl "http://localhost:8080/api/performance/slowest?limit=10"
```

### Reporting
```bash
# Отчеты за период
curl "http://localhost:8080/api/reports/daily?startDate=2025-10-01&endDate=2025-10-31"

# Экспорт в Excel
curl "http://localhost:8080/api/reports/export/excel?startDate=2025-10-01&endDate=2025-10-31" -o report.xlsx

# Отправить на email
curl -X POST "http://localhost:8080/api/reports/email/summary?email=admin@example.com&startDate=2025-10-01&endDate=2025-10-31"
```

---

## 🔄 ВЗАИМОСВЯЗЬ МОДУЛЕЙ

```
Analytics (real-time) → Reporting (daily aggregation)
     ↓                           ↓
  MongoDB                   PostgreSQL
  (30 days)                 (forever)

Performance (real-time)
     ↓
  MongoDB
  (30 days)
```

---

## ✅ ПРАВИЛО ВЫБОРА

1. **Нужны детали события?** → Analytics
2. **Нужно время выполнения?** → Performance
3. **Нужен отчет за период?** → Reporting

**Все три модуля работают независимо и дополняют друг друга!**
