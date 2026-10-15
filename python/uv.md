# uv — ściąga (Windows / PowerShell)

Ściąga z nauki zarządzania środowiskami Pythona. Główne narzędzie: **uv**.
Zakres: sesje 0–4 (instalacja, fundamenty środowisk, codzienna praca z projektem,
migracje cudzych repozytoriów, indeks firmowy, eksport, wheel i Databricks).

---

## 1. Instalacja i start projektu

### Instalacja uv (Windows)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# zamknij i otwórz terminal (najlepiej cały VS Code)
uv --version
uv self update          # aktualizacja (tylko przy instalatorze standalone)
```

- Instaluje się do `%USERPROFILE%\.local\bin` (dopisywane do PATH)
- Pythony zarządzane przez uv: `%APPDATA%\uv\python`
- Cache: `%LOCALAPPDATA%\uv\cache` → trzymaj projekty na tym samym dysku (hardlinki zamiast kopiowania)

### Nowy projekt

```powershell
uv python install 3.12
uv init --python 3.12 nazwa             # aplikacja: bez build-system, nie instalowana jako pakiet
uv init --lib --python 3.12 nazwa       # biblioteka: src layout, build-system, py.typed
uv init --package --python 3.12 nazwa   # pakiet z entry pointem ([project.scripts])
```

| Wariant | Kiedy |
|---|---|
| app (domyślny) | pipeline, skrypty, kod uruchamiany w miejscu |
| `--lib` | kod do zaimportowania / zbudowania jako wheel |
| `--package` | pakiet z komendą CLI |

Uwaga: przy `--lib` / `--package` projekt jest budowany przy każdym `uv sync` → w repo MUSZĄ być `src/<pakiet>/` i plik wskazany w `readme`.

### Co commitować

| Commit | Nie commitować |
|---|---|
| `pyproject.toml`, `uv.lock`, `.python-version`, `src/`, `tests/`, `README.md`, `.gitignore` | `.venv/`, `dist/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`, pliki danych (`*.parquet`) |

### Git — start repo

```powershell
git config --global init.defaultBranch main   # jednorazowo; inaczej powstaje master
git init
git add .
git status          # przejrzyj: jest src/ i README.md, nie ma .venv i danych
git commit -m "Init project with uv"
git remote add origin https://github.com/<login>/<repo>.git
git push -u origin main
```

- `error: src refspec main does not match any` → brak lokalnej gałęzi `main` z commitem.
  Diagnoza: `git branch`, `git log --oneline`. Naprawa: `git branch -M main` albo wykonaj pierwszy commit.
- `git add .` + dobry `.gitignore` > wyliczanie plików (whitelista gubi `src/`, `README.md`).

### Test „czy repo działa u kogoś innego”

```powershell
git clone C:\sciezka\repo C:\temp\check
cd C:\temp\check
uv sync --locked
uv run pytest
cd ..; Remove-Item C:\temp\check -Recurse -Force
```

---

## 2. Który Python? (diagnoza na Windows)

| Pytanie | Polecenie |
|---|---|
| Wszystkie `python.exe` w PATH (wygrywa pierwszy) | `where.exe python` (`where` w PS to alias Where-Object!) |
| Interpretery launchera `py` (własne reguły, nie PATH) | `py -0p` |
| Wszystko, co widzi uv | `uv python list` |
| Co faktycznie uruchomi `python` | `Get-Command python \| Select-Object Source` |
| Który interpreter i skąd | `python -c "import sys; print(sys.executable, sys.version)"` |
| Pełny obraz sys.path i user site | `python -m site` |
| Który pip i do jakiego Pythona | `where.exe pip`, `pip --version` |

- Alias `WindowsApps\python.exe` może prowadzić do prawdziwego interpretera (Store / Python install manager)
  → Ustawienia → Aplikacje → Zaawansowane → Aliasy wykonywania aplikacji.

---

## 3. Środowisko wirtualne — jak to działa

### venv pod maską

- venv = `pyvenv.cfg` + `Scripts\` + `Lib\site-packages\`; biblioteka standardowa NIE jest kopiowana
- Interpreter przy starcie szuka `pyvenv.cfg` obok exe lub poziom wyżej → **ustawia** `sys.prefix` na katalog venv
  (`home` = instalacja bazowa, z niej stdlib)
- Test w kodzie: `sys.prefix != sys.base_prefix` → jesteś w venv
- `include-system-site-packages = false` → brak globalnych pakietów i user site-packages
- uv: `home` wskazuje katalog minor (`cpython-3.12-...`, junction) → upgrade łatki Pythona nie psuje venv
- **venv nie jest przenośny** (ścieżki absolutne w `.pth`, `direct_url.json`, launcherach).
  Nie kopiuj, nie commituj — odtwórz przez `uv sync` (sekundy)

### sys.path — kolejność (pierwsze trafienie wygrywa)

1. `sys.path[0]`:
   - `python -c`, `python -m`, REPL → **bieżący katalog (CWD)**
   - `python sciezka\skrypt.py` → **katalog skryptu**
   - `pytest`, `ruff` (launchery) → lokalizacja launchera
2. `PYTHONPATH` — **przebija izolację venv**
3. biblioteka standardowa
4. site-packages (+ ścieżki z plików `.pth`, doklejane na końcu)

Obrona:
- `python -P` / `PYTHONSAFEPATH=1` — bez `sys.path[0]`
- `python -I` — tryb izolowany: bez CWD, `PYTHONPATH` i user site

Pułapki przesłaniania:
- plik `pydantic.py` / `json.py` / `logging.py` w projekcie przesłania prawdziwy moduł
- `uv run pytest` ≠ `uv run python -m pytest` (drugie dokleja CWD)
- pytest (domyślny tryb importu) dokleja katalog z testami
- Airflow / Databricks / obrazy Dockera często ustawiają `PYTHONPATH` → sprawdzaj przy dziwnych importach

### Aktywacja

- Zmienia TYLKO `PATH`, `VIRTUAL_ENV` i prompt — Pythona nie dotyka
- Nie jest potrzebna: `.venv\Scripts\python.exe` wywołany pełną ścieżką = to samo środowisko
- `uv run` = aktywacja na jedno polecenie + sprawdzenie zgodności z lockiem
- **Przecieka między projektami** po `cd` (to tylko zmienne powłoki)
- uv ignoruje niepasujący `VIRTUAL_ENV` i ostrzega; `--active` wymusza użycie aktywnego środowiska
- VS Code aktywuje venv wybranego interpretera → w każdym oknie: Select Interpreter → `.venv` TEGO projektu

### pip — pułapka w venv od uv

- venv tworzony przez uv **nie ma pipa** (uv instaluje sam; utrudnia obejście locka)
- `pip` = pierwszy `pip.exe` w PATH (z JEGO Pythonem)
  → w aktywnym venv bez pipa `pip install` trafia do **globalnego** Pythona, bez ostrzeżenia
- `python -m pip` = pip tego interpretera → w venv od uv pada głośno (`No module named pip`) — zawodzi bezpiecznie
- Potrzebny pip w venv (IDE, stare skrypty): `uv venv --seed` albo `uv add --dev pip`

| Polecenie | Zmienia pyproject/lock? | Przetrwa `uv sync`? |
|---|---|---|
| `uv add x` | tak | tak |
| `uv pip install x` | nie | **nie** (`uv sync` działa w trybie exact) |
| `pip install x` (w venv od uv) | nie | trafia do globalnego Pythona |

- `uv sync --inexact` — zachowaj pakiety spoza locka (rzadko, świadomie)
- `uv run` synchronizuje BEZ usuwania nadmiarowych pakietów; usuwa je dopiero `uv sync`

---

## 4. Pakiety: wheel, sdist, metadane

### Wheel vs sdist

| | Wheel (`.whl`) | sdist (`.tar.gz`) |
|---|---|---|
| Co to | zbudowana dystrybucja (zip) | kod źródłowy + `pyproject.toml` |
| Instalacja | rozpakowanie do site-packages, **bez wykonywania kodu** | build backend (PEP 517) w izolacji z deps z `[build-system]` (PEP 518) |
| Ryzyko | niskie | wykonanie cudzego kodu, kompilator (C/Rust), niepinowane build deps |

- Tagi: `nazwa-wersja-python-abi-platforma.whl`
  - `pydantic-2.13.5-py3-none-any.whl` → czysty Python, jeden plik na wszystko
  - `pydantic_core-2.46.5-cp312-cp312-win_amd64.whl` → osobny wheel na każdą kombinację
- Osie mnożenia wheeli: wersja Pythona × system × architektura (x86_64 / ARM) × libc na Linuxie
  (`manylinux` = glibc, `musllinux` = Alpine) × warianty (free-threaded `cp313t`, PyPy)
- `abi3` = jeden wheel binarny dla wielu wersji Pythona (stabilne ABI)
- Brak wheela dla twojej kombinacji (np. nowy Python) → instalator sięga po sdist → kompilacja
- Obrona: uv `--no-build` / `no-build-package`, pip `--only-binary :all:`, hashe

### .dist-info (paszport zainstalowanego pakietu)

- `METADATA` — wersja, `Requires-Dist`, `Requires-Python`, treść README
- `RECORD` — lista wszystkich zainstalowanych plików + hashe (podstawa odinstalowania)
- `INSTALLER` — kto instalował (`uv` / `pip`)
- `REQUESTED` — pakiet zamówiony wprost (nie tranzytywny)
- `direct_url.json` — skąd instalowano (editable → ścieżka źródła)

### Instalacja editable

- `uv sync` instaluje sam projekt jako editable
- W site-packages plik `<pakiet>.pth` ze ścieżką do `src/` → moduł `site` dopisuje ją do `sys.path`
  **raz, przy starcie interpretera** (import potem zwyczajnie przeszukuje `sys.path`)
- Linie `.pth` zaczynające się od `import` są **wykonywane** przy każdym starcie → wektor ataku supply-chain
- Kopia projektu z `.venv` na tej samej maszynie → `python` z kopii ładuje kod z ORYGINAŁU (cichy błąd);
  `uv run` wykrywa to przez `direct_url.json` i przeinstalowuje projekt
- W Dockerze chcemy instalacji nieedytowalnej: `--no-editable`

### uv build

```powershell
uv build                                            # dist\*.tar.gz + dist\*.whl
uv run python -m zipfile -l (Get-ChildItem dist\*.whl).FullName
tar -tzf (Get-ChildItem dist\*.tar.gz).FullName
```

- uv buduje sdist, a potem wheel **z sdist** → weryfikuje kompletność sdist
- Daty `1980-01-01` w wheelu → powtarzalne buildy (ten sam kod = ten sam hash)
- Zawartość wheela / sdist zależy od backendu (`uv_build` ≠ hatchling ≠ setuptools)
- `dist/` ma własny `.gitignore` z `*`

### Wheel a lock (ważne dla Databricks)

- `METADATA` wheela zawiera **luźne** `Requires-Dist` z `pyproject.toml` (`pydantic>=2.13.5`),
  NIE wersje z `uv.lock` — lock nie podróżuje z wheelem
- Grupy (`dev`, `lint`) nie trafiają do wheela
- Instalacja wheela na klastrze = resolucja od nowa → inne wersje niż lokalnie
- Przypięcie: `uv export --no-dev` → plik constraints/requirements instalowany razem z wheelem
- `uv pip sync` NIE przyjmuje `uv.lock`, tylko pliki w formacie requirements
- Biblioteka: luźne zakresy (łączy się z innymi). Aplikacja: zamrożone wszystko (końcowy konsument)
- Databricks: runtime ma preinstalowane pandas/pyarrow/numpy → pełne przypięcie może się z nimi kłócić

### Wersje uniwersalnego locka

- `uv.lock` zawiera rozwiązanie dla WSZYSTKICH platform i wersji Pythona z `requires-python`
  (np. 64 wheele `pydantic_core` z URL i hashami — nic nie jest pobierane na zapas)
- Przy `uv sync` wybierany jest jeden wheel pasujący do bieżącej maszyny
- `colorama` w drzewie pytesta = zależność tylko na Windows (marker `sys_platform == "win32"`)

---

## 5. Codzienna praca z projektem

### pyproject.toml vs uv.lock

| | `pyproject.toml` | `uv.lock` |
|---|---|---|
| Rola | co akceptuję (deklaracja) | co dokładnie instaluję (zamrożenie) |
| Przykład | `pandas>=2.2` | `pandas==2.3.x` + hashe + wszystkie tranzytywne |
| Edytuje | ty lub `uv add/remove` | wyłącznie uv |
| Python | `requires-python` (zakres) | — (konkretną wersję przypina `.python-version`) |

Lock gwarantuje identyczne **wersje pakietów** na wszystkich platformach — nie biblioteki systemowe,
zmienne środowiskowe ani wynik kompilacji z sdist.

### Polecenia

| Problem | Polecenie |
|---|---|
| Dodać zależność runtime | `uv add pandas pyarrow` |
| Dodać z extra | `uv add "pandas[parquet]"` |
| Dodać narzędzie dev/test | `uv add --dev pytest ruff` |
| Dodać do własnej grupy | `uv add --group lint sqlfluff` |
| Uruchomić z grupą spoza domyślnych | `uv run --group lint sqlfluff lint` |
| Usunąć zależność | `uv remove requests` |
| Drzewo zależności | `uv tree` / `uv tree --depth 1` |
| Kto ciągnie pakiet X? | `uv tree --invert --package numpy` |
| Co jest nieaktualne? | `uv tree --outdated --depth 1` |
| Zaktualizować jeden pakiet | `uv lock --upgrade-package pandas` |
| Zaktualizować wszystko | `uv lock --upgrade` + pełne testy (rzadko, świadomie) |
| Uruchomić polecenie w środowisku | `uv run pytest`, `uv run python main.py` |

- Domyślnie synchronizowana jest tylko grupa `dev`; inne włączasz jawnie (`--group`, `--all-groups`)
- `uv sync --no-dev` → usuwa pytest/ruff (obraz produkcyjny)
- `Resolved 31` vs `Checked 19` → lock ma wszystkie grupy i platformy, instalowane jest tylko to, co trzeba
- `uv add` zawsze dopisuje dolną granicę (`>=`); jej brak = ślad ręcznej edycji

### Lock — trzy tryby

| Polecenie | Co robi z lockiem | Gdzie |
|---|---|---|
| `uv sync` / `uv run` | aktualizuje, jeśli nie pasuje do pyproject | lokalnie |
| `uv sync --locked` / `uv lock --check` | **błąd**, gdy lock ≠ pyproject | CI |
| `uv sync --frozen` | nie sprawdza pyproject, bierze lock jak jest | Docker, **tylko gdy CI sprawdziło `--locked`** |

- Ktoś dopisał zależność do pyproject bez `uv lock`:
  - `--locked` → błąd w CI ✅
  - `--frozen` → przechodzi, pakietu brak, projekt przebudowany z nową deklaracją
    → środowisko wewnętrznie niespójne, błąd dopiero przy imporcie na produkcji ❌
- NIGDY zwykłe `uv sync` w Dockerze — może zmienić lock wewnątrz obrazu
- `uv add ... --frozen` = celowe wprowadzenie dryfu (pyproject bez locka)

### Konflikt wersji — jak czytać komunikat

Schemat: **„A wymaga X, ja wymagam Y, X i Y są rozłączne → brak rozwiązania”**.

Przykład (`uv add "numpy<1.20"` przy `pandas>=3.0.6`):
> Wymagam pandas>=3.0.6. Ten pandas na Pythonie ≥3.14 wymaga numpy>=2.3.3. Wymagam też numpy<1.20.
> Zakresy rozłączne → pandas i mój pin się wykluczają → projekt nierozwiązywalny.

- `split (markers: python_full_version >= '3.14' ...)` = konflikt w INNEJ wersji Pythona niż używana
  (lock musi działać dla całego `requires-python`)
- Aplikacja w znanym środowisku (runtime Databricks, obraz Dockera) → można zawęzić:
  `requires-python = ">=3.12,<3.13"` — ale nie po to, żeby uciszyć prawdziwy konflikt
- Lock przy konflikcie pozostaje nietknięty (uv nie zapisuje niespójnego stanu)
- Opcje przy konflikcie z legacy:
  1. podnieść kod do nowej wersji (najlepsze długoterminowo)
  2. zejść z wersjami (lawina: stary pandas → stary pyarrow → stary Python)
  3. odizolować stary komponent (osobne środowisko / job / obraz)

### Zależności opcjonalne (pułapka DE)

- pandas instaluje się bez pyarrow; `to_parquet()` pada dopiero w runtime (`ImportError`)
- Deklaruj jawnie: `uv add pyarrow` albo `uv add "pandas[parquet]"` (extra mówi, po co)
- `--locked` tego NIE wykryje (lock jest spójny z pyproject, który po prostu nie deklaruje pyarrow)
- Ochrona: testy wykonujące realny zapis/odczyt (np. Parquet w `tmp_path`)

---

## 6. Narzędzia i skrypty

### Narzędzie w projekcie czy obok?

| Sytuacja | Gdzie |
|---|---|
| Wersja wspólna dla zespołu i CI (pytest, ruff, sqlfluff, **dbt + adapter**) | grupa w projekcie |
| Osobiste / jednorazowe | `uvx` lub `uv tool` |

```powershell
uvx ruff --version            # efemerycznie, bez instalacji
uv tool install sqlfluff      # izolowane środowisko + launcher w ~\.local\bin (odpowiednik pipx)
uv tool list
```

- Globalnie zainstalowany dbt = częsta przyczyna „u mnie działa” (wersja niezgodna z adapterem/projektem)

### Skrypty PEP 723 (jednorazowe zadania DE)

```powershell
uv init --script profile_csv.py --python 3.12
uv add --script profile_csv.py pandas
uv run profile_csv.py
```

- Zależności zapisane w samym pliku (blok `# /// script`)
- Działa wszędzie, gdzie jest uv — bez venv i `requirements.txt` (mail, VM z Airflow)

