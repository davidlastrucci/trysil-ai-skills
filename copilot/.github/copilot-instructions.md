# Trysil - Delphi ORM

Instructions for using the Trysil ORM and its JSON and HTTP modules from Delphi.

---

# Trysil ORM

Trysil is a Delphi ORM. Entities are plain classes decorated with attributes; `TTContext` is the API; FireDAC is the data layer. Single-integer primary keys only, optimistic locking via a version column, RTTI required.

**Which unit declares what** (so every type lands in the right `uses` - a missing unit is the most common compile error):

| Type(s) | Unit |
|---|---|
| `TTContext` | `Trysil.Context` |
| `TTConnection` + `Connection.Execute` | `Trysil.Data` |
| `TTSQLiteConnection`, `TTSqlServerConnection`, … (drivers) | `Trysil.Data.FireDAC.<DB>` |
| `TTFireDACConnectionPool` | `Trysil.Data.FireDAC.ConnectionPool` |
| `TTTransaction`, `TTTransactionMode` | `Trysil.Transaction` |
| `TTFilter`, `TTFilterBuilder<T>` | `Trysil.Filter` |
| `TTProperty`, `TTExpression` (expression API) | `Trysil.Filter.Expression` |
| `TTSession<T>` | `Trysil.Session` |
| `TTList<T>` | `Trysil.Generics.Collections` |
| `TTLazy<T>`, `TTLazyList<T>` | `Trysil.Lazy` |
| `TTPrimaryKey`, `TTVersion`, `TTNullable<T>` | `Trysil.Types` |
| `[TTable]`, `[TColumn]`, `[TRelation]`, `[TDetailColumn]`, change-tracking attrs | `Trysil.Attributes` |
| `[TRequired]`, `[TMaxLength]`, `[TGreater]`, `[TEMail]`, … | `Trysil.Validation.Attributes` |
| `ETException`, `ETValidationException`, `ETConcurrentUpdateException` | `Trysil.Exceptions` |

## 0. Quickstart - end to end (SQLite)

A full round-trip: register a connection, create the schema (Trysil does not create tables - see section 3a), insert, select.

```delphi
uses
  Trysil.Data,
  Trysil.Context,
  Trysil.Generics.Collections,
  Trysil.Data.FireDAC.ConnectionPool,
  Trysil.Data.FireDAC.SQLite;

var
  LConnection: TTConnection;
  LContext: TTContext;
  LCustomer: TCustomer;
  LList: TTList<TCustomer>;
begin
  TTFireDACConnectionPool.Instance.Config.Enabled := False;
  TTSQLiteConnection.RegisterConnection('Demo', 'demo.db');
  LConnection := TTSQLiteConnection.Create('Demo');
  try
    LContext := TTContext.Create(LConnection);
    try
      LConnection.Execute(                       // you own the schema
        'CREATE TABLE IF NOT EXISTS Customers (' +
        ' ID INT NOT NULL PRIMARY KEY,' +
        ' CompanyName NVARCHAR(100),' +
        ' Email NVARCHAR(255),' +
        ' VersionID INT NOT NULL)');

      LCustomer := LContext.CreateEntity<TCustomer>();   // ID assigned up front
      LCustomer.CompanyName := 'Acme';
      LContext.Insert<TCustomer>(LCustomer);

      LList := TTList<TCustomer>.Create;
      try
        LContext.SelectAll<TCustomer>(LList);            // fills the list
      finally
        LList.Free;
      end;
    finally
      LContext.Free;        // always free the context before the connection
    end;
  finally
    LConnection.Free;
  end;
end;
```

`Connection.Execute(sql): Integer` runs arbitrary SQL (DDL/DML) and returns rows affected - that is how you create the schema. The `TCustomer` entity used here is defined in section 3.

## 1. Connection setup

Pick the concrete driver class for the target database. Register the connection (once), then create instances.

```delphi
uses
  Trysil.Data,
  Trysil.Data.FireDAC.ConnectionPool,
  Trysil.Data.FireDAC.SQLite;   // or .SqlServer, .PostgreSQL, .FirebirdSQL, .InterBase, .MariaDB, .Oracle

// SQLite
TTFireDACConnectionPool.Instance.Config.Enabled := False;
TTSQLiteConnection.RegisterConnection('Test', ADatabaseFileName);
FConnection := TTSQLiteConnection.Create('Test');

// SQL Server (pooling on)
TTFireDACConnectionPool.Instance.Config.Enabled := True;
TTSqlServerConnection.RegisterConnection('Test', 'Server', 'user', 'pwd', 'DbName');
FConnection := TTSqlServerConnection.Create('Test');

// PostgreSQL
TTPostgreSQLConnection.RegisterConnection('Pg', 'host', 5432, 'user', 'pwd', 'db');
FConnection := TTPostgreSQLConnection.Create('Pg');
```

`Config` is the **default** for every connection definition. To give one connection its own pool limits - typical when an application hosts a busy app database and a quiet log database - put the parameters in the `TTFireDACConnectionParameters` record and register through the factory:

```delphi
LParameters := Default(TTFireDACConnectionParameters);   // records: zero it first
LParameters.Driver := 'SQLite';
LParameters.DatabaseName := 'log.db';
LParameters.PoolParameters := TTFireDACPoolParameters.Create(True, 2);
TTFireDACConnectionFactory.Instance.RegisterConnection('Log', LParameters);
```

`TTFireDACPoolParameters` is immutable; `IsAssigned` separates "not declared, use the global `Config`" from "declared", so `Create(False, 1)` disables pooling for that one connection instead of inheriting the global setting.

All drivers extend the abstract `TTConnection`, so a `TTConnection` variable can hold any of them and is what `TTContext` expects - useful if the target database may change.

**FireDAC wait-cursor unit (easy to miss).** FireDAC needs a wait-cursor provider linked into the project, matched to the application type, or it errors the first time it touches the UI layer. Add the right one to the project's uses (typically the `.dpr` or main unit):

- Console app: `FireDAC.ConsoleUI.Wait`
- VCL app: `FireDAC.VCLUI.Wait`
- FireMonkey app: `FireDAC.FMXUI.Wait`

A Trysil console program therefore needs `FireDAC.ConsoleUI.Wait` somewhere in the project even though no Trysil unit references it directly.

## 2. Context

`TTContext` (`Trysil.Context`) is the entry point. Free the context before the connection.

```delphi
FConnection := TTSqlServerConnection.Create('Test');
FContext := TTContext.Create(FConnection);
// ... use ...
FContext.Free;
FConnection.Free;
```

Constructor overloads: `Create(AConnection)`, `Create(AConnection, AUseIdentityMap)`, `Create(AReadConnection, AWriteConnection)`, `Create(AReadConnection, AWriteConnection, AUseIdentityMap)`. The identity map is scoped to the context instance (safe for multi-tenant - no global state).

## 2a. Object ownership & lifetime (who frees what)

Trysil has **two memory models**, picked by the identity-map flag on the context.
Getting this wrong gives double-frees or leaks; it is the single most common
mistake. Decide the model up front and keep one unit of work on one model.

**Identity map OFF** (the default: `Create(AConnection)` or
`Create(AConnection, False)`) - **the caller owns** every entity it receives:

- Entities from `Get` / `TryGet` / `Select` / `SelectAll` / `RawSelect` /
  `CreateEntity` are yours to free.
- A `TTLazy<T>` N:1 reference **clones the entity you assign to it**.
  `LOrder.Customer := LCustomer` stores a copy: `LOrder` owns that copy and frees
  it when `LOrder` is freed (and also when you reassign the reference or change
  its id), while `LCustomer` stays yours to free. Consequences:
  - Free the entity you assigned, as usual - the lazy field never frees your
    instance, only its own clone.
  - Assigning **one** instance to several owners is safe: each owner gets its
    own clone.
  - Reading the reference back returns the clone, not your instance: changes made
    to your object after the assignment do not reach the stored reference.
- A `TTLazyList<T>` (1:N detail) owns the child entities it loads and frees them
  with the owner.

**Identity map ON** (`Create(AConnection, True)`) - **the context owns
everything**. The map is a `TObjectDictionary` with `doOwnsValues`, so every
entity from `Get` / `Select` / `CreateEntity` lives and dies with the context:

- Do **not** free any entity yourself - freeing the context frees them all
  (a manual `Free` is a double free).
