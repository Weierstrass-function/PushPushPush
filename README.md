# PushPushPush



## Запуск докер-контейнеров

>Данная инструкция позволит вам вывести приложение в локальную сеть и протестировать отправку уведомлений на разные физические устройства в пределах вашей локальной сети.

Для начала открыть директорию tech_studio
```
docker compose up --build -d
```
дождаться поднятия 4-х контейнеров

<img width="482" height="175" alt="image" src="https://github.com/user-attachments/assets/587e60ee-18a2-47bc-98c3-03ccb7f698fa" />

Создать таблицы и загрузить начальные данные базы данных.
```
docker compose exec app php artisan migrate --seed
```
(Этот вариант корректен и даже рекомендуем, поскольку мы можем обращатся не по имени контейнера а непосредственно к экземпляру этого контейнера сервису app)

<img width="1077" height="568" alt="image" src="https://github.com/user-attachments/assets/6f0bab9a-17f7-4059-8566-e735f13bbc1c" />

## Настройка IP-адреса для web

Дальше надо подготовить наш импровизированный сервер (в виде вашего компьютера на котором запущен докер с приложением (далее просто "ваш локальный сервер")) к работе и так сказать обслуживанию клиентов в пределах вашей локальной сети
>В продакшине вы просто поменяете ip.вашего.локального.сервера на полученный домен (вам все равно придется его получить, поскольку именно для доменов удостоверяющие центры готовы выдавать сертификаты, причем домены какого-нибудь 3го уровня вполне лего можно получить бесплатно)

Соберите все устройства, на которых вы собираетесь тестировать работу системы внутри одной локальной сети (подключите его к одному роутеру / телефону в режиме точки доступа)

Если вы под windows выполните команду
```
ipconfig
```

- Если ваш сетевой адаптер подключен через Ethernet вот ip вашего локального сервера

<img width="630" height="601" alt="Pasted image 20260701211803" src="https://github.com/user-attachments/assets/6de2cf8b-757c-4a4e-893a-c8b4bc978148" />

- Если WiFi

<img width="721" height="556" alt="Pasted image 20260701211935" src="https://github.com/user-attachments/assets/50192680-d5cb-4a32-bdc0-fc093f3cb412" />

Теперь у вас есть ip.вашего.локального.сервера

Перейдите в .env и вставьте его сюда

<img width="451" height="65" alt="image" src="https://github.com/user-attachments/assets/33b433d5-cb7d-4546-a042-46ab2e08bb8d" />

После чего выполните
```
docker compose exec app php artisan config:clear
```

## Создание SSL сертификата

Дальше есть одна маленькая скромная проблема браузеры очень не любят service workers от мутных ресурсов с неправильными сертификатами, поэтому если хотите протестировать на нескольких устройствах в своей локальной сети необходимо создать сертификат для ip вашего хоста на машину, где вы запускаете докер нужно поставить маленькую утилитку. Если Windows
```
winget install mkcert
```

<img width="1063" height="375" alt="image" src="https://github.com/user-attachments/assets/924d4e9c-43d1-4637-a743-fccb7f688206" />

Перезапустить терминал

<img width="1063" height="174" alt="image" src="https://github.com/user-attachments/assets/fba28e04-843d-4d47-b5bf-127d46e3de91" />

Установить локальный сертификат CA в доверенное хранилище вашей системы
для этого выполнить команду 
```
mkcert -install
```
Перейти в папку tech_studio\ssl

Создать сертификат для ip вашего локального сервера
```
mkcert ip.адрес.вашего.хоста
```
После переименуйте получившиеся файлы 
ip.адрес.вашего.хоста.pem -> selfsigned.crt
ip.адрес.вашего.хоста-key.pem -> selfsigned.key

Далее не забудьте выполнить
```
docker compose restart
```

ЕСЛИ НА ТЕКУЗИЙ МОМЕНТ ВСЕ СДЕЛАНО ПРАВИЛЬНО ВЫ УВИДИТЕ ПРИ ВБИВАНИИ В ПОИСКОВОЙ СТРОКЕ ip вашего локального сервера на нем же желанный замочек

<img width="232" height="95" alt="image" src="https://github.com/user-attachments/assets/9f36dd40-b693-49b2-b387-67929c830751" />

<img width="203" height="100" alt="image" src="https://github.com/user-attachments/assets/1481e060-7cb5-4940-80cb-e83fa60ee788" />

## Установка на Android 

Переместите файл C:\Users\ВАШЕ_ИМЯ_ПОЛЬЗОВАТЕЛЯ\AppData\Local\mkcert\rootCA.pem на ваш телефон.

Далее в настройках:

<img width="591" height="1280" alt="image" src="https://github.com/user-attachments/assets/d1b07b5b-c7be-47a0-8f0b-fef75caf8640" />

<img width="591" height="1280" alt="image" src="https://github.com/user-attachments/assets/d82a8326-1170-448d-9fed-344f24be6b74" />

<img width="591" height="1280" alt="image" src="https://github.com/user-attachments/assets/0fa70124-2a60-4ebf-8f5a-431bdb311cac" />

<img width="591" height="1280" alt="image" src="https://github.com/user-attachments/assets/aa7ee7e4-c614-4750-8217-9b28b911f5e6" />

Если все верно вы увидите надпись

<img width="591" height="1280" alt="image" src="https://github.com/user-attachments/assets/21f406d4-db29-4730-a6eb-4cd4a987a57e" />

## Настройка ip для приложения Android

Необходимо зайти в следующие файлы и изменить следующие строки:
- TechStudio\app\src\main\java\com\techstudio\app\MainActivity.kt
`webView.loadUrl("ip.адрес.вашего.хоста")`
- TechStudio\app\src\main\java\com\techstudio\app\MyFirebaseMessagingService.kt
`val url = "https://ip.адрес.вашего.хоста/api/save-fcm-token"`
- TechStudio\app\src\main\res\xml\network_security_config.xml
`<domain includeSubdomains="true">ip.адрес.вашего.хоста</domain>`

Далее необходимо собрать APK

<img width="542" height="288" alt="image" src="https://github.com/user-attachments/assets/64817b94-923a-449d-9be9-220fe47aa9cd" />

После чего можно приступать к тестированию уведомлений на android.

## Как это работает

Шаг 1: База данных. В архитектуре TechStudio это фундамент всей системы уведомлений, который отвечает за то, чтобы сервер всегда знал, «куда именно стучаться», когда нужно доставить сообщение конкретному человеку.

У каждого пользователя может быть несколько активных сессий (например, мобильное приложение на Android и открытая вкладка в браузере на ПК). Если мы просто привяжем уведомления к `user_id`, мы не сможем адресно отправлять пуши на конкретные физические устройства. Таблица `notification_clients` решает эту проблему, выступая реестром соответствия «Человек — Устройство».

---

### Техническая реализация

#### 1. Структура таблицы (Миграция)

Файл: `database/migrations/2026_06_26_080845_create_notification_client_table.php`

```php
Schema::create('notification_clients', function (Blueprint $table) {
    $table->id(); // Первичный ключ записи
    
    // Внешний ключ, привязывающий устройство к пользователю.
    // onDelete('cascade') — критически важная настройка: если пользователь 
    // удаляет аккаунт, его токены стираются автоматически[cite: 1].
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    
    // Тип устройства (web, android и т.д.) для фильтрации[cite: 1].
    $table->string('platform'); 
    
    // FCM-токен — это и есть физический адрес устройства.
    // Индекс ->index() превращает поиск токена в SQL-запросе из 
    // полного перебора таблицы в мгновенную операцию[cite: 1].
    $table->string('fcm_token')->index(); 
    
    $table->timestamps();
});

```

#### 2. Модель (ORM-слой)

Файл: `app/Models/NotificationClient.php`

Эта модель — ваш главный инструмент взаимодействия с данными в коде Laravel.

* **Массовое заполнение:** Свойство `$fillable` — это мера безопасности. Оно разрешает записывать в базу только те поля (`user_id`, `platform`, `fcm_token`), которые мы явно указали, защищая проект от уязвимости *Mass Assignment* (когда злоумышленник пытается отправить скрытые поля через форму).


```php
protected $fillable = [
    'user_id',
    'platform',
    'fcm_token',
];

```


* **Связи:** Метод `user()` позволяет мгновенно получать объект пользователя для конкретного устройства, используя Eloquent-связи:


```php
public function user()
{
    return $this->belongsTo(User::class);
}

```

---

Разберем **Шаг 2: Как «выманить» адрес у браузера (Web Push API)**. Это самая деликатная часть: браузеры по умолчанию настроены на «защиту от спама» и не позволят сайту просто так слать уведомления без явного согласия пользователя и сложной цепочки рукопожатий.

### Суть задачи

Нам нужно пройти три этапа:

1. Инициализировать Firebase в браузере.
2. Получить разрешение у пользователя и сгенерировать токен.
3. Заставить скрипт работать даже тогда, когда вкладка сайта закрыта (Service Worker).

---

### Техническая реализация

#### 1. «Паспорт» проекта (Инициализация)

Файл: `resources/js/firebase.ts`

Чтобы браузер «поверил», что он общается с нашими серверами Google, мы используем уникальные ключи конфигурации.

