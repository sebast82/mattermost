---
date: 2026-09-10T13:42:00+02:00
researcher: Sebastian Urbański
git_commit: dd28f26aeba352a651eead94efc2f858591deff9
git_base_upstream: 87168644a48fa66f0229a64d1706a3223c465cea
branch: master
repository: mattermost
topic: "Refactor opportunities — które problemy z analizy ABAC warto naprawić, w jakim kształcie i w jakiej kolejności"
tags: [research, refactor, technical-debt, abac, einterfaces, rest-contract, ci-safety-nets, feasibility, intentionality, verified]
status: complete
last_updated: 2026-09-10
last_updated_by: Sebastian Urbański
method: "3 osie eksploracji (kształt / intencjonalność / wykonalność) na 4 sub-agentach; wejście: context/changes/abac-work-in-progress/research.md"
verification_commit: dd28f26aeba352a651eead94efc2f858591deff9
verification_method: "ast-grep 0.45.3 + grep/Select-String dla wszystkich twierdzeń strukturalnych (liczby metod, call-site'y, pary lustrzanych typów); żadna zmiana w repo, weryfikacja na tym samym commicie co research"
---

# Research: Refactor opportunities z długu udokumentowanego w analizie ABAC

**Data**: 2026-09-10T13:42:00+02:00
**Badacz**: Sebastian Urbański
**Commit**: `dd28f26aeb` (lokalny, na bazie upstream `87168644a4`)
**Branch**: `master`
**Repozytorium**: mattermost

---

## Pytanie badawcze

Analiza [abac-work-in-progress/research.md](context/changes/abac-work-in-progress/research.md) zapisała dług techniczny i ryzyka strukturalne klastra ABAC, celowo powstrzymując się od propozycji zmian. Ta zmiana odpowiada na pytanie, które tamta analiza zostawiła otwarte: **KTÓRE z tych problemów warto naprawić, w jakim docelowym kształcie i w jakiej kolejności.**

Etap: **eksploracja**. Żadnego refaktoru, żadnej decyzji. Wynik to ranking opcji z trade-offami, będący propozycją do osobnej sesji planowania.

**Metoda.** Ustalenia analizy ABAC przyjęte jako zebrane dowody, nie wyprowadzane od nowa. Priory przeczytane dodatkowo: [repo-map.md](context/map/repo-map.md), [artifact-1-territory.md](context/map/artifact-1-territory.md), [artifact-2-structure.md](context/map/artifact-2-structure.md). `context/foundation/lessons.md` nie istnieje. Następnie trzy osie eksploracji na czterech sub-agentach w trybie read-only: **obecny kształt** (C1–C8, C9–C16), **historia i intencjonalność**, **wykonalność migracji + konfiguracja CI**.

---

## Summary

Trzy rzeczy zmieniły obraz w stosunku do analizy wejściowej.