---

## 7. Ruff

```powershell
uv run ruff check            # linter
uv run ruff format           # formatuje (naprawia)
uv run ruff format --check   # tylko sprawdza — do CI
```

- Magic trailing comma: przecinek po ostatnim argumencie → ruff wymusza jeden element na linię
- Diff z identycznym tekstem = białe znaki (brak nowej linii na końcu pliku, spacje na końcu linii)
- VS Code: rozszerzenie Ruff + formatowanie przy zapisie

---

## 8. Cudze repo i migracje na uv

### Złota zasada

**Zmieniaj narzędzie, nie wersje.** Migracja = jeden commit z identycznymi wersjami.
Aktualizacja zależności = osobne PR-y, później. Inaczej przy awarii nie wiadomo, co ją spowodowało.

### Rozpoznanie repo

| W repo widzisz | Narzędzie | Deklaracja / lock |
|---|---|---|
| `requirements.txt` z luźnymi wersjami | pip | deklaracja; locka brak — prawdziwe wersje zna tylko serwer (`pip freeze`) |
| `requirements.in` + `.txt` z nagłówkiem `pip-compile` i `# via` | pip-tools | `.in` / `.txt`; lock **dla jednej wersji Pythona i jednej platformy** |
| `[tool.poetry.dependencies]` + `poetry.lock` | Poetry 1.x | pyproject / `poetry.lock` |
| `[project]` **i** `[tool.poetry]` + `poetry.lock` | Poetry 2.x | jak wyżej; zależności w stylu `"pandas (>=2.1,<3.0)"` |
| `Pipfile` + `Pipfile.lock` | Pipenv | jak wyżej |
| `.python-version` bez `uv.lock` | pyenv | uv czyta ten sam plik |
| `setup.py` / `setup.cfg` | stary setuptools | deklaracja w kodzie |
| `environment.yml` | conda | **poza zasięgiem uv** (biblioteki niepythonowe) |

