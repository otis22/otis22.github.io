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

### Дополнительный пример

Из такого, сделает:

```
def monthly_revenue(user_limit, is_on_voip, is_on_mobile, is_on_kwikbi):
    revenue = users_price(user_limit)
    if is_on_voip:
        revenue = revenue + voip_price(user_limit)
    if is_on_mobile:
        revenue = revenue + mobile_price(user_limit)
    if is_on_kwikbi:
        revenue = revenue + kwikbi_price(user_limit)
    return revenue
```

Вот это: 

```
def monthly_revenue(user_limit, is_on_voip, is_on_mobile, is_on_kwikbi):
    # Определение тарифа
    if user_limit == 1:
        grade = 'solo'
    elif 2 <= user_limit <= 5:
        grade = 'clinic'
    elif 6 <= user_limit <= 19:
        grade = 'clinic_plus'
    elif user_limit >= 20:
        grade = 'clinic_network'

    # Расчет стоимости пользователей
    if grade == 'solo':
        users_cost = 728 * user_limit
    elif grade == 'clinic':
        users_cost = 632 * user_limit
    elif grade == 'clinic_plus':
        users_cost = 426 * user_limit
    elif grade == 'clinic_network':
        users_cost = 374 * user_limit

    # Добавление стоимости VoIP
    if is_on_voip:
        if grade == 'solo':
            voip_cost = 101
        elif grade == 'clinic':
            voip_cost = 271
        elif grade == 'clinic_plus':
            voip_cost = 925
        elif grade == 'clinic_network':
            voip_cost = 1817
    else:
        voip_cost = 0

    # Добавление стоимости Mobile
    if is_on_mobile:
        if grade == 'solo':
            mobile_cost = 101
        elif grade == 'clinic':
            mobile_cost = 271
        elif grade == 'clinic_plus':
            mobile_cost = 925
        elif grade == 'clinic_network':
            mobile_cost = 1817
    else:
        mobile_cost = 0

    # Добавление стоимости KwikBi
    if is_on_kwikbi:
        if grade == 'solo':
            kwikbi_cost = 400
        elif grade == 'clinic':
            kwikbi_cost = 800
        elif grade == 'clinic_plus':
            kwikbi_cost = 2400
        elif grade == 'clinic_network':
            kwikbi_cost = 5600
    else:
        kwikbi_cost = 0

    # Общая месячная выручка
    revenue = users_cost + voip_cost + mobile_cost + kwikbi_cost
    return revenue
```