**1. UNKNOWN U2 jest rozstrzygnięty i jest gorszy, niż zakładano.** Testy Go serwera (`channels/app`, `channels/api4`, storetesty przeciw prawdziwemu Postgresowi) biegną na **każdym PR-ze** — [server-ci.yml:179-204](.github/workflows/server-ci.yml#L179). Ale testy Playwright ABAC — te, które analiza wskazała jako **faktyczną siatkę bezpieczeństwa write-path** — **nie biegną na `pull_request` w ogóle**: [e2e-tests-ci.yml:15](.github/workflows/e2e-tests-ci.yml#L15) ma wyłącznie `workflow_dispatch`, wyzwalany zewnętrznie po zbudowaniu obrazu enterprise, a status commitowy da się **ręcznie nadpisać na zielony** ([e2e-tests-override-status.yml:46-52](.github/workflows/e2e-tests-override-status.yml#L46)). Luki R1–R4 nie mają ochrony w pętli PR.

**2. Repo ma dokładnie te narzędzia, których ABAC nie używa — i CI je egzekwuje.** `check-generated` ([server-ci.yml:82-121](.github/workflows/server-ci.yml#L82)) to **blokujący na PR** mechanizm „wygeneruj i porównaj diff", obejmujący `store-layers`, `store-mocks`, `einterfaces-mocks`, `pluginapi` i `migrations.list` naraz. `mattermost-govet` ma **11 zarejestrowanych, ale niewłączonych analizatorów** ([tools/mattermost-govet/main.go:33-59](tools/mattermost-govet/main.go#L33)) — gotowy, pusty slot. `openApiSync` **już** parsuje spec biblioteką `libopenapi` i **blokuje PR** ([server-ci.yml:159](.github/workflows/server-ci.yml#L159)) — tyle że porównuje wyłącznie ścieżki i metody HTTP, nie schematy. To domyka obserwację §6.4 analizy ABAC: dyscyplina jest tam, gdzie jest generator, **bo CI ją wymusza**. Luka narzędziowa ma precyzyjnie zlokalizowaną krawędź.

**3. Cztery twierdzenia analizy ABAC upadły w konfrontacji z kodem, a trzy kandydatury upadły w konfrontacji z historią.** `GET /{id}/activate` **nie jest martwy** — ma 18 (raport: 14) wywołań w suicie E2E. Bezwarunkowa rejestracja `Init*()` **jest normą** (61/61 modułów), nie anomalią ABAC. Częściowa warstwa localcache **jest normą** (ABAC 3/12 jest powyżej mediany). A `mergeFromStore=false` na ścieżce plugin to **udokumentowana decyzja**, potwierdzona trzema niezależnymi źródłami — nie przeoczenie. Odwrotnie: dwie mutacje omijające warstwę EE okazały się pięcioma, a błędy `ReconcilePolicyTeamScope` są połykane w trzech miejscach, nie jednym.

**Wniosek dla rankingu.** Najmocniejsze możliwości refaktoru nie leżą tam, gdzie dług jest największy, tylko tam, gdzie **istnieje już nośnik i egzekwujący go mechanizm CI**, a zmiana jest addytywna i odwracalna. Trzy takie miejsca: uwidocznienie szwu enterprise przez `check-generated`, rozszerzenie `openApiSync` o schematy, oraz przeniesienie serializacji jsonb na istniejącą konwencję `Scanner`/`Valuer`. Wszystkie trzy są **zmianami widoczności i konwencji, nie zmianami zachowania** — i to jest ich główna zaleta w obszarze, o którym mapa repo mówi „praca w toku, wysokie ryzyko kolizji".

---

# 1. Inwentarz problemów i klasyfikacja

Kryterium: **KANDYDAT** to problem, którego naprawa zmieniłaby strukturę kodu. Wszystko inne (brakujący test, luka w dokumentacji, defekt jednoliniowy, fakt organizacyjny) nie jest kandydatem, ale wchodzi jako wejście do oceny wykonalności i kosztu.

## 1.1 Kandydaci

| # | Kandydat | Źródło w analizie ABAC |
|---|---|---|
| **C1** | Szew enterprise bez egzekwowania kompilacyjnego — PAP/PDP 20 metod bez implementacji, 41 guardów `acs == nil`, 3 martwe metody, 28 funkcji `Register*Interface` | §2.1 |
| **C2** | Czwarta kopia kontraktu — `accessControlPolicyV0_1` duplikujący model dla kolumny `Data jsonb` | §2.2, S8 |
| **C3** | Kontrakt Go↔TS↔OpenAPI utrzymywany ręcznie — brak generatora/checkera schematów | §2.2, [repo-map.md](context/map/repo-map.md) 🔴1 |
| **C4** | `Props` jako nietypowany worek bez akcesorów i stałych kluczy | D2 |
| **C5** | Dwie ścieżki zapisu o rozbieżnych inwariantach (REST v0.3/merge=true vs Plugin v0.5/merge=false) | S10, §9.4 |
| **C6** | Mutacje store'a omijające warstwę enterprise | S3b |
| **C7** | `Save` jako delete-and-reinsert bez kontroli optymistycznej + wyścig przy równoległym assign | S2 |
| **C8** | Short-circuit `Save` wyklucza `Props` i `Active`; osobna ścieżka `SetActiveStatusMultiple` | S5 |
| **C9** | `AccessControlPolicyHistory` bez kolumny `Active` | S4 |
| **C10** | Nieatomowy zapis orkiestrowany przez klienta (5 żądań, brak rollbacku) | S1 |
| **C11** | Połykane błędy rekoncyliacji + odrzucane polityki-dzieci w odpowiedzi ASSIGN | S3, S7 |
| **C12** | Anonimowe struktury request-body w api4 — bez typu w `model`, bez `IsValid()` | S9 |
| **C13** | Niespójny model bramkowania — realna brama to `acs == nil`, config nigdy niesprawdzany na ścieżce autoringu | §1.7, S6 |
| **C14** | `localcachelayer` implementuje 3 z 12 metod; embedding przekazuje po cichu | blast #6, ciche #14 |
| **C15** | Martwa/nieosiągalna powierzchnia API — `GET /{id}/activate`, tryb lokalny 14/17, brak metod w `client4.go` | §2.2 |
| **C16** | `Channel.PolicyEnforced` liczone podzapytaniem `EXISTS` przy każdym odczycie kanału | §1.4 |

## 1.2 Nie-kandydaci (wejście do oceny, nie do refaktoru)

| Pozycja | Dlaczego nie kandydat |
|---|---|
| R1–R10 — luki testowe (masking domyślnie off w testach, nietestowany `masked_rule_deleted`, połowa blokady Actions/Role, ścieżki bez sesji, treść zdarzenia WS, write-path webappu) | brak testu to brak osłony, nie kształt kodu; **kluczowe wejście do oceny ryzyka każdego kandydata** |
| D1 — `create_at` vs `created_at` | jednoliniowa korekta deklaracji TS + 2 odczyty; zero zmiany struktury |
| Treściowy drift OpenAPI: 6 pól-widm, brak 10 realnych, brak schematu reguły, `total` vs `total_count`, `session_overrides` jako string, `$ref` na przestarzały schemat | to zawartość dokumentu; **mechanizm** jej gnicia jest kandydatem (C3), treść nie |
| S6 — nieaktualny komentarz „ABAC is gated at route registration" | poprawka komentarza; jego wartość dowodowa mieści się w C13 |
| S8 — myląca nazwa `accessControlPolicyV0_1` | rename; wchłonięty w C2 |
| Bus factor 2, `cursor[bot]` jako trzeci najpłodniejszy autor | fakt organizacyjny, nie kod |
| Playwright linkowany przez `file:` zamiast wersji | konfiguracja zależności buildowych; poza obszarem tej analizy |
| Zero specy Cypress dla ABAC | stan wygaszanego narzędzia, nie dług ABAC |
| Brakujące metody ABAC w `client4.go` | dług **addytywny** (dopisać kod), nie strukturalny — ale jest prerekwizytem testowalności, patrz C15 |

---

# 2. Kandydaci — kształt, intencjonalność, wykonalność

Legenda tagów: **[E]** evidence (plik:linia) · **[I]** inference · **[U]** unknown.

## C1 — Szew enterprise bez egzekwowania kompilacyjnego

**Obecny kształt.** [einterfaces/access_control.go](server/einterfaces/access_control.go) składa PAP ([pap.go:14](server/einterfaces/pap.go#L14)) + PDP ([pdp.go:14](server/einterfaces/pdp.go#L14)); wstrzyknięcie w [channels.go:204-206](server/channels/app/channels.go#L204) i [platform/service.go:576](server/channels/app/platform/service.go#L576). **[E]** Funkcji `Register*Interface` jest **28** (21 w [app/enterprise.go:13-133](server/channels/app/enterprise.go#L13), 7 w [platform/enterprise.go:13-49](server/channels/app/platform/enterprise.go#L13)), wszystkie o identycznym kształcie. **[E]**

**Korekta wobec analizy ABAC.** Guardów `acs == nil` jest **41 w `server/`, 21 (raport: 22) w `access_control.go`** (nie 39/21) — doszły m.in. [channel_join_request.go:331](server/channels/app/channel_join_request.go#L331) i [team_directory_visibility.go:64](server/channels/app/team_directory_visibility.go#L64). **[E]** Różnica nie zmienia wniosku.

**Ustalenie, które zmienia ocenę.** Rozsiany guard **nie jest anomalią ABAC, tylko normą repo**: `.Metrics()` 13× (raport: „.Metrics != nil” 66×), `.License()` 56× (raport: 53×), `.DataRetention()` 15× — liczba potwierdzona, ale operator to `== nil`, nie `!= nil` jak w raporcie — `.Cluster()` 21× (raport: 11×) (szczegóły w sekcji weryfikacji). **[E]** Żaden einterface w tym repo nie ma implementacji domyślnej, no-op ani null-objectu. **[E]** Zmiana tego wzorca **w samym ABAC** oznaczałaby rozjazd z resztą repo, nie zbliżenie do niej.

**Intencjonalność: ŚWIADOME OGRANICZENIE** (podział OSS/EE) / **UNKNOWN** (procedura zmiany interfejsu). Decyzja jest repo-wide i starsza niż ABAC — [server/enterprise/README.md:3](server/enterprise/README.md#L3): *„This folder contains source available enterprise code as well as import directives for closed source enterprise code… If you have a copy of https://github.com/mattermost/enterprise checked out as a peer to this repository, `enterprise` will be set automatically"*. Commit wprowadzający szew (`a344b3225b`, *„[MM-61756] Attribute Based Access Control - Phase 1 (#30785)"*) ma w body **wyłącznie listę 16 numerów ticketów** — zero uzasadnienia. Procedury bezpiecznej zmiany interfejsu **nie ma nigdzie**: sprawdzone `docs/develop/**`, [server/AGENTS.md](server/AGENTS.md), `CONTRIBUTING.md`, komentarze w `pap.go`. Jedyny ślad to [server/AGENTS.md:3](server/AGENTS.md#L3): *„Never run `go mod tidy` directly. Always run `make modules-tidy` instead — it excludes private enterprise imports that would otherwise break the tidy."*

**Wykonalność.** Osłony realne: **zero, i to zero strukturalne**. `BUILD_TAGS += enterprise` włącza się **tylko gdy katalog `../../enterprise` istnieje** ([server/Makefile:62-79](server/Makefile#L62)); CI klonuje wyłącznie `mattermost/mattermost`, więc `enterprise/external_imports.go` (`//go:build enterprise`) **nigdy się w CI nie kompiluje**. **[E]** Do tego `einterfaces-mocks` z `all: true` w [.mockery.yaml](server/einterfaces/.mockery.yaml) **aktywnie maskuje** rozjazd — mocki przegenerują się czysto pod dowolną nową sygnaturę. To jedyny kandydat, dla którego CI **strukturalnie nie może** wykryć regresji.

Ale nośnik istnieje i jest mocny: `check-generated` **blokuje na PR** i porównuje diff po `make generated` ([server-ci.yml:114-121](.github/workflows/server-ci.yml#L114), [server/Makefile:455](server/Makefile#L455)). Każdy artefakt wpięty w `make generated` dziedziczy tę egzekucję za darmo. Drugi nośnik: `mattermost-govet` z 11 wolnymi slotami analizatorów + własne CI ([tools-ci.yml:3-11](.github/workflows/tools-ci.yml#L3)).

**Pierwszy krok-prerekwizyt.** Wygenerować **sygnaturę obu interfejsów jako artefakt tekstowy** wpięty w `make generated` — zmiana szwu zaczyna produkować widoczny diff w PR. To pomiar, nie refaktor; zero zmian zachowania.

---

## C2 — `accessControlPolicyV0_1` jako czwarta kopia kontraktu

**Obecny kształt.** [sqlstore/access_control_policy_store.go:26-32](server/channels/store/sqlstore/access_control_policy_store.go#L26) + `toModel` [:49](server/channels/store/sqlstore/access_control_policy_store.go#L49) + `fromModel` [:81](server/channels/store/sqlstore/access_control_policy_store.go#L81); **15 zmierzonych wprost odwołań do `storeAccessControlPolicy`/`accessControlPolicyV0_1` (raport: 20 — definicja „odwołania” niejednoznaczna, patrz sekcja weryfikacji)**. **[E]** Dodanie pola niekolumnowego wymaga edycji [:26-32](server/channels/store/sqlstore/access_control_policy_store.go#L26), [:86-92](server/channels/store/sqlstore/access_control_policy_store.go#L86) i [:60-68](server/channels/store/sqlstore/access_control_policy_store.go#L60). **[E]**

**Ustalenie kluczowe: repo ma konwencję, a ABAC jej nie używa.** Nazwany wzorzec konwersji wiersz→model (`ToModel()`) występuje też w `channel_store.go`, `group_store.go`, `team_store.go` i `role_store.go`, ale w innej roli — mapuje cały wiersz DB na strukturę modelu, nie serializuje pojedynczą kolumnę `jsonb` przez prywatną strukturę-cień; w tej węższej roli para `fromModel`/`toModel` rzeczywiście występuje w `sqlstore/` **tylko tutaj** (doprecyzowanie, patrz sekcja weryfikacji). **[E]** Kolumn `jsonb` jest wiele (`Users.Props`, `Posts.Props`, `Jobs.Data`, `PropertyFields.Attrs`, `Channels.BannerInfo`), ale kanoniczny wzorzec to **`sql.Scanner`/`driver.Valuer` na typie modelowym** — [model/utils.go:100-207](server/public/model/utils.go#L100) (`StringArray`, `StringMap`, `StringInterface`) i [model/channel.go:59,72](server/public/model/channel.go#L59) (`ChannelBannerInfo`). **ABAC jest wyjątkiem, nie wzorcem.** **[E]**

**Intencjonalność: PRZYPADKOWA ZŁOŻONOŚĆ** — pewność wysoka. Struktura i oba komentarze pochodzą z `10b1f4c5ac` (*„[MM-63428] add access control policy store (#30597)"*, body puste). Komentarz [:46-47](server/channels/store/sqlstore/access_control_policy_store.go#L46) brzmi dosłownie: *„This needs to be updated with the new version of the policy. with the new name as this only supports v0.1"* — **to TODO, którego nigdy nie wykonano**. Dowód narastania: `c66bb0ecdb` (*„[MM-68109] Introduce new policy version v0.3"*) podbił wersję i przepisał `toModel()`, **zostawiając nazwę `accessControlPolicyV0_1`**; `Scope`/`ScopeID` dopchnięto do tej samej struktury w `80b977807a`. Śladu decyzji „jsonb zamiast kolumn" nie ma w żadnym z 20 commitów tego pliku.

**Wykonalność: ŁATWA I ODWRACALNA.** Osłona już istnieje i **biegnie na PR przeciw prawdziwemu Postgresowi**: storetest `ScopeRoundtrip` ([storetest/access_control_policy_store.go:1392-1497](server/channels/store/storetest/access_control_policy_store.go#L1392)) to gotowy wzorzec testu round-tripu pola przez blob. Jej wartość jest jednak **selektywna** — pinuje `Scope`, nie pinuje kompletności.

**Pierwszy krok-prerekwizyt.** Test wyliczeniowy (refleksyjny): każde pole `model.AccessControlPolicy` niebędące kolumną musi przetrwać `Save`→`Get`. Dopisanie do istniejącej suity [TestAccessControlPolicyStore](server/channels/store/storetest/access_control_policy_store.go#L24) to jedna linia. Zamienia cichą stratę w czerwony CI.

---

## C3 — Kontrakt Go↔TS↔OpenAPI bez generatora ani checkera schematów

**Obecny kształt narzędziowy.** `make pluginapi` ([server/Makefile:439-441](server/Makefile#L439)) = `go generate ./plugin` → `public/plugin/interface_generator/main.go`; produkuje 3 pliki glue **Go↔Go po RPC — nie dotyka REST ani TS**. **[E]** Wszystkie 4 dyrektywy `//go:generate` w `server/` to: oembed ×2, store, plugin. **[E]** Po stronie `api/`: [api/Makefile:11-71](api/Makefile#L11) konkatenuje 57 plików YAML, potem `swagger-cli validate` — czyli **poprawność strukturalna, nie zgodność z Go**. **[E]**

**Nośnik istnieje i jest mocniejszy, niż zakładała analiza.** [tools/mattermost-govet/openApiSync/openApiSync.go:140-181](tools/mattermost-govet/openApiSync/openApiSync.go#L140) **już** parsuje spec biblioteką `libopenapi`, buduje model v3, porównuje zarejestrowane `BaseRoutes.*.Handle(...).Methods(...)` z `paths` spec-u i **biegnie blokująco na PR** ([server-ci.yml:159](.github/workflows/server-ci.yml#L159)). Brakuje mu **wyłącznie porównania schematów**. **[E]** Drugi, częściowy nośnik: [api/server/main.go](api/server/main.go) parsuje AST katalogu `server/public/model` i dopasowuje funkcje `ExampleClient4_<operationId>` — ale **brak dopasowania kończy się cichym `return`**, a w `public/model` jest 33 takie funkcje, **żadna nie dotyczy ABAC**. **[E]** Po stronie TS nośnikiem jest własny plugin ESLint z 3 regułami i testami reguł ([webapp/platform/eslint-plugin/rules/](webapp/platform/eslint-plugin/rules/index.js)).

**Intencjonalność: MIESZANA, i to jest najciekawsze ustalenie tej osi.**
- *Czy próbowano automatyzacji Go→TS:* **nie**. `git log --all -S` dla `tygo` i `go2ts` → **zero trafień**; `openapi-generator` trafia tylko w Boards. **[E]**
- *Dlaczego `vet-api` ma komentarz „currently not passing":* **ŚWIADOME OGRANICZENIE, zadeklarowane jako tymczasowe w 2023 roku.** Commit `0577a5aaa2` (2023-10-16, Jesse Hallam, *„Fix OpenApi vetting (#23974)"*), cytat: *„The underlying mattermost-govet tool effectively hasn't been called for some time… Unfortunately, our API documentation isn't up-to-date, and this PR isn't fixing that. **For now**, add a discrete `make vet-api` and workflow that won't block the build **until the API documentation is back in sync**"*. Deklaracja tymczasowości ma ponad dwa lata.
- *Dlaczego review nie zgłasza spec-u:* **ŚWIADOME**, ale o innym zakresie niż sądzono. [docs-impact-review.yml:137](.github/workflows/docs-impact-review.yml#L137): *„Changes to `api/v4/source/` YAML files are part of the PR itself and are automatically published… Do not create action items for them."* To decyzja o zakresie **repo dokumentacji**, nie stwierdzenie poprawności specyfikacji.
- *Dlaczego `openApiSync` sprawdza tylko ścieżki:* **UNKNOWN** — analizator trafił do monorepo dopiero `4d20645a5b` (*„Inline mattermost-govet into the monorepo (#35869)"*), wcześniejsza historia jest w innym repo.

**Wykonalność: ŚREDNIA, rozcinalna na trzy niezależne osie** — (1) schemat OpenAPI vs Go, (2) TS vs Go, (3) `client4.go` vs router. Oś (1) da się wdrożyć w trybie **allow-list**: checker startuje z listą wyjątków obejmującą cały obecny dług i pilnuje wyłącznie *nowych* rozjazdów. Po tym kroku repo jest w pełni spójne, a dług przestaje rosnąć.

**Pierwszy krok-prerekwizyt.** Zbudować **inwentarz rozjazdu jako plik-baseline** (Go ↔ YAML ↔ TS, pole po polu) — materializacja tabeli z §2.2 analizy ABAC w formie maszynowo czytelnej. Bez niej każde uruchomienie checkera zaleje CI błędami zastanymi.

**Zastrzeżenie [U-CI-1].** Nie wiadomo, czy `make vet-api` **faktycznie przechodzi** na HEAD: [server/Makefile:1003](server/Makefile#L1003) mówi „currently not passing", a job nie ma `continue-on-error`. `go` nie jest dostępne w tym środowisku. Jeśli job jest de facto czerwony i tolerowany, jego wartość jako osłony jest zerowa — to trzeba rozstrzygnąć **przed** planowaniem.

---

## C4 — `Props` jako nietypowany worek

**Obecny kształt.** [access_policy.go:213](server/public/model/access_policy.go#L213) `Props map[string]any`; zapis wyłącznie przez indeks w [access_control.go:2098-2103](server/channels/app/access_control.go#L2098); TS [access_control.ts:16](webapp/platform/types/src/access_control.ts#L16) `props?: Record<string, unknown[]>`; 5 rzutowań `as unknown as` w webappie. **[E]**

**Ustalenie doprecyzowujące.** Gołe `map[string]any` w `public/model` mają też `Channel.Props`, `Product.Metadata`, `Manifest.Props`, `SessionAttributes.Attrs` — czyli worek jako taki jest powszechny. **Ale**: `Post` ma 5 akcesorów (`GetProps`, `SetProps`, `AddProp`, `DelProp`, `GetProp`, [post.go:749-779](server/public/model/post.go#L749)) plus **31 stałych kluczy `PostProps*`**, a samo pole jest oznaczone `// Deprecated: use GetProps()`. `Channel` ma `AddProp` i stałą klucza. **`AccessControlPolicy` nie ma żadnego akcesora ani stałej.** **[E]** Wyjątkiem jest więc **surowy dostęp indeksowy**, nie sam worek. **[I]**

**Intencjonalność: nie badana osobno** — [U]. Brak komentarza uzasadniającego; brak commitu z uzasadnieniem w zakresie badania.

**Wykonalność: ŁATWA I ODWRACALNA — najmniejszy blast radius z całej listy.** 3 zapisy Go + 5 odczytów TS. Rozcięcie naturalne: naprawić deklarację TS → usunąć 5 rzutowań → (opcjonalnie) akcesory + stałe po stronie Go. Osłony `check-types` i `check-lint` biegną na PR ([webapp-ci.yml:22,83](.github/workflows/webapp-ci.yml#L22)) — są sprawne, ale **celowo uciszone**: `as unknown as` to dokładnie ta składnia, której jedyną funkcją jest ich stłumienie.

**Pierwszy krok-prerekwizyt.** Reguła w istniejącym pluginie ESLint w trybie `warn`, wskazująca 5 miejsc. Zero zmian produkcyjnych, natychmiastowy pomiar.

---

## C5 — Dwie ścieżki zapisu o rozbieżnych inwariantach

**Obecny kształt.** Zestawienie krok po kroku (sub-agent kształtu, wszystko **[E]**):

| Krok | REST [access_control.go:123-180](server/channels/app/access_control.go#L123) | Plugin [plugin_access_control.go:170-240](server/channels/app/plugin_access_control.go#L170) |
|---|---|---|
| audit start | ❌ (w api4) | ✅ [:172-181](server/channels/app/plugin_access_control.go#L172) |
| `acs == nil` → 501 | ✅ [:125](server/channels/app/access_control.go#L125) | ✅ [:184](server/channels/app/plugin_access_control.go#L184) |
| ID | generuje, gdy pusty [:129](server/channels/app/access_control.go#L129) | **wymaga**, inaczej 400 [:188](server/channels/app/plugin_access_control.go#L188) |
| eligibility kanału | ✅ [:140-148](server/channels/app/access_control.go#L140) | ❌ |
| Version | `v0.3` [:150](server/channels/app/access_control.go#L150) | `v0.5` [:199](server/channels/app/plugin_access_control.go#L199) |
| `Active` | nietykane | wymuszane `true` [:200](server/channels/app/plugin_access_control.go#L200) |
| jawne `IsValid()` | ❌ (dopiero w store) | ✅ [:221](server/channels/app/plugin_access_control.go#L221) |
| write guards | `mergeFromStore=true` [:166](server/channels/app/access_control.go#L166) | `false` [:225](server/channels/app/plugin_access_control.go#L225) |
| sesja | z `rctx.Session()` | syntetyzowana [:231](server/channels/app/plugin_access_control.go#L231) |
| WS publish | ✅ | ❌ |

Wspólne jest **wyłącznie**: guard `acs == nil` + `enforceAccessControlPolicyWriteGuards` + `acs.SavePolicy`.

**Intencjonalność: ŚWIADOME OGRANICZENIE — pewność wysoka, trzy zbieżne dowody.** Doc-komentarz [plugin_access_control.go:164-169](server/channels/app/plugin_access_control.go#L164): *„Write guards go through enforceAccessControlPolicyWriteGuards with mergeFromStore=false: **plugin GET returns unmasked policies, so there is no masked-value round-trip to repair.**"* Symetrycznie po stronie guardów, [access_control.go:409-412](server/channels/app/access_control.go#L409): *„only when mergeFromStore is true (channel/team UI round-trips masked GET responses; plugin GET is unmasked…)"*. I chronologia: ścieżka plugin powstała w `c7eff700` (*„ABAC: plugin-keyed resource types, **trusted plugin** PAP/CEL APIs…"*), a parametr `mergeFromStore` dodano **miesiąc później** commitem `cfab0366`, którego body mówi wprost: *„**Extract enforceAccessControlPolicyWriteGuards so plugin saves get value-holding validation when AttributeValueMasking is on (without store merge)**"*. **To nie jest przeoczenie — parametr został dodany celowo, żeby podłączyć ścieżkę plugin z jawnie nazwanym wyjątkiem.** Open Question #7 analizy ABAC jest tym rozstrzygnięty.

**C5b — pięć wersji polityki.** v0.1..v0.5 wprowadzali różni ludzie w rocznym rozstrzale; **wszystkie cztery commity mają puste body**. Uzasadnienie żyje wyłącznie w kodzie i jest jednoznaczne dla dwóch najnowszych: [access_policy.go:477-479](server/public/model/access_policy.go#L477) (*„…remain membership-only at v0.4 (**multi-action support there is a follow-up iteration**)"*) i [:592-597](server/public/model/access_policy.go#L592) (*„v0.5 is the lane for plugin-owned resource policies, which differ from the core versions in ways that are **not expressible there**"*). Walidatory v0.1→v0.4 to **przyrost przez kopiuj-wklej** (te same 5 preambuł), ale nie czysty: v0.2 **usuwa** dwa checki v0.1. Migracja istnieje **jedna** (v0.2→v0.3, [migration.go:65](server/public/model/migration.go#L65)). **Kto zapisuje v0.4 — UNKNOWN**: wszystkie 4 wystąpienia stałej w kodzie non-test to *odczyty*; jedyna hipoteza (enterprise `acs.SavePolicy`) jest niesprawdzalna z tego repo.

**Wykonalność.** Blast radius mały (2 pliki), ale rozcięcie **ryzykowne semantycznie** — a teraz wiadomo, że ryzyko polega na czymś innym, niż sądzono: ujednolicenie ścieżek **skasowałoby udokumentowany feature**, nie naprawiło bug. Osłony asymetryczne: ścieżka plugin ma 51 wstrzyknięć mocka + 2 dedykowane pliki testowe (najlepiej pokryty fragment slice'u), ścieżka REST z `mergeFromStore=true` ma lukę R1 — zero pokrycia integracyjnego.

---

## C6 — Mutacje store'a omijające warstwę enterprise

**Korekta wobec analizy ABAC: nie dwa zapisy, a pięć mutacji.** Bezpośrednich wywołań `Store().AccessControlPolicy()` poza store'em jest **27 w 7 (raport: 8) plikach**; mutujących: `Save` ×2 ([team_access_control.go:279](server/channels/app/team_access_control.go#L279), [migrations.go:1311](server/channels/app/migrations.go#L1311)), `Delete` ×2 ([channel.go:4641](server/channels/app/channel.go#L4641), [team.go:2195](server/channels/app/team.go#L2195)), `SetActiveStatusMultiple` ×1 ([access_control.go:1941](server/channels/app/access_control.go#L1941)). **[E]**

**Intencjonalność: ŚWIADOME OGRANICZENIE dla 4 z 5, UNKNOWN dla migracji.**
- `ReconcilePolicyTeamScope`, [team_access_control.go:275-277](server/channels/app/team_access_control.go#L275): *„Save directly via the store — **scope is metadata that doesn't require enterprise-layer CEL normalization.** This avoids re-processing expressions and works consistently regardless of the enterprise layer state."*
- `Delete` w `channel.go`/`team.go` — **najlepiej udokumentowana decyzja w całym klastrze**, [channel.go:4600-4618](server/channels/app/channel.go#L4600): *„When the enterprise access control service is unavailable (acs == nil) or reports the operation as unsupported… **we still need to remove the underlying row to avoid leaving an orphaned policy behind. In those cases we fall back to deleting directly through the access control policy store.**"*
- **Korekta wobec sub-agenta kształtu:** `access_control.go:1941` **nie jest obejściem** — `UpdateAccessControlPoliciesActive` ma pełny guard `acs == nil → 501` na [:1913](server/channels/app/access_control.go#L1913) i guard maskingu przed zapisem. Store jest użyty *po* bramce, nie zamiast niej.
- `migrations.go:1311` — **UNKNOWN**: zero komentarza, commit bez body. Sub-agent kształtu ustalił jednak, że **migracja NIE biegnie przed inicjalizacją ABAC** — `NewChannels` (gdzie ustawiane jest `ch.AccessControl`) jest wołane w [server.go:305](server/channels/app/server.go#L305), a `doAppMigrations()` w [server.go:626](server/channels/app/server.go#L626). **[E]** Czyli popularne uzasadnienie „migracja biegnie za wcześnie" jest **fałszywe**.

**Wykonalność.** Powierzchnia minimalna, ryzyko w semantyce. Nośnik dla checkera istnieje: analizator govet z allow-listą, wzorowany na `noSelectStar` ([server/Makefile:978](server/Makefile#L978)), który robi dokładnie to samo dla SQL-a. Osłony: `ReconcilePolicyTeamScope` ma 8 testów biegnących na PR; migracja — **brak testów migracji w repo w ogóle** (0 plików `*_test.go` w `server/channels/db/`).

---

## C7 — `Save` jako delete-and-reinsert bez kontroli optymistycznej

**Obecny kształt.** [sqlstore:188-307](server/channels/store/sqlstore/access_control_policy_store.go#L188); klauzula `WHERE` przy `DELETE`/`INSERT` używa **wyłącznie `ID` — zero predykatu wersyjnego**. **[E]**

**Ustalenie: repo ma trzy wzorce kontroli optymistycznej i żadnego z nich tu nie użyto.**
- [job_store.go:216-245](server/channels/store/sqlstore/job_store.go#L216) `UpdateOptimistically` — `Where(sq.Eq{"Id":…, "Status": currentStatus})` + `RETURNING`, metoda jest w interfejsie store.
- [property_field_store.go:414-429](server/channels/store/sqlstore/property_field_store.go#L414) — komentarz *„Optimistic concurrency… closes the TOCTOU window"*, `Where("UpdateAt = …")` + weryfikacja `RowsAffected`.
- [plugin_store.go:101](server/channels/store/sqlstore/plugin_store.go#L101) — CompareAndSet.

Blokad pesymistycznych (`FOR UPDATE`) w `sqlstore/` **nie ma wcale**. **[E]**

**Intencjonalność: ŚWIADOME co do wzorca rewizji / UNKNOWN co do braku kontroli współbieżności.** [sqlstore:175-178](server/channels/store/sqlstore/access_control_policy_store.go#L175): *„**since policies are immutable, we need to create a new revision**…"*, plus [:227-229](server/channels/store/sqlstore/access_control_policy_store.go#L227): *„**Name changes are cosmetic and should not create a new revision in history.**"* To projekt niemutowalnych rewizji, nie efekt uboczny. **Ale**: `git log -S` po `rollback`/`transaction` na plikach ABAC app-layer → **zero wyników**; nikt w historii tego pliku nie napisał o wyścigu ani TOCTOU. Kontekst czasowy jest znaczący: `job_store.UpdateOptimistically` istnieje **od 2023-03-22, dwa lata przed** `Save` ABAC. Wzorzec był dostępny i nie został użyty, bez zapisanego powodu. **To jedyne miejsce w całym badaniu, gdzie brak dowodu sam w sobie coś sugeruje.**

**Wykonalność: RYZYKOWNA.** Kontrola optymistyczna wymaga, żeby klient przysyłał rewizję — a webapp nie wysyła nawet `version` ani `active` ([policy_details.tsx:277-282](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L277)), plugin API też nie, a `ReconcilePolicyTeamScope` robi read-modify-write bez niej. **Nie ma kroku pośredniego, po którym repo jest spójne, a zachowanie niezmienione** — poza samą detekcją (log ostrzegawczy). Osłony dla wyścigu: zero; `-race` biegnie tylko nocą i tak nie wykryje wyścigu międzytransakcyjnego w bazie.

---

## C8 — Short-circuit `Save` wyklucza `Props` i `Active`

**Obecny kształt — szerzej, niż podawała analiza.** Porównanie to `bytes.Equal(storePolicy.Data, tmp.Data) && storePolicy.Version == tmp.Version` ([:230-231](server/channels/store/sqlstore/access_control_policy_store.go#L230)). Poza porównaniem zostają **`Props` i `Active`** — oba są kolumnami i oba trafiają do `INSERT`. W gałęzi short-circuit aktualizowana jest wyłącznie `Name` i **zwracany jest stan sprzed zmiany**. **[E]**

**Konsekwencja strukturalna.** `SetActiveStatusMultiple` ([:416-490](server/channels/store/sqlstore/access_control_policy_store.go#L416)) istnieje jako osobna ścieżka **dlatego**, że `Save` nie potrafi przepchnąć zmiany różniącej się tylko `Active`. **[I]** Bliźniak `SetActiveStatus` dodatkowo propaguje `Active` na dzieci przez `Data->'imports' @> ?::jsonb`. **[E]**

**Wykonalność: ŁATWA**, ale z pułapką: włączenie `Props` do porównania **wygeneruje nowe rewizje historii przy operacjach, które dziś ich nie generują** (bo `Props` jest przeliczane przy każdym `HydrateChannelPolicyActions`). Osłony dobre — gałąź short-circuit **jest już zapinowana** testem *„Changing policy name should not bump revision"* ([storetest:497-536](server/channels/store/storetest/access_control_policy_store.go#L497)) biegnącym na PR.

---

## C9 — `AccessControlPolicyHistory` bez kolumny `Active`

**Ustalenie, które degraduje ten problem.** Grep po `AccessControlPolicyHistory` w `server/` daje 9 trafień w 3 plikach — dwie migracje i **jeden plik produkcyjny**. Dwa zapisy ([:260](server/channels/store/sqlstore/access_control_policy_store.go#L260), [:334](server/channels/store/sqlstore/access_control_policy_store.go#L334)) i **jeden odczyt** — `getHistoryT` [:542-545](server/channels/store/sqlstore/access_control_policy_store.go#L542), który ma **dokładnie jednego wywołującego** [:276](server/channels/store/sqlstore/access_control_policy_store.go#L276), służącego wyłącznie do odtworzenia `Revision+1`. **[E]** **Nikt nie czyta tej tabeli jako historii — to licznik rewizji, nie audyt.** Brak `Active` nie ma dziś konsumenta, który by ucierpiał.

**Intencjonalność: UNKNOWN** — pewność wysoka co do samego werdyktu. `10b1f4c5ac` ma puste body, migracja nie ma ani jednego komentarza SQL, żaden z 4 commitów dotykających tabeli nie wspomina o audycie ani o `Active`. Sygnały pośrednie są sprzeczne: historia jest zapisywana także na ścieżce `Delete` (pasuje do audytu), ale jedyny konsument liczy rewizje.

**Wykonalność: ŚREDNIA, technicznie wzorcowa** — migracja addytywna, konwencja dojrzała i weryfikowana w CI (`make new-migration` → `migrations.list` → `check-generated`). Rozcięcie: migracja (repo spójne, kod jej nie używa) → zapis → odczyt. Nowa migracja na `master` nie uruchamia `check-backport-migrations`.

---

## C10 — Nieatomowy zapis orkiestrowany przez klienta

**Obecny kształt.** `handleSubmit` ([policy_details.tsx:270-373](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L270)): maks. **5 żądań, 1 bezwarunkowe + 4 warunkowe** — `PUT` polityki [:277](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L277), `DELETE /unassign` [:315](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L315), `POST /assign` [:318](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L318), `PUT /activate` [:336](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L336), `POST /jobs` [:351](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L351). **[E]**

**Ustalenie ponad analizę ABAC — obsługa błędów kroków 2–5 jest MARTWA.** Kroki 2–5 wołają akcje zbudowane przez `bindClientFunc`, które **łapią wyjątek i zwracają `{error}`** ([helpers.ts:94-100](webapp/channels/src/packages/mattermost-redux/src/actions/helpers.ts#L94)) — nigdy nie rzucają. Otaczające je `try/catch` ([:322](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L322), [:337](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L337), [:354](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L354)) **nigdy się nie uruchomią**. Jedyny krok z realną obsługą to krok 1. **Jeśli krok 3 z 5 padnie, UI nawiguje do listy jak przy sukcesie.** **[E]** Wariant kanałowy ([channel_details.tsx:798-812](webapp/channels/src/components/admin_console/team_channel_settings/channel/details/channel_details.tsx#L798)) sprawdza zapis polityki **poprawnie**, ale ma ten sam defekt w gałęziach assign/unassign — i **świadomie toleruje** błąd aktywacji i jobu. Miejsc orkiestrujących ten sam multi-request zapis: **≥4**.

**Intencjonalność: UNKNOWN z przechyłem ku przypadkowej złożoności.** `git log --grep="atomic"` na plikach ABAC → jeden commit, dotyczy serwerowego `DeleteIfType`. `git log -S "rollback"` / `"transaction"` na `access_control.go`, `team_access_control.go` i całym `admin_console/access_control/` → **zero**. W `handleSubmit` nie ma komentarza „dlaczego" — są etykiety `--- Step 1 ---` … `--- Step 5 ---`. `git log -L 270,373` pokazuje **13 commitów** dotykających tego bloku między 2025-05-15 a 2026-05-29, każdy dokładający kolejny krok. Kształt wygląda na narosły, ale to wnioskowanie z historii, nie cytat.

**Wykonalność: RYZYKOWNA — największy blast radius spotyka najsłabszą osłonę.** Nowy endpoint → router → `openApiSync` (musi trafić do [api/v4/source/access_control.yaml](api/v4/source/access_control.yaml), inaczej `vet-api` zablokuje PR) → `client4.go` → `client4.ts` → typy → redux → komponent → tryb lokalny → specy Playwright. Osłony: R8 („write-path webappu praktycznie nietestowany"), a jedyna realna ochrona sekwencji to Playwright — **nie biegnący na PR i z nadpisywalnym statusem**.

---

## C11 — Połykane błędy i odrzucane polityki-dzieci

**Korekta: nie jedno miejsce, a trzy.** Błąd `ReconcilePolicyTeamScope` jest logowany i połykany w [:1111](server/channels/api4/access_control.go#L1111) (po assign), [:1193](server/channels/api4/access_control.go#L1193) (pre-flight unassign) i [:1215](server/channels/api4/access_control.go#L1215) (post-unassign). Wyniki `AssignAccessControlPolicyToChannels` odrzucane w [:1094](server/channels/api4/access_control.go#L1094) i [:1102](server/channels/api4/access_control.go#L1102). **[E]**

**To wzorzec lokalny, nie konwencja repo.** Grep po `best.effort|ignore error|log and continue|non-fatal` w całym `server/channels/api4/` daje **1 trafienie** ([content_flagging_report.go:99](server/channels/api4/content_flagging_report.go#L99)). Z 501 wystąpień `c.Logger.Warn(` w 63 plikach api4 dominuje „Error while writing response" — po udanej mutacji, przy serializacji. Nieliczne analogiczne przypadki istnieją, ale **bez komentarza deklarującego intencję**; tutaj komentarze są ([:1109-1110](server/channels/api4/access_control.go#L1109), [:1187-1191](server/channels/api4/access_control.go#L1187)). **[I]**

**Wykonalność: ŚREDNIA** — zmiana wąska w Go, ale **obserwowalna dla klienta**: dziś ASSIGN zawsze zwraca `{"status":"OK"}`; przepuszczenie błędu zmienia kod HTTP w scenariuszach, które dziś kończą się sukcesem. Krok pośredni: **zalogować i zliczyć**, nie zmieniając kodu odpowiedzi.

---

## C12 — Anonimowe struktury request-body

**Obecny kształt.** W [api4/access_control.go](server/channels/api4/access_control.go) jest **6 anonimowych struktur** — 5 request-body ([:383](server/channels/api4/access_control.go#L383), [:749](server/channels/api4/access_control.go#L749), [:1023](server/channels/api4/access_control.go#L1023), [:1126](server/channels/api4/access_control.go#L1126), [:1427](server/channels/api4/access_control.go#L1427)) plus 1 response. Struktury `assign`/`unassign` są **identyczne i zduplikowane**. **[E]**

**Konwencja istnieje i jest w połowie zastosowana.** [server/public/model/access_request.go](server/public/model/access_request.go) niesie `SubjectSearchOptions` [:166](server/public/model/access_request.go#L166), `QueryExpressionParams` [:270](server/public/model/access_request.go#L270), `PolicySimulationByUsersParams` [:626](server/public/model/access_request.go#L626) + 8 typów `PolicySimulation*`; w [access_policy.go](server/public/model/access_policy.go) mieszkają `AccessControlPolicySearch` i `AccessControlPolicyActiveUpdateRequest`. **7 z 12 endpointów ma nazwany typ, 5 nie ma.** **[E]** Koszt: `client4.go` duplikuje te kształty po raz trzeci, a jego `CheckExpression` **nie ma pola `teamId`**, które serwer akceptuje ([client4.go:8402](server/public/model/client4.go#L8402)). **[E]**

**Wykonalność: ŁATWA, najczystsze rozcięcie z całej listy.** Krok 1 — nowy typ w `model` + test `IsValid` (nikt go jeszcze nie używa, repo spójne); krok 2 — podmiana dekodowania; krok 3 opcjonalny — lustro TS. Osłona: `model/access_policy.go` ma **100% współzmienności ze swoim testem w oknie 6 miesięcy** — najlepsza dyscyplina w klastrze; testy `public/model` biegną na PR.

---

## C13 — Model bramkowania

**Ustalenie obalające tezę analizy ABAC.** W [api.go:358-418](server/channels/api4/api.go#L358) jest **61 wywołań `api.Init*()` i wszystkie są bezwarunkowe** — `InitAccessControlPolicy()` **nie jest wyjątkiem, jest normą**. Jedyna warunkowa rejestracja to `/manualtest` za `EnableTesting`. **[E]** Bramkowanie enterprise siedzi konsekwentnie **w handlerach**: helper `requireLicense(c)` ([handlers.go:237](server/channels/api4/handlers.go#L237)) użyty 22× w [group.go](server/channels/api4/group.go), plus 81 trafień wzorców licencyjnych w 22 plikach api4. **[E]**

**Realne odstępstwo jest inne, niż zapisano.** `EnableAttributeBasedAccessControl` ma **0 wystąpień w nietestowym `api4/access_control.go`**, choć w produkcji jest czytany w 8 miejscach — wszystkich na ścieżkach **enforcementu**, nie autoringu ([authorization.go:763](server/channels/app/authorization.go#L763), [channel.go:4578](server/channels/app/channel.go#L4578), [file.go:1610](server/channels/app/file.go#L1610), [post.go:1182](server/channels/app/post.go#L1182), [team.go:936](server/channels/app/team.go#L936), i in.). Istnieje kanoniczny predykat `attributeBasedAccessControlEnabled` ([access_control.go:28-32](server/channels/app/access_control.go#L28)) z komentarzem, że **celowo nie sprawdza licencji**. **[E]**

**Intencjonalność: PRZYPADKOWA ZŁOŻONOŚĆ — rozstrzygnięte.** `git log -S "InitAccessControlPolicy" -- server/channels/api4/api.go` zwraca **dokładnie jeden commit**: `a344b3225b`, którego diff wstawia `+ api.InitAccessControlPolicy()` w nieprzerwaną sekwencję `Init*()`, bez `if`, bez middleware. **Bramka przy rejestracji nigdy nie istniała.** Komentarz *„ABAC is gated at route registration"* dodał `d4471bece1` (2026-05-18, Pablo Vélez); przeszukanie całego body tego commitu pod kątem `gated|route registration|license|501` → **zero trafień**. **Komentarz był nieprawdziwy w chwili napisania; to nie regresja.**

**Wykonalność: ŚREDNIA, semantycznie ryzykowna** — dokręcenie bramki zmienia kody odpowiedzi 17 endpointów (dziś polityka da się zapisać przy `EnableAttributeBasedAccessControl=false`, a brak licencji daje 501 zamiast 403). Żaden test nie asertuje macierzy „flaga × licencja × endpoint".

---

## C14 — `localcachelayer` implementuje 3 z 12 metod

**Ustalenie degradujące.** Porównanie pokrycia (metody warstwy / metody interfejsu): channel **39/129**, user **15/90**, team **5/52**, post **9/56**, role 9/11, scheme 5/8, reaction 5/11, fileinfo 6/26, webhook 8/28. **Częściowa implementacja jest normą — ABAC (3/12 ≈ 25%) jest powyżej mediany**, wyraźnie ponad team (10%) i post (16%). **[E]** Mechanizmu wykrywającego brak inwalidacji **nie ma** — `timerlayer`/`retrylayer` są generowane ([layer_generators/main.go](server/channels/store/layer_generators/main.go)) i mają 100% pokrycia, a `localcachelayer` jest pisany ręcznie i osadza inner store, więc pominięta metoda kompiluje się i cicho przechodzi bez cache'u. Żaden z 20 analizatorów govet nie dotyczy cache'u. **[E]**

---

## C15 — Martwa powierzchnia API

**Ustalenie obalające tezę analizy ABAC.** `GET /{id}/activate` **nie jest martwy**. Helper `activatePolicy` ([e2e-tests/playwright/specs/functional/system_console/abac/support.ts:978-980](e2e-tests/playwright/specs/functional/system_console/abac/support.ts#L978)) woła dokładnie ten endpoint surowym `doFetch`, a jest importowany i wywoływany w **11 plikach spec (18, raport: 14, wywołań)** — m.in. `ldap_sync_add.spec.ts:60,126`, `advanced_policies.spec.ts:131,315,458`. **[E]** Trasa ma `Deprecated`, barierę CSRF i nagłówek `Deprecation` — **i jest żywą zależnością suite'u E2E ABAC**.

**Doprecyzowania.** Tryb lokalny rejestruje 14 z 17; brakujące trzy to `POST /cel/simulate_users`, `GET /{id}/activate`, `POST /access_control/decisions/actions/search`. Metod bez klienta Go jest **5, nie 4** (dochodzi `GET /{id}/activate`). **[E]**

**Intencjonalność: ŚWIADOME OGRANICZENIE (deprecacja) / UNKNOWN (plan usunięcia).** `ced9a56e39` (*„[MM-67126] Deprecate UpdateAccessControlPolicyActiveStatus API in favor of new one (#34940)"*, body puste). Uzasadnienie w kodzie, [api4/access_control.go:884-885](server/channels/api4/access_control.go#L884): *„…will be removed in a future release. Use PUT /api/v4/access_control/policies/activate instead, **which supports batch updates**."* Brak wersji docelowej i daty. Commit usunął konsumenta z webappu, ale `--stat` **nie obejmuje żadnego pliku z `e2e-tests/`** — E2E nie zostało zmigrowane i commit o nim nie wspomina.

---

## C16 — `Channel.PolicyEnforced` przez `EXISTS`

**Ustalenie: koszt jest dwa razy większy, niż zapisano.** Obok `PolicyEnforced` [:191](server/channels/store/sqlstore/channel_store.go#L191) stoi **drugie** skorelowane podzapytanie `PolicyIsActive` [:192](server/channels/store/sqlstore/channel_store.go#L192). `channelSliceColumns` ma **29 (raport: 28) wywołań, 27 (raport: 26) z `isSelect=true`**, a bazowe `tableSelectQuery` [:546](server/channels/store/sqlstore/channel_store.go#L546) jest reużywane 9 (raport: 11) razy. Gorące ścieżki — **wszystkie**: `GetChannels`, `GetChannelsByUser`, `GetMany`, `GetForPost`, `getByName(s)`, `Autocomplete*`, `SearchInTeam`, `SearchMore`, eksporty. Ten sam kształt zduplikowano dla zespołów ([team_store.go:240](server/channels/store/sqlstore/team_store.go#L240)). **[E]**

**Wykonalność: NAJGORSZY stosunek zysku do ryzyka.** Denormalizacja wprowadza inwariant do utrzymania w **≥5 miejscach zapisu** — w tym w dwóch omijających warstwę EE (C6) i w `ReconcilePolicyTeamScope`, którego błędy są połykane (C11). Wg konwencji z [README migracji](server/channels/db/migrations/README.md) wymaga trzech releasów. **A nie istnieje żaden pomiar dowodzący, że problem jest realny** — brak w repo benchmarków i `EXPLAIN`-ów. **[U]**

---

# 3. Osłony CI — stan faktyczny (rozstrzygnięcie U2 z analizy ABAC)

Skrót najważniejszych ustaleń; pełna tabela w raporcie sub-agenta wykonalności.

| Mechanizm | Konfiguracja | Na PR? | Co realnie chroni |
|---|---|---|---|
| Testy Go (Postgres, 4 shardy) | [server-ci.yml:179-204](.github/workflows/server-ci.yml#L179) → `make test-server` | **TAK** | `channels/app`, `channels/api4`, **storetesty przeciw prawdziwemu Postgresowi** |
| **`check-generated`** | [server-ci.yml:82-121](.github/workflows/server-ci.yml#L82), fail na `git status --porcelain` | **TAK** | „wygeneruj i porównaj diff": `store-layers`, `store-mocks`, `einterfaces-mocks`, `pluginapi`, `migrations.list` |
| `check-style` → `make vet` (govet) | [server/Makefile:966-981](server/Makefile#L966) | **TAK** | **11 z 24** analizatorów włączonych; **11 zarejestrowanych, ale niewłączonych = wolne sloty** |
| `vet-api` / `openApiSync` | [server-ci.yml:159](.github/workflows/server-ci.yml#L159) | **TAK, blokująco** | **tylko ścieżki + metody HTTP**; schematów, pól ani typów nie porównuje |
| Build tag `enterprise` | [server/Makefile:62-79](server/Makefile#L62) | **NIE** | katalog `../../enterprise` nie istnieje w CI → `external_imports.go` nigdy się nie kompiluje |
| Storetest ABAC | [storetest/access_control_policy_store.go:24-41](server/channels/store/storetest/access_control_policy_store.go#L24) | **TAK** | 17 pod-suite'ów — **najmocniejsza realna osłona klastra** |
| Testy migracji | brak plików `server/channels/db/**/*_test.go` | — | pokrycie tylko pośrednie (storetest buduje bazę migracjami) |
| Webapp: `check-types`, `check-lint`, jest | [webapp-ci.yml:22,83,171](.github/workflows/webapp-ci.yml#L22) | **TAK** | `tsc` + eslint (w tym **własny plugin z 3 regułami** — gotowy nośnik) |
| E2E Tests Check | [e2e-tests-check.yml:3-10](.github/workflows/e2e-tests-check.yml#L3) | **TAK** | wyłącznie lint + typecheck specy — **nie uruchamia żadnego testu** |
| **Playwright ABAC (144 testy)** | [e2e-tests-ci.yml:15](.github/workflows/e2e-tests-ci.yml#L15) — `workflow_dispatch` ONLY | **NIE** | uruchamiany zewnętrznie po zbudowaniu obrazu enterprise; status **nadpisywalny ręcznie** ([override-status.yml:46-52](.github/workflows/e2e-tests-override-status.yml#L46)) |
| Tools CI | [tools-ci.yml:3-11](.github/workflows/tools-ci.yml#L3) | **TAK** | nowy analizator govet dostaje własne CI **za darmo** |

**Trzy wnioski.**

1. **Dyscyplina generatorów jest egzekwowana twardo i blokująco** — to domyka §6.4 analizy ABAC: nie chodzi o kulturę zespołu, tylko o to, że CI wymusza. Każdy nowy artefakt wpięty w `make generated` dziedziczy tę egzekucję.
2. **Kontrakt REST jest pilnowany na poziomie ścieżek, nie schematów** — dlatego 16 ścieżek ABAC istnieje w spec-u, mimo że 77% commitów modelowych spec pomija, a schemat `AccessControlPolicy` jest fikcją. Luka narzędziowa ma jedną, precyzyjną krawędź.
3. **Luki R1–R4 nie mają ochrony w pętli PR.** Jedyne miejsce, gdzie `mergeStoredPolicyExpressions` spotyka prawdziwy silnik CEL, to spec Playwright w pipeline, który nie biegnie na `pull_request` i którego status da się nadpisać.

---

# 4. Korekty wobec analizy wejściowej

| Twierdzenie analizy ABAC | Werdykt | Dowód |
|---|---|---|
| `GET /{id}/activate` „nie ma klienta po żadnej stronie" | ❌ **OBALONE** | 18 (raport: 14) wywołań w 11 plikach spec przez [support.ts:978](e2e-tests/playwright/specs/functional/system_console/abac/support.ts#L978) |
| Bezwarunkowa rejestracja `InitAccessControlPolicy()` jako anomalia | ❌ **OBALONE** | **61/61** wywołań `Init*()` w [api.go:358-418](server/channels/api4/api.go#L358) jest bezwarunkowych |
| `localcachelayer` 3/12 jako dług ABAC | ❌ **ZDEGRADOWANE** | norma warstwy: team 5/52, post 9/56, user 15/90 — ABAC jest **powyżej mediany** |
| „Czy `mergeFromStore=false` to decyzja, czy przeoczenie?" (Open Q #7) | ✅ **ROZSTRZYGNIĘTE** | decyzja — doc-komentarz [:164-169](server/channels/app/plugin_access_control.go#L164) + komentarz guardów [:409-412](server/channels/app/access_control.go#L409) + body commitu `cfab0366` |
| U2 — „które linie CI uruchamiają `abac/`" | ✅ **ROZSTRZYGNIĘTE** | `workflow_dispatch` only; nie na PR; status nadpisywalny |
| Dwa zapisy omijające warstwę EE | ⚠️ **DOPRECYZOWANE → 4** | +`Delete` ×2; `SetActiveStatusMultiple` **nie jest** obejściem (guard `acs == nil` na [:1913](server/channels/app/access_control.go#L1913)) |
| Błędy `ReconcilePolicyTeamScope` połykane w 1 miejscu | ⚠️ **DOPRECYZOWANE → 3** | [:1111](server/channels/api4/access_control.go#L1111), [:1193](server/channels/api4/access_control.go#L1193), [:1215](server/channels/api4/access_control.go#L1215) |
| 39 guardów `acs == nil`, 21 w `access_control.go` | ⚠️ **DOPRECYZOWANE → 41 / 22** | +`channel_join_request.go:331`, +`team_directory_visibility.go:64` |
| Brak 4 metod ABAC w `client4.go` | ⚠️ **DOPRECYZOWANE → 5** | dochodzi `GET /{id}/activate` |
| `PolicyEnforced` = jedno podzapytanie | ⚠️ **DOPRECYZOWANE → para** | +`PolicyIsActive` [:192](server/channels/store/sqlstore/channel_store.go#L192), na 26 ścieżkach odczytu |
| „Migracja omija EE, bo biegnie za wcześnie" (hipoteza) | ❌ **OBALONE** | `NewChannels` w [server.go:305](server/channels/app/server.go#L305), `doAppMigrations` w [:626](server/channels/app/server.go#L626) — serwis jest już dostępny |
| Nieatomowy zapis = brak transakcji | ⚠️ **ZAOSTRZONE** | obsługa błędów kroków 2–5 jest **martwa** (`bindClientFunc` nie rzuca, `try/catch` nigdy nie odpali) |

---

# 5. Refactor opportunities

Ranking ocenia **koszt długu przeciw kosztowi zmiany**, z wagą dla: istniejącego nośnika, egzekwującego mechanizmu CI, addytywności i odwracalności. To propozycja do osobnej sesji planowania.

## 🥇 1. Uwidocznić szew enterprise (C1)

**Obecny → docelowy kształt.** Dziś: 20-metodowy interfejs PAP/PDP, którego jedyna implementacja żyje poza tym repozytorium, a zmiana jego sygnatury **kompiluje się, przechodzi wszystkie testy i regeneruje mocki czysto**. Docelowo: **sygnatura interfejsu jako wygenerowany, zacommitowany artefakt** wpięty w `make generated` — czyli krawędź runtime zamieniona na krawędź widoczną w diffie PR-a. Docelowym kształtem **nie jest** usunięcie szwu ani null-object: podział OSS/EE to udokumentowana decyzja licencyjna ([server/enterprise/README.md:3](server/enterprise/README.md#L3)), a rozsiany guard `acs == nil` jest normą repo (`.Metrics` 66×, `.License()` 53×) — jego zmiana w samym ABAC oznaczałaby rozjazd z resztą.

**Dlaczego pierwsze miejsce.** Koszt długu jest najwyższy z całej listy i **jakościowo inny**: to jedyny kandydat, dla którego CI **strukturalnie nie może** wykryć regresji — `BUILD_TAGS += enterprise` włącza się tylko przy obecnym katalogu `../../enterprise` ([server/Makefile:62-79](server/Makefile#L62)), którego CI nie klonuje, a `einterfaces-mocks` z `all: true` **aktywnie maskuje** rozjazd. Koszt zmiany jest przy tym najniższy w stosunku do wartości: artefakt tekstowy + target w Makefile, **zero zmian zachowania**, pełna odwracalność (usunięcie pliku i wpisu). I trafia w gotowy, **blokujący na PR** mechanizm `check-generated`, którego skuteczność jest w tym repo empirycznie potwierdzona.

**Blast radius zmiany:** ~2 pliki (nowy artefakt + `server/Makefile`). **Blast radius, który chroni:** 19 pakietów Go / 65 plików importujących `einterfaces` + prywatny moduł `mattermost/enterprise/access_control` + 28 funkcji `Register*Interface`.

**Szkic ścieżki inkrementalnej.** (1) Generator sygnatury obu interfejsów → plik tekstowy w repo. (2) Wpięcie w `make generated`; od tego momentu każda zmiana szwu produkuje diff, a jego brak wywala `check-generated`. (3) Opcjonalnie: analizator govet w jednym z 11 wolnych slotów, sprawdzający współbieżność sygnatury z mockami. Każdy krok pozostawia repo spójne; do kroku 2 nic nie blokuje.

**Pierwszy krok-prerekwizyt.** Wygenerować sygnaturę **jednorazowo, poza CI**, i sprawdzić, czy jest stabilna między uruchomieniami (kolejność metod, formatowanie) — bez tego artefakt będzie fałszywie migotał i zostanie wyłączony w pierwszym tygodniu.

**Trade-off, który trzeba nazwać.** To zmiana **widoczności, nie bezpieczeństwa**. Nie sprawi, że zmiana szwu przestanie psuć prywatne repo — sprawi, że recenzent PR-a zobaczy, że szew został ruszony. Prawdziwa weryfikacja wymagałaby budowania z tagiem `enterprise` w CI, co jest poza zasięgiem tego repozytorium **[U1]**.

## 🥈 2. Rozszerzyć `openApiSync` o porównanie schematów (C3)

**Obecny → docelowy kształt.** Dziś: cztery ręcznie utrzymywane kopie kontraktu, z których opublikowana specyfikacja OpenAPI dla `AccessControlPolicy` jest fikcją (6 pól-widm, brak 10 realnych, zero schematu reguły), a **77% commitów modelowych ABAC nie dotyka spec-u**. Docelowo: **checker schematów w istniejącym analizatorze `openApiSync`**, uruchamiany w trybie allow-list — dług zamrożony, nowe rozjazdy blokowane na PR.

**Dlaczego drugie miejsce.** Koszt długu jest repo-wide, nie ABAC-owy: to ryzyko 🔴1 z [repo-map.md](context/map/repo-map.md), a analiza ABAC dostarczyła mu **zmierzonego mechanizmu**. Koszt zmiany jest umiarkowany **wyłącznie dlatego**, że nośnik już istnieje i jest mocniejszy, niż zakładano: [openApiSync.go:140-181](tools/mattermost-govet/openApiSync/openApiSync.go#L140) już parsuje spec przez `libopenapi`, już buduje model v3 i **już blokuje PR**. Brakuje mu wyłącznie drugiej pętli porównania. Do tego `tools-ci.yml` daje nowemu analizatorowi własne CI za darmo.

Za tym kandydatem stoi też najmocniejszy dowód intencjonalny w całym badaniu: tymczasowość została **zadeklarowana wprost w 2023 roku** (*„For now… until the API documentation is back in sync"*, `0577a5aaa2`) i trwa ponad dwa lata. To nie jest decyzja nośna — to odroczenie, które się zastało.

**Blast radius zmiany:** `tools/mattermost-govet/openApiSync/` + plik baseline + wpis w `server/Makefile`. **Blast radius, który chroni:** `public/model` (140 pakietów / 1428 plików), `platform/types` (44 importerów samego `access_control`), `platform/client`, `api/v4/source`, plus 58 plików Playwright linkujących przez `file:`.

**Szkic ścieżki inkrementalnej.** (1) Zbudować baseline rozjazdu jako plik maszynowo czytelny. (2) Rozszerzyć analizator o porównanie `components.schemas` z tagami `json` struktur `model`, w trybie raportującym z allow-listą = baseline. (3) Włączyć blokująco dla **nowych** pól. (4) Osobno i później: oś TS (reguła w istniejącym pluginie ESLint) oraz oś `client4.go` ↔ router.

**Pierwszy krok-prerekwizyt.** **Rozstrzygnąć U-CI-1**: czy `make vet-api` faktycznie przechodzi na HEAD. [server/Makefile:1003](server/Makefile#L1003) mówi „currently not passing", a job nie ma `continue-on-error`. Jeśli jest de facto czerwony i tolerowany, cała podstawa tej kandydatury się sypie — bo nośnik okaże się atrapą. To pytanie za jedno uruchomienie `go vet`.

**Trade-off.** Zakres wykracza poza ABAC i poza właścicieli tego obszaru. Wymaga też decyzji, czy checker startuje z długiem w allow-liście (szybko, ale dług zostaje) czy z jego spłatą (wolno, ale czysto). Rekomendacja z dowodów: allow-lista — bo 4 na 5 zmian modelu historycznie omija spec, więc **zatrzymanie wzrostu jest warte więcej niż jednorazowa spłata**.

## 🥉 3. Przenieść serializację jsonb na konwencję repo (C2)

**Obecny → docelowy kształt.** Dziś: `accessControlPolicyV0_1` ([sqlstore:26-32](server/channels/store/sqlstore/access_control_policy_store.go#L26)) — ręcznie utrzymywana struktura-cień, przez którą nowe pole modelu **ginie po cichu**: bez błędu kompilacji, bez migracji, bez sygnału z testów. Docelowo: **`sql.Scanner`/`driver.Valuer` na typie modelowym**, czyli konwencja, którą repo stosuje wszędzie indziej — [model/utils.go:100-207](server/public/model/utils.go#L100), [model/channel.go:59,72](server/public/model/channel.go#L59). Para `fromModel`/`toModel` występuje w całym `sqlstore/` **tylko tutaj**.

**Dlaczego trzecie miejsce.** To jedyny kandydat z **bezpośrednim dowodem przypadkowej złożoności**: komentarz *„This needs to be updated with the new version of the policy. with the new name as this only supports v0.1"* to TODO z kwietnia 2025, którego nie wykonano — a commit `c66bb0ecdb` przepisał `toModel()` pod v0.3 i **zostawił nazwę V0_1**. Koszt zmiany jest niski (jeden plik, 20 odwołań), osłona **już biegnie na PR przeciw prawdziwemu Postgresowi** (`ScopeRoundtrip`), a warstwa store jest jednocześnie tym ogniwem, które analiza ABAC wskazała jako najsłabsze pod względem współzmienności testów (64%).

**Blast radius:** [access_control_policy_store.go](server/channels/store/sqlstore/access_control_policy_store.go) + [access_policy.go](server/public/model/access_policy.go) + storetest. Bez migracji, bez warstw generowanych, bez dotykania webappu.

**Szkic ścieżki inkrementalnej.** (1) Test wyliczeniowy round-tripu wszystkich pól niekolumnowych — czysty pomiar. (2) Przeniesienie serializacji na `Scanner`/`Valuer` po stronie modelu, z zachowaniem identycznego JSON-a w kolumnie (test z kroku 1 pilnuje). (3) Usunięcie struktury-cienia. Po każdym kroku repo jest spójne, a format danych w bazie niezmieniony — co czyni całość odwracalną.

**Pierwszy krok-prerekwizyt.** Test refleksyjny: każde pole `model.AccessControlPolicy` niebędące kolumną musi przetrwać `Save`→`Get`. Dopisanie do istniejącej suity to jedna linia; **zamienia cichą stratę w czerwony CI jeszcze przed jakąkolwiek zmianą struktury**.

**Trade-off.** Zysk jest prewencyjny, nie naprawczy — dziś nic nie jest zepsute. Wartość zależy od tego, jak długo `Data jsonb` pozostanie miejscem ewolucji schematu; analiza ABAC pokazuje, że od migracji `000194` (czerwiec 2026) **cały rozwój funkcjonalny idzie właśnie tamtędy**, więc trend jest rosnący.

---

## Runner-up, celowo poza podium: klaster tłumienia błędów (C10 + C11)

Zasługuje na wyróżnienie, bo ma **najwyższy koszt widoczny dla użytkownika**, a jednocześnie **nie jest dobrym pierwszym refaktorem**.

Trzy niezależne mechanizmy składają się na jeden efekt: serwer połyka błąd `ReconcilePolicyTeamScope` w trzech miejscach, `bindClientFunc` zamienia wyjątek w cichy `{error}`, a klienckie `try/catch` w krokach 2–5 są **kodem martwym**. Rezultat: **częściowo zapisana polityka kontroli dostępu wygląda dla administratora dokładnie tak samo jak zapisana w całości**. W funkcji bezpieczeństwa to poważniejsze niż wszystko powyżej.

Poza podium z trzech powodów: (1) najtańsza część naprawy — martwe `try/catch` — jest **defektem, nie refaktorem**, więc nie należy do tej listy; (2) strukturalna część (endpoint zbiorczy) ma **największy blast radius z całej listy** i spotyka **najsłabszą osłonę** (R8 + Playwright poza pętlą PR); (3) semantyka błędów jest **rozjechana między ≥4 miejscami orkiestracji**, więc ujednolicenie wymaga najpierw decyzji produktowej, które błędy wolno tolerować — a [channel_details.tsx:805-812](webapp/channels/src/components/admin_console/team_channel_settings/channel/details/channel_details.tsx#L805) pokazuje, że część z nich jest tolerowana **świadomie**.

**To jest punkt, w którym trzeba się zatrzymać.** Pytanie „co znaczy nieudany częściowy zapis polityki i co administrator ma wtedy zobaczyć" jest pytaniem o **pojęcia biznesowe**, nie o strukturę kodu. Zgodnie z granicą tej analizy: nazywam to i przekazuję do osobnej, późniejszej sesji.

---

# 6. Kandydaci rozważeni i odrzuceni

| # | Dlaczego odrzucony |
|---|---|
| **C5** | **Udokumentowana decyzja projektowa, nie dług.** Trzy zbieżne dowody (doc-komentarz, komentarz guardów, body commitu `cfab0366`) pokazują, że `mergeFromStore=false` to model „trusted plugin" wprowadzony celowo. Ujednolicenie skasowałoby feature. Pozostaje **luka testowa**, nie strukturalna: różnica nie jest zapinowana asercją. |
| **C6** | **4 z 5 mutacji ma jawne, merytoryczne uzasadnienie w kodzie** — w tym najlepiej udokumentowaną decyzję w klastrze (fallback delete chroniący przed osieroconym wierszem przy `acs == nil`). Zostaje jeden UNKNOWN (migracja), a popularne wytłumaczenie „biegnie przed inicjalizacją" jest **obalone**. Wartość: co najwyżej checker z allow-listą. |
| **C7** | Jedyne miejsce, gdzie milczenie samo coś sugeruje (`UpdateOptimistically` dostępne 2 lata wcześniej, nieużyte bez powodu) — **ale wzorzec niemutowalnych rewizji jest udokumentowany i sensowny**, a wprowadzenie kontroli optymistycznej wymaga jednoczesnej zmiany kontraktu klienta, plugin API i `ReconcilePolicyTeamScope`. **Brak kroku pośredniego, po którym repo jest spójne, a zachowanie niezmienione.** Wyścig S2 nie ma żadnego testu ani obserwacji produkcyjnej. |
| **C8** | Realny i tani, ale **konsekwencja jest szersza niż zmiana**: włączenie `Props` do porównania wygeneruje nowe rewizje historii przy operacjach, które dziś ich nie generują. Wymaga decyzji „czy `Props` jest częścią tożsamości rewizji" — a skoro historia jest tylko licznikiem (C9), pytanie jest przedwczesne. |
| **C9** | **Waga problemu upadła po pomiarze**: tabela historii ma jednego czytelnika, `getHistoryT`, używanego wyłącznie do wyliczenia `Revision+1`. To licznik, nie audyt. Brak `Active` nie ma dziś konsumenta. Naprawa byłaby dodaniem kolumny do tabeli, której nikt nie czyta. |
| **C12** | Najczystsze rozcięcie z całej listy i idzie w stronę istniejącej konwencji (7/12 endpointów już ją stosuje) — **ale koszt długu jest niski**: dwie zduplikowane struktury i brak `IsValid()`. Dobry kandydat na „przy okazji", nie na osobny refaktor. |
| **C13** | **Teza analizy obalona**: bezwarunkowa rejestracja to norma 61/61, a bramkowanie w handlerach jest utrwaloną konwencją z gotowym helperem. Realne odstępstwo (brak bramki configowej w autoringu) jest wąskie, a dokręcenie **zmienia kody HTTP 17 endpointów** bez żadnego testu macierzy „flaga × licencja". Prerekwizyt (test tabelaryczny) jest wart wykonania osobno; refaktor — nie teraz. |
| **C14** | **Zdegradowany pomiarem**: częściowa implementacja localcache to norma warstwy, a ABAC (3/12) jest powyżej mediany. Problem realny (brak sygnału przy dodaniu metody mutującej) dotyczy **całej warstwy**, nie ABAC — i jako taki należy do innej analizy. |
| **C15** | **Teza obalona**: `GET /{id}/activate` ma 14 żywych wywołań w E2E. Zostaje część addytywna (5 brakujących metod `client4.go`), która **nie jest refaktorem**, tylko prerekwizytem testowalności — i jako taka powinna wejść przy okazji dowolnej pracy nad ABAC. |
| **C16** | **Najgorszy stosunek zysku do ryzyka i jedyny kandydat bez metryki.** Inwariant do utrzymania w ≥5 miejscach zapisu, w tym dwóch omijających EE i jednym z połykanymi błędami; trzy release'y wg konwencji migracji; a **brak jakiegokolwiek dowodu, że `EXISTS` jest realnym problemem**. Pierwszym krokiem jest `EXPLAIN ANALYZE`, nie kod. |
| **C4** | Nie odrzucony, tylko **zdegradowany do „przy okazji"**: najmniejszy blast radius w całej liście (3 zapisy + 5 odczytów), gotowy nośnik (plugin ESLint), osłony sprawne i biegnące na PR — ale koszt długu jest kosmetyczny. Idealny materiał na rozgrzewkę, nie na priorytet. |

---

# 7. Code References

**Nośniki, na których opiera się ranking:**
- [.github/workflows/server-ci.yml:82-121](.github/workflows/server-ci.yml#L82) — `check-generated`, blokujący diff-gate po `make generated`
- [server/Makefile:455](server/Makefile#L455) — definicja celu `generated`
- [server/Makefile:62-79](server/Makefile#L62) — warunkowe włączenie `BUILD_TAGS += enterprise`
- [tools/mattermost-govet/main.go:33-59](tools/mattermost-govet/main.go#L33) — rejestr 24 analizatorów, 11 niewłączonych
- [tools/mattermost-govet/openApiSync/openApiSync.go:140-181](tools/mattermost-govet/openApiSync/openApiSync.go#L140) — porównanie route'ów ze spec-em
- [server/channels/store/storetest/access_control_policy_store.go:1392-1497](server/channels/store/storetest/access_control_policy_store.go#L1392) — `ScopeRoundtrip`, wzorzec testu round-tripu
- [server/public/model/utils.go:100-207](server/public/model/utils.go#L100) — kanoniczna konwencja `Scanner`/`Valuer`
- [webapp/platform/eslint-plugin/rules/](webapp/platform/eslint-plugin/rules/index.js) — własny plugin ESLint z testami reguł

**Punkty długu:**
- [server/einterfaces/pap.go](server/einterfaces/pap.go) / [pdp.go](server/einterfaces/pdp.go) — szew bez implementacji
- [server/channels/store/sqlstore/access_control_policy_store.go:26-32](server/channels/store/sqlstore/access_control_policy_store.go#L26) — struktura-cień
- [server/channels/app/plugin_access_control.go:164-169](server/channels/app/plugin_access_control.go#L164) — udokumentowany model „trusted plugin"
- [server/channels/app/team_access_control.go:275-279](server/channels/app/team_access_control.go#L275) — uzasadnione obejście EE
- [server/channels/app/channel.go:4600-4618](server/channels/app/channel.go#L4600) — najlepiej udokumentowana decyzja klastra
- [webapp/channels/src/packages/mattermost-redux/src/actions/helpers.ts:94-100](webapp/channels/src/packages/mattermost-redux/src/actions/helpers.ts#L94) — `bindClientFunc` nie rzuca; źródło martwych `try/catch`

---

# 8. Architecture Insights

1. **Repo ma konwencję dla trzech z badanych problemów, a ABAC żadnej z nich nie używa.** Serializacja jsonb → `Scanner`/`Valuer` (C2). Typowany dostęp do worka `Props` → akcesory + stałe kluczy jak w `Post` (C4). Kontrola optymistyczna → `UpdateOptimistically` / predykat `UpdateAt` (C7). Wszystkie trzy są dostępne i przetestowane w sąsiednich store'ach. To nie jest brak wiedzy o wzorcach — to **młody kod, który nie zdążył się z nimi zetknąć**.

2. **Rozgałęzianie zamiast parametryzacji jest tu powtarzalnym odruchem**: dwie ścieżki zapisu, dwie ścieżki aktywacji, pięć skopiowanych walidatorów wersji, cztery mutacje omijające EE. Jedyne miejsce, gdzie sięgnięto po parametr zamiast kopii — `mergeFromStore bool` — jest jednocześnie **jedyną faktycznie współdzieloną logiką obu ścieżek zapisu**, i jedyną decyzją rozstrzygniętą dowodowo z wysoką pewnością.

3. **Repozytorium ma dwie ery dokumentowania decyzji, i to wyjaśnia rozkład werdyktów lepiej niż cokolwiek innego.** Era 2025-04 → 2026-06: squash-merge z **systematycznie pustym body**, tytuł = numer ticketu. Era od 2026-08 (z `Co-authored-by: Cursor Agent`): commity z wielosetwyrazowymi body, jawnie nazwanymi modelami zagrożeń i doc-komentarzami tłumaczącymi „dlaczego". **C5 dało się rozstrzygnąć z wysoką pewnością wyłącznie dlatego, że wpadło w tę drugą erę.** To nie różnica jakości kodu — to różnica **dostępności intencji**. Paradoksalnie: obserwacja 🟡4 z mapy repo („rosnący udział agenta AI bez śladu autorstwa") ma tu odwrotny znak — commity z tej ery są **znacznie lepiej udokumentowane**.

4. **Model licencyjny jest generatorem kształtu, nie wypadkiem.** Wszystkie obejścia z C6 mają wspólną, spisaną przyczynę: operacja musi zadziałać także wtedy, gdy silnika ABAC nie ma, bo inaczej zostaje osierocony wiersz. Ta sama siła produkuje 41 powtórzeń guarda `acs == nil` i 28 funkcji `Register*Interface`. Kształt jest konsekwentny, powtarzalny i **nie da się go „naprawić" lokalnie w ABAC**.

5. **Zero ADR-ów w całym repozytorium.** Wyszukanie `adr`, `decision`, `rfc`, `design`, `architecture` zwraca wyłącznie trzy dokumenty topologii wdrożeniowej i obrazki PNG. Cała intencjonalność, którą udało się ustalić, siedzi w **komentarzach w kodzie** — i wszędzie tam, gdzie komentarza nie było, werdykt kończy się na UNKNOWN. To najtańsza możliwa obserwacja procesowa tego badania.

6. **Plan fazowy istnieje, ale nie obejmuje badanego długu.** `git log --grep="MM-61756"` zwraca wyłącznie „Phase 1" — Phase 2 tego epiku nigdy nie powstał. Fazowość realnie działa w innym, późniejszym epiku (atrybuty natywne, Phase 2/4/5/6). W **żadnym** z 81 commitów z „ABAC" w message nie ma zapowiedzi naprawy nazwy `V0_1`, generatora kontraktu ani bramki. **Nie ma dowodu, że którykolwiek z badanych długów jest tymczasowy z założenia** — poza `vet-api`, gdzie tymczasowość zadeklarowano wprost w 2023 i trwa do dziś.

---

# 9. Historical Context (from prior changes)

- [context/changes/abac-work-in-progress/research.md](context/changes/abac-work-in-progress/research.md) — jedyne wejście merytoryczne; §2 (dług), §2.4 (blast radius), §9 (weryfikacja ast-grep) użyte jako dane, nie mierzone ponownie. Cztery jej twierdzenia obalone, siedem doprecyzowanych, dwa UNKNOWN rozstrzygnięte (§4).
- [context/map/repo-map.md](context/map/repo-map.md) — ryzyko 🔴1 („rozjazd kontraktu REST — luka narzędziowa, nie luka wiedzy") jest bezpośrednią podstawą kandydatury nr 2. Ryzyko 🟠3 („krawędzie runtime niewidoczne dla CI") — podstawą kandydatury nr 1. Obserwacja 🟡4 o `cursor[bot]` została **częściowo odwrócona** przez §8.3.
- [context/map/artifact-2-structure.md](context/map/artifact-2-structure.md) §3b — „kontrakt pluginów **jest** maszynowo egzekwowany" jako dowód, że ta sama organizacja rozwiązała ten sam problem generatorem. Teraz wiadomo więcej: egzekwuje go **blokujący `check-generated`**, nie sam generator.
- `context/foundation/lessons.md` — **nie istnieje**; brak zapisanych priorów zespołowych do uwzględnienia.

---

# 10. Open Questions

Uszeregowane wg wpływu na planowanie.

1. **[U-CI-1] Czy `make vet-api` faktycznie przechodzi na HEAD?** [server/Makefile:1003](server/Makefile#L1003) mówi „currently not passing", a job [check-mattermost-vet-api](.github/workflows/server-ci.yml#L159) nie ma `continue-on-error`. **Rozstrzyga podstawę kandydatury nr 2** — jeśli job jest czerwony i tolerowany, nośnik jest atrapą. Koszt sprawdzenia: jedno uruchomienie `go vet` (w tym środowisku `go` nie było w PATH).
2. **[U-CI-2] Które statusy są *required* w branch protection?** Konfiguracja jest po stronie GitHuba, nie w repo. Bez tego nie wiadomo, czy czerwony `check-generated` faktycznie **blokuje merge**, czy tylko go zniechęca — a na tym stoi cała kandydatura nr 1.
3. **[U-CI-3/4] Czy pipeline E2E realnie biegnie dla każdego PR-a i jak często status jest nadpisywany?** Przesądza, czy 144 testy ABAC to osłona twarda czy dekoracyjna — i tym samym, jak ryzykowna jest jakakolwiek praca w tym obszarze.
4. **Kto zapisuje `Version = v0.4`?** Wszystkie 4 wystąpienia stałej w kodzie non-test to odczyty; żadna ścieżka zapisu jej nie ustawia. Jedyna hipoteza — enterprise `acs.SavePolicy` — jest niesprawdzalna z tego repo. Dopóki to trwa, **piąty walidator pilnuje kształtu, którego nikt nie produkuje**.
5. **Co robi warstwa enterprise wewnątrz `SavePolicy`** — kolejność normalizacji vs. persystencji, ewentualna własna kontrola współbieżności, ewentualna druga struktura dekodująca `Data`. Wpływa na C1, C2, C5 i C7 naraz. Nierozstrzygalne z tego repozytorium.
6. **Czy `GET /{id}/activate` ma konsumentów spoza repo** (wtyczki, klient mobilny). Wewnątrz repo wiadomo już, że ma 18 (raport: 14) wywołań w E2E — więc usunięcie wymagałoby najpierw migracji suite'u, o której commit deprecating nie wspomina.
7. **Realny koszt pary podzapytań `EXISTS`** na produkcyjnym wolumenie. Bez tej liczby C16 pozostaje optymalizacją bez metryki.
8. **Czy tabela `AccessControlPolicyHistory` miała być audytem?** Dziś jest licznikiem rewizji. Odpowiedź przesądza, czy C9 to przeoczenie do naprawy, czy poprawnie zwymiarowany mechanizm o mylącej nazwie.

---

## Weryfikacja twierdzeń (ast-grep)

Weryfikacja objęła wszystkie twierdzenia STRUKTURALNE, na których stoi ranking: liczby metod, pary lustrzanych typów, liczność call-site'ów i par „nadpisuje X, ale nie Y". Metoda: `ast-grep 0.45.3` do wzorców składniowych Go (sygnatury funkcji, wywołania metod), klasyczny grep/`Select-String` do potwierdzenia zer i wzorców tekstowych (komentarze, literały). Środowisko nie miało `go` w PATH, więc żadna weryfikacja nie wymagała kompilacji — wyłącznie analiza statyczna tekstu/AST.

Zgodnie z poleceniem: sekcja **„## 5. Refactor opportunities" (ranking) oraz werdykty intencjonalności NIE zostały zmienione**. Rozbieżności znalezione wewnątrz tekstu tej sekcji (np. „.Metrics 66×/.License() 53×" w kandydaturze #1, „20 odwołań"/„para występuje tylko tutaj" w kandydaturze #3) są odnotowane **wyłącznie tutaj**, z adnotacją „do decyzji na etapie planowania" tam, gdzie mogłyby naruszyć pozycję kandydata.

| Twierdzenie | Werdykt | Dowód (plik:linia) | Metoda (wzorzec/reguła) |
|---|---|---|---|
| 28 funkcji `Register*Interface` (21 w `app/enterprise.go` + 7 w `platform/enterprise.go`) | ✅ POTWIERDZONE | [app/enterprise.go:13-133](server/channels/app/enterprise.go#L13) (21), [platform/enterprise.go:13-49](server/channels/app/platform/enterprise.go#L13) (7) | ast-grep/regex `func Register\w*Interface\(` |
| 41 guardów `acs == nil` w `server/` | ✅ POTWIERDZONE | 11 plików, m.in. [access_control.go](server/channels/app/access_control.go), [plugin_access_control.go](server/channels/app/plugin_access_control.go), [channel.go:4626,4687](server/channels/app/channel.go#L4626) | regex `acs == nil` |
| 22 guardy `acs == nil` w `access_control.go` | ⚠️ DOPRECYZOWANE → **21** (raport: 22) | [access_control.go](server/channels/app/access_control.go) — 21 linii `if acs == nil {` (111…2589) | regex `if acs == nil \{` |
| `.Metrics != nil` 66×, `.License() != nil` 53×, `.DataRetention() != nil` 15×, `.Cluster != nil` 11× jako dowód „guard to norma repo" | ⚠️ DOPRECYZOWANE (rozbieżne) | `.Metrics()` nil-checki: 13 łącznie (9× `!= nil` + 4× `== nil`); `.License()`: 56 łącznie (18+38); `.DataRetention()`: **15 potwierdzone, ale operator `== nil`**, nie `!= nil` ([data_retention.go:13-111](server/channels/app/data_retention.go#L13)); `.Cluster()`: 21 łącznie (16+5) | regex `\.Metrics\(\) (!=|==) nil` itd.; generated `timerlayer.go` (501 wystąpień pola `.Metrics`) pominięty jako niereprezentatywny (kod generowany, inna semantyka) |
| 9 trafień `AccessControlPolicyHistory` w 3 plikach (2 migracje + 1 store) | ✅ POTWIERDZONE | [migration up.sql:13](server/channels/db/migrations/postgres/000134_create_access_control_policies.up.sql#L13), [down.sql:1](server/channels/db/migrations/postgres/000134_create_access_control_policies.down.sql#L1), [access_control_policy_store.go:136,260,261,334,335,544,545](server/channels/store/sqlstore/access_control_policy_store.go#L136) | regex `AccessControlPolicyHistory` |
| `AccessControlPolicyStore` ma 12 metod w interfejsie, `localcachelayer` implementuje 3 (C14, „3/12") | ✅ POTWIERDZONE | interfejs [store.go:1220-1255](server/channels/store/store.go#L1220) (12 metod); implementacja [access_control_policy_layer.go:33,48,55](server/channels/store/localcachelayer/access_control_policy_layer.go#L33) (3 metody) | ast-grep na sygnaturach `func (s LocalCacheAccessControlPolicyStore) $NAME(...)` vs deklaracje w interfejsie |
| 27 wywołań `Store().AccessControlPolicy()` poza store'em, w 8 plikach | ⚠️ DOPRECYZOWANE → 27 w **7** plikach (raport: 8) | [access_control.go](server/channels/app/access_control.go) (14), [team_access_control.go](server/channels/app/team_access_control.go) (5), [channel.go](server/channels/app/channel.go) (2), [migrations.go](server/channels/app/migrations.go) (2), [plugin_access_control.go](server/channels/app/plugin_access_control.go) (2), [post.go](server/channels/app/post.go) (1), [team.go](server/channels/app/team.go) (1) | `Select-String` grupowane po pliku, pliki `_test.go` wykluczone |
| Mutacje omijające EE: `Save` ×2 (team_access_control.go:279, migrations.go:1311), `Delete` ×2 (channel.go:4641, team.go:2195), `SetActiveStatusMultiple` ×1 (access_control.go:1941) | ✅ POTWIERDZONE (dokładne linie) | [team_access_control.go:279](server/channels/app/team_access_control.go#L279), [migrations.go:1311](server/channels/app/migrations.go#L1311), [channel.go:4641](server/channels/app/channel.go#L4641), [team.go:2195](server/channels/app/team.go#L2195), [access_control.go:1941](server/channels/app/access_control.go#L1941) | regex `Store\(\)\.AccessControlPolicy\(\)\.(Save\|Delete\|SetActiveStatusMultiple)\(` |
| 6 anonimowych struktur w `api4/access_control.go` (5 request-body + 1 response) na liniach 383, 749, 794, 1023, 1126, 1427 | ✅ POTWIERDZONE (dokładne linie) | [access_control.go:383,749,794,1023,1126,1427](server/channels/api4/access_control.go#L383) | ast-grep `var $X struct { $$$ }` / `$X := struct { $$$ }{...}` |
| 61 wywołań `api.Init*()` w `api.go:358-418`, wszystkie bezwarunkowe | ✅ POTWIERDZONE | [api.go:358-418](server/channels/api4/api.go#L358) — dokładnie 61 linii `api.Init...()` | ast-grep `api.Init$NAME()` w bloku funkcji |
| 4 dyrektywy `//go:generate` w `server/` (oembed ×2, store, plugin) | ✅ POTWIERDZONE | [oembed/endpoint.go:11](server/channels/app/oembed/endpoint.go#L11), [oembed/providers_gen.go:7](server/channels/app/oembed/providers_gen.go#L7), [store/store.go:1](server/channels/store/store.go#L1), [plugin/client_rpc.go:4](server/public/plugin/client_rpc.go#L4) | regex `^//go:generate` |
| 5 rzutowań `as unknown as` na `Props` w webappie (poza testami) | ✅ POTWIERDZONE (dokładne linie) | [policies.tsx:153-154](webapp/channels/src/components/admin_console/access_control/policies.tsx#L153), [policy_details.tsx:218-220](webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L218) | grep `as unknown as` z wykluczeniem `*.test.tsx` |
| `GET /{id}/activate` / `activatePolicy`: 11 plików spec, 14 wywołań | ⚠️ DOPRECYZOWANE → 11 plików **potwierdzone**, ale **18 wywołań** (raport: 14) | pełna lista: `ldap_sync_add.spec.ts` (2), `ldap_sync_admin.spec.ts` (2), `ldap_sync_bidirectional.spec.ts` (1), `ldap_sync_removal_bidirectional.spec.ts` (1), `ldap_sync_removal_equals.spec.ts` (1), `ldap_sync_removal_startsWith.spec.ts` (1), `advanced_policies_operators.spec.ts` (3), `advanced_policies.spec.ts` (3), `create_policies.spec.ts` (1), `attribute_changes_add.spec.ts` (1), `attribute_changes_remove.spec.ts` (2) = 18 | grep `await activatePolicy\(` per plik, z wykluczeniem linii importu i definicji w `support.ts:978` |
| „Para `fromModel`/`toModel` występuje w `sqlstore/` tylko tutaj" (C2, użyte też jako uzasadnienie kandydatury #3 w rankingu) | ⚠️ DOPRECYZOWANE — **do decyzji na etapie planowania** | Nazwany wzorzec `ToModel()` (bez `fromModel`) istnieje też w [channel_store.go:314,378,446,456](server/channels/store/sqlstore/channel_store.go#L314), [group_store.go:1155-1226](server/channels/store/sqlstore/group_store.go#L1155), [team_store.go:162,200](server/channels/store/sqlstore/team_store.go#L162), [role_store.go:75](server/channels/store/sqlstore/role_store.go#L75), oraz `botFromModel`/`NewRoleFromModel`/`NewTeamMemberFromModel` jako warianty `FromModel`. Różnica jakościowa: te konwertują **cały wiersz DB** (wszystkie kolumny) do struktury modelu, nie serializują **jedną kolumnę `jsonb`** przez prywatną strukturę-cień — w tej węższej, trafniejszej roli ABAC rzeczywiście jest jedyny. Literalne sformułowanie raportu jest zatem zbyt szerokie, choć węższy, właściwy dla argumentu wniosek (Scanner/Valuer to poprawna konwencja dla blobów jsonb) się utrzymuje. | ast-grep `func ($R $T) ToModel() *model.$M` / regex `func \w*[Ff]romModel\(` w całym `sqlstore/` |
| „20 odwołań w jednym pliku" do `accessControlPolicyV0_1`/`storeAccessControlPolicy` (C2) | ⚠️ DOPRECYZOWANE (niejednoznaczne) | Zmierzone wprost: `storeAccessControlPolicy` — 12 wystąpień identyfikatora ([:35,49,81,102,175,472,496,528,556,571,621,926](server/channels/store/sqlstore/access_control_policy_store.go#L35)); `accessControlPolicyV0_1` — 3 wystąpienia ([:26,61,86](server/channels/store/sqlstore/access_control_policy_store.go#L26)); łącznie **15**, nie 20. Rozbieżność może wynikać z innej definicji „odwołania" (np. wliczenie wywołań `.toModel()`/`fromModel(` — 8+4 dodatkowych call-site'ów, co zbliża się do 20 przy szerszej definicji). | regex `storeAccessControlPolicy\|accessControlPolicyV0_1` + osobno `\.toModel\(\)\|fromModel\(` |
| Komentarz TODO „needs to be updated… only supports v0.1" na liniach 46-47 | ✅ POTWIERDZONE (dokładne linie) | [access_control_policy_store.go:46-47](server/channels/store/sqlstore/access_control_policy_store.go#L46) | odczyt pliku, dosłowne dopasowanie tekstu |
| `PolicyEnforced`/`PolicyIsActive` jako podzapytania `EXISTS`/`COALESCE(SELECT…)` na liniach 191/192 | ✅ POTWIERDZONE (dokładne linie) | [channel_store.go:191-192](server/channels/store/sqlstore/channel_store.go#L191) | regex `PolicyEnforced\|PolicyIsActive` |
| `channelSliceColumns` — 28 wywołań, 26 z `isSelect=true`; `tableSelectQuery` reużywane 11 razy | ⚠️ DOPRECYZOWANE → **29** wywołań (raport: 28), **27** z `isSelect=true` (raport: 26); `tableSelectQuery` reużywane **9** razy (raport: 11) | `channelSliceColumns(` — 29 call-site'ów poza deklaracją funkcji ([channel_store.go:152](server/channels/store/sqlstore/channel_store.go#L152) to definicja, nie wywołanie), 2 z `isSelect=false` (811, 4452); `tableSelectQuery` — 11 wystąpień identyfikatora łącznie, z czego 1 deklaracja pola (29) + 1 przypisanie (546) + 9 realnych odczytów | `Select-String 'channelSliceColumns\('` i `'tableSelectQuery'` z ręcznym odjęciem deklaracji/przypisania |
| 3 miejsca połykania błędu `ReconcilePolicyTeamScope` (1111, 1193, 1215) | ✅ POTWIERDZONE (dokładne linie) | [access_control.go:1111,1193,1215](server/channels/api4/access_control.go#L1111) | regex `ReconcilePolicyTeamScope` |

**Uwagi końcowe do weryfikacji:**
- Wszystkie „zera" (`.DataRetention() != nil` → 0 wyników) zostały potwierdzone drugim, odwrotnym zapytaniem (`== nil`), które ujawniło właściwy wzorzec — zgodnie z poleceniem, żadne zero nie zostało przyjęte bez klasycznego grepa jako kontrpróby.
- Rozbieżności w liczbach są w większości **drobne korekty o 1-4 jednostki** (guardy, call-site'y, kolumny) — nie podważają żadnego z trzech kandydatów na podium.
- Jedyna rozbieżność, która dotyka **argumentacji** rankingu (nie tylko liczby) to twierdzenie „para `fromModel`/`toModel` występuje tylko tutaj" użyte w uzasadnieniu kandydatury #3 — literalnie zbyt szerokie, choć węższy wniosek (Scanner/Valuer jako poprawna konwencja dla jsonb) pozostaje prawdziwy. **Do decyzji na etapie planowania**, czy uzasadnienie #3 wymaga przeformułowania tego zdania.
- Podobnie liczby `.Metrics`/`.License()`/`.Cluster()` cytowane w uzasadnieniu kandydatury #1 są rozbieżne z pomiarem (patrz wiersz wyżej), choć sam wniosek jakościowy („guard jest normą repo") jest wspierany również przez `.License()` (56, blisko raportowanych 53) i `.DataRetention()` (15, dokładnie). **Do decyzji na etapie planowania**, czy warto podmienić przywołane liczby na zmierzone.
