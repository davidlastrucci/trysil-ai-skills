---
name: trysil-json
description: Serialize and deserialize Trysil entities to/from JSON using the Trysil.JSon module. Invoke when writing or reviewing Delphi code that uses TTJSonContext, TTJSonSerializerConfig, EntityToJSon/ListToJSon/EntityFromJSon, DatasetToJSon, or MetadataToJSon, or that exposes Trysil entities over JSON.
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
