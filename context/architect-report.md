---
title: Raport architektoniczny — moduł 4 (10xArchitect)
created: 2026-09-10
author: Sebastian Urbański
sources: [context/map/repo-map.md, context/changes/abac-work-in-progress/research.md, context/changes/refactor-opportunities/plan.md, context/domain/01-domain-distillation.md, context/domain/02-invariant-aggregate-refactor.md, context/domain/03-anti-corruption-layer.md]
---

# Raport architektoniczny — moduł 4

Każde twierdzenie liczbowe pochodzi z artefaktu wskazanego w nawiasie, nie z pamięci o kodzie.

---

## 1. Opisane projekty

**Jedno repozytorium dla wszystkich pięciu artefaktów: `mattermost`** (open core, self-hosted collaboration platform).

| | |
|---|---|
| **Stack** | Go (`server/`) + React/TypeScript (`webapp/`), PostgreSQL, single Linux binary ([01 §KROK 0](domain/01-domain-distillation.md) za README) |
| **Skala** | Go: 164 pakietów, `api4` = 764 endpointy, `client4.go` = 787 metod, `store.go` = 63 interfejsy / 1024 metody. Webapp: `channels/src` = 384 moduły, `client4.ts` = 557 metod, ~2700 nietkniętych plików komponentów ([repo-map §1-3](map/repo-map.md)) |
| **Artefakty** | **L2** mapa repo (HEAD `87168644a4`) · **L3** research ABAC (HEAD `87168644a4`) · **L4** plan refaktoru (bez zadeklarowanego HEAD) · **L5** trzy notatki DDD (01/02 — 2026-09-10; 03 — HEAD `5ce50eaa5c`) |

**BRAK artefaktu:** `context/foundation/prd.md` i `tech-stack.md` nie istnieją — cała klasyfikacja domenowa w L5 opiera się na README, mapie i kodzie, nie na spisanych celach biznesowych ([01 §KROK 0](domain/01-domain-distillation.md)).

---

## 2. Mapa projektu (L2)

1. **Centrum repo to `server/public/model`, nie `channels/app`** — fan-in 129, zasięg tranzytywny **84%** ze 164 pakietów; jego lustro `@mattermost/types` ma w webappie 2235 importów, więcej niż `react` (2101).
2. **Repo ma dwie różne architektury.** Go: 0 cykli, granica modułów egzekwowana kompilatorem. Webapp `channels/src`: **SCC = 192 z 384 modułów (50%)** — blast radius jest tam nieobliczalny z grafu, trzeba polegać na testach.
3. **Strefa ryzyka #1 — kontrakt REST**: 3 ręcznie pisane kopie, `client4.go` 787 vs `client4.ts` 557 metod, **30% zmian `client4.go` nie trafia nigdzie indziej**, zero commitów typu „sync spec" w 12 miesięcy. To luka narzędziowa — kontrakt wtyczek w tym samym repo ma generator + `plugin-checker` w CI.
4. **Strefa ryzyka #2 — ABAC**: 655 dotknięć / 125 commitów, migracje do ostatnich tygodni okna, pełny przekrój store→app→api4→admin UI→E2E, `app/access_control.go` bus factor 2.
5. **Entry pointy są cienkie, krawędzie runtime niewidoczne**: `main.go` = 23 linie z blank importami wpinającymi `enterprise/`; `einterfaces` 16 → 59 pakietów tranzytywnie; 4 moduły `channels/src` to de facto publiczne API wtyczek przez Module Federation z `strictVersion: false` — zero sygnału w CI.

**Unknowns mapy:** nie czytano kodu (brak `go` w PATH, brak `node_modules` — `go list`/`madge` niedostępne), nie mierzono jakości zmian, recenzenci PR-ów niewidoczni (squash merge), `CODEOWNERS` = 19 linii i nie pokrywa `public/model`, `platform/types` ani `api/v4/source`.

---

## 3. Analiza ficzera (L3)

**Co badane i dlaczego:** ścieżka **zapisu i przypisania polityki ABAC** (`PUT /api/v4/access_control_policies` + `POST /{id}/assign`) — wprost strefa ryzyka 🔴 nr 2 z mapy, jedyny przepływ dający pełny przekrój przez wszystkie warstwy wskazane w L2.

