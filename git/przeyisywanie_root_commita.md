## Usunięcie pliku z root commita (przed pushem)
git restore <plik>                                        # czysty katalog roboczy
GIT_SEQUENCE_EDITOR="sed -i '1s/^pick/edit/'" git rebase -i --root
git rm <plik>
git commit --amend -m "<nowy komunikat>"
git rebase --continue
git log --oneline -- <plik>                               # pusto = sukces
# awaryjnie: git rebase --abort
# Po pushu: najpierw rotacja sekretu, potem rewrite + force push