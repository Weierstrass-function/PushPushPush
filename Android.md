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
А конкретно в onCreate
- `MyFirebaseMessagingService.loadToken(this)` Тут мы загружаем из долговременной памяти FCM токен (а то вдруг он у нас уже получен), ну вернее скрываем это действие за этим методом.
- Дальше техенические действия по 

Если опустить чисто технические детали функционал сведется к:
- Открывает https://10.240.222.1
- Даёт сайту мост в Android: метод Android.setAuthToken(token)

Если смотреть по ходу то по сути у вас дергается onCreate
