---
layout: post
title: Такой неочевидный мемкеш
categories: Memcache Memcached Why
---

Недавно админы проводили плановые работы на сервере, ничего не предвещало беды, но мы получили неожиданный простой сервиса. 
Если у вас тоже несколько нод Мемкеш для сессий, и вы еще спите спокойно, то не лишним будет проверить свои конфиги.

Админы имея вот такой конфиг с 3мя серверами хотели по очереди остановливать сервера Мемкеш, проводить работы и вводить в строй. 

```
[session]
session.save_handler  = memcached
session.save_path = "memcached1:11211,memcached2:11211,memcached3:11211"

```

Гладко было на бумаге, да забыли про овраги. При выключении одной ноды с мемкешем сервис для пользователей полностью **стал колом**. Интересно как так произошло и что делать? 

По умолчанию в php вот так. Сессия Пользователя - это файл доступной для пхп директории. Название файла нам понятно из куки, которую мы отдаем браузеру и браузер нам её приносит при каждом запросе. Берем куку, берем по ней файл - вот нам сессия пользователя. Читаем и пишем в неё. 


```
[session]
session.save_handler  = files
```

Когда у нас 1 php бекенд - проблемы нет, но что если нам нужно несколько бекендов? На каждом бекенде будут свои сессии. Нам нужно единое хранилище. Для этого нам доступны различные хранилища и php модули для работы с ними. Можно даже свое извращение [придумать](https://www.php.net/manual/ru/function.session-set-save-handler.php). Мы используем мемкеш в качестве хранилища, старый и проверенный метод. 


То что старое и проверенное, не всегда очевидное и понятное. В php нам доступно 2 модуля для работы с мемкеш Memcache и Memcached! Настройки от Memcache не подходят к Memcached и наоборот. Используя `session.save_handler  = memcached` настройки нужно искать именно `memcached.*`. 2 модуля для одного мемкеш маловато, хотелось бы хотя бы 3. Не перепутайте!!!

Считается, что Memcache модуль устаревший и рекомендуется использовать именно Memcached. Мы так и делаем, при этом Memcache гораздо лучше документирован на php.net и к нему гуглится больше рецептов. 

Подключая несколько серверов с memcache `session.save_path = "memcached1:11211,memcached2:11211,memcached3:11211"`, мы не получаем более надежный сервис. Мемкеш распределяет сессии между серверами и для конкретного пользователя всегда ходит на конкретный сервер. Уронив этот сервер пользователи сервиса получат ошибку 500 пока сервер не поднимется. По идее, мемкеш должен распределять их равномерно, ведь сервера не имеют веса в конфиге. На 3 сервера я создавал 3 подключения и все они всегда упали в один мемкеш, возможно распределение начинается на больших числах пользователей. Такой конфиг точно не гарантирует, что только 3 ваших пользователей будут страдать от аварии, скорее всего доля страдальцев будет больше трети. 

Имея 3 ноды с мемкешем, нас такое поведение не устраивает. Мы хотим полную доступность. Для Memcache модуля я достаточно быстро нашел нужный конфиг, практически не выходя с php.net 

```
[memcache]

memcache.allow_failover = 1
memcache.max_failover_attempts = 1
memcache.hash_strategy = "consistent"
memcache.session_redundancy = 4 # количество нод + 1, у нас 3 ноды, ставлю в конфиг 4. 
```

При таком конфиге мы можем ронять сервера мемкеша по очереди в любом порядке и сервис будет работать. Приложение будет бросать Warning, но если мы их логируем, а не показываем пользователю, то пользователи не заметят проблемы.
Когда сервер на который попала сессия пользователя поднимется, и если на нем произошла очистка памяти, к сожалению, сессия будет пустой и наш **сервис заставит пользователя пройти авторизацию снова**. Это плохо, но лучше чем полная недоступность в момент падения.

А вот Memcached я просидел пару вечеров. Рецептов в интернете не нашел. Самая полная документация по конфигу только в github самого модуля [php-mecached](https://github.com/php-memcached-dev/php-memcached/blob/master/memcached.ini) нашлась. 

В итоге, только благодаря черной магии, подобрал такой конфиг. 

```
; Allow failed memcached server to automatically be removed.
; Default is Off. (In previous versions, this setting was called memcached.sess_remove_failed)
memcached.sess_remove_failed_servers = On

; Set this value to enable the server be removed after
; configured number of continuous times connection failure.
memcached.sess_server_failure_limit = 1

; Write data to a number of additional memcached servers
; This is "poor man's HA" as libmemcached calls it.
; If this value is positive and sess_remove_failed_servers is enabled
; when a memcached server fails the session will continue to be available
; from a replica. However, if the failed memcache server
; becomes available again it will read the session from there
; which could have old data or no data at all
memcached.sess_number_of_replicas = 3

; for session handling.
; When consistent hashing is used, one can add or remove cache
; node(s) without messing up too much with existing keys
; default is Off
memcached.default_consistent_hash = On
```


Несколько часов я потратил из-за  `memcached.sess_server_failure_limit = 1` этой настройки. Хотелось дать серверу шанс, дать несколько попыток, но инструкция работает только если значение равно 1. 

**Итого: Было очень интересно, но мы переносим сессии в редис.**