---
name: trysil-openapi
description: Describe a Trysil.Http REST API as an OpenAPI 3.1 document with the Trysil.OpenAPI module - TTOpenApi<C>.Register/RegisterAnonymous, TTOpenApiInfo, the [TTag]/[TSummary]/[TDescription]/[TParameter]/[TRequestBody]/[TResponse] attributes, TTOpenApiShape, the predefined TTOpenApiFilter/TTOpenApiMetadata/TTOpenApiError types and EntityClass on the controller. Invoke when writing or reviewing Delphi code that publishes or annotates an OpenAPI document for a TTHttpServer.
---

# Trysil OpenAPI

The `Trysil.OpenAPI` package (depends on `Trysil`, `Trysil.JSon`, `Trysil.Http`) builds an **OpenAPI 3.1.0** JSON document from the routes of a `TTHttpServer<C>` and serves it on `GET <BaseUri>/openapi.json`. There is no Swagger UI page: point any OpenAPI viewer or client generator at that URL. Read **trysil-http** first - controllers, routes, areas, authentication and Sqids are described there.

**Which unit declares what** (in addition to the trysil-orm, trysil-json and trysil-http units):

| Type(s) | Unit |
|---|---|
| `TTOpenApi<C>` | `Trysil.OpenAPI` |
| `TTOpenApiInfo`, `TTOpenApiShape`, `TTOpenApiParameterSource`, `TTOpenApiFilter`, `TTOpenApiMetadata`, `TTOpenApiError` | `Trysil.OpenAPI.Types` |
| `[TTag]`, `[TDescription]`, `[TSummary]`, `[TParameter]`, `[TRequestBody]`, `[TResponse]` | `Trysil.OpenAPI.Attributes` |
| `ETOpenApiException` | `Trysil.OpenAPI.Exceptions` |
| `TTHttpController<C>.EntityClass` | `Trysil.Http.Controller` |

`TTOpenApiShape` and `TTOpenApiParameterSource` are scoped enums: always write `TTOpenApiShape.List`, never `List`.

## 1. Activation

Register the document **after the last `RegisterController`** and **before `Start`**:

```delphi
uses
  Trysil.OpenAPI.Types,
  Trysil.OpenAPI;

FServer.RegisterAuthentication<TAPIAuthentication>();
FServer.RegisterController<TCompanyController>('/company');
FServer.RegisterController<TEmployeeController>('/employee');

TTOpenApi<TAPIContext>.Register(
  FServer, TTOpenApiInfo.Create('My API', '1.0'));

FServer.Start;
```

- `TTOpenApi<C>.Register(AServer, AInfo)` - the document is protected like any other route: the caller needs to authenticate (unless the server is `AllowAnonymous`).
- `TTOpenApi<C>.RegisterAnonymous(AServer, AInfo)` - the document is public.
- `TTOpenApiInfo` is a record: `Create(ATitle, AVersion)` or `Create(ATitle, AVersion, ADescription)`, read-only properties `Title`, `Version`, `Description`.
- The document is **built once, inside `Register`**, from the routes registered so far. A controller registered afterwards is not in it. The OpenAPI route itself is not described.

`Register` / `RegisterAnonymous` raise `ETOpenApiException` when:

- the server is already started;
- the document is already registered on this server;
- the document of the same context type `C` is already registered on another server - **one document per context type for the life of the process**, even after the first server is destroyed;
- the title or the version is empty;
- an attribute uses a shape without a type on a controller whose `EntityClass` is `nil` (section 3);
- a `[TRequestBody]` uses `TTOpenApiShape.Page`.

## 2. Attributes (`Trysil.OpenAPI.Attributes`)