- Pass **non-owning** result lists (`TTObjectList<T>.Create(False)`), otherwise
  the list and the map both free the same objects.
- `TTLazy<T>` / `TTLazyList<T>` do **not** free their entities in this mode (the
  map owns them).

**Two helpers that work in both modes** (prefer them so the same code is correct
either way):

- `Context.CreateEntityList<T>: TTList<T>` returns a list whose `OwnsObjects` is
  already set correctly for the current mode (`True` when off, `False` when on).
- `Context.FreeEntity<T>(AEntity)` frees the entity only when the identity map is
  off; it is a safe no-op when on.

```delphi
// Mode-agnostic read + cleanup:
LList := FContext.CreateEntityList<TCustomer>;   // ownership matches the mode
try
  FContext.SelectAll<TCustomer>(LList);
  // ... use ...
finally
  LList.Free;          // frees the entities only if it owns them
end;
```

Rule of thumb for a script/console unit of work: with the identity map **on**
nothing you hold needs manual freeing; with it **off** you free what you created
or received, and each lazy reference independently owns its own clone.

To set a foreign key without any object at all, model the FK as a plain `Integer`
`[TColumn]` field (e.g. `OrderDetail.OrderID`) and assign the id directly - no
lazy, no clone. N:1 references modeled as `TTLazy<T>` always go through object
assignment, and thus through the rules above.

## 3. Entity mapping

```delphi
unit Demo.Model;

interface

uses
  System.SysUtils,
  Trysil.Types,
  Trysil.Attributes,
  Trysil.Validation.Attributes;

type
  [TTable('Customers')]
  [TSequence('CustomersID')]
  TCustomer = class
  strict private
    [TColumn('ID')]
    [TPrimaryKey]
    FID: TTPrimaryKey;

    [TColumn('CompanyName')]
    [TRequired]
    [TMaxLength(100)]
    FCompanyName: String;

    [TColumn('Email')]
    [TMaxLength(255)]
    [TEMail]
    FEmail: String;

    [TColumn('VersionID')]
    [TVersionColumn]
    FVersionID: TTVersion;
  public
    property ID: TTPrimaryKey read FID;
    property CompanyName: String read FCompanyName write FCompanyName;
    property Email: String read FEmail write FEmail;
    property VersionID: TTVersion read FVersionID;
  end;
```

Key attributes: `[TTable('name')]`, `[TPrimaryKey]`, `[TColumn('col')]`, `[TVersionColumn]`, `[TSequence('seqID')]`, `[TRelation('Table','FK',OwnsObjects)]`, `[TWhereClause('sql')]` + `[TWhereClauseParameter('name', value)]` (compile-time constants only - for runtime filtering use `TTFilterBuilder<T>`).

### ID generation & `[TSequence]` per driver
Keep `[TSequence('Name')]` on every entity regardless of the target database. SQLite has no sequence objects - Trysil computes the next ID with `IFNULL(MAX(ROWID), 0) + 1` - but the other supported drivers (PostgreSQL/SQL Server/Firebird/InterBase/MariaDB/Oracle) rely on real sequences/identity. Leaving the attribute in place keeps the model portable: if you later switch driver, the sequence is already declared and nothing in the entity needs to change.

Add `{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}` to units that use Trysil attributes - Trysil resolves attributes via RTTI, so a misspelled attribute name compiles silently otherwise; this turns the typo into a compile error.

### Type system (`Trysil.Types`)
- `TTPrimaryKey` = `Int32` (single-integer PKs only).
- `TTVersion` = `Int32` (optimistic locking).
- `TTNullable<T>` - generic nullable wrapper, **no default constructor**; uninitialized = null. Set with `TTNullable<T>.Create(value)`. Test for null with `.IsNull` (the property is `IsNull`, **not** `HasValue` - there is no `HasValue`; for "has a value" write `not X.IsNull`). Read the payload with `.Value` (or `.GetValueOrDefault`).

### Mapping is cached
`TTMapper.Instance` is a global singleton converting classes to `TTTableMap` on first access.

## 3a. Database schema & column types

Trysil maps to **existing** tables - it does not create, migrate, or alter the schema. You create the tables; each `[TColumn('Name')]` must match a real column by name, and the column type must be compatible with the field type. A mismatch surfaces at **runtime**, not at compile time. Mapping by Delphi field type:

| Entity field type | FireDAC field | Typical SQL column |
|---|---|---|
| `Integer` / `TTPrimaryKey` / `TTVersion` | `ftInteger` (`ftSmallint`) | `INT` |
| `Int64` | `ftLargeint` | `BIGINT` |
| `String` | `ftWideString` / `ftString` | `NVARCHAR(n)` / `VARCHAR(n)` |
| `String` (long text) | `ftWideMemo` / `ftMemo` | `NTEXT` / `TEXT` |
| `Double` | `ftFloat` (`ftBCD`/`ftFMTBcd`/`ftCurrency`/`ftSingle`) | `FLOAT` / `NUMERIC(p,s)` |
| `Currency` | `ftBCD` / `ftFMTBcd` (`ftCurrency`/`ftFloat`) | `DECIMAL(19,4)` / `NUMERIC(19,4)` |
| `Boolean` | `ftBoolean` | `BOOLEAN` / `BIT` |
| `TDateTime` | `ftDateTime` (`ftDate`/`ftTimeStamp`) | `DATETIME` / `TIMESTAMP` |
| `TGuid` | `ftGuid` | `UNIQUEIDENTIFIER` / `CHAR(36)` / `UUID` |
| `TBytes` | `ftBlob` | `BLOB` / `VARBINARY` / `BYTEA` |

- Use `Currency` for money, not `Double`. `Currency` is a fixed-point type exact to four decimal places, and Trysil maps it end to end without ever converting through `Double`: values are read with `TField.AsCurrency` and written with the parameter's `AsCurrency`, so the four decimals survive the round trip. `TTNullable<Currency>` works the same way. Pair it with a `DECIMAL(19,4)` column - Firebird and InterBase cap precision at 18 digits, so declare `DECIMAL(18,4)` there, and Oracle spells it `NUMBER(19,4)`. SQLite has no decimal type at all: the column keeps NUMERIC affinity but the value is stored as a float, so exactness there is limited to what a double can hold.
- Use `Double` for genuine floating-point quantities (measures, ratios, coordinates), not for amounts of money.
- Enumerations are stored as their integer ordinal (`INT`) and converted via RTTI; no dedicated column type.
- `TTNullable<T>` maps to the same type as `T` - just make the DB column nullable.
- Primary key and `[TVersionColumn]` columns are `INT NOT NULL`. Soft-delete columns (section 8) are a nullable `DATETIME` (`*At`) plus a `NVARCHAR` (`*By`).

## 4. CRUD

`SelectAll`/`Select` are **procedures** that fill a caller-owned list - they do not return one.

```delphi
uses Trysil.Context, Trysil.Generics.Collections;

// Read all
LList := TTList<TCustomer>.Create;
try
  FContext.SelectAll<TCustomer>(LList);
finally
  LList.Free;
end;

// Get by PK (nil if missing) / TryGet
LCustomer := FContext.Get<TCustomer>(AID);
if FContext.TryGet<TCustomer>(AID, LCustomer) then ...;

// Insert - use CreateEntity to get a valid sequence-assigned ID up front
LCustomer := FContext.CreateEntity<TCustomer>();
LCustomer.CompanyName := 'Acme';
FContext.Insert<TCustomer>(LCustomer);

// Update / Delete
FContext.Update<TCustomer>(LCustomer);
FContext.Delete<TCustomer>(LCustomer);

// Save = insert-or-update (tracked via TTNewEntityCache)
FContext.Save<TCustomer>(LCustomer);

// Batch - all three lists in one transaction
FContext.ApplyAll<TCustomer>(LInsertList, LUpdateList, LDeleteList);
```

Notes:
- `CreateEntity<T>` returns an entity with the sequence ID already assigned (never ID=0) - no post-`Save` FK patching needed.
- `Update<T>` rewrites the whole row (no per-field diff); it emits an UPDATE even if nothing changed.
- When the identity map is on, the context **owns** the entities it returns - do not free them yourself.
- `Refresh<T>(entity)` reloads in place an entity you already hold; `Get<T>(id)` fetches by primary key and returns an instance.

## 5. TTFilterBuilder<T>

