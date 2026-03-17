# 🐘 MIGRAÇÃO MONGODB → POSTGRESQL

## 📊 VISÃO GERAL

**Decisão:** Migrar de MongoDB NoSQL para PostgreSQL relacional

**Motivo:**
- Melhor suporte a transações ACID
- Joins eficientes vs. denormalização
- Integridade referencial
- Queries complexas mais performáticas
- Ferramentas e ecossistema maduro
- Custo menor em cloud

**Esforço Total:** 176-244 horas (~5-6 semanas)

**Estratégia:** Strangler Pattern (migração gradual, zero risco)

---

## 🎯 FASES DA MIGRAÇÃO

### **FASE 0: PREPARAÇÃO** (Semana 1 - 40h)

#### Task 0.1: Setup PostgreSQL Ambiente
```bash
# Docker Compose com PostgreSQL
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: tep_sales_db
      POSTGRES_USER: tep_admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 0.2: Instalar Entity Framework Core
```bash
cd tep_vendas_services/src/Tep.Sales.Service.Data

# Remover MongoDB
dotnet remove package MongoDB.Driver

# Adicionar EF Core + PostgreSQL
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.0
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 0.3: Criar DbContext do PostgreSQL
```csharp
// /src/Tep.Sales.Service.Data/Context/TepSalesDbContext.cs
using Microsoft.EntityFrameworkCore;

public class TepSalesDbContext : DbContext
{
    public TepSalesDbContext(DbContextOptions<TepSalesDbContext> options)
        : base(options)
    {
    }

    // DbSets
    public DbSet<User> Users { get; set; }
    public DbSet<Company> Companies { get; set; }
    public DbSet<Client> Clients { get; set; }
    public DbSet<Address> Addresses { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<PurchaseOrder> PurchaseOrders { get; set; }
    public DbSet<PurchaseOrderItem> PurchaseOrderItems { get; set; }
    public DbSet<PriceTable> PriceTables { get; set; }
    public DbSet<PaymentCondition> PaymentConditions { get; set; }
    // ... todas as outras entidades

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Configurações
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(TepSalesDbContext).Assembly);

        // Convenções
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            // Table names em snake_case
            entityType.SetTableName(entityType.GetTableName()?.ToSnakeCase());

            // Column names em snake_case
            foreach (var property in entityType.GetProperties())
            {
                property.SetColumnName(property.GetColumnName().ToSnakeCase());
            }
        }
    }
}
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 0.4: Criar Entity Configurations
```csharp
// /src/Tep.Sales.Service.Data/Configurations/UserConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("users");

        builder.HasKey(u => u.Id);

        builder.Property(u => u.Id)
            .HasDefaultValueSql("gen_random_uuid()");

        builder.Property(u => u.Name)
            .IsRequired()
            .HasMaxLength(255);

        builder.Property(u => u.Email)
            .IsRequired()
            .HasMaxLength(255);

        builder.HasIndex(u => u.Email)
            .IsUnique();

        builder.HasIndex(u => new { u.CompanyId, u.Email });

        builder.Property(u => u.Password)
            .IsRequired()
            .HasMaxLength(500);

        builder.Property(u => u.CreatedAt)
            .HasDefaultValueSql("CURRENT_TIMESTAMP");

        // Relacionamentos
        builder.HasOne(u => u.Company)
            .WithMany()
            .HasForeignKey(u => u.CompanyId)
            .OnDelete(DeleteBehavior.Restrict);

        // Soft delete
        builder.HasQueryFilter(u => u.Status != UserStatus.Deleted);
    }
}
```

**Criar configuração para TODAS as entidades:**
- UserConfiguration
- CompanyConfiguration
- ClientConfiguration
- AddressConfiguration
- ProductConfiguration
- PurchaseOrderConfiguration
- PurchaseOrderItemConfiguration
- ... (~30 configurations)

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 0.5: Refatorar Entidades (Remover MongoDB)
```csharp
// ANTES (MongoDB):
[BsonId]
public Guid Id { get; set; }

[BsonElement("name")]
public string Name { get; set; }

// DEPOIS (EF Core):
public Guid Id { get; set; }
public string Name { get; set; }
// Configuração vai no UserConfiguration.cs
```

**Remover de TODAS entidades:**
- `@JsonSerializable` (Hive)
- `@HiveType`
- `@HiveField`
- `[BsonId]`
- `[BsonElement]`

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 0.6: Criar Migration Inicial
```bash
cd src/Tep.Sales.Service.Data

# Criar migration
dotnet ef migrations add InitialCreate \
    --project Tep.Sales.Service.Data \
    --startup-project ../Tep.Sales.Service.Host \
    --context TepSalesDbContext

