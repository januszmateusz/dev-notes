# Kafka — ściągawka komend + kluczowe koncepcje i lekcje

Zbudowane w trzech fazach projektu airline-cloud-warehouse:
1. **Fundamenty** — lokalny broker, CLI
2. **Producent i konsument lokalnie** — Python (confluent-kafka) + Spark Structured Streaming w labie Dockera
3. **Chmura** — zarządzana Kafka w Aiven, konsument w Databricks serverless, Joby z repozytorium Git

---

## 1. Lokalny broker (Docker, KRaft)

`kafka/docker-compose.yml` — pojedynczy węzeł w trybie KRaft (broker + kontroler, bez ZooKeepera), dostępny jednocześnie z Windowsa i z kontenerów labu Spark:

```yaml
services:
  kafka:
    image: apache/kafka:<przypięta_wersja>
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: EXTERNAL://0.0.0.0:9092,INTERNAL://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: EXTERNAL://localhost:9092,INTERNAL://kafka:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: EXTERNAL:PLAINTEXT,INTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    networks:
      - default
      - spark-lab

networks:
  spark-lab:
    name: environment_default
    external: true
```

| Listener | Adres ogłaszany | Kto korzysta |
|---|---|---|
| `EXTERNAL` | `localhost:9092` | Producent z Windowsa (przez mapowanie portu) |
| `INTERNAL` | `kafka:29092` | Kontenery w sieci Dockera (Spark) — port **nie** jest w `ports`, bo ruch nie idzie przez hosta |
| `CONTROLLER` | `kafka:9093` | Wewnętrzna komunikacja KRaft |

Pułapki:
- **`localhost` w kontenerze = ten kontener.** Konsument w kontenerze Spark łączący się z `localhost:9092` trafia w siebie
- **Połączenie jest dwuetapowe.** Klient łączy się z adresem startowym, a broker odsyła mu `advertised.listeners`. Przy złym `ADVERTISED_LISTENERS` pierwszy kontakt się udaje, a drugi trafia w złe miejsce — wyjątkowo mylący błąd
- **`getent hosts kafka` sprawdza tylko DNS**, nie to, czy listener działa
- **Nadpisanie konfiguracji zmiennymi = trzeba podać komplet.** Domyślny replication factor 3 dla `__consumer_offsets` jest niemożliwy przy jednym brokerze
- **`external: true`** — sieć labu Spark musi istnieć przed startem Kafki
- **Bez wolumenu `--force-recreate` kasuje topiki i offsety grup** — żyją w warstwie zapisywalnej kontenera

| Komenda | Co robi |
|---|---|
| `docker compose up -d` | Start brokera (z katalogu `kafka/`) |
| `docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --version` | Wersja Kafki (do przypięcia tagu obrazu) |

Narzędzia CLI w kontenerze: `/opt/kafka/bin/`.

---

## 2. Topiki

```powershell
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic aircraft-telemetry --partitions 3 --replication-factor 1
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic aircraft-telemetry
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic aircraft-telemetry

# liczba wiadomości per partycja (LOG-END-OFFSET)
docker exec -it kafka /opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry
```

**Pułapka — automatyczne tworzenie topików.** Domyślnie włączone: producent wysyłający do nieistniejącego topiku po cichu go tworzy z **jedną** partycją, a późniejsze `--create` kończy się błędem "already exists". Objawy: zero równoległości, kolejność "za darmo" (test kolejności niczego nie dowodzi), literówka w nazwie topiku tworzy nowy topik. W produkcji: `auto.create.topics.enable=false`. Rozpoznanie: offset w partycji 0 równy łącznej liczbie wiadomości − 1.

---

## 3. Producent i konsument z konsoli

