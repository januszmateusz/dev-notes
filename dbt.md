# dbt — ściągawka komend

## Podstawowe komendy

| Komenda | Co robi | Kiedy używać |
|---|---|---|
| `dbt run` | Buduje modele (bez testów) | Szybka iteracja nad SQL |
| `dbt build` | Seeds + run + test + snapshot, w poprawnej kolejności zależności | Pełny, bezpieczny przebieg — preferowany domyślnie |
| `dbt test` | Uruchamia tylko testy | Weryfikacja po zmianie w YAML, **wymaga** że modele już istnieją (`dbt run` najpierw) |
| `dbt seed` | Ładuje pliki CSV z `seeds/` do bazy | Po zmianie danych referencyjnych |
| `dbt snapshot` | Uruchamia SCD Type 2 | Po zmianie w źródle snapshotu; **nie** jest częścią `dbt run`, tylko `dbt build` |
| `dbt docs generate` | Generuje statyczną dokumentację (lineage, opisy) | Przed `dbt docs serve`/publikacją |
| `dbt docs serve` | Otwiera dokumentację lokalnie w przeglądarce | Przegląd lineage bez publikowania |
| `dbt debug` | Sprawdza połączenie z warehouse | Pierwsza komenda przy diagnozowaniu problemów z profilem/credentials |
| `dbt deps` | Instaluje pakiety z `packages.yml` (np. `dbt_utils`) | Po zmianie w `packages.yml`, zawsze przed pierwszym `run` na nowym środowisku |
| `dbt ls --resource-type model` | Lista modeli, które dbt widzi | Diagnostyka "czemu mój model nie istnieje/nie jest widoczny" |

## Selektory (`--select`)

| Wzorzec | Znaczenie |
|---|---|
| `--select model_name` | Tylko wskazany model |
| `--select model_name+` | Model + wszystko co po nim zależy (downstream) |
| `--select +model_name` | Wszystko przed modelem (upstream) + sam model |
| `--select tag:something` | Wszystkie modele z danym tagiem |

## Flagi

| Flaga | Znaczenie | Uwaga |
|---|---|---|
| `--full-refresh` | Przebuduj model od zera (DROP + CREATE) | **Nie działa** dla `dbt snapshot` — nie ma takiej opcji, snapshot trzeba resetować ręcznym `DROP TABLE` |
| `on_schema_change='append_new_columns'` (w configu modelu) | Bezpieczne dodawanie nowych kolumn w incremental | Nie zastępuje `--full-refresh` przy zmianie **typu** istniejącej kolumny |

## Kluczowe wzorce z tego projektu

### Incremental z watermarkiem po `_ingested_at`, nie po dacie biznesowej
Jeśli źródło może dostarczać dane z przeszłości w tym samym przebiegu (np. przesuwne okno generowania), filtr `WHERE flight_date > max(flight_date)` **gubi dane**. Właściwy watermark to kolumna techniczna śledząca moment załadowania:
```sql
{% if is_incremental() %}
where _ingested_at > (select max(_ingested_at) from {{ this }})
{% endif %}
```

### Accumulating snapshot — `merge_update_columns` NIE chroni przed nadpisaniem NULL-em
`merge_update_columns` mówi które kolumny **wolno** aktualizować, ale jeśli zapytanie źródłowe samo liczy `NULL` dla danej kolumny w danym przebiegu, MERGE i tak ją nadpisze. Potrzebny jawny `LEFT JOIN` do `{{ this }}` z `COALESCE(nowa_wartość, stara_wartość)` per kolumna:
```sql
coalesce(n.scheduled_at, e.scheduled_at) as scheduled_at
```

### SCD Type 2 (`dbt snapshot`)
- `unique_key` — jeśli chcesz śledzić **historię tej samej encji w czasie** (np. kurs waluty), klucz to sama encja (`currency_code`), NIE `(currency_code, data)` — inaczej każdy dzień tworzy nowy, niezależny byt zamiast budować historię
- `strategy='check'` + `check_cols=[...]` — świadomie wybieraj **tylko** kolumny, których zmiana ma znaczenie biznesowe; `check_cols='all'` złapie też nieistotny szum (np. korektę literówki)
- **Backdating pierwszej wersji** — snapshot zaczyna śledzić historię dopiero od momentu pierwszego uruchomienia; jeśli fakty mają daty wcześniejsze, join po zakresie dat zwróci puste wyniki, dopóki nie cofniesz `valid_from` pierwszej wersji do sensownej daty referencyjnej

### `EXCEPT` porównuje pozycyjnie, nie po nazwach kolumn
Przydatne przy audycie spójności między warstwami o różnych nazwach kolumn (Bronze vs Silver) — `EXCEPT` ignoruje nazwy, porównuje tylko pozycję i typ.

## Ślepe zaułki, które kosztowały czas

- **`sqlfluff` z dbt templaterem musi być uruchomiony z katalogu, gdzie leży `.sqlfluff`** (u nas: `dbt_project/airline_warehouse/`) — z innego katalogu templater cicho przełącza się na `jinja` i sypie fałszywymi błędami (`Undefined jinja template variable`)
- Test nowego modelu zawsze wymaga `dbt run` **przed** `dbt test` — `dbt test` na nieistniejącej jeszcze tabeli daje `TABLE_OR_VIEW_NOT_FOUND`, mylące jeśli zapomnisz o kolejności