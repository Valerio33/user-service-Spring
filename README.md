## Уточнение
Попросил чат гопоты создать простенькую веб-морду чтоб тыкать кнопочки, а не писать комманды для работы со списком пользователей.

## Архитектура

HTTP → UserController → UserService → UserRepository → PostgreSQL.

## Запуск на Windows / PowerShell

Если база ещё не создана, выполните в psql:

```sql
CREATE DATABASE user_db;
```

В терминале проекта безопасно запросите пароль (не сохраняется в истории команд):

```powershell
$credential = Get-Credential -UserName postgres -Message 'Данные PostgreSQL'
$env:DB_USERNAME = $credential.UserName
$env:DB_PASSWORD = $credential.GetNetworkCredential().Password
mvn spring-boot:run
```

По умолчанию БД: jdbc:postgresql://localhost:5432/user_db, API: http://localhost:8080.
При необходимости установите DB_URL и SERVER_PORT. Пароль обязателен через DB_PASSWORD.
Файл .env автоматически не читается. Для запуска через IDEA задайте эти переменные
в конфигурации запуска UserServiceApplication.

Для запуска собранного приложения (в том же терминале с переменными):

```powershell
mvn clean package
java -jar target/user-service-1.0-SNAPSHOT.jar
```

package выполняет быстрые тесты; полный цикл с интеграционными тестами — verify.

## Веб-интерфейс

После запуска приложения откройте http://localhost:8080 в браузере.
Форма позволяет создать пользователя или сохранить изменения выбранной записи.
В таблице доступны кнопки «Изменить» и «Удалить»; удаление требует подтверждения.
Кнопка «Обновить список» перечитывает записи из базы данных.
Ошибки API, в том числе занятый email, показываются на странице.

Интерфейс: обычные HTML/CSS/JavaScript в src/main/resources/static, без Node.js и отдельной сборки.
Spring Boot раздаёт страницу и API с одного адреса. Не открывайте index.html двойным щелчком:
страница должна работать через запущенное приложение. Запуск Spring Boot по-прежнему необходим,
после него CRUD выполняется кнопками в браузере.

Если версия со Spring уже установлена, достаточно скопировать каталог static в
src/main/resources, перезапустить приложение и открыть его адрес.

## API

| Метод | Путь | Успех |
|---|---|---|
| GET | /api/users | 200, массив DTO (пустой массив при отсутствии записей) |
| GET | /api/users/{id} | 200, DTO |
| POST | /api/users | 201, DTO и Location |
| PUT | /api/users/{id} | 200, обновлённый DTO |
| DELETE | /api/users/{id} | 204, пустое тело |

POST и PUT принимают полный JSON:

```json
{"name":"Иван","email":"ivan@example.com","age":25}
```

Имя: 1–100 символов после удаления пробелов по краям. Email обязателен, до 254 символов,
проверяется формат; регистр сохраняется. Возраст — целое число 0–150. Все три поля обязательны.
PUT заменяет name/email/age, сохраняет id и createdAt; отсутствующий пользователь не создаётся.
Поля id и createdAt задаёт сервер; неизвестные входные поля отклоняются.
createdAt — локальное время сервера без часового пояса, как в предыдущем задании.

Ошибки в формате ProblemDetail (application/problem+json):
400 — некорректные данные/JSON/ID; 404 — пользователь не найден;
409 — занятый email или конфликт ограничений БД; 503 — ошибка доступа к БД.
В ошибке валидации errors содержит сообщения по полям.
Уникальность email защищена ограничением PostgreSQL, в том числе при одновременных запросах.

## Ручная проверка в PowerShell

```powershell
$base = 'http://localhost:8080/api/users'
$body = @{ name = 'Иван'; email = 'ivan@example.com'; age = 25 } | ConvertTo-Json
$user = Invoke-RestMethod -Method Post -Uri $base -ContentType 'application/json; charset=utf-8' -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
$user
Invoke-RestMethod -Uri "$base/$($user.id)"
Invoke-RestMethod -Uri $base
$updated = @{ name = 'Пётр'; email = 'petr@example.com'; age = 30 } | ConvertTo-Json
Invoke-RestMethod -Method Put -Uri "$base/$($user.id)" -ContentType 'application/json; charset=utf-8' -Body ([System.Text.Encoding]::UTF8.GetBytes($updated))
Invoke-RestMethod -Method Delete -Uri "$base/$($user.id)"
```

## Автоматические тесты

```powershell
mvn clean test
```

34 теста без Docker и локальной БД: UserTest, UserServiceTest (Mockito), UserControllerTest
(@WebMvcTest, MockMvc, подменён только сервис). Проверяются HTTP-коды, DTO, Location,
валидация POST/PUT, неправильный JSON, отсутствующие записи и конфликты.

```powershell
mvn clean verify
```

Перед запуском включите Docker Desktop в режиме Linux containers.

Изоляция: Mockito-моки сбрасываются между тестами; интеграционные тесты очищают таблицу
перед каждым методом. 
Если Docker недоступен, verify завершается ошибкой, а не скрывает тесты через skip.
Отчёты: target/surefire-reports и target/failsafe-reports.