```powershell
# producent — interaktywny: najpierw komenda, potem wiadomości w prompcie ">"
docker exec -it kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --reader-property parse.key=true --reader-property key.separator=:

# konsument bez grupy — czyta od początku, nie zapisuje postępu
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --from-beginning --formatter-property print.key=true --formatter-property print.partition=true

# konsument w grupie — broker pamięta postęp grupy
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --group telemetry-readers --from-beginning --formatter-property print.key=true --formatter-property print.partition=true

# jedna partycja — bez grupy, bez offsetów, bez failover (debugowanie)
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --partition 1 --from-beginning --formatter-property print.key=true --formatter-property print.partition=true

# zamknięcie po 5 s bez nowych wiadomości + filtr
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --from-beginning --formatter-property print.key=true --formatter-property print.partition=true --timeout-ms 5000 | Select-String "PH-BVA"
```

- **Pułapka PowerShell:** wiadomość wklejona w tej samej linii co komenda — PowerShell parsuje `{...}` jako blok skryptu (`Unexpected token ':'`)
- `--property` jest przestarzałe: `--reader-property` (producent), `--formatter-property` (konsument)
- `key.separator` dzieli linię tylko na **pierwszym** separatorze — dwukropki wewnątrz JSON-a zostają w wartości
- Konsument bez `--group` sam tworzy grupę `console-consumer-<liczba>`

---

## 4. Grupy konsumentów

```powershell
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group telemetry-readers
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group telemetry-readers --members
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# cofnięcie offsetów na początek (grupa musi być nieaktywna)
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group telemetry-readers --topic aircraft-telemetry --reset-offsets --to-earliest --execute
```

`--describe` listuje partycje, więc konsument bez przydziału się w nim nie pojawi — do tego służy `--members`.

---

## 5. Kluczowe koncepcje (faza 1)

### Partycjonowanie po kluczu
- Partycja = `hash(klucz) mod liczba_partycji` — ten sam klucz zawsze w tej samej partycji
- W jedną stronę: jeden klucz → jedna partycja; jedna partycja → wiele kluczy
- Klucz wybiera się według tego, **w obrębie czego potrzebna jest kolejność** (telemetria: samolot)
- Gorący klucz = przeciążona partycja = skew (ten sam problem co w Sparku)
- Zmiana liczby partycji zmienia mapowanie kluczy i łamie kolejność — liczbę partycji planuje się z góry

### Kolejność
Gwarantowana **tylko w obrębie partycji**. Między partycjami kolejność wysłania przepada.

### Offsety i lag
- Offset liczony **osobno dla każdej partycji**, od 0
- `CURRENT-OFFSET` = numer **następnej** wiadomości do przeczytania przez grupę
- `LAG` = `LOG-END-OFFSET − CURRENT-OFFSET` — rosnący lag = konsument nie nadąża
- Commitowane offsety żyją w topiku `__consumer_offsets` — grupa pamięta postęp bez aktywnych konsumentów
- **Dziury w offsetach zawsze warto wyjaśnić**, zanim uzna się je za nieistotne (w projekcie: lokalny test producenta wysłał po jednej wiadomości do każdej partycji)

### `--from-beginning`
Działa tylko dla grupy **bez** zapisanego offsetu (`auto.offset.reset=earliest`). Grupa z historią zawsze wznawia od zapisanego miejsca.

### Grupy
- Partycje przydziela **broker** (group coordinator) — komendy różnią się tylko nazwą grupy
- Każda partycja ma dokładnie jednego właściciela w grupie; nadmiarowi konsumenci są gorącym zapasem
- Nierówny podział jest nieunikniony (3 partycje / 2 konsumentów = 2 + 1)
- **Rebalance** przy dołączeniu lub odejściu konsumenta; nowy właściciel kontynuuje od zapisanego offsetu
- **Grupy są niezależne** — nowa grupa czyta wszystko od początku. Kafka nie usuwa wiadomości po przeczytaniu (retencja), w przeciwieństwie do klasycznych kolejek JMS/MQ

### Gwarancje i kontrakt danych
- Domyślnie **at-least-once** — konsument musi znieść duplikaty
- Broker **nie sprawdza zawartości** — wartość to dla niego bajty. Kontrakt: po stronie konsumenta (walidacja, kwarantanna) albo przez Schema Registry

---

## 6. Producent w Pythonie (confluent-kafka)

