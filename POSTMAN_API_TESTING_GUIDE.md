# Руководство по тестированию API в Postman

Ниже описан практический сценарий тестирования всех текущих endpoint вашего API (приложения **accounts** и **main**).

## 1) Доступные endpoint

### 1.1 Auth (`apps.accounts`)

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

### 1.2 Main — посты и категории (`apps.main`)

Базовый префикс:
- `{{base_url}}/api/v1/posts/`

Категории:
- `GET/POST /categories/` — список и создание
- `GET/PUT/PATCH/DELETE /categories/<slug>/` — одна категория (по `slug`)
- `GET /categories/<category_slug>/posts/` — опубликованные посты в категории

Посты:
- `GET/POST /` — список и создание (корень префикса `/api/v1/posts/`)
- `GET /my-posts/` — посты текущего пользователя (только с авторизацией)
- `GET /popular/` — до 10 популярных по `views_count`
- `GET /recent/` — до 10 последних опубликованных
- `GET/PUT/PATCH/DELETE /<slug>/` — детали, правки, удаление (редактирование — только автор)

## 2) Подготовка Postman

Создайте Environment (например, `local`) и переменные:
- `base_url` = `http://127.0.0.1:8000`
- `access_token` = (пусто)
- `refresh_token` = (пусто)
- `email` = `testuser@example.com`
- `password` = `TestPass123!`
- `new_password` = `NewPass123!`
- `category_slug` = (пусто; заполнить после создания категории)
- `category_id` = (пусто; числовой `id` для привязки поста)
- `post_slug` = (пусто; `slug` поста для деталей и правок)

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

### 3.9 Main — общие замечания

1. Выполните **регистрацию или логин** (п. 3.1–3.2), чтобы получить `access_token` для **POST/PUT/PATCH/DELETE** по категориям и постам (где требуется автор).
2. Публичные **GET** (`/popular/`, `/recent/`, список постов, детали поста, посты по категории) разрешены без токена. Если в коллекции везде включён Bearer, а токена нет, для анонимных запросов на уровне **отдельного запроса** выставьте **Authorization → No Auth**, иначе возможны лишние `401`.
3. После **POST** создания поста ответ сериализатора содержит поля `title`, `content`, `image`, `category`, `status` (без `slug`). Чтобы взять `slug` для следующих шагов, сразу вызовите **GET** `.../my-posts/` и сохраните `slug` из нужного элемента (см. п. 3.18 и пример Tests ниже).

### 3.10 Список категорий

**GET** `{{base_url}}/api/v1/posts/categories/`  

Опционально query: `?search=техно`, `?ordering=name`, `?ordering=-created_at`.

Ожидание: `200 OK`, массив объектов с полями `id`, `name`, `slug`, `description`, `posts_count`, `created_at`.

### 3.11 Создание категории

**POST** `{{base_url}}/api/v1/posts/categories/`  
(с Bearer `{{access_token}}`)  
Body (raw JSON):

```json
{
  "name": "Technology",
  "description": "Tech news"
}
```

Ожидание: `201 Created`, в теле ответа есть `slug` и `id`.

Tests (сохранить slug и id):

```javascript
pm.test("Status is 201", () => pm.response.to.have.status(201));
const json = pm.response.json();
pm.environment.set("category_slug", json.slug);
pm.environment.set("category_id", String(json.id));
```

### 3.12 Категория по slug (просмотр / правка / удаление)

**GET** `{{base_url}}/api/v1/posts/categories/{{category_slug}}/` — без токена допустимо.

**PUT** `{{base_url}}/api/v1/posts/categories/{{category_slug}}/` — с Bearer, полное тело, например:

```json
{
  "name": "Technology",
  "description": "Updated description"
}
```

**PATCH** — частичное обновление тех же полей.

**DELETE** — удаление категории (посты с этой категорией получат `category: null` из-за `SET_NULL` в модели).

### 3.13 Посты в категории

**GET** `{{base_url}}/api/v1/posts/categories/{{category_slug}}/posts/`  

Ожидание: `200 OK`, объект вида `{ "category": { ... }, "posts": [ ... ] }` — только **опубликованные** посты.

### 3.14 Список постов (корень API постов)

**GET** `{{base_url}}/api/v1/posts/`  

