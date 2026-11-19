---
name: trysil-binding
description: Show Trysil entities in a Delphi user interface with the Trysil.Binding package - TTBinding with [TBind]/[TBindLookup] for a single entity on plain VCL/FMX controls (Apply/Retrieve/Populate), list binding for lists of entities in a VCL TListView or TStringGrid and an FMX TStringGrid or TListView ([TBindColumn] by position, [TBindItem] by appearance object name, sorting, typed selection), and TTDataset<T> for lists of entities in data-aware controls (TDBGrid, TDBEdit), with ApplyUpdates/CancelUpdates, Predicate, Comparison, Locate/Lookup and CreateDetail master/detail. Invoke when writing or reviewing Delphi form code that edits Trysil entities.
---

# Trysil Binding

The `Trysil.Binding` package (requires `rtl`, `dbrtl`, `Trysil`; no `Vcl` or `Fmx` unit) puts entities on a form in two ways. Read **trysil-orm** first - the context, the mapping, sessions and lazy lists are described there.

| Need | Use |
|---|---|
| a list of entities in a VCL `TListView` or `TStringGrid`, to look at and select | `TTListViewBinding<T>`, `TTStringGridBinding<T>` (package `Trysil.Binding.Vcl`) |
| a list of entities in an FMX `TStringGrid` or `TListView`, to look at and select | `TTFmxStringGridBinding<T>`, `TTFmxListViewBinding<T>` (package `Trysil.Binding.Fmx`) |
| one entity on plain controls (`TEdit`, `TLabel`, `TComboBox`, VCL or FMX) | `TTBinding` |
| a list of entities in data-aware controls (`TDBGrid`, `TDBEdit`, `TDBNavigator`) | `TTDataset<T>` |

**Which unit declares what:**

| Type(s) | Unit |
|---|---|
| `TTBinding` | `Trysil.Binding` |
| `[TBind]`, `[TBindLookup]` | `Trysil.Binding.Attributes` |
| `TTBindingConverterRegistry`, `TTBindingConverterClass` | `Trysil.Binding.Converter` |
| `ETBindingPropertyNotFound`, `ETBindingConverterRequired`, `ETBindingLookupNotFound` | `Trysil.Binding.Exceptions` |
| `TTDataset<T>`, `TTDatasetDetailGetter<M, D>` | `Trysil.Dataset` |
| `TTPredicate<T>` | `Trysil.Generics.Collections` |
| `TComparison<T>` | `System.Generics.Defaults` |
| `TTLazyList<T>` | `Trysil.Lazy` |
| `[TBindColumn]`, `[TBindItem]` | `Trysil.Binding.Attributes` |
| `TTListViewBinding<T>` (VCL, package `Trysil.Binding.Vcl`) | `Trysil.Binding.Vcl.ListView` |
| `TTStringGridBinding<T>` (VCL, package `Trysil.Binding.Vcl`) | `Trysil.Binding.Vcl.StringGrid` |
| `TTFmxStringGridBinding<T>` (FMX, package `Trysil.Binding.Fmx`) | `Trysil.Binding.Fmx.StringGrid` |
| `TTFmxListViewBinding<T>` (FMX, package `Trysil.Binding.Fmx`) | `Trysil.Binding.Fmx.ListView` |
| `TTListBinding<T>`, `ETBindingColumnsNotFound`, `ETBindingColumnCount`, `ETBindingItemNotFound` | `Trysil.Binding.List`, `Trysil.Binding.Exceptions` |

## 1. TTBinding - one entity, plain controls

A **snapshot**, not a live binding: nothing is synchronized until you call a method.

| Method | Direction |
|---|---|
| `Populate` | source lists to lookup combos |
| `Apply` | entity to controls |
| `Retrieve` | controls to entity |

Declare the binding on the control fields of the form; both property names are in the attribute (`Text` on an edit, `Caption` on a label):

```delphi
{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}

uses
  Trysil.Binding.Attributes,
  Trysil.Binding;

type
  TCustomerForm = class(TForm)
    [TBind('CompanyName', 'Text')]
    EditCompanyName: TEdit;

    [TBind('CreatedAt', 'Caption')]
    LabelCreatedAt: TLabel;

    [TBindLookup('CountryID', 'Name', 'ID')]
    ComboCountry: TComboBox;
  end;
```

`TTBinding.Create(AEntity, AContainer)` scans the published fields of the container for the attributes; `TTBinding.Create(AEntity)` binds nothing and you call `Bind` / `BindLookup`:

```delphi
LBinding := TTBinding.Create(LCustomer, Self);
try
  LBinding.SetLookupSource('CountryID', LCountries);
  LBinding.Populate;
  LBinding.Apply;

  // ... the user edits the controls ...

  LBinding.Retrieve;
  FContext.Save<TCustomer>(LCustomer);
finally
  LBinding.Free;
end;
```

