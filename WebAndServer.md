# Постановка проблемы

Что мы хотим и можем?
- Отправить пользователю на его браузер (если он разрешит) и приложение в телефоне уведомление.

Очевидные вопросы которые у нас возникнут:
- Как сопоставлять пользователя и его устройства?
	- С пользователем все ясно - это user_id они уже собраны и хранятся в базе данных.
	- А вот с браузером и телефоном возникают некоторые вопросы. Далее будем называть их notification clients.
	- Нужно их как-то собирать их идентификаторы и сразу при сборе сопоставлять пользователям.
	- <img width="277" height="169" alt="image" src="https://github.com/user-attachments/assets/b47a130d-3f0c-455d-84ae-2cd6b51a59b8" />

- Как доставлять уведомления нашим notification clients?
	- Android. Сделаем консервативное допущение - у всех наших клиентов есть google play которые и рулят всеми уведомлениями на смартфонах. Это позволяет избежать проблем с расходом заряда и прочих вещей. Даже если ваше приложение выключено - уведомление придет. Уведомления приходить сюда перестанут только если вы зайдете в настройки приложения и полностью отключите его Force stop. Это вычеркнет его для google play.
	- Web. Тут все груснее вам все же нужен браузер висящий хотя бы фоном. Соединение держит каждый браузер самостоятельно используя свои службы например Mozilla Push Service.

Существует Firebase с его службой Firebase Cloud Messaging который как раз позволяет решить задачу связи с Android и различными браузерами, а кроме того решает ряд других сопуствующих проблем. Будем называть его push провайдером и именно к нему нам и надо обратится, чтобы передать уведомление.

Для отправки надо:
1. доказать что мы вообще это мы и наше уведомление надо бы принять в обработку
2. FCM token - именно он идентифицирует пользователя для firebase, как телефонный номер для уведомлений.
3. собственно само уведомление как ни странно требуется
4. Кроме того необходимо чтобы клиент был готов принимать уведомления.

# FcmService.php
Для использования
```
use Kreait\Firebase\Factory;
use Kreait\Firebase\Messaging\CloudMessage;
```
Необходимо 
``
composer require kreait/laravel-firebase
``

# Что нужно
- Firebase JS `npm install firebase`
- `composer show kreait/laravel-firebase`

# Web Client
Его задачи в этой системе:
- Решить все вопросы регистрации Service Worker, разрешения на уведомления в браузере

---

# Так что делать?
## Тупо скопировать:
- app/Services/FcmService.php
- app/Http/Controllers/API/NotificationController.php
- app/Models/NotificationClient.php
- database/migrations/2026_06_26_080845_create_notification_client_table.php
- resources/js/firebase.ts
- resources/js/fcm.ts
- public/firebase-messaging-sw.js
## Скопировать, но изменить данные:
- resources/js/firebase.ts
- public/firebase-messaging-sw.js
- .env
- storage/app/private/firebase-key.json
## Встроить:
`routes/api.php`

Добавить:
- код `POST /api/save-fcm-token` (возможно вам захочется вынести его в отдельный контроллер)
- `Route::post('/notify-users', [\App\Http\Controllers\API\NotificationController::class, 'notifyUsers']);` в блок с админом.

### `resources/js/Pages/Auth/LoginPage.vue`

Добавить после успешного login:
```
        if (isAndroid) {
            window.Android.setAuthToken(token);
        }
```
О том что это такое читать в Android.md
И вот это:
```
        if (!isAndroid) {
            void setupFcm();
        }
```
О котором поговорим сейчас.

При авторизации пользователя в web-приложении вызывается функция setupFcm(), которая вызывается 
и дергает setupWebPush()

<img width="563" height="366" alt="image" src="https://github.com/user-attachments/assets/a3fd17ea-7494-4056-8c1f-5319590ada9b" />
 
Получаем токен и пишем в таблицу БД notification_clients

WebPush в resources/js/fcm.ts организует всю подготовку браузера к пиему увдомлений и получает FCM токен. Именно тут подтягивается firebase-messaging-sw.js. Именно тут начинается использование firebase библиотеки которую необходимо установить через `npm install firebase` 

Кроме того для того чтобы в дальнейшем всем этим управлять в дальнейшем и вообще как главный интерфейс взаимодействия с нашей системой уведомлений основанной на firebase создается объект messaging методом из resources\js\firebase.ts, который также использует упомянутую библиотеку и также должен был быть изменен подобно firebase-messaging-sw.js. Все это дело ServiceWorker, massaging, кроме того тут мы используем публичный VAPID KEY. Мы отправляем в библиотечную фукцию firebase.

<img width="1063" height="201" alt="image" src="https://github.com/user-attachments/assets/32f97c43-4a0e-412d-90a5-194190663d5f" />

Эта библиотека уже связывается непосредственно с серверами firebase, которые по итогу через возврат этой функции нам и вернут заветный FCM токен.

Но что за странный объект мы тут создаем massaging? Относитесь к нему как к технической необходимости для получения токена, хотя он также может быть использован например для отдельной обработки уведомлений когда вкладка открыта.