```typescript
import { initializeApp } from "firebase/app";
import { getMessaging } from "firebase/messaging";

const firebaseConfig = {
    apiKey: "AIzaSyBsFe53maRY3SPSD-jwHsEztF8my3MVb8A", // Ваш уникальный ключ
    projectId: "techstudio-4ff05",
    // ... прочие параметры проекта
};

const app = initializeApp(firebaseConfig);
export const messaging = getMessaging(app); // Экспортируем канал связи

```

#### 2. Сбор адреса (LoginPage.vue)

Это критический момент. Как только пользователь вошел в систему, мы запускаем процесс регистрации.

Файл: `resources/js/Pages/Auth/LoginPage.vue`

```javascript
// 1. Сначала регистрируем Service Worker (см. ниже)
const registration = await registerServiceWorker(); 

// 2. Получаем токен у Firebase
// VAPID-ключ — это наша «цифровая подпись», подтверждающая право на отправку.
const fcmToken = await getToken(messaging, {
    vapidKey: import.meta.env.VITE_VAPID_PUBLIC_KEY, 
    serviceWorkerRegistration: registration
});

// 3. Отправляем на бэкенд, чтобы сохранить в "Адресную книгу"
await axios.post('/api/save-fcm-token', {
    token: fcmToken,
    platform: 'web',
});

```

#### 3. Тайный агент в браузере (Service Worker)

Файл: `public/firebase-messaging-sw.js`

Обычный JavaScript на странице «умирает» вместе с закрытием вкладки. **Service Worker** — это отдельный мини-процесс в браузере, который живет своей жизнью.

* **Мгновенная активация:**
```javascript
self.addEventListener('install', () => self.skipWaiting());
self.addEventListener('activate', () => self.clients.claim());

```


Эти команды говорят браузеру: «Не жди перезагрузки, активируй меня прямо сейчас!».
* **Фоновое прослушивание:**
Когда приходит пуш, а сайт закрыт, именно этот скрипт принимает «письмо» от Google:
```javascript
messaging.onBackgroundMessage((payload) => {
    console.log('[SW] Фоновое сообщение:', payload);
    // Firebase сам покажет уведомление, если в данных есть блок 'notification'
});

```

---

**Шаг 3: Как это устроено на Android (Бережем батарейку и связываем миры)**.

В отличие от браузера, где код живет в одной вкладке, Android-приложение должно быть «энергоэффективным». Если бы приложение постоянно держало открытым соединение с сервером, телефон бы разрядился за пару часов. Поэтому мы используем нативные сервисы Google, которые работают как единый «почтовый центр» для всей системы.

### Суть задачи

1. **Нативный перехват:** Получить от Google `FCM-токен` устройства на уровне ОС.
2. **Мост (Bridge):** Передать данные об авторизации из WebView (нашего сайта) в нативный код Android.
3. **Регистрация:** «Поженить» токен устройства и токен пользователя на бэкенде.

---

### Техническая реализация

#### 1. «Вшитые» настройки (`build.gradle.kts` и `google-services.json`)

При сборке Android-приложения плагин `com.google.gms.google-services` считывает ваш `google-services.json`. Он «намертво» вшивает параметры Firebase (API-ключи, ID проекта) в архитектуру приложения, чтобы Google понимал: «Это запрос именно от приложения TechStudio».

#### 2. Нативная служба (`MyFirebaseMessagingService.kt`)

Этот класс работает в фоне. Главная его задача — слушать события от Firebase.

```kotlin
// Файл: MyFirebaseMessagingService.kt (стр. 58-63)
override fun onNewToken(token: String) {
    Log.d("FCM", "New FCM token: $token")
    
    // 1. Сохраняем в память телефона, чтобы не потерять
    saveToken(applicationContext, token) 
    
    // 2. Сразу отправляем на ваш бэкенд
    sendTokenToServer(token)              
}

```

*Зачем это нужно:* Google может обновить токен в любой момент (например, при обновлении Google Play Services). Метод `onNewToken` — это «предохранитель», который гарантирует, что у вашего сервера всегда будет актуальный адрес этого смартфона.

#### 3. Мост авторизации (`MainActivity.kt`)

Поскольку внутри приложения у нас `WebView` (мини-браузер с сайтом), возникает проблема: как сайту сказать приложению «Я авторизован, теперь ты тоже можешь слать пуши»?

Мы создали «мост» через JavaScript-интерфейс:

```kotlin
// Файл: MainActivity.kt (стр. 53-60)
@JavascriptInterface
fun setAuthToken(token: String) {
    authToken = token
    // Прямой вызов нативного метода, который сделает запрос к Laravel
    MyFirebaseMessagingService.sendTokenToServer(token) 
}

```

**Как это работает «на пальцах»:**

