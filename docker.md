# Docker — ściągawka komend + kluczowe koncepcje

## Fundamentalne rozróżnienie: obraz vs kontener

- **Obraz** — statyczny "przepis" + warstwy plików, niezmienny, wersjonowany (tag)
- **Kontener** — uruchomiona instancja obrazu, ma własny, tymczasowy "writable layer" na wierzchu

**Restart kontenera ≠ przebudowa obrazu.** Restart (Docker Desktop, `docker restart`) wznawia **ten sam, już zbudowany** obraz — żadna zmiana w `requirements.txt`/`Dockerfile`/`docker-compose.override.yml` się nie zastosuje. Trzeba **przebudować**:
```bash
astro dev stop
astro dev start   # to buduje obraz na nowo, nie tylko restartuje kontener
```

## Podstawowe komendy

| Komenda | Co robi |
|---|---|
| `docker ps` | Lista działających kontenerów |
| `docker ps -a` | Lista wszystkich kontenerów (też zatrzymanych) |
| `docker exec -it <container> <cmd>` | Wykonaj komendę wewnątrz działającego kontenera |
| `docker exec -it $(docker ps -qf "name=scheduler") id` | Sprawdź UID/GID procesu w kontenerze o nazwie zawierającej "scheduler" |
| `docker logs <container>` | Logi kontenera |
| `docker inspect <container>` | Pełna konfiguracja kontenera (mounty, sieć, env) — dobre do diagnozowania "dlaczego to nie widzi pliku" |
| `docker volume ls` / `docker volume inspect <name>` | Lista/szczegóły wolumenów |
| `docker system prune` | Czyści nieużywane zasoby (uwaga: nieodwracalne) |

## Astro CLI (wrapper na docker-compose dla Airflow)

| Komenda | Kiedy |
|---|---|
| `astro dev start` | Pierwsze uruchomienie / po zmianie `requirements.txt`, `Dockerfile`, `docker-compose.override.yml` |
| `astro dev stop` | Zatrzymanie — **zawsze przed** `start` po zmianie configu |
| `astro dev restart` | **Nie** wystarcza po zmianie `requirements.txt`/`.env` — tylko dla drobnych zmian w kodzie DAG-ów, nie w zależnościach |

## Uprawnienia plików — kontener działa jako inny użytkownik niż host

Astro Runtime (i większość obrazów Airflow) uruchamia procesy jako **UID 50000** (user `astro`), nie jako Twój użytkownik na hoście. Zamontowany katalog z hosta (np. `data/`) może mieć właściciela `root:root` — kontener nie ma prawa zapisu.

**Zła, ale szybka łatka:** `chmod -R 777` — otwiera zapis dla wszystkich, unikać na poważnym projekcie.

**Właściwe rozwiązanie:**
```bash
sudo chown -R 50000:50000 ścieżka/do/katalogu
sudo chmod -R 775 ścieżka/do/katalogu
```

UID 50000 jest **wypieczony w obrazie** (Dockerfile Astronomer/Apache Airflow) — nie zmienia się przy restarcie kontenera, przebudowie obrazu ani restarcie VM/hosta. Zmieniłby się tylko przy świadomej zmianie bazowego obrazu na inną dystrybucję.

## Wolumeny — trzy rodzaje

| Typ | Trwałość | Użycie |
|---|---|---|
| Named volume | Trwały, zarządzany przez Dockera | Dane, które mają przetrwać (np. Postgres) |
| Bind mount | Trwały, wskazuje na realną ścieżkę hosta | Live development — zmiana pliku na hoście widoczna od razu w kontenerze |
| Writable layer | Efemeryczny, znika z kontenerem | Wszystko, czego nie zamontowałeś jawnie |

## Nazwy usług systemowych różnią się między dystrybucjami Linuksa

- Ubuntu/Debian: `sudo systemctl restart ssh`
- RHEL/CentOS/Oracle Linux: `sudo systemctl restart sshd`

Binarka SSH zawsze nazywa się `sshd`, ale **jednostka systemd** różni się nazwą między dystrybucjami — stąd `sudo sshd -T | grep ...` (diagnostyka bezpośrednio przez binarkę) działa wszędzie tak samo, niezależnie od nazwy usługi.

## Auto-start kontenerów przy starcie VM (systemd)

```ini
# /etc/systemd/system/airflow-astro.service
[Unit]
Description=Astro Airflow Dev Environment
After=docker.service network-online.target
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=<user>
WorkingDirectory=/ścieżka/do/projektu/airflow
ExecStart=/usr/local/bin/astro dev start
ExecStop=/usr/local/bin/astro dev stop
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable airflow-astro.service
```

## Dockerfile — dobre praktyki z tego projektu

- **Multi-stage build** — realnie zmierzona redukcja rozmiaru obrazu (461MB → 386MB w ćwiczeniu)
- **Non-root user** — `USER` **po** instalacji zależności wymagających uprawnień roota, nie przed
- **`.dockerignore`** — analogicznie do `.gitignore`, wyklucza niepotrzebne pliki z kontekstu budowania

## Cache warstw

Kolejność instrukcji w `Dockerfile` ma znaczenie — zmiana w **wczesnej** warstwie unieważnia cache **wszystkich** warstw po niej. Kopiuj i instaluj zależności (rzadko się zmieniają) **przed** kopiowaniem kodu aplikacji (zmienia się często).

## Registry (GHCR)

```bash
docker tag lokalny-obraz:v1 ghcr.io/user/repo:v1
docker push ghcr.io/user/repo:v1
```
`docker tag` tworzy **etykietę**, nie kopię obrazu — ten sam obraz może mieć wiele tagów jednocześnie. `:latest` to zwykły tag, nie ma magicznego znaczenia — trzeba go jawnie nadać.