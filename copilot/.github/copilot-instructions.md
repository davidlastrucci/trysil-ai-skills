# Trysil - Delphi ORM

Instructions for using the Trysil ORM and its JSON and HTTP modules from Delphi.

---

# Trysil ORM

Trysil is a Delphi ORM. Entities are plain classes decorated with attributes; `TTContext` is the API; FireDAC is the data layer. Single-integer primary keys only, optimistic locking via a version column, RTTI required.

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

Declare the connection variable polymorphically as `TTConnection` (the abstract base), not as the concrete driver class.

## 2. Context

`TTContext` (`Trysil.Context`) is the entry point. Free the context before the connection.

```delphi
FConnection := TTSqlServerConnection.Create('Test');
FContext := TTContext.Create(FConnection);
// ... use ...
FContext.Free;
FConnection.Free;
```

Constructor overloads: `Create(AConnection)`, `Create(AConnection, AUseIdentityMap)`, `Create(AReadConnection, AWriteConnection)`, `Create(AReadConnection, AWriteConnection, AUseIdentityMap)`. The identity map is scoped to the context instance (safe for multi-tenant — no global state).

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

Key attributes: `[TTable('name')]`, `[TPrimaryKey]`, `[TColumn('col')]`, `[TVersionColumn]`, `[TSequence('seqID')]`, `[TRelation('Table','FK',OwnsObjects)]`, `[TWhereClause('sql')]` + `[TWhereClauseParameter('name', value)]` (compile-time constants only — for runtime filtering use `TTFilterBuilder<T>`).

Add `{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}` to units that use Trysil attributes — Trysil resolves attributes via RTTI, so a misspelled attribute name compiles silently otherwise; this turns the typo into a compile error.

### Type system (`Trysil.Types`)
- `TTPrimaryKey` = `Int32` (single-integer PKs only).
- `TTVersion` = `Int32` (optimistic locking).
- `TTNullable<T>` — generic nullable wrapper, **no default constructor**; uninitialized = null. Set with `TTNullable<T>.Create(value)`. Read via `.HasValue` / `.Value`.

### Mapping is cached
`TTMapper.Instance` is a global singleton converting classes to `TTTableMap` on first access.

## 4. CRUD

`SelectAll`/`Select` are **procedures** that fill a caller-owned list — they do not return one.

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

// Insert — use CreateEntity to get a valid sequence-assigned ID up front
LCustomer := FContext.CreateEntity<TCustomer>();
LCustomer.CompanyName := 'Acme';
FContext.Insert<TCustomer>(LCustomer);

// Update / Delete
FContext.Update<TCustomer>(LCustomer);
FContext.Delete<TCustomer>(LCustomer);

// Save = insert-or-update (tracked via TTNewEntityCache)
FContext.Save<TCustomer>(LCustomer);

// Batch — all three lists in one transaction
FContext.ApplyAll<TCustomer>(LInsertList, LUpdateList, LDeleteList);
```

Notes:
- `CreateEntity<T>` returns an entity with the sequence ID already assigned (never ID=0) — no post-`Save` FK patching needed.
- `Update<T>` rewrites the whole row (no per-field diff); it emits an UPDATE even if nothing changed.
- When the identity map is on, the context **owns** the entities it returns — do not free them yourself.
- To refresh an entity you already hold, prefer `Refresh<T>(entity)` over `Get<T>(id)`.

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

## 6. Lazy loading (`Trysil.Lazy`)

`TTLazy<T>` (N:1) and `TTLazyList<T>` (1:N). **Never** create or free these manually and never add a separate `FxxxID` field — the framework instantiates and releases them via RTTI and triggers reload when `.ID` is set. Expose plain-typed properties through getters.

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

## 7. TTSession<T> — Unit of Work (`Trysil.Session`)

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

Attribute pairs auto-populated by the resolver. `*At` fields are `TTNullable<TDateTime>`; `*By` fields are `String` and require `TTContext.OnGetCurrentUser: TFunc<String>` (empty string if unassigned).

| Attribute | Fired on |
|---|---|
| `[TCreatedAt]` / `[TCreatedBy]` | Insert |
| `[TUpdatedAt]` / `[TUpdatedBy]` | Update |
| `[TDeletedAt]` / `[TDeletedBy]` | Delete (soft) |

With `[TDeletedAt]` present, `Delete<T>` does **not** issue SQL DELETE — it UPDATEs `DeletedAt`/`DeletedBy` and bumps the version. All SELECTs add `DeletedAt IS NULL`. To include soft-deleted rows: `TTFilter.IncludeDeleted := True` or `.IncludeDeleted` on the builder.

## 9. JOIN queries (read-only)

`[TJoin(Kind, 'Table'[, 'Alias'][, 'SourceTableOrAlias'], 'SourceCol', 'TargetCol')]` (`TJoinKind` = `Inner`/`Left`/`Right`) plus the 2-arg `[TColumn('Alias','Col')]`. Join entities are read-only — `Insert`/`Update`/`Delete` raise `ETException`. `TTFilterBuilder` does not resolve join aliases; use `TTFilter.Create(whereClause)` with manually qualified column names.

## 10. Raw select

For SQL attributes can't express (subqueries, UNION, GROUP BY, aggregates), map raw results to a DTO whose fields carry only `[TColumn('ResultColName')]`:

```delphi
Context.RawSelect<TOrderSummary>(
  'SELECT c.CompanyName AS CustomerName, SUM(o.Amount) AS Total ' +
  'FROM Orders o JOIN Customers c ON o.CustomerID = c.ID GROUP BY c.CompanyName',
  LResult);
