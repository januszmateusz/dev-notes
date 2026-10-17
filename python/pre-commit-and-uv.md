# Bootstrap repo Pythona: uv + pre-commit + ruff + pytest

## Start

```bash
git config --global init.defaultBranch main
uv init --package --name <pakiet> --python 3.12   # src layout, sam robi git init
uv add --dev ruff pytest pre-commit
uv run pre-commit autoupdate      # zawsze po wklejeniu cudzego configu
uv run pre-commit install
uv run pre-commit run --all-files
```

- **src layout**: testy importują zainstalowany pakiet, nie pliki z bieżącego katalogu.
- **`uv.lock` commituję w aplikacji.** W publikowanej bibliotece decyduje resolver
  projektu, który ją instaluje.

## `pyproject.toml`

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra"
```

## `.pre-commit-config.yaml`

```yaml
default_install_hook_types: [pre-commit, commit-msg]
default_stages: [pre-commit]      # bez tego hooki bez etapu działają 2x

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files
        args: [--maxkb=500]
      - id: detect-private-key
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff                  # w nowszych wersjach: ruff-check
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.4.20
    hooks:
      - id: uv-lock
  - repo: https://github.com/kynan/nbstripout
    rev: 0.7.1
    hooks:
      - id: nbstripout
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.4.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
        args: [feat, fix, refactor, test, docs, chore, ci]
```

Wersje `rev` to punkt startowy: `uv run pre-commit autoupdate` ustawia najnowsze.

## Zasady

- W pre-commit tylko szybkie hooki (sekundy). Ciężkie kontrole w CI.
- `Unstaged files detected... Stashing` to nie błąd: pre-commit chowa niezacommitowane
  zmiany, żeby sprawdzić dokładnie zawartość commita.
- Hook zmodyfikował plik = commit przerwany. `git add` i commit ponownie.
- `uv-lock` pilnuje zgodności `uv.lock` z `pyproject.toml`. Zgodność środowisk
  lokalnego i CI daje dopiero `uv sync --locked` w CI.
- `uv run ruff check <ścieżka>` przed commitem, zamiast czekać na hook.

## ruff: częste sytuacje

**E501 (za długa linia) w f-stringu**: formatter nie dzieli napisów. Dzielę ręcznie
na sąsiadujące napisy, które Python skleja przy kompilacji:

```python
header = (
    f"{'config':<22}{'run':>4}{'ttft':>8}"
    f"{'in':>6}{'out':>6}  answer"     # uwaga na spacje na granicy
)
```

Nie: `# noqa: E501` z wygody ani podnoszenie `line-length` dla całego repo.

**F401 w `__init__.py`**: ruff nie usuwa nieużywanego importu automatycznie, bo to może
być re-eksport. Jawny re-eksport: `__all__` albo `import x as x`.

**F401 w testach**: importuj tylko to, czego test używa, inaczej hook blokuje commit.

## Conventional Commits

Typy: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`.
Scope = moduł lub obszar: `feat(llm): ...`, `chore(deps): ...`, `docs(s01): ...`.
Jedna logiczna zmiana na commit. Zmiana promptu = osobny commit z wynikiem ewaluacji.