Lives in `Trysil.Filter`. Obtain via `CreateFilterBuilder<T>`, chain, `Build` to a `TTFilter`, pass to `Select`. Free the builder; the `TTFilter` is a value record.

```delphi
LBuilder := FContext.CreateFilterBuilder<TCustomer>();
try
  LFilter := LBuilder
    .Where('CompanyName').Like('Acme%')
    .AndWhere('Email').IsNotNull
    .OrderByAsc('CompanyName')
    .Limit(20).Offset(0)
    .Build;
finally
  LBuilder.Free;
end;

LList := TTList<TCustomer>.Create;
try
  FContext.Select<TCustomer>(LList, LFilter);
finally
  LList.Free;
end;
```

Conditions: `Equal`, `NotEqual`, `Greater`, `GreaterOrEqual`, `Less`, `LessOrEqual`, `Like`, `NotLike`, `IsNull`, `IsNotNull`. Combine with `Where`/`AndWhere`/`OrWhere`. Paging/order: `OrderByAsc`/`OrderByDesc`, `Limit`, `Offset`. Use `TTFilter.Empty` for "no filter" and `SelectCount<T>(AFilter)` for counts.

The value-taking conditions (`Equal`, `Greater`, `Like`, …) each take a single `const AValue: TTValue` parameter - there are **no** per-type overloads. `TTValue` accepts any scalar (`String`, `Integer`, `Double`, `Currency`, `Boolean`, `TDateTime`, …) by implicit conversion, so pass the value directly whatever its type: `.Where('Description').Equal('Widget')`, `.AndWhere('Price').Greater(5.0)`, `.AndWhere('BrandID').Equal(LBrand.ID)`.

## 5a. Expression API - grouped conditions (`Trysil.Filter.Expression`)

The flat `Where`/`AndWhere`/`OrWhere` chain cannot express grouped logic like
`(A or B) and C`, because SQL binds `AND` tighter than `OR`. For that, use the
algebraic expression API. `TTProperty` wraps a column name and overloads the
comparison operators (plus `Like` / `NotLike` / `IsNull` / `IsNotNull` /
`Between` / `InValues`) to produce a `TTExpression`; `and` / `or` / `not`
combine expressions with **explicit** parentheses.

```delphi
LBuilder := FContext.CreateFilterBuilder<TCustomer>();
try
  LFilter := LBuilder
    .Where(
      ((TTProperty.Create('City') = 'Rome') or
       (TTProperty.Create('City') = 'Milan')) and
      (TTProperty.Create('Active') = True))
    .OrderByDesc(TTProperty.Create('CompanyName'))
    .Build;
finally
  LBuilder.Free;
end;
```

- **Each comparison must be parenthesized** due to Delphi operator precedence:
  write `(TTProperty.Create('Age') >= 18)`, not `TTProperty.Create('Age') >= 18`.
- Comparison operators: `=`, `<>`, `>`, `>=`, `<`, `<=`. Methods on `TTProperty`:
  `Like`, `NotLike`, `IsNull`, `IsNotNull`, `Between(low, high)`,
  `InValues([...])`.
- Builder overloads accept expressions: `Where` / `AndWhere` /
  `OrWhere(const AExpression: TTExpression)` and `OrderByAsc` /
  `OrderByDesc(const AProperty: TTProperty)`. They share the same `:pN`
  parameter counter as the fluent string form, so the two mix in one builder.
- For JOIN entities use the two-argument `TTProperty.Create(Alias, Column)` -
  it qualifies the WHERE reference as `Alias.Column`. The fluent string form
  (`.Where('Column')`) does not qualify aliases.

## 5b. In-memory filtering - `TTList<T>.Where` (`Trysil.Generics.Collections`)

`TTList<T>` (the list `SelectAll`/`Select` fill) descends from RTL `TList<T>` - index access, `Count`, and `for..in` all work. It adds a LINQ-style **lazy** filter for client-side use:

```delphi
TTPredicate<T> = reference to function(const AItem: T): Boolean;
function TTList<T>.Where(const APredicate: TTPredicate<T>): ITEnumerable<T>;
```

`Where` returns a lazy `ITEnumerable<T>` (interface-based, ref-counted enumerator) yielding only matching items during enumeration - no intermediate list is built:

```delphi
for LCustomer in LList.Where(
  function(const A: TCustomer): Boolean
  begin
    result := A.CompanyName.ToLower.Contains(LText);
  end) do
  // ...
```

`Where(nil)` enumerates everything. Use this to filter an **already-loaded** set (no DB round-trip, no raw SQL) - e.g. live search over a list held in memory; use `TTFilterBuilder<T>`/`TTFilter` when you want the database to do the filtering.

Other types in the unit: `TTObjectList<T>` (owns items - frees on remove when `OwnsObjects`), `TTObjectLazyList<T>` (adds `IsValid` for lazy 1:N lists), `TTHashList<T>` (dictionary-backed set: `Add`/`Contains`/`Remove`).

## 6. Lazy loading (`Trysil.Lazy`)

`TTLazy<T>` (N:1) and `TTLazyList<T>` (1:N). **Never** create or free these manually and never add a separate `FxxxID` field - the framework instantiates and releases them via RTTI and triggers reload when `.ID` is set. Expose plain-typed properties through getters.

`TTLazy<T>.IsLoaded` reports whether the reference is already in memory **without triggering the load** - reading `.Entity` to find out would load it. Use it to avoid N+1 in a grid, or in tests as the assertion for "no query was issued". `TTLazyList<T>` does not have it.

```delphi
[TTable('Orders')]
[TSequence('OrdersID')]
[TRelation('OrderDetails', 'OrderID', True)]
TOrder = class
strict private
  [TColumn('ID')]
  [TPrimaryKey]
  FID: TTPrimaryKey;

  [TColumn('CustomerID')]
  [TRequired]
  FCustomer: TTLazy<TCustomer>;

  [TDetailColumn('ID', 'OrderID')]
  FDetail: TTLazyList<TOrderDetail>;

  [TColumn('VersionID')]
  [TVersionColumn]
  FVersionID: TTVersion;

  function GetCustomer: TCustomer;
  procedure SetCustomer(const AValue: TCustomer);
  function GetDetail: TTList<TOrderDetail>;
public
  property Customer: TCustomer read GetCustomer write SetCustomer;
  property Detail: TTList<TOrderDetail> read GetDetail;
  property VersionID: TTVersion read FVersionID;
end;

// getters
function TOrder.GetCustomer: TCustomer;
begin
  result := FCustomer.Entity;       // loads on first access
end;

procedure TOrder.SetCustomer(const AValue: TCustomer);
begin
  FCustomer.Entity := AValue;
end;

function TOrder.GetDetail: TTList<TOrderDetail>;
begin
  result := FDetail.List;
end;
```

### Persisting a master with its detail (no cascade)
Lazy lists are **read-side only**. Adding objects to `Order.Detail` and then
calling `Insert<TOrder>(order)` does **not** write the children: there is no
cross-type cascade and the lines are silently lost. Persist the master first,
then insert each child with its foreign-key field set to the saved master's ID:

```delphi
LOrder := FContext.CreateEntity<TOrder>();
LOrder.Customer := LCustomer;          // N:1 lazy: sets the CustomerID column
FContext.Insert<TOrder>(LOrder);       // master first, ID is now assigned

LLine := FContext.CreateEntity<TOrderDetail>();
LLine.OrderID := LOrder.ID;            // FK to the just-saved master
LLine.Product := LProduct;             // N:1 lazy: sets the ProductID column
LLine.Quantity := 2;
FContext.Insert<TOrderDetail>(LLine);  // then each child, one by one
```

**Atomicity: wrap master + children in one transaction.** Each
`Insert`/`Update`/`Delete` runs in its own transaction, so a failure partway
through saving an order and its lines leaves a partial result (order saved, some
lines missing). To make the whole graph all-or-nothing, use
`Context.RunInTransaction`:

```delphi
FContext.RunInTransaction(
  procedure
  var
    LLine: TOrderDetail;
  begin
    FContext.Insert<TOrder>(LOrder);
    for LLine in LLines do
      FContext.Insert<TOrderDetail>(LLine);
  end);
```

It commits when the procedure returns normally, rolls back and re-raises on an
exception, and **joins the current transaction** when one is already active - so
a method written this way is atomic whether it is called on its own or from
inside a larger unit of work.

