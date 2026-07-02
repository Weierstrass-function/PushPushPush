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

Необъодимо зайти в следующие файлы и изменить следующие строки:

TechStudio\app\src\main\java\com\techstudio\app\MainActivity.kt
`webView.loadUrl("ip.адрес.вашего.хоста")`

TechStudio\app\src\main\java\com\techstudio\app\MainActivity.kt
`val url = "https://ip.адрес.вашего.хоста/api/save-fcm-token"`

TechStudio\app\src\main\res\xml\network_security_config.xml
`<domain includeSubdomains="true">ip.адрес.вашего.хоста</domain>`

Далее необходимо собрать APK

<img width="542" height="288" alt="image" src="https://github.com/user-attachments/assets/64817b94-923a-449d-9be9-220fe47aa9cd" />

После чего можно приступать к тестированию уведомлений на android.



