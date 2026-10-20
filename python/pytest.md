# pytest — sesja 1: fundamenty

## Model działania
- Test = funkcja; brak wyjątku = PASS, wyjątek = FAIL. Bez dziedziczenia.
- Assertion rewriting: import hook przepisuje `assert` przy imporcie (AST)
  → introspekcja wartości i diffy. Dotyczy: test_*.py, *_test.py, conftest.py, pluginy.
- Własne helpery z `assert` → `pytest.register_assert_rewrite("helpers")`
  w conftest.py (musi być PRZED pierwszym importem modułu).

## Discovery (ciche pominięcia!)
- Pliki: `test_*.py` / `*_test.py`; funkcje: prefiks `test` (też `testwaw`).
- Klasy: prefiks `Test`, BEZ `__init__` (pytest tworzy instancję sam,
  nowa na każdą metodę) → dane do testów przez fixtures.
- Plik podany jawnie w CLI jest zbierany mimo złej nazwy.
- Weryfikacja: `uv run pytest --collect-only -q`

## Wyniki i fazy
- `.` pass, `F` wyjątek w ciele testu (call), `E` wyjątek w setup/teardown
  (fixtures), `ERROR collecting` błąd importu modułu (przerywa całą sesję).
- Typ wyjątku nie ma znaczenia — liczy się faza. Dużo E = infrastruktura testów.
- Exit codes: 0 OK, 1 porażki, 2 przerwanie, 5 nie zebrano testów (CI = fail, celowo).

## Diffy
- Konwencja: `assert actual == expected` → `+` = actual, `-` = expected.
- dict: Differing items; list: At index N diff; set: Extra items. `-v`/`-vv` = pełny diff.

## Projekt
- src layout: kod importowalny tylko po instalacji (editable, `[build-system]`)
  → testy widzą pakiet jak produkcja; brak "działa, bo leży w CWD".
- `uv init --lib --name <pkg>` (nazwa modułu nie może zaczynać się od cyfry).
- Zawsze `uv run pytest`, nie gołe `pytest`.

## Konfiguracja startowa
[tool.pytest.ini_options]
minversion = "8.0"              # starszy pytest = błąd (wersję gwarantuje uv.lock)
testpaths = ["tests"]           # jedno źródło prawdy o zakresie testów
pythonpath = ["tests"]          # jawny wyjątek: importy helperów
addopts = ["-ra", "--strict-markers", "--strict-config", "--import-mode=importlib"]
# --strict-markers: literówka w markerze = błąd
# --strict-config: literówka w kluczu konfiguracji = błąd
# importlib: brak modyfikacji sys.path; testy nie importują się nawzajem

## CLI
-v/-q, -x, -s, -k "a and not b" (dopasowuje też nazwy modułów/klas),
node id: plik.py::Klasa::test, --lf / --ff (stan w .pytest_cache → .gitignore)

## Ochrona przed "testami, które nie istnieją"
- CI: `uv run pytest` bez ścieżek i filtrów.
- filterwarnings = ["error::pytest.PytestCollectionWarning"]
- coverage z --cov-fail-under (wykrywa skutek, niezależnie od przyczyny).