When you need to decide commit and rollback yourself, `Context.CreateTransaction`
takes a `TTTransactionMode` (unit `Trysil.Transaction` - add it to your `uses`):

```delphi
LTransaction := FContext.CreateTransaction(
  TTTransactionMode.RollbackOnDestroy);
try
  FContext.Insert<TOrder>(LOrder);
  for LLine in LLines do
    FContext.Insert<TOrderDetail>(LLine);

  LTransaction.Commit;
finally
  LTransaction.Free;
end;
```

Use `RollbackOnDestroy`: any path that leaves the block without reaching
`Commit` rolls back, which is what makes the plain `try .. finally` correct.
`CommitOnDestroy` is the old behaviour, where destruction is what commits, so
the same `try .. finally` would commit half-written work on an exception.
`CreateTransaction` with no argument is deprecated and means `CommitOnDestroy`.

`ApplyAll<T>` already runs its insert/update/delete lists in a single
transaction, but only within one entity type.

`ApplyAll<T>` opens a transaction **only when the write connection has none**, so
calling it inside a transaction you opened yourself joins that one instead of
nesting: several `ApplyAll<T>` calls on different entity types stay atomic
together, and commit/rollback remain yours.

Assigning a `TTLazy<T>` N:1 reference (e.g. `LLine.Product := LProduct`) writes
that field's `[TColumn]` foreign-key value on the next insert/update: you set
the related object, not its ID. `ApplyAll<T>` and `TTSession<T>` batch operations
within a **single** entity type only; they do not cascade across types either.

**Ownership when wiring references:** with the identity map off, the lazy field
stores a **clone** of the object you assign, so you keep owning (and must free)
the instance you passed, and one instance can be assigned to several owners.
See section 2a.

## 7. TTSession<T> - Unit of Work (`Trysil.Session`)

Clones entities on creation; compares clones to originals on `ApplyChanges`. Full cloning is the only correct implementation in Delphi (no dynamic proxies). The session has explicit state per entity (Original/Inserted/Updated/Deleted); cloning gives isolation, not field-level diffing.

```delphi
LSession := FContext.CreateSession<TCustomer>(LList);
try
  LNew := FContext.CreateEntity<TCustomer>();
  LNew.CompanyName := 'New';
  LSession.Insert(LNew);

  LSession.Entities[0].CompanyName := 'Changed';
  LSession.Update(LSession.Entities[0]);

  LSession.Delete(LSession.Entities[1]);

  LSession.ApplyChanges;            // one transaction
finally
  LSession.Free;
end;
```

## 8. Change tracking & soft delete

Attribute pairs auto-populated by the resolver. `*At` fields are `TTNullable<TDateTime>`; `*By` fields are `String` and require the context **instance** property `OnGetCurrentUser: TFunc<String>` - assign it on the context *after* you create it, not on the `TTContext` type (it is not a class-level member). Empty string if unassigned. The property is a `TFunc<String>`, so assign an anonymous method:

```delphi
FContext := TTContext.Create(LConnection, True);
FContext.OnGetCurrentUser :=
  function: String
  begin
    Result := 'demo-user';   // your real current-user lookup here
  end;
```

Expose every change-tracking column through a read-only property (as you do for `ID` / `VersionID`), the whole `*At` **and** `*By` set. The resolver writes these fields via RTTI, so a mapped private field you never reference from code draws a harmless `H2219 ... declared but never used` compiler hint; a read-only property both clears the hint and lets you read who/when.

| Attribute | Fired on |
|---|---|
| `[TCreatedAt]` / `[TCreatedBy]` | Insert |
| `[TUpdatedAt]` / `[TUpdatedBy]` | Update |
| `[TDeletedAt]` / `[TDeletedBy]` | Delete (soft) |

With `[TDeletedAt]` present, `Delete<T>` does **not** issue SQL DELETE - it UPDATEs `DeletedAt`/`DeletedBy` and bumps the version. All SELECTs add `DeletedAt IS NULL`. To include soft-deleted rows: `TTFilter.IncludeDeleted := True` or `.IncludeDeleted` on the builder.

## 9. JOIN queries (read-only)

`[TJoin(Kind, 'Table'[, 'Alias'][, 'SourceTableOrAlias'], 'SourceCol', 'TargetCol')]` (`TJoinKind` = `Inner`/`Left`/`Right`) plus the 2-arg `[TColumn('Alias','Col')]`. Join entities are read-only - `Insert`/`Update`/`Delete` raise `ETException`. `TTFilterBuilder` does not resolve join aliases; use `TTFilter.Create(whereClause)` with manually qualified column names.

## 10. Raw select

For SQL attributes can't express (subqueries, UNION, GROUP BY, aggregates), map raw results to a DTO whose fields carry only `[TColumn('ResultColName')]`:

```delphi
Context.RawSelect<TOrderSummary>(
  'SELECT c.CompanyName AS CustomerName, SUM(o.Amount) AS Total ' +
  'FROM Orders o JOIN Customers c ON o.CustomerID = c.ID GROUP BY c.CompanyName',
  LResult);
```

Read-only, no identity map, no lazy loading.

## 11. Exceptions & optimistic locking

All Trysil exceptions derive from `ETException` (`Trysil.Exceptions`):

| Exception | Raised when |
|---|---|
| `ETValidationException` | a validation rule fails on Insert/Update (the message lists the failures) |
| `ETConcurrentUpdateException` | optimistic-lock conflict: an Update/Delete in `KeyAndVersionColumn` mode matched **0 rows** (the version changed under you) |
| `ETDataIntegrityException` | an Update/Delete matched **more than one** row |
| `ETException` | base / generic errors |

- The version check is automatic when the entity has a `[TVersionColumn]` and `TTUpdateMode` is `KeyAndVersionColumn` (the default). Catch `ETConcurrentUpdateException` to handle "modified by another user". Use `KeyOnly` for tables without a version column.
- `ETValidationException` exposes the failures only through its `.Message` text - the per-field list is not individually iterable. Read `E.Message` for the formatted reasons.
- `Get<T>(id)` returns `nil` when the row does not exist; it does **not** raise. Use `TryGet<T>(id, entity): Boolean` for the explicit form.

## 12. Validation, custom validators & lifecycle events

**Field validation attributes** (`Trysil.Validation.Attributes`) - each also has an overload taking a custom error-message string:

`[TRequired]`, `[TMaxLength(n)]`, `[TMinLength(n)]`, `[TMinValue(n)]`, `[TMaxValue(n)]`, `[TLess(n)]`, `[TGreater(n)]`, `[TRange(min, max)]`, `[TRegex('pattern')]`, `[TEMail]`.

**Value type must match the field type** (`TMinValue` / `TMaxValue` / `TLess` / `TGreater` / `TRange`): the literal you pass picks the comparison type, and it has to match the field's type, or validation fails at runtime with `<Column> type not valid for validation`. For a `Double` or `Currency` field write a float literal - `[TGreater(0.0)]`, not `[TGreater(0)]` - and for an `Integer` field write an integer literal - `[TGreater(0)]`. So `Price: Currency` needs `[TGreater(0.0)]` / `[TMinValue(0.01)]`, while `Quantity: Integer` needs `[TGreater(0)]`. The attribute takes a `Double` argument in both cases; there is no `Currency` overload, and none is needed.

**`[TEMail]`** validates a basic email format with a built-in regex. An empty
string passes (the check is skipped when the value is empty, so add `[TRequired]`
as well if the field is mandatory). For stricter or non-standard rules, use
`[TRegex('<your pattern>')]` with your own pattern instead.

```delphi
[TColumn('Email')]
[TRequired]
[TMaxLength(255)]
[TEMail]
FEmail: String;
```

**Custom validators** - a method marked `[TValidator]` that records failures into a `TTValidationErrors`. Three accepted signatures (pick one):

```delphi
[TValidator]
procedure ValidateName(const AErrors: TTValidationErrors);
//  also valid:  procedure ValidateName;
//               procedure ValidateName(const AContext: TObject; const AErrors: TTValidationErrors);

procedure TCustomer.ValidateName(const AErrors: TTValidationErrors);
begin
  if FCompanyName = 'admin' then
    AErrors.Add('CompanyName', 'CompanyName cannot be "admin"');
end;
```