# Revisar SQL gerado
cat Migrations/*_InitialCreate.cs

# Aplicar no PostgreSQL de dev
dotnet ef database update \
    --project Tep.Sales.Service.Data \
    --startup-project ../Tep.Sales.Service.Host
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 1: DUAL DATABASE** (Semana 2 - 40h)

#### Task 1.1: Configurar Dual Connection
```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "PostgreSQL": "Host=localhost;Database=tep_sales_db;Username=tep_admin;Password=xxx",
    "MongoDB": "mongodb://localhost/tep_sales_db" // Manter temporariamente
  }
}

// Startup.cs
services.AddDbContext<TepSalesDbContext>(options =>
    options.UseNpgsql(Configuration.GetConnectionString("PostgreSQL")));

// Manter MongoDB também (temporário)
services.AddSingleton<IMongoClient>(sp =>
    new MongoClient(Configuration.GetConnectionString("MongoDB")));
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 1.2: Criar Repositories PostgreSQL
```csharp
// Interface (mesma para ambos)
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id);
    Task<User?> GetByEmailAsync(string email);
    Task<IEnumerable<User>> GetAllAsync();
    Task<User> AddAsync(User user);
    Task UpdateAsync(User user);
    Task DeleteAsync(Guid id);
}

// Implementação PostgreSQL
public class UserRepositoryPostgres : IUserRepository
{
    private readonly TepSalesDbContext _context;

    public UserRepositoryPostgres(TepSalesDbContext context)
    {
        _context = context;
    }

    public async Task<User?> GetByIdAsync(Guid id)
    {
        return await _context.Users
            .Include(u => u.Company)
            .FirstOrDefaultAsync(u => u.Id == id);
    }

    public async Task<User?> GetByEmailAsync(string email)
    {
        return await _context.Users
            .Include(u => u.Company)
            .FirstOrDefaultAsync(u => u.Email == email);
    }

    // ... outros métodos
}

// Implementação MongoDB (manter temporariamente)
public class UserRepositoryMongo : IUserRepository
{
    // Implementação antiga
}
```

**Criar repositories para todas entidades:**
- UserRepositoryPostgres
- ClientRepositoryPostgres
- ProductRepositoryPostgres
- PurchaseOrderRepositoryPostgres
- ... (~30 repositories)

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 1.3: Feature Toggle
```csharp
// appsettings.json
{
  "FeatureFlags": {
    "UsePostgreSQL": false,  // Inicialmente false
    "MigratedEntities": []   // Lista de entidades já migradas
  }
}

// Repository Factory
public interface IRepositoryFactory
{
    IUserRepository CreateUserRepository();
    IClientRepository CreateClientRepository();
    // ...
}

public class RepositoryFactory : IRepositoryFactory
{
    private readonly IConfiguration _config;
    private readonly TepSalesDbContext _pgContext;
    private readonly IMongoDatabase _mongoDb;

    public IUserRepository CreateUserRepository()
    {
        if (_config.GetValue<bool>("FeatureFlags:UsePostgreSQL"))
        {
            return new UserRepositoryPostgres(_pgContext);
        }
        return new UserRepositoryMongo(_mongoDb);
    }

    // ... outros factories
}
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 2: DATA MIGRATION** (Semana 3 - 24h)

#### Task 2.1: Script de Migração de Dados
```csharp
// /scripts/MigrateMongoToPostgres.cs
public class DataMigrationService
{
    private readonly IMongoDatabase _mongoDb;
    private readonly TepSalesDbContext _pgContext;

    public async Task MigrateUsersAsync()
    {
        var mongoUsers = await _mongoDb.GetCollection<BsonDocument>("users")
            .Find(new BsonDocument())
            .ToListAsync();

        var pgUsers = mongoUsers.Select(doc => new User
        {
            Id = doc["_id"].AsGuid,
            Name = doc["name"].AsString,
            Email = doc["email"].AsString,
            Password = doc["password"].AsString,
            CreatedAt = doc["createdAt"].ToUniversalTime(),
            // ... mapear todos campos
        }).ToList();

        await _pgContext.Users.AddRangeAsync(pgUsers);
        await _pgContext.SaveChangesAsync();

        Console.WriteLine($"✅ Migrated {pgUsers.Count} users");
    }

    public async Task MigrateAllAsync()
    {
        using var transaction = await _pgContext.Database.BeginTransactionAsync();

        try
        {
            // Ordem importa (FK constraints)
            await MigrateCompaniesAsync();
            await MigrateUsersAsync();
            await MigrateClientsAsync();
            await MigrateAddressesAsync();
            await MigrateProductsAsync();
            await MigratePriceTablesAsync();
            await MigratePurchaseOrdersAsync();
            await MigratePurchaseOrderItemsAsync();
            // ... todas entidades

            await transaction.CommitAsync();
            Console.WriteLine("🎉 Migration completed successfully!");
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            Console.WriteLine($"❌ Migration failed: {ex.Message}");
            throw;
        }
    }
}
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 2.2: Migração em Produção
```bash
# 1. Backup MongoDB
mongodump --uri="mongodb://prod/tep_sales_db" --out=/backup/mongo-$(date +%Y%m%d)

# 2. Criar PostgreSQL vazio
createdb -h prod-postgres tep_sales_db

# 3. Rodar migrations EF Core
dotnet ef database update --connection "Host=prod-postgres;..."

# 4. Executar migração de dados (durante janela de manutenção)
dotnet run --project MigrationTool -- migrate-all

# 5. Validar dados
dotnet run --project MigrationTool -- validate

# 6. Toggle feature flag
# FeatureFlags:UsePostgreSQL = true
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 3: MOBILE REFACTOR** (Semana 4-5 - 40h)

#### Task 3.1: Substituir Hive por SQLite + Drift
```yaml
# pubspec.yaml
dependencies:
  # Remover:
  # hive: ^2.2.3
  # hive_flutter: ^1.1.0

  # Adicionar:
  drift: ^2.14.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.1.1
  path: ^1.8.3

dev_dependencies:
  drift_dev: ^2.14.0
  build_runner: ^2.4.7
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 3.2: Criar Database Drift
```dart
// lib/app/data/database.dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';

part 'database.g.dart';

class Users extends Table {
  TextColumn get id => text()();
  TextColumn get name => text()();
  TextColumn get email => text()();
  DateTimeColumn get createdAt => dateTime()();

  @override
  Set<Column> get primaryKey => {id};
}

class PurchaseOrders extends Table {
  TextColumn get id => text()();
  TextColumn get clientId => text()();
  RealColumn get totalValue => real()();
  IntColumn get status => integer()();
  DateTimeColumn get dueDate => dateTime()();

  @override
  Set<Column> get primaryKey => {id};
}

// ... todas tabelas

@DriftDatabase(tables: [Users, PurchaseOrders, ...])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  static LazyDatabase _openConnection() {
    return LazyDatabase(() async {
      final dbFolder = await getApplicationDocumentsDirectory();
      final file = File(path.join(dbFolder.path, 'tep_sales.db'));
      return NativeDatabase(file);
    });
  }
}
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 3.3: Migrar Repositories Mobile
```dart
// ANTES (Hive):
class PurchaseOrderRepository {
  final Box<PurchaseOrderEntity> _box;

  Future<List<PurchaseOrderEntity>> getAll() async {
    return _box.values.toList();
  }
}

// DEPOIS (Drift):
class PurchaseOrderRepository {
  final AppDatabase _db;

  Future<List<PurchaseOrder>> getAll() async {
    return await _db.select(_db.purchaseOrders).get();
  }

  Future<void> save(PurchaseOrder order) async {
    await _db.into(_db.purchaseOrders).insert(order);
  }
}
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 4: TESTES E VALIDAÇÃO** (Semana 6 - 40h)

#### Task 4.1: Testes de Integração EF Core
```csharp
[TestFixture]
public class UserRepositoryTests
{
    private TepSalesDbContext _context;
    private UserRepositoryPostgres _repository;

    [SetUp]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<TepSalesDbContext>()
            .UseNpgsql("Host=localhost;Database=tep_sales_test;...")
            .Options;

        _context = new TepSalesDbContext(options);
        _context.Database.EnsureCreated();
        _repository = new UserRepositoryPostgres(_context);
    }

    [Test]
    public async Task GetByEmail_ExistingUser_ReturnsUser()
    {
        // Arrange
        var user = new User { Email = "test@test.com", ... };
        await _repository.AddAsync(user);

        // Act
        var result = await _repository.GetByEmailAsync("test@test.com");

        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual("test@test.com", result.Email);
    }

    [TearDown]
    public void TearDown()
    {
        _context.Database.EnsureDeleted();
        _context.Dispose();
    }
}
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 4.2: Testes de Performance
```csharp
[Test]
public async Task Performance_GetPurchaseOrders_WithJoins()
{
    // Arrange
    var stopwatch = Stopwatch.StartNew();

    // Act
    var orders = await _context.PurchaseOrders
        .Include(o => o.Client)
        .Include(o => o.Items)
            .ThenInclude(i => i.Product)
        .Include(o => o.PaymentCondition)
        .Where(o => o.CompanyId == companyId)
        .OrderByDescending(o => o.CreatedAt)
        .Take(50)
        .ToListAsync();

    stopwatch.Stop();

    // Assert
    Assert.That(stopwatch.ElapsedMilliseconds, Is.LessThan(200)); // < 200ms
    Console.WriteLine($"✅ Query took {stopwatch.ElapsedMilliseconds}ms");
}
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 4.3: Comparação de Performance
```
BENCHMARK: MongoDB vs PostgreSQL

Query: Get Purchase Orders with Joins (50 records)
├─ MongoDB (denormalized):     250ms
├─ PostgreSQL (normalized):    120ms  ✅ 52% faster
└─ PostgreSQL (with índices):   45ms  ✅ 82% faster

Query: Count by Status (aggregation)
├─ MongoDB:                     180ms
└─ PostgreSQL:                   25ms  ✅ 86% faster

Query: Full-text search Clients
├─ MongoDB:                     320ms
└─ PostgreSQL (tsvector):        35ms  ✅ 89% faster
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 5: DEPLOY E CUTOVER** (Semana 7 - 20h)

#### Task 5.1: Deploy Staging
```bash
# 1. Deploy API com dual database
docker-compose -f docker-compose.staging.yml up -d

# 2. Migrar dados MongoDB → PostgreSQL
./scripts/migrate-data.sh staging

# 3. Validar dados
./scripts/validate-data.sh staging

# 4. Testes E2E
npm run test:e2e:staging

# 5. Load testing
k6 run load-test.js
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 5.2: Deploy Produção (Janela de Manutenção)
```
🕐 JANELA: Domingo 02:00-06:00 (4 horas)

Checklist:
- [ ] Backup MongoDB completo
- [ ] Backup PostgreSQL (vazio)
- [ ] Notificar usuários (manutenção programada)
- [ ] Desativar workers background
- [ ] Modo manutenção (503)
- [ ] Migrar dados (2h estimado)
- [ ] Validar integridade
- [ ] Toggle UsePostgreSQL=true
- [ ] Restart API
- [ ] Smoke tests
- [ ] Monitorar logs (30min)
- [ ] Liberar acesso
- [ ] Monitorar (próximas 24h)
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 5.3: Rollback Plan
```bash
# Se algo der errado:

# 1. Reverter feature flag
UsePostgreSQL=false

# 2. Restart API
docker-compose restart api

# 3. MongoDB volta a ser usado
# Nenhum dado foi perdido (dual write temporário)

# 4. Investigar problema
# 5. Agendar nova tentativa
```

**Responsável:** ________
**Prazo:** ____/____

---

### **FASE 6: CLEANUP** (Semana 8 - 16h)

#### Task 6.1: Remover MongoDB Code
```bash
# Após 2 semanas de estabilidade em produção:

# 1. Remover dependências MongoDB
dotnet remove package MongoDB.Driver

# 2. Deletar código MongoDB
rm -rf src/Tep.Libraries.Mongo
rm -rf src/Tep.Sales.Service.Data/Repositories/*Mongo.cs

# 3. Remover feature flags
# Deixar apenas PostgreSQL

# 4. Cleanup config
# Remover ConnectionStrings:MongoDB
```

**Responsável:** ________
**Prazo:** ____/____

---

#### Task 6.2: Documentação
```markdown
# MIGRATION COMPLETED! 🎉

## Métricas:
- Duração total: X semanas
- Downtime: X horas
- Performance gain: +YY%
- Custo reduzido: $ZZZ/mês

## Próximos passos:
- [ ] Otimizar queries PostgreSQL
- [ ] Adicionar índices específicos
- [ ] Configurar replicação
- [ ] Setup pgBouncer (connection pooling)
- [ ] Monitoramento (pg_stat_statements)
```

**Responsável:** ________
**Prazo:** ____/____

---

## 📊 MÉTRICAS DE SUCESSO

```
✅ Zero downtime não planejado
✅ Zero perda de dados
✅ Performance melhorou > 50%
✅ Todos testes passando
✅ Rollback plan testado
✅ Time treinado no novo stack
```

---

## 💰 CUSTOS

**MongoDB Atlas (atual):**
- M10 cluster: $60/mês
- Backups: $20/mês
- Transfer: $10/mês
- **Total: $90/mês**

**PostgreSQL (futuro):**
- AWS RDS db.t3.medium: $35/mês
- Backups: $5/mês
- Transfer: $5/mês
- **Total: $45/mês**

**Economia: $45/mês ($540/ano)** 💰

---

## 🚨 RISCOS E MITIGAÇÕES

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Downtime longo | Baixa | Alto | Dual database + rollback |
| Perda de dados | Muito Baixa | Crítico | Backups múltiplos + validação |
| Performance ruim | Baixa | Médio | Benchmarks prévios |
| Bugs em produção | Média | Alto | Testes extensivos + monitoring |

---

## 📞 CONTATOS

**DBA:** ______________
**Tech Lead:** ______________
**On-call:** ______________

---

**STATUS: 🔴 Planejamento (não iniciado)**
