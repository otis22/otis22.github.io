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