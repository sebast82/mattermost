# Refactor opportunities z długu ABAC — plan implementacji

## Overview

Ten plan realizuje trzy najlepiej ocenione okazje refaktoru z [research.md](research.md) tej zmiany: **C2** (serializacja jsonb polityki dostępu przez konwencję `Scanner`/`Valuer`), **C1** (uwidocznienie szwu enterprise PAP/PDP w `check-generated`) i **C3** (rozszerzenie `openApiSync` o porównanie schematów + spłata długu specyfikacji OpenAPI dla `access_control`). Wszystkie trzy są zmianami widoczności i konwencji — żadna nie zmienia zachowania runtime API ani UI.

Kolejność faz (C2 → C1 → C3) to świadomy wybór użytkownika "fundamenty przed najszerszym kontraktem", nie porządek wg kosztu: wg rankingu w [research.md](research.md) C1 jest tańszy i w pełni addytywny (zero zmian w istniejącym kodzie produkcyjnym), podczas gdy C2 — mimo niskiego kosztu i małego blast radius — jest realnym refaktorem działającej ścieżki zapisu/odczytu store'a. Plan zaczyna mimo to od C2, żeby warstwa store (najbardziej narażona na przyszłe pola) miała solidną konwencję, zanim ruszymy szew kompilacyjny (C1) i najszerszy kontrakt — REST (C3).

## Current State Analysis

