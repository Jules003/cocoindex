# How Rust Gets Loaded Into Python — The Complete Source Code Chain

This document traces **every piece of source code** that makes the Rust ↔ Python bridge work, from build-time configuration all the way to a runtime function call crossing the boundary.

---

## The Chain at a Glance

```
Step 1: pyproject.toml        — tells Maturin "compile this Rust crate as a Python module"
Step 2: Cargo.toml             — tells Cargo "compile this crate as a C-compatible shared library (.so)"
Step 3: py/mod.rs              — Rust code that defines the Python module using PyO3 macros
Step 4: __init__.py            — Python loads the compiled .so via `from . import _engine`
Step 5: lib.py                 — Python calls `_engine.init()` which crosses into Rust
Step 6: py/mod.rs → init()     — Rust receives the call and initializes the engine
Step 7: lib_context.rs         — Rust creates DB pools, auth registries, and the runtime
```

---

## Step 1: Build Configuration — pyproject.toml

**File**: [pyproject.toml](../pyproject.toml)

This tells **Maturin** (the build tool) where the Rust code is and what name to give the compiled module:

```toml
[build-system]
requires = ["maturin>=1.10.0,<2.0"]
build-backend = "maturin"

[tool.maturin]
bindings = "pyo3"                              # Use PyO3 for the Python ↔ Rust bridge
python-source = "python"                       # Python source files live in ../python/
module-name = "cocoindex._engine"              # The compiled .so will be importable as cocoindex._engine
features = ["pyo3/extension-module"]           # Tells PyO3 to build as a CPython extension
manifest-path = "rust/cocoindex/Cargo.toml"    # Points to the Rust crate to compile
```

