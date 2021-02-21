---
layout: post
title: Еще раз про тестирование 2(Как тестировать legacy)
categories: TDD, Unit, Integration
---

На днях натолкнулся на статью от Тинькофф и понял, что мы сейчас решаем теже проблемы. 
У нас перекос в пирамиде тестирования в сторону End To End и Интеграционных тестов. 

[Как решает Тинькофф](https://m.habr.com/ru/company/tinkoff/blog/539670/)

Я уверен, что тесты нельзя начать писать по приказу, я про это писал в прошлой [статье](https://otis22.github.io/tdd,/unit,/integration/2021/01/28/about-testing.html).
Это как минимум не принесет пользы. 

Пока, считаю, что тесты будут появлятся если сделать процесс их написания максимально простым и пользу от их написания заметной.
Мы сделали запуск тестов в одну команду, сразу после клонирование репозитория `make unit`. 

Пока, у нас нет цели покрыть 100%, наша цель писать максимально простые и полезные тесты. 

Мы не заставляем себя через силу сочинять тесты на старые классы типа Invoice, MedicalCards, которые погрязли в зависимостях. 
Мы не рыдаем потом по ночам в душе. Я так делал, от этого только отвращение к тестам.

Мы пишем тесты там где они очевидно нужны, там где нет, мы не заставляем их себя писать:

1. Я пишу функцию факториал и на неё пишу юнит тесты(результат зависит от аргументов)
   ```php
   factorial(1) == 1;
   factorial(0) == 1;
   ```
2. Я пишу класс парсер строки и делаю юнит тест (результат зависит состояния и нам легко влиять на состояние)
   ```php
   (new DateParser(""Тут где-то дата 15.02.86""))->date() == "15.02.86"
   ```
3. Я пишу обработчик json, чистый класс, который зависит только от состояния свойств, делаю юнит
   (результат зависит состояния и нам легко влиять на состояние)
   
   ```php
   expectException(\Exception::class)
   (new SomeSericeResponse('{success: false}'))->userId();
   ```
4. Мы видим в коде длинную и коричневую колбаску, которой можно дать имя и сделать из неё класс
   Видим такое
   ```php
   if ($admissionPostData && is_numeric($userId) && $userId > 0) {
       $requestUri = "Admission";
       $admissionDtStart = new \DateTime($admissionPostData["admission_date"]);
   if (!array_key_exists('admission_length', $admissionPostData)) {
       $admissionLength = (new \DateInterval($this->admissionInterval))->format("%H:%I:%S");
       $admissionDtEnd = (new \DateTime($admissionPostData["admission_date"]))->add(new \DateInterval($this->admissionInterval));
   } elseif (array_key_exists('admission_length', $admissionPostData) && ($admissionPostData['admission_length'] != "00:00:00")) {
       $admissionLength = $admissionPostData['admission_length'];
       $admissionDtEnd = clone $admissionDtStart;
       $admissionLengthData = explode(':', $admissionLength);
       $duration = "PT" . $admissionLengthData[0] . "H" . $admissionLengthData[1] . "M" . $admissionLengthData[2] . "S";
       $admissionDtEnd->add(new \DateInterval($duration));
   } else {
       $admissionLength = (new \DateInterval($this->admissionInterval))->format("%H:%I:%S");
       $admissionDtEnd = (new \DateTime($admissionPostData["admission_date"]))->add(new \DateInterval($this->admissionInterval));
   }
   $formParams = [
       "reception_write_channel" => (isset($admissionPostData['reception_write_channel'])) ? $admissionPostData['reception_write_channel'] : 'vetmanager',
       "type_id" => $admissionPostData["admission_type_id"],
       "admission_date" => $admissionDtStart->format("Y-m-d H:i:s"),
       ...
   ]
   ```

ужасаемся, рыдаем и выносим длинный иф в объект и покрываем его тестами, попутно находим десяток багов, которые раньше невозможно было увидеть
```php
if ($admissionPostData && is_numeric($userId) && $userId > 0) {
    $requestUri = "Admission";
    $admissionDate = new AdmissionDateData(admissionPostData, $this->admissionInterval);
    
    $formParams = [
        "reception_write_channel" => (isset($admissionPostData['reception_write_channel'])) ? $admissionPostData['reception_write_channel'] : 'vetmanager',
        "type_id" => $admissionPostData["admission_type_id"],
        "admission_date" => $admissionDate->start()->format("Y-m-d H:i:s"),
        ...
    ];
}
```
 
И делаем класс вроде этого: 
```php
сlass AdmissionDateData {
    ...
    public function __construct($data, $interval);
    private isExistPostLength() : bool
    {
        return array_key_exists('admission_length', $this->data) && ($this->data['admission_length'] != "00:00:00");
    }
    public function start() : DateTime
    {
        return  new \DateTime($this->data["admission_date"]);
    }
    public function length() : DateInterval
    {
        return $this->isExistPostLength() ? $this->$data['admission_length'] : new \DateInterval($this->interval);
    }
```

И попутно находим десяток багов при написании тестов. Получаем кайф от написания тестов.

## Вывод 

Пару месяцев поработать в таком ключе, только очевидные тесты, от души исходят которые.  Главное не делать тесты там где они вредят:
1. Не начинайте тестировать класс, который знает про множество других классов и общается с ними напрямую
2. Не пытайтесь тестировать поведение, типа сколько раз вызвался метод если клиент активный, тестируйте только результат, там где можно сравнить $a == $b
3. Не пишите много ассертов в одном тесте
4. Не бойтесь дублирования в тестах, дублирование ради выразительности теста - добро, а не зло. Тест нам помогает понять как рабтать с нашим классом.