Checklist przy przejmowaniu repo:
- porównaj zadeklarowane zależności z faktycznymi importami
  (`Select-String -Path src\**\*.py -Pattern "^(import|from) "`) — komentarze bywają nieaktualne
- import to punkt startu, nie werdykt: `pyarrow` nie jest importowany, a pandas go używa
- nieużywana zależność → NIE usuwaj w migracji; odnotuj i potwierdź z zespołem (osobny PR)
- sprawdź, jak zespół uruchamiał testy (`python -m pytest` maskuje brak `pythonpath`)
- sprawdź, czy na produkcji nie ma narzędzi dev (mieszany `requirements.txt`)

### Operatory Poetry → PEP 440

| Poetry | PEP 440 | Uwaga |
|---|---|---|
| `^2.1` | `>=2.1,<3.0` | zgodne z major |
| `^0.1` | `>=0.1,<0.2` | przy 0.x `^` blokuje **minor** |
| `^0.0.3` | `>=0.0.3,<0.0.4` | przy 0.0.x blokuje patch |
| `~2.31` | `>=2.31,<2.32` | |
| `~2` | `>=2,<3` | |
| PEP 440 `~=2.31` | `>=2.31,<3.0` | **≠ Poetry `~2.31`** — ręczne tłumaczenie `~` → `~=` poluzowuje zakres |

