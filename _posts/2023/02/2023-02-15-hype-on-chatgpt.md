---
layout: post
title: Хайпую на ChatGPT
categories: ChatGPT Unit Testing
---

Решил вернуться к блогу и улыбнулся, последний мой пост был ровно 1 год и 1 день назад. 

Мой коллега Данил поигрался с ChatGPT и это вышло очень любопытно. 

У нас есть вот такая задача для стажеров: 

```
Библиотека для работы с курсами валют

Цели:

Разобраться как работает composer. Курсы валют получать по апи биржи ХХХХ.

Библиотека должна содержать класс ExchangedAmount($from, $to, $amount)  
с единственным публичным методом toDecimal, который возвращает сумму 
после обмена рассчитанную по курсу покупки(buy).

$from, $to - это название валюты из списка USD, EUR, RUR
$amount - сумма которую собираются обменять.

Библиотеку можно установить чере composer require.
```

Получается вот такой код: 

```php
class ExchangedAmount 
{
	public function __construct(
		private string $from,
		private string $to,
		private float $amount
	) 
	{}
	
	public function toDecimal(): float
	{
		// Где-то тут мы ходим в апи за курсами
	}
}
```

Стажера легко озадачить попросив потом написать юнит тест на этот класс, ChatGPT с этим легко справляется.

Отдаем боту наш код и просим его "Сделай мой код тестируемым". 

Получаем вот такой результат: 

```php 
class ExchangeDataFetcher
{
    public function getActualCurrenciesData($from, $to)
    {
        // получаем данные
    }
}

class ExchangedAmount 
{
	public function __construct(
		private string $from,
		private string $to,
		private float $amount,
		private ExchangeDataFetcher $exchangeDataFetcher // добавляется параметр
	) 
	{}
	
	public function toDecimal(): float
	{
		// Где-то тут мы ходим в апи за курсами
	}
}
```

И кроме этого, генерируется и сам тест. 

```php

class ExchangeText extends PHPUnit\Framework\TestCase 
{
	public function testToDecimal() 
	{
		$mockDataFetcher = $this->getMockBuilder(ExchangeDataFetcher::class)
			->setMethods(['getActualCurrenciesData'])
			->getMock();
		
		$mockDataFetcher->expects($this->once())
			->method('getActualCurrenciesData')
			->with('UAH', 'USD')
			->willReturn([
				/*тут бот вставил необходимый формат */
			]);

		$exchange = new Exchange('UAH', 'USD', 100, $mockDataFetcher);
		$this->assertEquals(3.84, $exchange->toDecimal());
	}
}
```

Бот отработал очень круто, я такого не ожидал. 
Даже курс взял реальный на дату запроса и тестовые данные близки к оригиналу. 
Бот успешно стправился с генерацией тестов и на другие классы из разных проектов. 

Тем кто не еще не может писать юнит тесты на уровне chatGPT нужно научиться это делать. 
В [этой статье](https://otis22.github.io/unit/testing/2021/08/07/for-interns.html) я рассказываю подробно про такой рефакторинг с написанием тестов.  

Нужно смотреть, что еще умеет этот бот. Нужно уметь не меньше его, а еще уметь им пользоваться.

