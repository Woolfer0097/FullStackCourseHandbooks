# Handbook. Streamlit Frontend + FastAPI

## Что получится в конце занятия

Рабочий frontend на Streamlit, который сразу взаимодействует с FastAPI:

```text
Streamlit -> requests -> FastAPI -> Service -> Repository -> Database
```

В проекте будут:

- регистрация и вход;
- хранение JWT-токена в `st.session_state`;
- профиль пользователя;
- каталог и подробная страница записи;
- избранное;
- создание, редактирование и удаление записей для администратора.

Frontend не обращается к базе данных напрямую. Все данные он получает через HTTP-запросы к backend.

---

# Шаг 1. Проверить контракт backend

В примере используются следующие endpoint’ы:

```text
POST   /auth/register/
POST   /auth/login/
GET    /users/me/

GET    /items/
GET    /items/{item_id}/
POST   /items/                  admin
PATCH  /items/{item_id}/        admin
DELETE /items/{item_id}/        admin

GET    /favorites/
POST   /favorites/{item_id}/
DELETE /favorites/{item_id}/
```

Регистрация отправляет:

```json
{
  "email": "user@example.com",
  "password": "secret-password",
  "full_name": "Иван Иванов"
}
```

Вход отправляет:

```json
{
  "email": "user@example.com",
  "password": "secret-password"
}
```

Успешный вход возвращает:

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
```

Профиль пользователя возвращает:

```json
{
  "id": 1,
  "email": "user@example.com",
  "full_name": "Иван Иванов",
  "role": "user"
}
```

Одна запись каталога возвращается в таком формате:

```json
{
  "id": 1,
  "title": "Название",
  "short_description": "Краткое описание",
  "description": "Полное описание",
  "image_url": null,
  "is_favorite": false
}
```

Названия endpoint’ов и полей во frontend должны полностью совпадать с backend.

---

# Шаг 2. Создать структуру frontend

```text
frontend/
├── .gitignore
├── requirements.txt
├── app.py
├── api/
│   ├── __init__.py
│   └── client.py
├── auth/
│   ├── __init__.py
│   └── state.py
├── components/
│   ├── __init__.py
│   └── item_card.py
├── pages/
│   ├── catalog.py
│   ├── details.py
│   ├── favorites.py
│   ├── create_item.py
│   ├── edit_item.py
│   ├── login.py
│   ├── registration.py
│   └── profile.py
└── .streamlit/
    └── config.toml
```

Файлы `__init__.py` оставить пустыми.

---

# Шаг 3. Установить зависимости

Файл `frontend/requirements.txt`:

```text
streamlit==1.58.0
requests==2.32.5
```

Создание окружения и установка:

```bash
cd frontend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows:

```powershell
.venv\Scripts\activate
pip install -r requirements.txt
```

Файл `frontend/.gitignore`:

```gitignore
.venv/
__pycache__/
*.pyc
```

---

# Шаг 4. Создать HTTP-клиент

Все запросы к FastAPI находятся в одном файле.

Файл `frontend/api/client.py`:

