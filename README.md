## 17.09.2025
Евгений:
-  Добавил в AdminServiceImpl @Transactional над сложными методами, чуть сократил метод createPerson с помощью modelMapper.
-  Убрал PatientDTO и VASInputDTO из admin/dto и пока перекинул в nurse/dto.
-  Из PersonLoginResponseDTO убрал token, так как он не используется, но добавил поле firstName для приветствия на фронте и в PersonService убрал всё что с токеном связано.
-  Добавил ENUM Roles, но пока не переделал в папке admin.
-  Перетащил общее для всех dto (LoginResponse,LoginRequest,ChangeCredentials) из admin/dto в папку common/dto.
-  Перетащил общий контроллер для всех из admin PersonalController в папку common/controller и PersonService в папку common/service.
-  Перетащил из admin/entity Approval в папку doctor/entity и anethesiologist/entity (может пригодиться).
-  Перетащил из admin/entity Patient и VAS в папку nurse/entity (может пригодиться).


 Nurse-logic:
 - Создал DTO со следующей структурой:
 * PatientDTO — базовые данные пациента + коллекции emr, vas, recommendations.
 * EmrDTO — медицинская карта (анализы, показатели организма).
 * VasDTO — жалобы на боль (VAS шкала).
 * RecommendationDTO — одна рекомендация (связана с конкретным случаем/жалобой).
 * DrugRecommendationDTO — конкретное лекарство внутри одной рекомендации.

## 18.09.2025
 Евгений:
 - Создал папку external_service с DTO и Service для реализации в будущем получения EMR данных пациента извне.

   Nurse-logic:
 - Создал Entity по схожей с DTO структурой.
 - Создал отдельные Repositories на все Entities.
 - Создал Controller со всеми эндпоинтами
 - Создал Service со всеми методами для Nurse (частично имплементировали)
 - Создал директорию treatment-protocol для хранения протокола лечения (Exel таблица)

## 19.09.2025
 Евгений:
 - Сделал папку nurse/dto/exceptions для красивых ошибок.
   Nurse-logic:
 - Реализовал все методы в Sevice. 
   Treatment Protocol:
 - добавил библиотеку в pom.xml liquibase для создания в будущем changelog таблицы инструкций лечения
 - добавил библиотеку Apache POI для считывания файлов .xlsx(Excel) формата

## 20.09.2025
  Евгений:
  - Создал папку excel_loader и клас TreatmentProtocolLoader implements CommandLineRunner.
  - В этом же классе использовал интерфейс библиотеки Apache POI Workbook workbook и его класс реализацию для Excel - XSSFWorkbook 
    для считывания протокола лечения и переноса в БД.

## 21/24.09.2025
  Евгений:
  - Внутри папки treatment_protocol создал entity для всех полей (21 поле) из протокола 
    и затем создал репозиторий для хранения этих сущностей, реализовал в классе TreatmentProtocolLoader метод для считывания данных таблицы 
    протокола и создания объекта entity для хранения в БД (протокол оцифрован)
  - Для реализации генерации рекомендации лекарства создал отдельный сервис TreatmentProtocolService и начал реализовывать
    алгоритмы (фильтры) отбора нужной рекомендации (внутри лекарство и альтернативное лекарство).
  - Реализовал отбор нужных строк из таблицы Протокола Лечения, отфильтровав по уровню болю.
  - Создал объекты рекомендаций в этом же методе и начал их корректировать по возрасту и по весу (отдельные подметоды в классе TreatmentProtocolService)
  - 
## 26.09.2025
  Евгений:
 - Создал во всех DTO и во всех Entity поля createdAt, updatedAt, createdBy, updateBy для будущего аудита действий пользователей.
 - Поменял все LocalDate на LocalDateTime

## 30.09.2025
 Евгений:
 - Переделал все поля согласно утверждённой концепции регистрации пациента и заполнении его мед. карты.
 - Теперь вместо personId везде используются MRN и все методы в контролере и сервисе переделаны с этим учётом.
 - Личный номер MRN генерируется на основе технического Id из БД и форматируется в сервисе при создании пациента и повторно сохраняется.
 - Добавлены методы в контролере и сервисе по query параметрам в общий метод по поиску пациентов, также и в сервис и репозиторий.
 Теперь универсальный метод поиска в зависимости от переданных параметров.

##17.09.2025
 Величайший Full Stack современности Ник:
