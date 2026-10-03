# Lintery — kiedy poprawić kod, kiedy opisać środowisko, kiedy wyciszyć

Zasada: wyciszenie reguły to ostateczność. Reguła, której nie wyciszysz, dalej pracuje i łapie problem przy następnym pliku — wyciszenie wyłącza ją na zawsze dla całego zakresu.

## Drzewo decyzji

1. **Czy kod da się poprawić?** → popraw kod. Bez zmian w konfiguracji.
2. **Czy to cecha środowiska, której nie zmienisz?** → opisz środowisko narzędziu (precyzyjnie, konkretnymi nazwami).
3. **Czy kod jest poprawny, a myli się narzędzie?** → wycisz **wąsko**: najmniejszy możliwy zakres + komentarz z uzasadnieniem.

| Sytuacja | Czy kod da się poprawić? | Decyzja | Przykład z projektu |
|---|---|---|---|
| Import w środku notebooka | Tak | Poprawić kod | ruff `E402` — import przeniesiony do pierwszej komórki, wyjątek `per-file-ignores` usunięty |
| `spark`, `dbutils`, `display` bez importu | Nie — wstrzykuje je Databricks | Opisać środowisko | ruff `F821` → `builtins = ["spark", "dbutils", "display"]` |
| Pole struktury `data.tail_number` | Nie — myli się narzędzie | Wyciszyć wąsko | sqlfluff `RF01/RF03/AL09` → zakres `noqa` tylko na blok rozpakowujący strukturę |

## Weryfikuj, że wyjątek jest potrzebny i niczego nie maskuje

- **Czy wyjątek jest potrzebny?** Usuń go na chwilę i uruchom linter. Jeśli nic się nie pojawi — wyjątek był zbędny
- **Czy nie maskuje prawdziwych błędów?** Wstaw celowy błąd, który powinien zostać złapany (np. `x = sparkk.read` — literówka musi dalej dać `F821`)
- Nie dodawaj wyjątków "na zapas" — konfiguruj rozwiązanie problemu, który faktycznie występuje

## Składnia — od najwęższego do najszerszego zakresu

### ruff (`pyproject.toml`)

```python
import x  # noqa: E402          ← jedna linia
```

```toml
[tool.ruff.lint.per-file-ignores]
"databricks/notebooks/*.py" = ["E402"]   # ← cały plik / katalog

[tool.ruff]
builtins = ["spark", "dbutils", "display"]   # ← opis środowiska, nie wyciszenie
```

`[tool.ruff.lint] ignore = [...]` wyłącza regułę w całym projekcie — praktycznie nigdy.

### sqlfluff

```sql
SELECT data.tail_number AS tail_number  -- noqa: RF01      ← jedna linia
```

```sql
-- The linter cannot distinguish struct field access from a table-qualified column.
-- noqa: disable=RF01,RF03,AL09
...blok kodu...
-- noqa: enable=RF01,RF03,AL09
```

```ini
[sqlfluff]
exclude_rules = LT01, ST06, RF02, RF04   ; ← cały projekt
```

**Pułapka:** komentarz zaczynający się od słowa `sqlfluff` jest traktowany jako konfiguracja w linii (`-- sqlfluff:rules:...`). Komentarz-uzasadnienie musi zaczynać się inaczej, inaczej pojawia się ostrzeżenie `Unable to process inline config statement`.

## Wyjątki obowiązujące w całym projekcie — klasyfikacja

| Reguła | Kategoria | Uzasadnienie |
|---|---|---|
| sqlfluff `LT01` | Świadomy wybór stylu | Wyrównane aliasy w modelach staging są czytelniejsze |
| sqlfluff `ST06` | Myli się narzędzie | Koliduje z poprawnym wzorcem `SELECT *, ROW_NUMBER() OVER (...)` |
| sqlfluff `RF02` | Myli się narzędzie | Fałszywe alarmy na podzapytaniach z `{{ this }}` w blokach `is_incremental()` |
| sqlfluff `RF04` | Odłożona poprawka | `year`/`month`/`quarter` w `dim_date` — zmiana nazw wymaga `--full-refresh` i aktualizacji modeli zależnych |

Wyjątek z kategorii "odłożona poprawka" to dług techniczny, a nie decyzja ostateczna — warto mieć go w backlogu.

## Fałszywy alarm po naprawie fałszywego alarmu

Zanim zaczniesz kombinować ze składnią, ustal, **czego narzędzie nie rozumie**. Przy polach struktury dodanie aliasu tabeli (`p.data.tail_number`) zamieniło `RF03` na `RF01` + `AL09`, bo sqlfluff odczytał trzy segmenty jako `schemat.tabela.kolumna`. Nie istniała składnia kropkowa, która zadowoliłaby parser — dopiero zrozumienie przyczyny doprowadziło do właściwego rozwiązania: rozpakowania struktury raz, w jednym miejscu.