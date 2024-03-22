# Тестовое задание

## 1. Вопросы для разогрева

1. Расскажите, с какими задачами в направлении безопасной разработки вы сталкивались?
   - Ручной анализ исходного кода на наличие уязвимостей с целью их дальнейшей эксплуатации (смарт-контракты в том числе)
   - Анализ исходного кода с помощью SAST-инструмента (в моем случае Semgrep)
   - Поиск проблем с памятью в коде С/С++ с помощью valgrind и т.д.
2. Если вам приходилось проводить security code review или моделирование угроз, расскажите, как это было?
   - Я играю в CTF-команде, мы с командой часто участвуем в классическом CTF (Attack-Defense), где надо анализировать исходный код приложения, чтобы найти уязвимости, которые потом пофиксить и начать атаковать других. Приходилось использовать SAST инструменты для автоматизации поиска уязвимостей.
3. Если у вас был опыт поиска уязвимостей, расскажите, как это было?
   - В основном на CTF и на специализированных площадках дял повышения навыков взлома (tryhackme, hackthebox, rootme, picoctf и т.д.). В задачах на реверс инжиниринг находила уязвимости в написанном кастомном шифровании, захардкоженные данные, в вебе все подряд: SSTI, XSS, CMDi, LFI/RFI, ошибки в конфигурации приложений.
4. Почему вы хотите участвовать в стажировке?
   - Считаю, что на этой стажировке я смогу получить опыт работы AppSec-специалистом и понимание, куда двигаться дальше.

## 2. Security code review

### Часть 1. Security code review: GO

Вижу в данном коде две проблемы:

#### SQL Injection

```go
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
rows, err := db.Query(query)
```

#### Data transmission without TLS

```go
log.Fatal(http.ListenAndServe(":8080", nil))
```

##### Последствия

1. Improrer Authentication: получение доступа к базе данных.
1. SQLi: Компрометация, уничтожение данных, повышение привилегий в системе, обход аутентификации.
2. Data transmission without TLS: Атака "человек посередине" (MITM), утечка конфиденциальной информации.

##### Как исправить

