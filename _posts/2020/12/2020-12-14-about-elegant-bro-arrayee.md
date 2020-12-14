---
layout: post
title: Elegant Way в php
tags: ElegantObjects
---

Кто не слышал про [Elegant Objects](https://www.elegantobjects.org/)? Советую познакомиться и с концепцией и с её автором. Даже если вы с автором не согласны, вам не будет скучно.

Чтобы ваши объекты были элегантными, они должны соотвествовать принципам:

* No null
* No code in constructors
* No getters and setters
* No mutable objects
* No readers, parsers, controllers, sorters, and so on
* No static methods, not even private ones
* No instanceof, type casting, or reflection
* No public methods without a contract (interface)
* No statements in test methods except assertThat
* No ORM or ActiveRecord
* No implementation inheritance

Мне нравится код, который получается в итоге. В мире php нашел ребят, которые уже написали несколько оберток над стандартными php функциями - [elegant-bro](https://github.com/elegant-bro).

Например [Arrayee](https://github.com/elegant-bro/arrayee) - библиотека для элегантной работы с массивами

```php
# Чтобы из простого массива получить элегантный
$elegantArray = new Just([1, 2, 3]);
# Чтобы вернуться к простому массиву
$notElegantArray = $elegantArray->asArray();
```

```php
# объеденим 2 массивам 
$megred = new Merged(
    new Just([1, 2, 3]),
    new Just([4, 5, 6])
);
# умножим на 2 все элементы
$mapped = new Mapped(
    $merged,
    function ($item) {
        return $item*2;
    }
);
# ... сделаем что-то еще с классами Reversed, Sorted, JsonDecoded
# и получим обработанный массив когда он будет нам нужен
$mapped->asArray();
```
Можно строить матрешки из классов разделяя ответсвенность. Мне нравится, а вам? 