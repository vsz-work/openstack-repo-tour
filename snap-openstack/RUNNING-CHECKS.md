# Running sunbeam-python checks locally

All commands run from `snap-openstack/sunbeam-python/`. They need only `uv`
(`sudo snap install astral-uv --classic`). The environments are defined in
`sunbeam-python/tox.ini`.

## Through tox (the same checks CI runs)

`uvx` runs tox in a throwaway environment, so you don't need to install tox.

```sh
uvx tox -e unit,pep8,mypy     # everything CI checks
uvx tox -e unit               # unit tests only
uvx tox -e pep8               # ruff format --diff + ruff check
uvx tox -e mypy               # type checking
uvx tox -e fmt                # auto-fix: import sorting + formatting
uvx tox -e cover              # unit tests with coverage report (HTML in cover/)
```

Pass extra arguments to pytest after `--`:

```sh
uvx tox -e unit -- tests/unit/sunbeam/core/test_deployments.py
uvx tox -e unit -- -k backup
```

## Direct commands (faster when iterating)

### Unit tests

```sh
# all unit tests
uv run --frozen --isolated --extra=dev python -m pytest -q tests/unit

# one file
uv run --frozen --isolated --extra=dev python -m pytest -q tests/unit/sunbeam/provider/test_commands.py

# tests matching a name, verbose, without the pydantic deprecation warnings
uv run --frozen --isolated --extra=dev python -m pytest -vv -p no:warnings tests/unit -k backup
```

### Lint and format

CI checks `sunbeam-python` and the sibling `snap-wrappers`, `tools` and
`.github/scripts` directories.

```sh
# check only (what the pep8 env does)
uvx ruff format --diff . ../snap-wrappers ../tools ../.github/scripts
uvx ruff check . ../snap-wrappers ../tools ../.github/scripts

# fix (what the fmt env does); narrow to specific files if you like
uvx ruff check --select I --fix <paths>   # import sorting
uvx ruff format <paths>
```

### Type checking

```sh
uv run --frozen --isolated --extra=dev mypy sunbeam
uv run --frozen --isolated --extra=dev mypy tests/functional
uv run --frozen --isolated --extra=dev mypy ../tools
```

Unit tests are not type-checked yet (see the TODO in `tox.ini`).

## What the uv flags mean

- `--frozen`: install exactly what `uv.lock` pins, without re-resolving.
- `--isolated`: use a temporary environment, so your system Python isn't touched.
- `--extra=dev`: include the dev dependencies (pytest, mypy, ...).

## Typical pre-push routine

```sh
uvx tox -e fmt                # fix formatting first
uvx tox -e unit,pep8,mypy     # then run what CI runs
```