В service worker вы можете это видеть
```
const messaging = firebase.messaging();

// --- Обработчик фоновых сообщений ---
// Вызывается, когда приложение не активно (закрыто или свёрнуто).
// Если в payload есть notification, FCM сам покажет уведомление.
// Здесь можно добавить кастомную логику (например, обработку data-поля).
messaging.onBackgroundMessage((payload) => {
    console.log('[SW] Фоновое сообщение:', payload);
    // Дополнительные действия описывать тут...
});
```


### `resources/js/Pages/AdminPage.vue` (если нужна админка)

Кнопка «Уведомить» + диалог + `sendNotification()` → `POST /api/admin/notify-users`.  
Это **~50 строк** из AdminPage, не весь файл.

### `nginx.conf` (опционально, но полезно)

Блок для SW без кэша:
```nginx
location = /firebase-messaging-sw.js {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Service-Worker-Allowed "/";
    try_files $uri =404;
}
```

### `docker-compose.yml` (если Docker + проблемы DNS)

```yaml
dns:
  - 8.8.8.8
  - 1.1.1.1
```
на сервисе `app`.



---
---
---
---
---

Разделю на: **целиком**, **с правками**, **кусочки в существующие файлы**, **не файлы**.

---

## 1. Копировать целиком (ядро)

| Файл | Зачем |
|------|--------|
| `app/Services/FcmService.php` | отправка push через Kreait |
| `app/Http/Controllers/API/NotificationController.php` | API «уведомить пользователей» |
| `app/Models/NotificationClient.php` | модель token в БД |
| `database/migrations/2026_06_26_080845_create_notification_client_table.php` | таблица `notification_clients` |
| `resources/js/firebase.ts` | init Firebase на клиенте |
| `resources/js/fcm.ts` | SW + permission + `getToken` |
| `public/firebase-messaging-sw.js` | фоновые push |

---

## 2. Копировать, но **обязательно поменять**

| Файл | Что поменять |
|------|----------------|
| `resources/js/firebase.ts` | `firebaseConfig` из **своего** Firebase Console |
| `public/firebase-messaging-sw.js` | **тот же** config + версия SDK (сейчас 12.15.0) |
| `.env` | `VITE_VAPID_PUBLIC_KEY=...` |
| `storage/app/private/firebase-key.json` | **свой** service account (не из git) |

Config в `firebase.ts` и SW **должен совпадать**.

---

## 3. Встроить в **существующие** файлы (не тащить файл целиком)

### `routes/api.php`

Добавить:
- `POST /api/save-fcm-token` (closure или отдельный контроллер)
- `POST /api/admin/notify-users` → `NotificationController@notifyUsers`

### `resources/js/Pages/Auth/LoginPage.vue`

Добавить после успешного login:
```javascript
import { setupWebPush } from '@/fcm';
// ...
void setupFcm();  // как у вас сейчас
```

### `resources/js/Pages/AdminPage.vue` не могу знать что там изменилось, поэтому вероятно придется встраивать:

Кнопка «Уведомить» + диалог + `sendNotification()` → `POST /api/admin/notify-users`.  
Это **~50 строк** из AdminPage, не весь файл.

### `nginx.conf` (опционально, но полезно)

Блок для SW без кэша:
```nginx
location = /firebase-messaging-sw.js {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Service-Worker-Allowed "/";
    try_files $uri =404;
}
```

### `docker-compose.yml` во время тестирования и отладки из-за этого возникали непредсказуемые проблемы, поэтому луче жестко прописать:

```yaml
dns:
  - 8.8.8.8
  - 1.1.1.1
```
на сервисе `app`.

---

## 4. Зависимости

**`composer.json`:**
```json
"kreait/laravel-firebase": "*"
```
→ `composer install`

**`package.json`:**
```json
"firebase": "^12.15.0"
```
→ `npm install` + `npm run build`

**`.env.example`:**
```
VITE_VAPID_PUBLIC_KEY=
```

---

## 5. Минимальный набор «если лень тащить всё»

Если нужен **только web push без админ-UI**:

```
✅ fcm.ts
✅ firebase.ts
✅ firebase-messaging-sw.js
✅ FcmService.php
✅ NotificationClient.php + migration
✅ routes: save-fcm-token + notify-users
✅ кусок LoginPage (setupFcm)
✅ firebase-key.json + .env VAPID
❌ AdminPage (можно слать через Postman/tinker)
```

---

## 6. Что **не** перетаскивать / не нужно

| Не нужно | Почему |
|----------|--------|
| `repomix-output.xml` | дамп, не код |
| Весь `LoginPage.vue` / `AdminPage.vue` | только логика FCM |
| `ssl/`, `mkcert` | инфра конкретного сервера |
| `node_modules/`, `vendor/` | ставятся через composer/npm |

---

## 7. Чеклист после копирования

1. Firebase Console → свой проект  
2. `composer install` + `npm install`  
3. `php artisan migrate`  
4. Подставить config / VAPID / `firebase-key.json`  
5. `npm run build`  
6. Войти в браузере → token в БД  
7. `POST notify-users` → push пришёл  

---

**Итог:** переносится **~7 файлов ядра** + **правки в 3–4 существующих** (routes, login, опционально admin, nginx) + **зависимости и секреты из Firebase Console**. Остальное проекта — не про уведомления.