Any failure added makes the resolver raise `ETValidationException` before the row is written.

**Lifecycle events** - parameterless methods marked with one of these attributes (`Trysil.Events.Attributes`), fired by the resolver around the operation:

`[TBeforeInsertEvent]`, `[TAfterInsertEvent]`, `[TBeforeUpdateEvent]`, `[TAfterUpdateEvent]`, `[TBeforeDeleteEvent]`, `[TAfterDeleteEvent]`.

```delphi
[TBeforeInsertEvent]
procedure OnBeforeInsert;   // must take no parameters
```

These run your code; they are distinct from the change-tracking attributes in section 8, which auto-fill columns.

## 13. Logging - see the generated SQL

`TTLogger.Instance` dispatches SQL events to a background writer you register. There is **no simple callback**: subclass `TTLoggerThread` (`Trysil.Logger`), override the abstract `Log*` methods, and register the class once at startup.

```delphi
type
  TConsoleLogger = class(TTLoggerThread)
  strict protected
    procedure LogStartTransaction(const AID: TTLoggerItemID); override;
    procedure LogCommit(const AID: TTLoggerItemID); override;
    procedure LogRollback(const AID: TTLoggerItemID); override;
    procedure LogParameter(const AID: TTLoggerItemID;
      const AName: String; const AValue: String); override;
    procedure LogSyntax(
      const AID: TTLoggerItemID; const ASyntax: String); override;
    procedure LogCommand(
      const AID: TTLoggerItemID; const ASyntax: String); override;
    procedure LogError(
      const AID: TTLoggerItemID; const AMessage: String); override;
  end;

procedure TConsoleLogger.LogSyntax(
  const AID: TTLoggerItemID; const ASyntax: String);
begin
  Writeln('SELECT: ' + ASyntax);            // SELECT statements
end;

procedure TConsoleLogger.LogCommand(
  const AID: TTLoggerItemID; const ASyntax: String);
begin
  Writeln('DML/DDL: ' + ASyntax);           // INSERT/UPDATE/DELETE and DDL
end;
// ... implement the remaining overrides (empty bodies are fine) ...

// startup:
TTLogger.Instance.RegisterLogger<TConsoleLogger>();
```

`LogSyntax` receives SELECT SQL, `LogCommand` receives INSERT/UPDATE/DELETE/DDL, `LogParameter` receives name/value pairs. `TTLoggerItemID` carries `ConnectionID` and `ThreadID` for multi-threaded correlation. The writer runs on its own background thread.

## 14. Inheritance, threading & testing

- **Inheritance:** an entity may inherit mapped members from a base class. The mapper walks the full inheritance chain (fields, properties, and class-level attributes), so a base class can hold shared `[TColumn]` fields (e.g. `ID`, `VersionID`, change-tracking columns) and subclasses add their own. This is mapping reuse, not polymorphic persistence (no single-table/joined inheritance).
- **Threading:** a `TTContext` (and its `TTConnection`) is a unit of work for one thread - do **not** share an instance across threads. Concurrency is by isolation, not locking: create one context per request/thread (the HTTP module does exactly this per request) and let `TTFireDACConnectionPool` pool the underlying DB connections.
- **Testing:** the framework is tested against SQLite, one connection per fixture, no mocking. Register a SQLite connection on a temp file (`TPath.GetTempFileName`), run your `CREATE TABLE` DDL via `Connection.Execute(...)`, build a `TTContext`, exercise it, then free the context and delete the file in teardown.

## Architectural facts worth knowing
- `TTContext` delegates reads to `TTProvider` and writes to `TTResolver`.
- Validation, custom validators (`[TValidator]`), and lifecycle events: see sections 12 and 11.
- `TTUpdateMode`: `KeyAndVersionColumn` (default, optimistic lock) vs `KeyOnly` (table without `[TVersionColumn]`).
- Interfaces in Delphi cannot have generic methods, so `TTContext`/`TTProvider`/`TTResolver` are concrete classes by necessity - test via SQLite in-memory, not mocks.

---

# Trysil JSON

`TTJSonContext` (`Trysil.JSon.Context`) extends `TTContext`, so it has the full ORM API (`Select`, `Insert`, `CreateEntity`, ...) plus JSON serialization. Read the **trysil-orm** skill for entities, connections, and CRUD - all of it applies here.

**Which unit declares what** (in addition to the trysil-orm units):

| Type(s) | Unit |
|---|---|
| `TTJSonContext` | `Trysil.JSon.Context` |
| `TTJSonSerializerConfig` | `Trysil.JSon.Types` |
| `[TJSonIgnore]` | `Trysil.JSon.Attributes` |
| `ETJSonException` | `Trysil.JSon.Exceptions` |

## Critical: no identity map

`TTJSonContext` **must not** use the identity map. Passing `AUseIdentityMap = True` raises `ETJSonException` ("TTJSonContext can not use IdentityMap."). This is by design: JSON contexts are request-scoped/stateless and an identity map would conflict during deserialization. Always construct with the map off.

## Construction

```delphi
uses
  Trysil.Data,
  Trysil.JSon.Context,
  Trysil.JSon.Types;

FJSonContext := TTJSonContext.Create(FConnection);
```

Overloads mirror `TTContext` (read/write split supported), minus any identity-map-on form:
- `Create(AConnection)`
- `Create(AReadConnection, AWriteConnection)`

## TTJSonSerializerConfig

A **record** in `Trysil.JSon.Types` - has no default; always construct it explicitly.

```delphi
constructor Create(const AMaxLevels: Integer; const ADetails: Boolean);
```

- `AMaxLevels` - max nesting depth for related entities; `-1` = unlimited, `0` = none (relations emitted as IDs only).
- `ADetails` - include detail (1:N) collections.

`AMaxLevels` bounds **queries**, not just payload: past the level the serializer does not resolve the lazy reference at all, it emits the foreign key id and moves on. Use `0` on list endpoints to avoid `rows x N:1 relations` round trips.

```delphi
LConfig := TTJSonSerializerConfig.Create(-1, False);  // defaults: unlimited depth, no details
LConfigGet    := TTJSonSerializerConfig.Create(1, True);   // one level deep, with details
LConfigSelect := TTJSonSerializerConfig.Create(0, False);  // list endpoint: relations as ids, no per-row round trips
LConfigFind   := TTJSonSerializerConfig.Create(0, False);
```

## Serialize

```delphi
function EntityToJSon<T: class>(const AEntity: T; const AConfig: TTJSonSerializerConfig): String;
function EntityToJSonObject<T: class>(const AEntity: T; const AConfig: TTJSonSerializerConfig): TJSonObject;
function ListToJSon<T: class>(const AList: TList<T>; const AConfig: TTJSonSerializerConfig): String;
function ListToJSonArray<T: class>(const AList: TList<T>; const AConfig: TTJSonSerializerConfig): TJSonArray;
```

```delphi
LCustomer := FJSonContext.CreateEntity<TCustomer>();
LCustomer.CompanyName := 'Acme';
FJSonContext.Insert<TCustomer>(LCustomer);
LJson := FJSonContext.EntityToJSon<TCustomer>(LCustomer, LConfig);
```

The `*Object`/`*Array` variants hand you a `TJSonObject`/`TJSonArray` you own and must free (or add into a parent that takes ownership). Typical "count + data" envelope:

```delphi
LJSon := TJSonObject.Create;
try
  LJSon.AddPair('count', TJSonNumber.Create(Context.SelectCount<T>(AFilter)));
  LList := TTObjectList<T>.Create(True);
  try
    Context.Select<T>(LList, AFilter);
    LJSonData := Context.ListToJSonArray<T>(LList, LConfigSelect);
    try
      LJSon.AddPair('data', LJSonData);   // parent now owns LJSonData
    except
      LJSonData.Free;
      raise;
    end;
    FResponse.Content := LJSon.ToJSon();
  finally
    LList.Free;
  end;
finally
  LJSon.Free;
end;
```

## Deserialize

```delphi
function EntityFromJSon<T: class>(const AJSon: String): T;
function EntityFromJSonObject<T: class>(const AJSon: TJSonValue): T;
procedure ListFromJSon<T: class>(const AJSon: String; const AList: TList<T>);
procedure ListFromJSonArray<T: class>(const AJSon: TJSonArray; const AList: TList<T>);
```