```

Read-only, no identity map, no lazy loading.

## Architectural facts worth knowing
- `TTContext` delegates reads to `TTProvider` and writes to `TTResolver`.
- Validation attributes (`Trysil.Validation.Attributes`) collect into `TTValidationErrors`, raising `ETValidationException`. Custom validators via `[BeforeInsert]`/`[BeforeUpdate]` methods.
- Events: Before/After Insert/Update/Delete, declared via attributes.
- `TTUpdateMode`: `KeyAndVersionColumn` (default, optimistic lock) vs `KeyOnly` (table without `[TVersionColumn]`).
- Interfaces in Delphi cannot have generic methods, so `TTContext`/`TTProvider`/`TTResolver` are concrete classes by necessity — test via SQLite in-memory, not mocks.

---

# Trysil JSON

`TTJSonContext` (`Trysil.JSon.Context`) extends `TTContext`, so it has the full ORM API (`Select`, `Insert`, `CreateEntity`, ...) plus JSON serialization. Read the **trysil-orm** skill for entities, connections, and CRUD — all of it applies here.

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

A **record** in `Trysil.JSon.Types` — has no default; always construct it explicitly.

```delphi
constructor Create(const AMaxLevels: Integer; const ADetails: Boolean);
```

- `AMaxLevels` — max nesting depth for related entities; `-1` = unlimited, `0` = none (relations emitted as IDs only).
- `ADetails` — include detail (1:N) collections.

```delphi
LConfig := TTJSonSerializerConfig.Create(-1, False);  // defaults: unlimited depth, no details
LConfigGet    := TTJSonSerializerConfig.Create(1, True);   // one level deep, with details
LConfigSelect := TTJSonSerializerConfig.Create(1, False);
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

`EntityFromJSon*` returns a **new entity that you own** — free it (or add it to an owning list). `ListFromJSon*` fills a caller-owned list (use `TTObjectList<T>.Create(True)` so the list frees its items).

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
- `*Object`/`*Array` serializers and `EntityFromJSon*`/`ListFromJSon*` hand you objects you own — free them or add them to an owning parent/list.

---

# Trysil HTTP

REST hosting with attribute-based routing on top of the JSON module. `TTHttpContext` (`Trysil.Http.Context`) extends `TTJSonContext` (which extends `TTContext`), so controllers have the full ORM + JSON API. Read **trysil-orm** and **trysil-json** first — their APIs apply here.

## Architecture

- `TTHttpServer<C>` — the server. `C` is your **per-request context type** (holds the connection + `TTHttpContext`, injected into every controller).
- `TTHttpController<C>` — base controller. Has `FContext: C`, `FRequest: TTHttpRequest`, `FResponse: TTHttpResponse`.
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
| `[TAuthorizationType(TTHttpAuthorizationType.None)]` | class/method | bypass auth (e.g. login) |

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

**Request** — read input:
- `FRequest.JSonContent: TJSonValue` — parsed body. `FRequest.JSonContent.GetValue<String>('username', '')`.
- `FRequest.Parameters` — query params; `FRequest.Headers`; `FRequest.User` (username/password/areas); `FRequest.RemoteIP`.

**Response** — write output:
- `FResponse.Content: String` — body (typically JSON).
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

- **Basic** — `TTHttpAuthenticationBasic<C>`: override `IsValid(const AUser: TTHttpUser): Boolean`, set `Realm`.
- **Bearer/JWT** — `TTHttpAuthenticationBearer<C, P: TTHttpJWTAbstractPayload>`: override `CreatePayload` and `IsValid(const APayload: P)`.
- **Digest** — `TTHttpAuthenticationDigest<C>`: override `GetNonce`, `IsValidNonce`, `GetUserMD5`.

### JWT (`Trysil.Http.JWT`)

Define a payload by subclassing `TTHttpJWTAbstractPayload` (override `GetSecret`, `ToJSon`, `FromJSon`). Encode/verify with `TTHttpJWT<P>`:

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

## 8. Multi-tenant (`Trysil.Http.MultiTenant`)

`TTMultiTenant<T: TTTenantConfig>` is a thread-safe singleton mapping a tenant name to a `TTTenant<T>` (its own config + connection). Subclass `TTTenantConfig` (override `GetConnectionName`, `GetParameters`); in your per-request context resolve the tenant (e.g. from a header) via `TTMultiTenant<T>.Instance.GetOrAdd(name)` and build the `TTHttpContext` on `tenant.Connection.CreateConnection`.

## 9. Logging

Subclass `TTHttpLogAbstractWriter` (override `WriteAction`/`WriteRequest`/`WriteResponse`) and register with `RegisterLogWriter<W>()`. The writer can persist log rows through its own Trysil context.

## Lifecycle reminders
- One `TAPIContext` (connection + `TTHttpContext`) is created and destroyed **per request** — keep `Create` cheap (assignments/object creation only; heavier work in `AfterConstruction`).
- Always free entities you deserialize (`EntityFromJSonObject`) and lists you build inside handlers.
- `TTHttpContext` never uses the identity map (inherited constraint from `TTJSonContext`).
