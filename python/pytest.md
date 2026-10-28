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

# pytest — sesja 3: fixtures (podstawy)

## Czym jest fixture
- Funkcja z @pytest.fixture; test prosi o nią nazwą parametru → pytest wywołuje i wstrzykuje wynik.
- Nie wywołuj fixture ręcznie. Zależności jawne w sygnaturze, tworzone leniwie (tylko gdy test prosi).
- Domyślnie scope="function": ciało fixture wykonuje się OD NOWA dla każdego testu → izolacja.
- Nazwa = rzeczownik opisujący co dostarcza (mixed_flights, airport_db), nie "setup_data".
- Klasy testowe: fixtures jako argumenty metod po self (zamiast __init__).

## Izolacja danych
- Dane twórz W CIELE fixture. `return MODUL_LISTA` → wszystkie testy dzielą jeden obiekt
  → testy zależne od kolejności (pytest-randomly pomaga je wykryć).
- list(x) = kopia płytka: nowa lista, te same elementy. Mutowalne rekordy (dict) dalej się "przelewają".
- Rekordy testowe jako @dataclass(frozen=True): niemodyfikowalne, porównywalne przez ==.

## yield: setup i teardown
- Kod przed yield = setup, po yield = teardown. Typ: -> Iterator[T] (test dostaje T).
- Teardown wykonuje się po teście zawsze (PASS i FAIL): wyjątek z testu nie trafia do fixture,
  pytest po prostu wznawia generator → try/finally dla porażek testu NIEpotrzebny.
- Wyjątek w setupie (przed yield) → E, test się nie uruchamia, kod po yield NIE wykona się
  → zasób otwarty przed błędem WYCIEKA.
- Zabezpieczenie: `with` obejmujący także yield:
    with closing(sqlite3.connect(":memory:")) as conn:
        ...setup...
        yield conn
- Pułapka: `with sqlite3.connect(...)` NIE zamyka połączenia (tylko commit/rollback) → contextlib.closing.
- --setup-show: SETUP/TEARDOWN F <fixture>; F = scope function (nie "failed").
  To księgowość pytesta, nie dowód, że Twój kod teardown się wykonał.

## tmp_path
- Unikalny, pusty katalog (pathlib.Path) na test; izolacja, bezpieczne przy równoległości,
  brak śmieci w repo, katalogi z ostatnich uruchomień zachowane do diagnozy.
- tmpdir = legacy (py.path) → używaj tmp_path.
- Statyczne pliki w repo: Path(__file__).parent / "data" / "x.csv"
  (NIE Path("tests/data/x.csv") — zależy od cwd).

## conftest.py
- Fixtures widoczne w katalogu i podkatalogach; nigdy nie importujesz conftest.py.
- Bliższa definicja (lokalny conftest / plik testowy) przesłania dalszą.
- Przesłaniaj, gdy zmienia się IMPLEMENTACJA przy tym samym znaczeniu
  (db_connection: SQLite w unit/, PostgreSQL w integration/).
- Nowa nazwa, gdy zmienia się ZNACZENIE danych (mixed_flights → all_active_flights).
- Do conftest.py: fixtures. Zwykłe funkcje pomocnicze → helpers.py.
- Diagnostyka: --fixtures (lista + docstringi — pisz docstringi!),
  --fixtures-per-test (która definicja w którym teście), --setup-show.

## Kiedy NIE fixture
- Mystery guest: dane kluczowe dla testu ukryte poza nim.
- Inline: dane krótkie, specyficzne dla testu, będące jego sednem (wartości graniczne!).
- Fixture: dane duże/współdzielone lub zasoby z cyklem życia (pliki, bazy, połączenia).
- W testach DAMP (czytelność) > DRY. Powtarzalne przypadki → parametrize (sesja 5).

## Testy, które nie testują kodu (powtarzający się błąd!)
- Pytanie przed każdą asercją: czy ten test może paść, jeśli kod w src/ jest zepsuty?
- Logika liczona w teście (list comprehension, sum/len) zamiast wywołania funkcji = sygnał,
  że brakuje funkcji produkcyjnej albo test sprawdza sam siebie.
- Negatywny test: jeden defekt na test; nazwa, docstring, dane i match muszą mówić to samo.

## mypy przy wielu conftest.py
[tool.mypy]
strict = true
files = ["src", "tests"]
mypy_path = "src,tests"          # przecinek — bezpieczny też na Windows
explicit_package_bases = true    # nazwy modułów względem baz → conftest vs integration.conftest
- Uruchamiaj `uv run mypy` bez argumentów (argumenty zastępują `files`).
- Parametry tylko do odczytu: Mapping/Sequence/Iterable (dict/list są inwariantne);
  zwracaj typy konkretne (dict/list).

## Dodatkowe lekcje
- CSV daje same stringi; wartości słownikowe ("Cancelled" vs "CANCELLED") rozjeżdżają się
  między komponentami → kontrakt danych z dozwolonymi wartościami na wejściu.
- Zapytania SQL w testach z placeholderami (?), nie wklejanymi wartościami.
- Porównuj całe krotki/obiekty: assert row == ("Warsaw",), nie row[0].

