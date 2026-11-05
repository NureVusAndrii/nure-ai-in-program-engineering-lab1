# Mini Issue Tracker — API-контракт (ЛР1)

Бізнес-правила, поля й обмеження визначені в `requirements.md`; тут вони не дублюються,
а наводяться посиланнями: контракт даних — §6, переходи статусів — §7, PATCH — §8,
каталог ситуацій — §9, упорядкування — §10.

## 1. Загальні положення

- Базова адреса під час локального запуску: `http://127.0.0.1:8000`; документація — `/docs`.
- Тіла запитів і відповідей — JSON, UTF-8, `Content-Type: application/json`.
- Час — ISO 8601 в UTC із суфіксом `Z` (NFR-06).
- `{id}` у шляху — ціле число; нечисловий `{id}` → 422.

**Об'єкти відповіді**

| Об'єкт | Поля |
|---|---|
| Issue | `id`, `title`, `description`, `status`, `priority`, `assignee_id`, `created_at` |
| Comment | `id`, `issue_id`, `author_id`, `text`, `created_at` |
| User | `id`, `name` |
| Error | `error.code`, `error.message`, `error.details[]` (`field`, `reason`) |

`reason` у `details`: `required`, `invalid_type`, `null_not_allowed`, `length_out_of_range`,
`invalid_enum`, `unknown_user`, `unknown_field`, `empty_update`. Для 404 і 409 `details = []`.

## 2. Коди відповідей (єдине джерело для HTTP-кодів)

| Ситуація (`requirements.md` §9) | HTTP | `error.code` |
|---|---|---|
| R-01 створення заявки або коментаря | 201 | — |
| R-02, R-03, R-04 читання, зміна, повтор статусу, порожній список | 200 | — |
| R-05 некоректні дані | 422 | `VALIDATION_ERROR` |
| R-06 заявку не знайдено | 404 | `ISSUE_NOT_FOUND` |
| R-07 заборонений перехід статусу (T-04, T-07, T-08) | 409 | `INVALID_STATUS_TRANSITION` |
| R-07 редагування закритої заявки (§7.1) | 409 | `ISSUE_CLOSED` |

Порядок перевірок: 422 → 404 → 409. Код 400 не використовується.

## 3. Маршрути

| # | Метод і шлях | Призначення | Успіх | Помилки |
|---|---|---|---|---|
| 1 | GET /health | Перевірка запуску | 200 | — |
| 2 | GET /users | Довідник із двох тестових користувачів | 200 | — |
| 3 | POST /issues | Створити заявку | 201 | 422 |
| 4 | GET /issues | Список з фільтрами `status`, `priority` (AND) | 200 | 422 |
| 5 | GET /issues/{id} | Одна заявка | 200 | 404, 422 |
| 6 | PATCH /issues/{id} | Часткова зміна `title`, `description`, `priority`, `assignee_id` | 200 | 404, 409, 422 |
| 7 | PATCH /issues/{id}/status | Окрема зміна статусу | 200 | 404, 409, 422 |
| 8 | POST /issues/{id}/comments | Додати коментар | 201 | 404, 422 |
| 9 | GET /issues/{id}/comments | Коментарі заявки | 200 | 404, 422 |
| 10 | POST /issues/{id}/ai-recommendation | Рекомендація AI (ЛР4, поза поточним обсягом) | 200 | 404, 422 |

### GET /health

Без параметрів. Відповідь `200 OK`:
```json
{"status": "ok"}
```

### GET /users

Без параметрів. Відповідь `200 OK`:
```json
[{"id": 1, "name": "Agent A"}, {"id": 2, "name": "Agent B"}]
```

### POST /issues

Тіло: `title` (обов'язкове), `description`, `priority` (необов'язкові). Інші поля → 422 (§6).
Відповідь — Issue зі `status=OPEN`, `assignee_id=null`.

Запит:
```json
{"title": "  Login fails  ", "description": "500 on submit", "priority": "high"}
```

Відповідь `201 Created`:
```json
{"id": 1, "title": "Login fails", "description": "500 on submit", "status": "OPEN", "priority": "high", "assignee_id": null, "created_at": "2026-11-05T09:15:02.481233Z"}
```

Запит без необов'язкових полів:
```json
{"title": "Typo on page"}
```

Відповідь `201 Created`:
```json
{"id": 2, "title": "Typo on page", "description": "", "status": "OPEN", "priority": "medium", "assignee_id": null, "created_at": "2026-11-05T09:16:40.003114Z"}
```

Помилковий запит:
```json
{"title": " ab ", "priority": "High", "status": "CLOSED"}
```

Відповідь `422 Unprocessable Content`:
```json
{"error": {"code": "VALIDATION_ERROR", "message": "Request data is invalid", "details": [{"field": "title", "reason": "length_out_of_range"}, {"field": "priority", "reason": "invalid_enum"}, {"field": "status", "reason": "unknown_field"}]}}
```

### GET /issues

Query: `status` ∈ {`OPEN`, `IN_PROGRESS`, `CLOSED`}, `priority` ∈ {`low`, `medium`, `high`};
обидва необов'язкові, разом — логічне AND. Інші query-параметри ігноруються.
Порядок — §10.