**Feature overview.** Input przychodzi z System Console (`policy_details.tsx` → thunk redux → `client4.ts` PUT); api4 dekoduje, przepuszcza przez dwie bramki flag i switch uprawnień per typ polityki, po czym woła `app.CreateOrUpdateAccessControlPolicy`. Stan zmienia się dopiero za szwem enterprise: app wymusza `Version=v0.3`, uruchamia strażników zapisu (merge zamaskowanych wyrażeń, `masked_rule_deleted`, self-inclusion) i woła `acs.SavePolicy`, który trafia do `sqlstore.Save` — **delete-and-reinsert w transakcji**, z polami `Imports/Rules/Roles/Scope/ScopeID` zwiniętymi w kolumnę `Data jsonb`. Wraca pełna polityka z wyrażeniami zamaskowanymi **tylko w odpowiedzi**, plus zdarzenie WS `channel_access_control_updated`. Pełny zapis z Admin Console to **pięć osobnych żądań HTTP bez rollbacku** (create → unassign → assign → set-active → sync-job).

**Technical debt — trzy najważniejsze:**

| # | Ryzyko | Dowód |
|---|---|---|
| **T1** | **Szew enterprise bez implementacji.** `AccessControlServiceInterface` = 20 metod, zero implementacji w repo (**ast-grep: 19 metod w `pap.go` + 1 w `pdp.go`**). Zmiana sygnatury kompiluje się, przechodzi testy i regeneruje mocki czysto — pęka dopiero w prywatnym module, którego to CI nie buduje. **39 guardów `acs == nil` w `server/`** (ast-grep; mapa/wcześniejszy opis mówiły o „pięciu entry pointach"). Trzy metody PAP mają zero wywołań i kolidują nazwą z metodami store o innej sygnaturze — grep tekstowy zwraca 16 mylących trafień | §2.1, §9.1-9.3 |
| **T2** | **Kontrakt ma nie trzy, a cztery ręcznie utrzymywane kopie.** Czwarta to `accessControlPolicyV0_1` w sqlstore — nowe pole modelu niebędące kolumną ginie po cichu (brak migracji, brak błędu kompilacji, brak testu). Opublikowana spec OpenAPI to fikcja: 6 pól-widm, brak 10 realnych, **zero schematu `AccessControlPolicyRule`** — czyli typu niosącego wyrażenie CEL. Mechanizm potwierdzony historią: **77% commitów modelu ABAC nie rusza spec-u**, ~45% nie rusza typów TS | §2.2, §2.4 |
| **T3** | **Luka testowa dokładnie tam, gdzie bezpieczeństwo.** Wszystkie testy write-path wyłączają `AttributeValueMasking`, które produkcyjnie jest **domyślnie włączone** (13 użyć `maskingOffTestConfig` — ast-grep potwierdził co do jednego). Gałąź `mergeFromStore=true` — ta z każdego realnego zapisu REST — ma zero pokrycia integracyjnego; strażnik `masked_rule_deleted` jest nietestowany (jedyny test mockuje `HasMaskedValuesForCaller` na `false`). Faktyczną siatką bezpieczeństwa jest **6 testów Playwright** wymagających licencjonowanego serwera EE | §2.3 |

**Weryfikacja ast-grepem (§9)** obaliła **4 twierdzenia**, wszystkie tego samego typu — przedwczesne „tylko tutaj". Najważniejsze: `enforceAccessControlPolicyWriteGuards` ma dwa call-site'y, a drugi to **nieopisana wcześniej druga ścieżka zapisu przez Plugin API**, która wymusza `v0.5` zamiast `v0.3` i świadomie omija merge zamaskowanych wyrażeń.

---

## 4. Plan refaktoryzacji (L4)

**Co refaktoryzowane:** trzy najlepiej ocenione okazje z długu ABAC, wszystkie będące zmianami **widoczności i konwencji** — żadna nie zmienia zachowania runtime API ani UI.

- **C2** — likwidacja `accessControlPolicyV0_1` (czwartej kopii kontraktu) na rzecz konwencji `sql.Scanner`/`driver.Valuer` już stosowanej w repo (`StringArray`, `ChannelBannerInfo`); format JSON w kolumnie **bajtowo identyczny**.
- **C1** — sygnatura PAP/PDP jako wygenerowany, zacommitowany artefakt wpięty w `make generated`, przez co dziedziczy blokujący `check-generated`; zmiana interfejsu bez regeneracji psuje PR.
- **C3** — naprawa schematów `AccessControlPolicy`/`AccessControlPolicyRule` w `definitions.yaml`, a potem rozszerzenie **istniejącego, już blokującego** analizatora `openApiSync` o porównanie schematów (dziś porównuje tylko ścieżki i metody).

**Czego świadomie NIE robimy:** nie ujednolicamy dwóch ścieżek zapisu REST/Plugin (`mergeFromStore=false` to udokumentowana decyzja); nie dodajemy kontroli optymistycznej do `Save`; nie budujemy generatora Go→TS ani porównania `client4.go` ↔ router; **nie zmieniamy nazwy pola JSON `total` ani typu `SessionOverrides` w Go** — to byłaby zmiana zachowania API, poprawiamy wyłącznie specyfikację; nie ruszamy mocków; zero migracji bazy w którejkolwiek fazie.

| Faza | Treść | Weryfikacja |
|---|---|---|
| **1 (C2)** | Test round-tripu pól → typ `Scanner`/`Valuer` → usunięcie struktury-cienia i przepięcie 5 call-site'ów | **auto**: `storetest` przeciw Postgresowi, `check-style`, `go build` · **ręcznie**: porównanie surowych bajtów kolumny `Data` przed/po |
| **2 (C1)** | Sprawdzenie determinizmu → generator sygnatury (wzorem `layer_generators`) → wpięcie w `make generated` | **auto**: dwukrotne uruchomienie daje identyczny plik, `make generated` czysty na `master` · **ręcznie**: dry-run PR z nieaktualnym artefaktem musi failować |
| **3 (C3)** | Krok 0: zmierzyć, czy `make vet-api` **dziś** przechodzi → naprawa spec-u → włączenie porównania schematów blokująco | **auto**: `make vet-api`, testy jednostkowe komparatora, `swagger-cli validate` · **ręcznie**: celowy rozjazd schematu musi zostać zgłoszony |

Kolejność faz to **świadomy wybór użytkownika** („fundamenty przed najszerszym kontraktem"), a nie porządek wg kosztu — wg rankingu w research C1 jest tańszy i w pełni addytywny.

---

## 5. Domena wg DDD (L5)

**Ubiquitous language (wybór):** **AccessControlPolicy** (polityka jako wyrażenie CEL nad atrybutami; typy `parent`/`channel`/`team`/`permission` + typy pluginowe `<pluginID>:<resourceType>`) · **AccessControlPolicyRule** (reguła: akcje + wyrażenie + opcjonalna rola kanałowa) · **Inherit** (przypięcie polityki nadrzędnej dopisuje `parent.ID` do `Imports`) · **Masking** (ukrywanie literałów przed wywołującym bez uprawnień; domyślnie **włączone**) · **PAP/PDP** (szew enterprise — 20 metod bez implementacji w repo).

**Najważniejsze rozjazdy model-vs-kod:** (a) Go emituje `create_at`, TS deklaruje `created_at?` — webapp czyta pole, którego serwer nigdy nie wysyła, i zawsze wpada w fallback `Date.now()`; (b) TS deklaruje `props: Record<string, unknown[]>`, Go zapisuje skalary — obejście przez `as unknown as` w **5 miejscach produkcyjnych** (system typów zgłosił problem pięć razy, pięć razy go uciszono); (c) `Channel.PolicyEnforced` wygląda jak stan encji, a jest liczone przy każdym odczycie przez `EXISTS (...)`; (d) komentarz „ABAC is gated at route registration" jest nieaktualny — `InitAccessControlPolicy()` rejestruje się bezwarunkowo, realną bramką jest `acs == nil → 501`.

**Niezmiennik #1 i jego agregat.** Destylacja (01) wskazała jako #1 strażnika `masked_rule_deleted`. **Ponowna, bezpośrednia weryfikacja kodu (02) to odwołała**: guard jest dziś współdzielony przez REST i Plugin API, a `mergeFromStore=false` po stronie pluginu jest spójne z tym, że GET pluginu w ogóle nie maskuje. Rozbieżność wersji `v0.3`/`v0.5` również okazała się celowa (dwa pod-typy, nie jeden agregat). Realnym #1 pozostaje:

> **`AccessControlPolicy.Type` jest niemutowalny po utworzeniu** — agregat **AccessControlPolicy**.

Cztery inne miejsca (`Channel.PolicyEnforced`, ochrona własności typu pluginowego, jednolite 404 dla pluginu, rozróżnienie no-policy/deny) **milcząco zakładają** ten niezmiennik, nie weryfikując go. Mimo to jest egzekwowany wyłącznie jako efekt uboczny jednej implementacji SQL-store, gołym `errors.New` (bez `AppError`, więc bez statusu HTTP i i18n), osiągalny z **sześciu** niezależnych miejsc zapisu w `app`, z czego tylko jedno ma własną, nazwaną barierę. Projekt: reguła przejścia stanu `ValidateRevision(existing)` na modelu + jedno repozytorium `Revise` jako wspólne wejście, store zostaje ostatnią linią obrony (sentinel error).

**Anti-Corruption Layer.** Przeciekająca zależność: **`github.com/dyatlov/go-opengraph`** (link preview). Przecieka przez **6 warstw** — model, app, store SQL, kontrakt store, wire REST i cache, plus ręcznie odtworzone lustro w webappie — a jej typ `opengraph.OpenGraph` jest jednocześnie **polem domenowym, formatem persystencji (kolumna `Data`), kontraktem wire i typem cache**. Dziś zna ją **7 plików produkcyjnych + 4 testowe** w 4 pakietach; po refaktorze ma zostać **1 plik adaptera**. Wybrana spośród trzech kandydatek (obok `gorilla/websocket` i trójki bibliotek semver), bo jako jedyna spełnia wszystkie kryteria naraz: biblioteka nieutrzymywana (pin z 2022), będąca parserem HTML w grafie zależności modułu SDK o fan-in 129 kompilowanego do każdej wtyczki, z udokumentowanym rozjazdem intencja-vs-kod (`Data any` „dla separacji", której `IsValid()` i tak nie egzekwuje) i blokiem `exclude` w manifeście jako dowodem, że jej niestabilność już dziś boli.

---

## 6. Decyzje, które należą do mnie

AI dobrze wykonywało pomiar i inwentaryzację, ale trzy rozstrzygnięcia były moje i szły **wbrew** jego pierwszemu wynikowi. Po pierwsze, nie przyjąłem raportów sub-agentów na słowo: zbudowałem osobną warstwę weryfikacji strukturalnej ast-grepem, która obaliła 4 twierdzenia o unikalności („tylko tutaj") i przy okazji odkryła drugą, nieopisaną ścieżkę zapisu przez Plugin API — a każde zwrócone zero konfrontowałem z grepem tekstowym, żeby odróżnić realny brak wystąpień od złego wzorca (i faktycznie złapałem cztery fałszywe zera). Po drugie, przed zaprojektowaniem refaktoru domenowego zweryfikowałem **własną wcześniejszą destylację** zamiast na niej budować — i odrzuciłem dwa z trzech wskazanych tam „złamanych" niezmienników jako nieaktualne albo celowe, zostawiając jeden realny (`Type`); gdybym pominął ten krok, powstałby plan naprawy problemu, którego już nie ma. Po trzecie, ustawiłem kolejność faz planu refaktoru na C2 → C1 → C3 **wbrew rankingowi kosztu**, który stawiał najpierw C1 jako tańszy i w pełni addytywny — bo chciałem, żeby warstwa store miała solidną konwencję, zanim ruszę szew kompilacyjny i najszerszy kontrakt. Świadomie też zawęziłem zakres: odrzuciłem ujednolicanie dwóch ścieżek zapisu i zmiany nazw pól JSON, bo to zmiany zachowania API przebrane za sprzątanie. Wreszcie, tam gdzie brakowało podstaw — brak PRD, brak `go` w PATH przy weryfikacji `make vet-api` — nie zgadywałem, tylko zapisałem to jako otwarte i wstawiłem do planu krok-prerekwizyt zamieniający założenie w zmierzony fakt.
