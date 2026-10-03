# Terraform — ściągawka

Notatki z nauki Terraform na realnej infrastrukturze projektu `airline-cloud-warehouse` (Azure + Databricks Free Edition).
Stan: sesje 1–3 (fundamenty, język, pilotaż Databricks).

Wersje użyte w notatkach: Terraform `1.16.x`, `hashicorp/azurerm` `5.x`, `databricks/databricks` `1.134+`.
Wersje zmieniają się szybko — zawsze sprawdzaj aktualną w [Terraform Registry](https://registry.terraform.io/), nie z pamięci.

---

## Cykl pracy

| Komenda | Co robi | Łączy się z chmurą? | Zapisuje stan? |
|---|---|---|---|
| `terraform init` | Pobiera dostawców z rejestru, tworzy `.terraform/` i `.terraform.lock.hcl` | ❌ (tylko rejestr) | ❌ |
| `terraform init -upgrade` | Jak wyżej, ale ignoruje lock i wybiera najnowszą wersję pasującą do ograniczeń | ❌ | ❌ |
| `terraform validate` | Sprawdza składnię i spójność odwołań lokalnie | ❌ | ❌ |
| `terraform fmt` | Formatuje kod | ❌ | ❌ |
| `terraform plan` | Odświeża zasoby ze stanu (refresh), porównuje z kodem, pokazuje zmiany | ✅ | ❌ **nigdy** |
| `terraform plan -destroy` | Pokazuje, co usunąłby `destroy` — bez wykonania | ✅ | ❌ |
| `terraform plan -out=tfplan` | Zapisuje plan do pliku | ✅ | ❌ |
| `terraform apply` | Ponownie wylicza plan, pyta o zgodę, wykonuje | ✅ | ✅ |
| `terraform apply tfplan` | Wykonuje **dokładnie** zapisany plan, bez pytania | ✅ | ✅ |
| `terraform destroy` | Usuwa wszystkie zasoby zarządzane | ✅ | ✅ |
| `terraform providers` | Drzewo wymaganych dostawców i skąd pochodzą | ❌ | ❌ |

Zasady:
- `apply` / `destroy` akceptują **tylko dokładne `yes`** — `y`, `tak` itp. anulują operację.
- `apply` **to nie transakcja** — brak rollbacku, postęp zapisywany po każdym zasobie. Po błędzie część zasobów już istnieje.
- `plan -out` + `apply <plik>` = gwarancja, że wykonasz to, co przejrzałeś. Podstawa w CI.
- `-target=<adres>` ogranicza operację do jednego zasobu i jego zależności. Narzędzie awaryjne, nie codzienne.

---

## Czytanie planu

| Symbol | Znaczenie |
|---|---|
| `+` | create |
| `~` | update in-place |
| `-` | destroy |
| `-/+` | destroy and then create replacement — **nowy obiekt, nowe ID** |
| `<=` | odczyt data source |

Czerwone flagi — **stop i czytaj plan od góry**:
- `destroy > 0` w podsumowaniu.
- `# forces replacement` przy atrybucie — wskazuje winowajcę zamiany.
- `# Warning: this will destroy the imported resource` — import, po którym zasób zostanie od razu zniszczony.
- Zamiana liczy się w podsumowaniu jako **add + destroy**, nie jako change. `0 to change` nie znaczy „nic się nie stanie".

Inne:
- `(known after apply)` — wartość nadawana przez chmurę (np. `id`).
- `(sensitive value)` — wartość zamaskowana w konsoli (ale **nie** w stanie).
- `(write-only attribute)` — argument write-only, nie trafia do stanu ani planu.
- `# (N unchanged attributes hidden)` — atrybuty bez zmian ukryte dla czytelności.
- O tym, czy zmiana jest `~` czy `-/+`, decyduje **schemat dostawcy**, nie logika ani Terraform.

**Nawyk:** czytaj podsumowanie planu przy **każdym** `apply`, nawet gdy „wiesz", co zrobi.
Lekcja z pilotażu: `apply` miał tylko zmienić jedną flagę, a odtworzył 7 wcześniej usuniętych zasobów, bo nadal były w kodzie.

---

## Wersje i plik lock

```hcl
terraform {
  required_version = "~> 1.16"          # wersja Terraform CLI

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"                # wersja DOSTAWCY
    }
    databricks = {
      source  = "databricks/databricks" # NIE hashicorp/ — dostawca publikowany przez Databricks
      version = "~> 1.134"
    }
  }
}
```

- `~> 1.16` = `>= 1.16, < 2.0`. `~> 5.0` = `>= 5.0, < 6.0`.
- Ograniczenie w złym bloku daje złą wersję **bez ostrzeżenia**. Przykład: `~> 1.16` w bloku `azurerm` → zainstalował azurerm `1.44.0` sprzed lat. Po `init` zawsze sprawdź, jaką wersję faktycznie wybrał.
- `.terraform.lock.hcl` zapisuje wybraną wersję + hashe → **commitujemy do repo** (jak `uv.lock` / `poetry.lock`).
- `.terraform/` (binarki dostawców) → **nie commitujemy**.
- Lock ma pierwszeństwo przed ograniczeniami w kodzie. Zmiana ograniczenia bez `init -upgrade` → błąd `locked provider ... does not match configured version constraint`. To celowe — aktualizacja dostawcy to świadoma decyzja z osobnym commitem.
- Dostawca to osobny program (`.exe` w `.terraform/providers/...`). Terraform sam nie zna Azure ani Databricks.
- Brak wpisu w `required_providers` → Terraform zgaduje `hashicorp/<nazwa>` → błąd `registry does not have a provider named registry.terraform.io/hashicorp/databricks`.

---

## Uwierzytelnienie

**Azure (`azurerm`):**
- Korzysta z tokena zapisanego przez `az login`, nie z otwartego terminala.
- `azurerm` 4.x+ wymaga jawnej subskrypcji. Najwygodniej jako zmienna użytkownika Windows (jednorazowo):
  ```powershell
  [Environment]::SetEnvironmentVariable("ARM_SUBSCRIPTION_ID", (az account show --query id -o tsv), "User")
  ```
  Po tym zamknij i otwórz ponownie VS Code (terminal dziedziczy środowisko procesu VS Code).
- ID subskrypcji nie jest sekretem, ale nie wpisuj go w kod — w CI użyjesz tych samych zmiennych `ARM_*`.

**Databricks:**
```hcl
provider "databricks" {
  profile = var.databricks_profile   # nazwa profilu z ~/.databrickscfg
}
```
- Dostawca czyta `%USERPROFILE%\.databrickscfg` (ten sam plik co Databricks CLI i Python SDK — *unified auth*). W kodzie jest tylko **nazwa** profilu, nigdy token.
- PAT: token w pliku. OAuth: token z cache po `databricks auth login`. Błąd uwierzytelnienia przy OAuth → ponowny `databricks auth login`, nie zmiana kodu.
- Zmienne środowiskowe `DATABRICKS_HOST` / `DATABRICKS_TOKEN` mają **pierwszeństwo** przed profilem — mogą namieszać.
- Diagnoza: `databricks auth profiles`, `databricks auth describe --profile <nazwa>`.

**Zmienne z sekretami:** `$env:TF_VAR_<nazwa> = "..."` → Terraform mapuje na `var.<nazwa>`. Zmienna bez `default` i bez wartości → Terraform zapyta interaktywnie (także przy `destroy`).

---

## Stan (state)

- **Terraform nie szuka zasobów w chmurze po nazwie.** Zna tylko to, co ma w stanie. Istniejąca infrastruktura jest dla niego niewidzialna, dopóki jej nie zaimportujesz.
- Zasób w kodzie, ale nie w stanie → plan pokaże `+ create`, nawet jeśli obiekt o tej nazwie istnieje (wtedy `apply` wyłoży się na „already exists").
- Stan łączy **adres w kodzie** (`azurerm_resource_group.sandbox`) z **ID w chmurze** (`/subscriptions/.../resourceGroups/...`).
- `refresh` (część `plan`) odpytuje chmurę tylko o zasoby, które są w stanie → wykrywa **drift** (zmiany zrobione poza Terraformem).

Pola w pliku stanu:

| Pole | Znaczenie |
|---|---|
| `lineage` | UUID nadany przy pierwszym utworzeniu stanu — „**który** to stan" |
| `serial` | Licznik zwiększany przy każdym zapisie — „**która** to wersja" |
| `terraform_version` | Wersja, która **ostatnio zapisała** stan (starszy CLI może odmówić pracy) |
| `mode` | `managed` (zarządzany) vs `data` (tylko odczyt) |
| `dependencies` | Zależności między zasobami — kolejność przy `destroy` nawet bez kodu |

- `terraform.tfstate.backup` = **jedna** poprzednia wersja, nadpisywana przy każdym zapisie.
- Po `destroy` plik stanu zostaje — z `resources: []`, tym samym `lineage`, wyższym `serial`.
- Stan i backup **nigdy do gita**.

```powershell
terraform state list
terraform state show azurerm_resource_group.sandbox
terraform state show 'azurerm_subnet.sandbox[\"snet-ingest\"]'   # PowerShell: escape cudzysłowów w adresie
```

**Drift — dwie drogi rozwiązania (świadoma decyzja, nie automat):**
- cofnąć zmianę → `apply` (kod wygrywa),
- zaakceptować zmianę → dopisać ją do kodu (rzeczywistość wygrywa).

---

## Język

### Pliki
Terraform wczytuje **wszystkie** `*.tf` z bieżącego katalogu jako jedną całość. Kolejność plików i bloków nie ma znaczenia — kolejność wykonania wynika z grafu zależności (jak optymalizator w SQL). Podkatalogi nie są wczytywane (to osobne moduły).

| Plik | Pytanie, na które odpowiada |
|---|---|
| `versions.tf` | Czym i z czym to uruchamiać? (`terraform {}`, `provider {}`) |
| `variables.tf` | Co można zmienić z zewnątrz? |
| `locals.tf` | Co wyliczam wewnętrznie? (nie da się nadpisać z zewnątrz) |
| `main.tf` | Co powstaje i co odczytuję? (`resource`, `data`) |
| `outputs.tf` | Co udostępniam na zewnątrz? |

Analogia: moduł = funkcja. `variables` = parametry, `locals` = zmienne lokalne, `main` = ciało, `outputs` = wartość zwracana. `variables.tf` + `outputs.tf` = interfejs modułu.

Każdy katalog = osobny root module = **osobny stan**. Osobne katalogi dla rzeczy o różnym cyklu życia / uwierzytelnieniu (np. Azure vs Databricks) → mniejszy blast radius.

### Odwołania
```hcl
var.location                                   # zmienna
local.common_tags                              # local (liczba pojedyncza!)
azurerm_resource_group.sandbox.name            # zasób zarządzany
data.azurerm_resource_group.airline.location   # data source
each.key / each.value                          # w bloku z for_each
path.module                                    # katalog z kodem
```

### `resource` vs `data`

| | `resource` | `data` |
|---|---|---|
| Własność | Mój obiekt, pełny cykl życia | Cudzy obiekt, tylko odczyt |
| W stanie | `mode: managed` | `mode: data` |
| W planie | `+` / `~` / `-` / `-/+` | tylko odczyt, nigdy akcje |
| `destroy` | usuwa obiekt | tylko „zapomina" odczyt |
| Gdy obiekt nie istnieje | tworzy go | błąd planu |

- Data source to zapytanie z zapamiętanym wynikiem, czytane w czasie `plan` (chyba że zależy od wartości znanych dopiero po `apply`).
- Liczenie zasobów do usunięcia: **tylko bez prefiksu `data.`**.
- Zastosowania: infrastruktura innego zespołu, `azurerm_client_config`, obiekty tworzone ręcznie (np. katalog UC na Free Edition).

### Zależności
- **Niejawne:** odwołanie do atrybutu = wartość **i** zależność. Wystarczy jedno odwołanie — zależność jest na poziomie zasobu, nie atrybutu.
- **Wartość wpisana na sztywno** zamiast odwołania = brak zależności → zasoby tworzone równolegle → na pustym środowisku np. `ResourceGroupNotFound`. Kod może działać miesiącami „przypadkiem" i wyłożyć się przy budowie od zera.
- **`depends_on`** tylko dla zależności ukrytych (nie wynikających z żadnego odwołania), np. rola dla Managed Identity przed runbookiem.
- Tworzenie: od korzeni grafu, niezależne zasoby równolegle (domyślnie 10). Usuwanie: w odwrotnej kolejności.

### `for_each` vs `count`
```hcl
variable "subnets" {
  type = map(string)
  default = {
    snet-ingest  = "10.10.1.0/24"
    snet-compute = "10.10.2.0/24"
  }
}

resource "azurerm_subnet" "sandbox" {
  for_each             = var.subnets
  name                 = each.key
  resource_group_name  = azurerm_resource_group.sandbox.name
  virtual_network_name = azurerm_virtual_network.sandbox.name
  address_prefixes     = [each.value]
}

output "subnet_ids" {
  value = { for k, v in azurerm_subnet.sandbox : k => v.id }
}
```
- `for_each` → adresy z kluczem: `azurerm_subnet.sandbox["snet-ingest"]`. Klucz = tożsamość. Zmiana klucza = destroy + create.
- `count` → adresy z indeksem: `[0]`, `[1]`. Usunięcie elementu z listy **przesuwa indeksy** → zamiana/usunięcie zasobów, których nikt nie ruszał.
- Reguła: `for_each` dla kolekcji z tożsamością; `count` głównie jako przełącznik `count = var.enabled ? 1 : 0`.

---

## Sekrety

- `sensitive = true` = **maskowanie wyświetlania**, nie szyfrowanie. Wartość trafia do stanu **otwartym tekstem**.
- Stan trzeba chronić jak magazyn sekretów: zdalny backend, szyfrowanie, RBAC.
- Sekret, który **kiedykolwiek** był w stanie, jest ujawniony (backup, wersje backendu, artefakty CI, kopie na innych maszynach) → **rotacja**. Usunięcie ze stanu nie cofa ekspozycji.
- **Write-only arguments** (Terraform 1.11+): nie trafiają ani do stanu, ani do planu.
  ```hcl
  resource "databricks_secret" "example" {
    scope                   = databricks_secret_scope.example.name
    key                     = "api-key"
    string_value_wo         = var.api_key      # zmienna z sensitive = true
    string_value_wo_version = 1                # podbij, żeby wysłać nową wartość (rotacja)
  }
  ```
  Terraform nie przechowuje wartości `_wo`, więc nie wykryje jej zmiany — jedynym sygnałem jest numer wersji.
- Prawdziwe sekrety: **od pierwszego dnia `_wo`**, nie „najpierw zwykły argument, potem poprawimy".
- Plan zapisany przez `plan -out` zawiera wartości zmiennych → zmienne z sekretami w CI oznaczać dodatkowo `ephemeral = true`.
- Outputy trafiają do logów CI i komentarzy w PR → rozważ `sensitive = true` dla outputów (np. e-mail).

---

## `lifecycle`

```hcl
lifecycle {
  prevent_destroy = true            # każdy plan niszczący zasób kończy się błędem
  ignore_changes  = [storage_root]  # różnice na tym atrybucie są ignorowane
}
```

| | `prevent_destroy` | `ignore_changes` |
|---|---|---|
| Chroni przed | zniszczeniem z **dowolnej** przyczyny | zmianą/zamianą wywołaną **tym jednym** atrybutem |
| Kiedy | krytyczne zasoby (VM z danymi, katalogi produkcyjne) | atrybuty świadomie oddane platformie |

- Nie łącz wpisanej wartości z `ignore_changes` na tym samym atrybucie („hybryda") — kod coś deklaruje, a nikt tego nie weryfikuje. Albo atrybut w kodzie (świadoma decyzja), albo poza kodem + `ignore_changes` + komentarz dlaczego.
- `force_destroy` to flaga **po stronie dostawcy**, czytana ze **stanu** przy `destroy`. Dodanie jej w kodzie wymaga najpierw `apply`, dopiero potem `destroy`. Sam `plan` nic nie zmieni.

---

## Import istniejących zasobów

Import = wpisanie istniejącego obiektu do stanu. **Niczego nie zmienia w chmurze** — cel planu: `1 to import, 0 to add, 0 to change, 0 to destroy`.

### Trzy fazy

**Faza 1 — generowanie** (bloku `resource` jeszcze **nie ma**):
```hcl
import {
  to       = databricks_catalog.pilot
  id       = "tf_pilot"
  provider = databricks   # tylko w fazie generowania; lokalna nazwa z required_providers, bez cudzysłowów
}
```
```powershell
terraform plan -generate-config-out="generated.tf"
```

**Faza 2 — przegląd i czyszczenie** wygenerowanego kodu (to zrzut wszystkiego, co dostawca odczytał, nie docelowa konfiguracja):
- usuń `null`-e,
- usuń bloki wyliczane (`effective_*`, znaczniki czasu, `created_by` itp.),
- zastanów się nad wartościami na sztywno (`owner`, ID metastore'u, `provider_config`),
- **nie usuwaj atrybutów z realną wartością, których zmiana wymusza zamianę** (np. `storage_root`) — albo zostaw, albo `ignore_changes`.

**Faza 3 — przełączenie:** przenieś blok do `main.tf`, usuń `provider` z bloku `import`, przepnij odwołania, `plan` → `apply` → usuń blok `import` (ślad zostaje w historii gita) → `plan` = `No changes`.

### Pułapki (z praktyki)
- `failed to instantiate provider "registry.terraform.io/hashicorp/..."` przy generowaniu → dodaj `provider = <lokalna_nazwa>` w bloku `import`.
- `Reference to undeclared resource` przy generowaniu → inne zasoby odwołują się do bloku, który dopiero ma powstać (jajko i kura). Na czas generowania trzymaj stary punkt zaczepienia (np. data source).
- `Invalid import provider argument` (opakowane w mylący komunikat o lock file) → blok `resource` już istnieje, usuń `provider` z bloku `import`. **Nie** uruchamiaj `init -upgrade`, choć komunikat to sugeruje.
- Usunięcie z kodu atrybutu, który dostawca uważa za niezmienny → `-/+` + `Warning: this will destroy the imported resource`. **Nie rób `apply`.**
- Import zmienia status obiektu z „cudzy, do odczytu" na „mój, pełny cykl życia" — od tej chwili `destroy` go usunie.

---

## Uprawnienia — autorytatywność

Zanim użyjesz zasobu do uprawnień, sprawdź, **jaki jest jego zakres**:

| Zasób (Databricks) | Autorytatywny dla |
|---|---|
| `databricks_grants` | **wszystkich** uprawnień **wszystkich** podmiotów na obiekcie — usunie ręcznie nadane uprawnienia innych zespołów |
| `databricks_grant` | wszystkich uprawnień **jednego podmiotu** na obiekcie — ręcznie dodane uprawnienie temu podmiotowi zostanie usunięte przy `apply` |

- Zasada „czego nie ma w stanie, tego Terraform nie rusza" jest prawdziwa, ale **zakres zasobu** może obejmować więcej niż wpisy w kodzie.
- Jeden podmiot na jednym obiekcie = jedno źródło prawdy (Terraform **albo** UI).
- Ten sam wzorzec w Azure: reguły NSG inline (autorytatywnie) vs osobne `azurerm_network_security_rule`.
- Nazwy w UI i API bywają różne: grupa `account users` (API) = **All account users** (UI).

---

## Databricks Free Edition — wyniki pilotażu

| Obiekt | Wynik | Uwagi |
|---|---|---|
| Data sources (`current_user`, `catalogs`) | ✅ | `samples` (Delta Sharing) i `system` też widoczne — filtruj przy `for_each` |
| `databricks_catalog` — tworzenie | ❌ | brak storage root metastore'u; Default Storage tylko przez UI |
| `databricks_catalog` — import | ✅ | `storage_root` w `ignore_changes`, inaczej wymuszona zamiana |
| `databricks_catalog` — destroy | ⚠️ | wymaga `force_destroy` — automatyczny schemat `default` blokuje usunięcie |
| `databricks_schema` | ✅ | storage dziedziczony z katalogu |
| `databricks_volume` (MANAGED) | ✅ | |
| `databricks_grant` | ✅ | autorytatywny dla jednego podmiotu |
| `databricks_secret_scope` | ✅ | |
| `databricks_secret` (`string_value`) | ⚠️ | wartość otwartym tekstem w stanie |
| `databricks_secret` (`string_value_wo`) | ✅ | wartość poza stanem |
| `databricks_notebook` | ✅ | `source = "${path.module}/notebooks/x.py"` |
| `databricks_job` (serverless) | ✅ | brak definicji klastra = serverless; harmonogram `PAUSED` działa |

- Free Edition działa na **AWS (us-east-2)**, nie na Azure.
- Przebiegi jobów (runs) nie są zasobami Terraforma — zarządza tylko definicją joba.
- Formaty ID przydatne przy imporcie: sekret `<scope>|||<key>`, grant `schema/<catalog>.<schema>/<principal>`, schemat `<catalog>.<schema>`, volume `<catalog>.<schema>.<volume>`.

---

## Diagnoza typowych błędów

| Objaw | Przyczyna | Rozwiązanie |
|---|---|---|
| `Reference to undeclared resource` | Plik niezapisany w VS Code / literówka / zły katalog | `Select-String -Path *.tf -Pattern '...'`, `terraform validate` |
| `Finding latest version of hashicorp/<x>` | Terraform nie widzi `required_providers` | Sprawdź, czy `versions.tf` zapisany i poprawnie nazwany |
| `locked provider ... does not match configured version constraint` | Lock ma starą wersję | `terraform init -upgrade` (świadomie) |
| `ResourceGroupNotFound` przy tworzeniu | Wartość na sztywno zamiast odwołania → brak zależności → równoległe tworzenie | Odwołanie do atrybutu zasobu |
| `apply` / `destroy` nic nie zrobił | Odpowiedź inna niż dokładne `yes` | Powtórz, wpisz `yes` |
| `cannot delete catalog: ... is not empty` | `force_destroy = false` w stanie | Dodaj `force_destroy = true` → `apply` → `destroy` |

Ogólna zasada: **czytaj błąd do końca, nie tylko nagłówek**, i nie wykonuj automatycznie sugerowanej komendy.

---

## `.gitignore`

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
!*.tfvars.example
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc
```

Commitujemy: `*.tf`, `.terraform.lock.hcl`, `*.tfvars.example`.
Szybki test: `git status` po `init` / `apply` — nie może pokazywać `.terraform/` ani `*.tfstate*`.
`*_override.tf` są doklejane na końcu i nadpisują definicje — lokalne obejścia, nie do repo.