### Mapa poleceń

| Stare | uv |
|---|---|
| `python -m venv venv` + `pip install -r requirements.txt` | `uv sync` / `uv venv` + `uv pip install -r` |
| `pip-compile requirements.in` | `uv lock` / `uv pip compile requirements.in --universal -o requirements.txt` |
| `pip-sync` | `uv sync` / `uv pip sync` |
| `pyenv install 3.11` / `pyenv local 3.11` | `uv python install 3.11` / `uv python pin 3.11` |
| `pipx install X` / `pipx run X` | `uv tool install X` / `uvx X` |
| `poetry install --with dev` | `uv sync` |
| `poetry install --with lint` / `-E parquet` | `uv sync --group lint` / `uv sync --extra parquet` |
| `poetry add` / `remove` / `lock` / `run` | `uv add` / `remove` / `lock` / `run` |
| `poetry update` | `uv lock --upgrade` + `uv sync` |
| `poetry show --tree` | `uv tree` |

Tryb zgodności (`uv pip ...`) = krok pośredni, gdy klient nie chce pełnej migracji.

### Migracja: requirements.txt + freeze z produkcji

```powershell
uv init --bare --python 3.11
# pyproject: requires-python = ">=3.11,<3.12"  (wersja z produkcji)
uv add pandas pyarrow requests python-dotenv -c prod-freeze.txt
uv add --dev pytest ruff -c prod-freeze.txt
uv sync
Compare-Object (Get-Content prod-freeze.txt) (uv pip freeze)
```