| Mechanizm | Znaczenie |
|---|---|
| `produce()` | **Asynchroniczne** — wkłada do bufora i wraca natychmiast |
| `callback=delivery_report` + `poll(0)` | Potwierdzenie z partycją i offsetem; `poll()` oddaje zaległe callbacki |
| `flush(timeout)` | Czeka na opróżnienie bufora; zwraca liczbę **niedostarczonych** wiadomości. Bez niego wiadomości mogą po cichu przepaść |
| `acks=all` + `enable.idempotence=True` | Zapis potwierdzony dopiero po bezpiecznym zapisie; ponowienie po błędzie sieci nie tworzy duplikatu |

**Pułapka — różne partycjonery w Pythonie i Javie.** librdkafka domyślnie używa `consistent_random` (CRC32), a klienci Javy (w tym narzędzia konsolowe) — **murmur2**. Ten sam klucz trafiał do różnych partycji zależnie od klienta (`PH-BVA`: 0 z konsoli, 1 z Pythona), bez żadnego ostrzeżenia. Naprawa:

```python
"partitioner": "murmur2_random"
```

**Fail-fast i kod wyjścia:**
- Przy nieistniejącym topiku librdkafka ponawia do `message.timeout.ms` (domyślnie 5 min) — callback z błędem może się nigdy nie wykonać, a wiadomości przepadają przy zamykaniu
- Sprawdzenie topiku na starcie: `producer.list_topics(TOPIC, timeout=10)` → `metadata.topics[TOPIC].error`
- Po `flush` **rzucaj wyjątek**, a nie tylko loguj — inaczej skrypt kończy się kodem 0 i orkiestrator uzna porażkę za sukces

`Failed to acquire idempotence PID ... Coordinator load in progress: retrying` — nieszkodliwe; broker świeżo wystartował, biblioteka sama ponawia.

**Jeden moduł, trzy środowiska — wybór wyłącznie konfiguracją:**
- lokalny broker PLAINTEXT (brak zmiennych)
- Aiven z plikami certyfikatów (`KAFKA_SECURITY_PROTOCOL=SSL` + ścieżki w `.env`)
- Aiven z Databricks Secret Scopes (`--secret-scope`), certyfikaty jako treść: `ssl.ca.pem`, `ssl.certificate.pem`, `ssl.key.pem`; import `dbutils` wewnątrz funkcji, żeby nie psuć uruchomień lokalnych

---

## 7. Spark Structured Streaming + Kafka

```powershell
docker exec -it spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.3 --conf spark.jars.ivy=/tmp/.ivy2 /tmp/consume_aircraft_telemetry.py
```

- **Konektor to JAR, nie pakiet pip.** Spark działa na JVM; `--packages` pobiera bibliotekę z **Maven Central** (współrzędne `grupa:artefakt:wersja`) razem z zależnościami i rozsyła do executorów. Sufiks `_2.12` = wersja Scali, `3.5.3` = dokładnie wersja Sparka
- `spark.jars.ivy=/tmp/.ivy2` — domyślny cache pakietów bywa w kontenerze niezapisywalny
- W produkcji JAR-y wbudowuje się w obraz; na Databricks konektor jest już zainstalowany
- `key` i `value` przychodzą jako **bajty** — `.cast("string")`
- `topic`, `partition`, `offset`, `timestamp` dostępne jako zwykłe kolumny
- `(topic, partition, offset)` jednoznacznie identyfikuje wiadomość — **klucz deduplikacji** (nie treść)
- Bez `checkpointLocation` Spark tworzy **tymczasowy** checkpoint i kasuje go po zakończeniu — każde uruchomienie czyta wszystko od nowa
- **Spark nie widać w `--list` grup.** Executory używają ręcznego przypisania partycji (jak `--partition`), nie commitują offsetów, a driver pyta o offsety przez AdminClient. Postęp żyje wyłącznie w checkpoincie — narzędzia do lagu grup nie pomogą w monitorowaniu
- AQE jest wyłączane w streamingu
- `query.recentProgress` → suma `numInputRows` = liczba przeczytanych wiadomości
- `awaitTermination()` — w porządku w samodzielnym skrypcie `spark-submit`; w Databricks Jobs: `processAllAvailable()`
- **Klaster = wspólna przestrzeń dyskowa.** Ścieżka zapisu musi być widoczna pod tym samym adresem z drivera i wszystkich executorów (w labie: montowanie `/opt/spark/work-dir`)

