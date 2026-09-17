# The GIL Is Optional. Your Dependencies Aren’t.

Companion lab for the free-threading / GIL article.

Python 3.14 made the free-threaded build officially supported ([PEP 779](https://peps.python.org/pep-0779/)). It is still a **separate interpreter** (`3.14t`), not a flag on default 3.14. Threads only speed up CPU-bound work if every native extension has opted in. Import a C module that hasn't, and the GIL comes back on for the rest of the process.

This repo is the copy-paste lab: a stdlib-only threaded benchmark, a GIL detector, and a tiny unmarked C extension that reproduces the silent re-enable.

## Requirements

- [uv](https://docs.astral.sh/uv/)
- Linux or macOS with a free-threaded CPython build available (`uv python list 3.14t`)

```bash
uv python install 3.14 3.14t
```

`.python-version` pins `3.14+freethreaded` (same build as `3.14t`) so `uv run` in this project cannot silently fall back to the GIL build. `requires-python` in `pyproject.toml` stays `>=3.14` — freethreaded is a build variant, not a language version.

## Run the benchmark

Same CPU-bound workload, 1 thread vs N threads, two interpreters. Use `--isolated` and `3.14+gil` so the pin in `.python-version` does not swallow the GIL comparison.

```bash
uv run --no-project --isolated --python 3.14+gil python benchmark.py
uv run --no-project --isolated --python 3.14t python benchmark.py
```

Or the whole lab:

```bash
chmod +x run_lab.sh
./run_lab.sh
```

The workload is pure-Python parse-and-fold on purpose. `hashlib` and NumPy can release the GIL internally, which would fake a speedup on default 3.14.

## Audit imports

```bash
uv run --python 3.14t python gil_detector.py
uv run --python 3.14t python gil_detector.py json hashlib
uv run --python 3.14t --with numpy --with pandas --with pydantic --with fastapi \
  python gil_detector.py numpy pandas pydantic fastapi
```

## The gotcha

`trap/` is a C extension that does **not** declare `Py_MOD_GIL_NOT_USED`. Importing it on 3.14t re-enables the GIL:

```bash
uv run --no-project --isolated --python 3.14t --with ./trap python gil_detector.py gil_trap
```

Track ecosystem wheels at [py-free-threading](https://py-free-threading.github.io/tracking/) and [free-threaded wheels](https://hugovk.github.io/free-threaded-wheels/).

## Docker

```bash
docker build -t python-threads-gil .
docker run --rm python-threads-gil
```

## Layout

| Path | Role |
| --- | --- |
| [`benchmark.py`](./benchmark.py) | Stdlib-only 1 vs N thread CPU benchmark |
| [`gil_detector.py`](./gil_detector.py) | Import modules; watch for a GIL flip |
| [`trap/`](./trap/) | Unmarked C extension that re-enables the GIL |
| [`Dockerfile`](./Dockerfile) | uv + 3.14t image |
| [`run_lab.sh`](./run_lab.sh) | One-shot GIL vs 3.14t + trap demo |