**What this does**: When you run `pip install .` or `maturin develop`, Maturin reads this file, runs `cargo build` on the Rust crate, and places the resulting `.so` file inside the [cocoindex/](../rust/cocoindex/src/py/mod.rs#609-651) package directory so Python can find it as `cocoindex._engine`.

---

## Step 2: Rust Crate Configuration — Cargo.toml

**File**: [rust/cocoindex/Cargo.toml](../rust/cocoindex/Cargo.toml)

This tells **Cargo** (Rust's build tool) to compile the crate as a C-compatible dynamic library:

```toml
[package]
name = "cocoindex"

[lib]
name = "cocoindex_engine"      # The internal name of the compiled library
crate-type = ["cdylib"]        # ← THIS IS THE KEY LINE

[dependencies]
pyo3 = { workspace = true }    # PyO3: the Rust ↔ Python bridge library
```

**What `cdylib` means**: In C terms, this is equivalent to compiling with `gcc -shared -o libcocoindex_engine.so`. It produces a `.so` (Linux), `.dylib` (Mac), or `.pyd` (Windows) file with standard C calling conventions. This is the format CPython expects for extension modules.

Without `cdylib`, Cargo would produce a `.rlib` (Rust's static library format), which Python can't load.

---

## Step 3: Defining the Python Module in Rust — py/mod.rs

**File**: [rust/cocoindex/src/py/mod.rs](../rust/cocoindex/src/py/mod.rs)

This is where Rust **declares what Python will see** when it imports [\_engine](../rust/cocoindex/src/py/mod.rs#609-651). The `#[pymodule]` macro generates all the C FFI boilerplate that CPython requires:

```rust
/// A Python module implemented in Rust.
#[pymodule(gil_used = false)]
#[pyo3(name = "_engine")]
fn cocoindex_engine(m: &Bound<'_, PyModule>) -> PyResult<()> {
    // Add a version string
    m.add("__version__", env!("CARGO_PKG_VERSION"))?;

    // Add functions (these become callable from Python as _engine.init(), etc.)
    m.add_function(wrap_pyfunction!(init_pyo3_runtime, m)?)?;
    m.add_function(wrap_pyfunction!(init, m)?)?;
    m.add_function(wrap_pyfunction!(set_settings_fn, m)?)?;
    m.add_function(wrap_pyfunction!(start_server, m)?)?;
    m.add_function(wrap_pyfunction!(stop, m)?)?;
    m.add_function(wrap_pyfunction!(register_source_connector, m)?)?;
    m.add_function(wrap_pyfunction!(register_function_factory, m)?)?;
    m.add_function(wrap_pyfunction!(register_target_connector, m)?)?;
    // ... more functions ...

    // Add classes (these become instantiable from Python)
    m.add_class::<builder::flow_builder::FlowBuilder>()?;
    m.add_class::<Flow>()?;
    m.add_class::<FlowLiveUpdater>()?;
    // ... more classes ...

    Ok(())
}
```

**What the macros generate**: The `#[pymodule]` macro generates a C function called `PyInit__engine` (following CPython's naming convention). When Python tries to `import _engine`, CPython's import machinery looks for this exact symbol in the `.so` file, calls it, and the function registers all the Rust functions/classes with the Python interpreter.

> [!NOTE]
> In C extension module terms, this is equivalent to writing the `PyInit_modulename` function by hand with `PyModule_Create()`, `PyModule_AddObject()`, etc. — but PyO3 auto-generates all of that from the macro.

---

## Step 4: Python Loads the Rust Module — \_\_init\_\_.py

**File**: [python/cocoindex/**init**.py](../python/cocoindex/__init__.py)

When **any** Python code does `import cocoindex`, Python executes this [**init**.py](../python/cocoindex/__init__.py). Two critical lines near the top:

```python
from . import _engine  # type: ignore    ← Loads the compiled Rust .so file
```

This single line triggers CPython to:

1. Search for [\_engine](../rust/cocoindex/src/py/mod.rs#609-651) in the [cocoindex/](../rust/cocoindex/src/py/mod.rs#609-651) package directory
2. Find `_engine.cpython-311-darwin.so` (the compiled Rust library)
3. Call the `PyInit__engine` C function inside it (generated by the `#[pymodule]` macro in Step 3)
4. The [\_engine](../rust/cocoindex/src/py/mod.rs#609-651) module is now live in Python's memory with all the Rust functions/classes registered

Then, later in the same file:

```python
_engine.init_pyo3_runtime()    ← First actual call into Rust! Initializes the Tokio async runtime
```

This calls the Rust function:

```rust
// In py/mod.rs
#[pyfunction]
fn init_pyo3_runtime() {
    pyo3_async_runtimes::tokio::init_with_runtime(get_runtime()).unwrap();
}
```

Which calls:

```rust
// In lib_context.rs
static TOKIO_RUNTIME: LazyLock<Runtime> = LazyLock::new(|| Runtime::new().unwrap());

pub fn get_runtime() -> &'static Runtime {
    &TOKIO_RUNTIME
}
```

This creates the **Tokio async runtime** — a thread pool that will execute all async Rust operations. In C terms, this is like calling `pthread_create()` to spin up a pool of worker threads that will handle all the heavy computation later.

---

## Step 5: Python Calls init() — lib.py

**File**: [python/cocoindex/lib.py](../python/cocoindex/lib.py)

When user code (or the CLI) calls `cocoindex.init()`:

```python
from . import _engine  # type: ignore

def init(settings: setting.Settings | None = None) -> None:
    """Initialize the cocoindex library."""
    _engine.init(prepare_settings(settings) if settings is not None else None)
```

[prepare_settings()](../python/cocoindex/lib.py#15-20) converts the Python [Settings](../rust/cocoindex/src/settings.rs#19-32) dataclass into a dict that Rust can deserialize:

```python
def prepare_settings(settings: setting.Settings) -> Any:
    """Prepare the settings for the engine."""
    return dump_engine_object(settings)  # Converts dataclass → dict via serde-compatible format
```

> [!NOTE]
> Even before [init()](../rust/cocoindex/src/py/mod.rs#44-51) is called, at **module import time**, [lib.py](../python/cocoindex/lib.py) also registers a settings callback with Rust:
>
> ```python
> _engine.set_settings_fn(lambda: prepare_settings(setting.Settings.from_env()))
> ```
>
> This stores a Python callable inside Rust that it can invoke later to get settings from environment variables.

---

## Step 6: Rust Receives the init() Call — py/mod.rs

**File**: [rust/cocoindex/src/py/mod.rs](../rust/cocoindex/src/py/mod.rs)

The `#[pyfunction]` macro exposes this Rust function to Python as `_engine.init()`:

```rust
#[pyfunction]
fn init(py: Python<'_>, settings: Pythonized<Option<Settings>>) -> PyResult<()> {
    py.detach(|| -> Result<()> {
        get_runtime().block_on(async move {
            init_lib_context(settings.into_inner()).await
        })
    })
    .into_py_result()
}
```

Breaking this down:

- `py: Python<'_>` — a token proving we hold the GIL (Python's Global Interpreter Lock)
- `Pythonized<Option<Settings>>` — PyO3 auto-deserializes the Python dict into a Rust [Settings](../rust/cocoindex/src/settings.rs#19-32) struct
- `py.detach(|| ...)` — **releases the GIL** so other Python threads can run while Rust works
- [get_runtime().block_on(...)](../rust/cocoindex/src/lib_context.rs#169-172) — runs the async [init_lib_context](../rust/cocoindex/src/lib_context.rs#359-368) on the Tokio thread pool
- `.into_py_result()` — converts any Rust `Error` into a Python exception

> [!IMPORTANT]
> The `Pythonized<T>` wrapper (defined in [py/convert.rs](../rust/cocoindex/src/py/convert.rs)) uses `pythonize::depythonize()` to convert a Python dict → Rust struct. This is the same principle as C's `fread()` reading bytes into a struct, but with automatic field name matching and type conversion.

---

## Step 7: Rust Initializes the Engine — lib_context.rs

**File**: [rust/cocoindex/src/lib_context.rs](../rust/cocoindex/src/lib_context.rs)

[init_lib_context()](../rust/cocoindex/src/lib_context.rs#359-368) creates the global singleton that holds all engine state:

```rust
pub(crate) async fn init_lib_context(settings: Option<Settings>) -> Result<()> {
    let settings = match settings {
        Some(settings) => settings,
        None => get_settings()?,   // ← calls back into Python to get settings from env!
    };
    let mut lib_context_locked = LIB_CONTEXT.lock().await;
    *lib_context_locked = Some(Arc::new(create_lib_context(settings).await?));
    Ok(())
}
```

Which calls [create_lib_context()](../rust/cocoindex/src/lib_context.rs#291-336):

```rust
pub async fn create_lib_context(settings: Settings) -> Result<LibContext> {
    // 1. Initialize logging
    LIB_INIT.get_or_init(|| {
        let env_filter = EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info"));
        let _ = tracing_subscriber::registry()
            .with(fmt::layer())
            .with(env_filter)
            .try_init();
    });

    // 2. Create database connection pools (if configured)
    let db_pools = DbPools::default();
    let persistence_ctx = if let Some(database_spec) = &settings.database {
        let pool = db_pools.get_pool(database_spec).await?;  // connects to Postgres
        // ... reads existing setup state from DB ...
        Some(PersistenceContext { builtin_db_pool: pool, /* ... */ })
    } else {
        None
    };

    // 3. Build the global context struct
    Ok(LibContext {
        db_pools,
        persistence_ctx,
        flows: Mutex::new(BTreeMap::new()),           // empty — flows get added later
        app_namespace: settings.app_namespace,
        global_concurrency_controller: /* ... */,
        multi_progress_bar: /* ... */,
    })
}
```

At this point, the engine is alive. [LibContext](../rust/cocoindex/src/lib_context.rs#243-254) is stored in a global static:

```rust
static LIB_CONTEXT: LazyLock<tokio::sync::Mutex<Option<Arc<LibContext>>>> =
    LazyLock::new(|| tokio::sync::Mutex::new(None));
```

Every subsequent operation (building flows, running evaluations, starting the server) reads from this global context.

---

## Bonus: How Data Crosses the Boundary — py/convert.rs

**File**: [rust/cocoindex/src/py/convert.rs](../rust/cocoindex/src/py/convert.rs)

Every time data moves between Python and Rust, it goes through conversion functions in this file:

**Python → Rust** (e.g., when you pass arguments to a transform):

```rust
pub fn value_from_py_object(typ: &ValueType, v: &Bound<'_, PyAny>) -> PyResult<Value> {
    if v.is_none() {
        return Ok(Value::Null);
    }
    match typ {
        ValueType::Basic(typ) => Ok(Value::Basic(basic_value_from_py_object(typ, v)?)),
        ValueType::Struct(schema) => Ok(Value::Struct(field_values_from_py_seq(&schema.fields, v)?)),
        ValueType::Table(schema) => { /* ... */ }
    }
}
```

**Rust → Python** (e.g., when returning results):

```rust
pub fn value_to_py_object(py: Python<'_>, v: &Value) -> PyResult<Bound<'_, PyAny>> {
    match v {
        Value::Null => Ok(py.None().into_bound(py)),
        Value::Basic(v) => basic_value_to_py_object(py, v),
        Value::Struct(v) => field_values_to_py_object(py, v.fields.iter()),
        Value::UTable(v) | Value::LTable(v) => { /* convert rows to Python list of tuples */ }
        Value::KTable(v) => { /* convert key-value pairs */ }
    }
}
```

In C terms, this is like marshalling data between two different representations — converting between Python's `PyObject*` heap-allocated objects and Rust's stack-allocated enums/structs.

---

## Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         BUILD TIME                                  │
│                                                                     │
│  pyproject.toml ──→ Maturin ──→ Cargo ──→ _engine.cpython-311.so   │
│  (module-name)       (build     (crate-type                         │
│                      tool)       = cdylib)                          │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    │ produced file placed in
                                    │ site-packages/cocoindex/
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         RUN TIME                                    │
│                                                                     │
│  import cocoindex                                                   │
│       │                                                             │
│       ▼                                                             │
│  __init__.py:  from . import _engine   ─────────────────────┐      │
│       │        CPython calls PyInit__engine()                │      │
│       │        inside the .so file                           │      │
│       │                                                      ▼      │
│       │                                          ┌──────────────┐  │
│       ▼                                          │ py/mod.rs    │  │
│  __init__.py:  _engine.init_pyo3_runtime() ────→ │  Tokio init  │  │
│                                                  └──────────────┘  │
│                                                                     │
│  cocoindex.init()                                                   │
│       │                                                             │
│       ▼                                                             │
│  lib.py:  _engine.init(settings_dict) ──────────────────────┐      │
│                                                              ▼      │
│                                                  ┌──────────────┐  │
│                                                  │ py/mod.rs    │  │
│                                                  │  init() fn   │  │
│                                                  │    │         │  │
│                                                  │    ▼         │  │
│                                                  │ lib_context  │  │
│                                                  │  .rs:        │  │
│                                                  │  DB pools    │  │
│                                                  │  Auth reg    │  │
│                                                  │  LibContext  │  │
│                                                  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```