-  Создал полноценный модуль doctor с правильной архитектурой по слоям (entity, dto, repository, service, controller).
-  Реализовал сущности Patient и Recommendation с @PrePersist/@PreUpdate для автоматического управления временными метками.
-  Создал ENUM RecommendationStatus (PENDING, APPROVED, REJECTED) для типизации статусов рекомендаций.
-  Разделил DTO на Request (PatientCreationDTO, RecommendationRequestDTO) и Response (PatientResponseDTO, RecommendationDTO) для четкого разделения входящих и исходящих данных.
-  Добавил @Transactional аннотации: для методов изменения данных и @Transactional(readOnly = true) для методов чтения.
-  Реализовал полный CRUD для рекомендаций: создание, просмотр, одобрение, отклонение, обновление, удаление.
-  Реализовал полный CRUD для пациентов: создание, просмотр, обновление, мягкое удаление (через поле active).
-  Создал DoctorController с REST API эндпоинтами, используя @RequestParam(defaultValue = "system") как временную заглушку до внедрения аутентификации.
-  Настроил валидацию входных данных через @Valid и @NotBlank/@NotNull аннотации.
-  Использовал ModelMapper для конвертации между Entity и DTO, что упрощает маппинг данных.
-  **ВАЖНО: Решил конфликты имен JPA сущностей** - переименовал nurse/entity/Patient в NursePatient и anesthesiologist/entity/Recommendation в AnesthesiologistRecommendation, так как JPA требует уникальные имена сущностей в рамках всего приложения.
-  Исправил конфликты типов String vs Roles enum в AdminServiceImpl и PersonService при работе с ролями пользователей.

## 18.09.2025
Nick:
-  **КРИТИЧЕСКОЕ ИСПРАВЛЕНИЕ: Починил смену креденшиалов** - добавил поле currentLogin в ChangeCredentialsDTO с валидацией @NotBlank(message = "Current login is required").
-  Исправил логику PersonService.changeCredentials() - теперь система ищет пользователя по currentLogin (текущий логин) вместо newLogin, что устранило ошибку "User not found".
-  Добавил обновление логина пользователя через person.setLogin(request.getNewLogin()) в методе changeCredentials для корректной смены логина.
-  **КРИТИЧЕСКОЕ ИСПРАВЛЕНИЕ: Починил удаление пользователей** - изменил сигнатуру AdminService.deletePerson() с Long id на String personId для работы с документным ID вместо технического.
-  Переписал AdminServiceImpl.deletePerson() - теперь использует personRepository.findByPersonId(personId) и personRepository.delete(person) вместо deleteById для корректного поиска по документному ID.
-  Обновил AdminController.deletePerson() - изменил @DeleteMapping("/{id}") на @DeleteMapping("/{personId}") и @PathVariable Long id на @PathVariable String personId.
-  Исправил валидационные сообщения в ChangeCredentialsDTO - изменил "Login is required" на "New login is required" для поля newLogin.
-  **АРХИТЕКТУРНОЕ РЕШЕНИЕ**: Использование personId (документный ID) вместо технического Long id обеспечивает более логичную работу API и соответствует бизнес-логике приложения.

## 21.09.2025:
Nick:
-  **🚀 СОЗДАЛ ПОЛНОЦЕННЫЙ МОДУЛЬ ANESTHESIOLOGIST** с использованием DTO паттерна и ModelMapper для профессиональной архитектуры enterprise-уровня.
-  **🏗️ Реализовал полную архитектуру по слоям**: entity (Escalation, TreatmentProtocol, TreatmentProtocolComment), dto (6 классов), repository (3 интерфейса), service (интерфейс + имплементация), controller.
-  **📊 Создал продвинутые DTO классы**: EscalationResponseDTO, ProtocolRequestDTO, ProtocolResponseDTO, CommentRequestDTO, CommentResponseDTO, EscalationStatsDTO.
-  **🔧 Создал AnesthesiologistServiceImpl** с @Transactional на уровне класса и @Transactional(readOnly = true) для read-only операций - оптимальный подход для производительности.
-  **🌐 Реализовал 13 REST API эндпоинтов** в AnesthesiologistController: управление эскалациями (5 эндпоинтов), протоколами (5 эндпоинтов), комментариями (3 эндпоинта).
-  **📈 Добавил статистику эскалаций** через EscalationStatsDTO с подсчетом по статусам и приоритетам для аналитики и дашбордов.
-  **✅ Добавил PENDING_APPROVAL статус** в ProtocolStatus enum для корректной работы workflow одобрения протоколов.
-  **🛡️ РЕАЛИЗОВАЛ ПОЛНУЮ ВАЛИДАЦИЮ** во всем приложении - добавил @Size, @Past, @Positive, @Max аннотации во все DTO с разумными ограничениями.
-  **📏 Установил ограничения длины полей**: title ≤500, content ≤5000, names ≤100, IDs ≤50, descriptions ≤2000, passwords 6-100 символов.
-  **📅 Добавил валидацию дат и чисел**: @Past для дат рождения, @Positive для веса/возраста, @Max(150) для реалистичного возраста.
-  **🔍 Добавил недостающие @Valid аннотации** в DoctorController для RecommendationApprovalDTO в методах approve/reject.
-  **🚦 Обеспечил data integrity**: все входящие данные теперь валидируются автоматически с понятными сообщениями об ошибках для фронтенда.
-  **⚡ РЕЗУЛЬТАТ**: Создана enterprise-архитектура с полным разделением ответственности, автоматической валидацией, статистикой и готовностью к масштабированию.


