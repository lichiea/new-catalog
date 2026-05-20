---
order: 1
title: Описание разрабатываемых модулей
---

Программная система ProofAPI представляет собой приложение, компоненты которого взаимодействуют через чётко определённые интерфейсы. Центральным звеном, координирующим весь цикл тестирования, выступает модуль управления приложением (МУП), реализованный классом TestOrchestrator. Получив от пользовательского интерфейса объект ApiSpec, содержащий все эндпоинты, параметры и схемы данных тестируемого API, а также конфигурацию аутентификации и критерии проверок, оркестратор запускает параллельное выполнение нагрузочного тестирования и тестирования безопасности. Для этого он обращается к интерфейсам ILoadTestService и ISecurityTestService, которые после завершения возвращают соответственно объект LoadTestMetric и список Vulnerability. Код фрагмента класса TestOrchestrator представлен листингом 1.

Листинг 1 - Фрагмента кода класса TestOrchestrator

```c#
public class TestOrchestrator : ITestOrchestrator
{
private readonly ILoadTestService _load;
private readonly ISecurityTestService _security;
private readonly ITestManager _manager;
private readonly IDatabaseService _db;
private readonly IReportGenerator _report;
 
public TestOrchestrator(ILoadTestService load, ISecurityTestService security, ITestManager manager,
IDatabaseService db, IReportGenerator report)
{
_load = load;
_security = security;
_manager = manager;
_db = db;
_report = report;
}
 
public async Task<TestSuiteResult> RunAllTestsAsync(ApiSpec spec, int virtualUsers, int durationSeconds)
{
var startedAt = DateTime.UtcNow;
var loadTask = _load.RunLoadTestAsync(spec, virtualUsers, durationSeconds);
var securityTask = _security.RunSecurityScanAsync(spec);
 
await Task.WhenAll(loadTask, securityTask);
 
var loadResult = await loadTask;
var vulns = await securityTask;
 
var result = _manager.GenerateResult(loadResult, vulns, startedAt);
 
await _db.SaveTestRunAsync(spec.Title ?? "Unnamed", result);
await _report.GenerateHtmlReportAsync(result);
 
return result;
}
}
```

 

Далее оркестратор передаёт полученные сырые результаты модулю управления тестами (МУТ) – компоненту TestManager, который анализирует метрики производительности и состав обнаруженных уязвимостей, формирует общий вердикт (PASSED, WARNING или FAILED) и агрегирует данные в итоговый объект TestSuiteResult. После этого оркестратор последовательно вызывает модуль формирования отчётных материалов (МФОМ) для генерации HTML-отчёта и модуль взаимодействия с базой данных (МВБД) для сохранения результатов в локальное хранилище SQLite.

Модуль нагрузочного тестирования (МНТ) предназначен для оценки производительности API в условиях одновременной работы множества виртуальных пользователей. В классе LoadTestService реализован алгоритм, имитирующий поведение реальных клиентов. Основными входными параметрами являются число виртуальных пользователей, продолжительность теста, время разгона и набор эндпоинтов. Для каждого виртуального пользователя создаётся отдельная асинхронная задача, внутри которой в цикле до истечения заданного периода выполняются случайные запросы к тестируемому API. Каждый запрос обрабатывается через экземпляр HttpClient, при этом с помощью Stopwatch замеряется время отклика. По завершении всех виртуальных пользователей производится расчёт средней задержки, а также подсчёт общего числа запросов и ошибок. Для повышения точности нагрузочного профиля в сервис интегрирована поддержка разгона, когда виртуальный пользователь добавляются постепенно, что позволяет избежать одномоментного шока для тестируемой системы. Метрики возвращаются в виде структуры LoadTestMetric, которая затем сохраняется в JSON-поле базы данных и визуализируется в отчёте. Код фрагмента класса LoadTestService представлен в приложении А.

Модуль тестирования безопасности (МТБ) – наиболее сложный компонент системы, предназначенный для активного поиска уязвимостей. Класс SecurityTestService реализует два основных режима работы: пассивные проверки (анализ заголовков безопасности, поиск утечек конфиденциальных данных) и активный фаззинг с подстановкой вредоносных нагрузок в параметры запросов. Алгоритм работы модуля представлен на рисунке 2.