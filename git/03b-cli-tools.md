# Git: narzędzia CLI (delta, tig, lazygit)

## Role
| Rola | Narzędzie | Co zmienia |
|---|---|---|
| pager | delta | tylko WYGLĄD diffów w `diff`/`show`/`log -p` |
| przeglądarka | tig | szybkie czytanie historii, drzew, blame |
| operacje | lazygit | staging linii, commit, rebase, push z klawiatury |

Wszystkie wywołują pod spodem zwykłego Gita. Każdy skrót = znana operacja
(reword starszego commita = rebase + amend + pick; reword ostatniego = amend).

## Instalacja (Windows)
```powershell
winget search <nazwa>
winget install --id dandavison.delta -e
winget install --id JesseDuffield.lazygit -e
```
- tig jest zwykle w Git for Windows (`tig --version`).
- Po instalacji ZAMKNIJ CAŁĄ aplikację terminala (i VS Code): proces dziedziczy
  PATH przy starcie, nowe karty dziedziczą stary PATH od rodzica.
- Diagnoza: PowerShell `(Get-Command delta).Source` vs bash `which delta`.
- Jeśli `core.pager=delta`, a delta niedostępna → KAŻDE `git diff/log/show` pada
  (`cannot spawn delta`). Awaryjnie: `git config --global --unset core.pager`.

## Aliasy
```bash
git config --global alias.lg  "log --oneline --graph --all --decorate"
git config --global alias.objs "cat-file --batch-all-objects --batch-check"
git config --global alias.sbs '-c delta.side-by-side=true show'
```

## delta
```bash
git config --global core.pager delta
git config --global interactive.diffFilter "delta --color-only"   # git add -p
git config --global delta.navigate true                           # n / N
git config --global delta.line-numbers true
git -c delta.side-by-side=true show HEAD                          # jednorazowo
```
- Git uruchamia pager TYLKO gdy stdout to terminal (TTY).
  `| grep`, `| xargs`, `> plik.patch` → czysty tekst, bez pagera i kolorów.
  Skrypty i CI są nietknięte.
- Bez pagera: `git --no-pager <komenda>`.

## tig
```bash
tig                       # historia z HEAD (domyślnie, czysty widok)
tig --all                 # wszystkie refy (relacje branchy, po fetch)
tig main..feature         # przyjmuje argumenty git log
tig --first-parent main
tig -- src/loader.py      # historia pliku
tig blame src/loader.py
tig status
```
| Klawisz | Widok |
|---|---|
| `m` | main (commity + graf) |
| `Enter` | diff commita / treść pliku |
| `t` | tree commita (= obiekt tree) |
| `r` | refs |
| `q` | wstecz / wyjście |
| `h` | pomoc |
- Krzaki zamiast ramek: `export LANG=en_US.UTF-8` w `~/.bashrc`
  albo `set line-graphics = ascii` w `~/.tigrc`.
- Blame po squash merge: wszystkie linie z PR → jeden commit S (utrata szczegółów).

## lazygit
Panele: 1 Status, 2 Files, 3 Branches, 4 Commits, 5 Stash. `?` = aktualne skróty.
Commity jak `git log`: NAJNOWSZY NA GÓRZE.

| Klawisz (Commits) | Operacja |
|---|---|
| `r` / `R` | reword (inline / w edytorze) |
| `f` | fixup do commita PONIŻEJ (= linia powyżej w liście rebase) |
| `s` | squash do commita poniżej |
| `d` | drop |
| `Ctrl+k` | przesuń w górę |
| `Ctrl+j` | przesuń w dół (często nie działa: Ctrl+J = LF = Enter) |
| `z` | undo (przez reflog) |
| `m` | menu rebase (continue / abort) |

- Brak `Ctrl+j`: przesuń w górę commit, który ma być wyżej (zamiana sąsiadów jest symetryczna).
- Files: `Space` stage pliku, `Enter` → staging linii/hunków (największy zysk vs CLI).
- Wklejanie: obsługuje terminal (mintty `Shift+Insert`, Windows Terminal `Ctrl+V`).
  Nie kopiuj końca linii, bo zadziała jako Enter.

## Odzyskiwanie po pomyłce w narzędziu
```bash
git log -g --oneline | grep "<fragment message>"   # przejdź reflog jak historię
git branch rescue <hash>                            # zabezpiecz
git cherry-pick <hash>                              # weź TYLKO ten commit
```