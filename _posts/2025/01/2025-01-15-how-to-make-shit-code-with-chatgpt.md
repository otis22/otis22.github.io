---
layout: post
title: Как из хорошего кода сделать говно при помощи ChatGPT?
categories: Python, ChatGPT, Redash
---

Есть аналитический скрипт, который запускается в Jupyter Notebook. Он рассчитан для подсчёта платных опций на разных тарифах. Выглядит всё красиво и понятно:

```python
def tariff_grade(user_limit):
    if user_limit == 1:
        return 'solo'
    elif 2 <= user_limit <= 5:
        return 'clinic'
    elif 6 <= user_limit <= 19:
        return 'clinic_plus'
    elif user_limit >= 20:
        return 'clinic_network'

def kwikbi_price(user_limit):
    price = {
        'solo': 400,
        'clinic': 800,
        'clinic_plus': 2400,
        'clinic_network': 5600
    }
    return price.get(tariff_grade(user_limit))

def voip_price(user_limit):
    price = {
        'solo': 101,
        'clinic': 271,
        'clinic_plus': 925,
        'clinic_network': 1817
    }
    return price.get(tariff_grade(user_limit))
```

Скрипт работает как часы: всё структурировано, повторяющийся код вынесен в отдельные функции. Но стоит перенести этот код в **Redash**, как начинается превращение в говно.

---

### Проблемы песочницы Redash

**Redash** запускает Python-скрипты в ограниченном окружении, и там вас поджидают сюрпризы:

#### Ограничения:

- Нельзя использовать классы.
- Функции не могут вызывать другие функции.
- Импорты внутри функций не работают, и их приходится повторять.

#### Адаптация кода: 

Чтобы обойти ограничения, код приходится переделывать вручную, превращая его в кашу из `if-else` и дублируя логику.

---

### ChatGPT и магия превращения

Просто попросите ChatGPT переписать код с учётом ограничений Redash. И вот, вместо аккуратного и читаемого скрипта, вы получите это:

```python
def voip_price(user_limit):
    if user_limit == 1:
        return 101
    elif 2 <= user_limit <= 5:
        return 271
    elif 6 <= user_limit <= 19:
        return 925
    elif user_limit >= 20:
        return 1817

def kwikbi_price(user_limit):
    if user_limit == 1:
        return 400
    elif 2 <= user_limit <= 5:
        return 800
    elif 6 <= user_limit <= 19:
        return 2400
    elif user_limit >= 20:
        return 5600
```

### Что с этим не так?

- **Дублирование кода:** Логика, которая раньше находилась в одной функции, теперь размазана по всему скрипту.  
- **Сложности в поддержке:** Любое изменение нужно вносить сразу в несколько мест.  
- **Утрата структуры:** Скрипт становится нечитаемым, и разбираться в нём сложнее.  

---

### Итог

ChatGPT действительно справляется с превращением кода в говно. Разрабатываю аналитические скрипты в Jupyter Notebook не думая про ограничения и легко их переношу в Redash. Пора уже в Redash встроить опцию в редакторе - "сделать говно".