# PyMuPDF Lambda Layer (`pymupdf-layer.zip`)

An AWS Lambda **layer** that packages the [PyMuPDF](https://pymupdf.readthedocs.io/) PDF library (a Python binding for the MuPDF engine) so the PDF compression function can `import pymupdf` without bundling the library inside the function itself. It is used by `PDF-Compressor-Function` in the parent project (S3 → Lambda → S3 PDF compression pipeline).

---

## Table of Contents

1. [Package Facts](#package-facts)
2. [Compatibility](#compatibility)
3. [Folder Structure](#folder-structure)
4. [File-by-File Reference](#file-by-file-reference)
5. [Importing: `pymupdf` vs `fitz`](#importing-pymupdf-vs-fitz)
6. [Publishing and Attaching the Layer](#publishing-and-attaching-the-layer)
7. [Smoke Test](#smoke-test)
8. [Rebuilding the Layer](#rebuilding-the-layer)
9. [Licensing](#licensing)
10. [Troubleshooting](#troubleshooting)

---

## Package Facts

| Property | Value |
|---|---|
| Library | PyMuPDF **1.28.2** (MuPDF **1.28.2**) |
| Installed with | `pip`, into a virtual environment |
| Wheel tag | `cp310-abi3-manylinux_2_28_x86_64` |
| CPU architecture | **x86_64** (all four `.so` files are 64-bit x86-64 ELF) |
| Python version | Built with **Python 3.14** (the bundled `.pyc` files are `cpython-314`). Package metadata requires Python 3.10 or newer |
| Archive size | 25.2 MB zipped, 59.0 MB (about 56 MiB) uncompressed |
| Contents | 40 entries (directories and files) |
| License | GNU AGPL 3.0 **or** Artifex commercial license (see [Licensing](#licensing)) |

## Compatibility

| Lambda setting | Required value |
|---|---|
| Architecture | `x86_64` (an `arm64` function cannot load these binaries) |
| Runtime | Python 3.14 (matches the build). Newer or older Python versions may work because the wheel is `abi3`, but the tested pairing is 3.14 |
| Layer path | Files must live under a top-level `python/` directory. Lambda extracts layers to `/opt`, and `/opt/python` is on the Python import path |
| Size budget | The layer uses about 56 MiB of Lambda's **250 MiB** combined uncompressed limit for the function plus all layers |

## Folder Structure

```
pymupdf-layer.zip
└── python/
    ├── pymupdf/                  # the PyMuPDF package (Python + compiled code)
    │   ├── __init__.py
    │   ├── __main__.py
    │   ├── _apply_pages.py
    │   ├── _build.py
    │   ├── _extra.so
    │   ├── _mupdf.so
    │   ├── _table_headers.py
    │   ├── _table_refine.py
    │   ├── _table_spans.py
    │   ├── _table_union.py
    │   ├── _wxcolors.py
    │   ├── extra.py
    │   ├── libmupdf.so.28.2
    │   ├── libmupdfcpp.so.28.2
    │   ├── mupdf.py
    │   ├── py.typed
    │   ├── pymupdf.py
    │   ├── table.py
    │   └── utils.py
    ├── pymupdf-1.28.2.dist-info/ # pip install metadata
    │   ├── COPYING
    │   ├── INSTALLER
    │   ├── METADATA
    │   ├── README.md
    │   ├── RECORD
    │   ├── REQUESTED
    │   ├── WHEEL
    │   └── entry_points.txt
    ├── fitz/                     # legacy import name (deprecated alias)
    │   ├── __init__.py
    │   ├── table.py
    │   ├── utils.py
    │   └── __pycache__/          # 3 compiled .pyc files (cpython-314)
    └── bin/
        └── pymupdf               # command-line launcher script
```

## File-by-File Reference

### `python/pymupdf/`: the library

| File | Size | Role |
|---|---|---|
| `__init__.py` | 924 KB | Main Python implementation of the PyMuPDF API (documents, pages, geometry classes, text and image extraction, saving and optimization). `import pymupdf` loads this file |
| `mupdf.py` | 2.7 MB | Python bindings for the MuPDF C++ API, generated with SWIG 4.4.1 |
| `_mupdf.so` | 15.5 MB | Compiled extension module behind `mupdf.py`; the bridge between Python and the MuPDF libraries |
| `libmupdf.so.28.2` | 36.2 MB | The **MuPDF C core library**: the engine that actually parses, renders and rewrites PDFs. By far the largest file in the layer |
| `libmupdfcpp.so.28.2` | 3.1 MB | MuPDF **C++ wrapper library** used by the bindings |
| `_extra.so` | 250 KB | Small compiled extension for performance-critical routines |
| `extra.py` | 6.7 KB | Python glue for `_extra.so` |
| `utils.py` | 39.7 KB | Higher-level helper functions used by the Page and Document classes |
| `table.py` | 109 KB | Table detection and extraction |
| `_table_headers.py`, `_table_refine.py`, `_table_spans.py`, `_table_union.py` | 15 to 39 KB each | Internal support modules for the table feature |
| `_apply_pages.py` | 7.7 KB | Internal helper for applying a function across the pages of a document |
| `_wxcolors.py` | 20.5 KB | Table of named colors used by color-name lookups |
| `__main__.py` | 41.4 KB | Implementation of the `python -m pymupdf` command-line interface |
| `_build.py` | 471 B | Build metadata recorded by the wheel build: version 1.28.2, git commit, MuPDF source URL, SWIG version |
| `pymupdf.py` | 64 B | Tiny shim that re-exports the package (`from . import *`) |
| `py.typed` | 0 B | Empty marker telling type checkers the package ships type information (PEP 561) |

### `python/pymupdf-1.28.2.dist-info/`: pip metadata

| File | Role |
|---|---|
| `METADATA` | Package name, version, author (Artifex), license, `Requires-Python >= 3.10`, project links and long description |
| `WHEEL` | Wheel format and platform tag (`cp310-abi3-manylinux_2_28_x86_64`) |
| `RECORD` | Pip's list of every installed file with hashes and sizes |
| `INSTALLER` | Contains `pip`, recording which tool installed the package |
| `REQUESTED` | Empty marker showing the package was installed explicitly rather than as a dependency |
| `entry_points.txt` | Declares the `pymupdf` console command (`pymupdf.__main__:main`) |
| `COPYING` | Licence notice: dual-licensed AGPL 3.0 or Artifex commercial |
| `README.md` | PyMuPDF's own project README |

### `python/fitz/`: legacy compatibility

| File | Role |
|---|---|
| `__init__.py` | Re-exports everything from `pymupdf` and logs a **deprecation warning** on import |
| `utils.py`, `table.py` | One-line re-exports of `pymupdf.utils` and `pymupdf.table` |
| `__pycache__/*.pyc` (3 files) | Python 3.14 bytecode caches. Safe to delete; Python regenerates them |

### `python/bin/`

| File | Role |
|---|---|
| `pymupdf` | Console launcher generated by pip. Its first line points at a Python interpreter path on the machine where the layer was built, so it is **not usable on Lambda**. Harmless, and can be removed. Lambda only imports from `python/` |

## Importing: `pymupdf` vs `fitz`

PyMuPDF used to be imported as `fitz`. Both names work with this layer:

```python
import pymupdf            # recommended, current name
import pymupdf as fitz    # use if your code still refers to "fitz."
import fitz               # legacy name; works but logs a deprecation warning
```

The instructor-supplied `lambda_function.py` and this layer are a matching pair, so use whichever import the code already uses. If you write new code, prefer `import pymupdf`.

## Publishing and Attaching the Layer

At 25 MB zipped, the file is under Lambda's 50 MB direct-upload limit, so uploading it straight from your computer should work without staging it in S3.

**Console:** Lambda, *Layers, Create layer*, name `pymupdf-layer`, upload the ZIP, choose `x86_64` and Python 3.14, create. Then open the function, *Layers, Add a layer, Custom layers*, and select it. Creating a layer does not attach it automatically.

**AWS CLI:**

```bash
aws lambda publish-layer-version \
  --layer-name pymupdf-layer \
  --zip-file fileb://pymupdf-layer.zip \
  --compatible-runtimes python3.14 \
  --compatible-architectures x86_64 \
  --region us-east-1

# Attach the returned LayerVersionArn to the function
aws lambda update-function-configuration \
  --function-name PDF-Compressor-Function \
  --layers <LayerVersionArn> \
  --region us-east-1
```

## Smoke Test

Create a temporary Lambda test (or add temporarily to the handler) to confirm the layer loads:

```python
import pymupdf

def lambda_handler(event, context):
    return {"pymupdf_version": pymupdf.__version__}
```

Expected: a response containing `1.28.2`. If this fails, fix the layer before debugging the compression logic.

## Rebuilding the Layer

If you need to reproduce the layer (for example after a PyMuPDF update), install the Linux x86_64 wheel into a `python/` folder and zip it:

```bash
mkdir -p build/python && cd build
pip install pymupdf==1.28.2 -t python \
  --platform manylinux_2_28_x86_64 --only-binary=:all: \
  --python-version 3.14 --implementation cp
rm -rf python/bin python/fitz/__pycache__    # optional trimming
zip -r ../pymupdf-layer.zip python
```

Building on Linux (or Amazon Linux 2023 in a container) avoids shipping Windows or macOS binaries by mistake.

## Licensing

PyMuPDF and MuPDF are **dual-licensed**: GNU AGPL 3.0, or a commercial license from Artifex. This ZIP contains the full library, including compiled MuPDF binaries. Practical consequences (this is general information, not legal advice):

- **Do not commit the ZIP or its binaries to a public GitHub repository** unless you are prepared to meet the AGPL's redistribution terms. Prefer committing this README and the rebuild instructions, and add `*.zip` to `.gitignore`.
- The AGPL has obligations for software offered to users over a network. For anything beyond a learning lab, check the terms or consider Artifex's commercial license.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `No module named 'pymupdf'` or `'fitz'` | Layer not attached to the function, or the ZIP lacks the top-level `python/` folder |
| `invalid ELF header` or a shared-library import error | The function is `arm64`, but this layer is `x86_64`. Recreate the function as `x86_64` |
| Import errors that mention the Python version | Function runtime differs from the layer's build (Python 3.14). Switch the runtime or rebuild the layer |
| Layer rejected for size | Combined function plus layers must stay under 250 MiB uncompressed. This layer uses about 56 MiB. Check the function package and other layers |
| `fitz` deprecation warning in CloudWatch logs | Harmless. Switch the code to `import pymupdf` to remove it |
