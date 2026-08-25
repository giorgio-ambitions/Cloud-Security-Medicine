GitHub Code Review — DataRepo

Overall assessment: 8/10

The architecture is clean and conceptually strong:

    Catalog
       ↓
    Database
       ↓
    Table
       ↓
    NlkDataFrame

The main strength is the separation between the public data-access API and the underlying implementation. However, there are several areas where correctness, error handling, typing, and test coverage could be improved.

1. HIGH PRIORITY — Fix __getattr__ and _get_table()

Current code:

    def __getattr__(self, name: str):
        return self._get_table(name)

    def _get_table(self, name: str):
        table = getattr(self.db, name)

This can produce unexpected AttributeError behavior and can interfere with normal Python introspection.

Recommended implementation:

    def __getattr__(self, name: str):
        table = self._get_table(name)

        if table is None:
            raise AttributeError(
                f"{type(self).__name__} has no attribute {name!r}"
            )

        return table

    def _get_table(self, name: str) -> TableProtocol | None:
        table = getattr(self.db, name, None)

        if table is None or not hasattr(table, "table_metadata"):
            return None

        return cast(TableProtocol, table)

This preserves normal Python attribute semantics.

2. HIGH PRIORITY — Make the Table contract explicit

Currently, an object is effectively considered a table if it has:

    table_metadata

This is an implicit contract.

The TableProtocol should explicitly describe what a table is, for example:

    class TableProtocol(Protocol):
        table_metadata: TableMetadata

        def __call__(
            self,
            *args: Any,
            **kwargs: Any
        ) -> NlkDataFrame:
            ...

This makes the architecture easier to understand and improves static analysis.

3. MEDIUM PRIORITY — Review table discovery

The current implementation uses:

    for name in dir(self.db):
        table = self._get_table(name)

This is simple and may be perfectly adequate for normal Python modules.

However, it discovers tables by inspecting every attribute, including:

    functions
    classes
    constants
    imported modules
    Python internals

I would NOT immediately replace this with a registry.

Instead, first benchmark the current approach.

If discovery becomes expensive, introduce caching:

    self._table_cache: dict[str, TableProtocol] = {}

This preserves the current "code as catalog" design while reducing repeated lookups.

4. MEDIUM PRIORITY — Keep global argument precedence explicit

This code:

    new_kwargs = {**self._global_args, **kwargs}

is actually correct if the intended behavior is:

    global arguments
          ↓
    local arguments override globals

For example:

    global_args = {"limit": 100}

    db.table("measurements", limit=20)

results in:

    limit = 20

This behavior should be documented explicitly.

The real issue is whether global arguments are valid for every table. Validation should probably remain at the table/query layer rather than being duplicated in DatabaseWithGlobalArgs.

5. MEDIUM PRIORITY — Do not force one concrete Database implementation

Catalog.db() currently returns either:

    ModuleDatabase

or:

    DatabaseWithGlobalArgs

Both implement:

    Database

Therefore this is not necessarily a type-stability problem.

The public abstraction remains:

    Database

Keeping the wrapper conditional also avoids creating an unnecessary wrapper when no global arguments exist.

I would leave this unchanged unless there is a concrete debugging or typing problem.

6. LOW PRIORITY — Database Protocol implementation

The Database Protocol contains:

    def tables(...):
        return list(self.get_tables(...).keys())

This is not inherently wrong.

Protocols can contain concrete method implementations.

In this case, the implementation is useful because it avoids duplicating the same logic in every Database implementation.

I would keep it.

7. LOW PRIORITY — Clarify callable Table semantics

ModuleDatabase currently does:

    return tbl(*args, **kwargs)

This means Table objects are callable.

That is a valid design if intentional.

The important thing is to document the contract clearly:

    Table
      ├── metadata
      └── callable interface
               ↓
          NlkDataFrame

If tables are intended to behave like query factories, this is reasonable.

If they represent persistent table objects, a method such as:

    table.query(...)

or:

    table.load(...)

might be more conventional.

This should be decided based on the wider DataRepo architecture, not changed in isolation.

8. LOW PRIORITY — Improve deprecation warnings

Current:

    warnings.warn(
        f"The table '{name}' is deprecated",
        DeprecationWarning
    )

Better:

    warnings.warn(
        f"The table '{name}' is deprecated",
        DeprecationWarning,
        stacklevel=2,
    )

This makes the warning point closer to the user's code.

9. CatalogMetadata

CatalogMetadata currently contains:

    jupyterhub_url: str | None = None

The field is stored by Catalog but does not appear to be used in this file.

Before removing it, search the entire repository for:

    jupyterhub_url

It may be consumed by the static-site exporter or another component.

Do not remove it based only on this file.

10. Add caching if profiling justifies it

A simple cache could be:

    class ModuleDatabase(Database):

        def __init__(self, db: ModuleType):
            self.db = db
            self._table_cache = {}

Then _get_table() can reuse previously discovered tables.

However, caching should only be introduced after deciding whether modules can be dynamically modified after initialization.

11. Testing

The most important tests should focus on API behavior rather than implementation details.

Discovery:

    - empty module
    - module containing valid tables
    - module containing functions
    - module containing constants
    - module containing imported modules
    - deprecated tables

Lookup:

    - existing table
    - missing table
    - invalid attribute
    - dynamic attribute access

Invocation:

    - valid table arguments
    - invalid arguments
    - table factory exceptions

Global arguments:

    - global arguments only
    - local arguments only
    - local arguments override global arguments
    - incompatible global arguments
    - empty global arguments

Compatibility:

    db.table("table_name")
    db.table_name

If dynamic attribute access exists for backwards compatibility, both APIs should behave consistently.

12. Medical/production considerations

If this system is eventually used for clinical, neural, or other sensitive medical data, the data-access abstraction should not be confused with a security boundary.

A production medical-data architecture should additionally address:

    User identity
        ↓
    Authentication
        ↓
    Authorization
        ↓
    Dataset permissions
        ↓
    Audit logging
        ↓
    Data access
        ↓
    Analysis

Additional concepts may include:

    - data classification
    - access policies
    - audit trails
    - role-based permissions
    - encryption
    - provenance
    - data retention
    - regulatory compliance

These concerns should preferably be implemented as explicit layers rather than hidden inside Table or Database.

Recommended implementation order:

    PR #1 — Correctness
        - fix _get_table()
        - fix __getattr__()
        - add warning stacklevel
        - add missing-table tests

    PR #2 — API contracts
        - strengthen TableProtocol
        - improve typing
        - document global-argument semantics
        - document callable Table behavior

    PR #3 — Performance
        - benchmark discovery
        - add table caching if necessary
        - consider a registry only if profiling justifies it

    PR #4 — Production/medical readiness
        - authorization
        - audit trail
        - data classification
        - access policies
        - provenance

Final assessment:

    Architecture       8.5/10
    API                 8/10
    Type safety         7/10
    Error handling      6.5/10
    Discovery           7/10
    Testability         8/10
    Simplicity          9/10
    Scalability         8/10

The most important point is that I would NOT radically redesign the architecture.

The current model:

    Catalog
       ↓
    Database
       ↓
    Table
       ↓
    NlkDataFrame

is simple and powerful.

The best improvements are therefore incremental: make the contracts explicit, improve error handling, add tests, and only introduce registries or additional abstraction layers when there is evidence that they are needed.
