# Тестовое задание

## 1. Вопросы для разогрева

1. Расскажите, с какими задачами в направлении безопасной разработки вы сталкивались?
    - Ручной анализ исходного кода на наличие уязвимостей с целью их дальнейшей эксплуатации (блокчейн в том числе)
    - Анализ исходного кода с помощью SAST-инструмента (в моем случае Semgrep)
    - Поиск проблем с памятью в коде С/С++ с помощью valgrind и т.д.

2. Если вам приходилось проводить security code review или моделирование угроз, расскажите, как это было?
    - Я играю в CTF-команде, мы с командой часто участвуем в классическом CTF (Attack-Defense), где надо анализировать исходный код приложения, чтобы найти уязвимости, которые потом пофиксить и начать атаковать других.
3. Если у вас был опыт поиска уязвимостей, расскажите, как это было?
    - Как я уже написала выше, в основном на CTF
4. Почему вы хотите участвовать в стажировке?

## 2. Security code review

### Часть 1. Security code review: GO

Вижу в данном коде две проблемы: 
SQL Injection

```go
    query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
    rows, err := db.Query(query)
```

Data transmission without TLS

```go
log.Fatal(http.ListenAndServe(":8080", nil))
```

### Часть 2: Security code review: Python

#### Пример №2.1

В приведенном коде два уязвимых места:

Server Site Template Injection

```python
name = request.values.get('name')
age = request.values.get('age', 'unknown')
output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
```

Active Debug Code

```python
if __name__ == "__main__":
    app.run(debug=True) 
```

- Последствия:
    1. SSTI: получение доступа к конфиденциальной информации, RCE, DoS (CVSS 7-9)
    2. Active Debug Code: раскрытие конфиденциальной информации о приложении (злоумышленник может найти какой-то конкретный вектор атаки, который применим к стеку, на котором работает приложение)
- Как можно исправить:
    1. Санитайзинг входных данных, например, с помощью escape:

        ```python
        from jinja2 import Template, escape
        name = escape(request.values.get('name'))
        age = escape(request.values.get('age', 'unknown'))
        output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
        ```

    2. 

#### Пример №2.2

В приведенном коде тоже два уязвимых места:

OS Command Injection:

```python
hostname = request.values.get('hostname')
cmd = 'nslookup ' + hostname
output = subprocess.check_output(cmd, shell=True, text=True)
```

Active Debug Code

```python
if __name__ == "__main__":
    app.run(debug=True) 
```

- Последствия: 
    1. OS Command Injection: RCE, кража конфиденциальных данных (CVSS critical)
    2. Active Debug Mode: раскрытие конфиденциальной информации о приложении (злоумышленник может найти какой-то конкретный вектор атаки, который применим к стеку, на котором работает приложение)
- Как можно исправить: 
    1. Добавить фильтрацию вводимых пользователем значений (запретить, например, спецсимволы, которые могут привести к атаке - &,|, можно запретиьт конкретные слова)

## 3. Моделирование угроз