# pytest — sesja 4: fixtures w głąb

## scope = rachunek koszt vs ryzyko
- function (F, domyślny) | class (C) | module (M) | package (P) | session (S).
- Szerszy scope tylko gdy: tworzenie KOSZTOWNE i stan NIEMODYFIKOWANY lub resetowany.
- Tani zasób → function, nawet jeśli szerszy byłby "bezpieczny" (brak zysku, jest ryzyko).
- Fixture sesyjna tworzona leniwie (pierwszy test, który jej potrzebuje), sprzątana na końcu sesji.
- Przesłonięta fixture ma własny scope (to osobna definicja).
- frozen rekordy nie chronią kontenera: list.append w fixture sesyjnej = przeciek.
  Rozważ tuple[Flight, ...] dla danych tylko do odczytu.
- ScopeMismatch: szerszy scope nie może zależeć od węższego → tmp_path_factory.mktemp() zamiast tmp_path.
- Fabryka tworząca pliki na żądanie → function (wspólny katalog = nadpisywanie, problemy przy xdist).
- Plik referencyjny tylko do odczytu → session.

## Wzorzec: zasób na sesję, izolacja na test
- session: schemat + dane bazowe (raz).
- function: połączenie w jawnej transakcji, rollback w teardownie:
    with closing(sqlite3.connect(path, isolation_level=None)) as conn:
        conn.execute("BEGIN"); yield conn; conn.rollback()
- Psuje się gdy: commit() w teście/kodzie, wiele połączeń, DDL z niejawnym commitem (Oracle, MySQL).
- Alternatywy: unit of work (kod nie commituje, robi to orkiestrator), czyszczenie tabel (DELETE/TRUNCATE),
  baza-szablon (kopia pliku SQLite na test / CREATE DATABASE ... TEMPLATE), savepointy,
  osobny schemat/baza na test.
- Podmiana commit(): sqlite3.Connection nie pozwala na przypisanie metody; i test przestaje
  sprawdzać, czy kod w ogóle commituje.
- Baza w tmp_path_factory → każde uruchomienie od czystego stanu (przeciek ograniczony do sesji).

## Fabryki fixtures
- Fixture zwracająca funkcję: chowa wspólną mechanikę (nagłówek, tmp_path, zapis),
  dane będące sednem testu zostają inline (przeciwieństwo mystery guest).
- Fabryka przyjmuje to, co testowany kod CZYTA (czytnik CSV → surowe wiersze tekstu,
  nie obiekty Flight — inaczej nie da się zapisać złych danych).
- Parametr podawany jawnie tylko gdy odbiega od domyślnego (jawny = "ten test o tym jest").
- Typ: Protocol z __call__ (Callable[..., T] nie sprawdza argumentów).
- Typy/stałe importowane przez testy: osobny moduł (tests/factories.py) —
  nie conftest.py (nie importujemy), nie helpers.py (kolejność register_assert_rewrite).
- Lista wierszy: przecinek po każdym elemencie (bez niego Python skleja stringi).

## autouse
- Tylko infrastruktura/siatki bezpieczeństwa: blokada sieci (pytest-socket), reset stanu globalnego.
- Nigdy dane: niewidoczna zależność, koszt dla testów bez bazy, zmienia stan początkowy WSZYSTKICH testów.
- Zamiast: kompozycja — airport_db_with_gdn(airport_db) z nazwą mówiącą, co dodaje.
- TZ przez monkeypatch nie działa na Windows (brak time.tzset) → kod z jawnymi strefami (UTC).

## Logi: caplog
- logger = logging.getLogger(__name__); logger.warning("... %d ...", x) (nie f-string, ruff G004).
- with caplog.at_level(logging.WARNING, logger="flights.loader"): tylko wywołanie; asercje po bloku.
- Filtruj po record.name (caplog łapie też logi z innych modułów) i levelno.
- "Dokładnie WARNING" → ==; "brak problemów" → >= logging.WARNING.
- Podciągi z granicą: "line 3 in", nie "line 3" (dopasuje "line 30").
- Sprawdzaj liczbę wpisów (len == 1), nie tylko istnienie.
- caplog.record_tuples gdy znasz dokładną treść; capsys.readouterr() dla print/CLI.
- Numer linii w logu = to, co zobaczy człowiek w pliku (nagłówek = linia 1).

## PySpark w testach
- spark: scope session, local[1|2], spark.sql.shuffle.partitions=2, spark.ui.enabled=false,
  spark.sql.session.timeZone=UTC; konfiguracja przed pierwszym getOrCreate().
- Izolacja na test: spark.newSession() (osobna konfiguracja SQL i widoki tymczasowe,
  wspólny SparkContext); unikalne nazwy (uuid), osobna baza na test + DROP ... CASCADE,
  clearCache() w teardownie.
- Fixtures z yield → Iterator[T].

## Diagnostyka losowych porażek
- Różnice CI/lokalnie: kolejność, równoległość (xdist), strefa czasowa, katalog uruchomienia.
- pytest-randomly (ziarno z CI), zawężanie zbioru poprzedników, --setup-show.
- Jedno źródło prawdy: lokalnie uruchamiaj komendy dokładnie jak workflow CI (ten sam katalog).

