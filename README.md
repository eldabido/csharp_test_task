# Vulnerability Tracker API

Тестовое задание на C#: REST API для учёта уязвимостей и управления фоновыми задачами сканирования.

Реализована обязательная часть и одна дополнительная задача: настраиваемое ограничение параллельных сканирований с очередью.

Сканирование — имитация. Сервис проверяет формат URL, асинхронно ожидает заданное время и возвращает фиксированный демонстрационный результат. Репозиторий не скачивается, его существование и доступность не проверяются, исходный код не анализируется. В результате явно указано `isSimulation: true` и `filesScanned: 0`; находка и путь `demo/Example.cs` вымышлены.

Сборка и проверка:

```
dotnet restore
dotnet build --no-restore
dotnet test --no-build
dotnet run --project src/VulnerabilityTracker.Api
```

Адрес по умолчанию: `http://localhost:5080`. Корневой путь `/` возвращает `404`: сервис предоставляет только API.

Другой адрес можно передать при запуске:

```
dotnet run --project src/VulnerabilityTracker.Api -- --urls http://localhost:8080
```

## API

Для запросов с телом используется `Content-Type: application/json`. 

| Метод | Путь | Назначение | Успех |
| --- | --- | --- | --- |
| `POST` | `/api/vulnerabilities` | Создать уязвимость | `201 Created` |
| `GET` | `/api/vulnerabilities/{id}` | Получить уязвимость | `200 OK` |
| `POST` | `/api/scans` | Поставить сканирование в очередь | `202 Accepted` |
| `GET` | `/api/scans/{id}` | Получить статус и результат | `200 OK` |

Идентификаторы - GUID. Для неизвестной записи или некорректного GUID в пути возвращается `404`. Ошибки валидации и JSON возвращаются с `400`, неподдерживаемый Content-Type — `415`, неподдерживаемый HTTP-метод - `405`. Неожиданные ошибки запроса возвращаются с `500` без внутренних подробностей.
### Создание уязвимости

```http
POST /api/vulnerabilities
Content-Type: application/json

{
  "name": "SQL-инъекция",
  "severity": "high"
}
```

Оба поля обязательны. Имя не может быть пустым или состоять только из пробелов; Критичность: `low`, `medium`, `high`, `critical`.

Ответ `201` содержит запись с автоматически созданными `id` и `createdAt` в UTC:

```json
{
  "id": "4763dca7-5b22-4534-9042-cf5b689ec0d8",
  "name": "SQL-инъекция",
  "severity": "high",
  "createdAt": "2026-01-01T12:00:00+00:00"
}
```

### Запуск и статус сканирования

```http
POST /api/scans
Content-Type: application/json

{
  "repositoryUrl": "https://gitlab.com/example/project"
}
```

URL обязателен, не длиннее 2048 кодовых единиц UTF-16, абсолютный, с протоколом HTTP или HTTPS, без логина и пароля в адресе.

Ответ `202` сразу возвращает ID задачи и первоначальный снимок со статусом `queued`, не ожидая сканирования. К моменту следующего GET задача уже может выполняться или завершиться.

| Статус | Значение |
| --- | --- |
| `queued` | Ожидает свободного обработчика |
| `running` | Выполняется |
| `completed` | Успешно завершена; поле `result` содержит результат |
| `failed` | Сканирование завершилось ошибкой; поле `error` содержит сообщение |

`createdAt` — время постановки в очередь, `updatedAt` — время последнего изменения статуса. Все даты в UTC. Опрос статуса не меняет время. В состояниях, отличных от `completed`, `result` равен `null`. Для `failed` подробная ошибка пишется в лог; клиент получает краткое сообщение.

Пример результата завершённой задачи:

```json
{
  "id": "4763dca7-5b22-4534-9042-cf5b689ec0d8",
  "repositoryUrl": "https://gitlab.com/example/project",
  "status": "completed",
  "createdAt": "2026-01-01T12:00:00+00:00",
  "updatedAt": "2026-01-01T12:00:05+00:00",
  "result": {
    "isSimulation": true,
    "filesScanned": 0,
    "summary": "Демонстрационный результат. Репозиторий не скачивался и не сканировался.",
    "findings": [
      {
        "name": "Демонстрационная находка",
        "severity": "low",
        "filePath": "demo/Example.cs",
        "line": 12
      }
    ]
  },
  "error": null
}
```

Для удобной проверки очереди:

```sh
dotnet run --project src/VulnerabilityTracker.Api -- --Scanning:MaxConcurrentScans=1 --Scanning:SimulationDelaySeconds=30
```

Для проверки ошибки сканирования:

```sh
dotnet run --project src/VulnerabilityTracker.Api -- --Scanning:SimulateFailure=true --Scanning:SimulationDelaySeconds=1
```

Еще доступно, например `Scanning__MaxConcurrentScans=1`.

## Пример работы

Сначала запустить сервис, затем откройте второй терминал:

```powershell
$base = 'http://localhost:5080'

# Создание и получение уязвимости.
$body = @{ name = 'SQL-инъекция'; severity = 'high' } | ConvertTo-Json
$vulnerability = Invoke-RestMethod -Method Post -Uri "$base/api/vulnerabilities" -ContentType 'application/json; charset=utf-8' -Body $body
Invoke-RestMethod -Uri "$base/api/vulnerabilities/$($vulnerability.id)"

# Постановка сканирования в очередь и проверка статуса.
$body = @{ repositoryUrl = 'https://gitlab.com/example/project' } | ConvertTo-Json
$scan = Invoke-RestMethod -Method Post -Uri "$base/api/scans" -ContentType 'application/json; charset=utf-8' -Body $body
Invoke-RestMethod -Uri "$base/api/scans/$($scan.id)" | ConvertTo-Json -Depth 6
```

Для проверки очереди запустить сервис как показано выше, затем:

```powershell
$body = @{ repositoryUrl = 'https://gitlab.com/example/project' } | ConvertTo-Json
$first = Invoke-RestMethod -Method Post -Uri "$base/api/scans" -ContentType 'application/json; charset=utf-8' -Body $body
$second = Invoke-RestMethod -Method Post -Uri "$base/api/scans" -ContentType 'application/json; charset=utf-8' -Body $body
Invoke-RestMethod -Uri "$base/api/scans/$($first.id)"
Invoke-RestMethod -Uri "$base/api/scans/$($second.id)"
```

## Структура и хранение данных

| Путь | Содержание |
| --- | --- |
| `src/VulnerabilityTracker.Api/Program.cs` | Регистрация зависимостей и HTTP-конвейер |
| `Controllers/` внутри API-проекта | HTTP-эндпоинты |
| `Contracts/` | Входные данные и валидация |
| `Models/` | Уязвимость, статусы, снимок задачи и результат |
| `Storage/VulnerabilityStore.cs` | Хранилище уязвимостей в `ConcurrentDictionary` |
| `Scanning/ScanService.cs` | Очередь, фиксированное число обработчиков и статусы |
| `Scanning/SimulatedCodeScanner.cs` | Имитация сканирования |
| `Infrastructure/` | Логирование запросов и обработка исключений |
| `tests/VulnerabilityTracker.Tests/` | Юнит-тесты |

## Тесты

```sh
dotnet test
```

Проверяются валидация, создание и чтение уязвимостей, конкурентное сохранение, ответы контроллеров, логирование и безопасный ответ при исключении, переходы статусов, результат и ошибка сканирования, предел параллельности, остановка сервиса и сама имитация.

Тесты не используют сеть и настоящий репозиторий. Для управления выполнением сканирования применяется тестовая реализация `ICodeScanner`.
