# Техническая документация: Program.cs

## Обзор
`Program.cs` является точкой входа ASP.NET Core приложения Innohoot. Файл конфигурирует веб-приложение, регистрирует сервисы и настраивает middleware pipeline.

## Версия
- **Стиль**: Minimal Hosting Model (ASP.NET Core 6.0+)
- **Язык**: C# (.NET)

---

## Зависимости

### Используемые пространства имен
```csharp
using Innohoot.DataLayer;
using Innohoot.DataLayer.Services.Implementations;
using Innohoot.DataLayer.Services.Interfaces;
using Microsoft.EntityFrameworkCore;
using Newtonsoft.Json;
```

### Внешние пакеты
- **Entity Framework Core** - ORM для работы с PostgreSQL
- **Newtonsoft.Json** - сериализация/десериализация JSON
- **AutoMapper** - маппинг объектов
- **SignalR** - real-time коммуникация
- **Swagger** - документация API

---

## Архитектура приложения

### 1. Инициализация приложения (строки 9-11)

```csharp
var MyAllowSpecificOrigins = "_myAllowSpecificOrigins";
var builder = WebApplication.CreateBuilder(args);
```

**Описание**: 
- Создается экземпляр `WebApplicationBuilder` для конфигурации приложения
- Объявлена переменная для CORS политики (не используется в текущей версии)

---

### 2. Конфигурация сервисов

#### 2.1 CORS политика (строки 13-19)

```csharp
builder.Services.AddCors(options => options.AddPolicy("CorsPolicy", b =>
{
    b.AllowAnyOrigin()
        .AllowAnyMethod()
        .AllowAnyHeader();
})
);
```

**Назначение**: Настройка Cross-Origin Resource Sharing
- **Политика**: "CorsPolicy"
- **Разрешения**:
  - Любые источники (origins)
  - Любые HTTP методы (GET, POST, PUT, DELETE и т.д.)
  - Любые заголовки

⚠️ **Предупреждение безопасности**: Конфигурация `AllowAnyOrigin()` небезопасна для production окружения. Рекомендуется ограничить список разрешенных источников.

**Закомментированный код** (строки 22-27): Альтернативная CORS конфигурация (не используется).

---

#### 2.2 SignalR (строка 30)

```csharp
builder.Services.AddSignalR();
```

**Назначение**: Подключение SignalR для WebSocket коммуникации в реальном времени.

**Применение**: Используется для синхронизации данных между клиентами (например, обновление результатов голосования в реальном времени).

---

#### 2.3 Controllers и сериализация JSON (строки 32-36)

```csharp
builder.Services.AddControllers().AddNewtonsoftJson(x =>
{
    x.SerializerSettings.ReferenceLoopHandling = ReferenceLoopHandling.Ignore;
}).AddJsonOptions(options =>
    options.JsonSerializerOptions.PropertyNamingPolicy = null);
```

**Конфигурация**:
1. **AddControllers()** - регистрация MVC контроллеров
2. **AddNewtonsoftJson()** - использование Newtonsoft.Json вместо System.Text.Json
   - `ReferenceLoopHandling.Ignore` - игнорирование циклических ссылок при сериализации
3. **AddJsonOptions()** - сохранение оригинального регистра имен свойств (без camelCase преобразования)

**Закомментированный код** (строки 38-45): Альтернативные варианты обработки циклических ссылок.

---

#### 2.4 Swagger документация (строка 47)

```csharp
builder.Services.AddSwaggerGen();
```

**Назначение**: Генерация OpenAPI (Swagger) документации для REST API.

**Доступ**: `/swagger` endpoint (только в non-development режиме согласно конфигурации middleware).

---

#### 2.5 Repository Pattern (строка 49)

```csharp
builder.Services.AddScoped<IDBRepository, DBRepository>();
```

**Lifetime**: Scoped (создается один экземпляр на HTTP запрос)

**Назначение**: Базовый репозиторий для работы с базой данных.

---

#### 2.6 Domain сервисы (строки 52-57)

```csharp
builder.Services.AddTransient<IPollService, PollService>();
builder.Services.AddTransient<ISessionService, SessionService>();
builder.Services.AddTransient<IUserService, UserService>();
builder.Services.AddTransient<IVoteRecordService, VoteRecordService>();
builder.Services.AddTransient<IOptionService, OptionService>();
builder.Services.AddTransient<IPollCollectionService, PollCollectionService>();
```

**Lifetime**: Transient (создается новый экземпляр при каждом запросе)

**Зарегистрированные сервисы**:

| Интерфейс | Реализация | Назначение |
|-----------|------------|------------|
| `IPollService` | `PollService` | Управление опросами |
| `ISessionService` | `SessionService` | Управление сессиями голосования |
| `IUserService` | `UserService` | Управление пользователями |
| `IVoteRecordService` | `VoteRecordService` | Обработка записей голосов |
| `IOptionService` | `OptionService` | Управление вариантами ответов |
| `IPollCollectionService` | `PollCollectionService` | Управление коллекциями опросов |

---

#### 2.7 AutoMapper (строка 59)

