# GitHub Actions i Dependabot dla repo Pythona z uv

## Workflow CI

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read                  # najmniejsze uprawnienia na poziomie workflow

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
      - run: uv sync --locked
      - name: pre-commit (gitleaks w osobnym jobie)
        run: uv run pre-commit run --all-files --show-diff-on-failure
        env:
          SKIP: gitleaks
      - run: uv run pytest
```

Skan historii sekretów: osobny job, patrz `security/gitleaks.md`.

## Ochrona `main`

Settings → Branches: wymagany PR, wymagane statusy `checks` i `secrets-scan`.
Bez tego CI to tylko informacja, a nie bramka.

## Weryfikacja, że CI łapie błędy

Świadomie omiń warstwę lokalną i sprawdź, czy CI zareaguje:

```bash
git commit --no-verify -m "test: bypass local hooks"
git push
```

Potem napraw błąd osobnym commitem `fix`.

## Pułapki

- **Płytki checkout** (`fetch-depth: 1`, domyślnie): narzędzia historyczne widzą 1 commit.
- **Wersje akcji** (`@v4`, `@v6`) sprawdzaj na GitHub Marketplace. Aktualizuje je Dependabot.
- **Obraz w kroku `run:`** (np. `docker run ghcr.io/...:v8.30.0`) nie jest aktualizowany
  przez Dependabota, bo ten parsuje tylko `uses:`. Synchronizuj ręcznie.

## Dependabot

Dwie osobne rzeczy:

1. **Alerty i aktualizacje bezpieczeństwa**: Settings → Code security (bez pliku).
2. **Regularne aktualizacje wersji**: `.github/dependabot.yml`.

```yaml
version: 2
updates:
  - package-ecosystem: "uv"
    directory: "/"
    schedule:
      interval: "monthly"
    groups:
      python:
        patterns: ["*"]
    commit-message:
      prefix: "chore"
      include: "scope"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    groups:
      actions:
        patterns: ["*"]
    commit-message:
      prefix: "ci"

  - package-ecosystem: "pre-commit"
    directory: "/"
    schedule:
      interval: "monthly"
    groups:
      hooks:
        patterns: ["*"]
    commit-message:
      prefix: "chore"
      include: "scope"
```

- `groups` z `patterns: ["*"]`: jeden PR na ekosystem zamiast wielu.
- `commit-message`: commity Dependabota zgodne z Conventional Commits
  (lokalny hook commit-msg ich nie sprawdza).
- Status i błędy konfiguracji: Insights → Dependency graph → Dependabot.
  Tam też ręczne „Check for updates”.
- **PR-y od Dependabota nie mają dostępu do zwykłych sekretów Actions.**
  Mają osobną pulę „Dependabot secrets”. Ważne, gdy job (np. ewaluacja LLM) potrzebuje
  klucza API.
- Aktualizacja SDK modelu może zmienić zachowanie: testy i ewaluacja w CI to zabezpieczenie.