# Anthropic API: konfiguracja i pułapki

## Konta: claude.ai vs Claude Console

- **claude.ai**: aplikacja z subskrypcją (ryczałt, limity użycia).
- **Claude Console** (platform.claude.com): organizacja, workspace'y, klucze API,
  rozliczenie za tokeny.
- Ten sam e-mail, ale **osobne konta, salda i rozliczenia.** Subskrypcja nie obejmuje API,
  kredyty API nie opłacają aplikacji. Limity z konsoli dotyczą tylko API.
- Claude Code z ustawionym `ANTHROPIC_API_KEY` rozlicza się z API, a nie z subskrypcji.
- Usługi konsumenckie i komercyjne działają na innych warunkach (ważne przy danych klienta).

## Klucz API

- Format: `sk-ant-api03-...`, ponad 100 znaków.
- `apikey_...` (ok. 31 znaków) to **ID klucza**, nie klucz. Daje 401 `invalid x-api-key`.
- Pełny sekret konsola pokazuje **tylko raz, przy tworzeniu.** Zgubiony = nowy klucz.
- **Zakres: workspace, nie organizacja** (najmniejsze uprawnienia). Klucz organizacyjny
  działa w wielu workspace'ach (wymaga nagłówka `anthropic-workspace-id`) i ma dostęp
  do endpointów Admin API.
- Ustaw **datę wygaśnięcia** (np. 90 dni): wymusza rotację.
- Nieużywane klucze usuwaj.

## Workspace i limity

- Osobny workspace na projekt (osobny limit, osobne raportowanie kosztów dla klienta).
- **Default Workspace nie ma własnych limitów** (dziedziczy limit organizacji).
- Settings → Workspaces → `<workspace>` → Limits:
  - **limit wydatków** (miesięczny, niższy niż limit organizacji),
  - **powiadomienie e-mail** (np. 50% limitu),
  - **limity zapytań** (RPM, tokeny wejścia/wyjścia na minutę).
- Limit wydatków ogranicza, ile stracisz w miesiącu. Limit zapytań ogranicza,
  jak szybko możesz stracić.

## Diagnoza 401 bez wyświetlania klucza

```powershell
uv run python -c "import os; from dotenv import dotenv_values; e=os.environ.get('ANTHROPIC_API_KEY') or ''; v=dotenv_values('.env').get('ANTHROPIC_API_KEY') or ''; raw=open('.env','rb').read(4); print('systemowa:', bool(e), e[:10], len(e)); print('.env:', v[:10], len(v), 'biale znaki:', v != v.strip()); print('takie same:', e == v); print('poczatek pliku:', raw)"
```

| Objaw | Przyczyna |
| :--- | :--- |
| systemowa: True, takie same: False | stara zmienna środowiskowa wygrywa z `.env` |
| początek pliku `\xff\xfe` lub `\xef\xbb\xbf` | `.env` w UTF-16 / UTF-8 z BOM |
| prefiks `apikey_`, długość ok. 31 | ID klucza zamiast klucza |
| białe znaki: True | uszkodzona wartość |

- **`load_dotenv()` NIE nadpisuje istniejących zmiennych środowiskowych.**
  Lepiej usunąć konflikt niż używać `override=True` (w CI zmienne środowiskowe
  powinny wygrywać).
- Usunięcie zmiennej użytkownika w Windows (potem nowy terminal):
  `[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "User")`
- `request_id` z błędu można bezpiecznie udostępniać. Klucza nigdy, nawet częściowo.

## SDK: wzorce

```python
# Liczenie tokenów bez generacji
client.messages.count_tokens(model=MODEL, messages=[...]).input_tokens

# Streaming z pomiarem TTFT
with client.messages.stream(model=MODEL, max_tokens=2_000, messages=[...]) as stream:
    for text in stream.text_stream:     # tylko tekst, bez bloków thinking
        ...
    final = stream.get_final_message()
final.usage.input_tokens, final.usage.output_tokens   # output zawiera thinking

# Pole API, którego sygnatura SDK nie zna (nowe albo usunięte)
client.messages.create(..., extra_body={"output_config": {"effort": "low"}})
```

- **Bloki odpowiedzi wybieraj po `type`, nie po pozycji** (thinking może być pierwszy).
- **`extra_body` omija walidację SDK w obie strony.** Używaj świadomie, z komentarzem.
- **Dwie warstwy walidacji:** SDK (`TypeError` przy nieznanym argumencie) i API (400).
- Logowanie błędów: kod statusu, typ błędu, `request_id`, a nie sklejone `error.message`.
- **400 nigdy nie ponawiaj.** Ponawia się błędy przejściowe (np. 429, 529).

## Haiku 5.5: zmiany względem 4.5 (z przewodnika migracji)

- Model ID: `claude-haiku-5-5`.
- Usuń `temperature`, `top_p`, `top_k` (inne niż domyślne wartości = 400).
- Thinking: nieustawiony albo `{"type": "adaptive"}`. Sterowanie przez
  `output_config.effort`.
- Mały `max_tokens` z 4.5: podnieś (miejsce na myślenie) albo obniż effort.
- Szacunki kosztów z 4.5 przelicz na tokenach i cenach 5.5.