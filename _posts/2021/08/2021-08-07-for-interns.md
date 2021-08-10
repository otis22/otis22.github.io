---
layout: post
title: Стажерам про код, который можно протестировать
categories: Unit Testing
---

Сейчас наблюдаю за работой одного стажера и вижу, что людям очень тяжело написать код, который можно было бы протестировать. Я знаю, что как-то это объяснить одной статьей или книгой невозможно. Это навык, его нужно практиковать. 

Нескольким стажерам я давал задачу типа: 

*Получить данные у пользователя, провалидировать их. При помощи этих данных получить что-то у стороннего сервиса по REST API и выдать пользователю результат. Написать код с высоким покрытием юнит тестами. Именно юнит, тесты не должны взаимодейстовать с реальным сервисом.*  

Недавно я для своего pet проекта делал небольшую [библиотеку по работе с api reverso.context.net](https://github.com/otis22/reverso). Библиотека позволяет получать перевод слова на другой язык, с синонимами и вариантами использования. Там я решал такую же задачу. 

Задача: Получить у пользователя слово для перевода и информацию о языках. Провалидировать слово и языки. Получить в апи информацию и пребразовать в удобный для использования вид. Покрыть код юнит и интеграционными тестами. 

Обычно стажер пишет:

```php
use GuzzleHttp\Client;

class MyCoolClass {
    private $validLanguages = ['ru', 'en', ...];
    public function getTranslation($langFrom, $langTo, $word) {

        $client = new Client([
            'base_uri' => 'https://context.reverso.net/translation',
        ]);
        if (!in_array($langFrom, $this->validLanguages)) {
            throw new \InvalidArgumentException($langFrom . ' is not valid language');
        }
        // Валидируем еще $langTo и $word

        $response = $client->post(
            "/bst-query-service",
            [
                'body' => json_encode([
                    "source_text" => $word,
                    "target_text" => "",
                    "source_lang" => $langFrom,
                    "target_lang" => $langTo
                ])
            ]
        );  
        return json_decode($response->getBody()->getContents());      
    }
}
```

Потом присылает мне код и говорит что код написан, но протестировать это невозможно. В лучшем случае пишет интеграционный тест где дергает реальное апи. Вариантов как решить такую задачу у него нет. Подсказки не работают, тупик.

Как мы знаем, чтобы код можно было легко тестировать он должен соответствовать принципам SOLID, хотя бы двум:

* Single responsibility — принцип единственной ответственности
* Dependency inversion — принцип инверсии зависимостей

Давайте применим DIP к нашему коду и внедрим зависимость через инъекцию, а не будем создавать зависимость внутри класса. 


```php
class MyCoolClass {
    private $validLanguages = ['ru', 'en', ...];
    private ClientInterface $client;
    public function __construct(ClientInterface $client)
    {
        $this->client = $client;
    }
    ...
```

Ну и как бы все. Наш код уже можно тестировать прям по [инструкции guzzle](https://docs.guzzlephp.org/en/stable/testing.html) 

Реальный сервер мы можем заменить своим фейковым, который будет отдавать те ответы, которые нам нужно. Только благодаря соблюдению принципа DIP нам уже удалось написать тесты. 

```php
$mock = new MockHandler([
    new Response(200, [], $testJsonAnswer),

]);

$client = new Client(['handler' => HandlerStack::create($mock)]);

$myCool = new MyCoolClass($client);
#пишем любые ассерты
$this->assert...
```

Но наш код делает слишком много, мы это увидим когда будем писать тесты, сами тесты будут выглядеть очень страшно. Класс и валидирует и отправляет запросы. Еще и ответ как-то преобразовать в более удобный формат нужно. 

В своей версии решения этой задачи я разделил класс `MyCoolClass` на разные сущности. Я выделил такие классы как "Слово", "Язык", "Контекст", ... 

Приведу примеры: 

Нам в нашей задаче требуется валидировать язык, этим занимается сущность "Язык"

```php
final class Language implements Stringify
{
    ...
    public function __construct(string $language)
    {
        $this->language = $language;
    }

    public function asString(): string
    {
        if (!in_array($this->language, $this->validLanguagesList)) {
            throw new \InvalidArgumentException(...);
        }
        return $this->language;
    }
}
```

Код теста максимально простой

```php
$sut = new Language('klingon');
$this->expectException(\InvalidArgumentException::class);
$sut->asString();
```

Аналогично я поступил и с "Словом", выделил для него свой класс.

```php
$sut = new Word('this is a test phrase');
$this->expectException(\InvalidArgumentException::class);
$sut->asString();
```

Но у меня появилась проблема. Мой классный класс принимает слишком много аргументов в конструктор. 

```php
class MyCoolClass {
    ...
    public function __construct(Language $from, Language $to, Word, $word, ClientInterface $client)
    ...
}
```

Пытаюсь понять как исправить? Я беру близкие по смыслу сущности "Языки" и "Слово" и агрегирую их в одну ["Перевод"](https://github.com/otis22/reverso/blob/master/src/Translation.php). Сразу же на эту сущность накладываю обязанность формировать "body" запроса. 

Я протестировал еще один кусок логики. 

```php
$this->assertEquals(
    (
        new Translation(
            new Language("en"),
            new Language('ru'),
            new Word("test")
        )
    )->asArray()['source_text'],
    "test"
);
```

Дальше уже точно вкусовщина, но я попытаюсь объяснить почему я пошел по этому пути. 

Вместо класса `MyCoolClass` или какого-то `ReversoGateway` я создал класс [`Context`](https://github.com/otis22/reverso/blob/master/src/Context.php), который занимается обработкой ответа сервера и предоставляет удобный интерфейс к ответу. У класса есть один основной конструктор, который принимает ассоциативный массив ответа.

Вот такой простой тесты у меня получился. 

```php 
$this->assertEquals(
    'example',
    (
        new Context(
            json_decode(
                $this->ruToEngTranslateWordPrimerResponse,
                true
            )
        )
    )->firstInDictionary()
);
```

Дальше я использую вторичный конструктор. К сожалению в php не работает классическая перегрузка методов и мы не можем написать несколько раз один метод с разными сигнатурами. Поэтому я делаю это по пхпшному, через статический метод. 

```php
public static function fromTranslation(ClientInterface $client, Translation $translation): self
{
    return new self(
        json_decode(
            $client->request(
                "POST",
                "/bst-query-service",
                [
                    'headers' =>  [
                        "User-Agent" => "Mozilla/5.0",
                        "Content-Type" => "application/json; charset=UTF-8"
                    ],
                    'body' => json_encode(
                        $translation->asArray()
                    ),
                ]
            )->getBody()->getContents(),
            true,
            JSON_THROW_ON_ERROR
        )
    );
}
```

И тоже могу написать юнит тест, без реального сервера который проверит этот метод. 

```php
$mock = new MockHandler(
    [
        new Response(
            200,
            [],
            $this->ruToEngTranslateWordPrimerResponse
        )
    ]
);
$sut = Context::fromTranslation(
    new Client(['handler' => HandlerStack::create($mock)]),
    new Translation(
        new Language("en"),
        new Language('ru'),
        new Word("test")
    )
);
$this->assertEquals(
    "example",
    $sut->firstInDictionary()
);
```

**Будьте осторожны**, у меня класс контекст делает один простой запрос к серверу. Поэтому я решился оставить его в вторичном конструкторе. В вашей ситуации возможно потребуется отдельный класс вроде `ReversoGateway` который будет делать более сложные запросы и отдавать экземпляры класса `Context` которые будут заниматься исключительно обработкой.

Дальше я посмотрел на интерфейс моего класса, конечно это не удобно для клиентов библиотеки. Никто не захочет так возиться. 

```php
$sut = Context::fromTranslation(
    new Client(['handler' => HandlerStack::create($mock)]),
    new Translation(
        new Language("en"),
        new Language('ru'),
        new Word("test")
    )
);
```

Поэтому я сделал еще один вторичный конструктор, который внутри заниматься оборачиванием голых данных. 

```php
public static function fromLanguagesAndWord(string $languageFrom, string $languageTo, string $word): self
{
    return self::fromTranslation(
        new Client(['base_uri' => 'https://context.reverso.net/translation']),
        new Translation(
            new Language($languageFrom),
            new Language($languageTo),
            new Word($word)
        )
    );
}
```

В самую последнюю очередь я написал один интеграционный тест, который работает с реальным сервером.

```php
$this->assertEquals(
    "example",
    Context::fromLanguagesAndWord("ru", "en", "пример")
        ->firstInDictionary()
);
```

В итоге я получил код, в котором все изначальные требования покрыты юнит тестами. Код не страшно расширять и рефакторить. Тесты дают уверенность, что расширяя и измення код мы его не сломаем. SRP дает надежду, что каждое отдельное изменение не будет затрагивать большую часть классов, а значить они и не могут быть сломаны. Например, завтра мне нужно будет сделать так, чтобы библиотека работала не только с "ru", но и с "russian", "русский" и другими вариантами. Исправления произойдут только в классе Language. Интерфейс библиотеки получился удобным для конечного потребителя.

```php
use Otis22\Reverso\Context;

$context = Context::fromLanguagesAndWord("ru", "en", "пример");

$context->firstInDictionary(); # return "example" word, because it is the most popular variant in the reverso.net

$context->dictionary(); #return synonyms array ['example', 'sample', 'case', ...]

$context->examples(); #return examples sentences
```

P.S. Если у вас не получается написать тесты или тесты получаются громоздкими, то скорее всего ваш код тяжело переиспользовать. Ведь тесты - это частный случай переиспользования кода. Не стесняйтесь разбивать ваш код на классы, обычно это не усложняет его использование а упрощает. Сложную структуру классов легко скрыть паттерном Facade. Лапшекод, который переплетается в жутчайших хитросплетениях не исправит никакой паттерн.