```python
from streamlit import session_state

import requests


BACKEND_URL = "http://127.0.0.1:8000"

LOGIN_ENDPOINT = f"{BACKEND_URL}/auth/login/"
REGISTER_ENDPOINT = f"{BACKEND_URL}/auth/register/"
PROFILE_ENDPOINT = f"{BACKEND_URL}/users/me/"
ITEMS_ENDPOINT = f"{BACKEND_URL}/items/"
FAVORITES_ENDPOINT = f"{BACKEND_URL}/favorites/"


def register(email: str, password: str, full_name: str) -> requests.Response:
    data = {
        "email": email,
        "password": password,
        "full_name": full_name,
    }
    return requests.post(REGISTER_ENDPOINT, json=data)


def login(email: str, password: str) -> requests.Response:
    data = {
        "email": email,
        "password": password,
    }
    return requests.post(LOGIN_ENDPOINT, json=data)


def request_with_authorization_header(
    request_type: str,
    endpoint: str,
    params: dict | None = None,
    payload: dict | None = None,
) -> requests.Response:
    headers = {
        "Authorization": f"Bearer {session_state['access_token']}"
    }

    if request_type == "GET":
        response = requests.get(endpoint, headers=headers, params=params)
    elif request_type == "POST":
        response = requests.post(endpoint, headers=headers, params=params, json=payload)
    elif request_type == "PATCH":
        response = requests.patch(endpoint, headers=headers, params=params, json=payload)
    elif request_type == "DELETE":
        response = requests.delete(endpoint, headers=headers, params=params)
    else:
        raise ValueError("Неизвестный тип запроса")

    # Если JWT больше не действителен, удаляем авторизацию.
    if response.status_code == 401:
        session_state.pop("access_token", None)
        session_state.pop("profile", None)

    return response


def get_error_message(response: requests.Response) -> str:
    try:
        detail = response.json().get("detail")
        return str(detail or f"Ошибка backend: HTTP {response.status_code}")
    except ValueError:
        return f"Ошибка backend: HTTP {response.status_code}"


def get_profile() -> requests.Response:
    return request_with_authorization_header("GET", PROFILE_ENDPOINT)


def get_items() -> requests.Response:
    # Авторизованному пользователю backend вернёт его is_favorite.
    if session_state.get("access_token"):
        return request_with_authorization_header("GET", ITEMS_ENDPOINT)
    return requests.get(ITEMS_ENDPOINT)


def get_item(item_id: int) -> requests.Response:
    endpoint = f"{ITEMS_ENDPOINT}{item_id}/"

    if session_state.get("access_token"):
        return request_with_authorization_header("GET", endpoint)
    return requests.get(endpoint)


def get_favorites() -> requests.Response:
    return request_with_authorization_header("GET", FAVORITES_ENDPOINT)


def add_favorite(item_id: int) -> requests.Response:
    endpoint = f"{FAVORITES_ENDPOINT}{item_id}/"
    return request_with_authorization_header("POST", endpoint)


def remove_favorite(item_id: int) -> requests.Response:
    endpoint = f"{FAVORITES_ENDPOINT}{item_id}/"
    return request_with_authorization_header("DELETE", endpoint)


def create_item(payload: dict) -> requests.Response:
    return request_with_authorization_header(
        "POST",
        ITEMS_ENDPOINT,
        payload=payload,
    )


def update_item(item_id: int, payload: dict) -> requests.Response:
    endpoint = f"{ITEMS_ENDPOINT}{item_id}/"
    return request_with_authorization_header(
        "PATCH",
        endpoint,
        payload=payload,
    )


def delete_item(item_id: int) -> requests.Response:
    endpoint = f"{ITEMS_ENDPOINT}{item_id}/"
    return request_with_authorization_header("DELETE", endpoint)
```

Если backend использует другие пути, изменить только константы endpoint’ов в начале файла.

Ошибку соединения с backend обрабатываем на той странице, где выполняется запрос:

```python
try:
    response = login(email, password)
except requests.RequestException:
    st.error("Backend недоступен")
    st.stop()
```

---

# Шаг 5. Создать функции состояния авторизации

Файл `frontend/auth/state.py`:

```python
import streamlit as st


def save_auth(access_token: str, profile: dict) -> None:
    st.session_state["access_token"] = access_token
    st.session_state["profile"] = profile


def clear_auth() -> None:
    st.session_state.pop("access_token", None)
    st.session_state.pop("profile", None)


def is_authenticated() -> bool:
    return bool(st.session_state.get("access_token"))


def current_profile() -> dict | None:
    return st.session_state.get("profile")


def is_admin() -> bool:
    profile = current_profile()
    return bool(profile and profile.get("role") == "admin")


def require_login() -> None:
    if is_authenticated():
        return

    st.warning("Сначала войдите в аккаунт.")
    if st.button("Перейти ко входу", key="require_login_button"):
        st.switch_page("pages/login.py")
    st.stop()


def require_admin() -> None:
    require_login()

    if not is_admin():
        st.error("Эта страница доступна только администратору.")
        st.stop()
```

JWT хранится в памяти текущей вкладки Streamlit. После закрытия сессии или очистки состояния пользователь должен войти снова.

---

# Шаг 6. Настроить приложение и навигацию

Файл `frontend/.streamlit/config.toml`:

```toml
[client]
showSidebarNavigation = false

[theme]
primaryColor = "#2563EB"
```

Файл `frontend/app.py`:

```python
import streamlit as st

from auth.state import is_admin


st.set_page_config(
    page_title="Каталог",
    page_icon="📚",
    layout="wide",
)

pages = {
    "Каталог": [
        st.Page(
            "pages/catalog.py",
            title="Каталог",
            icon=":material/store:",
            url_path="catalog",
            default=True,
        ),
        st.Page(
            "pages/details.py",
            title="Подробнее",
            icon=":material/article:",
            url_path="details",
        ),
    ],
    "Пользователь": [
        st.Page(
            "pages/favorites.py",
            title="Избранное",
            icon=":material/favorite:",
            url_path="favorites",
        ),
        st.Page(
            "pages/profile.py",
            title="Профиль",
            icon=":material/person:",
            url_path="profile",
        ),
    ],
    "Авторизация": [
        st.Page(
            "pages/login.py",
            title="Вход",
            icon=":material/login:",
            url_path="login",
        ),
        st.Page(
            "pages/registration.py",
            title="Регистрация",
            icon=":material/person_add:",
            url_path="registration",
        ),
    ],
}

if is_admin():
    pages["Администратор"] = [
        st.Page(
            "pages/create_item.py",
            title="Создать запись",
            icon=":material/add:",
            url_path="create-item",
        ),
        st.Page(
            "pages/edit_item.py",
            title="Редактировать запись",
            icon=":material/edit:",
            url_path="edit-item",
        ),
    ]

navigation = st.navigation(pages)
navigation.run()
```

Для каждой страницы указан уникальный `url_path`. Это предотвращает ошибку о повторяющихся URL страниц.

Запуск frontend:

```bash
streamlit run app.py
```

---

# Шаг 7. Создать страницу регистрации

Файл `frontend/pages/registration.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, register


st.header("Регистрация")

with st.form("registration_form"):
    full_name = st.text_input("ФИО", key="registration_full_name")
    email = st.text_input("Почта", key="registration_email")
    password = st.text_input(
        "Пароль",
        type="password",
        key="registration_password",
    )
    submitted = st.form_submit_button("Зарегистрироваться")

if submitted:
    if not full_name.strip() or not email.strip() or not password:
        st.error("Заполните все поля.")
        st.stop()

    try:
        response = register(
            email=email.strip(),
            password=password,
            full_name=full_name.strip(),
        )
    except requests.RequestException:
        st.error("Backend недоступен. Проверьте, запущен ли FastAPI.")
        st.stop()

    if response.status_code in (200, 201):
        st.success("Регистрация выполнена. Теперь войдите в аккаунт.")
        st.switch_page("pages/login.py")
    else:
        st.error(get_error_message(response))

st.page_link("pages/login.py", label="Уже есть аккаунт? Войти")
```

Не нужно вручную присваивать значение `st.session_state["registration_full_name"]` после создания `st.text_input` с таким же ключом. Streamlit сам управляет значением виджета.

---

# Шаг 8. Создать страницу входа

После получения JWT сразу запрашиваем профиль. Авторизация сохраняется только тогда, когда успешно получены и токен, и пользователь.

Файл `frontend/pages/login.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_profile, login
from auth.state import clear_auth, is_authenticated, save_auth


st.header("Вход")

if is_authenticated():
    st.info("Вы уже вошли в аккаунт.")
    st.page_link("pages/profile.py", label="Открыть профиль")
    st.stop()

with st.form("login_form"):
    email = st.text_input("Почта", key="login_email")
    password = st.text_input(
        "Пароль",
        type="password",
        key="login_password",
    )
    submitted = st.form_submit_button("Войти")

if submitted:
    if not email.strip() or not password:
        st.error("Введите почту и пароль.")
        st.stop()

    try:
        login_response = login(email.strip(), password)
    except requests.RequestException:
        st.error("Backend недоступен. Проверьте, запущен ли FastAPI.")
        st.stop()

    if not login_response.ok:
        st.error(get_error_message(login_response))
        st.stop()

    access_token = login_response.json().get("access_token")

    if not access_token:
        st.error("Backend не вернул access_token.")
        st.stop()

    # Временно сохраняем JWT, чтобы запросить защищённый профиль.
    st.session_state["access_token"] = access_token

    try:
        profile_response = get_profile()
    except requests.RequestException:
        clear_auth()
        st.error("Не удалось получить профиль пользователя.")
        st.stop()

    if not profile_response.ok:
        error_message = get_error_message(profile_response)
        clear_auth()
        st.error(error_message)
        st.stop()

    save_auth(access_token, profile_response.json())
    st.success("Вход выполнен.")
    st.switch_page("pages/catalog.py")

st.page_link("pages/registration.py", label="Нет аккаунта? Зарегистрироваться")
```

---

# Шаг 9. Создать страницу профиля и выход

Файл `frontend/pages/profile.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_profile
from auth.state import clear_auth, require_login, save_auth


require_login()
st.header("Профиль")

try:
    response = get_profile()
except requests.RequestException:
    st.error("Не удалось выполнить запрос к backend.")
    st.stop()

if not response.ok:
    st.error(get_error_message(response))
    st.stop()

profile = response.json()
save_auth(st.session_state["access_token"], profile)

st.write(f"**ФИО:** {profile.get('full_name', 'Не указано')}")
st.write(f"**Почта:** {profile.get('email', 'Не указана')}")
st.write(f"**Роль:** {profile.get('role', 'user')}")

if st.button("Выйти", type="primary"):
    clear_auth()
    st.switch_page("pages/login.py")
```

