# MoneyBuddy — Spécification & Plan de développement (.NET MAUI, XAML + MVVM)

> Version: 1.0 — Utilisation libre dans ton repo GitHub. Ce document consolide: objectifs, architecture, modèle de données, UX, checklist du cahier des charges, feuille de route “feature/fix”, et CI/CD + mises à jour.

---

## 1) Objectif & périmètre
**But**: App .NET MAUI (Android, iOS, Windows) pour la gestion **mois par mois** des finances perso.
- Saisie **entrées/sorties**
- **Abonnements** (récurrence, auto‑post)
- **Prêts** (échéancier, restant dû)
- **Budgets** par catégorie
- **Dashboard** (KPI + graphiques)
- **Rapports** (catégories, tendance)
- **Notifications** (rappel quotidien, échéances)
- **Export/Import** (CSV/JSON, V1.3+)

**Hors périmètre V1**: cloud sync, comptes multiples (backlog).

---

## 2) Architecture (MVVM, sans code‑behind)
**Pattern**: MVVM + DI (CommunityToolkit.MVVM).  
**Projets** (suggestion multi‑libs):
```
MoneyBuddy.sln
  ├─ MoneyBuddy.Core/   (modèles, règles métier, services abstraits)
  ├─ MoneyBuddy.Data/   (SQLite + impl services)
  ├─ MoneyBuddy.Maui/   (UI XAML, VMs, Shell, notifs, charts)
  └─ MoneyBuddy.Tests/
```
**Dossiers (UI)**:
- `Models/` – POCOs: Transaction, Category, Subscription, Loan, Budget, Account, Settings
- `Data/` – `AppDatabase` (sqlite-net-pcl), seed
- `Services/` – `ITransactionService`, `ISubscriptionService`, `ILoanService`, `IBudgetService`, `IReportService`, `INotificationScheduler`, `IBackupService`
- `ViewModels/` – `DashboardVM`, `TransactionsVM`, `AddTransactionVM`, `SubscriptionsVM`, `LoansVM`, `BudgetsVM`, `ReportsVM`, `SettingsVM`
- `Views/` – pages XAML correspondantes
- `Resources/` – Styles, MergedDictionaries, icônes
- `Converters/` – formats monnaie, dates…
- `Platforms/*` – capteurs/permissions spécifiques

**Packages NuGet (OSS)**
- `CommunityToolkit.MVVM` / `CommunityToolkit.Maui`
- `sqlite-net-pcl`
- `Microcharts.Maui` + `SkiaSharp.Views.Maui.Controls`
- `Plugin.LocalNotification`

**MauiProgram.cs (extrait)**
```csharp
builder.UseMauiApp<App>()
       .UseMauiCommunityToolkit()
       .UseLocalNotification()
       .UseSkiaSharp();

var dbPath = Path.Combine(FileSystem.AppDataDirectory, "moneybuddy.db3");
builder.Services.AddSingleton(new AppDatabase(dbPath));
builder.Services.AddSingleton<ITransactionService, TransactionService>();

builder.Services.AddTransient<DashboardViewModel>();
builder.Services.AddTransient<DashboardPage>();
```

**Navigation (Shell)**
```xml
<TabBar>
  <ShellContent Title="Tableau de bord" ContentTemplate="{DataTemplate views:DashboardPage}" />
  <ShellContent Title="Mouvements"     ContentTemplate="{DataTemplate views:TransactionsPage}" />
  <ShellContent Title="Abonnements"    ContentTemplate="{DataTemplate views:SubscriptionsPage}" />
  <ShellContent Title="Prêts"          ContentTemplate="{DataTemplate views:LoansPage}" />
  <ShellContent Title="Rapports"       ContentTemplate="{DataTemplate views:ReportsPage}" />
  <ShellContent Title="Réglages"       ContentTemplate="{DataTemplate views:SettingsPage}" />
</TabBar>
```

---

