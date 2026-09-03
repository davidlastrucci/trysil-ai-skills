---
name: trysil-json
description: Serialize and deserialize Trysil entities to/from JSON using the Trysil.JSon module. Invoke when writing or reviewing Delphi code that uses TTJSonContext, TTJSonSerializerConfig, EntityToJSon/ListToJSon/EntityFromJSon, DatasetToJSon, or MetadataToJSon, or that exposes Trysil entities over JSON.
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
- `ADetails` - include detail (1:N) collections. It is a switch, not a depth: `AMaxLevels` bounds detail collections as well, so `Create(0, True)` emits no details at all and `Create(1, True)` emits one level of them.

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
procedure EntityFromJSon<T: class>(const AJSon: String; const AEntity: T);
procedure EntityFromJSonObject<T: class>(const AJSon: TJSonValue; const AEntity: T);
procedure ListFromJSon<T: class>(const AJSon: String; const AList: TList<T>);
procedure ListFromJSonArray<T: class>(const AJSon: TJSonArray; const AList: TList<T>);
```

`EntityFromJSon*` returns a **new entity that you own** - free it (or add it to an owning list). `ListFromJSon*` fills a caller-owned list (use `TTObjectList<T>.Create(True)` so the list frees its items).

The two-argument overloads fill an entity you already loaded instead of building a fresh one. **Use them for an update endpoint**: `Update<T>` writes the whole row, so with a fresh entity every column absent from the request body is written back blank. Filling a loaded entity keeps what the body does not mention.

```delphi
LEntity := FJSonContext.Get<TCustomer>(AID);
try
  FJSonContext.EntityFromJSonObject<TCustomer>(FRequest.JSonContent, LEntity);
  FJSonContext.Update<TCustomer>(LEntity);
finally
  LEntity.Free;
end;
```

Three rules for that overload: the identity map is forbidden in a `TTJSonContext`, so `Get<T>` hands back an entity nobody owns and the `try..finally` is mandatory; the semantics are replace-all, not merge, so a `TTNullable` column absent from the body becomes NULL; and if the entity carries a `TTLazyList<T>` detail, the list is cleared and its already-loaded items are **freed**, so take references to detail entities after the call, never before.

Change tracking columns (`[TCreatedAt]`, `[TCreatedBy]`, `[TUpdatedAt]`, `[TUpdatedBy]`, `[TDeletedAt]`, `[TDeletedBy]`) are skipped by every deserialization entry point: they are the framework's to write, and a value for them in a request body is ignored. If you build a response from a fresh entity you just updated, call `Refresh<T>` first or those columns come back blank.

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
- `MetadataToJSon<T>` reports each column as `name`, `type` and, when they are not zero, `size` and `precision`. On a decimal column the names mislead: **`size` is the scale**, `precision` the total number of digits, so `decimal(19,4)` comes back as `"size": 4, "precision": 19`. A client validating an amount needs both.
- `Currency` fields serialize as an exact decimal number - four decimals, invariant format, so `1234.5678` stays `1234.5678` instead of coming out as a rebuilt float - and deserialize back the same way.
- `*Object`/`*Array` serializers and `EntityFromJSon*`/`ListFromJSon*` hand you objects you own - free them or add them to an owning parent/list.
