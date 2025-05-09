---
title: "Build a Polars Plugin Expression— Rust Snowball as a Demo"
date: 2025-05-06
draft: false
author: "tchou"
tags:
  - Polars Plugin
  - NLP
  - Snowball Stemmer
description: ""
toc: true
---

# **Why Build a Polars Plugin?**

Polars delivers lightning-fast performance and low memory usage—but its
full power depends on writing logic with its native expression system.

That is where **Polars plugins** come in: They let us write custom logic
(like text processing with a Rust stemmer) without sacrificing
performance. In this post, I will show how to build a plugin that brings
`rust-stemmers` into your Polars workflow as a native
expression—achieving both flexibility and speed.

## You will get

1.  A [Polars
    plugin](https://docs.pola.rs/user-guide/plugins/expr_plugins/#writing-the-expression)
    that implements the rust-stemmers in Rust. I will try to publish
    this plugin on [PyPI](https://pypi.org/) so you can easily install
    and use it without creating the plugin yourself.
2.  A step-by-step procedure showing how to build a Python package, from
    setting the environment to installation of the package on your
    computer.

## Note

- You can find the usage example in Section [Example](#example) if you
  are not interested in knowing how to build a plugin package yourself.
- You can find all the details, including the implementation of snowball
  stemmer, on this [Polars plugins
  tutorial](https://marcogorelli.github.io/polars-plugins-tutorial/).
- You can find my project on
  [GitHub](https://github.com/TsungHanChou/polars-snowball-stemmer).

# Workflow

Before diving into coding, it is essential to understand the whole
development process — from setting up the environment to building the
final package. Here’s a step-by-step breakdown:

1.  Set up configuration files
    1.  `Cargo.toml`: This defines the Rust crate, dependencies, and
        package metadata.
    2.  `pyproject.toml`: This is the Python-side configuration used for
        building and distributing the package.
2.  Implement core expressions in Rust: Write all the necessary
    functions in [expressions.rs](#expressions).
3.  Create the Rust-Python Bridge: Set up `lib.rs` to define how the
    Rust functions will be exposed to Python.
4.  Expose Functions in Python: In the Python module’s `__init__.py`
    file, import and expose the Rust-implemented functions. This allows
    Python users to interact with your package.
5.  Use the package in your code.

![Dependency flow in a Rust–Python hybrid project. From left to right,
configuration files define the build and integration logic, linking to
the Rust core where modules are composed and exposed. Rust
functionalities are then bridged to Python, enabling their use within
the Python environment. The arrows indicate the direction of dependency
and integration across components.](/images/rust_stemmer_image.png)

## Recommended Strategy for Rust–Python Integration

In practice, I recommend a **backward strategy**: Begin by identifying
the desired functionality and run time environment in Python.

### Goal

For this project, the goal is to call the `snowball_stem` function with:

- **Python**: version 3.13  
- **Polars**: version 1.26  
- **Script**: `file.py`

### Steps to Implement

1.  **Define** the `snowball_stem` function in Python side. Ensure the
    function **imports the corresponding Rust implementation** via the
    `__init__.py` file.
2.  Since `__init__.py` acts as the entry point for Python–Rust
    integration, it must:
    - Import relevant objects from `lib.rs`
    - Connect to the core logic defined in `expressions.rs`
3.  **Configure the development environment** to support:
    - Rust–Python interoperability
    - The required dependencies and versions

This structured approach ensures clear communication between Python and
Rust components and maintains compatibility with the intended runtime
environment.

# Preparation

Here, we need to set up our development tool. The easiest way is to
develop the package using
[cookiecutter](https://cookiecutter.readthedocs.io/en/stable/README.html#installation),
which provides all the necessary ingredients. [Rust](https://rustup.rs/)
must also be installed. The plugin expression is compiled in Rust for
optimal performance.

## cookiecutter

We generate new projects using cookiecutter templates, which provide
standardized framing for Python packages. We use the template
`cookiecutter-polars-plugins`, which is designed for a new plugin for
the Polars data frame library.

Follow the recommended installation:

``` python
# install cookiecutter
pip install pipx  
pipx install cookiecutter 
# create a new polars-plugin project
cookiecutter https://github.com/MarcoGorelli/cookiecutter-polars-plugins 
```

You will see these files under your project root. These are the
ingredients you need to make a package.

![Dependency flow in a Rust–Python hybrid project. From left to right,
configuration files define the build and integration logic, linking to
the Rust core where modules are composed and exposed. Rust
functionalities are then bridged to Python, enabling their use within
the Python environment. The arrows indicate the direction of dependency
and integration across components.](/images/rust_stemmer_image_1.png)

# Developing the package

## From python side

Set up `__init__.py` for our Python usage. The file marks a directory as
a Python package and can be used to run initialization code, expose
specific functions or classes, and control how the package is imported.
In our case, we will call snowball_stem in the Python code for stemming.

``` python
from __future__ import annotations
from pathlib import Path

from typing import TYPE_CHECKING

import polars as pl
from polars.plugins import register_plugin_function

from polars_snowball_stemmer._internal import __version__ as __version__

if TYPE_CHECKING:
    from polars_snowball_stemmer.typing import IntoExprColumn

LIB = Path(__file__).parent

def snowball_stem(expr: IntoExprColumn) -> pl.Expr:
    return register_plugin_function(
        args=[expr],
        plugin_path=LIB,
        function_name="snowball_stem",
        is_elementwise=True,
    )
```

## From rust side

There are two files under the `src` folder. One contains the main
functions we need in this package. The other contains how we interact
with Python code.

### `expressions.rs`

This Rust function defines a custom Polars expression that applies
Snowball stemming (via `rust-stemmers`) to a column of strings in a
Polars DataFrame — and is exported to Python using `pyo3-polars`

``` rust
use polars::prelude::*;
use pyo3_polars::derive::polars_expr;
use std::fmt::Write;
use rust_stemmers::{Algorithm, Stemmer};

#[polars_expr(output_type=String)]
fn snowball_stem(inputs: &[Series]) -> PolarsResult<Series> {
    let ca: &StringChunked = inputs[0].str()?;
    let en_stemmer = Stemmer::create(Algorithm::English);
    let out: StringChunked = ca.apply_into_string_amortized(|value: &str, output: &mut String| {
        write!(output, "{}", en_stemmer.stem(value)).unwrap()
    });
    Ok(out.into_series())
}
```

### `lib.rs`

It sets up the Rust-Python bridge and memory allocator so the custom
Polars expressions can be accessed and executed from Python.

- \[pymodule\]: Defines the `_internal` module that Python will import
  as `polars_snowball_stemmer._internal`
- \[global_allocator\]: Sets `PolarsAllocator` as the global allocator
  for this Rust crate.

``` rust
mod expressions;
use pyo3::prelude::*;
use pyo3_polars::PolarsAllocator;

#[pymodule]
fn _internal(_py: Python, m: &Bound<PyModule>) -> PyResult<()> {
    m.add("__version__", env!("CARGO_PKG_VERSION"))?;
    Ok(())
}

#[global_allocator]
static ALLOC: PolarsAllocator = PolarsAllocator::new();
```

## Environment setting

`Cargo.toml` is the configuration file for Rust. There are three parts
to specify:

- \[package\]: The package name, the project’s current version, and the
  Rust language edition being used.
- \[lib\]: The name of the library crate — this determines the output
  file name. `"cdylib"` tells Cargo to compile a **C-compatible dynamic
  library** (`.so` on Linux, `.dylib` on macOS, `.dll` on Windows).
- \[dependencies\]: External libraries (crates) in the project.

Below is a configuration for our project.

``` rust
[package]
name = "polars_snowball_stemmer"
version = "0.1.0"
edition = "2021"

[lib]
name = "polars_snowball_stemmer"
crate-type= ["cdylib"]

[dependencies]
polars = { version = "0.46.0", default-features = false } 
polars-lazy = "0.46.0"
polars-arrow = "0.46.0"
pyo3 = { version = "0.23.0", features = ["extension-module"] }
pyo3-polars = { version = "0.20.0", features = ["derive"] }
rust-stemmers = "1.2.0"
serde = { version = "1", features = ["derive"] }
```

The `pyproject.toml` file is a configuration file for Python projects
used to define build system requirements.

- \[build-system\]: tells Python tools how to build the project. We need
  `maturin` and `polars`. This project uses
  [**Maturin**](https://github.com/PyO3/maturin) as the build backend.
- \[project\]: We need to specify our Python package name, the
  requirements for the Python version, and the classifiers used by PyPI
  to categorize the package. dynamic = \[“version”\] means the version
  number is not hardcoded here.
- \[tool.maturin\]: Specify how it should expose the Rust code to
  Python. When we compile the Rust code into a Python module, call it
  `polars_snowball_stemmer._internal`. You can see this in
  `__init__.py`.
- \[\[tool.mypy.overrides\]\]: Mypy (a type checker for Python) will
  ignore missing type hints in the `polars.utils.udfs` module so it does
  not raise type-checking errors for it.

``` rust
[build-system]
requires = ["maturin>=1.0,<2.0", "polars>=1.3.0"]
build-backend = "maturin"

[project]
name = "polars-snowball-stemmer"
requires-python = ">=3.12"
classifiers = [
  "Programming Language :: Rust",
  "Programming Language :: Python :: Implementation :: CPython",
  "Programming Language :: Python :: Implementation :: PyPy",
]
dynamic = ["version"]

[tool.maturin]
module-name = "polars_snowball_stemmer._internal"

[[tool.mypy.overrides]]
module = "polars.utils.udfs"
ignore_missing_imports = true
```

# Building and installing the package

This part shows how to build and install the package on our computer.

1.  Build the package: `maturin build --release`. This step will
    generate a .whl (wheel) file stored in the target/wheels/ directory.
    We can use the file to install our package.
2.  Install the built wheel locally: `pip install target/wheels/*.whl`.
    Once installed, we can test your package, as shown in the example in
    the next section.

**Note**: You may encounter several errors at this stage, often due to
the environment not being specified adequately for each file. To avoid
these issues, ensure the environment is correctly configured before
proceeding.

``` python
maturin build --release
pip install target/wheels/*.whl
```

# Example

This example demonstrates how to implement a plugin function in Polars.
We start by creating a Polars DataFrame with a column of words, then
apply `snowball_stem('word')` to perform stemming, storing the results
in a new column `'b'`. This showcases how to integrate custom Rust-based
functionality directly into a Polars workflow.

``` python
import polars as pl
from polars_snowball_stemmer import snowball_stem

df = pl.DataFrame({
    'word': [
        "hopelessly", 
        "fearfulness", 
        "amazingly", 
        "careless", 
        "beautifully", 
        "kindness"
    ]
})

print(df.with_columns(b=snowball_stem('word')))
```

Here is the result.

``` python
shape: (6, 2)
┌─────────────┬──────────┐
│ word        ┆ b        │
│ ---         ┆ ---      │
│ str         ┆ str      │
╞═════════════╪══════════╡
│ hopelessly  ┆ hopeless │
│ fearfulness ┆ fear     │
│ amazingly   ┆ amaz     │
│ careless    ┆ careless │
│ beautifully ┆ beauti   │
│ kindness    ┆ kind     │
└─────────────┴──────────┘
```

One performance highlight from my project: given an input dataset of
shape **(292_940_036, 5)** with one of the columns requiring word
stemming, the entire processing pipeline completes in **3 minutes and 10
seconds**. This includes filtering stopwords, applying Snowball
stemming, deduplicating by identification, grouping by stem and class to
extract first-use dates and decade counts, computing total frequencies,
filtering for relevant terms, and finally, unnesting and sorting by
frequency. The entire process—from raw data import to final output—uses
**under 30 GB of memory**.

# Conclusion

This project showcases the power of extending Polars with custom
Rust-based expressions through plugin development. By integrating the
rust-stemmers library into a native Polars expression, we enable fast
and memory-efficient word stemming directly within the Polars pipeline.
The result is a plugin that adheres to Polars’ expression system,
preserving performance while expanding functionality.

From setting up the development environment using cookiecutter, building
and bridging Rust and Python via maturin, to packaging and publishing
the plugin, this walkthrough provides a complete guide to writing
performant, idiomatic Polars extensions. With a production-scale input
size of nearly 300 million rows processed in just over 3 minutes and
under 30 GB of memory, this example underscores how Rust-backed plugins
can unlock new dimensions of speed and scalability in Python data
workflows.

## Next steps

- 🗣️ **Extend the plugin** to support multiple stemming algorithms,
  parallel computing, or languages using `rust-stemmers`.
- 🧱 **Build a broader NLP pipeline**, combining this plugin with others
  for tokenization, lemmatization, or even embedding generation, all
  within the Polars ecosystem.

By combining the **performance of Rust** with the **Python**, Polars
plugins offer a powerful pattern for scalability, custom data
transformations. This example is just the beginning.