## 3) Modèle de données (SQLite)
```csharp
public enum TransactionType { Income, Expense, Transfer }
public enum Recurrence { None, Daily, Weekly, Monthly, Quarterly, Yearly }

public class Category {
  [PrimaryKey, AutoIncrement] public int Id { get; set; }
  public string Name { get; set; }
  public string ColorHex { get; set; }
  public TransactionType Type { get; set; } // Expense/Income
}

public class Transaction {
  [PrimaryKey, AutoIncrement] public int Id { get; set; }
  public DateTime Date { get; set; }
  public decimal Amount { get; set; }
  public TransactionType Type { get; set; }
  public int CategoryId { get; set; }
  public int? AccountId { get; set; }
  public int? SubscriptionId { get; set; }
  public string Note { get; set; }
  public string Tags { get; set; } // "courses,carte"
}

public class Subscription {
  [PrimaryKey, AutoIncrement] public int Id { get; set; }
  public string Name { get; set; }
  public decimal Amount { get; set; }
  public int CategoryId { get; set; }
  public Recurrence Recurrence { get; set; } = Recurrence.Monthly;
  public DateTime NextDueDate { get; set; }
  public bool AutoPost { get; set; } = true;
}

public class Loan {
  [PrimaryKey, AutoIncrement] public int Id { get; set; }
  public string Name { get; set; }
  public decimal Principal { get; set; }
  public double InterestRate { get; set; } // % annuel
  public decimal MonthlyPayment { get; set; }
  public DateTime StartDate { get; set; }
  public DateTime? EndDate { get; set; }
}

public class Budget {
  [PrimaryKey, AutoIncrement] public int Id { get; set; }
  public int Year { get; set; }
  public int Month { get; set; }
  public int CategoryId { get; set; }
  public decimal LimitAmount { get; set; }
}

public class Settings {
  [PrimaryKey] public int Id { get; set; } = 1;
  public int ReminderHour { get; set; } = 21;
  public string Currency { get; set; } = "EUR";
  public bool NotificationsEnabled { get; set; } = true;
}
```

