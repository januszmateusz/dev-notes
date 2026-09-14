# sqlfluff — ściągawka komend + uzasadnienie konfiguracji

## Komendy

| Komenda | Co robi |
|---|---|
| `sqlfluff lint models/` | Sprawdza styl, nic nie zmienia |
| `sqlfluff lint models/plik.sql` | Lint pojedynczego pliku |
| `sqlfluff fix models/` | Automatyczna naprawa tego, co da się naprawić |
| `sqlfluff fix models/plik.sql --rules CP01` | Naprawa tylko wybranej reguły |

## Krytyczne — katalog roboczy musi być tam, gdzie leży `.sqlfluff`

Templater `dbt` **nie działa**, jeśli uruchomisz `sqlfluff` z innego katalogu niż ten, w którym leży `.sqlfluff` z konfiguracją. Z niewłaściwego katalogu sqlfluff cicho przełącza się na goły templater `jinja`, co daje fałszywe błędy (`Undefined jinja template variable: 'dbt_utils'`, `unparsable section`).

```powershell
cd dbt_project\airline_warehouse   # tam, gdzie leży .sqlfluff
sqlfluff lint models/
```

## Konfiguracja projektu (`.sqlfluff`) — pełne uzasadnienie

```ini
[sqlfluff]
templater = dbt
dialect = databricks
max_line_length = 120
exclude_rules = LT01, ST06, RF02, RF04

[sqlfluff:templater:dbt]
project_dir = .
profiles_dir = ~/.dbt

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = upper
```

| Ustawienie | Dlaczego |
|---|---|
| `templater = dbt` | Konieczne, żeby poprawnie rozwiązać Jinja (`{{ ref() }}`, makra) — goły `jinja` templater nie ma kontekstu projektu dbt |
| `capitalisation_policy = upper` | Świadomy wybór stylu — słowa kluczowe SQL wielkimi literami |
| `max_line_length = 120` | Spójne z limitem w `pyproject.toml` (ruff) |

## Wykluczone reguły — dlaczego każda z nich

| Reguła | Co sprawdza | Dlaczego wykluczona |
|---|---|---|
| `LT01` | Dokładnie jedna spacja przed `AS` | Koliduje z **celowym** wyrównaniem aliasów kolumn w modelach staging (czytelność ważniejsza niż mechaniczna spójność spacji) |
| `ST06` | Kolejność `SELECT *` względem obliczeń | Koliduje ze wzorcem `SELECT *, ROW_NUMBER() OVER(...)` używanym do deduplikacji — bardzo częsty, poprawny wzorzec w dbt |
| `RF02` | Niejednoznaczne odwołanie do kolumny przy wielu tabelach w zapytaniu | Fałszywy alarm na skorelowanych podzapytaniach w blokach `is_incremental()`, np. `(SELECT MAX(x) FROM {{ this }})` — sqlfluff nie rozumie, że to niezależne podzapytanie |
| `RF04` | Słowa kluczowe SQL jako nazwy kolumn | `year`/`month`/`quarter` w `dim_date` — świadomie odłożona zmiana (przemianowanie wymagałoby `--full-refresh` + aktualizacji downstream), nie przeoczenie |

## Auto-fixable vs wymaga ręcznej decyzji

| Reguła | Auto-fix? |
|---|---|
| `CP01` (wielkość liter słów kluczowych) | Tak, zawsze bezpieczne |
| `LT12` (brak nowej linii na końcu pliku) | Tak, zawsze bezpieczne |
| `LT13` (plik zaczyna się od pustej linii) | Tak |
| `AL01` (brak jawnego `AS` przy aliasie tabeli) | Tak |
| `LT02` (wcięcia, zwłaszcza wokół bloków Jinja `{% if %}`) | Tak, ale **zawsze zweryfikuj diff po fixie** przy plikach z Jinja — sprawdzone bezpieczne, ale warto się upewnić |
| `ST02` (zbędny `CASE` tam, gdzie wystarczy samo wyrażenie logiczne) | **Nie** — wymaga ręcznej oceny, czy uproszczenie zmienia semantykę |

## Typowy przebieg pracy

```powershell
cd dbt_project\airline_warehouse
sqlfluff lint models/                    # zobacz co jest do naprawy
sqlfluff fix models/                     # napraw automatyczne
sqlfluff lint models/                    # potwierdź "All Finished 🎉"
cd ..\..
git diff dbt_project/                    # zweryfikuj diff przed commitem, zwłaszcza pliki z Jinja
```