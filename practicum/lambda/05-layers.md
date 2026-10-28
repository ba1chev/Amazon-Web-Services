# Lab 05 — Lambda Layers

## Goals
- A shared dependency is packaged as a Layer and attached to a function
- The function imports from the Layer without bundling the library in its own deployment package
- Layer versioning is understood: updating a Layer publishes a new immutable version

---

## How Layers work

A Layer is a zip archive extracted into `/opt` before the function runs.
Python packages placed at `python/` inside the zip are automatically on `sys.path`.

```
layer.zip
└── python/
    └── requests/         ← importable as `import requests`
```

Up to **5 Layers** can be attached per function. Combined unzipped size (code + layers) must be ≤ 250 MB.

---

## Task 1 — Publish a Layer with `requests`

A Layer named `requests-layer` (version 1) exists, compatible with Python 3.12.

The zip has this structure:

```
requests-layer.zip
└── python/
    └── requests/
    └── certifi/
    └── charset_normalizer/
    └── idna/
    └── urllib3/
```

Build it locally:

```bash
mkdir -p layer/python
pip install requests -t layer/python --quiet
cd layer && zip -r ../requests-layer.zip python/ && cd ..
```

---

## Task 2 — Use the Layer in a function

A function `layer-lab` (Python 3.12) has `requests-layer` version 1 attached.

The deployment package contains **only** the handler file — `requests` is not included:

```python
import requests

def lambda_handler(event, context):
    r = requests.get("https://httpbin.org/get")
    return {"status": r.status_code, "origin": r.json().get("origin")}
```

When invoked, the function returns HTTP 200 and the caller's IP. Without the Layer attached, the invocation fails with `ModuleNotFoundError: No module named 'requests'`.

---

## Task 3 — Layer versioning

A second version of `requests-layer` exists, built with an additional package (e.g., `certifi`).

`layer-lab` is updated to use **version 2**.

Functions that still reference version 1 are unaffected — Layer versions are immutable. Both versions coexist.

The Layer versions list shows:

| Version | Description |
|---------|-------------|
| 1 | requests only |
| 2 | requests + certifi |

---

## Task 4 — Layer for shared internal code

A Layer named `internal-utils` exists with this structure:

```
internal-layer.zip
└── python/
    └── utils/
        ├── __init__.py
        └── helpers.py
```

```python
# python/utils/helpers.py
import logging

def get_logger(name):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    return logger
```

Any function with this Layer attached can do:

```python
from utils.helpers import get_logger
logger = get_logger(__name__)
```

without copying the code into its own package.

---

## Task 5 — Observe the size limit

A zip containing `pandas` and `numpy` exceeds 50 MB. A direct upload attempt is rejected. The zip must first be uploaded to S3, and the Layer is created by referencing the S3 object URL.

---

## Key facts to remember

| Property | Value |
|----------|-------|
| Max Layers per function | 5 |
| Extraction path | `/opt` |
| Python auto-path | `/opt/python` |
| Node.js auto-path | `/opt/nodejs/node_modules` |
| Max total unzipped size | 250 MB |
| Layer versions | Immutable — update = publish a new version |
| Sharing | Layers can be made public or shared with specific accounts |

---

## Clean up

`layer-lab` is deleted. Both versions of `requests-layer` and `internal-utils` are deleted.
