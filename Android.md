# Android

## Какую задачу решает созданное приложение?
Push уведомления в web не обладают достаточной надежностью. По факту вам постоянно нужен работающий браузер или в случае Chrome или фоновые его процессы в случае Firefox. Или висящий фоновый процесс вашего десктопного приложения (которого у вас нет), но если бы было, то ему тоже нужно было бы висеть в фоне как Telegram Однако на Android все интереснее и на мой взгляд значительно удобнее, ведь, по понятным причинам, жесткой экономии заряда висеть куче Android приложений невозможно. Поэтому google в свое время подсадил нас на свои сервисы и вместо кучи разношерстных браузеров и иных приложений на Android висит только соответствующая служба google play.

>По факту все мобильное приложение было реализовано просто как прослойка через WebView-wrapper между сайтом и системой Android.

## AndroidManifest.xml
Большая часть этого файла шаблон полученный Android studio. Что касается интересного нам:
- `<uses-permission android:name="android.permission.INTERNET" />`
Чтобы WebView мог открывать сайт и приложение могло слать HTTP-запросы на сервер (FCM-токен).
- `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />`
С Android 13 без этого разрешения push-уведомления пользователю не покажутся.

- `android:networkSecurityConfig="@xml/network_security_config"`
Отдельные правила SSL для 10.240.222.1: доверять не только системным CA, но и пользовательским (сертификат, установленный вручную на телефон).

- `android:windowSoftInputMode="adjustResize"`
Когда на сайте открывается клавиатура (логин, формы), экран сжимается, а не перекрывается. Поля ввода остаются видимыми.

- Сервис Firebase: приложение получает push-токен устройства и может реагировать на обновление токена.
  ```
  <service
      android:name=".MyFirebaseMessagingService"
      android:exported="false">
      <intent-filter>
          <action android:name="com.google.firebase.MESSAGING_EVENT" />
      </intent-filter>
  </service>
  ```

Точка фхода 
```
   <activity
            android:name=".MainActivity"
            android:exported="true"
            android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```
Т.е. преходим в
## MainActivity

Проблема: чтобы отправить FCM токен на сервер надо подтвердить, что пользователь вошел в аккаунт. На web все это скрывают от нас различные механизмы Loravel. Тут же нам придется в ручном режиме отправлять токен авторизации (далее auth token или bearer токен).

А конкретно в onCreate нам наиболее интересны:
- `MyFirebaseMessagingService.loadToken(this)` Тут мы загружаем из долговременной памяти FCM токен (а то вдруг он у нас уже получен), ну вернее скрываем это действие за этим методом.
- `webView.settings.javaScriptEnabled = true` для setAuthToken (стр. 77)
- `webView.addJavascriptInterface(this, "Android")` чтобы сам сайт мог дернуть setAuthToken так называемый JS-мост, собственно дает на web в файле LoginPage.vue вызвать ее `window.Android.setAuthToken(token);` (стр. 93)
- собственно после этого вызова мы попадем в
   ```
   @JavascriptInterface
    fun setAuthToken(token: String) {
        authToken = token
        android.util.Log.d("AUTH", "Received token from WebView: $token")

        // Как только мы получили авторизацию, отправляем FCM-токен на сервер,
        // чтобы привязать устройство к пользователю.
        MyFirebaseMessagingService.sendTokenToServer()
    }
  ```
В котором мы записываем Berarer token в переменную authToken
И главное вызываем MyFirebaseMessagingService.sendTokenToServer()

## MyFirebaseMessagingService.kt
Содержит
- Методы для работы с долговременной памятью через SharedPreferences для хранения fcm токена
- sendTokenToserver единственная задача всего большого и страшного кода в котором просто собрать и отправить корректный http запрос и отправить его, кроме того тут можно реализовать обработку ошибок.

Но собственно откуда же берется FCM-токен?

В самом низу вы найдете маленькую переопределенную функцию из firebase
```
    override fun onNewToken(token: String) {
        Log.d("FCM", "New FCM token: $token")

        // Сохраняем токен в долговременную память
        saveToken(applicationContext, token)

        // Отправляем на сервер (асинхронно)
        sendTokenToServer()
    }
```

Данная функция премечательна тем, что он вызывается сам когда приложение получает новый FCM-токен от плагина, например при первом запуске. Нам остается просто его сохранить в долговременной памяти и передать на сервер.

А паьриаеи это за счет вот этого (возвращаемся в мнифест):
```
        <service
            android:name=".MyFirebaseMessagingService"
            android:exported="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>
```
Это договор с Android:

Что-то типа: «Если придёт событие com.google.firebase.MESSAGING_EVENT — запусти MyFirebaseMessagingService».

Firebase/Google Play Services как раз шлёт такие intent'ы, когда:
- пришёл новый FCM-токен → вызовется onNewToken()
- пришло push-сообщение → вызвался бы onMessageReceived() (да если вы хотите обрабатывать эти уведомления внутри приложения стоит отметить что и этот метод вы можете переопределить)

Важно сказать что этот плагин должен быть как бы подключен при сборке, это происходит в TechStudio\build.gradle.kts кроме того он нуждается в файле TechStudio\app\google-services.json

Чтобы получить google-services.json
В созданном проекте в console.firebase.google.com

<img width="729" height="289" alt="image" src="https://github.com/user-attachments/assets/a2f366d2-2192-4cd6-acf3-d44328140cbf" />

Пролистайте вниз и нажмите

<img width="166" height="144" alt="image" src="https://github.com/user-attachments/assets/6fcb67c2-30f3-4819-abb1-d2950810df9c" />

Введите ваши данные

<img width="527" height="809" alt="image" src="https://github.com/user-attachments/assets/24070dd7-c79e-44e4-8536-fa06ab03b081" />

Теперь необходимый файл будет доступен для скачивания

<img width="441" height="185" alt="image" src="https://github.com/user-attachments/assets/f3df21c6-9df4-430d-8e10-875ea0ccf96d" />

---

По факту если:
- Вас устраивает такой вариант реализации
- Ваш web корректно отображается в браузерах на телефонах
Вам достаточно просто получить собственный файл google-service.json и заменить его в проекте.
