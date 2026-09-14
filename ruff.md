# ruff — ściągawka komend + uzasadnienie konfiguracji

## Komendy

| Komenda | Co robi |
|---|---|
| `ruff check ingestion/ airflow/dags/` | Sprawdza styl, nic nie zmienia |
| `ruff check --fix ingestion/ airflow/dags/` | Automatyczna naprawa tego, co da się naprawić |
| `ruff check plik.py` | Lint pojedynczego pliku |

## Katalog roboczy — w przeciwieństwie do sqlfluff, prostsze

Config (`pyproject.toml`) leży w **głównym katalogu repo**, więc `ruff` działa poprawnie uruchomiony stamtąd — nie ma tu odpowiednika problemu z katalogiem, który dotyka sqlfluff.

```powershell
cd C:\DATA_ENGINEER\airline-cloud-warehouse   # główny katalog repo
ruff check ingestion/ airflow/dags/
```

## Konfiguracja projektu (`pyproject.toml`)

```toml
[tool.ruff]
line-length = 120
target-version = "py313"

[tool.ruff.lint]
select = ["E", "F", "I"]
```

| Ustawienie | Co obejmuje | Dlaczego |
|---|---|---|
| `line-length = 120` | Max długość linii | Spójne z limitem w `.sqlfluff` |
| `select = ["E", "F", "I"]` | E=pycodestyle (styl), F=pyflakes (realne błędy: nieużywane importy/zmienne), I=isort (kolejność importów) | Świadomie **wąski** zestaw na start — celowo pominięte `D` (docstringi) i `ANN` (wymuszanie type hints), żeby nie zalać projektu setkami uwag przy pierwszym uruchomieniu; rozszerzać świadomie, nie od razu |

## Auto-fixable vs wymaga ręcznej decyzji

| Reguła | Auto-fix? | Uwaga |
|---|---|---|
| `I001` (kolejność/organizacja importów) | Tak, zawsze bezpieczne |
| `E501` (linia za długa) | **Nie** — ruff nie decyduje **jak** podzielić linię, wymaga ludzkiego osądu o czytelności |
| `BLE001` (łapanie gołego `Exception`) | Nie, wymaga przemyślenia jaki konkretny wyjątek złapać |

## Typowy przebieg pracy

```powershell
cd C:\DATA_ENGINEER\airline-cloud-warehouse
ruff check --fix ingestion/ airflow/dags/    # napraw automatyczne (głównie kolejność importów)
ruff check ingestion/ airflow/dags/          # potwierdź "All checks passed!"
git diff                                     # zweryfikuj co się zmieniło
```

## Kontekst rynkowy (dla pamięci)

ruff w dużej mierze **zastąpił** wcześniejszą kombinację `flake8` + `isort` + częściowo `black` w ekosystemie Pythona — napisany w Rust, znacznie szybszy, obecnie de facto standard. Nauka jednego dobrego lintera (koncepcja: kategorie reguł, auto-fix vs ręczne, świadome wykluczenia z uzasadnieniem, integracja z CI/pre-commit) przenosi się niemal 1:1 na inne narzędzia tego typu (ESLint, RuboCop, nadchodzące SDF dla SQL).