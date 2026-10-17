# gitleaks: skanowanie sekretów

## Instalacja przez pre-commit

```yaml
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.30.0
  hooks:
    - id: gitleaks
```

- pre-commit klonuje repo gitleaks w wersji `rev` do cache'u (`~/.cache/pre-commit`)
  i kompiluje binarkę (hook `language: golang`).
- Inne hooki z tego repo: `gitleaks-docker`, `gitleaks-system`.
- Lokalnie działa binarka z pre-commit, w CI obraz Dockera. Ta sama wersja = te same reguły:
  pilnuj zgodności `rev` i tagu obrazu.

## Tryby

| Komenda | Źródło danych | Zastosowanie |
| :--- | :--- | :--- |
| `gitleaks git --staged` | staged diff | hook pre-commit |
| `gitleaks git` | `git log -p` (cała historia) | CI |
| `gitleaks git --log-opts="origin/main..HEAD"` | zakres commitów | PR |
| `gitleaks dir <ścieżka>` | pliki na dysku | diagnoza |
| `gitleaks stdin` | wejście standardowe | logi, pipe |

`detect` i `protect` to poprzednie, przestarzałe nazwy trybów.

## Jak działa (na każdą regułę)

1. **keywords**: szybki filtr. Brak słowa kluczowego = regex się nie uruchamia.
2. **regex**: dopasowanie wzorca.
3. **entropy**: odrzuca ciągi o niskiej entropii (`xxxx`, `changeme`).
4. **allowlisty**: globalne i per reguła (ścieżki, regexy, stopwords, commity).
5. **finding** z fingerprintem, kod wyjścia 1 (zatrzymuje commit/job).

## Czego NIE łapie (sprawdzone)

- **Plików UTF-16**: bajty zerowe między znakami, więc filtr keywords nic nie znajduje.
- **Sekretów już w historii**, gdy działa jako hook (`--staged` widzi tylko dodane linie).
- **Sekretów bez dedykowanej reguły** i o niskiej entropii.
- W CI: niczego przy płytkim checkoucie (`fetch-depth: 1`).

## Skan historii w CI

```yaml
secrets-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0              # bez tego skanowany jest 1 commit
    - run: |
        docker run --rm \
          --user "$(id -u):$(id -g)" \
          -v "${{ github.workspace }}:/repo" \
          ghcr.io/gitleaks/gitleaks:v8.30.0 \
          git /repo --redact --verbose
```

- Weryfikacja: w logu `N commits scanned` z N > 0. Zielony job to za mało.
- `--redact`: sekrety nie trafiają do logów CI.
- `--user`: unika „dubious ownership” w kontenerze.
- Hook gitleaks w kroku z `pre-commit run --all-files` pomijam (`SKIP: gitleaks`),
  bo na świeżym checkoucie nic nie jest staged.

## Konfiguracja `.gitleaks.toml`

Kolejność wyszukiwania: `--config`, `GITLEAKS_CONFIG`, `GITLEAKS_CONFIG_TOML`,
`.gitleaks.toml` w skanowanym katalogu.

```toml
[extend]
useDefault = true            # bez tego własny plik ZASTĘPUJE domyślne reguły

[[rules]]
id = "internal-service-token"
regex = '''\bsvc_[a-z0-9]{32}\b'''
keywords = ["svc_"]
entropy = 3.5

[[allowlists]]
paths = ['''tests/fixtures/.*''']
```

`[[allowlists]]` (lista) jest w nowszych wersjach 8.x. Starsze przykłady używają `[allowlist]`.

## Fałszywe alarmy: od najwęższego wyjątku

1. `.gitleaksignore`: fingerprint konkretnego wyniku (najbezpieczniejsze).
2. `# gitleaks:allow` na końcu linii.
3. `--baseline-path`: raport z wcześniejszego skanu jako zaakceptowany stan.
4. Allowlista w `.gitleaks.toml`.

## Test, że działa

Fałszywy klucz w formacie Anthropic (108 znaków), zapisany jako UTF-8:

```bash
uv run python -c "import secrets, pathlib; pathlib.Path('probe.txt').write_text('ANTHROPIC_API_KEY=sk-ant-api03-' + secrets.token_urlsafe(80)[:93] + 'AA\n', encoding='utf-8', newline='\n')"
file probe.txt            # ASCII text
git add probe.txt && git commit -m "test: leak check"   # oczekiwane: blokada
git restore --staged probe.txt && rm probe.txt
```

Test na NOWYM pliku. Plik już śledzony nie sprawdzi reguły (patrz „Czego NIE łapie”).