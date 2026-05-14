# Руководство по тестированию API в Postman

Ниже описан практический сценарий тестирования всех текущих endpoint вашего API (приложения **apps.accounts**, **apps.main** и **apps.comments**).

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

### 1.3 Comments — комментарии к постам (`apps.comments`)

Базовый префикс:
- `{{base_url}}/api/v1/comments/`

Endpoint'ы:
- `GET/POST /` — список (фильтры, поиск, сортировка) и создание комментария (`POST` только с авторизацией)
- `GET/PUT/PATCH/DELETE /<id>/` — детали, правка текста, «удаление» (мягкое: `is_active=false`; правки — только автор)
- `GET /my-comments/` — комментарии текущего пользователя (только с авторизацией)
- `GET /post/<post_id>/` — дерево комментариев к **опубликованному** посту (корневые комментарии с вложенными `replies`)
- `GET /<comment_id>/replies/` — ответы на конкретный комментарий (плоский список + метаданные родителя)

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
- `post_id` = (пусто; числовой `id` **опубликованного** поста для комментариев)
- `comment_id` = (пусто; числовой `id` комментария для деталей, правок и ответов)

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

Tests (опционально — сохранить `post_id` для сценария комментариев):

```javascript
const json = pm.response.json();
if (json.id) pm.environment.set("post_id", String(json.id));
```

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

### 3.21 Comments — общие замечания

1. Для **POST** комментария нужен Bearer `{{access_token}}`. **GET** списка (`/`) и деталей (`/<id>/`) разрешены без токена (чтение для всех).
2. Комментарий можно создать только к посту со `status=published` (иначе валидация сериализатора → `400`).
3. В теле ответа **POST /** сериализатор создания отдаёт в основном `post`, `parent`, `content`. Числовой **`id`** нового комментария удобно взять из **GET** `.../my-comments/` или **GET** `.../?post={{post_id}}` (см. примеры Tests ниже).
4. Перед сценарием комментариев опубликуйте пост (`PATCH` поста с `"status": "published"`) или создайте пост сразу с `"status": "published"`, и сохраните его **`id`** в `post_id` (например из **GET** `.../posts/{{post_slug}}/` в ответе есть `id`).

### 3.22 Список комментариев

**GET** `{{base_url}}/api/v1/comments/`  

Query (опционально):
- фильтры: `?post=`, `?author=`, `?parent=` (для корневых к посту часто `?parent=` пусто не фильтрует — для «только корневые» используйте **GET** `/post/<post_id>/`);
- поиск: `?search=текст`;
- сортировка: `?ordering=-created_at`, `updated_at` и т.д.

Ожидание: `200 OK`, массив объектов с полями вроде `id`, `content`, `author_info`, `parent`, `replies_count`, `is_reply`, `created_at`, `updated_at`.

### 3.23 Создание комментария

**POST** `{{base_url}}/api/v1/comments/`  
(с Bearer)  
Body (raw JSON), корневой комментарий:

```json
{
  "post": 1,
  "parent": null,
  "content": "Отличная статья, спасибо!"
}
```

Подставьте реальный id опубликованного поста вместо `1` или `"post": {{post_id}}` (в Postman для числа можно оставить без кавычек в raw JSON, подставив значение вручную).

Ответ на другой комментарий (тот же пост):

```json
{
  "post": 1,
  "parent": 5,
  "content": "Согласен с автором комментария выше."
}
```

`parent` должен относиться к тому же `post`, иначе `400`.

Ожидание: `201 Created`.

Tests (сохранить `comment_id` из списка по посту):

```javascript
pm.test("Status is 201", () => pm.response.to.have.status(201));
```

Затем **GET** `{{base_url}}/api/v1/comments/?post={{post_id}}` (подставьте число вручную, если переменная не подставляется в query), сортировка по умолчанию новые сверху — в **Tests**:

```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
const items = pm.response.json();
if (Array.isArray(items) && items.length) {
  pm.environment.set("comment_id", String(items[0].id));
}
```

### 3.24 Детальный просмотр комментария

**GET** `{{base_url}}/api/v1/comments/{{comment_id}}/`  

Ожидание: `200 OK`, расширенные поля, для корневого комментария — вложенный массив `replies` (активные ответы).

### 3.25 Изменение комментария

**PUT** или **PATCH** `{{base_url}}/api/v1/comments/{{comment_id}}/`  
Только **автор** комментария (иначе `403` на запись). Тело для **PATCH**:

```json
{
  "content": "Обновлённый текст комментария"
}
```

Ожидание: `200 OK`.

### 3.26 Удаление комментария (мягкое)

**DELETE** `{{base_url}}/api/v1/comments/{{comment_id}}/` — только автор.

Ожидание: `204 No Content` — запись помечается `is_active=false` и больше не попадает в выдачу списков с активными комментариями.

### 3.27 Мои комментарии

**GET** `{{base_url}}/api/v1/comments/my-comments/`  
Только с Bearer. Фильтры: `?post=`, `?parent=`, `?is_active=true|false`, плюс `search`, `ordering`.

Tests (зафиксировать последний комментарий):

```javascript
const items = pm.response.json();
if (Array.isArray(items) && items.length) {
  pm.environment.set("comment_id", String(items[0].id));
}
```

### 3.28 Комментарии к посту (дерево для UI)

**GET** `{{base_url}}/api/v1/comments/post/{{post_id}}/`  

Публичный запрос. Ожидание: `200 OK`, JSON вида:

- `post`: `id`, `title`, `slug`
- `comments`: массив корневых комментариев с вложенными `replies`
- `comments_count`: число активных комментариев к посту

Если пост не опубликован или не существует → `404`.

### 3.29 Ответы на комментарий

**GET** `{{base_url}}/api/v1/comments/{{comment_id}}/replies/`  

Публичный запрос. Ожидание: `200 OK`, поля `parent_comment`, `replies`, `replies_count`. Если родитель не найден или неактивен → `404`.

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

Дополнительно для **comments**:
- `POST /api/v1/comments/` без Bearer -> `401`
- `GET /api/v1/comments/my-comments/` без Bearer -> `401`
- `POST /api/v1/comments/` с `post`, указывающим на черновик или несуществующий id -> `400`
- `PUT/PATCH/DELETE /api/v1/comments/<id>/` чужого комментария -> `403` (для методов с телом)
- `GET /api/v1/comments/post/<id>/` для неопубликованного или несуществующего поста -> `404`
- `POST` с `parent`, относящимся к другому посту, чем в `post` -> `400`

## 5) Рекомендуемая структура коллекции

Папки в Postman:
- `Auth Positive`
- `Auth Negative`
- `Profile`
- `Tokens`
- `Main Categories` (список, создание, деталь, посты по категории, удаление)
- `Main Posts` (список, создание, деталь, мои посты, popular, recent, правки, удаление)
- `Main Negative`
- `Comments` (список, создание, деталь, мои, по посту, ответы, правка, удаление)
- `Comments Negative`

Названия запросов:
- `[POS] Register`
- `[POS] Login`
- `[NEG] Login wrong password`
- `[POS] Main Create Category`
- `[POS] Main Create Post`
- `[POS] Main My Posts`
- `[POS] Comments Create`
- `[POS] Comments By Post`
- `[POS] Comments Replies`
- и т.д.