```delphi
LBinding := TTBinding.Create(LCustomer);
LBinding.Bind('CompanyName', EditCompanyName, 'Text');
LBinding.Bind('Price', EditPrice, 'Text', TMoneyBindingConverter);
LBinding.BindLookup('CountryID', ComboCountry, LCountries, 'Name', 'ID');
```

- The binding does not own the entity, the controls or the source lists.
- The source list of a lookup is given at run time with `SetLookupSource` (an attribute cannot carry it); `Repopulate(AEntityProperty, ANewList)` swaps it and refills the combo.
- **Lookup mode** follows the type of the entity property: a scalar foreign key (`CountryID: Integer`) matches and writes the `KeyProperty` value; an object property (a relation, e.g. `Customer: TCustomer`) matches and writes **by reference** and ignores `KeyProperty`.
- **Converters**: `Integer`, `Int64`, `Double`, `TDateTime`, `Boolean` to and from `String` are built in; `TCaption` and other string aliases need none. Others: `TTBindingConverterRegistry.Instance.RegisterConverter<S, T>(AConverterClass)` or the `Bind` overload with a converter class.
- **Nullable**: a `TTNullable<T>` is shown through its inner type; null clears a string control; an empty string comes back as null on `Retrieve`.
- Errors: `ETBindingPropertyNotFound` (no such property on entity or control), `ETBindingConverterRequired` (types differ and no converter), `ETBindingLookupNotFound` (`SetLookupSource`/`Repopulate` on a property not bound as lookup).

## 2. TTDataset<T> - a list of entities, data-aware controls

A `TDataSet` that reads its own entities and keeps the changes in a `TTSession<T>` until `ApplyUpdates`.

```delphi
uses
  Trysil.Dataset;

FCustomers := TTDataset<TCustomer>.Create(FContext);            // every row
// or TTDataset<TCustomer>.Create(FContext, LFilter);           // a TTFilter
DataSourceCustomers.DataSet := FCustomers;
FCustomers.Open;
```

- The context is borrowed and must outlive the dataset. The dataset has no owner: free it yourself (`FCustomers.Free` in `FormDestroy`).
- One field per mapped column, **named after the column** (`FieldByName('CompanyName')`). Field types come from the database column, except a `Currency` member (`ftCurrency`) and a `TGUID` member (`ftGuid`).
- A `TTLazy<T>` member is a field holding the related key; writing it changes the relation.
- Primary key, `[TVersionColumn]` and change tracking columns are **read-only fields**.

### Editing

```delphi
FCustomers.Edit;
FCustomers.FieldByName('CompanyName').AsString := 'Acme';
FCustomers.Post;            // validates the edited copy, keeps it in memory

FCustomers.ApplyUpdates;    // writes inserts/updates/deletes in one transaction, then reads the rows again
FCustomers.CancelUpdates;   // discards what was not applied, reads the rows again
```

- `Edit` works on a copy; `Post` runs the entity validation (`ETValidationException`) and copies the values.
- `Insert` / `Append` create the entity with `CreateEntity<T>`; without a `Comparison` the new row goes **at the end**.
- Nothing reaches the database before `ApplyUpdates`. After it the dataset is reopened on the same record.

### Filtering, sorting, searching in memory

```delphi
FCustomers.Predicate :=
  function(const ACustomer: TCustomer): Boolean
  begin
    result := ACustomer.City = 'Florence';
  end;

FCustomers.Comparison :=
  function(const ALeft, ARight: TCustomer): Integer
  begin
    result := CompareText(ALeft.CompanyName, ARight.CompanyName);
  end;

if FCustomers.Locate('CompanyName', 'ac', [loCaseInsensitive, loPartialKey]) then
  ...
LCity := FCustomers.Lookup('ID', 42, 'City');   // Null when not found, cursor unchanged
```

- `Predicate` hides rows from navigation, `RecordCount`, `Locate` and `Lookup`; `nil` shows them all. **`Filter`, `Filtered` and `OnFilterRecord` are not read** - never generate them for a `TTDataset`. To read fewer rows, open with a `TTFilter`.
- `Comparison` sorts on assignment, at every `Open` and after every `Post`; `nil` stops sorting.
- `OldValue` is not supported.

### Master and detail

The detail rows are the `TTLazyList<D>` of the master, mapped with `[TDetailColumn]`. The master entity must **expose the `TTLazyList<D>` itself** as a property:

```delphi
[TDetailColumn('ID', 'CategoryID')]
FProducts: TTLazyList<TProduct>;
...
property Products: TTLazyList<TProduct> read FProducts;
```

```delphi
FCategories := TTDataset<TCategory>.Create(FContext);
FCategories.Open;

FProducts := FCategories.CreateDetail<TProduct>(
  function(const ACategory: TCategory): TTLazyList<TProduct>
  begin
    result := ACategory.Products;
  end);
FProducts.Open;
```