---

# Шаг 10. Создать карточку записи

Файл `frontend/components/item_card.py`:

```python
import requests
import streamlit as st

from api.client import (
    add_favorite,
    delete_item,
    get_error_message,
    remove_favorite,
)
from auth.state import is_admin, is_authenticated


def render_favorite_button(item: dict, key_prefix: str) -> None:
    if not is_authenticated():
        st.caption("Войдите, чтобы добавить запись в избранное.")
        return

    item_id = item["id"]
    is_favorite = item.get("is_favorite", False)
    button_text = "Убрать из избранного" if is_favorite else "В избранное"

    if st.button(button_text, key=f"{key_prefix}_favorite_{item_id}"):
        try:
            if is_favorite:
                response = remove_favorite(item_id)
            else:
                response = add_favorite(item_id)
        except requests.RequestException:
            st.error("Не удалось выполнить запрос к backend.")
            return

        if response.ok:
            st.rerun()
        else:
            st.error(get_error_message(response))


def render_admin_actions(item_id: int, key_prefix: str) -> None:
    if not is_admin():
        return

    edit_column, delete_column = st.columns(2)

    if edit_column.button(
        "Редактировать",
        key=f"{key_prefix}_edit_{item_id}",
    ):
        st.session_state["edit_item_id"] = item_id
        st.switch_page("pages/edit_item.py")

    if delete_column.button(
        "Удалить",
        key=f"{key_prefix}_delete_{item_id}",
        type="primary",
    ):
        try:
            response = delete_item(item_id)
        except requests.RequestException:
            st.error("Не удалось выполнить запрос к backend.")
            return

        if response.ok:
            st.success("Запись удалена.")
            st.switch_page("pages/catalog.py")
        else:
            st.error(get_error_message(response))


def render_item_card(item: dict) -> None:
    item_id = item["id"]

    with st.container(border=True):
        if item.get("image_url"):
            st.image(item["image_url"], use_container_width=True)
        else:
            st.info("Изображение не добавлено")

        st.subheader(item["title"])
        st.write(item.get("short_description", ""))

        render_favorite_button(item, key_prefix="card")

        if st.button("Подробнее", key=f"card_details_{item_id}"):
            st.session_state["selected_item_id"] = item_id
            st.switch_page("pages/details.py")

        render_admin_actions(item_id, key_prefix="card")
```

`key` каждой кнопки содержит `item_id`, поэтому Streamlit не считает кнопки одинаковыми.

---

# Шаг 11. Создать страницу каталога

Файл `frontend/pages/catalog.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_items
from auth.state import is_admin
from components.item_card import render_item_card


st.header("Каталог")

if is_admin():
    if st.button("Создать запись"):
        st.switch_page("pages/create_item.py")

try:
    response = get_items()
except requests.RequestException:
    st.error("Backend недоступен. Проверьте запуск FastAPI.")
    st.stop()

if not response.ok:
    st.error(get_error_message(response))
    st.stop()

items = response.json()

if not items:
    st.info("В каталоге пока нет записей.")
    st.stop()

columns = st.columns(3)

for index, item in enumerate(items):
    with columns[index % 3]:
        render_item_card(item)
```

---

# Шаг 12. Создать подробную страницу

Файл `frontend/pages/details.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_item
from components.item_card import render_admin_actions, render_favorite_button


item_id = st.session_state.get("selected_item_id")

if item_id is None:
    st.info("Сначала выберите запись в каталоге.")
    st.page_link("pages/catalog.py", label="Перейти в каталог")
    st.stop()

try:
    response = get_item(item_id)
except requests.RequestException:
    st.error("Не удалось выполнить запрос к backend.")
    st.stop()

if not response.ok:
    st.error(get_error_message(response))
    st.stop()

item = response.json()

st.header(item["title"])

if item.get("image_url"):
    st.image(item["image_url"], width=500)

st.write(item.get("description", ""))
render_favorite_button(item, key_prefix="details")
render_admin_actions(item["id"], key_prefix="details")
```

---

# Шаг 13. Создать страницу избранного

Файл `frontend/pages/favorites.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_favorites
from auth.state import require_login
from components.item_card import render_item_card


require_login()
st.header("Избранное")

try:
    response = get_favorites()
except requests.RequestException:
    st.error("Не удалось выполнить запрос к backend.")
    st.stop()

if not response.ok:
    st.error(get_error_message(response))
    st.stop()

items = response.json()

if not items:
    st.info("В избранном пока ничего нет.")
    st.stop()

for item in items:
    render_item_card(item)
```