---

## 8. Dwie pamięci: checkpoint + ujście

| Pamięć | Gdzie | Co zapamiętuje |
|---|---|---|
| Checkpoint | `checkpoints/...` | Które offsety Kafki przeczytano, pod jakim numerem batcha |
| Dziennik ujścia | Parquet: `_spark_metadata` w katalogu danych; Delta: log transakcji | Które batche zapisano |

Usunięcie **samego** checkpointu rozsynchronizowuje je — numeracja batchy startuje od 0:

| | Parquet (faza 2) | Delta (faza 3) |
|---|---|---|
| Skutek | **Cicha utrata danych** — ujście uznaje nowe batche za już zapisane i je pomija | **Duplikaty** — nowy checkpoint = nowy identyfikator zapytania, Delta nie rozpoznaje powtórki |
| Widoczność | Żadna na poziomie WARN; `Skipping already committed batch` tylko na INFO. Nawet `numInputRows` = 0, bo pominięty batch nie wykonuje odczytu | Widoczne, jeśli porównasz liczbę wierszy z liczbą unikalnych offsetów |
| Naprawa | Pełna przebudowa (usunąć oba) w oknie retencji Kafki | `RESTORE TABLE ... TO VERSION AS OF n` — **tylko** gdy wersja docelowa zawiera dokładnie to, co zapamiętał checkpoint |

- **`RESTORE` nie cofa checkpointu** — checkpoint to osobny folder, a nie metadana tabeli
- Zasady:
  - checkpoint i dane resetuje się **razem albo wcale**
  - nigdy nie usuwaj samego checkpointu, żeby przetworzyć dane ponownie
  - naprawa możliwa tylko w oknie retencji Kafki (Aiven free: 3 dni)
  - audyt: suma offsetów z `kafka-get-offsets.sh` vs `count(DISTINCT partition, offset)` w Bronze

```sql
SELECT partition, COUNT(*) AS messages, COUNT(DISTINCT offset) AS unique_offsets, MIN(offset), MAX(offset)
FROM bronze.aircraft_telemetry_raw
GROUP BY partition;
```

Brak dziur i duplikatów: `messages = unique_offsets = MAX(offset) + 1`.

---

## 9. Silver i kwarantanna

| Zły rekord | Poprawny JSON? | Sygnał | Powód |
|---|---|---|---|
| `to nie jest json` | Nie | treść w `_corrupt_record` | `malformed_json` |
| `{"txt": 3222}` | Tak | pola oczekiwane = `NULL` | `schema_mismatch` |
| klucz `SP-LRA`, w treści `N104NN` | Tak | klucz ≠ `tail_number` | `key_mismatch` |

- `from_json(..., {"mode": "PERMISSIVE", "columnNameOfCorruptRecord": "_corrupt_record"})` — pole musi być w schemacie; działa identycznie w Sparku i w Databricks SQL (`map(...)` zamiast słownika)
- `key_mismatch` jest groźny: o partycji decyduje **klucz**, więc telemetria innego samolotu ląduje w cudzej partycji i gwarancja kolejności staje się fikcją
- **Pułapka NULL:** `NULL != 'SP-LRA'` daje `NULL`, nie `true` — `key_mismatch` nie zadziała dla pustego `tail_number`. Do tego rekord dostaje tylko **pierwszy** pasujący powód z łańcucha `when()`/`CASE`
- `when()` bez `otherwise()` / `CASE` bez `ELSE` → brak powodu = rekord poprawny
- Testy na pustej tabeli zawsze przechodzą — kod obsługi błędów trzeba sprawdzić **celowo złymi rekordami** (`ingestion/produce_bad_telemetry.py`)
- dbt: wspólna logika w modelu `ephemeral`, z którego czytają Silver i kwarantanna — logika nie może się rozjechać
- Test pojedynczy (singular): każda wiadomość z Bronze trafia **dokładnie w jedno** miejsce — Silver albo kwarantannę
- sqlfluff nie odróżnia pola struktury (`data.tail_number`) od kolumny z tabeli → `RF01/RF03/AL09`. Rozwiązanie: rozpakować strukturę **raz**, w jednym bloku z `-- noqa: disable=...` / `-- noqa: enable=...` i komentarzem. Komentarz nie może zaczynać się od słowa `sqlfluff` — linter traktuje go jako konfigurację

