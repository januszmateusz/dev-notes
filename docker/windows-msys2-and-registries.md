# Docker: Windows (MSYS2 / Git Bash), rejestry, Compose, Ollama

## Konwersja ścieżek w MSYS2 / Git Bash

Objaw: ścieżka w kontenerze wygląda jak `C:/Program Files/Git/repo`
albo `C:/msys64/repo`. Bash przerobił argumenty POSIX na ścieżki Windows,
zanim trafiły do `docker.exe`.

```bash
MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" docker run --rm \
  -v "$(pwd -W):/repo" \
  <obraz> <komenda> /repo
```

- `MSYS_NO_PATHCONV=1`: Git for Windows. `MSYS2_ARG_CONV_EXCL="*"`: MSYS2.
- `$(pwd -W)`: ścieżka hosta w formacie `C:/...`, zrozumiała dla Docker Desktop.
- W PowerShellu problemu nie ma: `docker run --rm -v "${PWD}:/repo" ...`
- W CI (Linux) problemu nie ma.

## Rejestry obrazów

- `Unable to find image ... locally` to nie błąd: Docker zaraz pobierze obraz.
- `denied` z ghcr.io dla publicznego obrazu: najpierw `docker logout ghcr.io`
  (stary token wysyłany przy każdym pullu). Dopiero potem podejrzewaj brak obrazu.
- Alternatywnie sprawdź Docker Hub (np. gitleaks: `zricethezav/gitleaks`).
- Diagnoza: `docker pull <obraz>` osobno, zanim uruchomisz `docker run`.
- Obrazy przypinaj do konkretnej wersji, nie `latest`.

## Docker Compose w repo

- Plik w korzeniu repo, nazwa `compose.yaml` (`docker-compose.yml` działa dla zgodności).
- Compose szuka pliku także w katalogach nadrzędnych.
- `.env` w korzeniu jest czytany do podstawiania zmiennych w pliku (`${VAR}`),
  ale NIE trafia do kontenerów bez `env_file:`. Klucze API nie wyciekają do kontenerów.
- Gdy serwisów przybywa: `profiles: [rag]` i `docker compose --profile rag up -d`.

## Ollama

```yaml
services:
  ollama:
    image: ollama/ollama:<wersja>
    ports:
      - "11434:11434"
    volumes:
      - ollama:/root/.ollama      # wagi modeli w wolumenie, nie w repo

volumes:
  ollama:
```

```bash
docker compose up -d
docker compose exec ollama ollama pull llama3.2:3b
```

- API: `POST /api/chat`, `stream: true` zwraca NDJSON, ostatnia linia ma `done: true`.
- Temperatura: `"options": {"temperature": 0.8}`.
- Metryki w odpowiedzi (czasy w nanosekundach):
  `prompt_eval_count`, `prompt_eval_duration`, `eval_count`, `eval_duration`.
- Pierwsze wywołanie = ładowanie modelu do pamięci. Zawsze rozgrzewka przed pomiarem.
- **Ollama cache'uje wspólny prefiks promptu.** Identyczne prompty w pomiarze dają
  sztucznie niskie TTFT. W benchmarku zmieniaj początek promptu w każdym przebiegu.