`EntityFromJSon*` returns a **new entity that you own** - free it (or add it to an owning list). `ListFromJSon*` fills a caller-owned list (use `TTObjectList<T>.Create(True)` so the list frees its items).

```delphi
LRestored := FJSonContext.EntityFromJSon<TCustomer>(LJson);
try
  // use LRestored
finally
  LRestored.Free;
end;

LRestoredList := TTObjectList<TCustomer>.Create(True);
try
  FJSonContext.ListFromJSon<TCustomer>(LJson, LRestoredList);
finally
  LRestoredList.Free;
end;
```

## Dataset & metadata

```delphi
function DatasetToJSon(const ADataset: TDataset): String;       // any TDataset → JSON
function MetadataToJSon<T: class>(): String;                    // entity schema → JSON
```

```delphi
LDataset := FJSonContext.CreateDataset('SELECT * FROM Customers');
try
  LJson := FJSonContext.DatasetToJSon(LDataset);
finally
  LDataset.Free;
end;

LMeta := FJSonContext.MetadataToJSon<TCustomer>();
```

## Excluding fields

`[TJSonIgnore]` (`Trysil.JSon.Attributes`) on a field omits it from serialization while keeping it mapped for the database:

```delphi
[TColumn('InternalCode')]
[TJSonIgnore]
FInternalCode: String;
```

## Reminders
- Free the context before the connection.
- `TTNullable<T>` fields serialize as `null` when unset.
- `Currency` fields serialize as an exact decimal number - four decimals, invariant format, so `1234.5678` stays `1234.5678` instead of coming out as a rebuilt float - and deserialize back the same way.
- `*Object`/`*Array` serializers and `EntityFromJSon*`/`ListFromJSon*` hand you objects you own - free them or add them to an owning parent/list.

---

# Trysil HTTP

REST hosting with attribute-based routing on top of the JSON module. `TTHttpContext` (`Trysil.Http.Context`) extends `TTJSonContext` (which extends `TTContext`), so controllers have the full ORM + JSON API. Read **trysil-orm** and **trysil-json** first - their APIs apply here.

**Which unit declares what** (in addition to the trysil-orm and trysil-json units):

| Type(s) | Unit |
|---|---|
| `TTHttpServer<C>` | `Trysil.Http` |
| `TTHttpController<C>` | `Trysil.Http.Controller` |
| `TTHttpContext` | `Trysil.Http.Context` |
| `TTHttpRequest`, `TTHttpResponse`, `TTHttpUser` | `Trysil.Http.Classes` |
| `[TUri]`, `[TGet]`, `[TPost]`, `[TPut]`, `[TDelete]`, `[TArea]`, `[TAuthorizationType]` | `Trysil.Http.Attributes` |
| `TTHttpAuthorizationType` | `Trysil.Http.Types` |
| CORS config (`FServer.CorsConfig`) | `Trysil.Http.Cors` |
| `TTHttpAuthenticationBasic`/`Bearer`/`Digest<C>` | `Trysil.Http.Authentication.{Basic,Bearer,Digest}` |
| `TTHttpJWT<P>` | `Trysil.Http.JWT` |
| `TTHttpJWTAbstractPayload`, `ETHttpJWTException` | `Trysil.Http.JWT.Payload` |
| `TTHttpJWTHS256Payload` | `Trysil.Http.JWT.Payload.HS256` |
| `TTHttpJWTRS256Payload` | `Trysil.Http.JWT.Payload.RS256` |
| `TTHttpJWTRSAPrivateKey`, `TTHttpJWTRSAPublicKey` | `Trysil.Http.JWT.RSAKey` |
| `TTHttpFilter<T>` | `Trysil.Http.Filter` |
| `TTMultiTenant<T>`, `TTTenantConfig` | `Trysil.Http.MultiTenant` |
| `TTHttpLogAbstractWriter` | `Trysil.Http.Log.Writer` |
| `ETHttpException` + `ETHttp*` subclasses | `Trysil.Http.Exceptions` |

## Architecture

- `TTHttpServer<C>` - the server. `C` is your **per-request context type** (holds the connection + `TTHttpContext`, injected into every controller).
- `TTHttpController<C>` - base controller. Has `FContext: C`, `FRequest: TTHttpRequest`, `FResponse: TTHttpResponse`.
- Routing comes from attributes on controller methods. One controller class can be registered under a URI prefix; generic controllers (`TController<T>`) let one class serve many entity types.

## 1. Per-request context (dependency injection target)

The server constructs one of these per request and passes it to the controller. It owns the connection and the `TTHttpContext`.

```delphi
TAPIContext = class
strict private
  FConnection: TTConnection;
  FContext: TTHttpContext;
public
  constructor Create;
  destructor Destroy; override;

  property Context: TTHttpContext read FContext;
end;

constructor TAPIContext.Create;
begin
  inherited Create;
  TTFireDACConnectionPool.Instance.Config.Enabled := True;
  FConnection := TTSqlServerConnection.Create(TAPIConfig.Instance.Database.ConnectionName);
  FContext := TTHttpContext.Create(FConnection);
end;

destructor TAPIContext.Destroy;
begin
  FContext.Free;
  FConnection.Free;
  inherited Destroy;
end;
```

## 2. Controllers & routing attributes

Attributes from `Trysil.Http.Attributes`:

| Attribute | Scope | Meaning |
|---|---|---|
| `[TUri('/prefix')]` | class | URI prefix for the controller |
| `[TGet]` / `[TGet('/sub')]` | method | maps GET (optionally a sub-path) |
| `[TPost]` / `[TPost('/sub')]` | method | maps POST |
| `[TPut]` / `[TPut('/sub')]` | method | maps PUT |
| `[TDelete]` / `[TDelete('/?/?')]` | method | maps DELETE |
| `[TArea('name')]` | method | required user area/permission |
| `[TAuthorizationType(TTHttpAuthorizationType.None)]` | class/method | bypass auth (e.g. login) - read on both, the method wins |

`?` segments are positional path parameters bound to the method's parameters in order. `/?` → one param, `/?/?` → two.

```delphi
TAPIReadOnlyController<T: class> = class(TAPIController)
public
  [TGet]
  [TArea('read')]
  procedure SelectAll;

  [TGet('/?')]
  [TArea('read')]
  procedure Get(const AID: TTPrimaryKey);

  [TPost('/select')]
  [TArea('read')]
  procedure Select;                 // body carries a JSON filter

  [TGet('/metadata')]
  [TArea('read')]
  procedure Metadata;
end;

TAPIReadWriteController<T: class> = class(TAPIReadOnlyController<T>)
public
  [TPost]
  [TArea('write')]
  procedure Insert;

  [TPut]
  [TArea('write')]
  procedure Update;

  [TDelete('/?/?')]
  [TArea('write')]
  procedure Delete(const AID: TTPrimaryKey; const AVersionID: TTVersion);
end;
```

Implementations use the injected context and write `FResponse.Content`:

```delphi
procedure TAPIReadWriteController<T>.Insert;
var
  LEntity: T;
begin
  LEntity := Context.EntityFromJSonObject<T>(FRequest.JSonContent);
  try
    if Context.GetID<T>(LEntity) <= 0 then
      Context.SetSequenceID<T>(LEntity);
    Context.Insert<T>(LEntity);
    FResponse.Content := Context.EntityToJSon<T>(LEntity, ConfigGet);
  finally
    LEntity.Free;
  end;
end;

procedure TAPIReadWriteController<T>.Delete(
  const AID: TTPrimaryKey; const AVersionID: TTVersion);
begin
  Context.Delete<T>(AID, AVersionID);
end;
```

## 3. Request / response (`Trysil.Http.Classes`)

**Request** - read input:
- `FRequest.JSonContent: TJSonValue` - parsed body. `FRequest.JSonContent.GetValue<String>('username', '')`.
- `FRequest.Parameters` - query params; `FRequest.Headers`; `FRequest.User` (username/password/areas).
- `FRequest.RemoteIP` - TCP peer address. `FRequest.ClientIP` - originating caller: same as `RemoteIP` for direct connections, but when the peer is loopback (local reverse proxy) it returns the last `X-Forwarded-For` entry, port and IPv6 brackets stripped. Use `ClientIP` for audit and rate limiting.

**Response** - write output:
- `FResponse.Content: String` - body (typically JSON).
- `FResponse.StatusCode`, `FResponse.ContentType`, `FResponse.AddHeader(name, value)`, `FResponse.ContentStream` (for binary).

