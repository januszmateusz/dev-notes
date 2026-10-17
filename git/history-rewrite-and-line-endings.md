# Git: przepisywanie historii i końce linii

## Diagnoza: czy plik jest w historii?

```bash
git status --short              # ?? = nieśledzony, M = śledzony i zmieniony
git log --oneline -- <plik>     # commity, które dotykały pliku
git log origin/main --oneline -- <plik>   # czy jest już na serwerze
git show --stat <commit>        # co zawiera konkretny commit
```

`modified` zamiast `Untracked` w `git status` oznacza, że plik jest już w historii.

## Cofnięcie ostatnich commitów (przed pushem)

```bash
git reset <commit_przed_zmianami>   # mixed: gałąź i indeks cofnięte, pliki na dysku zostają
git reset --hard <commit>           # UWAGA: kasuje też zmiany na dysku
```

Mixed reset zachowuje niezacommitowane poprawki, więc jest bezpieczniejszym wyborem.

## Usunięcie pliku z root commita (przed pushem)

Root commit nie ma rodzica (`<root>~1` nie istnieje), więc reset nie wystarczy.

```bash
git restore <plik>                                   # czysty katalog roboczy
GIT_SEQUENCE_EDITOR="sed -i '1s/^pick/edit/'" git rebase -i --root
git rm <plik>
git commit --amend -m "<nowy komunikat>"
git rebase --continue
git log --oneline -- <plik>                          # pusto = sukces
# awaryjnie: git rebase --abort
```

`GIT_SEQUENCE_EDITOR` z `sed` zamienia `pick` na `edit` w pierwszej linii planu,
bez otwierania edytora.

## Wyciek sekretu

1. **Rotacja sekretu NATYCHMIAST**, jeśli commit był wypchnięty. Przepisanie historii
   nie cofa wycieku.
2. Dopiero potem przepisanie historii i `git push --force-with-lease`.
3. Przed pushem wystarczy przepisanie historii lokalnie.

## Niezacommitowane zmiany przy zmianie brancha

`git switch <branch>` przenosi niezacommitowane zmiany na nowy branch, jeśli nie ma
konfliktu. Na chroniony `main` nie commituj: `git switch -c <nowy-branch>` zabiera
zmiany ze sobą.

## Rozbicie zmian na kilka commitów

```bash
git add <konkretne ścieżki>     # nigdy `git add -A` w repo z .env
git commit -m "type(scope): opis"
```

pre-commit sprawdza tylko zawartość danego commita (resztę chowa), więc każdy commit
jest weryfikowany osobno. Jeśli hook poprawi plik: `git add` ponownie i powtórz commit.

## Końce linii

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf
```

```bash
git add --renormalize .         # przepisuje śledzone pliki według .gitattributes
git ls-files --eol              # i/ = w repo, w/ = na dysku
git ls-files --eol | grep "w/crlf"
```

- Ostrzeżenie `CRLF will be replaced by LF` przy `git add` jest nieszkodliwe przy
  `eol=lf`: do repo trafia LF, różni się tylko kopia na dysku.
- Atrybut `eol` w `.gitattributes` ma pierwszeństwo przed `core.autocrlf`.
- Zapobieganie po stronie edytora: `.editorconfig` z `end_of_line = lf`.
- Dodatkowa warstwa: hook `mixed-line-ending` z `--fix=lf` (z wyjątkiem `.bat`/`.ps1`).