1. Пользователь заходит на сайт внутри приложения.
2. Сайт отправляет логин/пароль на бэкенд.
3. Бэкенд возвращает `Bearer-токен` авторизации.
4. Сайт вызывает `window.Android.setAuthToken(token)`.
5. `MainActivity` перехватывает этот токен, сохраняет его в переменную `authToken` и инициирует отправку FCM-токена на ваш сервер.

---

Разберем **Шаг 4: Наведение порядка на сервере и «Захват токена»**.

Этот этап — «фильтр безопасности» и «администратор адресной книги». Когда токен устройства (будь то Android или браузер) прилетает на бэкенд, нам нужно не просто сохранить его в базу, а убедиться, что мы случайно не «подсунули» чужое устройство другому пользователю.

### Проблема: «Эффект общего планшета»

Представь, что у вас в компании есть один рабочий Android-планшет.

1. Сначала под своим логином заходит **сотрудник А** — сервер привязывает планшет к нему.
2. Потом сотрудник А выходит, и заходит **сотрудник Б**.
3. Если мы просто добавим запись, в базе будет путаница. А если мы «забудем» старую связь, то пуши сотрудника А будут приходить на планшет сотрудника Б.

Для этого мы используем механизм **перехвата (захвата)**.

---

### Техническая реализация (API)

Файл: `routes/api.php`

Маршрут `/api/save-fcm-token` принимает токен и платформу, а затем выполняет проверку:

```php
// 1. Поиск: ищем запись с этим токеном для конкретной платформы
$existing = NotificationClient::where('platform', $platform)
    ->where('fcm_token', $token)
    ->first();

if ($existing) {
    // 2. Если токен уже наш, и пользователь тот же — ничего делать не надо
    if ($existing->user_id == $user->id) {
        return ['status' => 'exists_same_user'];
    }

    // 3. МЕХАНИЗМ ЗАХВАТА: 
    // Устройство сменило владельца. Перепривязываем токен новому юзеру.
    $existing->user_id = $user->id;
    $existing->save();

    return ['status' => 'reassigned'];
}

// 4. Если токена в базе нет — создаем новую связь
NotificationClient::create([
    'user_id'   => $user->id,
    'platform'  => $platform,
    'fcm_token' => $token,
]);

```

---

Разберем **Шаг 5: «Кнопка админа» (Механика массовой рассылки)**. Это кульминация всей системы, где данные из «адресной книги» превращаются в реальные уведомления на устройствах ваших сотрудников.

### Суть задачи

1. **Интерфейс:** Админ выбирает пользователей и пишет текст.
2. **Контроллер:** Собирает физические адреса (токены) всех выбранных пользователей.
3. **Сервис:** Отправляет пакетный запрос в Google (Firebase), чтобы не перегружать сервер.

---

### Техническая реализация

#### 1. Интерфейс (AdminPage.vue)

Админ взаимодействует с компонентом выбора, который формирует массив ID. При нажатии кнопки «Отправить» вызывается метод `sendNotification`:

```javascript
// AdminPage.vue (логика отправки)
const sendNotification = async () => {
    notifyLoading.value = true;
    try {
        // Собираем массив ID (например, [1, 5, 12])
        const ids = selectedData.value.map(u => u.id); 
        await axios.post('/api/admin/notify-users', {
            user_ids: ids,
            title: notifyTitle.value,
            message: notifyMessage.value
        });
    } finally {
        notifyLoading.value = false;
    }
}

```

#### 2. Сбор данных (NotificationController.php)

Контроллер выступает «фильтром». Его задача — превратить список ID пользователей в чистый список FCM-токенов.

```php
// NotificationController.php (метод notifyUsers)
$tokens = NotificationClient::whereIn('user_id', $request->user_ids)
    ->pluck('fcm_token') // Достаем только токены
    ->filter()           // Убираем пустые/null значения
    ->unique()           // Важно: один токен = одна доставка
    ->values()
    ->toArray();         // Превращаем в простой массив ['token1', 'token2', ...]

```

#### 3. Главпочтамт (FcmService.php)

Мы используем библиотеку `Kreait\Firebase`. Это профессиональный стандарт для работы с Firebase в PHP.

```php
// FcmService.php
public function sendToTokens(array $tokens, string $title, string $body)
{
    // 1. Создаем объект уведомления
    $notification = Notification::create($title, $body);

    // 2. Упаковываем данные (title/body дублируются в data, 
    // чтобы Android-приложение могло их прочитать даже в фоне)
    $message = CloudMessage::new()
        ->withNotification($notification)
        ->withData(['title' => $title, 'body' => $body]);

    // 3. Пакетная отправка (Multicast)
    return $this->messaging->sendMulticast($message, $tokens);
}

```

---