Запит: `GET /issues?status=OPEN&priority=high`. Відповідь `200 OK`:
```json
[{"id": 1, "title": "Login fails", "description": "500 on submit", "status": "OPEN", "priority": "high", "assignee_id": null, "created_at": "2026-11-05T09:15:02.481233Z"}]
```

Запит: `GET /issues?status=CLOSED` без відповідних заявок. Відповідь `200 OK`:
```json
[]
```

Запит: `GET /issues?status=open`. Відповідь `422 Unprocessable Content`:
```json
{"error": {"code": "VALIDATION_ERROR", "message": "Request data is invalid", "details": [{"field": "status", "reason": "invalid_enum"}]}}
```

### GET /issues/{id}

Запит: `GET /issues/1`. Відповідь `200 OK` — Issue (формат як у POST /issues).

Запит: `GET /issues/999`. Відповідь `404 Not Found`:
```json
{"error": {"code": "ISSUE_NOT_FOUND", "message": "Issue 999 not found", "details": []}}
```

### PATCH /issues/{id}

Тіло: непорожня підмножина `title`, `description`, `priority`, `assignee_id`.
Семантика відсутнього ключа, `null`, порожнього тіла й атомарності — `requirements.md` §8
(P-01–P-09). Відповідь — повна актуальна Issue.

Запит:
```json
{"priority": "high", "assignee_id": 2}
```

Відповідь `200 OK`:
```json
{"id": 2, "title": "Typo on page", "description": "", "status": "OPEN", "priority": "high", "assignee_id": 2, "created_at": "2026-11-05T09:16:40.003114Z"}
```

Зняття призначення (P-02):
```json
{"assignee_id": null}
```

Змішані валідні й невалідні зміни (P-06), відповідь `422 Unprocessable Content`:
```json
{"error": {"code": "VALIDATION_ERROR", "message": "Request data is invalid", "details": [{"field": "title", "reason": "length_out_of_range"}]}}
```

Редагування закритої заявки (§7.1), відповідь `409 Conflict`:
```json
{"error": {"code": "ISSUE_CLOSED", "message": "Issue 1 is CLOSED and cannot be edited", "details": []}}
```

### PATCH /issues/{id}/status

Тіло: `status` (обов'язкове, єдине поле). Переходи — `requirements.md` §7 (T-01–T-09).
Відповідь — повна актуальна Issue; для T-01, T-05, T-09 дані не змінюються.

Запит:
```json
{"status": "IN_PROGRESS"}
```

Відповідь `200 OK`:
```json
{"id": 2, "title": "Typo on page", "description": "", "status": "IN_PROGRESS", "priority": "high", "assignee_id": 2, "created_at": "2026-11-05T09:16:40.003114Z"}
```

Запит до заявки зі статусом CLOSED (T-07):
```json
{"status": "OPEN"}
```

Відповідь `409 Conflict`:
```json
{"error": {"code": "INVALID_STATUS_TRANSITION", "message": "Transition CLOSED -> OPEN is not allowed", "details": []}}
```

Запит зі значенням поза enum, відповідь `422 Unprocessable Content`:
```json
{"error": {"code": "VALIDATION_ERROR", "message": "Request data is invalid", "details": [{"field": "status", "reason": "invalid_enum"}]}}
```

### POST /issues/{id}/comments

Тіло: `author_id` (1 або 2), `text` (1–1000 символів після trim). Дозволено для CLOSED (§7.1).

Запит:
```json
{"author_id": 1, "text": "Reproduced on Firefox"}
```

Відповідь `201 Created`:
```json
{"id": 1, "issue_id": 1, "author_id": 1, "text": "Reproduced on Firefox", "created_at": "2026-11-05T09:20:11.512900Z"}
```

Помилковий запит:
```json
{"author_id": 3, "text": "   "}
```

Відповідь `422 Unprocessable Content`:
```json
{"error": {"code": "VALIDATION_ERROR", "message": "Request data is invalid", "details": [{"field": "author_id", "reason": "unknown_user"}, {"field": "text", "reason": "length_out_of_range"}]}}
```

Коментар до неіснуючої заявки, відповідь `404 Not Found`:
```json
{"error": {"code": "ISSUE_NOT_FOUND", "message": "Issue 999 not found", "details": []}}
```

### GET /issues/{id}/comments

Відповідь — масив Comment у порядку створення (§10); для заявки без коментарів — `[]`;
для неіснуючої заявки — 404.

Запит: `GET /issues/1/comments`. Відповідь `200 OK`:
```json
[{"id": 1, "issue_id": 1, "author_id": 1, "text": "Reproduced on Firefox", "created_at": "2026-11-05T09:20:11.512900Z"}, {"id": 2, "issue_id": 1, "author_id": 2, "text": "Fix in progress", "created_at": "2026-11-05T09:31:47.100058Z"}]
```

### POST /issues/{id}/ai-recommendation

Запланований маршрут ЛР4 (FR-08, US-08, ADR-02); у ЛР2–ЛР3 не реалізується.
Повертає пропозицію категорії, пріоритету й короткого підсумку; заявку не змінює.
Склад полів і перелік категорій визначаються в ЛР4.