Query (опционально):
- фильтры: `category`, `author`, `status` (например `?status=published&category=1`);
- поиск: `?search=заголовок`;
- сортировка: `?ordering=-created_at`, `views_count`, `title` и т.д.

Поведение: **без** авторизации в выдаче только `status=published`. **С** токеном — опубликованные и **ваши** черновики (`draft`).

### 3.15 Создание поста

**POST** `{{base_url}}/api/v1/posts/`  
(с Bearer)  

JSON (категория по числовому `id`; если переменная `category_id` ещё пуста, поставьте `"category": null` или уберите поле):

```json
{
  "title": "My first post",
  "content": "Полный текст статьи. Для списка на сервере контент может обрезаться до ~200 символов.",
  "category": 1,
  "status": "draft"
}
```

(подставьте реальный id вместо `1` или используйте `"category": {{category_id}}`, когда в environment уже записан числовой id.)

Для загрузки **картинки** удобнее **body → form-data**: поля `title`, `content`, `category`, `status` (Text) и `image` (тип **File**).

Ожидание: `201 Created`.

Tests (сохранить `post_slug` из списка «мои посты», т.к. в ответе POST может не быть `slug`):

```javascript
pm.test("Status is 201", () => pm.response.to.have.status(201));
```

После этого отдельным запросом **GET** `{{base_url}}/api/v1/posts/my-posts/` найдите созданный пост и во вкладке **Tests** последнего запроса:

```javascript
const items = pm.response.json();
if (Array.isArray(items) && items.length) {
  pm.environment.set("post_slug", items[0].slug);
}
```

(либо выберите элемент по `title` вручную в UI Postman и скопируйте `slug` в переменную `post_slug`).

### 3.16 Детальный просмотр поста

**GET** `{{base_url}}/api/v1/posts/{{post_slug}}/`  

Каждый успешный **GET** увеличивает `views_count` на сервере. Ожидание: `200 OK`, расширенные поля (`author_info`, `category_info` и т.д.).

### 3.17 Изменение поста

**PUT** или **PATCH** `{{base_url}}/api/v1/posts/{{post_slug}}/`  
Только **автор** поста (иначе `403`). Тело в формате как при создании (для PATCH — только изменяемые поля).

### 3.18 Мои посты

**GET** `{{base_url}}/api/v1/posts/my-posts/`  
Только с Bearer. Фильтры: `?category=`, `?status=`, плюс `search`, `ordering` как у списка.

### 3.19 Популярные и свежие посты

**GET** `{{base_url}}/api/v1/posts/popular/` — до 10 постов по убыванию `views_count`.

**GET** `{{base_url}}/api/v1/posts/recent/` — до 10 последних опубликованных.

Оба запроса публичные (`AllowAny`).

### 3.20 Удаление поста

**DELETE** `{{base_url}}/api/v1/posts/{{post_slug}}/` — только автор.

## 4) Негативные тесты (обязательно)

Минимальный набор:
- `register`: разные `password` и `password_confirm` -> `400`
- `login`: неверный пароль -> `400`
- `profile` без Bearer токена -> `401`
- `change-password`: неверный `old_password` -> `400`
- `token/refresh`: битый или пустой refresh -> `401/400`
- `logout`: невалидный `refresh_token` -> `400`

Дополнительно для **main**:
- `POST /api/v1/posts/categories/` без Bearer -> `401`
- `POST /api/v1/posts/` без Bearer -> `401`
- `GET /api/v1/posts/my-posts/` без Bearer -> `401`
- `PUT/PATCH/DELETE` чужого поста (`/{{post_slug}}/`) под другим пользователем -> `403`
- `GET /api/v1/posts/categories/non-existent-slug/posts/` -> `404`
- невалидные данные при создании (пустой `title` и т.п.) -> `400`

## 5) Рекомендуемая структура коллекции

Папки в Postman:
- `Auth Positive`
- `Auth Negative`
- `Profile`
- `Tokens`
- `Main Categories` (список, создание, деталь, посты по категории, удаление)
- `Main Posts` (список, создание, деталь, мои посты, popular, recent, правки, удаление)
- `Main Negative`

Названия запросов:
- `[POS] Register`
- `[POS] Login`
- `[NEG] Login wrong password`
- `[POS] Main Create Category`
- `[POS] Main Create Post`
- `[POS] Main My Posts`
- и т.д.
