---
layout: post
title: Классный сервис для дебага серверной части - ngrok
categories: ngrok, utils
---

Нашел очень удачное решение для локального дебага серверных приложений.

`ngrok` - имеет множество полезных фич, но для меня это пока самый простой способ протестировать серверное приложение, когда нужен внешний доступ к нему.
ngrok - дает вам внешний URL и транслирует запросы с него на ваш localhost.

Все очень просто. Качаем файл `ngrok` с [офф сайта](https://ngrok.com/download) и кидаем его в любую директорию из `echo $PATH`.

Можем запускать. `ngrok http 8080` - порт, который вы хотите расшарить и получаете внешние ссылки по которым доступен ваш localhost.

```shell
Session Status                online                                                                             
Account                       Vladimir Romanichev (Plan: Free)                                                   
Version                       2.3.35                                                                             
Region                        United States (us)                                                                 
Web Interface                 http://127.0.0.1:4040                                                              
Forwarding                    http://44cf8e9d3b6e.ngrok.io -> http://localhost:8080                              
Forwarding                    https://44cf8e9d3b6e.ngrok.io -> http://localhost:8080   
                                                                    
```

Отдельно отмечу встроенный инспектор запросов по адресу http://127.0.0.1:4040/inspect/http

Задача возникает часто, когда нужно протестировать взаимодействие между сервисами. Пока это самое простое решение, которое мне попадалось. 