**AppDatabase (extrait)**
```csharp
public class AppDatabase {
  readonly SQLiteAsyncConnection _db;
  public AppDatabase(string path) =>
    _db = new SQLiteAsyncConnection(path, SQLiteOpenFlags.Create|SQLiteOpenFlags.ReadWrite|SQLiteOpenFlags.SharedCache);

  public async Task InitAsync() {
    await _db.CreateTableAsync<Category>();
    await _db.CreateTableAsync<Transaction>();
    await _db.CreateTableAsync<Subscription>();
    await _db.CreateTableAsync<Loan>();
    await _db.CreateTableAsync<Budget>();
    await _db.CreateTableAsync<Settings>();

    if (await _db.Table<Category>().CountAsync() == 0) {
      await _db.InsertAllAsync(new[]{
        new Category { Name="Loisir", ColorHex="#9b59b6", Type=TransactionType.Expense },
        new Category { Name="Courses", ColorHex="#27ae60", Type=TransactionType.Expense },
        new Category { Name="Bien-être", ColorHex="#e67e22", Type=TransactionType.Expense },
        new Category { Name="Salaire", ColorHex="#2980b9", Type=TransactionType.Income },
        new Category { Name="Autre revenu", ColorHex="#16a085", Type=TransactionType.Income }
      });
    }
    if (await _db.Table<Settings>().CountAsync() == 0) {
      await _db.InsertAsync(new Settings());
    }
  }

  public Task[List[Transaction]] GetMonthTransactionsAsync(int year,int month) =>
    _db.Table[Transaction]().Where(t => t.Date.Year==year && t.Date.Month==month).ToListAsync();

  public Task<int> InsertAsync<T>(T item) where T:new() => _db.InsertAsync(item);
  public Task<int> UpdateAsync<T>(T item) where T:new() => _db.UpdateAsync(item);
  public Task<int> DeleteAsync<T>(T item) where T:new() => _db.DeleteAsync(item);
}
```
*(Note: adapte la syntaxe C# si tu copies directement — ici le markdown peut casser certains `[]`.)*

**Services (extraits)**
```csharp
public interface ITransactionService {
  Task AddAsync(Transaction t);
  Task<IReadOnlyList<Transaction>> GetMonthAsync(int year,int month);
  Task<(decimal income, decimal expense, decimal savings)> SummaryAsync(int year,int month);
}

public sealed class TransactionService : ITransactionService {
  private readonly AppDatabase _db;
  public TransactionService(AppDatabase db) => _db = db;
  public Task AddAsync(Transaction t) => _db.InsertAsync(t);
  public Task<IReadOnlyList<Transaction>> GetMonthAsync(int y,int m) => _db.GetMonthTransactionsAsync(y,m);
  public async Task<(decimal,decimal,decimal)> SummaryAsync(int y,int m) {
    var items = await _db.GetMonthTransactionsAsync(y,m);
    var inc = items.Where(i=>i.Type==TransactionType.Income).Sum(i=>i.Amount);
    var exp = items.Where(i=>i.Type==TransactionType.Expense).Sum(i=>i.Amount);
    return (inc, exp, inc - exp);
  }
}
```

**Abonnements: auto‑post**: si `NextDueDate <= Today` → créer `Transaction` + recaler `NextDueDate` selon `Recurrence`.

**Notifications**
```csharp
public interface INotificationScheduler {
  Task ScheduleDailyReminderAsync(int hourLocal);
  Task CancelAllAsync();
}
```
Android/iOS: `Plugin.LocalNotification` (Repeat Daily) — Windows: toast via Windows App SDK.

---

## 4) UX (priorités & patterns)
- Au-dessus de la ligne: Épargne (objectif vs réalisé), Reste à vivre, Dépenses & Revenus, Alertes, Actions rapides, À venir.
- Graphes: Donut (répartition), Barres (jour), ProgressBar (budgets).
- Micro-copy FR/EUR; dates `dd MMM`; alertes courtes + CTA.
- Accessibilité: contraste AA, tailles min 14 pt, zones tactiles 44×44, clavier desktop.

---

## 5) Checklist — Cahier des charges
- Cadrage (plateformes, FR/EUR), Tech (MVVM, Shell, SQLite, Notifs, Charts)
- Données (Transactions/Categories/Subscriptions/Loans/Budgets/Settings)
- Écrans (Dashboard, Mouvements, Abos, Prêts, Budgets, Rapports, Réglages)
- Parcours (QuickAdd, deeplink notif, Snackbar + Undo)
- Notifications (quotidien/hebdo/mensuel, veille abo)
- Règles (Épargne, Reste à vivre, Δ vs M‑1, alertes 80/100%)
- Accessibilité/Responsive, Export/Import (V1.3+), Logs, Diagnostics, Tests

---

## 6) Roadmap — Branches feature/fix (ordre)
1. `feat/bootstrap-project`  2. `feat/models-and-db`  3. `feat/transactions-crud`
4. `feat/dashboard-kpi`     5. `feat/quickadd-sheet` 6. `feat/budgets-envelopes`
7. `feat/alerts`            8. `feat/subscriptions`  9. `feat/loans-amortization`
10. `feat/reports-monthly`  11. `feat/notifications` 12. `feat/settings`
13. `feat/update-check`     14. `feat/ci-pipelines`  15. `feat/windows-appinstaller` (option)

Definition of Done: tests service + scénario UI, logs, accessibilité, doc/screenshot.

---

## 7) CI/CD & mises à jour
- GitHub Actions: tags `vX.Y.Z` → AAB signé + MSIX en Release.
- Windows hors Store: `.appinstaller` (check OnLaunch).
- Vérif Release: GitHub `releases/latest` → bannière “nouvelle version”.

---

## 8) Dépendances (OSS)
- CommunityToolkit.MVVM, CommunityToolkit.Maui, sqlite-net-pcl
- Microcharts.Maui, SkiaSharp.Views.Maui.Controls
- Plugin.LocalNotification

---

## 9) Quickstart
```bash
dotnet new maui -n MoneyBuddy
cd MoneyBuddy
dotnet add package CommunityToolkit.MVVM
dotnet add package CommunityToolkit.Maui
dotnet add package sqlite-net-pcl
dotnet add package Microcharts.Maui
dotnet add package SkiaSharp.Views.Maui.Controls
dotnet add package Plugin.LocalNotification
```

_Fin._
