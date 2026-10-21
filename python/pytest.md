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

# pytest — sesja 2: dobre testy

## Struktura
- AAA (Arrange / Act / Assert) oddzielone pustymi liniami; komentarze tylko gdy "dlaczego" nie wynika z kodu.
- Jeden test = jedno zachowanie (nie "jedna asercja"): pierwsza porażka przerywa test i ukrywa resztę.
- Nazwa: test_<jednostka>_<oczekiwany_wynik>_<warunek>, z czasownikiem (returns/raises).
  Nazwa to komunikat błędu w CI — musi mówić, jaka reguła biznesowa padła.
- Bez duplikatów testów; uwaga na zduplikowaną nazwę funkcji (druga nadpisuje pierwszą,
  test cicho znika) → ruff F811.

## Wartości brzegowe
- Klasy równoważności: w środku przedziału zachowanie jest takie samo → testuj GRANICE.
- Każda granica = para: ostatnia wartość przed i pierwsza po (14/15, 59/60, -121/-120).
- Dane testowe projektuj tak, żeby błąd ZMIENIAŁ wynik (np. odwołany lot z dużym opóźnieniem).
- Weryfikacja jakości testów: mutacja (< → <=) musi dać czerwony test.
  Automatycznie: mutmut. Mutant, który przeżył = luka w testach lub testujesz co innego.

## Wyjątki
- with pytest.raises(KonkretnyTyp, match="stabilny fragment"): tylko JEDNO wywołanie w bloku.
- Za szeroki blok = fałszywie zielony test (np. json.loads → JSONDecodeError ⊂ ValueError;
  pydantic ValidationError ⊂ ValueError).
- match = regex przez re.search (fragment). Znaki . ^ $ * + ? { } [ ] \ | ( ) → re.escape(...).
- Dopasowuj fragment wyrażający regułę biznesową, nie całe zdanie.
- exc_info: asercje POZA blokiem with.

## Floaty i pieniądze
- 0.1 + 0.2 != 0.3 (IEEE 754). Wynik float → zawsze `actual == pytest.approx(expected)`.
- approx po stronie oczekiwanej: tolerancja względna liczona od wartości owiniętej.
- Wartość oczekiwana z niezależnego źródła (literał, specyfikacja) — nie z tej samej formuły.
- Pieniądze: Decimal("1.22") ze stringa (nigdy Decimal(1.22)) lub int w groszach.

## Rodzaje złych testów
- Trywialnie prawdziwy: assert "WAW" == "WAW" (ruff PLR0133).
- Tautologiczny: expected liczone tą samą logiką co kod → zgadza się zawsze, także z błędem.
- Połykający porażki: try/except AssertionError → test nie może się nie udać.

## Piramida w DE
- Unit (setki, ms): czyste funkcje — parsowanie/walidacja, filtry, transformacje, metryki.
- Integracja (dziesiątki): I/O — pliki (tmp_path), DB w Dockerze, emulowane S3, lokalna Delta.
- E2E (kilka): cały potok w prawdziwej infrastrukturze.
- Testowalność to decyzja projektowa: functional core, imperative shell.
- Walidacja na granicy (po odczycie): zła struktura pliku → stop; złe wiersze → kwarantanna.
  Za granicą dane typowane (dataclass/TypedDict → Pydantic), CSV daje same stringi.
- Testy kodu (pytest, CI) ≠ testy danych (dbt tests, expectations; każdy przebieg na produkcji).

## Coverage
- Mierzy wykonane linie, nie sprawdzone zachowania. Off-by-one przejdzie przy 100%.
- Błąd na produkcji: najpierw czerwony test reprodukujący → poprawka → test zostaje jako regresyjny.

## Narzędzia w CI: ruff → mypy (strict) → pytest
- mypy: statyczne typy, wszystkie ścieżki kodu, bez uruchamiania; Python ignoruje adnotacje w runtime.
- Pydantic: typy w runtime na danych z zewnątrz. mypy pilnuje wnętrza, Pydantic granicy.

## Lekcje z praktyki
- Przed commitem: git diff --stat (wklejenie kodu potrafi nadpisać cały plik).
- Po przeniesieniu pakietu: uv sync --reinstall-package <pkg>.
- mypy: jedyny __init__.py w src/<pakiet>/ — dodatkowe __init__.py psują ustalanie nazw modułów.