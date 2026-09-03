---
trigger: model_decision
description: Build REST APIs over Trysil entities with the Trysil.Http module - attribute-routed controllers, TTHttpServer bootstrap, dependency-injected context, CORS, Basic/Bearer/Digest/JWT authentication, multi-tenant hosting, and JSON request/response. Invoke when writing or reviewing Delphi HTTP/REST code that uses TTHttpController, TTHttpServer, TTHttpContext, or the Trysil.Http.* attributes.
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
| `TTHttpAuthenticationBasic`/`Bearer<C>` | `Trysil.Http.Authentication.{Basic,Bearer}` (`Digest` exists in `...Authentication.Digest` but is deprecated) |
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

**A route without `[TAuthorizationType]` requires authentication.** `Start` refuses to bind the socket if any route needs authentication and no authentication class was registered, so a `RegisterAuthentication` left inside a conditional branch fails loudly instead of serving every route anonymously. A server that really is fully public declares it in one line: `FServer.AllowAnonymous := True;` - which waives that check only, never the two below.

**`Start` also refuses two ways of writing an area that cannot work**: a route carrying `[TArea]` with no authentication class registered (`AllowAnonymous` or not - with no user there are no areas, so the attribute would restrict nothing), and a route carrying both `[TArea]` and `[TAuthorizationType(TTHttpAuthorizationType.None)]` (authentication does not run there, so its user carries no area and the route would answer 403 to everyone forever). An area declared on the controller class, or on an ancestor of an overridden method, is inherited; overloads of the same name do not share areas. The area check itself does not depend on an authentication class being present: with no user, a route carrying `[TArea]` answers 403.

- **Basic** - `TTHttpAuthenticationBasic<C>`: override `IsValid(const AUser: TTHttpUser): Boolean`, set `Realm`.
- **Bearer/JWT** - `TTHttpAuthenticationBearer<C, P: TTHttpJWTAbstractPayload>`: override `CreatePayload` and `IsValid(const APayload: P)`.
- **Digest** - `TTHttpAuthenticationDigest<C>`: **deprecated, do not use for new code.** It implements RFC 2069, the 1997 form: no `qop`, no `nc`, no `cnonce`, and a captured response stays replayable while its nonce is accepted. Use Bearer with JWT, or Basic over TLS.

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
- **`OnRedactContent`** is the hook for the bodies: `TFunc<TTHttpRequest, String, String>` receiving the request and the content, returning what gets logged. Headers were already redacted through a fixed list, bodies were not, so without it a JSON API logs the password of every `POST /login` and the token in every login response. It is applied to the request body and the response body, is `nil` by default, and like `OnCanLog` must be assigned before `Start`. Note the scope: it does **not** reach the query parameters, which go through the fixed header-name list (`Authorization`, `Proxy-Authorization`, `Cookie`, `Set-Cookie`, `X-Api-Key`), so a parameter called `token` or `password` is still logged in clear - keep secrets out of the URL.
- **A hook that raises cannot break the request.** `LogRequest`, `LogResponse`, `LogError` and `LogAction` swallow exceptions: logging must never break the operation being logged. What a hook returns is not required to be JSON either - a body that does not parse is logged as a string rather than disappearing from the entry.
- **`LogRequest` runs before routing and authentication.** `WriteRequest` is invoked for 404s and 401s too, so in a per-database multi-tenant app an anonymous caller can push rows into the log database of whatever tenant it names in `Host`. `OnCanLog` is where you reject or divert them.
- **Bodies above `MaxContentLength` are not captured at all**, and the size is measured without touching the body. The omission is declared: `ToJSon` always writes `ContentLength`, and `ContentOmitted: true` replaces `Content`. `Params` follows `Content`, because with `application/x-www-form-urlencoded` Indy decodes the whole body into the parameters - the parameters *are* the body. `MaxItemCount` caps parameters and headers by count, which a byte cap does not do; `ParamsOmitted` and `HeadersOmitted` declare it, and the counts are always written.
- **Header lookup is case insensitive.** `ARequest.Headers.Value['Authorization']` matches whatever case the client sent, which matters because HTTP/2 mandates lower-case header names. Query parameters are **not** case insensitive: `ARequest.Parameters` keeps exact-match semantics, as the URL spec requires.
- **A repeated header keeps the first occurrence.** Names now collapse regardless of case, and the first one to arrive wins - the later ones are dropped from the lookup and from the enumeration alike. Do not build anything on a second `Authorization` being visible; a proxy in front of the application should reject the duplicate before it gets here.
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
  | `ETHttpUnprocessableContent` | 422 |
  | `ETHttpInternalServerError` | 500 |
  | `ETHttpException.Create(code, msg)` | any code you pass |

- **Every other exception becomes HTTP 500**, and the body of any response of status 500 or above is a **constant plus the task id** - `{"status":500,"message":"Internal server error.","taskId":"..."}`. The listener routes by status code, not by class, so `raise ETHttpInternalServerError.Create(E.Message)` does not put that message on the wire either. The exception message never reaches the client on this path, and there is no serialized exception chain: the detail goes to the log writer's `WriteError`, and the `taskId` is what correlates the two - which means an application with no registered log writer has a 5xx with no detail anywhere. Two ORM exceptions are mapped for you, from 2.0.0: an `ETConcurrentUpdateException` that escapes a controller becomes **409** and an `ETValidationException` becomes **422**, both carrying the original message in the usual `status` / `message` body, both logged with `LogAction`. `ETDataIntegrityException` is **not** mapped and still becomes a 500. Catch them in the controller only when you want a different status:

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
