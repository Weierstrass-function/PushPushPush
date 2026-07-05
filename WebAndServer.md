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
- `composer show kreait/laravel-firebase`

# Web Client
Его задачи в этой системе:
- Решить все вопросы регистрации Service Worker, разрешения на уведомления в браузере

---

# Так что делать?
## Backend:
- Тупо скопировать `app/Services/FcmService.php` Принимает массив FCM-токенов, тему и тело уведомления, собирает под формат принимаемый библеотекой Kreait и передает на сервера Firebase с использованием библиотеки при этом используя `app/private/firebase-key.json` именно данный файл ваш ключ от firebase как для отправителя уведомлений. Кроме того принимает ответ от серверов Firebase и возвращает в `NotificationController.php` различные списки токенов, с которыеми что-то.
- _Да в этом и есть вся прелесть использования Firebase он сам контролирует их жизненный цикл вам не надо ловить логауты (хотя в целом эта опция бывает полезна) и прочие нюансы жизненного цикла FCM токенов это делает сами сервисы Firebase и возвращают вам соответствующие ошибки"._
	- Для его работы надо установить данную библиотеку например так: `composer require kreait/laravel-firebase`
 	- файл `firebase-key.json` следует получить в https://console.firebase.google.com/
  		- <img width="730" height="325" alt="image" src="https://github.com/user-attachments/assets/3056bc8e-d08f-4c17-be66-8c2e1d105f22" />
		- прокрутите вниз
  		- <img width="356" height="181" alt="image" src="https://github.com/user-attachments/assets/620b083a-fac9-4381-beec-6bd10c92829a" />
		- Вы получите что-то такое:
		- <img width="466" height="160" alt="image" src="https://github.com/user-attachments/assets/ec7037f3-7787-497a-a747-a0904248439c" />
		- Остается преименовть в `firebase-key.json` и закинуть сюда: `storage\app\private\firebase-key.json`
  - Кроме того тут возникала ошибка свзи с сверерами firebase, поэтому необходимо в docker compose в блок `app` добавить:
  - ```
    dns:
            - 8.8.8.8
            - 1.1.1.1
    ```

- Продолжаем тупо копировать файлы `app/Http/Controllers/API/NotificationController.php` вызывается через через роуты принимает в качестве запроса массив id пользователей, заголовок уведомления и сам текст уведомления. Далее находит соответствующие данным пользователям записи в базе данных используя модель `NotificationClient`. Затем по данным из возврата из fcm->sendToTokens удаляет все неактуальные токены из базы. А еще я понял что не смогу простить если оставлю вам в api.php ту гадость, поэтому во время написания я еще раз отрефакторил код и теперь saveFcmToken который принимает собственно сам FCM токен и название платформы, которые помещает в базу данных, кроме того тут обрабатывается данная возможная ошибка, если вы ее еще помните:
	- <img width="660" height="306" alt="image" src="https://github.com/user-attachments/assets/892f826c-adba-46ba-9cb0-0b7546cec914" />

- Далее копируем `app/Models/NotificationClient.php`, который содержит описание упомянутой модели связывая их с реальной таблицей `notification_clients` механизмами Illuminate, штатными для предоставленного проекта.
- Копируем `database/migrations/2026_06_26_080845_create_notification_client_table.php`, который строит упомянутую таблицу. Вроде тут тоже нет ничего особо интересного для данного разбора.

## API:
- Внезапно НЕ копируем, а открываем `routes/api.php`, нужно иметь возможность по роутам дергать функции добавленного контроллера, необходимо добавить:
	- `Route::post('/save-fcm-token', [NotificationController::class, 'saveFcmToken']);` в блоке `Route::middleware('auth:sanctum')` чтобы убедится, что пользователь отправляющий нам токен все же прошел аутентификацию.
 	- `Route::post('/notify-users', [NotificationController::class, 'notifyUsers']);` в блок `Route::prefix('admin')`

## Frontend:
- В файл `resources/js/Pages/Auth/LoginPage.vue`
	- Добавить после успешного login:
 	-  ```
        if (isAndroid) {
            window.Android.setAuthToken(token);
        }
     	```
	- О том что это такое читать в Android.md
	- И еще добавить вот это:
 	- ```
        if (!isAndroid) {
            void setupFcm();
        }
    	```
 	- Кроме того добавить функцию
  	- ```
		const setupFcm = async () => {
		    const label = isEdge() ? 'FCM (Edge)' : 'FCM';
		
		    try {
		        const fcmToken = await setupWebPush();
		        if (!fcmToken) {
		            return;
		        }
		
		        await axios.post('/api/save-fcm-token', {
		            token: fcmToken,
		            platform: 'web',
		        });
		        console.log(`${label}: токен сохранён на сервере`);
		    } catch (error) {
		        console.error(`${label}:`, error);
		    }
		};
		```
	- Данный код вызывает setupWebPush() из fcm.ts, который возвращает fcm токен и отправляет запрос по уже известному маршруту /api/save-fcm-token для сохранения токена на бэкэнд