```csharp
builder.Services.AddAutoMapper(typeof(AppMappingProfile));
```

**Назначение**: Регистрация профилей маппинга для автоматического преобразования между сущностями и DTO.

**Профиль**: `AppMappingProfile` (находится в папке `Profiles/`)

---

#### 2.8 База данных (строки 62-65)

```csharp
builder.Services.AddDbContext<ApplicationContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default"));
});
```

**Конфигурация**:
- **ORM**: Entity Framework Core
- **СУБД**: PostgreSQL (Npgsql провайдер)
- **Connection String**: Берется из конфигурации под ключом "Default"
- **Lifetime**: Scoped (по умолчанию для DbContext)

**Источники конфигурации**:
- `appsettings.json`
- `appsettings.Development.json`
- Переменные окружения

---

### 3. Создание приложения (строка 67)

```csharp
var app = builder.Build();
```

**Описание**: Компиляция конфигурации и создание экземпляра `WebApplication`.

---

### 4. Конфигурация HTTP Pipeline

#### 4.1 Environment-зависимая конфигурация (строки 70-76)

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

**Для Non-Development окружения**:
- **HSTS** (HTTP Strict Transport Security) - принудительное использование HTTPS
  - Срок действия: 30 дней (по умолчанию)
- **Swagger UI** - интерактивная документация API

⚠️ **Примечание**: Логика кажется инвертированной - обычно Swagger включается только в Development режиме.

---

#### 4.2 Middleware Pipeline (строки 78-82)

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors("CorsPolicy");
```

**Порядок выполнения**:
1. **UseHttpsRedirection** - перенаправление HTTP → HTTPS
2. **UseStaticFiles** - обслуживание статических файлов (React приложение)
3. **UseRouting** - определение endpoint'ов
4. **UseCors** - применение CORS политики "CorsPolicy"

---

#### 4.3 Routing конфигурация (строки 84-88)

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller}/{action=Index}/{id?}");

app.MapFallbackToFile("index.html");
```

**Маршрутизация**:
- **Паттерн**: `/{controller}/{action}/{id?}`
  - `action` по умолчанию: `Index`
  - `id` - опциональный параметр
- **Fallback**: Все неизвестные маршруты перенаправляются на `index.html` (для поддержки SPA клиент-стороннего роутинга React)

---

#### 4.4 Запуск приложения (строка 89)

```csharp
app.Run();
```

**Описание**: Запуск веб-сервера и начало прослушивания входящих HTTP запросов.

---

## Паттерны проектирования

### 1. Dependency Injection (DI)
Все сервисы регистрируются через встроенный DI контейнер ASP.NET Core.

### 2. Repository Pattern
Использование `IDBRepository` для абстрагирования работы с данными.

### 3. Service Layer Pattern
Бизнес-логика инкапсулирована в отдельных сервисах (PollService, SessionService и т.д.).

### 4. DTO Pattern
Использование AutoMapper для преобразования между доменными моделями и DTO.

---

## Рекомендации по улучшению

### Безопасность
1. ❌ **Ограничить CORS политику** - заменить `AllowAnyOrigin()` на конкретные домены
2. ❌ **Переместить Swagger в Development** - не показывать API документацию в production
3. ✅ **Добавить аутентификацию/авторизацию** - JWT или Identity

### Производительность
1. ✅ **Response Caching** - добавить кеширование для часто запрашиваемых данных
2. ✅ **Response Compression** - сжатие HTTP ответов

### Мониторинг
1. ✅ **Logging** - настроить структурированное логирование (Serilog)
2. ✅ **Health Checks** - добавить проверки состояния БД и сервисов

### Код
1. ✅ **Удалить закомментированный код** - очистить строки 22-27, 38-45
2. ✅ **Удалить неиспользуемую переменную** - `MyAllowSpecificOrigins` (строка 9)

---

## Конфигурационные файлы

### appsettings.json
Должен содержать:
```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=innohoot;Username=user;Password=pass"
  }
}
```

---

## Зависимости между компонентами

```
Program.cs
├── Controllers (7 контроллеров)
├── Services (6 сервисов)
│   └── IDBRepository
│       └── ApplicationContext (EF Core)
│           └── PostgreSQL
├── AutoMapper
│   └── AppMappingProfile
└── React SPA (ClientApp)
```

---

## Диагностика

### Проверка запуска
```bash
dotnet run --project Innohoot/Innohoot.csproj
```

### Типичные проблемы

| Проблема | Причина | Решение |
|----------|---------|---------|
| Connection refused | PostgreSQL не запущен | Запустить БД |
| CORS ошибка | Неправильная политика | Проверить origin клиента |
| 404 для API | Неверный routing | Проверить паттерн маршрута |
| Циклическая ссылка JSON | Навигационные свойства EF | Уже решено через ReferenceLoopHandling.Ignore |

---

## Версионирование
- **Создано**: 2022 (судя по миграциям)
- **Последнее обновление**: Неизвестно
- **Версия .NET**: 6.0+

---

## Авторы / Контакты
См. файл LICENSE и README.md в корне проекта.

---

## Лицензия
См. файл LICENSE в корне проекта.