---

# Шаг 14. Создать страницу добавления записи

Файл `frontend/pages/create_item.py`:

```python
import requests
import streamlit as st

from api.client import create_item, get_error_message
from auth.state import require_admin


require_admin()
st.header("Новая запись")

with st.form("create_item_form"):
    title = st.text_input("Название")
    short_description = st.text_area("Краткое описание")
    description = st.text_area("Полное описание")
    image_url = st.text_input("Ссылка на изображение")
    submitted = st.form_submit_button("Создать")

if submitted:
    if not title.strip():
        st.error("Укажите название.")
        st.stop()

    payload = {
        "title": title.strip(),
        "short_description": short_description.strip(),
        "description": description.strip(),
        "image_url": image_url.strip() or None,
    }

    try:
        response = create_item(payload)
    except requests.RequestException:
        st.error("Не удалось выполнить запрос к backend.")
        st.stop()

    if response.status_code in (200, 201):
        created_item = response.json()
        st.session_state["selected_item_id"] = created_item["id"]
        st.switch_page("pages/details.py")
    else:
        st.error(get_error_message(response))
```

Frontend проверяет только очевидные ошибки формы. Полную валидацию данных и роль пользователя повторно проверяет FastAPI.

---

# Шаг 15. Создать страницу редактирования

Файл `frontend/pages/edit_item.py`:

```python
import requests
import streamlit as st

from api.client import get_error_message, get_item, update_item
from auth.state import require_admin


require_admin()
st.header("Редактирование записи")

item_id = st.session_state.get("edit_item_id")

if item_id is None:
    st.info("Сначала выберите запись для редактирования.")
    st.stop()

try:
    item_response = get_item(item_id)
except requests.RequestException:
    st.error("Не удалось получить запись с backend.")
    st.stop()

if not item_response.ok:
    st.error(get_error_message(item_response))
    st.stop()

item = item_response.json()

with st.form(f"edit_item_form_{item_id}"):
    title = st.text_input("Название", value=item["title"])
    short_description = st.text_area(
        "Краткое описание",
        value=item.get("short_description", ""),
    )
    description = st.text_area(
        "Полное описание",
        value=item.get("description", ""),
    )
    image_url = st.text_input(
        "Ссылка на изображение",
        value=item.get("image_url") or "",
    )
    submitted = st.form_submit_button("Сохранить")

if submitted:
    if not title.strip():
        st.error("Укажите название.")
        st.stop()

    payload = {
        "title": title.strip(),
        "short_description": short_description.strip(),
        "description": description.strip(),
        "image_url": image_url.strip() or None,
    }

    try:
        response = update_item(item_id, payload)
    except requests.RequestException:
        st.error("Не удалось выполнить запрос к backend.")
        st.stop()

    if response.ok:
        st.session_state["selected_item_id"] = item_id
        st.switch_page("pages/details.py")
    else:
        st.error(get_error_message(response))
```

---

# Шаг 16. Запустить backend и frontend

Первый терминал, из корня backend:

```bash
uvicorn app.main:app --reload
```

Второй терминал:

```bash
cd frontend
source .venv/bin/activate
streamlit run app.py
```

Windows:

```powershell
cd frontend
.venv\Scripts\activate
streamlit run app.py
```

Адреса для проверки:

```text
FastAPI Swagger: http://127.0.0.1:8000/docs
Streamlit:       http://localhost:8501
```

---

# Шаг 17. Проверить результат

## Авторизация

- регистрация создаёт пользователя в backend;
- вход возвращает JWT;
- после входа загружается профиль;
- JWT сохраняется в `st.session_state`;
- выход удаляет JWT и профиль;
- при ответе `401` данные авторизации очищаются.

## Каталог

- данные загружаются из `GET /items/`;
- подробная страница получает одну запись по `id`;
- отсутствие изображения не ломает карточку;
- у каждой кнопки уникальный `key`.

## Избранное

- анонимный пользователь не может изменять избранное;
- авторизованный пользователь добавляет и удаляет записи;
- после действия выполняется `st.rerun()` и состояние карточки обновляется.

## Администратор

- admin видит страницы создания и редактирования;
- create, update и delete отправляют JWT;
- обычный пользователь не проходит `require_admin()`;
- FastAPI повторно проверяет роль на каждом admin endpoint.

## Связь с backend

- названия endpoint’ов совпадают;
- названия полей JSON совпадают;
- backend запущен на адресе из `BACKEND_URL`;
- frontend не содержит mock-данных и не обращается к базе напрямую.