- `-c` = constraints: wersje z pliku, ale tylko dla pakietów faktycznie potrzebnych
- Constraints potrzebne raz: lock jest „lepki” — kolejne `uv lock` preferują zapisane wersje
- Nie deklaruj zależności tranzytywnych „dla kontroli wersji” — kontrolę daje lock.
  Wyjątek: musisz OGRANICZYĆ wersję (np. `numpy<2`) — wtedy z komentarzem dlaczego
- Flat layout bez pakietu: `[tool.pytest.ini_options] pythonpath = ["."]`
- Dopisz `.venv/` do `.gitignore` (legacy często ma tylko `venv/`)

### Migracja: pip-tools

```powershell
uv init --bare --python 3.11
uv add -r requirements.in -c requirements.txt
uv add --dev -r requirements-dev.in -c requirements-dev.txt
```

- Warstwowanie `-c requirements.txt` w `.in` niepotrzebne — jeden lock, wszystkie grupy spójne
- Locki pip-tools kompilowane na Linuksie nie mają `colorama` (Windows) → uniwersalny lock to naprawia
- Konfiguracja narzędzi (`pytest.ini`) zostaje — uv zarządza zależnościami, nie konfiguracją

### Migracja: Poetry

```powershell
uv tool install poetry                    # Poetry jako narzędzie, nie zależność
poetry lock; poetry install --with dev
poetry show --only main | Out-File before-main.txt
poetry show --only dev  | Out-File before-dev.txt   # zapisz też dev!
poetry env remove --all                   # PRZED migracją (potem Poetry nie odczyta konfiguracji)
uvx migrate-to-uv --dry-run
uvx migrate-to-uv
uv python pin 3.11                        # ten sam Python, którego używało Poetry
```

- `migrate-to-uv` tłumaczy operatory, grupy (opcjonalne → poza `default-groups`), extras, skrypty;
  przy złożonym `packages = [...]` zmienia backend na hatchling → sprawdź zawartość wheela
- **`--ignore-locked-versions` = rezygnacja z wersji, NIE rozwiązanie.**
  Gdy lock nie przechodzi (np. kilka wersji numpy dla różnych Pythonów → sprzeczne constraints):

```powershell
Get-Content before-main.txt | Where-Object { $_.Trim() } |
    ForEach-Object { $p = $_.Trim() -split '\s+'; "$($p[0])==$($p[1])" } |
    Set-Content constraints.txt
uv add "pandas>=2.1,<3" ... -c constraints.txt     # te same deklaracje co w pyproject
uv sync --no-dev
Compare-Object (Get-Content constraints.txt) (uv pip freeze)
```

- `pyarrow==(!)` w zrzucie = `(!)` w `poetry show` oznacza „zadeklarowany, niezainstalowany” (extra)
- Weryfikację rób PRZED usunięciem plików pomocniczych
- Jeśli wersji nie da się odtworzyć → to aktualizacja, nie migracja: lista zmian, pełne testy, akceptacja klienta

### Commit i raport migracyjny

- Typ `build:` (zależności/system budowania), nie `feat:`
- Pierwsze zdanie: czy wersje się zmieniły i jak to zweryfikowano
- Każdy punkt: co + **dlaczego**; sekcja „Note”/„Follow-up” dla problemów i decyzji odłożonych
- `MIGRATION.md`: co zmieniono / jak zweryfikowano (i czego NIE) / znane problemy /
  kolejne kroki jako osobne PR-y / ściąga poleceń dla zespołu
- **Tylko fakty, które sprawdziłeś.** „Nie zweryfikowano” > ładnie brzmiące przypuszczenie

---

## 9. Indeks firmowy

### Konfiguracja

```toml
[[tool.uv.index]]
name = "firmowy"
url = "https://pkgs.dev.azure.com/<org>/<projekt>/_packaging/<feed>/pypi/simple/"
explicit = true

[tool.uv.sources]
de-python-libs = { index = "firmowy" }
```

- `explicit = true` → indeks używany WYŁĄCZNIE dla pakietów przypisanych w `sources`;
  zależności tranzytywne (np. pydantic) i tak idą z PyPI, nawet jeśli leżą w indeksie firmowym
- Lokalny katalog z wheelami jako indeks (test): `url = "C:/sciezka"`, `format = "flat"`
- Lock zapisuje źródło (`source = { registry = ... }`) i hash pliku

