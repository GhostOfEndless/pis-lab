# Техническое задание на реализацию бэкенда страницы купленных курсов

## 1. Входные параметры

| Параметр | Тип параметра | Обязательность | Описание |
|----------|---------------|----------------|-----------|
| userId | uuid | + | Идентификатор пользователя |
| courseId | uuid | - | Идентификатор курса (для операций с конкретным курсом) |
| page | number | - | Номер страницы для пагинации |
| limit | number | - | Количество элементов на странице |

## 2. Авторизация

### a. По бизнесу
- Доступ к списку купленных курсов предоставляется только авторизованным пользователям
- Пользователь может видеть только свои купленные курсы
- Доступ к чату поддержки предоставляется всем авторизованным пользователям

### b. Техническая реализация
```sql
-- Проверка прав доступа к курсам пользователя
SELECT id FROM tbl_user_courses 
WHERE user_id = {userId} 
AND course_id = {courseId}

-- Проверка статуса курса
SELECT status FROM tbl_courses 
WHERE id = {courseId} 
AND status = 'ACTIVE'
```

## 3. Передача данных при инициализации

```json
{
  "courses": [
    {
      "id": 1,
      "title": "JavaScript для начинающих",
      "progress": 45,
      "startDate": "2024-09-01",
      "endDate": "2024-12-01",
      "status": "in_progress",
      "lastAccessDate": "2024-10-15"
    }
  ],
  "pagination": {
    "currentPage": 1,
    "totalPages": 5,
    "totalItems": 48,
    "itemsPerPage": 10
  }
}
```

### Описание атрибутов

| Атрибут, уровень 1 | Уровень 2 | Тип | Название атрибута | Формирование на бэкенде | Обязательность |
|-------------------|-----------|-----|-------------------|------------------------|----------------|
| courses | id | number | ID курса | SELECT id FROM tbl_courses | + |
| courses | title | string | Название курса | SELECT title FROM tbl_courses | + |
| courses | progress | number | Прогресс прохождения | SELECT ROUND(COUNT(completed_lessons) * 100.0 / total_lessons) FROM tbl_user_progress | + |
| courses | startDate | string | Дата начала | SELECT start_date FROM tbl_user_courses | + |
| courses | endDate | string | Дата окончания | SELECT end_date FROM tbl_user_courses | + |
| courses | status | string | Статус курса | SELECT status FROM tbl_user_courses | + |
| courses | lastAccessDate | string | Дата последнего посещения | SELECT last_access_date FROM tbl_user_courses | + |
| pagination | currentPage | number | Текущая страница | Входной параметр page | + |
| pagination | totalPages | number | Всего страниц | CEIL(COUNT(*) / {limit}) | + |

## 4. REST-запросы

### 4.1 Получение списка купленных курсов

| Название | Получение списка курсов пользователя |
|----------|-------------------------------------|
| URL | /api/user/courses |
| Тип метода | GET |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | SELECT c.*, uc.progress, uc.start_date, uc.end_date, uc.status, uc.last_access_date FROM tbl_courses c JOIN tbl_user_courses uc ON c.id = uc.course_id WHERE uc.user_id = {userId} |
| Query parameters | page: номер страницы<br>limit: элементов на странице |
| Responses | 200 OK: {courses, pagination}<br>401 Unauthorized: {"message": "Требуется авторизация"} |

### 4.2 Получение деталей чата поддержки

| Название | Инициализация личного чата с поддержкой |
|----------|----------------------------------------|
| URL | /api/personal-chat/{userId} |
| Тип метода | GET |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | 1. Проверка существующего чата<br>2. Создание нового чата при отсутствии<br>3. Получение истории сообщений |
| Query parameters | - |
| Responses | 200 OK: {"chatId": "uuid", "messages": [...]}<br>401 Unauthorized: {"message": "Требуется авторизация"} |

### 4.3 Обновление прогресса курса

| Название | Обновление прогресса прохождения курса |
|----------|----------------------------------------|
| URL | /api/user/courses/{courseId}/progress |
| Тип метода | PUT |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | UPDATE tbl_user_courses SET progress = {progress}, last_access_date = CURRENT_TIMESTAMP WHERE user_id = {userId} AND course_id = {courseId} |
| Body | {"progress": number} |
| Responses | 200 OK: {"message": "Прогресс обновлен"}<br>400 Bad Request: {"message": "Некорректные данные"}<br>404 Not Found: {"message": "Курс не найден"} |
