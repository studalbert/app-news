# Руководство по тестированию API в Postman

Ниже описан практический сценарий тестирования всех текущих endpoint вашего API.

## 1) Доступные endpoint

Базовый префикс:
- `{{base_url}}/api/v1/auth/`

Endpoint'ы:
- `POST /register/`
- `POST /login/`
- `POST /logout/`
- `GET /profile/`
- `PUT /profile/`
- `PATCH /profile/`
- `PUT /change-password/`
- `POST /token/refresh/`

## 2) Подготовка Postman

Создайте Environment (например, `local`) и переменные:
- `base_url` = `http://127.0.0.1:8000`
- `access_token` = (пусто)
- `refresh_token` = (пусто)
- `email` = `testuser@example.com`
- `password` = `TestPass123!`
- `new_password` = `NewPass123!`

В Collection настройте Authorization:
- Type: `Bearer Token`
- Token: `{{access_token}}`

Тогда для защищенных endpoint токен будет подставляться автоматически.

## 3) Сценарий тестирования

### 3.1 Регистрация

**POST** `{{base_url}}/api/v1/auth/register/`  
Body (raw JSON):

```json
{
  "username": "testuser",
  "email": "{{email}}",
  "password": "{{password}}",
  "password_confirm": "{{password}}",
  "first_name": "Test",
  "last_name": "User"
}
```

Ожидание:
- `201 Created`
- В ответе есть `access`, `refresh`, `user`

Во вкладке **Tests**:

```javascript
pm.test("Status is 201", () => pm.response.to.have.status(201));
const json = pm.response.json();
pm.environment.set("access_token", json.access);
pm.environment.set("refresh_token", json.refresh);
```

### 3.2 Логин

**POST** `{{base_url}}/api/v1/auth/login/`  
Body:

```json
{
  "email": "{{email}}",
  "password": "{{password}}"
}
```

Ожидание:
- `200 OK`
- Новые `access` и `refresh`

Tests:

```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
const json = pm.response.json();
pm.environment.set("access_token", json.access);
pm.environment.set("refresh_token", json.refresh);
```

### 3.3 Профиль (просмотр)

**GET** `{{base_url}}/api/v1/auth/profile/`  
(с Bearer `{{access_token}}`)

Ожидание:
- `200 OK`
- JSON с полями пользователя (`email`, `first_name`, `last_name` и т.д.)

### 3.4 Профиль (полное обновление)

**PUT** `{{base_url}}/api/v1/auth/profile/`  
Body:

```json
{
  "first_name": "Updated",
  "last_name": "Name",
  "bio": "Updated bio from Postman"
}
```

Ожидание:
- `200 OK`
- Значения обновлены

### 3.5 Профиль (частичное обновление)

**PATCH** `{{base_url}}/api/v1/auth/profile/`  
Body:

```json
{
  "bio": "Patched bio"
}
```

Ожидание:
- `200 OK`
- Меняется только переданное поле

### 3.6 Обновление access токена

**POST** `{{base_url}}/api/v1/auth/token/refresh/`  
Body:

```json
{
  "refresh": "{{refresh_token}}"
}
```

Ожидание:
- `200 OK`
- Новый `access` (и, возможно, новый `refresh`, т.к. включен rotate)

Tests:

```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
const json = pm.response.json();
if (json.access) pm.environment.set("access_token", json.access);
if (json.refresh) pm.environment.set("refresh_token", json.refresh);
```

### 3.7 Смена пароля

**PUT** `{{base_url}}/api/v1/auth/change-password/`  
Body:

```json
{
  "old_password": "{{password}}",
  "new_password": "{{new_password}}",
  "new_password_confirm": "{{new_password}}"
}
```

Ожидание:
- `200 OK`
- `message: Password changed succesfully.`

После этого:
- обновите переменную `password = {{new_password}}`
- проверьте логин с новым паролем (`/login/`)

### 3.8 Логаут (blacklist refresh)

**POST** `{{base_url}}/api/v1/auth/logout/`  
Body:

```json
{
  "refresh_token": "{{refresh_token}}"
}
```

Ожидание:
- `200 OK`
- `message: Logout succesful.`

Дополнительная проверка:
- повторный `POST /token/refresh/` со старым refresh должен дать ошибку (`400`/`401`)

## 4) Негативные тесты (обязательно)

Минимальный набор:
- `register`: разные `password` и `password_confirm` -> `400`
- `login`: неверный пароль -> `400`
- `profile` без Bearer токена -> `401`
- `change-password`: неверный `old_password` -> `400`
- `token/refresh`: битый или пустой refresh -> `401/400`
- `logout`: невалидный `refresh_token` -> `400`

## 5) Рекомендуемая структура коллекции

Папки в Postman:
- `Auth Positive`
- `Auth Negative`
- `Profile`
- `Tokens`

Названия запросов:
- `[POS] Register`
- `[POS] Login`
- `[NEG] Login wrong password`
- и т.д.
