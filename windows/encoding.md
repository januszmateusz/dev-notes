# Windows: kodowanie plików

## Problem

Przekierowanie wyjścia na Windowsie potrafi zapisać plik w **UTF-16 LE**:

- `uv run python -c "print(...)" > plik` (zaobserwowane także w bashu MSYS2),
- `echo ... > plik` w Windows PowerShell 5.x.

W edytorze plik wygląda normalnie, ale narzędzia działające na bajtach
(gitleaks, grep, python-dotenv, parsery) widzą bajty zerowe między znakami
i nic nie dopasowują.

## Diagnoza

```bash
file <plik>              # "Unicode text, UTF-16, little-endian" vs "ASCII text"
```

```python
open(".env", "rb").read(4)
# b'\xff\xfe...'   UTF-16 LE
# b'\xef\xbb\xbf'  UTF-8 z BOM
# b'ANTH'          zwykły tekst (OK)
```

Podejrzana długość też jest wskazówką: `awk` liczy ok. 2× więcej znaków w UTF-16.

## Rozwiązanie

Gdy liczy się kodowanie, plik zapisuje kod, a nie przekierowanie:

```python
from pathlib import Path

Path("plik.txt").write_text(tekst, encoding="utf-8", newline="\n")
```

- `newline="\n"` wymusza LF niezależnie od systemu.
- W VS Code: kodowanie w prawym dolnym rogu → „Save with Encoding” → UTF-8.
- `.editorconfig` z `charset = utf-8` dla plików tworzonych w edytorze.

## Zasada diagnostyczna

Zanim podejrzewasz narzędzie, sprawdź dane wejściowe: kodowanie, długość,
białe znaki, końce linii.