## Lekcje z praktyki
- Każdy zasób z close() od razu w `with` (closing) — także w fixture sesyjnej.
- yield tylko gdy jest co sprzątać; inaczej return.
- path.write_text/path.open zamiast wbudowanego open; write_text na Windows zapisuje CRLF
  (newline="\n" gdy liczy się zgodność bajtowa, np. golden files).
- Przed diagnozą błędu w kodzie — sprawdź kod, nie skopiowany output z terminala (zawijanie linii).

# pytest — sesja 5: parametryzacja

## Podstawy
- @pytest.mark.parametrize(("a", "expected"), [(1, "x"), (2, "y")]) → osobny test na przypadek.
- Nazwy parametrów jako krotka (nie "a, expected"); pojedynczy parametr: "a", [1, 2].
- Krotkę jednoelementową tworzy przecinek: (1,), nie (1).
- Pętla for w teście = jeden test: pierwsza porażka ukrywa resztę. Parametryzacja = niezależne wyniki.
- Tabela przypadków czyta się jak specyfikacja: luki (brak pary 14/15) widać gołym okiem.
- Każda wartość w tabeli ma uzasadnienie (granica / klasa równoważności) albo jej nie ma.
- Testy jednej funkcji trzymaj razem w pliku (najpierw sukces, potem wyjątki).

## ids
- Domyślne ID czytelne dla liczb/krótkich stringów; dla list/obiektów → "delays_list0".
- pytest.param(..., id="boundary_14_on_time_15_late"): ID przy przypadku, mówi DLACZEGO jest w tabeli.
- Osobna lista ids=[...] może rozjechać się z przypadkami.
- Jeden przypadek: uv run pytest "plik.py::test[id]" (cudzysłów w PowerShell); -k też dopasowuje ID.
- Nazwa parametru nie może przesłaniać importu (delays vs delays_list).

## Iloczyn kartezjański
- Kilka dekoratorów parametrize = wszystkie kombinacje. Sens: niezależne wymiary.
  Ryzyko: eksplozja liczby testów, kombinacje bez sensu biznesowego.

## Warianty: gdzie parametryzować
- Parametr testu: dane wejściowe i oczekiwany wynik tego testu.
- Parametr fabryki (line_ending="\n"): test jawnie decyduje; domyślnie jeden wariant.
- @pytest.fixture(params=[...]) + request.param: WSZYSTKIE zależne testy (po łańcuchu fixtures)
  biegną w każdym wariancie — ukryty efekt (42 → 49). Używaj, gdy cała grupa testów ma działać
  w wariantach (np. SQLite i PostgreSQL). Udokumentuj w docstringu fixture.
- request.param ma typ Any → jawny typ: rows: list[str] = request.param / str(request.param).

## indirect
- indirect=True / indirect=["nazwa"]: wartość trafia do fixture (request.param), test dostaje wynik fixture.
- Lepszy od fabryki: parametr = konfiguracja zasobu (format, wersja schematu);
  fixture ze scope i cache per parametr (zasób tworzony raz na wariant).
- Gorszy: proste dane — fabryka czytelniejsza (jawny przepływ, łatwy debug, kilka zasobów w jednym teście).

## xfail
- pytest.param(..., marks=pytest.mark.xfail(reason=..., strict=True)): znany błąd udokumentowany testem.
- x = oczekiwana porażka (CI zielone). XPASS bez strict → X, CI zielone, nieaktualny znacznik zostaje.
- strict=True → XPASS(strict) = FAILED: wymusza usunięcie znacznika po naprawie.
- xfail_strict = true w konfiguracji → strict dla całego projektu.
- reason = PRZYCZYNA (+ ID zgłoszenia), nie objaw. Jedna przyczyna → jeden marker w zmiennej.

## Kiedy NIE parametryzować
- if w ciele sparametryzowanego testu = dwa testy udające jeden. Parametryzuj dane, nie zachowanie.
- Sukces i wyjątek → osobne testy, każdy z własną tabelą.
- Tabela z wieloma kolumnami nieczytelna → rozbij test, fabryki, nazwane obiekty.

## Duże tabele z zewnątrz (np. mapowanie od biznesu)
- Oczekiwania z niezależnego źródła: jeśli produkcja czyta ten sam plik → test tautologiczny.
- Plik wczytywany do parametrize czytany przy zbieraniu → błąd w pliku = ERROR collecting całej sesji.
- Path(__file__).parent, id z kodu źródłowego, test unikalności/kompletności specyfikacji,
  osobny test dla wartości spoza mapowania.

## Normalizacja wartości słownikowych
- Naprawa w filtrze (.upper()) to łatka: każda funkcja musi powtarzać, nie obsłuży "CNL"/spacji,
  niespójne wartości płyną dalej. Docelowo: mapowanie na Enum na granicy (parser), nieznane → kwarantanna.
- Testy zapisują decyzje projektowe: zmiana miejsca walidacji = przeniesienie testów.