## 4. Server bootstrap

```delphi
FServer := TTHttpServer<TAPIContext>.Create;
FServer.BaseUri := '/api';
FServer.Port := 8022;

FServer.CorsConfig.AllowHeaders := 'Content-Type, Authorization';
FServer.CorsConfig.AllowOrigin := '*';

TTSqlServerConnection.RegisterConnection('Api', 'Server', 'user', 'pwd', 'Db');

FServer.RegisterLogWriter<TAPILogWriter>();
FServer.RegisterAuthentication<TAPIAuthentication>();

FServer.RegisterController<TAPILogonController>();
FServer.RegisterController<TAPIReadWriteController<TCompany>>('/company');
FServer.RegisterController<TAPIReadWriteController<TEmployee>>('/employee');

FServer.Start;
// ...
FServer.Stop;
```

Routes produced by `TAPIReadWriteController<TCompany>` under `/company`:

```
GET    /company              -> SelectAll
GET    /company/123          -> Get(123)
POST   /company/select       -> Select  (JSON filter in body)
GET    /company/metadata     -> Metadata
POST   /company              -> Insert
PUT    /company              -> Update
DELETE /company/123/1        -> Delete(123, version 1)
```

## 5. CORS (`Trysil.Http.Cors`)

Set `FServer.CorsConfig.AllowHeaders` / `AllowOrigin`. The server adds CORS headers and handles OPTIONS preflight per registered controller automatically.

## 6. Authentication (`Trysil.Http.Authentication.*`)

Subclass the scheme you want, override the validation hooks, register one implementation with `RegisterAuthentication<H>()`. Mark public endpoints (login) with `[TAuthorizationType(TTHttpAuthorizationType.None)]`.

- **Basic** - `TTHttpAuthenticationBasic<C>`: override `IsValid(const AUser: TTHttpUser): Boolean`, set `Realm`.
- **Bearer/JWT** - `TTHttpAuthenticationBearer<C, P: TTHttpJWTAbstractPayload>`: override `CreatePayload` and `IsValid(const APayload: P)`.
- **Digest** - `TTHttpAuthenticationDigest<C>`: override `GetNonce`, `IsValidNonce`, `GetUserMD5`.

### JWT (`Trysil.Http.JWT`)

The payload is also the signer. Subclass the algorithm base class, never `TTHttpJWTAbstractPayload` (it only declares the contract):

| Base class | Unit | Overrides |
|---|---|---|
| `TTHttpJWTHS256Payload` | `Trysil.Http.JWT.Payload.HS256` | `GetSecret`, `ToJSon`, `FromJSon` |
| `TTHttpJWTRS256Payload` | `Trysil.Http.JWT.Payload.RS256` | `GetVerificationKey`, `GetSigningKey` (issuer only), `ToJSon`, `FromJSon` |

HS256 for a single app that issues and validates; RS256 when the issuer and the resource servers are separate, so verifiers hold only the public key.

RS256 keys are **objects, not PEM strings**: `TTHttpJWTRSAPrivateKey` (`Sign` + `Verify`) and `TTHttpJWTRSAPublicKey` (`Verify` only), from `Trysil.Http.JWT.RSAKey`. They parse the PEM in the constructor and hold it for their lifetime. Create them **once at startup** and return them from the payload's getters, which borrow them:

```delphi
// startup, owned by the application
FSigningKey := TTHttpJWTRSAPrivateKey.Create(LPrivatePem, '2026-07');

function TAPIJWTPayload.GetSigningKey: TTHttpJWTRSAPrivateKey;
begin
  result := TAPIConfig.Instance.SigningKey;
end;

function TAPIJWTPayload.GetVerificationKey(
  const AKeyID: String): TTHttpJWTRSAAbstractKey;
begin
  result := TAPIConfig.Instance.KeyFor(AKeyID);   // AKeyID = the token's kid
end;
```

Never construct a key inside the payload: a payload is per-request, so that parses RSA on every request. One key instance is safe to share across threads. Requires OpenSSL `libcrypto` at runtime (loaded dynamically when the first key is built, `ETHttpJWTException` if missing).

Key rotation: outgoing, override `GetSigningKeyID` (HS256) or pass the key ID to the key constructor (RS256, `Create(APem, AKeyID)`) and the payload emits it as `kid`. Incoming, the `kid` is an **argument, never payload state**: `Verify` receives it and forwards it to `GetSecretFor(AKeyID)` (HS256, defaults to `GetSecret`) or `GetVerificationKey(AKeyID)` (RS256, `nil` fails closed).

`ToJSon` / `FromJSon` define the claims: Trysil imposes no claim set, so validity (expiry) is the payload's own `IsValid`.

Encode/verify with `TTHttpJWT<P>`:

```delphi
LJWT := TTHttpJWT<TAPIJWTPayload>.Create(LPayload);
try
  LToken := LJWT.ToToken();          // create token at login
  // LJWT.LoadFromToken(AToken) returns False if signature/format invalid
finally
  LJWT.Free;
end;
```

Bearer auth example wiring:

```delphi
TAPIAuthentication = class(TTHttpAuthenticationBearer<TAPIContext, TAPIJWTPayload>)
strict protected
  function CreatePayload: TAPIJWTPayload; override;
  function IsValid(const APayload: TAPIJWTPayload): Boolean; override;
end;

function TAPIAuthentication.IsValid(const APayload: TAPIJWTPayload): Boolean;
begin
  result := APayload.IsValid;
  if result then
  begin
    FRequest.User.Username := APayload.Username;
    // copy areas into FRequest.User.Areas for [TArea] checks
  end;
end;
```

Do not hardcode JWT secrets in shipping code; read them from configuration.

## 7. Server-side filtering from JSON (`Trysil.Http.Filter`)

`TTHttpFilter<T>` turns a JSON body into a `TTFilter`, validating column names and conditions against entity metadata.

```delphi
procedure TAPIReadOnlyController<T>.Select;
var
  LHttpFilter: TTHttpFilter<T>;
begin
  LHttpFilter := TTHttpFilter<T>.Create(Context, FRequest.JSonContent);
  InternalSelect(LHttpFilter.Filter);
end;
```

Expected JSON shape:

```json
{
  "where":  [ { "columnName": "Name", "condition": "LIKE", "value": "%Test%" } ],
  "orderBy":[ { "columnName": "Name", "direction": "ASC" } ],
  "start": 0,
  "limit": 10
}
```

Allowed conditions: `=`, `<>`, `<`, `<=`, `>`, `>=`, `LIKE`, `NOT LIKE`. Directions: `ASC`, `DESC`.

**`[TNotFilterable]`** (`Trysil.Attributes`) marks an entity column as not reachable from the filter: `where` and `orderBy` on it answer 400. Every mapped column is filterable unless annotated, so put it on the columns a caller should not be able to probe - a password hash, an internal cost - remembering that `LIKE` plus the row count is enough to read a value one character at a time without ever seeing it serialized.

```delphi
[TColumn('Password')]
[TNotFilterable]
FPassword: String;
```

Values are **bound as typed parameters**, never inlined: each condition emits `:p0`, `:p1` and the value is converted from the column's `TFieldType` (a `[TGuid]` or `[TCurrency]` column is converted and bound as such, not as its raw field type). The column name that reaches the SQL text is the **canonical one from the metadata**, not the string the client sent. Anything that does not add up raises `ETHttpBadRequest` (400) at parse time - unknown column, operator outside the closed list, value incompatible with the column type, a non-object item inside `where` **or inside `orderBy`**, or `LIKE` on a non-string column.

**The payload cannot widen the query.** `TTHttpFilterParameters` carries the ceilings, and the two-argument `TTHttpFilter<T>.Create` applies `TTHttpFilterParameters.Defaults`: `MaxLimit` 1000, `MaxWhereConditions` 32, `MaxOrderByColumns` 8, `IncludeDeleted` False.

- `limit` is **clamped**, not trusted: absent, zero or negative means `MaxLimit`, and a larger value is capped to it. There is no "return the whole table" request a client can make. A negative `start` is normalized to `0`.
- More `where` conditions or `orderBy` columns than the maximum is a 400, not a slow query.
- **`includeDeleted` is not read from the payload.** Showing soft-deleted rows is a server-side decision: pass parameters that say so, after your own authorization check.

