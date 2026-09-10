# Refactor opportunities z długu ABAC — Plan Brief

> Pełny plan: `context/changes/refactor-opportunities/plan.md`
> Research: `context/changes/refactor-opportunities/research.md`

## Co i dlaczego

Research.md tej zmiany skatalogował 16 okazji refaktoru z długu udokumentowanego w analizie ABAC i wybrał TOP 3 wg stosunku kosztu długu do kosztu zmiany. Ten plan realizuje te trzy: serializację jsonb polityki dostępu (C2), widoczność szwu enterprise PAP/PDP w CI (C1) i rozszerzenie kontraktu OpenAPI o porównanie schematów (C3). Każda jest zmianą widoczności/konwencji, nie zachowania.

## Punkt wyjścia

Dziś: (1) `accessControlPolicyV0_1` w sqlstore ręcznie duplikuje pola modelu do kolumny `Data jsonb` — nowe pole może zniknąć bez błędu kompilacji; (2) interfejs PAP/PDP ma jedyną implementację w prywatnym module enterprise, którego CI nigdy nie kompiluje — zmiana sygnatury nie zostawia śladu; (3) specyfikacja OpenAPI dla `AccessControlPolicy` jest fikcją (6 pól-widm, brak ~10 realnych), a istniejący analizator `openApiSync` sprawdza tylko ścieżki/metody, nie schematy.

## Docelowy stan

Po tym planie: kolumna `Data` jest czytana/zapisywana przez typ `Scanner`/`Valuer` (konwencja repo, nie duplikat); sygnatura PAP/PDP jest commitowanym artefaktem wpiętym w `make generated` — zmiana szwu produkuje diff w PR; specyfikacja `AccessControlPolicy`/`AccessControlPolicyRule` odpowiada rzeczywistemu modelowi, a `make vet-api` blokuje PR przy nowym rozjeździe schematu.

## Kluczowe decyzje

| Decyzja | Wybór | Dlaczego (1 zdanie) | Źródło |
| --- | --- | --- | --- |
| Zakres kandydatów | Tylko TOP 3 (C1, C3, C2) | Najlepszy stosunek kosztu długu do kosztu zmiany, każdy ma już naszkicowaną ścieżkę w research | Research/Plan |
| Struktura | Jeden plan, fazy sekwencyjne | Zachowuje wspólną narrację rankingu i jedno miejsce śledzenia postępu | Plan |
| Kolejność faz | C2 → C1 → C3 (od najpewniejszego) | Użytkownik: budować solidne podstawy zanim ruszymy najbardziej rozległy kontrakt (REST) | Plan |
| C3: zakres naprawy | Spłata długu specyfikacji w tym samym planie (nie allow-list) | Świadomy wybór użytkownika mimo rekomendacji allow-listy w research | Plan |
| C2: głębokość | Pełne 3 kroki, usunięcie shadow struct | Kończy temat raz a dobrze, zero pozostałości długu | Plan |
| C1: głębokość | Tylko artefakt + `check-generated` (bez opcjonalnego analizatora govet) | Minimalny, w pełni odwracalny zakres zgodny z pierwszym miejscem w rankingu | Plan |
| U-CI-1 (czy `make vet-api` przechodzi) | Rozstrzygnięte: TAK, blokuje bez `continue-on-error` | Zweryfikowane w `server-ci.yml` — komentarz „currently not passing" w Makefile jest nieaktualny | Plan |

## Zakres

**W zakresie:** C2 (serializacja jsonb → Scanner/Valuer), C1 (artefakt sygnatury PAP/PDP w `make generated`), C3 oś 1 (Go↔YAML: naprawa spec + rozszerzenie `openApiSync`).

**Poza zakresem:** pozostałe 13 kandydatów z research.md (C4–C16 poza C2), osie 2 i 3 kandydatury C3 (Go↔TS, `client4.go`↔router), jakakolwiek zmiana zachowania API/UI, migracje bazy danych.

## Architektura / podejście

Trzy niezależne technicznie ścieżki (warstwa store / szew kompilacyjny / narzędzie CI), połączone we wspólnym planie ze względu na wspólne pochodzenie z jednego rankingu. Każda faza kończy się realnym, blokującym na PR mechanizmem CI, zanim plan przechodzi dalej.

## Fazy w skrócie

| Faza | Co dostarcza | Kluczowe ryzyko |
| --- | --- | --- |
| 1. Serializacja jsonb (C2) | `Scanner`/`Valuer` zamiast `accessControlPolicyV0_1`, test round-tripu | Wybór kształtu typu (osadzony pod-typ vs typ-widok) — patrz Critical Implementation Details w planie |
| 2. Widoczność szwu enterprise (C1) | Artefakt sygnatury PAP/PDP wpięty w `make generated` | Niestabilność generowanego artefaktu (migotanie) bez weryfikacji determinizmu |
| 3. Schematy OpenAPI (C3) | Naprawiona spec `AccessControlPolicy`/`Rule` + blokujący checker schematów | Większy blast radius niż allow-list — naprawa specyfikacji musi poprzedzić włączenie checkera |

**Prerekwizyty:** brak zależności od innych zmian w toku; brak migracji bazy danych.
**Szacowany nakład:** ~3 sesje implementacyjne, po jednej na fazę.

## Otwarte ryzyka i założenia

- Wybór kształtu typu Scanner/Valuer w Fazie 1 (osadzony pod-typ vs typ-widok w sqlstore) jest pozostawiony implementatorowi — oba spełniają test round-tripu.
- Rozszerzenie `openApiSync` o porównanie schematów (Faza 3) dotyka wyłącznie typów związanych z `access_control`; generalizacja na cały `public/model` to potencjalny follow-up, nie część tego planu.

## Kryteria sukcesu (skrót)

- `accessControlPolicyV0_1` nie istnieje w kodzie; pełna suita `storetest` dla `AccessControlPolicy` zielona.
- `make generated` wykrywa zmianę sygnatury PAP/PDP jako diff blokujący PR.
- `make vet-api` blokuje PR przy rozjeździe schematu `AccessControlPolicy`/`AccessControlPolicyRule`.
