# Migracja: Poetry → uv

## Co zmieniono
- Menedżer zależności: Poetry → uv. `poetry.lock` zastąpiony przez `uv.lock`.
- `pyproject.toml`: sekcje `[tool.poetry]` przekonwertowane do standardu PEP 621 (`[project]`).
  Operatory przetłumaczone na jawne zakresy (`^2.1` → `>=2.1,<3`, `~2.31` → `>=2.31,<2.32`).
- Grupa opcjonalna `lint` → grupa zależności spoza domyślnych (`uv sync --group lint`).
  Extra `parquet` i skrypt `flight-loader` bez zmian w działaniu.
- Backend budowania: poetry-core → hatchling.
- Dodano `.python-version` (3.11) — ten sam interpreter, którego używało Poetry.

## Weryfikacja
- **Runtime:** wersje wszystkich pakietów grupy main zgodne 1:1
  (`uv pip freeze` vs zrzut `poetry show --only main` ze środowiska zbudowanego z `poetry.lock`).
- **Działanie:** `uv run pytest` i `uv run flight-loader` na `sample.jsonl` — OK.
- **Wheel:** zawiera te same moduły i entry point co przed migracją.
- **Nie zweryfikowano:** wersji narzędzi dev (pytest, ruff i zależności) — zrzut objął tylko
  grupę main. Zostały rozwiązane ponownie w tych samych zakresach (pytest 7.4.x, ruff 0.1.x).
  Nie porównano też zawartości sdist (tylko wheel).

## Znane problemy
1. **`poetry.lock` nie mógł zostać użyty bezpośrednio.** Zawierał dwie wersje numpy dla różnych
   wersji Pythona, co narzędzie migracyjne zamieniło na sprzeczne ograniczenia. Wersje odtworzono
   ze zrzutu środowiska. Oryginalny `poetry.lock` nie był w historii repozytorium.
2. **Przestarzałe narzędzia:** `ruff ^0.1` blokuje ruff na serii 0.1.x (dwa lata wstecz),
   `pytest ^7.4` blokuje pytest 8.
3. **`requests ~2.31`** blokuje requests poniżej 2.32, w tym poprawki bezpieczeństwa
   z nowszych wersji. Do tego requests nie jest importowany w kodzie repozytorium.
4. **`requires-python = ">=3.11,<4"`:** górna granica przeniesiona z Poetry — do decyzji.

## Rekomendowane kolejne kroki (osobne PR-y)
1. CI: `uv sync --locked` + testy — blokuje merge przy niezsynchronizowanym locku.
2. Potwierdzenie z zespołem i usunięcie nieużywanego `requests`.
3. Aktualizacja ruff do bieżącej wersji (spodziewane zmiany w formatowaniu/regułach — osobny PR).
4. Aktualizacja pytest do 8.x.
5. Decyzja o górnych granicach zakresów i `requires-python`.

## Ściąga: Poetry → uv
| Poetry | uv |
|---|---|
| `poetry install --with dev` | `uv sync` |
| `poetry install --with lint` | `uv sync --group lint` |
| `poetry install -E parquet` | `uv sync --extra parquet` |
| `poetry add <pakiet>` | `uv add <pakiet>` |
| `poetry remove <pakiet>` | `uv remove <pakiet>` |
| `poetry update` | `uv lock --upgrade` + `uv sync` |
| `poetry update <pakiet>` | `uv lock --upgrade-package <pakiet>` + `uv sync` |
| `poetry run pytest` | `uv run pytest` |
| `poetry export` (wtyczka) | `uv export --no-dev -o requirements.txt` |