- Файл `resources\js\fcm.ts` необходимо скопировать. Он реализует полную подготовку клиента к получению уведомления финальным шагом которой является получение fcm токена функцией `getToken` из библиотеки firebase
	- Как вы понимаете для этого ему требуется библиотека, которую необходимо установить через `npm install firebase`
 	- getToken в качестве одного из параметров принимает публичный ключ vapid, который необходимо получить через на https://console.firebase.google.com таким образом:
  		- <img width="254" height="239" alt="image" src="https://github.com/user-attachments/assets/1d8439d9-71b5-42e6-afb0-8c3efb2ca6a4" />
		- Листаем вниз
		- <img width="449" height="346" alt="image" src="https://github.com/user-attachments/assets/18611456-4ad0-4e29-9e50-f14756d47b9a" />
		- После нажатия вы увидите публичный VAPID ключ который необходимо скопировать в .env
  		- И тут вы спросите, где брать приватный. Но на самом деле больше такой необходимости нет. Раньше когда проект использовал радикально разные подходы к web и к android я его использовал, но затем решил все централизовать, что значительно упростило код и хранение notification_clients раньше там вообще хранились не только fcm токены но и разные объекты все напоминание что об этом осталось - VAPID PRIVATE key и поле platform в базе данных. Кроме простоты в тестах FCM показал себя значительно надежнее.
    	- Другим не менее важным параметром `getToken` является объект который порождается регистрацией service worker.
     		- Для этого для начала скопируйте файл `public\firebase-messaging-sw.js`
       		- Затем получите на https://console.firebase.google.com новый конфиг таким образом:
         		- <img width="682" height="200" alt="image" src="https://github.com/user-attachments/assets/9e0803ce-ed17-4ad1-8120-2c230e365a32" />
				- Пролистайте вниз и выберите
         		- <img width="193" height="163" alt="image" src="https://github.com/user-attachments/assets/e45c0b18-05df-4de8-a0d4-d8154f1c918f" />
				- Введите какое-нибудь название
				- <img width="579" height="547" alt="image" src="https://github.com/user-attachments/assets/ebe7488b-fce4-44ab-9a29-cc5b9e36da40" />
				- Скопируйте
         		- <img width="689" height="441" alt="image" src="https://github.com/user-attachments/assets/2b029f4f-21ca-4db5-a2e0-d2bcf2407627" />
				- вставьте в файл `public\firebase-messaging-sw.js` аккуратно именно внутрь блока firebase.initializeApp
         		- Есть еще одна проблема. Браузеры очень щепитильны к регистрациям service worker. Поэтому нам необходим https решение этой проблемы в рамках разаработки и локальной сети описано в видеоинструкциях, а решением на продакшине должно стать полноценное получение доменного имени (можете даже взять какой-нибудь бесплатный домен уровня третьего, но чтобы на него можно было выдать сертификат, который будут готов подтверждить центр сертификации)
         		- На самом деле этот файл задает откуда брать код для реального service workera любезно предоставленного нам Firebase как вы момжете видеть внутри он прямо сам подключается к серверам и вытягивает код. Что касается нас в этом файле можно например добавить кастомный код обработки push.
		- но вернемся к `getToken` и еще одному его параметру massaging. Для него необходим файл `resources\js\firebase.ts`
   			- скопируйте данный файл
      		- возьмите конфиг, который вы использовали для service-worker и поместите его еще и в данный файл
        - `messenging` и все что с ним связано чисто технический код для нормальной для того чтобы самой библиотеке понять что вот тут мы работаем с таким-то конкретным конфигом.





		

О котором поговорим сейчас.

- При авторизации пользователя в web-приложении вызывается функция setupFcm(), которая вызывается 
и дергает setupWebPush() из файла `resources\js\fcm.ts` который надо также тупо скопировать.

<

Данный файл

---
---
---

- `resources/js/firebase.ts`
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

Важно!!! Не путайте с messaging, который
```
class FcmService
{
    protected $messaging;
```
в app\Services\FcmService.php


### `resources/js/Pages/AdminPage.vue`

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