- Returning anything but the mapped `TTLazyList<D>` raises `ETException` when the detail opens.
- The detail follows the master: when the master moves, the detail posts its edit, **discards its changes not applied**, and shows the new list. Apply before moving to keep them.
- `Insert` on the detail writes the master key into the detail column.
- `ApplyUpdates` on the **master** writes the master and every open detail in one transaction. `ApplyUpdates` on the detail writes the detail only.
- The detail is yours to free, before or after the master.
- Do not wire a `MasterSource` / `MasterFields`: use `CreateDetail<D>`.

## 3. List binding - a list of entities in a TListView or TStringGrid

Display and selection only, as a snapshot: edit through a detail form with `TTBinding`, or use `TTDataset<T>` for editable grids. Columns are designed in the IDE (`ViewStyle = vsReport`); the binding says what each one shows, in column order.

```delphi
// The TListView is a field of this form: attributes on the field
[TBindColumn('ID')]
[TBindColumn('OrderDate', 'dd/mm/yyyy')]
[TBindColumn('Customer.CompanyName')]
[TBindColumn('Total', '#,##0.00')]
OrdersListView: TListView;

FBinding := TTListViewBinding<TOrder>.Create(Self, OrdersListView);
```

```delphi
// The TListView is inherited from a base form: columns in code
FBinding := TTListViewBinding<TOrder>.Create(ListView);
FBinding.AddColumn('ID');
FBinding.AddColumn('OrderDate', 'dd/mm/yyyy');
FBinding.AddColumn('Customer.CompanyName');
```

```delphi
FContext.Select<TOrder>(FOrders, LFilter);
FBinding.Apply(FOrders);                 // any TEnumerable<TOrder>
LOrder := FBinding.Selected;             // typed, nil without selection
FBinding.Select(LOrder);
FBinding.Select(                         // after a reload: find the row by key
  function(const AOrder: TOrder): Boolean
  begin
    result := AOrder.ID = LID;
  end);
```

- A column is a property path; dots follow related objects, `TTLazy<T>` included (it loads: one query per row; prefer a JOIN entity for long lists). An empty link gives an empty cell; a null `TTNullable<T>` too.
- Format: `FormatDateTime` for dates, `FormatFloat` for numbers; without it the field-binding converters.
- The number of declared columns must equal the `TListView` columns: `ETBindingColumnCount`, none at all `ETBindingColumnsNotFound`.
- Header click sorts by value (numbers as numbers, dates as dates, empties last), a second click reverses; the caller's list is not reordered; `SortOnColumnClick := False` turns it off; `Sort(AColumn, ADescending)` from code.
- The entities stay the caller's: after reading or freeing the list, call `Apply` again.
- No events of its own: use `OnSelectItem` / `OnDblClick` of the `TListView` and read `Selected`. Never cast `ListView.Selected.Data` yourself.
- The binding takes `OnColumnClick` (calling a previous handler) and restores it when freed: free the binding before the `TListView`, e.g. in `FormDestroy`.

### Other list controls

Same `Create(Self, Control)` / `Create(Control)`, `Apply`, `Selected`, `Select`, `Sort`, `SortOnColumnClick`.

- **VCL `TStringGrid`** - `TTStringGridBinding<T>`, `[TBindColumn]` / `AddColumn` by position on the non-fixed columns (`ColCount - FixedCols`); rows go below the fixed rows. The titles are cells of the fixed row and the application writes them (`Grid.Cells[0, 0] := 'Date'`). An empty list leaves one empty row (`Selected` is nil); the grid always has a current row, so a `Select` with no match leaves it. A click on the fixed row sorts (`OnFixedCellClick`, `goFixedRowClick` turned on), no arrow.
- **FMX `TStringGrid`** - `TTFmxStringGridBinding<T>`, `[TBindColumn]` / `AddColumn` by position on the `TStringColumn` items designed in the IDE (with their `Header`). A header click sorts (`OnHeaderClick`, `TGridOption.HeaderClick` turned on), no arrow.
- **FMX `TListView`** - `TTFmxListViewBinding<T>`, bound **by name** to the objects of the item appearance, not by position:

```delphi
[TBindItem('CompanyName', 'Text')]          // standard appearance: Text, Detail
[TBindItem('City', 'Detail')]
CustomersListView: TListView;

[TBindItem('CompanyName', 'Name')]          // dynamic appearance: object names
[TBindItem('Balance', 'Balance', '#,##0.00')]
AccountsListView: TListView;

FBinding := TTFmxListViewBinding<TCustomer>.Create(Self, CustomersListView);
// in code: FBinding.AddItem('City', 'Detail');
```

  Every name must be a visible object of the item appearance, otherwise `ETBindingItemNotFound` (`Detail` on an appearance without detail fails too). Text objects only. No headers: sort from code with `Sort(AColumn, ADescending)`, the column being the position of the `[TBindItem]`. Never use `[TBindColumn]` on an FMX `TListView`.
