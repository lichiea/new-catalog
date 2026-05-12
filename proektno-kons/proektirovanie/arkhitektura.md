---
order: 1.5
title: Архитектура ПО
---

Схема архитектуры

<drawio path="./arkhitektura.svg" width="211px" height="101px"/>

Таблица модулей и реализующих их файлов

<table header="row">
<colgroup><col width="243"/><col width="346"/></colgroup>
<tr>
<td>

Модуль

</td>
<td>

Файлы

</td>
</tr>
<tr>
<td>

Модуль управления приложением (МУП)

</td>
<td>

Service/TestOrchestrator.cs,

Service/ITestOrchestrator.cs

</td>
</tr>
<tr>
<td>

Модуль управления тестами (МУТ)

</td>
<td>



</td>
</tr>
<tr>
<td>

Модуль нагрузочного тестирования (МНТ)

</td>
<td>



</td>
</tr>
<tr>
<td>

Модуль тестирования безопасности (МТБ)

</td>
<td>

Service/SecurityTestService.cs,

Service/ISecurityTestService.cs

</td>
</tr>
<tr>
<td>

Модуль взаимодействия с базой данных (МВБД)

</td>
<td>



</td>
</tr>
<tr>
<td>

Модуль обработки данных пользователя (МОДП)

</td>
<td>

Service/DataImportService.cs,

Service/IDataImportService.cs,

Models/ApiSec.cs

</td>
</tr>
<tr>
<td>

Модуль аутентификации (МА)

</td>
<td>

Service/AuthService.cs,

Service/IAuthService.cs

</td>
</tr>
<tr>
<td>

Модуль формирования отчетных материалов (МФОМ)

</td>
<td>

Service/ReportGenerator.cs,

Service/IReportGenerator.cs

</td>
</tr>
</table>

Таблица взаимодействия модулей

<drawio path="./arkhitektura-2.svg" width="211px" height="101px"/>

## Описание модулей

-  <highlight color="peony">Модуль управления приложением (МУП).</highlight> Все модули работают под управлением оркестратора



-  <highlight color="peony">Модуль обработки данных пользователя (МОДП).</highlight>

Вход: путь к файлу `.yaml`/`.json` (OpenAPI 3.x)

Выход: объект `ApiSpec` (список эндпоинтов, параметры, схемы)

Алгоритм:

```
1. Загрузить файл в строку.
2. Определить формат (JSON или YAML) по расширению или содержанию.
3. Использовать Microsoft.OpenApi.Readers.OpenApiDocumentReader.
4. Прочитать в OpenApiDocument.
5. Пройти по всем путям (Paths):
   Для каждого PathItem и Operation (GET, POST, ...):
     - Извлечь operationId (или сгенерировать: method+path)
     - Сохранить список параметров (path, query, header, cookie)
     - Для requestBody: если есть схема, запомнить content-type и пример тела (генерируем по схеме)
     - Для responses: запомнить ожидаемые статусы (200, 201, 400, 401, 403, 500...)
6. Построить и вернуть ApiSpec.
```



-  <highlight color="peony">Модуль аутентификации (МА).</highlight>

Вход: тип аутентификации (Basic/Bearer/None) + данные (логин/пароль или токен)

Выход: `AuthenticationHeaderValue`, готовый для вставки в `HttpClient`.

Алгоритм:

```
1. Если тип = Basic:
   - Конвертировать "username:password" в Base64
   - Сформировать заголовок "Authorization: Basic {base64}"
2. Если тип = Bearer:
   - Заголовок "Authorization: Bearer {token}"
3. Если тип = None:
   - Не добавлять заголовок.
4. Сохранить заголовок в поле класса.
```



-  <highlight color="peony">Модуль управления тестами (МУТ).</highlight>

Вход: `ApiSpec`, настройки (какие тесты запускать)

Выход: `TestSuiteResult` (содержит результаты МТБ и, в будущем, МНТ)

Алгоритм:

```
1. Создать пустой TestSuiteResult.
2. Если включён режим безопасности:
   - Вызвать ISecurityTestService.RunScan(spec, authHeaders)
   - Результат (список уязвимостей) добавить в result.Vulnerabilities
3. Если включён нагрузочный тест (опционально):
   - Вызвать ILoadTestService.RunLoadTest(spec, authHeaders, нагрузочные параметры)
   - Результат добавить в result.LoadMetrics
4. Записать общий вердикт: если есть уязвимости Critical/High → FAILED, иначе PASSED/WARNING.
5. Вернуть result.
```



-  <highlight color="peony">Модуль нагрузочного тестирования (МНТ).</highlight> 

Алгоритм:

```
1. Параметры: число виртуальных пользователей (VU), длительность (сек), время разгона (ramp-up).
2. Для каждого VU в отдельной задаче:
   - Цикл до окончания времени теста:
     - Выбрать случайный эндпоинт (или заданный сценарий)
     - Отправить запрос, замерить время ответа
     - Обновить глобальные счётчики (Interlocked)
     - Подождать think time
3. Собрать статистику: общее число запросов, ошибки, p50, p95, p99.
4. Сохранить таймсерии (запросов в секунду) для графика.
```



-  <highlight color="peony">Модуль тестирования безопасности (МТБ).</highlight> МТБ должен:

   -  Генерировать варианты запросов с вредоносными payload'ами (SQLi, XSS, Path Traversal, ...)

   -  Проверять ответы на признаки уязвимости

   -  Искать утечки PII (конфиденциальные данные)

   -  Проверять заголовки безопасности и SSL/TLS

   **Общий поток МТБ**

   ```
   1. Принять на вход ApiSpec, IAuthService, словари payload'ов, настройки сканирования.
   2. Создать список TestTask – для каждого эндпоинта и каждого типа атаки.
   3. Для каждого TestTask (можно параллельно, ограничивая степень параллелизма):
      - Сконструировать HttpRequestMessage с корректными параметрами (базовый запрос)
      - Создать мутированные версии запроса (подстановка payload'ов в параметры/тело/URL)
      - Выполнить запросы к API (с использованием одного HttpClient с заголовками аутентификации)
      - Анализировать ответ (статус, тело, заголовки)
      - При обнаружении уязвимости записать Vulnerability (тип, эндпоинт, payload, evidence)
   4. Отдельно выполнить проверку TLS/заголовков (без мутаций, просто анализ конфигурации).
   5. Вернуть список найденных уязвимостей.
   ```

   **Генерация вредоносных запросов (фаззинг)**

   Для каждого параметра (path, query, header, body) и для каждого типа payload применяется мутация.

   **Типы атак и соответствующие payload'ы:**

   | **Тип** | **Пример payload** | **Куда вставляется** | **Признак уязвимости** |
   |---------|--------------------|----------------------|------------------------|
   |         |                    |                      | SQLi                   | `' OR '1'='1` |
   |         |                    |                      | XSS (Reflected)        | `<script>alert(1)</script>` |
   |         |                    |                      | Path Traversal         | `../../../etc/passwd` |
   |         |                    |                      | NoSQL Injection        | `{"$gt": ""}` |
   |         |                    |                      | XXE                    | `<?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM "`[`file:///etc/passwd">]><root>&test;</root>`](file:///etc/passwd">]><root>&test;</root>) |
   |         |                    |                      | SSTI                   | `{{7*7}}` |
   |         |                    |                      | Long string            | `A`\*100000 |
   |         |                    |                      | Null byte injection    | `%00` |

   **Алгоритм для одного эндпоинта:**

   ```
   Для каждого параметра (in: query, path, header, body):
     Для каждого типа атаки:
       - Взять базовый корректный запрос.
       - Заменить значение параметра на payload (для body – вставить в соответствующее поле JSON/XML).
       - Отправить запрос.
       - Если статус ответа 5xx → зафиксировать как потенциальный сбой (может быть DoS).
       - Запустить анализатор ответа.
   ```

   **Анализ ответа на уязвимости**

   **Для SQLi:**

   -  Искать в теле ответа и в заголовках ошибок подстроки: `SQL syntax`, `mysql_fetch`, `ORA-`, `PostgreSQL`, `Unclosed quotation mark`, `Microsoft OLE DB`.

   -  Если есть задержка ответа > ожидаемой (time-based) – тоже признак.

   **Для XSS:**

   -  Отправить payload, который содержит уникальный маркер, например `xsstest123<script>alert(1)</script>`

   -  Проверить, встречается ли этот маркер в теле ответа **неэкранированным** (просто подстрока, а не `&lt;script&gt;`).

   -  Можно также проверить, что ответ имеет Content-Type text/html и маркер вставлен в DOM.

   **Для PII (конфиденциальные данные):**

   -  **Не мутируем запросы**, а анализируем **все ответы** (включая нормальные) на наличие паттернов:

      -  Email: `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b`

      -  Номер кредитной карты: `\b(?:\d[ -]*?){13,16}\b`

      -  Российский паспорт: `\d{4} \d{6}`

      -  Телефон: `\+?\d[\d\(\)\s-]{9,}\d`

   -  Если паттерн найден – записать как уязвимость типа "Утечка PII".

   **Проверка заголовков безопасности**

   **Для данного эндпоинта (или для корневого):**

   -  Отправить GET-запрос на `/` или на любой существующий эндпоинт.

   -  Проверить наличие и корректность заголовков:

      -  `Strict-Transport-Security` – должен быть с max-age ≥ 31536000

      -  `Content-Security-Policy` – наличие базовых директив

      -  `X-Content-Type-Options: nosniff`

      -  `X-Frame-Options: DENY` или `SAMEORIGIN`

      -  `Referrer-Policy: no-referrer` / strict-origin-when-cross-origin

      -  `Permissions-Policy` – наличие

   -  Отсутствие или неправильное значение помечать как уязвимость низкой/средней критичности.

   **3\.2.5. Проверка TLS**

   **Алгоритм:**

   -  Использовать [`System.Net.Security`](http://System.Net.Security)`.SslStream` для установления соединения с портом 443.

   -  Вручную перебрать протоколы: SslProtocols.Tls12, Tls13.

   -  Проверить сертификат: дата валидности, CommonName.

   -  Если TLS 1.0 или 1.1 разрешён – уязвимость средней критичности.

   -  Если сертификат самоподписанный – предупреждение.

   **Упрощение в MVP:** можно вызвать внешнюю утилиту `openssl s_client` и распарсить вывод, но лучше использовать встроенные средства `SslStream`.

   **3\.2.6. Формат выходных данных (Vulnerability)**

   ```
   public class Vulnerability
   {
       public string Id { get; set; }          // GUID
       public string Type { get; set; }        // SQLi, XSS, PII, MissingHeader, WeakTLS...
       public string Severity { get; set; }    // Critical, High, Medium, Low
       public string Endpoint { get; set; }    // GET /api/users?id=...
       public string Payload { get; set; }     // Вредоносная строка (если применимо)
       public string Evidence { get; set; }    // Сниппет ответа или заголовка
       public DateTime DetectedAt { get; set; }
   }
   ```

-  Модуль формирования отчетных материалов (МФОМ)

Вход: `TestSuiteResult` (содержит список уязвимостей, метрики, вердикт)

Выход: HTML-файл

Алгоритм (HTML):

```
1. Загрузить шаблон HTML (внедрённый ресурс или файл).
2. Заполнить динамические секции:
   - Общий вердикт (PRODUCTION_READY / HAS_WARNINGS / FAILED)
   - Таблица уязвимостей (тип, severity, endpoint, payload, evidence)
   - Таблица проверенных заголовков безопасности (заголовок, присутствует, корректно)
   - Список проверенных эндпоинтов с количеством тестов
3. Сгенерировать cURL команды для каждой уязвимости (воспроизведение).
4. Сохранить HTML в файл.
```