```delphi
LHttpFilter := TTHttpFilter<T>.Create(
  Context,
  FRequest.JSonContent,
  TTHttpFilterParameters.Create(200, 16, 4, UserMaySeeDeleted));
```

A negative ceiling means unlimited, for an internal export endpoint that genuinely needs it.

## 8. Multi-tenant (`Trysil.Http.MultiTenant`)

`TTMultiTenant<T: TTTenantConfig>` is a thread-safe singleton mapping a tenant name to a `TTTenant<T>` (its own config + connection). Subclass `TTTenantConfig` (override `GetConnectionName`, `GetParameters`) and build the `TTHttpContext` on `tenant.Connection.CreateConnection`.

Two ways in, and the choice matters:

- `TryGet(AName, ATenant): Boolean` - looks the tenant up under a read lock and **never constructs**. Use it for every read-only consumer: resolving the inbound host, log writers, support services.
- `GetOrAdd(AName)` - **creates** the tenant if missing, which reads and parses its configuration file. Use it only where creation is intended: after authentication on the request path, or at startup when building the registry of known tenants.

A `GetOrAdd` that cannot build the tenant always raises **`ETTenantUnavailable`**, carrying `TenantName` and `OriginalClassName` and keeping the original message - the underlying exception never reaches the caller, so the class you catch does not depend on timing. Failures are not memoized (deliberately - a repaired tenant must not stay broken until restart), but they are **rate limited**: a failed `GetOrAdd` records the name with a cooldown (`FailureCooldown`, 5000 ms by default, settable, `0` disables it) and repeats within that window fail without touching the disk. The cost of an anonymous caller rotating the `Host` header stops being a function of traffic, and an operator's repair takes effect within the cooldown rather than at restart. The failure table is capped at 128 names and expired entries are swept on every insert, so the client cannot grow it.

`Remove(AName)` detaches the tenant from the registry - `TryGet` and `GetAll` stop seeing it - but **does not destroy it**. Tenants handed out by `TryGet`/`GetOrAdd` are borrowed references, and a thread that resolved one an instant earlier may be about to use its connection; the instance is released with the registry. Reclaiming memory per tenant would need a use count, which is a design decision, not a call you can make from `Remove`.

In `GetParameters`, zero the record first: `Result := Default(TTFireDACConnectionParameters);`. A local record only initialises its managed fields.

## 9. Logging

Subclass `TTHttpLogAbstractWriter` (override `WriteAction`/`WriteRequest`/`WriteResponse`, optionally `WriteDiscarded`) and register with `RegisterLogWriter<W>()`. The writer can persist log rows through its own Trysil context.

Three registration overloads: no argument (one log thread), a thread pool size, or a `TTHttpLogParameters` record carrying `ThreadPoolSize`, `QueueCapacity`, `MaxContentLength` and `MaxItemCount` (negative = unlimited). The two shorter forms apply **finite defaults** - 64 KB of content, 128 items - so unlimited is something you ask for, not something you get by not asking. `Authorization`, `Proxy-Authorization`, `Cookie`, `Set-Cookie` and `X-Api-Key` reach the writer with their name and `<redacted>` instead of the value.

```delphi
FServer.RegisterLogWriter<TAPILogWriter>(
  TTHttpLogParameters.Create(4, 10000, 65536));

FServer.OnCanLog :=
  function(ARequest: TTHttpRequest): Boolean
  begin
    Result := TenantLogEnabled(ARequest.Host);
  end;
```

- **`OnCanLog`** is asked on the request thread **before** the log record is built. Returning `False` costs nothing: no body serialization, no copy, no queue, no INSERT. Assign it before `Start`; it runs on every request thread, so keep it thread-safe and resolve tenants with `TryGet`.
- **`LogRequest` runs before routing and authentication.** `WriteRequest` is invoked for 404s and 401s too, so in a per-database multi-tenant app an anonymous caller can push rows into the log database of whatever tenant it names in `Host`. `OnCanLog` is where you reject or divert them.
- **Bodies above `MaxContentLength` are not captured at all**, and the size is measured without touching the body. The omission is declared: `ToJSon` always writes `ContentLength`, and `ContentOmitted: true` replaces `Content`. `Params` follows `Content`, because with `application/x-www-form-urlencoded` Indy decodes the whole body into the parameters - the parameters *are* the body. `MaxItemCount` caps parameters and headers by count, which a byte cap does not do; `ParamsOmitted` and `HeadersOmitted` declare it, and the counts are always written.
- **Header lookup is case insensitive.** `ARequest.Headers.Value['Authorization']` matches whatever case the client sent, which matters because HTTP/2 mandates lower-case header names. Query parameters are **not** case insensitive: `ARequest.Parameters` keeps exact-match semantics, as the URL spec requires.
- **The queue has a cap.** When full the newest entry is rejected and counted per host; the log thread then reports it through `WriteDiscarded(ADiscarded: TTHttpLogDiscarded)`, which carries `Host` and `Count`. It is virtual but not abstract - existing writers still compile, and the default forwards to `WriteAction`. The host is sanitized and truncated before it becomes a key or reaches the report, distinct hosts are capped at 64 with the rest accumulating under `<other>`, and the counts are flushed on a timer as well as when the queue empties - under sustained load the queue never empties.
- **Unhandled errors have their own hook.** `WriteError(ALogError: TTHttpLogError)` receives the task id, host, uri, exception class and message, and the recorded nested class and message. Like `WriteDiscarded` it is virtual, not abstract, and its default forwards to `WriteAction`. The exception is rendered to strings **synchronously on the request thread** and only the strings are queued - the exception object does not survive the handler. It is not gated by `OnCanLog`: an error row is always worth writing.

## 10. Errors & HTTP status codes

The listener catches every exception centrally and turns it into the JSON response. The mapping is **narrow** - know it exactly:

- `ETHttpException` and its subclasses (`Trysil.Http.Exceptions`) carry a status code, used as-is:

  | Exception | Status |
  |---|---|
  | `ETHttpBadRequest` | 400 |
  | `ETHttpUnauthorized` | 401 |
  | `ETHttpForbidden` | 403 |
  | `ETHttpNotFound` | 404 |
  | `ETHttpMethodNotAllowed` | 405 |
  | `ETHttpConflict` | 409 |
  | `ETHttpInternalServerError` | 500 |
  | `ETHttpException.Create(code, msg)` | any code you pass |

- **Every other exception becomes HTTP 500**, and the body of any response of status 500 or above is a **constant plus the task id** - `{"status":500,"message":"Internal server error.","taskId":"..."}`. The listener routes by status code, not by class, so `raise ETHttpInternalServerError.Create(E.Message)` does not put that message on the wire either. The exception message never reaches the client on this path, and there is no serialized exception chain: the detail goes to the log writer's `WriteError`, and the `taskId` is what correlates the two - which means an application with no registered log writer has a 5xx with no detail anywhere. That includes the ORM's `ETValidationException`, `ETConcurrentUpdateException`, and `ETDataIntegrityException`. There is **no automatic ORM-to-HTTP mapping**. To return 400 on a failed validation or 409 on a version conflict, catch the ORM exception in the controller and re-raise as an `ETHttp*`:

```delphi
procedure TAPIReadWriteController<T>.Update;
var
  LEntity: T;
begin
  LEntity := Context.EntityFromJSonObject<T>(FRequest.JSonContent);
  try
    try
      Context.Update<T>(LEntity);
      FResponse.Content := Context.EntityToJSon<T>(LEntity, ConfigGet);
    except
      on E: ETValidationException do
        raise ETHttpBadRequest.Create(E.Message);            // 400
      on E: ETConcurrentUpdateException do
        raise ETHttpConflict.Create(E.Message);                // 409 Conflict
    end;
  finally
    LEntity.Free;
  end;
end;
```

- On success the status is always **200 OK** (the `201 Created` constant exists but is unused). Set `FResponse.StatusCode` yourself for a different success code.

## Lifecycle reminders
- One `TAPIContext` (connection + `TTHttpContext`) is created and destroyed **per request** - keep `Create` cheap (assignments/object creation only; heavier work in `AfterConstruction`).
- Always free entities you deserialize (`EntityFromJSonObject`) and lists you build inside handlers.
- `TTHttpContext` never uses the identity map (inherited constraint from `TTJSonContext`).