### Uwierzytelnianie

```powershell
$env:UV_INDEX_FIRMOWY_USERNAME = "dowolny"
$env:UV_INDEX_FIRMOWY_PASSWORD = "<token>"     # nazwa indeksu WIELKIMI, '-' → '_'
```

| Indeks | Login / hasło |
|---|---|
| Azure Artifacts | dowolny niepusty login + PAT (Packaging Read) |
| JFrog Artifactory | użytkownik + token |
| AWS CodeArtifact | `aws` + token z `get-authorization-token` (12 h) |

- Nigdy tokenu w `pyproject.toml` ani w URL; w CI z sekretów → te same zmienne
- Nowy pipeline pada na `uv sync`, a laptop działa → brak sekretu z danymi logowania

### Dependency confusion

- Atak: publiczny pakiet o nazwie wewnętrznego, z wysoką wersją
- `pip --extra-index-url` łączy indeksy i bierze najwyższą wersję → podatny
- uv: `index-strategy = "first-index"` (zatrzymuje się na pierwszym indeksie z pakietem);
  najsilniej: `explicit` + `sources`. `unsafe-best-match` = zachowanie pip
- **Granice ochrony:** znika po usunięciu wpisu w `sources` (wtedy pakiet z PyPI wygrywa po cichu)
  i znika w `requirements.txt` po eksporcie
- Praktyki: prefiks w nazwach + rezerwacja nazwy na PyPI; review zmian w `[[tool.uv.index]]`/`sources`;
  jeden firmowy indeks-proxy z PyPI jako upstream

### Wydanie nowej wersji biblioteki

```powershell
uv version --bump minor                          # w bibliotece
uv build --out-dir <indeks>                      # (realnie: uv publish)
uv lock --upgrade-package de-python-libs         # w projekcie konsumującym
```

- **Opublikowanej wersji nigdy się nie nadpisuje** — inny plik pod tą samą wersją = błąd hasha w locku
- Samo `uv lock` nie podniesie wersji (lock „lepki”) — aktualizacja to jawna decyzja
- Lock = czego używasz; dolna granica w pyproject = czego potrzebujesz.
  Używasz funkcji z 0.2.0 → `uv add "de-python-libs>=0.2.0"`

---

## 10. uv export (platformy bez uv)

```powershell
uv export --no-dev -o requirements.txt
uv export --no-dev --no-hashes --no-emit-project -o requirements.txt
```

| Flaga | Kiedy |
|---|---|
| `--no-dev` | zawsze dla produkcji |
| `--no-hashes` | platforma doinstaluje coś bez hasha (np. twój wheel) — pip z hashami: wszystko albo nic |
| `--no-emit-project` | projekt instalowany osobno jako wheel (Databricks) → bez linii `-e .` |
| `--only-group lint` | środowisko jednego narzędzia |
| `--format pylock.toml` | nowszy standard (PEP 751) |

- `uv.lock` = źródło prawdy; `requirements.txt` = artefakt, nigdy edytowany ręcznie
- Markery platform zostają — ocenia je pip na platformie docelowej
- **Eksport gubi konfigurację indeksów** → pakiet firmowy jako `nazwa==wersja` bez źródła;
  platforma musi znać indeks (`PIP_INDEX_URL`, ustawienia platformy, indeks-proxy).
  Unikaj `--extra-index-url`
- Pilnowanie rozjazdu:
  1. nie commitować — generować w buildzie (najczyściej)
  2. commitować + w CI: to samo polecenie co w nagłówku, potem `git diff --exit-code requirements.txt`
  3. hook pre-commit `uv-export` (`astral-sh/uv-pre-commit`)

---

## 11. Wheel i Databricks

### Gdzie kończy się uv

| Etap | Narzędzie | Co gwarantuje wersje |
|---|---|---|
| Rozwój, testy, CI | `uv sync` / `uv run` / `uv lock` | `uv.lock` |
| Granica | `uv build` (wheel), `uv export` (piny) | metadane wheela / wyeksportowane piny |
| Databricks | pip (`%pip`, środowisko joba) | nic, czego sam nie przekażesz — resolucja od nowa na preinstalowanych pakietach |

Na Databricks kod jest biblioteką, nawet jeśli to job: lock nie jedzie, deklaracje stają się kontraktem.

### Deklaracje w pakiecie pod Databricks

- **Dolna granica = minimum, którego kod potrzebuje**, NIE wartość z `uv add` (najnowsza wersja z dnia dodania)
- Zbyt wysoka dolna granica → pip podmienia preinstalowane pakiety (i ich zależności, np. `typing-extensions`)
- Górnych granic unikaj (blokują aktualizacje platformy); wyjątek: znana niezgodność
- **pyspark nigdy w `dependencies`** — Databricks dostarcza własną dystrybucję; pyspark z PyPI by ją przesłonił
- Lokalnie kod sparkowy: `databricks-connect` w osobnej grupie, wersja dopasowana do runtime'u
  (nie współistnieje z `pyspark` w jednym środowisku)
- Logikę oddzielaj od Sparka → testowalna bez Sparka; import pyspark wewnątrz funkcji
- `requires-python` musi pasować do Pythona w środowisku docelowym
- Usuń placeholder `description` z `uv init` (widoczny w indeksie i na platformie)

### Dowód, że dolna granica jest prawdziwa

```powershell
uv sync --resolution lowest-direct
uv tree --frozen --depth 1
uv run --frozen pytest          # --frozen OBOWIĄZKOWE
git restore uv.lock
uv sync
```

