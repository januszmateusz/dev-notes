# Kafka — ściągawka komend + kluczowe koncepcje

## Uruchomienie lokalnego brokera

`kafka/docker-compose.yml` w repo airline-cloud-warehouse — pojedynczy węzeł w trybie KRaft
(broker i kontroler w jednym procesie, bez ZooKeepera).

| Komenda | Co robi |
|---|---|
| `docker compose up -d` | Start brokera (z katalogu `kafka/`) |
| `docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --version` | Wersja Kafki (do pinowania tagu obrazu) |

Wszystkie narzędzia CLI leżą w kontenerze w `/opt/kafka/bin/`.

## Topiki

```powershell
# utworzenie topiku z 3 partycjami
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic aircraft-telemetry --partitions 3 --replication-factor 1

# szczegóły topiku (partycje, lider, repliki)
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic aircraft-telemetry

# lista wszystkich topików
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

`--replication-factor 1` — przy jednym brokerze inaczej się nie da (replika musi leżeć na innym brokerze). W produkcji typowo 3.

## Producent (wysyłanie wiadomości z kluczem)

```powershell
docker exec -it kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --reader-property parse.key=true --reader-property key.separator=:
```

Producent jest **interaktywny**: najpierw uruchom komendę, a dopiero w jego prompcie `>` wpisuj wiadomości w formacie `klucz:wartość`, każdą zatwierdzając Enterem. Zakończenie: `Ctrl+C`.

SP-LRA:{"alt": 1000}
SP-LRA:{"alt": 2000}
N104NN:{"alt": 500}
PH-BVA:{"alt": 1234}


**Pułapka PowerShell:** nie wklejaj wiadomości w tej samej linii co komendę — PowerShell sparsuje `{...}` jako własny blok skryptu i zwróci `Unexpected token ':'`, zanim komenda w ogóle trafi do Kafki.

## Konsument

```powershell
# bez grupy — czyta wszystko od początku, nie zapisuje postępu
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --from-beginning --formatter-property print.key=true --formatter-property print.partition=true

# w grupie — Kafka zapamiętuje, gdzie grupa skończyła czytać
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --group telemetry-readers --from-beginning --formatter-property print.key=true --formatter-property print.partition=true

# jedna konkretna partycja — bez grupy, bez offsetów, bez failover (głównie do debugowania)
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aircraft-telemetry --partition 1 --from-beginning --formatter-property print.key=true --formatter-property print.partition=true
```

## Grupy konsumentów

```powershell
# stan grupy per partycja: CURRENT-OFFSET, LOG-END-OFFSET, LAG, przypisany konsument
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group telemetry-readers

# członkowie grupy — widać też konsumentów bez przydzielonych partycji
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group telemetry-readers --members

# lista wszystkich grup
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# cofnięcie offsetów grupy na początek (grupa musi być nieaktywna — zatrzymaj konsumentów)
docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group telemetry-readers --topic aircraft-telemetry --reset-offsets --to-earliest --execute
```

## Kluczowe koncepcje

### Partycjonowanie po kluczu
- Partycja = `hash(klucz) mod liczba_partycji` → ten sam klucz **zawsze** trafia do tej samej partycji (sprawdzone: `PH-BVA` wysłany w różnych momentach zawsze w partycji 0)
- Działa w jedną stronę: jeden klucz → jedna partycja, ale jedna partycja → wiele kluczy (`SP-LRA` i `N104NN` wylądowały razem w partycji 1)
- Klucz wybiera się według tego, **w obrębie czego potrzebna jest kolejność** — dla telemetrii: samolot
- Gorący klucz (jeden klucz z większością ruchu) = przeciążona partycja = skew, ten sam problem co w Sparku
- Zmiana liczby partycji zmienia mapowanie kluczy → łamie kolejność dla istniejących kluczy. Liczbę partycji planuje się z góry

### Kolejność
- Kafka gwarantuje kolejność **tylko w obrębie partycji**, nie w całym topiku
- Między partycjami kolejność wysłania przepada (sprawdzone: `PH-BVA` wysłany jako ostatni, a wyświetlony jako pierwszy, bo leżał w partycji 0)

### Offsety i lag
- Offset jest liczony **osobno dla każdej partycji**, od 0 — nie ma jednego offsetu dla całego topiku
- `CURRENT-OFFSET` = numer **następnej** wiadomości do przeczytania przez grupę, nie ostatniej przeczytanej
- `LOG-END-OFFSET` = miejsce, gdzie kończy się partycja
- `LAG` = `LOG-END-OFFSET - CURRENT-OFFSET` — ile wiadomości grupa ma jeszcze do przeczytania. Kluczowa metryka produkcyjna: rosnący lag = konsument nie nadąża
- Commitowane offsety są trzymane w samej Kafce (topik `__consumer_offsets`), więc grupa pamięta postęp nawet bez aktywnych konsumentów

### `--from-beginning`
Działa **tylko dla grupy bez zapisanego offsetu**. Grupa z historią zawsze wznawia tam, gdzie skończyła — flaga jest wtedy ignorowana.

### Grupy konsumentów
- Partycje przydziela **broker** (group coordinator), nie konsument — komendy różnią się tylko nazwą grupy, podział zależy od liczby członków
- Każda partycja jest przypisana do dokładnie jednego konsumenta w grupie
- Więcej konsumentów niż partycji → nadmiarowi są bezczynni (gorący zapas)
- Nierówny podział: 3 partycje / 2 konsumentów = 2 + 1 (partycja jest niepodzielna)
- **Rebalance**: gdy konsument dołącza lub odpada, grupa na nowo dzieli partycje; nowy właściciel kontynuuje od zapisanego offsetu (sprawdzone testem failover)

### Grupy są od siebie niezależne
- Każda grupa ma własne offsety; nowa grupa czyta wszystko od początku
- Kafka **nie usuwa** wiadomości po przeczytaniu — trzyma je przez czas retencji. Konsumpcja to tylko przesunięcie "zakładki" grupy
- Różnica względem klasycznych kolejek (JMS/MQ), gdzie odebrana wiadomość znika — w Kafce ten sam strumień może czytać wielu niezależnych odbiorców

### Gwarancje dostarczenia
- Domyślnie **at-least-once**: jeśli konsument przeczyta wiadomość i padnie przed commitem offsetu, następca przeczyta ją ponownie
- Konsument musi umieć obsłużyć duplikaty (w projekcie: deduplikacja w Silver przez `ROW_NUMBER()`)

### Broker nie sprawdza zawartości
- Wartość wiadomości to dla Kafki tylko bajty — `{"txt": ...}` zamiast `{"alt": ...}` przeszło bez słowa
- Kontrakt danych trzeba wymusić gdzie indziej: po stronie konsumenta (parsowanie z obsługą niepasujących rekordów) albo przez Schema Registry

### Spark Structured Streaming + Kafka
- Spark śledzi przeczytane offsety we **własnym checkpoincie**, a nie w `__consumer_offsets` — ten sam mechanizm co w Auto Loaderze
- Klucz i wartość przychodzą jako tablice bajtów — deserializację JSON-a robi się w kodzie