1. Improrer Authentication: удаление аутентификационных данных из кода, и сокрытие их в переменных окруженых окружения либо какой-нибудь другой удобный способ.
2. SQLi
	1. Использование параметризованных запросов: нельзя использовать просто Sprintf без фильтрации ([Ссылка на go.dev с описанием](https://go.dev/doc/database/sql-injection)).
	Исправленный код:

		```go
		query := "SELECT * FROM products WHERE name LIKE ?"
		rows, err := db.Query(query, "%"+searchQuery+"%")
		```

   1. Тщательная фильтрация пользовательского ввода (четко задан тип, запрет символов или ключевых слов, которые могут привести к инъекции).
   2. Использование защищенных ORM.
3. Data Transmssion Without TLS: Нужно добавить поддержку TLS, исправленный код выглядит вот так:

    ```go
    log.Fatal(http.ListenAndServeTLS(":8080", certFile, keyFile, nil))
    ```

##### Лучший способ исправления уязвимости

Для SQLi - первый вариант, потому что при реализации тщательной фильтрации может возникнуть так называемый "человеческий фактор", из-за чего программист может прописать в blacklist не все возможные моменты, из-за чего приложение все равно останется уязвимым. В первом варианте модуль sql под капотом экранирует спецсимволы, из-за чего становится невозможным провести SQLi. Недостаток третьего варианта в том, что Gorm хоть и использует параметризацию запросов под капотом, но в связке со Sprintf без предварительной обработки запроса использование этой ORM тоже становится уязвимым.

### Часть 2: Security code review: Python

#### Пример №2.1

В приведенном коде два уязвимых места:

##### Server Site Template Injection

```python
name = request.values.get('name')
age = request.values.get('age', 'unknown')
output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
```

##### Active Debug Code и Data transmission without TLS

```python
if __name__ == "__main__":
    app.run(debug=True) 
```

##### Последствия

1. SSTI: получение доступа к конфиденциальной информации, RCE, DoS
2. Active Debug Code: раскрытие конфиденциальной информации о приложении (злоумышленник может найти какой-то конкретный вектор атаки, который применим к стеку, на котором работает приложение).
3. Data Transmission Without TLS: то же, что и в Примере №1.

##### Как можно исправить

1. SSTI
   1. Санитайзинг входных данных, например, с помощью escape:

        ```python
        from jinja2 import Template, escape
        name = escape(request.values.get('name'))
        age = escape(request.values.get('age', 'unknown'))
        output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
        ```

   2. Проверка типа передаваемого значения и фильтрация передаваемых данных: age - это число, поэтому надо сделать проверку, является ли переданное значение числом, a в параметре name запретить спец. символы, например:

      ```python
	  name = request.values.get('name')
	  age = request.values.get('age', 0)
	  if isistance(age, int) and re.findall(r"[${<['}%\.]", name):
		output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
	  ```

2. Active Debug Code: убераем Debug=True в app.run().
3. Data transmission without TLS: добавляем поддержку tls/ssl в помощью сертификата и ключа в параметр ssl_context в app.run().

##### Лучший способ исправления уязвимости

SSTI - первый способ, потому что благодаря функции escape Jinja2 любую последовательность символов не будет воспринимать как служебные, что исключает надобность в фильтрации, при реализации которой можно забыть какой-нибудь символ.

#### Пример №2.2

В приведенном коде тоже два уязвимых места:

##### OS Command Injection

```python
hostname = request.values.get('hostname')
cmd = 'nslookup ' + hostname
output = subprocess.check_output(cmd, shell=True, text=True)
```

##### Active Debug Code и Data transmission without TLS

```python
if __name__ == "__main__":
    app.run(debug=True) 
```

##### Последствия

1. OS Command Injection: RCE, кража конфиденциальных данных
2. Active Debug Mode: то же, что и в Примере №2.1.
3. Data Transmission Without TLS: то же, что и в Примере №1.

##### Как можно исправить

1. Os Command Injection
    1. Добавить фильтрацию вводимых пользователем значений (запретить, например, спецсимволы, которые могут привести к атаке - фигурные скобки и т.д, можно запретить конкретные слова, которые могут привести к удаленному выполнению кода).
    2. Использование специализированных библиотек для работы с DNS (aiodns, dnspython и т.д).
2. Active Debug Code: решается так же, как и в Примере №2.1.
3. Data Transmission Without TLS: так же, как и в Примере №1.

##### Лучший способ исправления уязвимости

Os Command Injection - второй вариант, потому что программист может реализовать недостаточную фильтрацию, из-за чего приложение все равно остается уязвимым. Вот полностью исправленный код c учетом исправлений Active Debug Code и Data transmission without TLS:

```python
from flask import Flask, request
import asyncio
import aiodns

app = Flask(__name__)

async def dns_lookup(hostname):
    resolver = aiodns.DNSResolver()
    try:
        result = await resolver.query(hostname, 'A')
        return str(result)
    except aiodns.error.DNSError as e:
        return str(e)

@app.route("/dns")
def handle_dns():
    hostname = request.values.get('hostname')
    result = asyncio.run(dns_lookup(hostname))
    return result

if __name__ == "__main":
    app.run(ssl_context=('cert.pem', 'key.pem'))

```

## 3. Моделирование угроз

### Потенциальные проблемы безопасности

Можно выделить несколько возможных проблем в безопасности:

1. Проблемы с аутентификацией и авторизацией в общении пользователя и веб-приложения.
2. Проблемы с безопасных хранением данных.
3. Проблемы с коммуникацией между сервисами.

### Последствия эксплуатации проблем

1. Проникновение в сеть, несанкционированный доступ к информации.
2. Утечка конфиденциальной информации либо ее потеря.
3. Dos или DDoS, модификация или подмена передаваемой информации, перехват данных.

### Cпособы исправления уязвимостей и смягчения рисков

1. Нужно убедиться, что нет никаких изъянов в реализации аутентификации. Например, если используется JWT, надо быть уверенным, что при генерации токена используется сильный секрет, который злоумышленник не сможет пробрутить, и что используется надежный алгоритм подписи.
2. Нужно убедиться, что база данных PostgreSQL не использует дефолтные креды и что вся передаваемая информация между базой данных и backend application фильтруется?
3. Нужно убедиться, что настроены механизмы аутентификации и авторизации между микросервисами (например, JWT или OAuth2), настроен контроль доступа.

### Уточняющие вопросы

1. Какой метод аутентификации в приложении реализован?
2. Как деплоятся сервисы, проходят ли они проверку SAST или DAST инструментом?
3. Шифруется ли трафик между микрофонтом и Backend Application?
4. Как происходит оркестрация микросервисов, правильно ли настроены системные компоненты оркестратора?
5. С какими правами запускаются Docker-контейнеры с сервисами?
