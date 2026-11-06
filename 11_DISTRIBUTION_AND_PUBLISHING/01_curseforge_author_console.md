# 01. Публикация на CurseForge: новый Author Console API

> Проверено в 2026 году. Старый `www.curseforge.com/api` для загрузки файлов больше не работает; всё переехало в `authors.curseforge.com`, API живёт под `/_api/`, авторизация — cookie-сессия SSO.

## 1. Что умерло и почему это важно

| Попытка | Результат |
| :--- | :--- |
| `POST https://www.curseforge.com/api/projects/{id}/upload` | `404 Resource Not Found` даже с валидным author-токеном |
| `POST /api/upload`, `/api/files`, `/api/v1/mods/{id}/files` | `404` / `403 HTML` (CloudFront) |
| `X-Api-Token` (UUID с `legacy.curseforge.com/account/api-tokens`) | авторизация проходит, но маршрутов загрузки нет |
| API-ключ из `console.curseforge.com` (`$2a$10$...`) | валиден только для `api.curseforge.com` (чтение), авторский API его не принимает |

Вывод: ключи и токены для загрузки больше не нужны. Нужна сессия авторизованного браузера.

## 2. Новый backend

- Хост: `https://authors.curseforge.com`
- API: `https://authors.curseforge.com/_api/...`
- Без сессии: `401 {"statusCode":401,"message":"Unauthorized"}`
- С сессией: JSON. Признак жизни — `GET /_api/project-members/project-titles`.

## 3. Загрузка файла (обратный инжиниринг бандла)

### Шаг 1. Допустимые версии игры

```
GET /_api/project-files/{projectId}/get-game-versions
→ [{"value": 11457, "name": "1.21"}, ...]
```

В списке смешаны категории: версия Minecraft, загрузчик (`fabric`, `neoforge`), окружение (`client`), Java (`java 21`, `java 25`). В `gameVersionIds` нужно отправлять **числовые value**, а не имена.

### Шаг 2. Создать запись файла и получить `uploadId`

```
POST /_api/project-files/{projectId}/upload-file-details
Content-Type: application/json
```

```json
{
  "fileLength": 756000,
  "fileName": "MyMod-Fabric-1.21.1-v1.1.jar",
  "displayName": "MyMod Fabric 1.21.1 v1.1",
  "releaseType": 1,
  "gameVersionIds": [11779, 7499, 9638, 11135],
  "changelog": "…",
  "changelogType": 1,
  "isMarkedForManualRelease": false,
  "parentFileId": null,
  "addedProjectRelations": [],
  "includeDefaultProjectRelations": true
}
```

- `releaseType`: `1` = Release, `2` = Beta, `3` = Alpha.
- `changelogType`: `1` = текст, `5` = markdown (сервер сам конвертирует в HTML).
- `gameVersionIds`: например `[версия MC, загрузчик, "client", "java 21"/"java 25"]`. Без `client`/`java` файл опубликуется с неполными метаданными.
- Ответ: `{ "data": { "uploadId": "…" } }`.

### Шаг 3. Залить чанки

```
POST /_api/project-files/upload-file-chunk
Content-Type: multipart/form-data
```

Поля формы:

| Поле | Значение |
| :--- | :--- |
| `chunkedUploadId` | `uploadId` из шага 2 |
| `chunkIndex` | **с 1** |
| `totalChunks` | `ceil(size / 5 MiB)` |
| `file` | бинарный чанк (5 МиБ) |

Ответ должен содержать `data`; после последнего чанка сервер собирает файл сам.

### Прочие маршруты

| Метод | Маршрут | Назначение |
| :--- | :--- | :--- |
| `PUT` | `/_api/project-files/{projectId}/files/{fileId}/publish-pending` | опубликовать файл, ожидающий проверки |
| `DELETE` | `/_api/project-files/{projectId}/files/{fileId}` | удалить файл |

### Троттлинг

При массовой загрузке приходит `429 {"statusCode":429,"message":"ThrottlerException: Too Many Requests"}`. Нужен ретрай с backoff (10/20/30/40 с) и пауза между файлами 5–8 с.

### Дубликаты

CurseForge не принимает файл с тем же SHA-256. Если файл удалён в консоли, а хэш остался в истории проекта, помогает пересборка: `clean build` меняет таймстемпы внутри jar → новый хэш.

## 4. Автоматизация через Selenium + Chrome

### Почему нельзя просто взять cookies из обычного Chrome

- **Chrome 127+**: cookies шифруются App-Bound Encryption и привязаны к исходному `user-data-dir`. Копирование `Local State` + `Cookies` в другой профиль не расшифровывается.
- **Chrome 136+**: `--remote-debugging-port` **игнорируется** на стандартном профиле.

Следствие: нужен отдельный профиль и один ручной вход.

### Рабочая схема

1. Профиль: `%LOCALAPPDATA%\<Mod>\cf_real` (пустой).
2. Запуск Chrome:

```
chrome.exe --remote-debugging-port=9222 --user-data-dir=<cf_real> \
  --no-first-run --no-default-browser-check https://authors.curseforge.com/
```

3. Пользователь один раз входит (Google/Discord/GitHub/Twitch, 2FA).
4. Selenium подключается:

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
opts = Options()
opts.debugger_address = "127.0.0.1:9222"
driver = webdriver.Chrome(options=opts)
```

5. Все запросы выполняются **из страницы** (same-origin, cookies):

```javascript
fetch('/_api/project-files/123/upload-file-details', {
  method: 'POST', credentials: 'include',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify(metadata)
})
```

6. Файл передаётся в JS как base64 (jar < 1 МиБ → один чанк), затем собирается в `Uint8Array` и уходит в `FormData`.

### Cloudflare

`curl`, `requests`, `curl_cffi` получают JS-челлендж. Реальный Chrome проходит его сам; headless — нет. Поэтому браузер запускается видимым (можно за экраном: `--window-position=-32000,-32000`).

### Проверка статуса

После загрузки файл в консоли висит в `Processing` (готовится), затем становится `Approved`. Публичная страница проекта показывает его только после этого.

## 5. Чек-лист

- [ ] Сессия жива: `/_api/project-members/project-titles` → `200 application/json`.
- [ ] `get-game-versions` вернул непустой список.
- [ ] В `gameVersionIds` есть версия MC, загрузчик, `client` и Java.
- [ ] `chunkIndex` начинается с 1, `totalChunks` совпадает.
- [ ] При `429` — ретрай, а не пропуск файла.
- [ ] Файлы с одинаковым именем и хэшем не заливать повторно.
