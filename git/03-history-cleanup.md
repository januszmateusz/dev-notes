# Git: porządkowanie historii

## Po co
Pracuj chaotycznie, publikuj porządnie. Czysta historia = czytelne review,
precyzyjny revert, skuteczny bisect, zrozumiałe `git log` za pół roku.
Zasada: sprzątasz TYLKO swój branch, zanim inni na nim zbudują (złota zasada).

## Interactive rebase
`git rebase -i <base>` = zwykły rebase z edycją listy przed wykonaniem.
- `<base>` jest wykluczający: `main` → commity `main..HEAD`; `HEAD~3` → 3 ostatnie.
- Lista: NAJSTARSZY NA GÓRZE (kolejność wykonywania). Plik: `.git/rebase-merge/git-rebase-todo`.

| Komenda  | Efekt                                                   |
|----------|---------------------------------------------------------|
| `pick`   | weź bez zmian                                           |
| `reword` | zmień message                                           |
| `edit`   | zatrzymaj się po nałożeniu (zmiana treści, podział)     |
| `squash` | połącz z linią POWYŻEJ, oba message'e do edycji         |
| `fixup`  | połącz z linią POWYŻEJ, message tego commita wyrzuć     |
| `drop`   | wyrzuć (= usunięcie linii)                              |
| `exec`   | uruchom komendę po poprzednim kroku                     |
| kolejność linii | kolejność nakładania                             |

- Commit identyczny z oryginałem (ten sam rodzic, bez zmian) zachowuje hash.
  Od pierwszej zmiany w górę: nowe hashe (nowy `parent`), nawet bez zmian treści.
- Przestawianie: niezależne commity (różne pliki) swobodnie, zależne ostrożnie.
- NIE `drop`uj commitów, na których coś dalej buduje (np. "wip") → łącz je.
  Typowy skutek: konflikt modify/delete.
- Wewnątrz rebase pracujesz w detached HEAD; branch przesuwa się dopiero na końcu,
  więc `git rebase --abort` działa na każdym etapie.

### Podział commita
```bash
git rebase -i main          # pick → edit na commicie do podziału
git reset HEAD~             # zmiany z commita wracają na dysk (niezacommitowane)
git add <pliki_1>  && git commit -m "..."
git add -p <plik>  && git commit -m "..."   # -p: wybór hunków w jednym pliku
git rebase --continue
```

## Amend
`git commit --amend` = NOWY commit z tym samym rodzicem co ostatni.
- `--no-edit` zostaw message; `-m "..."` nowy message.
- Amend NIE ustawia ORIG_HEAD. Stara wersja: `HEAD@{1}` / `git reflog`.

## Fixup + autosquash (poprawki po review)
```bash
git branch backup/before-fixup
git add <konkretne_pliki>          # NIE `git add .` / `commit -a`
git diff --staged                  # tylko ta poprawka?
git commit --fixup=<sha_celu>      # message: "fixup! <subject celu>"
git log --stat main..feature       # para message ↔ pliki się zgadza?
git rebase -i --autosquash main
git log --stat main..feature       # poprawka siedzi we właściwym commicie?
git diff backup/before-fixup feature   # = dokładnie poprawki, nic więcej
```
- Autosquash dopasowuje po MESSAGE. Zły hash przy `--fixup` = poprawka w złym commicie.
- Warianty: `--fixup=amend:<c>`, `--fixup=reword:<c>`, `--squash=<c>`.
- `git config --global rebase.autoSquash true`
- Bez oglądania listy: `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash main`

## Weryfikacja po sprzątaniu
| Test | Sprawdza |
|---|---|
| `git diff backup feature` | stan KOŃCOWY (pusty przy samej reorganizacji; = nowe zmiany przy fixupach) |
| `git log --stat` / `git show` per commit | czy każda zmiana jest w swoim commicie |
| `git range-diff backup...feature` | różnice między dwiema wersjami serii commitów |
| `git rebase -i --exec "pytest -q" main` | czy KAŻDY commit pośredni działa |
| diffstat | zdrowy rozsądek: "jedna funkcja, a 8 linii?" |

## Commit messages
```
<typ>(<zakres>)!: <opis w trybie rozkazującym, ~50 znaków, bez kropki>

<body: DLACZEGO, nie CO; zawijaj na ~72 znakach>

BREAKING CHANGE: <co musi zrobić konsument>
Refs: <ticket>
```
- Test subjectu: "If applied, this commit will ___".
- Opisz zachowanie ("return empty list for empty file"), nie "handle"/"update".
- Podawaj konkrety: nowa nazwa, wersja docelowa.
- Zakres = obszar/moduł (stabilny), nie nazwa funkcji.
- Jeden commit = jedna logiczna zmiana ("and" w subject → pewnie dwa commity).

| Typ | Kiedy | semver |
|---|---|---|
| feat | nowa funkcjonalność | minor |
| fix / perf | błąd / wydajność | patch |
| refactor, test, docs, build, ci, chore | bez zmiany zachowania dla użytkownika | brak |
| `!` / `BREAKING CHANGE:` | niekompatybilne wstecz | major |

**DE:** zmiana nazwy/typu kolumny, usunięcie modelu, zmiana ziarna = BREAKING
dla konsumentów downstream. Zależne modele w tym samym repo aktualizuj w TYM SAMYM
commicie (inaczej `dbt build` pada → psuje bisect/revert).

## Pułapka "Squash and merge"
GitHub tworzy na main JEDEN nowy commit S z jednym rodzicem (lokalnie: `git merge --squash`
+ `git commit`). Treść S = stan brancha, ale commity brancha NIE są przodkami S.
Git ocenia "zmergowane" po GRAFIE, nie po treści:
- `git branch -d` odmawia → `git branch -D`
- `main..feature` dalej pokazuje stare commity
- dalsza praca na tym branchu → nowy PR pokazuje stare commity; merge daje konflikty (add/add)

**Profilaktyka:** po squash merge usuń branch; nowa praca od świeżego main.
GitHub: Settings → Automatically delete head branches.

**Naprawa:** przenieś TYLKO nowe commity:
```bash
git rebase --onto main <sha_ostatniego_zsquashowanego> feature
# albo:
git switch main && git pull && git switch -c feature-2
git cherry-pick <sha_nowy_1> <sha_nowy_2>
```
- `git rebase --onto <nowa_baza> <stara_baza> <branch>` = commity `stara_baza..branch` na `nowa_baza`.
- Starą bazę podawaj HASHEM. Względne `feature~1` zmienia znaczenie po każdej operacji.
- Zwykły `git rebase main` odtworzyłby też stare commity → konflikty z już zmergowaną pracą.