| Attribute | Scope | Effect |
|---|---|---|
| `[TTag('Name')]` | class | the tag of every operation of the controller |
| `[TDescription('text')]` | class | the description of that tag |
| `[TSummary('text')]` | method | operation summary |
| `[TDescription('text')]` | method | operation description |
| `[TParameter('name', TTOpenApiParameterSource.Query, 'description')]` | method, repeatable | an optional string parameter, `Query` or `Header` |
| `[TRequestBody(TTOpenApiShape.Single, 'description')]` | method | request body = the entity of the controller (section 3) |
| `[TRequestBody(TSomeClass, TTOpenApiShape.Single, 'description')]` | method | request body of an explicit class |
| `[TResponse(200, 'description')]` | method, repeatable | a response with no body schema |
| `[TResponse(200, TTOpenApiShape.List, 'description')]` | method, repeatable | a response whose body is the entity of the controller |
| `[TResponse(200, TSomeClass, TTOpenApiShape.Single, 'description')]` | method, repeatable | a response whose body is an explicit class |

`TTOpenApiShape`:

| Value | Schema |
|---|---|
| `Single` | the object (`$ref`) |
| `List` | array of the object |
| `Page` | `{count: integer, data: array of the object}`, both required - the answer of `TTHttpEntityReader<T>.Select`/`SelectAll`. Not allowed in `[TRequestBody]`. |

**Predefined types** for the explicit-class form, matching what Trysil.Http reads and writes:

| Type | Schema |
|---|---|
| `TTOpenApiFilter` | the JSON filter of `TTHttpFilter<T>`: `where[]` (`columnName`, `condition` enum `=` `<>` `<` `<=` `>` `>=` `LIKE` `NOT LIKE`, `value`), `orderBy[]` (`columnName`, `direction` `ASC`/`DESC`), `start`, `limit` |
| `TTOpenApiMetadata` | the answer of `Metadata`: `entity`, `primaryKey`, `versionColumn`, `properties[]` (`name`, `type`, `size`, `precision`, `filterable`, `readable`, `writable`) |
| `TTOpenApiError` | the error body: `status`, `message`, `taskId` |

**Override rule.** For each attribute kind, the most derived declaration of the method that carries at least one attribute of that kind wins; `[TResponse]` attributes are replaced as a block, not merged. So write the responses **once on the generic base controller** and put `[TTag]` / class `[TDescription]` on the concrete controllers:

```delphi
TAPIReadOnlyController<T: class> = class(TAPIController)
public
  class function EntityClass: TClass; override;

  [TGet('/?')]
  [TArea('read')]
  [TSummary('Get by ID')]
  [TResponse(200, TTOpenApiShape.Single, 'The entity')]
  procedure Get(const AID: TTPrimaryKey);

  [TPost('/select')]
  [TArea('read')]
  [TSummary('Select with a filter')]
  [TRequestBody(TTOpenApiFilter, TTOpenApiShape.Single, 'Filter')]
  [TResponse(200, TTOpenApiShape.Page, 'Count and rows')]
  procedure Select;

  [TGet('/metadata')]
  [TArea('read')]
  [TResponse(200, TTOpenApiMetadata, TTOpenApiShape.Single, 'Metadata')]
  procedure Metadata;
end;

TAPIReadWriteController<T: class> = class(TAPIReadOnlyController<T>)
public
  [TPost]
  [TArea('write')]
  [TRequestBody(TTOpenApiShape.Single, 'Entity to insert')]
  [TResponse(200, TTOpenApiShape.Single, 'Inserted')]
  procedure Insert;

  [TDelete('/?/?')]
  [TArea('write')]
  [TResponse(200, 'Deleted')]
  [TResponse(409, 'The version changed')]
  procedure Delete(
    const AID: TTPrimaryKey;
    [TNotSqid] const AVersionID: TTVersion);
end;

[TTag('Companies')]
[TDescription('Customer companies')]
TCompanyController = class(TAPIReadWriteController<TCompany>);
```

`[TTag]` and the class `[TDescription]` are inherited, so a tag on the generic base would give every controller the same tag.

## 3. The entity of the controller - `EntityClass`

The forms without a class (`[TRequestBody(Shape, ...)]`, `[TResponse(Code, Shape, ...)]`) use `TTHttpController<C>.EntityClass`, a `class function EntityClass: TClass; virtual;` that returns `nil` by default. Override it **once in the generic base**:

```delphi
uses
  System.TypInfo;

class function TAPIReadOnlyController<T>.EntityClass: TClass;
begin
  result := GetTypeData(TypeInfo(T))^.ClassType;
end;
```

