# Git: merge, rebase, konflikty

## Merge base
Najlepszy wspólny przodek dwóch branchy: `git merge-base A B`.
Od niego zaczyna się każdy merge i każdy rebase.

## Merge
- **Fast-forward**: branch docelowy JEST merge base (nie ma własnych commitów)
  → przesuwa się tylko wskaźnik, 0 nowych obiektów, brak śladu brancha w historii.
- **3-way merge**: base + ours (HEAD) + theirs, reguła per hunk:
  zmiana po jednej stronie → wygrywa automatycznie; obie strony identycznie → OK;
  obie strony różnie → KONFLIKT. Base mówi, KTO co zmienił.
- Wynik 3-way = merge commit z 2 rodzicami. `^1` = branch, na którym byłeś; `^2` = wciągnięty.
- `git log --first-parent` = linia brancha bez wnętrza zmergowanych branchy
  (na main z merge commitami: jeden wpis na PR).
- Nowe obiekty przy mergeu: nowe tree dla każdego katalogu z nową kombinacją
  plików + commit. Bloby zwykle już istnieją.

| Flaga        | Zachowanie                                          |
|--------------|-----------------------------------------------------|
| (domyślnie)  | FF jeśli się da, inaczej 3-way                      |
| `--no-ff`    | zawsze merge commit (= GitHub "Create a merge commit") |
| `--ff-only`  | tylko FF, inaczej odmowa (bezpieczna aktualizacja)  |

## Rebase (`git rebase main` będąc na feature)
1. merge base → 2. lista `main..feature` → 3. HEAD (detached) na czubek main
→ 4. każdy commit nakładany jako patch = NOWY commit (nowy parent → nowy hash)
→ 5. `feature` przesunięty na ostatni nowy commit.
- Ruszane są TYLKO commity bieżącego brancha. Commity main bez zmian.
- Stare commity zostają (ORIG_HEAD, reflog), nieosiągalne z brancha.
- Author date zachowany, committer date = teraz.
- Po rebase merge do main = fast-forward → liniowa historia.
- Każdy rebase = lista `pick` (widać w `git status`); interactive rebase = edycja tej listy.
- W każdym `pick` base = RODZIC nakładanego commita.

## Złota zasada
Nie rebase'uj commitów, które ma już ktoś inny (main, develop, release, współdzielone branche).
Rebase własnego feature brancha po pushu → OK, ale push z `--force-with-lease`.

| Sytuacja                                   | Wybór           |
|--------------------------------------------|-----------------|
| Aktualizacja własnego feature o main       | rebase          |
| Branch współdzielony                       | merge           |
| main / develop / release                   | nigdy rebase    |
| Lokalne niewypchnięte commity przy pull    | `git pull --rebase` |
| Długi branch, dużo konfliktów              | często merge (konflikty raz) |

## Konflikty
- Merge zatrzymuje się PRZED commitem. Merge commit jeszcze nie istnieje.
  Ślady: prompt `|MERGING`, `.git/MERGE_HEAD`.
- Index: plik ma 3 wpisy, stage 1 base, 2 ours, 3 theirs (pełne wersje pliku!).
  Podgląd: `git show :1:plik`, `:2:`, `:3:`.
- Rozwiązanie: popraw plik → `git add` (stage 1/2/3 → stage 0) → `git commit` / `--continue`.
- Wyjście: `git merge --abort`, `git rebase --abort` (powrót do stanu sprzed operacji).
- Rebase: konflikt może wystąpić przy KAŻDYM commicie; `--continue`, `--skip`, `--abort`.
- `git config --global merge.conflictStyle zdiff3` → markery pokazują też base.

## ours / theirs
**ours = to, na co wskazuje HEAD.**
| Operacja                         | ours                | theirs            |
|----------------------------------|---------------------|-------------------|
| `git merge feature` (na main)    | main                | feature           |
| `git rebase main` (na feature)   | main / nowa baza    | TWÓJ commit       |

- `git checkout --ours/--theirs plik` bierze CAŁY plik, nie tylko konfliktowy blok:
  - rebase + `--ours` → tracisz własną zmianę (pusty commit),
  - rebase + `--theirs` → po cichu cofasz zmiany z main w liniach bez konfliktu.
- Przyciski IDE działają zwykle per blok konfliktu, ale ours/theirs mają to samo znaczenie.
- **Nawyk: przed `--continue` / `commit` zawsze `git diff --staged`.**

## Komendy
```bash
git merge-base A B
git merge-base --is-ancestor A B && echo "A jest przodkiem B"
git log --oneline main..feature      # commity na feature, których nie ma main
git log --oneline --first-parent
git rev-parse HEAD^1 HEAD^2          # rodzice merge commita
git log --oneline --graph --all ORIG_HEAD   # stan sprzed rebase/merge obok obecnego
git range-diff A..B C..D             # porównanie dwóch serii commitów (review po rebase)
GIT_EDITOR=true git rebase --continue   # bez otwierania edytora
```