- **C2**: [access_control_policy_store.go:26-32](../../../server/channels/store/sqlstore/access_control_policy_store.go#L26) definiuje `accessControlPolicyV0_1` — prywatną strukturę-cień duplikującą pola `model.AccessControlPolicy` (`Imports`, `Rules`, `Roles`, `Scope`, `ScopeID`) tylko po to, by zserializować je do kolumny `Data jsonb`. Komentarz [:46-47](../../../server/channels/store/sqlstore/access_control_policy_store.go#L46) to niewykonane TODO z commitu `10b1f4c5ac`. `toModel()`/`fromModel()` są jedynym miejscem w `sqlstore/` z tym wzorcem; repo ma gdzie indziej konwencję `sql.Scanner`/`driver.Valuer` na typie modelowym ([model/utils.go:100-130](../../../server/public/model/utils.go#L100) `StringArray`, [model/channel.go:53-72](../../../server/public/model/channel.go#L53) `ChannelBannerInfo`).
- **C1**: [einterfaces/access_control.go](../../../server/einterfaces/access_control.go) składa `PolicyAdministrationPointInterface` ([pap.go](../../../server/einterfaces/pap.go)) i `PolicyDecisionPointInterface` ([pdp.go](../../../server/einterfaces/pdp.go)). Jedyna implementacja żyje w prywatnym module `mattermost/enterprise`, którego `//go:build enterprise` **nigdy się nie kompiluje w CI** (CI klonuje wyłącznie `mattermost/mattermost`). Zmiana sygnatury interfejsu dziś kompiluje się, przechodzi testy i regeneruje mocki czysto — zero sygnału w PR.
- **C3**: `AccessControlPolicy` w [definitions.yaml:4786-4802](../../../api/v4/source/definitions.yaml#L4786) ma 6 pól-widm (`display_name`, `description`, `expression`, `is_active`, `update_at`, `delete_at`) i brakuje mu ~10 realnych pól modelu ([access_policy.go:196-214](../../../server/public/model/access_policy.go#L196): `active`, `revision`, `version`, `roles`, `imports`, `rules`, `scope`, `scope_id`, `props`), zero schematu dla `AccessControlPolicyRule`. Zweryfikowane dodatkowo: `AccessControlPoliciesWithCount.total_count` ([definitions.yaml:4768](../../../api/v4/source/definitions.yaml#L4768)) nie zgadza się z `json:"total"` w kodzie ([access_policy.go:192-195](../../../server/public/model/access_policy.go#L192)); `PolicySimulationUserOverride.session_overrides` jest otypowany jako `additionalProperties: type: string`, a `model.PolicySimulationByUsersParams.SessionOverrides` to `map[string]any` ([access_request.go:597](../../../server/public/model/access_request.go#L597)). Nośnik dla checkera już istnieje: [openApiSync.go](../../../tools/mattermost-govet/openApiSync/openApiSync.go) parsuje spec przez `libopenapi`, a `check-mattermost-vet-api` w [server-ci.yml:159-173](../../../.github/workflows/server-ci.yml#L159) uruchamia `make vet-api` **bezwarunkowo, bez `continue-on-error`**, na każdym relewantnym PR.

### Key Discoveries

- **U-CI-1 nadal wymaga jednorazowej weryfikacji**: `check-mattermost-vet-api` blokuje PR bez `continue-on-error` ([server-ci.yml:159-173](../../../.github/workflows/server-ci.yml#L159)), co sugeruje, że `make vet-api` dziś przechodzi — inaczej każdy PR byłby czerwony. Ale to wniosek z konfiguracji CI, nie z wykonanego uruchomienia: research.md zostawił to pytanie jako otwarte (brak `go` w środowisku badawczym), komentarz „currently not passing" w [server/Makefile:1003](../../../server/Makefile#L1003) wciąż tam jest, i to samo środowisko nie ma dziś `go` w PATH. Faza 3 zaczyna się więc od kroku 0, który zamienia to założenie w zmierzony fakt zanim powstanie jakikolwiek kod schema-comparatora.
- `make generated` ([server/Makefile:455](../../../server/Makefile#L455)) agreguje `mocks layers gen-serialized migrations-extract build-templates mmctl-docs modules-tidy default-roles-permissions`; `check-generated` ([server-ci.yml:82-121](../../../.github/workflows/server-ci.yml#L82)) failuje na niepusty `git status --porcelain` po jego uruchomieniu — każdy nowy artefakt wpięty tutaj dziedziczy blokadę za darmo.
- `tools/mattermost-govet/main.go` rejestruje analizatory (`equalLenAsserts`, `rawSql`, `apiAuditLogs`, `immut`, `emptyInterface`, `errorVarsName`, `errorVars`, `pointerToSlice`, `mutexLock`, `wraperrors`, `openApiSync` i in.) w `unitchecker.Main(...)`, ale `make vet` w [server/Makefile:966-981](../../../server/Makefile#L966) włącza tylko część flag — `openApiSync` jest już włączony i już blokujący przez osobny target `vet-api`.
- `server/channels/store/layer_generators/main.go` jest istniejącym precedensem: mały program Go generujący commitowane pliki (`retry_layer.go`, `timer_layer.go`) z szablonów, wpięty w `make generated` przez `store-layers`. To wzorzec do naśladowania dla generatora sygnatury C1.
- `TestAccessControlPolicyStore` w [storetest/access_control_policy_store.go:24-41](../../../server/channels/store/storetest/access_control_policy_store.go#L24) już biegnie na PR przeciw prawdziwemu Postgresowi — miejsce do dopisania testu round-tripu dla C2.

## Desired End State

- **C2**: `accessControlPolicyV0_1` i `storeAccessControlPolicy.toModel/fromModel`-dla-jsonb nie istnieją. Kolumna `Data` jest czytana/zapisywana przez typ implementujący `sql.Scanner`/`driver.Valuer`, zdefiniowany raz, używany przez wszystkie call site'y (`Save`, `Delete`, `getT`, `getHistoryT`, `SetActiveStatusMultiple`). Format JSON w bazie jest bajtowo identyczny z obecnym.
- **C1**: sygnatura `AccessControlServiceInterface` (PAP+PDP) jest wygenerowanym, zacommitowanym artefaktem tekstowym wpiętym w `make generated`. Zmiana sygnatury interfejsu bez regeneracji artefaktu psuje `check-generated` na PR.
- **C3**: schemat `AccessControlPolicy` + nowy schemat `AccessControlPolicyRule` w `definitions.yaml` odpowiadają dokładnie polom `model.AccessControlPolicy`/`model.AccessControlPolicyRule`; `openApiSync` porównuje też schematy (nie tylko ścieżki/metody) dla typów używanych na trasach `access_control` i blokuje PR przy rozjeździe — bez allow-listy, bo dług został spłacony w tym samym planie.

Weryfikacja końcowa: `make generated` nie generuje diffu na `master` po scaleniu; `make vet-api` przechodzi lokalnie/CI; pełna suita `storetest` dla `AccessControlPolicy` jest zielona.

## What We're NOT Doing

- Nie ujednolicamy dwóch ścieżek zapisu REST/Plugin (C5) — `mergeFromStore=false` jest udokumentowaną decyzją, nie przeoczeniem.
- Nie zmieniamy modelu bramkowania (C13), nie dodajemy kontroli optymistycznej do `Save` (C7), nie ruszamy pozostałych kandydatów z research.md poza C1/C2/C3.
- Nie budujemy generatora Go→TS ani porównania `client4.go` ↔ router (osie 2 i 3 z kandydatury C3) — to osobne, późniejsze kroki wg research.md.
- Nie zmieniamy nazwy pola JSON `total` w `AccessControlPoliciesWithCount` ani typu `SessionOverrides` w kodzie Go — to byłaby zmiana zachowania API. Poprawiamy wyłącznie specyfikację, żeby odzwierciedlała rzeczywisty kod.
- Nie dotykamy `einterfaces-mocks` ani mockery — generator C1 produkuje osobny artefakt sygnatury, nie zastępuje mocków.
- Nie wprowadzamy migracji bazy danych w żadnej z trzech faz.

## Implementation Approach

Trzy fazy sekwencyjne, każda addytywna i odwracalna. Każda kończy się realnym pomiarem CI (test / generator / checker), zanim ruszymy do kolejnej — zgodnie z zasadą z research.md, że najmocniejsze okazje refaktoru to te ze zmierzonym mechanizmem egzekwującym.

## Critical Implementation Details

- **C2 — kształt typu Scanner/Valuer.** W przeciwieństwie do `ChannelBannerInfo` (dedykowane pole/kolumna) czy `StringArray` (cały typ = cała kolumna), pola serializowane do `Data` (`Imports`, `Rules`, `Roles`, `Scope`, `ScopeID`) są dziś płaskimi polami na `model.AccessControlPolicy`, obok pól mapowanych na osobne kolumny (`ID`, `Name`, `Type`, `Active`, `CreateAt`, `Revision`, `Version`). Nie da się więc wprost zaimplementować `Scanner`/`Valuer` na całym `model.AccessControlPolicy`. Implementator musi wybrać jeden z dwóch kształtów: (a) wydzielić te pola do osadzonego pod-typu na modelu, który implementuje `Scanner`/`Valuer` i jest bezpośrednio serializowany/deserializowany do kolumny `Data`, lub (b) zostawić pola płaskie na modelu i zaimplementować `Scanner`/`Valuer` na małym typie-widoku używanym wyłącznie wewnątrz `sqlstore` (wskaźniki do pól modelu, żywy tylko na czas zapytania). Wariant (a) jest bliższy duchowi „typ modelowy" z rankingu i eliminuje ryzyko rozjazdu przy przyszłych polach; wariant (b) nie zmienia kształtu publicznego modelu. Test z kroku 1 tej fazy musi przejść niezależnie od wyboru.
- **C3 — kolejność w fazie.** Baza musi być zmierzona (krok 1: czy `make vet-api` dziś przechodzi) **przed** naprawą spec-u (krok 2), która z kolei musi poprzedzić włączenie porównania schematów blokująco (krok 3) — inaczej pierwsze uruchomienie checkera zaleje CI błędami na starym, fikcyjnym schemacie, a fail na kroku 3 będzie trudny do odróżnienia od fail-a niezwiązanego z tą fazą.

## Phase 1: Serializacja jsonb polityki dostępu na konwencję Scanner/Valuer (C2)

### Overview

Usuwa `accessControlPolicyV0_1` jako czwartą, ręcznie utrzymywaną kopię kontraktu polityki dostępu, zastępując ją konwencją `sql.Scanner`/`driver.Valuer` już stosowaną gdzie indziej w repo.

### Changes Required:

#### 1. Test wyliczeniowy round-tripu pól (krok-prerekwizyt)

**Plik**: [server/channels/store/storetest/access_control_policy_store.go](../../../server/channels/store/storetest/access_control_policy_store.go)

**Intencja**: Zamienić cichą utratę pola przy przyszłej zmianie serializacji w czerwony test, zanim cokolwiek w produkcji się zmieni — czysty pomiar. Ten test musi zostać dodany i zweryfikowany jako zielony **przeciw obecnej implementacji** (`accessControlPolicyV0_1`) — dopiero wtedy kroki 2-3 mogą zmieniać mechanizm serializacji, mając czerwoną linię obrony na wypadek regresji.

**Kontrakt**: Nowy subtest zarejestrowany w `TestAccessControlPolicyStore` (np. `testAccessControlPolicyStoreFieldRoundtrip`), który zapisuje politykę z niezerowymi wartościami we wszystkich polach niebędących kolumną (`Imports`, `Rules`, `Roles`, `Scope`, `ScopeID`, `Props`) i asertuje, że `Get` po `Save` zwraca dokładnie te same wartości.

#### 2. Typ Scanner/Valuer dla zawartości kolumny Data

**Plik**: [server/public/model/access_policy.go](../../../server/public/model/access_policy.go)

**Intencja**: Zastąpić ręczne `json.Marshal`/`Unmarshal` przez `accessControlPolicyV0_1` jednym, kanonicznym typem zaimplementowanym raz, tak by nowe pole modelu nie mogło po cichu zniknąć z persystencji.

**Kontrakt**: Nowy typ z tymi samymi tagami `json` co obecny `accessControlPolicyV0_1` (`imports`, `rules`, `roles,omitempty`, `scope,omitempty`, `scope_id,omitempty`), implementujący `Scan(any) error` i `Value() (driver.Value, error)` wzorem [ChannelBannerInfo](../../../server/public/model/channel.go#L53). Patrz sekcja „Critical Implementation Details" co do wyboru kształtu (osadzony pod-typ vs typ-widok w sqlstore).

#### 3. Usunięcie struktury-cienia i przepięcie call site'ów

**Plik**: [server/channels/store/sqlstore/access_control_policy_store.go](../../../server/channels/store/sqlstore/access_control_policy_store.go)

**Intencja**: Usunąć `accessControlPolicyV0_1` oraz jsonb-ową część `toModel`/`fromModel`; `storeAccessControlPolicy` zostaje wyłącznie nośnikiem pól mapowanych 1:1 na kolumny (`ID`, `Name`, `Type`, `Active`, `CreateAt`, `Revision`, `Version`, plus surowe bajty `Data`/`Props` do przekazania nowemu typowi).

**Kontrakt**: `Save`, `Delete`, `getT`, `getHistoryT`, `SetActiveStatusMultiple` czytają/piszą kolumnę `Data` przez `Scan`/`Value` nowego typu zamiast ręcznego `json.Unmarshal`/`Marshal`. Zawartość JSON w kolumnie pozostaje bajtowo identyczna (pilnowana testem z kroku 1).

### Success Criteria:

#### Automated Verification:

- [ ] Nowy subtest `testAccessControlPolicyStoreFieldRoundtrip` przechodzi: `go test ./server/channels/store/storetest/...` (przez `make test-server` lub uruchomienie lokalne przeciw Postgresowi)
- [ ] Cała suita `TestAccessControlPolicyStore` (Save/Delete/SetActive/SetActiveMultiple/ScopeRoundtrip/PluginPolicy/TypeImmutableOnSave) pozostaje zielona
- [ ] `make check-style` (govet) przechodzi bez nowych ostrzeżeń
- [ ] `go build ./...` w `server/` przechodzi

#### Manual Verification:

- [ ] Lokalnie: zapisać politykę z niepustymi `Imports`/`Rules`/`Roles`/`Scope`/`ScopeID`/`Props` przed i po zmianie, porównać surową zawartość kolumny `Data` w Postgresie — musi być bajtowo identyczna
- [ ] Przegląd diffu: `accessControlPolicyV0_1` i ręczne `json.Marshal`/`Unmarshal` dla `Data` nie występują już w pliku

**Implementation Note**: Po ukończeniu tej fazy i przejściu weryfikacji automatycznej, zatrzymaj się i poczekaj na potwierdzenie manualne przed przejściem do Fazy 2.

---

## Phase 2: Uwidocznienie szwu enterprise PAP/PDP (C1)

### Overview

Zamienia niewidoczny w CI kontrakt interfejsu `AccessControlServiceInterface` (PAP+PDP) na wygenerowany, zacommitowany artefakt egzekwowany przez `check-generated`.

### Changes Required:

#### 1. Weryfikacja stabilności generowanej sygnatury (krok-prerekwizyt)

**Intencja**: Upewnić się, że generator produkuje identyczny wynik przy powtórnym uruchomieniu na tym samym kodzie (stała kolejność metod, stałe formatowanie) — bez tego artefakt będzie fałszywie migotał w `check-generated` i zostanie wyłączony w pierwszym tygodniu.

**Kontrakt**: Uruchomienie generatora (patrz punkt 2) dwukrotnie z rzędu bez zmian w kodzie musi dać bajtowo identyczny plik wyjściowy.

#### 2. Generator sygnatury interfejsu

**Plik**: nowy pakiet, np. `server/einterfaces/signature_generator/main.go` (wzorem [layer_generators/main.go](../../../server/channels/store/layer_generators/main.go))

**Intencja**: Wyprodukować deterministyczny, czytelny dla człowieka artefakt tekstowy z pełną sygnaturą `PolicyAdministrationPointInterface` ([pap.go](../../../server/einterfaces/pap.go)) i `PolicyDecisionPointInterface` ([pdp.go](../../../server/einterfaces/pdp.go)) — nazwy metod, parametry, typy zwracane, w stabilnej kolejności (np. alfabetycznej lub kolejności deklaracji w źródle).

**Kontrakt**: `//go:generate` w `server/einterfaces` wskazujący na ten generator; plik wyjściowy (np. `server/einterfaces/access_control.sig.txt`) zacommitowany do repo.

#### 3. Wpięcie w make generated

**Plik**: [server/Makefile](../../../server/Makefile)

**Intencja**: Odziedziczyć istniejącą, blokującą na PR egzekucję `check-generated` za darmo.

**Kontrakt**: Nowy target `.PHONY` (np. `einterfaces-signature`) uruchamiający `go generate ./einterfaces`; dodany do listy prerekwizytów agregatu `generated:` obok `mocks layers gen-serialized migrations-extract build-templates mmctl-docs modules-tidy default-roles-permissions` ([server/Makefile:455](../../../server/Makefile#L455)).

### Success Criteria:

#### Automated Verification:

- [ ] `make einterfaces-signature` (nowy target) uruchamia się bez błędu i produkuje plik
- [ ] Dwukrotne uruchomienie `make einterfaces-signature` bez zmian w kodzie daje bajtowo identyczny plik (test stabilności)
- [ ] `make generated` na niezmienionym `master` zostawia czysty `git status --porcelain`
- [ ] `go build ./...` i `make check-style` przechodzą

#### Manual Verification:

- [ ] Na gałęzi roboczej: zmienić sygnaturę jednej metody w `pap.go` lub `pdp.go`, uruchomić `make generated`, potwierdzić że `git diff` pokazuje zmianę w artefakcie sygnatury
- [ ] Potwierdzić, że `check-generated` w CI failuje, gdy artefakt jest nieaktualny (dry-run na testowym PR)

**Implementation Note**: Po ukończeniu tej fazy i przejściu weryfikacji automatycznej, zatrzymaj się i poczekaj na potwierdzenie manualne przed przejściem do Fazy 3.

---

## Phase 3: Rozszerzenie openApiSync o schematy + spłata długu specyfikacji access_control (C3)

### Overview

Naprawia treść specyfikacji OpenAPI dla `AccessControlPolicy`/`AccessControlPolicyRule` (dziś fikcyjną), a następnie rozszerza istniejący, blokujący na PR analizator `openApiSync` o drugą pętlę porównania: schematy, nie tylko ścieżki i metody HTTP.

### Changes Required:

#### 1. Weryfikacja bazowa: czy `make vet-api` faktycznie przechodzi dziś (krok-prerekwizyt)

**Intencja**: research.md zostawił to pytanie otwarte jako [U-CI-1] z powodu braku `go` w środowisku badawczym — ten sam brak potwierdzono ponownie na etapie code review tego planu. Zanim powstanie jakikolwiek kod schema-comparatora, trzeba wiedzieć, czy `make vet-api` na niezmienionym kodzie faktycznie przechodzi; inaczej Success Criterion 3.1 może zawieść z powodów niezwiązanych z tą fazą.

**Kontrakt**: Uruchomić `make vet-api` na niezmienionym `master` (lokalnie lub w CI na pustym PR) i zapisać wynik. Jeśli fail — rozstrzygnąć przyczynę przed krokiem 2; jeśli przyczyna nie dotyczy `access_control`, odnotować to jako ryzyko blokujące resztę fazy zanim ruszy naprawa specyfikacji.

#### 2. Naprawa schematów w specyfikacji

**Plik**: [api/v4/source/definitions.yaml](../../../api/v4/source/definitions.yaml)

**Intencja**: Doprowadzić `AccessControlPolicy` do zgodności z `model.AccessControlPolicy` i dodać brakujący schemat `AccessControlPolicyRule`, żeby specyfikacja przestała być fikcją.

**Kontrakt**: Zastąpić właściwości `components.schemas.AccessControlPolicy` (dziś: `id`, `name`, `display_name`, `description`, `expression`, `is_active`, `create_at`, `update_at`, `delete_at`) polami zgodnymi z [access_policy.go:196-214](../../../server/public/model/access_policy.go#L196): `id`, `name`, `type`, `active`, `create_at`, `revision`, `version`, `roles`, `imports`, `rules` (`$ref` do nowego `AccessControlPolicyRule`), `scope`, `scope_id`, `props`. Dodać `components.schemas.AccessControlPolicyRule` z polami `actions`, `expression`, `name`, `role` (patrz [access_policy.go:216-225](../../../server/public/model/access_policy.go#L216)). Poprawić `AccessControlPoliciesWithCount.total_count` → `total` ([definitions.yaml:4768](../../../api/v4/source/definitions.yaml#L4768), zgodnie z `json:"total"` w kodzie). Poprawić typ `PolicySimulationUserOverride.session_overrides` z `additionalProperties: {type: string}` na typ odzwierciedlający `map[string]any` (dowolne wartości, nie tylko stringi).

#### 3. Rozszerzenie analizatora o porównanie schematów

**Plik**: [tools/mattermost-govet/openApiSync/openApiSync.go](../../../tools/mattermost-govet/openApiSync/openApiSync.go)

**Intencja**: Dodać drugą pętlę porównania obok istniejącego `processRouterInit` (który sprawdza wyłącznie ścieżki/metody): dla typów Go używanych jako request/response body na trasach `access_control`, porównać tagi `json` struktury z właściwościami odpowiadającego schematu OpenAPI (`components.schemas`) — brakujące pola, pola-widma, niezgodność typów.

**Kontrakt**: Nowa funkcja analogiczna do `processRouterInit`, wołana z `run()`, operująca na `v3high.Schema.Properties` z już sparsowanego modelu spec (`libopenapi`) oraz na AST struktur `model.AccessControlPolicy`, `model.AccessControlPolicyRule`, `model.AccessControlPoliciesWithCount`. Zgłoszenia przez `pass.Reportf` tym samym mechanizmem co dziś dla rozjazdu ścieżek — brak osobnej allow-listy, bo dług specyfikacji został spłacony w kroku 2 tej fazy. Pola typu `any`/`map[string]any` (np. `Props`) są sprawdzane wyłącznie pod kątem obecności w schemacie (dopasowanie do `type: object`), nie pod kątem głębokiej zgodności typu — nie mają stałego kształtu, więc porównanie typu nie ma tu znaczenia.

### Success Criteria:

#### Automated Verification:

- [ ] `make vet-api` przechodzi lokalnie/w CI po poprawkach specyfikacji i rozszerzeniu analizatora
- [ ] Nowe testy jednostkowe dla funkcji porównania schematów w `go test ./tools/mattermost-govet/...` (fixture: mała spec + fixture struktura Go, wzorem istniejących testów w pakiecie)
- [ ] `swagger-cli validate` (istniejący krok w [api/Makefile](../../../api/Makefile)) przechodzi na poprawionej specyfikacji
- [ ] `make check-generated` / `make generated` bez zmian (ta faza nie dotyka artefaktów generowanych)

#### Manual Verification:

- [ ] Na gałęzi testowej: celowo wprowadzić pole-widmo lub usunąć realne pole ze schematu `AccessControlPolicy`, potwierdzić że nowy checker to zgłasza i `make vet-api` failuje
- [ ] Przegląd wygenerowanej dokumentacji API (`api/v4/html`) — schemat `AccessControlPolicy` wygląda sensownie dla czytelnika zewnętrznego

**Implementation Note**: Po ukończeniu tej fazy i przejściu weryfikacji automatycznej, zatrzymaj się i poczekaj na potwierdzenie manualne. To ostatnia faza planu.

---

## Testing Strategy

### Unit Tests:

- Faza 1: nowy subtest round-tripu pól w `storetest`, uruchamiany jako część istniejącej suity `TestAccessControlPolicyStore`
- Faza 3: nowe testy jednostkowe funkcji porównania schematów w `tools/mattermost-govet/openApiSync`

### Integration Tests:

- Faza 1: pełna suita `storetest` przeciw prawdziwemu Postgresowi (już biegnie na PR)
- Faza 3: `make vet-api` jako integracyjny test kontraktu Go↔YAML (już biegnie na PR)

### Manual Testing Steps:

1. Faza 1: porównanie surowej zawartości kolumny `Data` przed/po zmianie serializacji
2. Faza 2: celowa zmiana sygnatury interfejsu na gałęzi roboczej, potwierdzenie że `check-generated` to wykrywa
3. Faza 3: celowe wprowadzenie rozjazdu schematu na gałęzi roboczej, potwierdzenie że `make vet-api` to wykrywa

## Performance Considerations

Brak istotnych implikacji wydajnościowych. Faza 1 zmienia mechanizm serializacji tej samej ilości danych na tej samej ścieżce zapisu/odczytu (bez zmiany złożoności). Fazy 2 i 3 dotyczą wyłącznie czasu budowania/CI, nie ścieżki runtime.

## Migration Notes

Żadna z trzech faz nie wymaga migracji bazy danych ani migracji danych istniejących. Faza 1 zachowuje identyczny format JSON w kolumnie `Data` — istniejące wiersze pozostają czytelne bez zmian.

## References

- Badanie źródłowe: [research.md](research.md)
- Analiza wejściowa (dług ABAC): [context/changes/abac-work-in-progress/research.md](../abac-work-in-progress/research.md)
- Wzorzec Scanner/Valuer: [server/public/model/channel.go:53-72](../../../server/public/model/channel.go#L53) (`ChannelBannerInfo`), [server/public/model/utils.go:100-130](../../../server/public/model/utils.go#L100) (`StringArray`)
- Wzorzec generatora commitowanego artefaktu: [server/channels/store/layer_generators/main.go](../../../server/channels/store/layer_generators/main.go)
- Analizator do rozszerzenia: [tools/mattermost-govet/openApiSync/openApiSync.go](../../../tools/mattermost-govet/openApiSync/openApiSync.go)

## Progress

> Konwencja: `- [ ]` do zrobienia, `- [x]` zrobione. Dopisz ` — <commit sha>` gdy krok wyląduje. Nie zmieniaj nazw kroków.

### Phase 1: Serializacja jsonb polityki dostępu na konwencję Scanner/Valuer (C2)

#### Automated

- [ ] 1.1 Nowy subtest round-tripu pól przechodzi (storetest przeciw Postgresowi)
- [ ] 1.2 Cała suita `TestAccessControlPolicyStore` zielona
- [ ] 1.3 `make check-style` przechodzi bez nowych ostrzeżeń
- [ ] 1.4 `go build ./...` przechodzi

#### Manual

- [ ] 1.5 Zawartość kolumny `Data` bajtowo identyczna przed/po zmianie
- [ ] 1.6 Przegląd diffu — brak `accessControlPolicyV0_1` i ręcznego marshalowania `Data`

### Phase 2: Uwidocznienie szwu enterprise PAP/PDP (C1)

#### Automated

- [ ] 2.1 Nowy target generatora uruchamia się bez błędu
- [ ] 2.2 Dwukrotne uruchomienie generatora daje identyczny plik
- [ ] 2.3 `make generated` czysty na niezmienionym `master`
- [ ] 2.4 `go build ./...` i `make check-style` przechodzą

#### Manual

- [ ] 2.5 Zmiana sygnatury na gałęzi roboczej widoczna w diffie po `make generated`
- [ ] 2.6 `check-generated` w CI failuje przy nieaktualnym artefakcie (dry-run)

### Phase 3: Rozszerzenie openApiSync o schematy + spłata długu specyfikacji access_control (C3)

#### Automated

- [ ] 3.1 `make vet-api` przechodzi po poprawkach
- [ ] 3.2 Nowe testy jednostkowe porównania schematów przechodzą
- [ ] 3.3 `swagger-cli validate` przechodzi
- [ ] 3.4 `make check-generated` / `make generated` bez zmian

#### Manual

- [ ] 3.5 Celowy rozjazd schematu na gałęzi testowej zgłoszony przez checker
- [ ] 3.6 Przegląd wygenerowanej dokumentacji API dla `AccessControlPolicy`
