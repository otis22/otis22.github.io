---
layout: post
title: Чать 2. Пирамида Тестирования
categories: PhpVrnMeetup Тесты Деньги Пирамида
---

Тезисы: 

4. Пирамида тестирования. Какие бывают тесты?
5. Почему юнит тестов должно быть много? 

Навигация по серии статей:

* [Часть 1. Тесты стоят денег]({% post_url 2021/04/2021-04-01-tests-its-money %})
* [Часть 3. Реальная жизнь]({% post_url 2021/04/2021-04-06-real-life-of-tests %})

# Пирамида тестирования. Какие бывают тесты?

Пирамида тестирования выглядит так:

![pyramid](/images/2021/04/pyramid.png)

И отображает, что правильный проект должен иметь большое юнит тестов,
меньше интеграционных и совсем немного end-to-end тестов. 

Юнит-тестом называется автоматизированный тест, который проверяет правильность работы небольшого фрагмента кода (также называемого
юнитом); делает это быстро и поддерживая изоляцию от другого кода.

Интеграционные - любые тесты, который не юнит. Это может быть долгий тест участка кода, или же тест взаимодействия нескольких компонентов программы между собой.

Скзвозные тесты - тесты, которые максимально иммитируют действия пользователя и работают практически со всеми зависимостями проекта. В нашем случае это скорее selenium или подобные тесты.    

# Почему юнит тестов должно быть много? 
Это все правильная и логичная картина. Очевидно, что лучше писать больше быстрых тестов, чем больше медленных.

Когда-то я учился на курсах тестировщиков и особенно меня поразили 
2 вещи. Поразили прям очень сильно, жизнь, буквально, поделилась на до и после. 

## Позитивные и негативные тесты

Вам приходит задача вида: Пользователь нажимает на кнопку, затем выбирает что-то и сохраняет форму. 
Это позитивный сценарий. В нем все идет хорошо. Жизнь она сложнее. 

Недавно, моя жена вызвала такси в ЯндексГо и пока мы ждали такси, она решила сменить телефон своего Яндекс Аккаунта. 
Конечно, это привело к сбою. Как программист, смог бы вообразить,
что человек в момент ожидания будет менять телефон своего аккаунта?

Вот это уже негативный сценарий. Тот который мы не ожидали. Множество найденных багов в моем опыте были из этой серии.

Пользователь может нажать не на ту кнопку, на которую вы рассчитывали или внести неверные данные, может нажать что-то на определенной стадии, на которой это нельзя было нажимать.

Стоит ли думать про это когда мы пишем тесты? Думаю да, некоторые негативные сценарии нам следует автоматизировать.

## Классы эквивалентности 

Эта штука нужна тестерам, чтобы убедиться в работе компонента за минимальное число шагов. 

Допустим, у нас есть поле ввода даты рождения на сайте. У него есть ограничения. Дата рождения 
должна быть больше 1900 года и меньше 2021 года. Сколько нужно заполнить поле данными для его проверки?

Данные для проверки вы делите на классы эквивалентности, числа внутри этого класса равны для нашего поля:

1. Все числа, которые ниже минимальной границы
2. Число, которое равно нижнее границе
3. Числа, которые входят в наш диапазон
4. Число, которое равно верхней границе
5. Числа, которые выше верхней границы
6. Не числа

Итого 6 раз нам нужно заполнить поле, чтобы убедиться, что оно работает верно. Я, в лучшем случае ввожу одно число.

Эта история сильно расширяет границы того, что можно понаписать. Но в какой части пирамиды это писать? 

## Пример

У нас есть какой-то сервис. Мы показываем информер насколько авторизованный пользователь использует сервис в процентах. 

![pyramid](/images/2021/04/usage-power.jpg)

0-30 % - danger 
31-70 % - warning
71-100 % - good

Согласно нашей пирамиде нам нужно минимальное число сквозных тестов:

Сделаем тест, который при помощи какого-то Selenium зайдет на страницу сайта, и убедится, что информер у нас есть.

Интеграционных тестов может быть больше.(Авторизацию я упростил)

```
GET /usage-power?secret=$secret
```

Давайте еще обработаем какой-то негативный сценарий и попробуем достать данные совсем без авторизации.

```
GET /usage-power
```

А теперь давайте писать юнит тесты, главная наша функция вот, которая определяет на каком уровне
использования наш пользователь.

```php
    function usageLevel(float $usagePower): string
```

Как-то так выглядит наша функция и возвращает danger, warning или good.

Проверим, правильность работы для 'danger' зная про классы эквивалентности и `0-30 % - danger`. 

Выбираем цифры для тестирования. Я не привязываюсь к фреймворкам, для тестирования. Тестируем как есть при помощи `assert()`.

```php
assert(usageLevel(0) === 'danger', "Must be 'danger'");
assert(usageLevel(15) === 'danger', "Must be 'danger'");
assert(usageLevel(30) === 'danger', "Must be 'danger'");
assert(usageLevel(31) == 'warning', "Must be 'warning'");
```

Допишем тестов и на другие уровни. 3 теста на каждый. А еще допишем тестов для классов чисел, которые вне нашего диапазона от 0 до 100.

`$usagePower = -1`, класс чисел, которые ниже нашей границы. Мы ждем InvalidArgumentException, проверю так.

```php
expectException(InvalidArgumentException::class, fn() => usageLevel(-1));
```

Аналогично проверим и класс чисел, которые выше границы `$usagePower = 101`

Кроме этой функции у нас точно будет какой-то еще код, который тоже нужно покрыть тестами. Вот так и складывается наша пирамида. Мы могли бы сделать все теже проверки на уровне интеграционных или даже сквозных тестов. Но мы стараемся максимальное число проверок вынести на Unit уровень. Они запускаются чаще, у них меньше ложных срабатываний, они работают быстрее, они способствуют улучшению качества кода.

Но такие тесты более чувствительны к рефакторингу. Если бы мы решили усложнить функцию и сделать из неё класс, то нам прийдется переписать и все тесты, при этом интеграционные тесты мы бы не трогали.