- Lock zapisuje `resolution-mode`; polecenie bez tej samej flagi uzna lock za nieaktualny
  i po cichu wróci do `highest` (uv: „Ignoring existing lockfile due to change in resolution mode”)
- `lowest-direct` obniża tylko zależności bezpośrednie; tranzytywne przypięte `==` idą za nimi
- Starsza wersja = czasem inny zestaw pakietów tranzytywnych, nie tylko inne numery

Test na wersjach z platformy (szybko):

```powershell
uv pip install "pydantic==2.8.2" "pandas==1.5.3"
uv run --no-sync pytest         # --no-sync: inaczej uv przywróci lock
uv sync
```

### Środowisko Pythona na Databricks

| Mechanizm | Zasięg | Free Edition |
|---|---|---|
| Wersja środowiska serverless / Databricks Runtime | Python + preinstalowane pakiety | tak (serverless) |
| `%pip install` | sesja notebooka | tak |
| Panel Environment / `environments` w jobie | notebook / zadanie serverless | tak |
| Biblioteki klastra | klasyczny klaster | nie |

- **„Serverless” to nie jedno środowisko** — `environment_version` decyduje o CAŁYM zestawie pakietów
  (zaobserwowane: notebook pandas 2.2.3 / pydantic 2.10.6, job env 3: pandas 1.5.3 / pydantic 2.8.2)
- Wersję środowiska trzymaj w `databricks.yml`; tę samą ustaw w notebooku deweloperskim
- Testuj na wersjach z JOBA, nie z notebooka
- `%pip` nie usuwa pakietów platformy (`Not uninstalling ... outside environment`) — instaluje
  do środowiska notebooka/joba, które przesłania bazowe (warstwy `sys.path`)
- Po `%pip` w notebooku: `dbutils.library.restartPython()` — inaczej stary moduł zostaje w pamięci
  (`pydantic.VERSION` ≠ `importlib.metadata.version("pydantic")`)
- Serverless działa na **aarch64** — zależności binarne muszą mieć wheele `manylinux_*_aarch64`
- Pakiety idą przez wewnętrzny indeks-proxy — pakiety firmowe wymagają konfiguracji repozytoriów workspace'u

Rozpoznanie środowiska (komórka notebooka / początek joba):

```python
import sys, importlib.metadata as md
print(sys.version)
for p in ["pydantic", "pandas", "pyarrow", "numpy"]:
    try: print(p, md.version(p))
    except md.PackageNotFoundError: print(p, "-")
```

### entry_points.txt

- Plik w `.dist-info` (INI): grupa → `nazwa = moduł:funkcja`; generowany z `[project.scripts]`
- `console_scripts` → instalator tworzy launcher (`.venv\Scripts\nazwa.exe` / `.venv/bin/nazwa`)
- Inne grupy = wtyczki odkrywane w runtime (`importlib.metadata.entry_points(group=...)`):
  pytest (`pytest11`), providery Airflow, dialekty SQLAlchemy
- Databricks `python_wheel_task`: `entry_point` = nazwa z LEWEJ strony `=`
- Funkcja wejścia nie dostaje argumentów → parametry przez `sys.argv` → `argparse`
- Wzorzec testowalny: `def main(argv: list[str] | None = None)` + `parser.parse_args(argv)`

### Ręczne wdrożenie (Volume)

```powershell
databricks volumes create workspace default wheels MANAGED
uv build --wheel
databricks fs cp dist\pakiet-0.1.0-py3-none-any.whl dbfs:/Volumes/workspace/default/wheels/
```

```python
%pip install /Volumes/workspace/default/wheels/pakiet-0.1.0-py3-none-any.whl
```

### Asset Bundle z wheelem budowanym przez uv

```yaml
artifacts:
  pakiet:
    type: whl
    build: uv build --wheel
    path: .

resources:
  jobs:
    pakiet_job:
      tasks:
        - task_key: run_wheel
          environment_key: default
          python_wheel_task:
            package_name: pakiet
            entry_point: pakiet-cli
            parameters: ["--fail-on-errors"]
      environments:
        - environment_key: default
          spec:
            environment_version: "3"      # starsze CLI: client: "3"
            dependencies:
              - ./dist/*.whl

targets:
  dev:
    mode: development
    default: true
    workspace:
      profile: <profil>                   # jawnie — chroni przed wdrożeniem w złe miejsce
```

