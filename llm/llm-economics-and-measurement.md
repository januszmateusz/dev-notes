# LLM: ekonomia, latencja, sampling i pomiary

## Koszt

```
koszt = in_tok × cena_in + out_tok × cena_out + cache_read × cena_cache
        (+ próg długiego kontekstu, + zapis do cache, − zniżka Batch)
```

- **Tokeny liczę TYM modelem, którego użyję.** Tokenizery różnią się między rodzinami
  i wersjami (Haiku 5.5 ≈ +30% tokenów względem Haiku 4.5).
- **Polski ≈ 1,64× więcej tokenów niż angielski** (pomiar: 1,98 vs 3,33 znaku/token).
  Szacunek z długości tekstu: `tokeny ≈ znaki / 1,98`. Nie stosuj drugi raz,
  jeśli tokeny zmierzono na polskich danych.
- **Thinking = tokeny wyjściowe.** Effort to dźwignia kosztu i latencji.
- **Wyceniam z rozkładu, nie z jednego pomiaru:** średnia + wariant pesymistyczny (p95).
  Ta sama prośba dała 30 i 35 tokenów wyjścia. „Ok. 150 słów” dało 312–631 tokenów.
- **Cena za 1M tokenów to jednostka, nie rozmiar zapytania.** Przy progu cenowym
  zapytanie na 1M tokenów nigdy nie jest w taryfie podstawowej.
- **Ceny w jednym module, z datą i źródłem.** Granice progów w testach (`>` vs `>=`).
- **Batch API**: −50% kosztu, wyniki asynchronicznie (do 24 h). Nie szacuj czasu
  z latencji.

### Haiku 5.5 (stan 2026-10, weryfikuj w cenniku)

| | ≤ 100k tokenów promptu | > 100k |
| :--- | ---: | ---: |
| Wejście / 1M | $0,10 | $0,50 |
| Wyjście / 1M | $0,50 | $2,50 |
| Odczyt cache / 1M | $0,01 | $0,05 |

Okno kontekstu 1M tokenów, do 128k tokenów wyjścia.

## Latencja

- **Prefill** (przetwarzanie promptu): równoległy, szybki.
- **Generacja**: sekwencyjna, token po tokenie. Zwykle dominuje w latencji.
- **TTFT** = czas do pierwszego widocznego tokenu (z myśleniem włącznie).
- Czas lokalny na rekord: `in / prefill_tps + out / gen_tps + narzut`.
  Jedna wspólna stawka dla wejścia i wyjścia zawyża czas kilkanaście razy.
- Pierwsze wywołanie: ładowanie modelu (lokalnie) lub zestawianie połączenia (API).
  Zawsze kilka przebiegów i rozgrzewka.
- Krótka odpowiedź z API: latencję dominuje sieć i narzut, a nie generacja.

## Sampling

- Temperatura zmienia kształt rozkładu: niska = ostrzej, wysoka = płasko (i bzdurnie).
- **Temperatura 0 ≠ determinizm.**
- **Haiku 5.5: brak `temperature`/`top_p`/`top_k`** (400). Sterowanie promptem i `effort`.
- **Zmienność zależy od zadania:** zamknięta lista kategorii jest stabilna,
  zadanie kreatywne przy tej samej temperaturze bardzo zmienne.
- Stabilność zapewnia struktura odpowiedzi + pomiar na zbiorze testowym, nie sampling.

## Effort i adaptive thinking (Claude)

- Domyślnie włączony. Model może pominąć myślenie przy prostych zadaniach.
- Myślenie pojawiło się dopiero przy prompcie z regułą do zastosowania.
- Wyższy effort ≠ lepsza odpowiedź. **Wybieram najniższy effort, który osiąga
  wymaganą jakość na golden secie.**

## Jakość i halucynacje

- **Prompt jest specyfikacją.** Niejednoznaczne kategorie = wyniki zależne od konfiguracji.
  Definicje i przypadki graniczne w prompcie.
- **Stabilność ≠ poprawność.** Trzy identyczne odpowiedzi nic nie mówią o trafności.
- **Zmiana modelu lub parametru potrafi po cichu zmienić wyniki.** Model przypinam,
  zmianę traktuję jak aktualizację zależności (testy + ewaluacja).
- **Format odpowiedzi się rozjeżdża** (cudzysłowy, kropki). Wynik do tabeli:
  structured output + walidacja. Przed porównywaniem: standaryzacja wartości.
- **Model przyjmuje założenia z promptu bez sprawdzania.** Błędny kontekst = błędna
  odpowiedź podana pewnym tonem.

## Pomiary: pułapki

- **Cache prefiksu (Ollama, prompt caching w API) fałszuje benchmarki.** Zmieniaj
  początek promptu w każdym przebiegu.
- Przepustowość generacji mierz na długich odpowiedziach. Z 2 tokenów wychodzi szum.
- Tokenów wejściowych nie porównuj między modelami (różne tokenizery i szablony).
  Porównuj dolary i sekundy.
- Zapisuj sprzęt przy pomiarach lokalnych (CPU vs GPU = rząd wielkości różnicy).

## Model lokalny: kiedy ma sens

Nie dla kosztu przy dużej skali (mój pomiar na CPU: ok. 73 tys. rekordów w oknie 8 h,
1M rekordów = 4,6 doby; Haiku Batch: $8,30 za 1M). Tak, gdy:

- dane nie mogą opuścić infrastruktury,
- małe wolumeny i iteracje nad promptem,
- praca offline,
- jest GPU.

## Projekt promptu pod koszt

Wspólna instrukcja na początku, dane rekordu na końcu: działa cache prefiksu
(Ollama lokalnie, prompt caching w Claude).