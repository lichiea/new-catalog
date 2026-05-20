---
order: 2
title: Использование в программном обеспечении
---

Для работы с описанной моделью создается класс AppDbContext, наследующий DbContext. Код фрагмента программы представлен листингом 1.

Листинг 1 - Фрагмент кода класса AppDbContext

```c#
public class AppDbContext : DbContext
{
public DbSet<Project> Projects { get; set; }
public DbSet<TestRun> TestRuns { get; set; }
public DbSet<VulnerabilityEntity> Vulnerabilities { get; set; }
public DbSet<SecurityHeadersResult> SecurityHeadersResults { get; set; }
 
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
// Путь к файлу БД в локальной папке приложения
var dbPath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "ProofAPI.db");
optionsBuilder.UseSqlite($"Data Source={dbPath}");
}
 
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
// Настройка связей и индексов
modelBuilder.Entity<TestRun>()
.HasOne<Project>()
.WithMany()
.HasForeignKey(tr => tr.ProjectId)
.OnDelete(DeleteBehavior.Cascade);
 
modelBuilder.Entity<VulnerabilityEntity>()
.HasOne<TestRun>()
.WithMany()
.HasForeignKey(v => v.TestRunId)
.OnDelete(DeleteBehavior.Cascade);
 
modelBuilder.Entity<SecurityHeadersResult>()
.HasOne<TestRun>()
.WithMany()
.HasForeignKey(h => h.TestRunId)
.OnDelete(DeleteBehavior.Cascade);
}
}
```

 Использование миграций EF Core позволяет автоматически создавать и обновлять схему базы данных при изменении модели.

Сервис DatabaseService реализует интерфейс IDatabaseService и предоставляет методы:

-  SaveTestRunAsync(Project project, TestSuiteResult result) – сохраняет новый запуск со всеми вложенными уязвимостями и результатами проверки заголовков;

-  GetProjectHistoryAsync(int projectId) – возвращает список всех запусков проекта для отображения и сравнения;

-  GetTestRunDetailAsync(int testRunId) – загружает полные данные запуска для повторной генерации отчета.

Такой подход обеспечивает надежное локальное хранение с возможностью версионирования и сравнения, как того требует задание на ВКР.