```powershell
Add-Content .gitignore "`n.databricks/"
Remove-Item dist -Recurse -Force          # ./dist/*.whl łapie WSZYSTKIE wheele
databricks bundle validate
databricks bundle deploy                  # buduje wheel (uv build) + wysyła wheel + synchronizuje pliki projektu
databricks bundle run pakiet_job; $LASTEXITCODE
databricks bundle destroy
```

- `Files: N uploaded` = pliki projektu (z pominięciem `.gitignore`), nie zawartość wheela
- `mode: development` → prefiks `[dev użytkownik]`, katalog użytkownika, wstrzymane harmonogramy;
  produkcja: `mode: production`, wdrażane z CI (service principal)
- `bundle run` zwraca kod ≠ 0 przy nieudanym jobie → CI zatrzyma się samo

### Wersjonowanie

```powershell
uv version --bump patch     # przed KAŻDYM wdrożeniem zmienionego kodu
```

- pip nie przeinstaluje pakietu w już zainstalowanej wersji
- Klasyczne klastry: biblioteki instalowane przy starcie — podmiana pliku bez restartu nie działa
- Indeksy nie pozwalają nadpisać wersji; lock wykrywa inny plik pod tą samą wersją (hash)
- **Identyfikowalność**: job wypisuje swoją wersję → wiadomo, który kod się wykonał

### Raportowanie błędów w jobach

- `sys.exit(1)` na Databricks → „SystemExit: 1 … use %tb” (entry point działa w środowisku IPython);
  własny komunikat tylko w run output
- Lepiej: wyjątek z opisem (`class ValidationFailedError(RuntimeError)`) → treść w błędzie zadania,
  UI i powiadomieniach; lokalnie też kod wyjścia 1
- `sys.exit` zostaw dla narzędzi CLI; `INTERNAL_ERROR` = każdy nieobsłużony błąd w zadaniu wheel

---

## 12. Pułapki z praktyki (szybka diagnoza)

| Objaw | Przyczyna | Rozwiązanie |
|---|---|---|
| `pip install` OK, a `import` w projekcie pada | pip z PATH należy do innego Pythona | `uv add`; diagnoza `pip --version` |
| `ModuleNotFoundError` po `python -c` | `python` ≠ Python projektu (PATH) | `uv run python ...` |
| Import ładuje „dziwny” moduł | przesłanianie (CWD, `PYTHONPATH`, plik o nazwie biblioteki) | `python -I`, `envinfo.py` |
| Pakiet zniknął po `uv sync` | instalowany przez `uv pip install`, spoza locka | `uv add` |
| Ostrzeżenie `VIRTUAL_ENV=... does not match` | aktywny venv innego projektu | `deactivate`; Select Interpreter w VS Code |
| Edytuję kod w kopii, a testy go nie widzą | editable wskazuje oryginał | `uv run` / `uv sync` w kopii |
| CI: `lockfile needs to be updated` | pyproject zmieniony bez `uv lock` | `uv lock` + commit `uv.lock` |
| Klon repo: błąd budowania projektu | brak `src/` / `README.md` w commicie | `git add .` + `git status` |
| `ImportError: pyarrow` w nocnym jobie | zależność opcjonalna niezadeklarowana | `uv add pyarrow` + test I/O |
| Instalacja nagle wymaga kompilatora | brak wheela dla tej wersji Pythona/platformy → sdist | zmień wersję Pythona lub pakietu, `--no-build` |
| `uv run pytest` pada na imporcie własnego modułu po migracji | flat layout; zespół uruchamiał `python -m pytest` | `[tool.pytest.ini_options] pythonpath = ["."]` |
| `migrate-to-uv`: sprzeczne constraints z `poetry.lock` | lock Poetry ma kilka wersji pakietu dla różnych Pythonów | constraints ze zrzutu środowiska, nie `--ignore-locked-versions` |
| Po migracji inny Python niż wcześniej | brak `.python-version`; uv wybrał inną wersję | `uv python pin <wersja>` |
| `de-python-libs was not found in the package registry` | brak przypisania w `[tool.uv.sources]` do indeksu explicit | przywróć wpis w `sources` |
| Błąd hasha pakietu firmowego | przebudowano wheel bez podbicia wersji | `uv version --bump` przed każdym buildem |
| `uv lock` nie widzi nowej wersji biblioteki | lock preferuje zapisane wersje | `uv lock --upgrade-package <pakiet>` |
| Platforma bez uv nie znajduje pakietu firmowego | eksport gubi konfigurację indeksów | indeks w konfiguracji pip / platformy |
| `git diff --exit-code requirements.txt` = 1 w CI | zależności zmienione bez ponownego eksportu | `uv export` tym samym poleceniem + commit |
| Test `lowest-direct` przechodzi „podejrzanie łatwo” | polecenie bez `--frozen` przywróciło `highest` | `uv run --frozen pytest` |
| `uv run` lokalnie: `No module named 'pyspark'` | pyspark celowo niezadeklarowany | logika poza Sparkiem; databricks-connect w osobnej grupie |
| Job na Databricks ma inne wersje niż notebook | inna `environment_version` | ta sama wersja środowiska w bundle i notebooku |
| `%pip install` podmienia `typing-extensions` i inne | za wysoka dolna granica w wheelu | dolna granica = minimum dla kodu |
| Po `%pip` stara wersja pakietu w notebooku | moduł w pamięci | `dbutils.library.restartPython()` |
| Poprawka nie działa po wdrożeniu | ta sama wersja wheela | `uv version --bump patch` |
| Job instaluje kilka wersji wheela | `./dist/*.whl` łapie stare pliki | wyczyść `dist\` przed deploy |
| `bundle validate`: `no such host` | DNS / sieć, nie konfiguracja | sieć, `Resolve-DnsName`, `ipconfig /flushdns` |
| Alert z joba: „SystemExit: 1” bez treści | `sys.exit` w entry poincie | wyjątek z opisowym komunikatem |

### Narzędzie diagnostyczne

`scripts/envinfo.py` w repo `de-python-libs`:

```powershell
uv run python -m scripts.envinfo <modul>
```

Pokazuje: interpreter, venv (`sys.prefix` vs `base_prefix`), `VIRTUAL_ENV`, `PYTHONPATH`, flagi
`isolated`/`safe_path`, `sys.path` w kolejności, źródło modułu (bez importu), dystrybucję i wersję,
instalację editable oraz wykrywanie przesłaniania (kod wyjścia 1).
Uruchamiaj przez `-m`, żeby `sys.path[0]` był taki sam jak w diagnozowanym przypadku.