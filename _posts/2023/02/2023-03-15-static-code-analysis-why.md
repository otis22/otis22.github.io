---
layout: post
title: Зачем статический анализ кода?
categories: phpstan static code analysis
---

После просмотра докладов о инструментах разработки на последней [подлодке](https://podlodka.io/phpcrew)
у некоторых коллег возник вопрос: а зачем нужен статический анализ?

Все это выглядит каким-то усложнением. У нас прекрасная пышечка, мы знаем, что динамическая типизация это так легко и быстро.
Мало того, что нас теперь заставляют декларировать типы, так еще и какой-то psalm|phpstan завезли. Все декларируй, пиши какие-то сложные инструкции в него. 
Чем больше ты напишешь, тем строже анализатор к тебе будет и тем чаще будет тебя ругать. Никому не хочется быть наруганным. 

Зачем же все про это говорят? Может строгий надзиратель и не так плохо? Давайте посмотрим как работается с статическим анализом и вы решите нужен он вам или нет. 

Для статьи я придумал несложный, но очень важный проект. 

Мы будем ходить в API ЦБ РФ по этому адресу и получать json. 

```json
{
    "disclaimer": "https://www.cbr-xml-daily.ru/#terms",
    "date": "2023-03-15",
    "timestamp": 1678827600,
    "base": "RUB",
    "rates": {
        "AUD": 0.02000176,
        "AZN": 0.02260857769,
...
    }
}
```

Для проекта важна свежесть курса, поэтому нам нужен класс, при помощи которого мы удобно сможем получать timestamp из этого json.

Перейдем к коду: 

```php
use GuzzleHttp\ClientInterface;

final readonly class Latest
{
    /**
     * @param array<string, mixed> $latest
     */
    public function __construct(
        private array $latest
    ) {
    }

    public static function fromHttpClient(ClientInterface $client): self
    {
        $response = $client->request('GET', 'https://www.cbr-xml-daily.ru/latest.js')
            ->getBody()
            ->getContents();
        return new self(
            json_decode($response, true)
        );
    }

    public function timestamp(): int
    {
        return $this->latest['timestamp'];
    }
}
```

Вот такой код. Будет он работать? Возможно, PHPStorm на него не ругается и ничего не подсвечивает. Давайте напишем на него unit тестов и узнаем. 

```php
    public function testLatestTimestampFromPreparedArray(): void
    {
        $this->assertEquals(
            1678741200,
            (
              new Latest(
                  $this->validDecodedJson()
              )
          )->timestamp()
        );
    }
``` 

Все хорошо: `OK (1 test, 1 assertion)`.

Попробуем узнать, что и с валидным ответом сервера наш код будет работать. 

```php
    public function testLatestTimestampWithValidResponse(): void
    {
        $handlerStack = HandlerStack::create(
            new MockHandler([new Response(200, [], $this->validStringJson())])
        );
        $this->assertEquals(
            1678741200,
            Latest::fromHttpClient(
            	new Client(['handler' => $handlerStack])
            )->timestamp()
        );
    }

```

`OK (2 tests, 2 assertions)` 

Все супер. У нас есть код, он протестирован. Можем написать еще один интеграционный тест с реальным сервером.

```php
    public function testLatestTimestampFromRealServer(): void
    {
        $this->assertTrue(
            Latest::fromHttpClient(new Client())->timestamp() > 1000
        );
    }
```

`OK (3 tests, 3 assertions)`. 

Все выглядит прилично, мы уверрены, что код работает. Запустим на нем статанализ, а у нас все красное. Что от нас хочет этот фашист? 

```
vendor/bin/phpstan analyse --level=max src tests

 ------ --------------------------------------------------------------------- 
  Line   src/Latest.php                                                       
 ------ --------------------------------------------------------------------- 
  23     Parameter #1 $latest of class Latest constructor  
         expects array<string, mixed>, mixed given.                           
  29     Method Latest::timestamp() should return int but  
         returns mixed.                                                       
 ------ --------------------------------------------------------------------- 


                                                                                
 [ERROR] Found 2 errors 

```

Я использовал phpstan, возможно, лучше бы использовать psalm. Не важно, давайте пройдемся по ошибкам. 

Первая ошибка на строке 23 `Parameter #1 $latest of class Latest constructor expects array<string, mixed>, mixed given.`.

```php
        return new self(
            json_decode($response, true)
        );

```

Анализатор нам подсказал, что json_decode возвращает mixed. Я не убедился в том, что у нас есть нормальный ответ сервера. 
Могли моргнуть сервера ЦБ РФ, мог моргнуть интернет на нашем сервере. Сервис ЦБ РФ мог отдать невалидный json.
Функционал опирающийся на наш код сломается, нам будут писать возмущенные пользователи, мы пойдем проверять, а у нас все будет работать. 
Такую плавающую ошибку можно годами отлавливать. Статический анализатор призывает нас проверить ответ сервера и написать понятное сообщение о ошибке. 

Исправим код и запустим проверку. 

```
    $decodedJson = json_decode($response, true);
    if (!is_array($decodedJson)) {
        throw new \Exception("Invalid server www.cbr-xml-daily.ru response: " . $response);
    }
```

`[ERROR] Found 1 error` - на одну ошибку стало меньше, при этом наш код стал отдавать понятные ошибки, которые можно перехватить. 
Время на решение потенциальных проблемы уменьшается. 

Давайте уберем вторую ошибку `Method Latest::timestamp() should return int but returns mixed.`. Анализатор нам точно подсказывает, что в этом месте тоже код не безопасен. Если сервер нам вместо `1678741200` вернет `"1678741200"`, а у нас declare(strict_types=1), то мы получим  `Fatal error: Uncaught TypeError`. 

Просто приведение типов `(int) $this->latest['timestamp'];` не помогает, анализатор ругается, что в поле timestamp может прийти mixed. 


Анализатор вынуждает писать еще одну проверку. По невнимательности я пишу `if (!is_numeric($this->latest))` и получаю от анализатора такое предупреждение:
`Call to function is_numeric() with array<string, mixed> will always evaluate to false.`. Конечно, я забыл про ключ в массиве `latest` и анализатор меня предупредил, что моя проверка никогда бы не сработала. Кто не допускал таких тупых ошибок? Благодаря анализатору я не потратил никаких сил и времени на поиск этой ошибки. 

Исправим и добавим проверочек.

```
    public function timestamp(): int
    {
        if (!isset($this->latest['timestamp'])) {
            throw new \Exception("Timestamp value does not exists.");
        }
        if (!is_numeric($this->latest['timestamp'])) {
            throw new \Exception("Can't convert timestamp value to numeric. Value: " . $this->latest['timestamp']);
        }
        return (int) $this->latest['timestamp'];
    }
```

Ура!!! Все позеленело. Я получил массу удовольствия от того, что анализатор несколько раз отверг мой код. [Говорят](https://www.youtube.com/watch?v=7rtQ4yQVAK0), что хорошие программисты получают от этого удовольствие. 


**Итого: ** мы написали код, покрыли его тестами и мы были уверены, что наш код рабатает и надежен.
Статический анализ выявил еще несколько недочетов, которые однозначно улучшат надежность и удобство использования нашего кода. 
Анализатор помог избежать тупых ошибок, как в примере с `will always evaluate to false` и сэкономить моё время. 

Нужен ли статический анализ конкретно вам? Незнаю, а вы установите и попробуйте запустить на своем проекте постепенно повышая строгость проверки. 
Проверьте насколько вы крутой программист.