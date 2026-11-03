# Git: model mentalny (obiekty, refs, HEAD, index)

## Jedno zdanie
Git = baza klucz-wartość adresowana treścią (obiekty) + graf snapshotów (DAG)
+ przesuwalne wskaźniki (refs). Każda komenda to pytanie: jakie obiekty powstają
i który wskaźnik przesuwa się gdzie.

## Obiekty (`.git/objects`, niezmienne, klucz = SHA-1 z `typ rozmiar\0treść`)
| Typ    | Zawiera                                   | NIE zawiera              |
|--------|-------------------------------------------|--------------------------|
| blob   | treść pliku                               | nazwy, uprawnień, daty   |
| tree   | wpisy `tryb typ hash nazwa` (katalog)     | treści plików            |
| commit | tree root, parent(s), autor, data, message | diffa                    |
| tag    | wskaźnik na obiekt + opis (adnotowany)    |                          |

- Ta sama treść = ten sam hash = jeden obiekt (deduplikacja).
- Commit = **pełny snapshot**, ale tani: niezmienione pliki współdzielą bloby.
- **Diff nie jest zapisany**, liczy się na żądanie z dwóch snapshotów.
  (Packfile'y kompresują deltami, ale to optymalizacja, nie model.)
- Tryby: `100644` plik, `100755` wykonywalny, `040000` tree, `120000` symlink, `160000` submoduł.

## Reguła bąbelkowania
Zmiana/dodanie/usunięcie pliku → nowy blob (jeśli nowa treść)
→ nowe tree dla KAŻDEGO katalogu na ścieżce do roota → nowy commit.
Przykład: `models/staging/airline/x.sql` = 1 blob + 4 tree + 1 commit.

## Niezmienność = „przepisanie historii” tworzy NOWE commity
Hash commita zależy od rodzica i daty → amend/rebase/filter-repo tworzą nowe commity,
stare zostają w bazie (nieosiągalne) do czasu `git gc`.

## Refs i HEAD
- Branch = plik `.git/refs/heads/<nazwa>` z hashem commita (albo `packed-refs`/`reftable`).
  Utworzenie brancha = zapis 41 bajtów, nic się nie kopiuje.
- Obiekty nie należą do branchy, baza obiektów jest wspólna.
- `HEAD` normalnie: `ref: refs/heads/main` (wskazuje na branch).
- Detached HEAD: `HEAD` zawiera hash commita. Commity tu zrobione nie są
  śledzone przez żaden branch → przed wyjściem: `git switch -c <nazwa>`.
- `git commit`: buduje tree z indeksu → commit z parent = HEAD → przesuwa branch,
  na który wskazuje HEAD (HEAD się nie zmienia).
- `git switch`: przepina HEAD + ustawia index i working tree na snapshot celu
  (nadpisuje różniące się pliki, usuwa nieobecne w celu, nieśledzone zostawia).

## Trzy obszary
```
working tree ──git add──► index ──git commit──► HEAD
     └──── git diff ───────┘└──── git diff --staged ───┘
```
- Index = **pełna** płaska lista ścieżek → blobów (projekt następnego commita), nie lista zmian.
- `git add` od razu zapisuje blob do `.git/objects`.
- `git status` = dwa diffy: HEAD↔index (to be committed), index↔dysk (not staged).
- `git commit -a` dodaje tylko **śledzone** zmodyfikowane pliki, nowych nie.

## Adresowanie
- `main~1` pierwszy rodzic, `main~3` trzy pokolenia wstecz.
- `HEAD^{tree}` drzewo commita, `HEAD:src` drzewo/plik pod ścieżką.
- `git show <commit>:<ścieżka>` treść pliku w danej wersji bez przełączania.

## Komendy do zaglądania pod maskę
```bash
git cat-file -t <obj>          # typ obiektu
git cat-file -p <obj>          # treść obiektu
git ls-tree -r HEAD            # snapshot: wszystkie pliki w commicie
git ls-files --stage           # zawartość indeksu
git hash-object <plik>         # hash bez zapisu (-w zapisuje)
git rev-parse <wyrażenie>      # zamień nazwę/wyrażenie na pełny hash
git log --oneline --graph --all
```

## Ratunek: zgubiony branch
```bash
git branch <nazwa> <hash>               # odtwórz branch
git update-ref refs/heads/<nazwa> <hash> # niskopoziomowo (waliduje, pisze reflog)
git fsck --unreachable --no-reflogs     # znajdź commity bez żadnego refa
```
Nieosiągalne obiekty usuwa `git gc` po okresie ochronnym, więc nie zwlekaj.
Nie edytuj plików w `.git/refs` ręcznie (`echo`): brak walidacji i wpisu w reflogu.