If `EntityClass` returns `nil`, a shape without a class raises `ETOpenApiException` at `Register`.

## 4. What is derived without attributes

- **Paths** from the routes. Each `?` becomes `{name}`, from the method parameter name without the `A` prefix, in camelCase (`AID` -> `id`, `AVersionID` -> `versionID`). Path parameters are required.
- **Same path, different parameter names**: OpenAPI forbids two templates that differ only in the names, so every route of a path uses the names of the **first registered** route (within one controller the order follows RTTI and is not predictable). Use the same parameter names on routes that share a path.
- **Path parameter type**: with Sqids on (`TTJSonSqids.Instance.UseSqids`, read at `Register`) `{type: string, minLength: 8}`; with `[TNotSqid]` or Sqids off, `integer` `int32`.
- **Security**: with an authentication registered, `components.securitySchemes` holds one `{type: http, scheme: <name>}` entry named after the lower-case scheme of the authentication class (`bearer`, `basic`, `digest`); protected routes get `security: [{<name>: []}]`. Anonymous routes (`[TAuthorizationType(TTHttpAuthorizationType.None)]`) always get `security: []`. Without authentication protected routes carry no `security`.
- **Implicit error responses**, schema `TTOpenApiError`:

  | Code | When |
  |---|---|
  | 500 | always |
  | 401 | the route is protected and the server has an authentication |
  | 403 | the route has `[TArea]` |
  | 404 | the path has a placeholder |
  | 400, 413, 422 | the operation has a `[TRequestBody]` |

  409 is never implicit: declare it. An explicit `[TResponse]` with the same code wins.

- A route without attributes gets only its path, parameters, security and implicit errors: no request body, no 200.
- Without `[TTag]` the operations have no tag.

## 5. Entity schemas (`components.schemas`)

An entity schema is the JSON the Trysil serializer writes, not the database table:

- Property names are the JSON names (camelCase). `[TJSonIgnore]` columns are absent, `[TJSonIgnoreSerialize]` columns are `writeOnly`, `[TJSonIgnoreDeserialize]` and change-tracking columns are `readOnly`; the version column is a plain integer.
- Types:

  | Delphi | Schema |
  |---|---|
  | `String` | `string` |
  | `Int16`, `Int32` | `integer` `int32` |
  | `Int64` | `integer` `int64` |
  | `Double`, `Extended` | `number` `double` |
  | `Currency` | `number` |
  | `Boolean` | `boolean` |
  | `TDateTime` / `TDate` / `TTime` | `string` `date-time` / `date` / `time` |
  | `TGuid` | `string` `uuid` |
  | `TBytes` | `string`, `contentEncoding: base64` |
  | `TTNullable<T>` | type `[<T>, "null"]` |
  | primary key | the id schema (integer, or Sqids string) |
  | `TTLazy<T>` field `FCustomer` | `customerID` (id schema) plus `customer` (`$ref`, `readOnly`) |
  | `TTLazyList<T>`, detail columns | array of `$ref` |

- Validation attributes (`Trysil.Validation.Attributes`) become constraints:

  | Attribute | Schema |
  |---|---|
  | `[TRequired]` | listed in `required` |
  | `[TDisplayName]` | `title` |
  | `[TEMail]` | `format: email` + `pattern` |
  | `[TMinLength]` / `[TMaxLength]` | `minLength` / `maxLength` |
  | `[TMinValue]` / `[TMaxValue]` | `minimum` / `maximum` |
  | `[TRange]` | `minimum` + `maximum` (inclusive) |
  | `[TGreater]` / `[TLess]` | `exclusiveMinimum` / `exclusiveMaximum` |
  | `[TRegex]` | `pattern` |

- The schema name is the class name. When two different classes share a class name, all of them use the qualified name (`Unit.TClass`) instead.

## 6. Limits

- One document per context type `C` for the life of the process.
- The document is built at `Register`: controllers registered later are not described.
- Placeholder names: the first route of a path wins.
- No automatic tags and no automatic success responses: declare them with attributes.
