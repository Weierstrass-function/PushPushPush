# Действия по интеграции
## Backend:
- Тупо скопировать `app/Services/FcmService.php` Данный файл принимает массив FCM-токенов, тему и тело уведомления, собирает под формат принимаемый библеотекой Kreait и передает на сервера Firebase с использованием библиотеки при этом используя `app/private/firebase-key.json` именно данный файл ваш ключ от firebase как для отправителя уведомлений. Кроме того принимает ответ от серверов Firebase и возвращает в `NotificationController.php` различные списки токенов, с которыеми что-то не так.
- _Да в этом и есть вся прелесть использования Firebase он сам контролирует их жизненный цикл вам не надо ловить логауты (хотя в целом эта опция бывает полезна) и прочие нюансы жизненного цикла FCM токенов это делают сами сервисы Firebase и возвращают вам соответствующие ошибки"._
	- Для его работы надо установить данную библиотеку например так: `composer require kreait/laravel-firebase`
 	- Кроме того необходим файл `firebase-key.json`, его следует получить в https://console.firebase.google.com/ таким образом:
  		- <img width="730" height="325" alt="image" src="https://github.com/user-attachments/assets/3056bc8e-d08f-4c17-be66-8c2e1d105f22" />
		- прокрутите вниз
  		- <img width="356" height="181" alt="image" src="https://github.com/user-attachments/assets/620b083a-fac9-4381-beec-6bd10c92829a" />
		- Вы получите что-то такое:
		- <img width="466" height="160" alt="image" src="https://github.com/user-attachments/assets/ec7037f3-7787-497a-a747-a0904248439c" />
		- Остается преименовать в `firebase-key.json` и закинуть сюда: `storage\app\private\firebase-key.json`
  - Еще тут возникала ошибка связи с сверерами firebase, поэтому необходимо в docker compose в блок `app` добавить:
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
**Итог:** переносится **~7 файлов ядра** + **правки в 3–4 существующих** (routes, login, опционально admin, nginx) + **зависимости и секреты из Firebase Console**. Остальное проекта — не про уведомления.