---

## 10. Faza 3 — Aiven + Databricks

### Aiven (darmowy plan)
- Maks. **2 partycje** na topik (mapowanie kluczy inne niż lokalnie przy 3 — nie porównuj partycji między środowiskami)
- Retencja **3 dni**
- Wyłączanie serwisu przy dłuższej bezczynności

### Najpierw test największego ryzyka
Przed jakąkolwiek konfiguracją — czy serverless Databricks dosięgnie niestandardowego portu brokera:

```python
import socket

with socket.create_connection((host, port), timeout=10):
    print("OK")
```

### mTLS vs SASL

| | Client certificate (mTLS) | SASL (SCRAM) |
|---|---|---|
| Uwierzytelnienie | Klucz prywatny + certyfikat | Login i hasło |
| Sekret w sieci | Nigdy | Nie w jawnej postaci |
| Koszt | Trzy pliki, format klucza, rotacja | Niższy |

Decyduje głównie **sposób przechowywania sekretu**. Truststore (`ca.pem`) = komu ufam; keystore (`service.cert` + `service.key`) = kim jestem. "Mutual" = obie strony weryfikują się nawzajem.

- Klucz musi być **PKCS#8**: `BEGIN PRIVATE KEY` (a nie `BEGIN RSA PRIVATE KEY`)
- Certyfikaty poza repo + `.gitignore`: `*.pem`, `*.key`, `*.cert`
- Do Secret Scopes z zachowaniem nowych linii:

```powershell
databricks secrets put-secret kafka-aiven ca_pem --string-value (Get-Content -Raw C:\...\ca.pem)
```

### Konsument w Databricks

```python
.option("kafka.security.protocol", "SSL")
.option("kafka.ssl.truststore.type", "PEM")
.option("kafka.ssl.truststore.certificates", ca_pem)
.option("kafka.ssl.keystore.type", "PEM")
.option("kafka.ssl.keystore.certificate.chain", service_cert)
.option("kafka.ssl.keystore.key", service_key)
```

Opcje z prefiksem `kafka.` Spark przekazuje bezpośrednio do klienta Kafki.

### Joby z repozytorium Git

| Pułapka | Przyczyna | Rozwiązanie |
|---|---|---|
| `NameError: __file__` w Jobie typu Python script | Databricks wykonuje plik przez `exec()`, który nie ustawia `__file__` | `Path(globals().get("__file__") or sys.argv[0])` w jednym wywołaniu `sys.path.insert` (inaczej ruff `E402`) |
| `Cannot read the python file` | Ścieżka notebooka bez rozszerzenia | Przy źródle Git ścieżka z `.py` |
| Notebook jako Python script "działa" | `spark`/`dbutils` są akurat w kernelu | Typ **Notebook** — wstrzykiwanie jest wtedy gwarantowane |
| Job na gałęzi feature | Konfiguracja z testów | Po mergu przełączyć na `master` |

Moduły importowane (np. `reference_data.py`) mają `__file__` — problem dotyczy tylko pliku uruchamianego.

### Harmonogramy i monitoring
- Producent i konsument z **niezależnymi** harmonogramami — Kafka rozdziela nadawcę od odbiorcy, konsument przy każdym przebiegu odbiera wszystko od offsetu z checkpointu
- Świeżość Bronze (`_ingested_at`) łapie trzy awarie naraz: zatrzymany konsument, zatrzymany producent, wyłączony Aiven
- Progi świeżości są **powiązane z harmonogramami** (warn = jeden pominięty przebieg, error = dwa) i z retencją (zapas na reakcję przed utratą danych) — przy zmianie harmonogramu trzeba wrócić do progów
- Kontrola lagu przy `availableNow` byłaby trzecim alertem o tym samym zdarzeniu — ma sens dopiero przy ciągłym streamingu