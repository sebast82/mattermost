---
title: Invariant Aggregate Refactor — AccessControlPolicy.Type immutability
created: 2026-09-10
type: refactor-plan
---

# Plan refaktoru: agregat-strażnik dla niezmiennika `AccessControlPolicy.Type`

> **To jest PLAN.** Kod produkcyjny nie został zmodyfikowany. Dokument bazuje na
> ponownej, bezpośredniej weryfikacji kodu (nie tylko na
> [01-domain-distillation.md](01-domain-distillation.md)) — dwa z trzech
> niezmienników wskazanych tam jako złamane okazały się już naprawione lub
> celowe; poniżej wyjaśniam dlaczego i co pozostaje realnym problemem.

## KROK 0 — Kontekst

Brak `context/foundation/prd.md`/`tech-stack.md` — jak w
[01-domain-distillation.md §KROK 0](01-domain-distillation.md). Stack: Go
(`server/public/model` → `server/channels/app` → `server/channels/api4` →
`server/channels/store/sqlstore`), silnik decyzyjny ABAC (PDP/PAP,
`AccessControlServiceInterface`) jest szwem enterprise **poza tym repozytorium**
([einterfaces/pap.go](../../server/einterfaces/pap.go),
[einterfaces/pdp.go](../../server/einterfaces/pdp.go)) — co ma bezpośredni wpływ
na projekt w KROKU 4: nie możemy zagwarantować niezmiennika wewnątrz `acs`, bo
nie widzimy jego implementacji.

**Weryfikacja poprzedniej destylacji.** [01-domain-distillation.md](01-domain-distillation.md)
wskazywała trzy złamane niezmienniki ABAC. Ponowne przeczytanie kodu pokazuje,
że stan zmienił się (lub został źle zdiagnozowany):

| Niezmiennik z poprzedniej destylacji | Stan po weryfikacji |
|---|---|
| `masked_rule_deleted` pomijany na ścieżce Plugin API | **Nieaktualne.** `enforceAccessControlPolicyWriteGuards` jest dziś dzieloną funkcją dla REST i Plugin API ([access_control.go:424-472](../../server/channels/app/access_control.go#L424)); Plugin API świadomie przekazuje `mergeFromStore=false`, bo GET pluginu **nie maskuje** wartości — nie ma więc czego "cicho usuwać" ([plugin_access_control.go:164-172](../../server/channels/app/plugin_access_control.go#L164)). Guard `masked_rule_deleted` żyje w `mergeStoredPolicyExpressions` i jest wywoływany prawidłowo wtedy, gdy w ogóle ma sens (`mergeFromStore=true`, REST). |
| Rozbieżna wersja (`v0.3` REST vs `v0.5` plugin) | **Celowe, nie błąd.** To dwa różne pod-typy tego samego modelu (polityki core: parent/channel/team/permission vs polityki plugin-owned o nieprzezroczystym typie `<pluginID>:<resourceType>`) — każdy ma własny, stabilny schemat wersji. Nie ma tu jednego agregatu z dwiema sprzecznymi regułami. |
| `Type` niemutowalny po utworzeniu — egzekwowany tylko w store | **Potwierdzone i pogłębione.** To pozostaje realnym, aktywnym problemem — patrz KROK 1-3 poniżej. Jest to dziś **jedyny** z trzech kandydatów, który nadal pasuje do profilu "core + słabo egzekwowany". |

Reszta tego dokumentu skupia się wyłącznie na niezmienniku `Type`.

---

## KROK 1 — Niezmienniki zidentyfikowane wokół `AccessControlPolicy`

| # | Niezmiennik | Cytat źródłowy |
|---|---|---|
| A | Typ polityki (`Type`) nie zmienia się po utworzeniu | `if existingPolicy.Type != policy.Type { return nil, errors.New("cannot change type of existing policy") }` — [access_control_policy_store.go:216-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L216) |
| B | Polityka pluginu nie może zostać przejęta przez inny plugin ani zmienić się w typ core (i odwrotnie) — **konsekwencja A** | test `testAccessControlPolicyStoreTypeImmutableOnSave`, przypadki *"plugin type cannot become a core type"*, *"core type cannot become a plugin type"*, *"plugin type cannot be taken over by another plugin"* — [access_control_policy_store.go:137-172](../../server/channels/store/storetest/access_control_policy_store.go#L137) |
| C | Istnienie 404 dla pluginu zależy od tego, że zapisany `Type` jest wiarygodny (nie mógł się zmienić między odczytem a decyzją) | *"Gating on the stored type is sound because Type is immutable: the store rejects type changes on an existing policy."* — [plugin_access_control.go:257-259](../../server/channels/app/plugin_access_control.go#L257) |
| D | Kolizja ID między pluginami nigdy nie zamienia się w fałszywy alarm/przejęcie, bo własna polityka pluginu "nigdy nie stanie się obca" | *"...the caller's own policy can never become foreign (Type is save-immutable and plugins may only save their own types)."* — [plugin_access_control.go:63-67](../../server/channels/app/plugin_access_control.go#L63) |
| E | `Channel.PolicyEnforced` jest liczone na podstawie `AccessControlPolicies.Type = 'channel'` przypiętego do ID kanału — **kolejna konsekwencja A** | `EXISTS (SELECT 1 FROM AccessControlPolicies acp WHERE acp.ID = %sId AND acp.Type = 'channel')` — [channel_store.go:191](../../server/channels/store/sqlstore/channel_store.go#L191) |

Niezmienniki B–E to nie osobne reguły — to **odbiorcy** niezmiennika A, którzy
milcząco zakładają, że A trzyma się zawsze i wszędzie. To czyni A rdzeniowym:
jego złamanie unieważnia jednocześnie bezpieczeństwo własności pluginu (B, D),
poprawność odpowiedzi 404 (C) i egzekwowanie polityki na kanale (E).

---

## KROK 2 — Klasyfikacja i wybór

| Oś | Ocena niezmiennika A |
|---|---|
| (a) Jak rdzeniowy | Bardzo. `AccessControlPolicy` to Core subdomena ([01-domain-distillation.md §KROK 2](01-domain-distillation.md)); `Type` decyduje, która gałąź walidacji (`accessPolicyVersionV0_X`), która ścieżka ewaluacji PDP i który mechanizm egzekwowania (import/dziedziczenie dla `parent`, `PolicyEnforced` dla `channel`, scope dla `team`, ownership pluginu dla typów opaque) obowiązuje dla danego wiersza. Zmiana `Type` "pod spodem" po utworzeniu przenosi wiersz między tymi reżimami bez żadnej migracji danych. |
| (b) Jak rozsmarowany | Silnie. Sześć miejsc zapisu w tym repo dociera do tej samej reguły: [access_control.go:171](../../server/channels/app/access_control.go#L171) (REST create/update), [access_control.go:1601](../../server/channels/app/access_control.go#L1601) i [:1711](../../server/channels/app/access_control.go#L1711) (przypisanie polityki-dziecka do kanału/zespołu), [access_control.go:1655](../../server/channels/app/access_control.go#L1655) i [:1766](../../server/channels/app/access_control.go#L1766) (odpięcie), [plugin_access_control.go:199-217](../../server/channels/app/plugin_access_control.go#L199) (plugin), plus bezpośredni zapis w [team_access_control.go:279](../../server/channels/app/team_access_control.go#L279) (rekoncyliacja zasięgu zespołu). Z tych sześciu tylko jedno (plugin) ma nazwaną, testowalną barierę na poziomie `app`; pozostałe pięć polega wyłącznie na tym, że *nie da się* obejść store'u. |
| (c) Egzekwowany czy tylko deklarowany | Egzekwowany — ale **wyłącznie w jednej implementacji jednego interfejsu** (`SqlAccessControlPolicyStore.Save`), i to zwykłym `errors.New(...)`, nie domenowym `*model.AppError`. `model.AccessControlPolicy.IsValid()` w ogóle nie zna pojęcia "poprzedni stan" — bo waliduje pojedynczy snapshot, nie przejście — więc reguła strukturalnie nie może dziś żyć w modelu. |

**Wybór: niezmiennik A** (`Type` jest niemutowalny po utworzeniu). Jest
jednocześnie najbardziej rdzeniowy (czterej "konsumenci" B–E zakładają go bez
weryfikacji) i najsłabiej egzekwowany spośród realnie wciąż otwartych
kandydatów — jedyna faktyczna bariera to gołe porównanie w warstwie
persystencji, nieosiągalne z poziomu domeny i niewidoczne dla właściwego
wywołującego produkcyjnego (`AccessControlServiceInterface` / PAP), który żyje
poza tym repozytorium i którego implementacji nie możemy zweryfikować.

---

## KROK 3 — Diagnoza

**Gdzie dziś żyje reguła, warstwa po warstwie:**

1. **Model (`server/public/model/access_policy.go`)** — nieobecna. `IsValid()`
   ([access_policy.go:275-297](../../server/public/model/access_policy.go#L275))
   waliduje wyłącznie kształt pojedynczego obiektu (`Type` musi być jedną z
   dozwolonych wartości dla danej wersji), nigdy w relacji do stanu
   poprzedniego. Agregat nie ma żadnej metody przyjmującej "poprzednią wersję
   siebie".

2. **App / REST (`access_control.go`)** — `CreateOrUpdateAccessControlPolicy`
   ([access_control.go:123-180](../../server/channels/app/access_control.go#L123))
   ustawia `Version`, przepuszcza politykę przez
   `enforceAccessControlPolicyWriteGuards` (maskowanie + self-inclusion), po
   czym woła `acs.SavePolicy` ([:171](../../server/channels/app/access_control.go#L171))
   **bez żadnego uprzedniego sprawdzenia Type**. Błąd, jeśli nastąpi, przyjdzie
   dopiero z wnętrza store'u (o ile `acs.SavePolicy`, którego nie widzimy,
   w ogóle deleguje do tego store'u).

3. **App / przypisanie polityki-dziecka** — `AssignAccessControlPolicyToChannels`
   ([access_control.go:1555-1607](../../server/channels/app/access_control.go#L1555))
   i `AssignAccessControlPolicyToTeams` ([:1665](../../server/channels/app/access_control.go#L1665))
   w ogóle nie przechodzą przez `enforceAccessControlPolicyWriteGuards` — komentarz
   w kodzie to wprost przyznaje: *"the parent-policy AssignAccessControlPolicyToChannels
   flow, which validates eligibility there but bypasses this entry point"*
   ([access_control.go:138-139](../../server/channels/app/access_control.go#L138)).
   Dla `Type` to bez znaczenia praktycznego dziś (kod ustawia `Type` na
   `channel`/`team` tylko gdy `child == nil`, czyli przy tworzeniu), ale
   pokazuje, że **nie ma jednego, wspólnego "wejścia" do zapisu polityki** — są
   cztery niezależne miejsca w `access_control.go`, jedno w
   `plugin_access_control.go` i jedno w `team_access_control.go`.

4. **App / plugin (`plugin_access_control.go`)** — **jedyne miejsce z nazwaną
   barierą domenową na poziomie `app`**: `SavePluginAccessControlPolicy`
   ([:199-221](../../server/channels/app/plugin_access_control.go#L199)) samo
   odczytuje `existing` i porównuje `Type`, zwracając
   `app.access_control.plugin.type_conflict.app_error` (400) —
   [:216-217](../../server/channels/app/plugin_access_control.go#L216).
   Redundantne ze store'em (dobre - "belt-and-braces"), ale **niespójne**: ta
   sama reguła ma nazwę i kod błędu tylko na jednej z sześciu ścieżek.

5. **Store (`access_control_policy_store.go`)** — jedyna warstwa, która
   **naprawdę zawsze** egzekwuje regułę, bo wszystkie sześć ścieżek zapisu
   ostatecznie trafia do `SqlAccessControlPolicyStore.Save`
   ([:189-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L189)).
   Ale: (a) błąd to gołe `errors.New("cannot change type of existing policy")`
   — nie `*model.AppError`, więc traci status HTTP i i18n; (b) sprawdzenie
   następuje *po* rozpoczęciu transakcji i zbudowaniu `storePolicy`, mieszając
   czystą regułę domenową z konstrukcją zapytania SQL; (c) jest osiągalne
   wyłącznie wtedy, gdy wywołujący (w tym enterprise PAP, którego nie widzimy)
   faktycznie deleguje do tego store'u — nic w typach nie gwarantuje, że
   przyszła implementacja `AccessControlServiceInterface` tego nie ominie
   (np. przez inną metodę zapisu, cache, czy bezpośredni SQL).

**Kto "połyka" błąd zamiast zatrzymać operację:** nikt jawnie — ale ponieważ
błąd store'u jest gołym `error`, a nie domenowym typem, każdy przyszły
wywołujący, który go opakuje `if err != nil { return genericAppError }`, zgubi
informację "to było naruszenie niezmiennika `Type`", co utrudnia observability
i test asercje (`CheckErrorID` w testach API4 nie zadziała, bo nie ma
identyfikowalnego `app_error` id na większości ścieżek).

**Blast radius złamania:** gdyby `Type` zmienił się z `channel` na `parent` (lub
odwrotnie) na istniejącym wierszu z ID = ID kanału, `Channel.PolicyEnforced`
([channel_store.go:191](../../server/channels/store/sqlstore/channel_store.go#L191))
i filtrowanie `ExcludeAccessControlPolicyEnforced`/`AccessControlPolicyEnforced`
([channel_store.go:1434-1437](../../server/channels/store/sqlstore/channel_store.go#L1434))
zaczęłyby kłamać o tym, czy kanał jest objęty polityką — bez żadnej zmiany po
stronie kanału. Dla typów pluginowych: przejęcie ID przez inny plugin (typ `B` w
KROKU 1) unieważniłoby założenie z [plugin_access_control.go:63-67](../../server/channels/app/plugin_access_control.go#L63),
na którym opiera się cała logika "no-policy vs deny" w `resolvePluginPolicyExistence`.

---

## KROK 4 — Projekt agregatu-strażnika

### Decyzja projektowa

Reguła jest **z natury regułą przejścia stanu** (porównuje `existing` vs
`incoming`), więc nie może żyć w `IsValid()` (walidacja pojedynczego
snapshotu). Potrzebna jest osobna metoda domenowa przyjmująca oba stany, plus
**jeden** punkt wejścia repozytorium, przez który przechodzą wszystkie sześć
dzisiejszych miejsc zapisu.

Ograniczenie architektoniczne: `AccessControlServiceInterface` (PAP) jest szwem
enterprise poza tym repo — nie możemy zagwarantować, że jego implementacja
wywoła naszą nową metodę domenową. Projekt musi więc **nie polegać wyłącznie**
na tym, że `app` zawsze o tym pamięta — store pozostaje ostateczną linią obrony
(defense-in-depth), ale przestaje być *jedyną*.

### Metoda domenowa (model)

```go
// server/public/model/access_policy.go

// ErrAccessControlPolicyTypeImmutable is returned when a revision attempts to
// change the Type of an existing policy.
var ErrAccessControlPolicyTypeImmutable = errors.New("access control policy type is immutable after creation")

// ValidateRevision enforces invariants that depend on the prior state of this
// policy, not just its own shape (which IsValid already covers). existing is
// nil for a brand-new policy — nothing to compare, so it always passes.
//
// Precondition: Type must equal existing.Type when existing is non-nil.
// Violation is fail-fast: a named AppError, never a silent Type overwrite.
func (p *AccessControlPolicy) ValidateRevision(existing *AccessControlPolicy) *AppError {
	if existing == nil {
		return nil
	}
	if existing.Type != p.Type {
		return NewAppError(
			"AccessControlPolicy.ValidateRevision",
			"model.access_policy.validate_revision.type_immutable.app_error",
			map[string]any{"StoredType": existing.Type, "RequestedType": p.Type},
			"",
			http.StatusBadRequest,
		).Wrap(ErrAccessControlPolicyTypeImmutable)
	}
	return nil
}
```

Pseudokod preconditions (dla przyszłych reguł przejścia, nie tylko `Type`):

```
ValidateRevision(existing):
    if existing is nil:            # tworzenie — brak przejścia do sprawdzenia
        return ok
    if existing.Type != self.Type:
        return AppError(type_immutable, 400)   # nigdy: existing.Type = self.Type po cichu
    return ok
```

### Repozytorium (jeden punkt wejścia zamiast sześciu)

```go
// server/channels/store: nowa, wąska fasada nad istniejącym store'em.
// Nie zastępuje SqlAccessControlPolicyStore — opakowuje go, żeby domenowa
// reguła przejścia biegła RAZEM z odczytem "existing", zamiast być
// duplikowana per wywołujący.

type AccessControlPolicyRepository interface {
	// Revise loads the current row (if any) via store.Get, runs
	// incoming.ValidateRevision(existing), and only then delegates to
	// store.Save — in the store's existing single transaction. Returns the
	// domain AppError verbatim on precondition failure; never reaches SQL.
	Revise(rctx request.CTX, incoming *model.AccessControlPolicy) (*model.AccessControlPolicy, *model.AppError)
}

func (r *sqlAccessControlPolicyRepository) Revise(rctx request.CTX, incoming *model.AccessControlPolicy) (*model.AccessControlPolicy, *model.AppError) {
	existing, err := r.store.Get(rctx, incoming.ID) // nil + not-found → creation path
	if err != nil && !errors.As(err, &notFoundErr) {
		return nil, model.NewAppError("AccessControlPolicyRepository.Revise", "app.pap.get_policy.app_error", nil, err.Error(), http.StatusInternalServerError)
	}
	if appErr := incoming.ValidateRevision(existing); appErr != nil {
		return nil, appErr // fail-fast — store.Save never called
	}
	saved, err := r.store.Save(rctx, incoming) // istniejąca transakcja store'u bez zmian
	if err != nil {
		return nil, model.NewAppError("AccessControlPolicyRepository.Revise", "app.pap.save_policy.app_error", nil, err.Error(), http.StatusInternalServerError)
	}
	return saved, nil
}
```

Atomowość: `store.Save` już dziś owija całość (odczyt `existingPolicy`, zapis
historii, insert/update) w jedną transakcję SQL
([access_control_policy_store.go:189-219+](../../server/channels/store/sqlstore/access_control_policy_store.go#L189)).
`Revise` nie rozszerza granicy transakcji — dokłada tylko **jeden dodatkowy
odczyt przed transakcją**, który w praktyce już się dzieje w większości
wywołujących dzisiaj (np. `mergeStoredPolicyExpressions` i
`SavePluginAccessControlPolicy` już wołają `Get` po coś innego) — więc koszt
jest w większości przypadków zerowy netto po konsolidacji.

Store zachowuje swój dzisiejszy check
([access_control_policy_store.go:216-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L216))
jako drugą linię obrony, ale zmieniony na sentinel error
(`store.ErrAccessControlPolicyTypeImmutable`), żeby `Revise` (i każdy, kto
jednak wywoła `store.Save` bezpośrednio) mógł go rozpoznać zamiast dostawać
gołe `errors.New(...)`.

### Cienkie API / route

Trasy api4 (`InitAccessControlPolicy` i pokrewne) się nie zmieniają w kształcie
— nadal: `parse wejścia → app.CreateOrUpdateAccessControlPolicy → mapowanie
AppError na HTTP` (framework Mattermost robi to mapowanie automatycznie z
`model.AppError`, patrz istniejący wzorzec w `api4/access_control.go`). Zmienia
się tylko to, co dzieje się **wewnątrz** `app`: `CreateOrUpdateAccessControlPolicy`,
`SavePluginAccessControlPolicy`, `AssignAccessControlPolicyToChannels/Teams` i
`ReconcilePolicyTeamScope` przestają wołać `acs.SavePolicy` /
`Store().AccessControlPolicy().Save()` bezpośrednio i zamiast tego wołają
`AccessControlPolicyRepository.Revise` — egzekucja przenosi się z "każdy
wywołujący musi pamiętać" na "repozytorium pamięta za wszystkich".

Uwaga o granicy enterprise: wywołania idące przez `acs` (interfejs PAP, poza
repo) nadal mogą ominąć `Revise`, jeśli enterprise PAP ma własną ścieżkę
zapisu, która nie przechodzi przez open-core store. Ten projekt **nie
rozwiązuje** tego poza-repowego ryzyka — jedynie usuwa je z pięciu z sześciu
ścieżek, które i tak żyją w tym repozytorium, i pozostawia w store'cie
(warstwa wspólna dla wszystkich implementacji `Store`) ostatnią, twardą
barierę.

---

## KROK 5 — Before/after, plan, testy

### Before/after per dzisiejsze miejsce

| Miejsce | Dziś | Po refaktorze |
|---|---|---|
| `CreateOrUpdateAccessControlPolicy` [:171](../../server/channels/app/access_control.go#L171) | `acs.SavePolicy(rctx, policy)` — brak sprawdzenia Type w `app` | `repo.Revise(rctx, policy)` |
| `SavePluginAccessControlPolicy` [:199-221](../../server/channels/app/plugin_access_control.go#L199) | własny, zduplikowany check `existing.Type != policy.Type` + `acs.SavePolicy` | `repo.Revise(rctx, policy)` — duplikat usunięty, ten sam AppError co wszędzie indziej |
| `AssignAccessControlPolicyToChannels` [:1601](../../server/channels/app/access_control.go#L1601) | `acs.SavePolicy(rctx, child)` | `repo.Revise(rctx, child)` |
| `AssignAccessControlPolicyToTeams` [:1711](../../server/channels/app/access_control.go#L1711) | `acs.SavePolicy(rctx, child)` | `repo.Revise(rctx, child)` |
| `UnassignPoliciesFromChannels/Teams` [:1655](../../server/channels/app/access_control.go#L1655), [:1766](../../server/channels/app/access_control.go#L1766) | `acs.SavePolicy(rctx, child)` | `repo.Revise(rctx, child)` |
| `ReconcilePolicyTeamScope` [team_access_control.go:279](../../server/channels/app/team_access_control.go#L279) | `Store().AccessControlPolicy().Save(rctx, policy)` bezpośrednio, z komentarzem "scope is metadata that doesn't require [guards]" | `repo.Revise(rctx, policy)` — zachowuje pominięcie maskowania/self-inclusion (to metadata scope'u, nie treść reguł), ale zyskuje check Type za darmo |
| `SqlAccessControlPolicyStore.Save` [:216-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L216) | `errors.New("cannot change type of existing policy")` | ten sam check, ale zwraca `store.ErrAccessControlPolicyTypeImmutable` (sentinel) zamiast gołego `errors.New` |

### Plan faz (repo prowadzi testy Go standardowe — `go test`, patrz [server/AGENTS.md](../../server/AGENTS.md); brak dedykowanego runnera TDD, więc rekomenduję test-first tam gdzie to nowa logika domenowa)

1. **Faza 1 — test-first: `model.AccessControlPolicy.ValidateRevision`.**
   Nowy plik `access_policy_revision_test.go`. Przypadki legalne/nielegalne
   poniżej. Dopiero po zielonych testach dodać metodę do
   `access_policy.go`.
2. **Faza 2 — test-first: `AccessControlPolicyRepository.Revise`.** Test na
   istniejącym harnessie `storetest` (wzorem
   `testAccessControlPolicyStoreTypeImmutableOnSave`): potwierdzić, że `Revise`
   zwraca `*model.AppError` z rozpoznawalnym ID błędu, **nie** wywołując
   `store.Save`, gdy `Type` się różni.
3. **Faza 3 — podłączenie wywołujących.** Zamienić 6 miejsc z tabeli
   before/after na `repo.Revise`. Usunąć zduplikowany check w
   `SavePluginAccessControlPolicy`. Nie-TDD (mechaniczna podmiana wywołania) —
   zabezpieczona istniejącymi testami integracyjnymi w `access_control_test.go`,
   `plugin_access_control_test.go`, `team_access_control_test.go`, które dziś
   już przechodzą i muszą przejść nadal.
4. **Faza 4 — store: sentinel error.** Zmienić `errors.New(...)` na
   `store.ErrAccessControlPolicyTypeImmutable` w
   [access_control_policy_store.go:218](../../server/channels/store/sqlstore/access_control_policy_store.go#L218).
   Test-first: rozszerzyć istniejący `testAccessControlPolicyStoreTypeImmutableOnSave`
   o `errors.Is(err, store.ErrAccessControlPolicyTypeImmutable)`.

### Przypadki testowe dla niezmiennika (`ValidateRevision`)

**Legalne:**
- `existing == nil`, dowolny `Type` → ok (tworzenie).
- `existing.Type == incoming.Type` (ten sam typ core, np. `channel→channel`) → ok.
- `existing.Type == incoming.Type` dla typu pluginowego (`acme:agent→acme:agent`) → ok.
- Zmiana `Revision`/`Rules`/`Version` przy niezmienionym `Type` → ok (to nie jest przedmiotem tego niezmiennika).

**Nielegalne (muszą zwrócić `AppError` z ID `model.access_policy.validate_revision.type_immutable.app_error`, 400, bez wywołania `store.Save`):**
- `channel → parent`.
- `parent → team`.
- `team → permission`.
- `acme:agent → widgets:agent` (przejęcie przez inny plugin — typ B z KROKU 1).
- `acme:agent → channel` (plugin → core).
- `channel → acme:agent` (core → plugin).

### Nowe "load-bearing" nazwy do zarejestrowania

- `model.AccessControlPolicy.ValidateRevision(existing *AccessControlPolicy) *AppError`
- `model.ErrAccessControlPolicyTypeImmutable` (sentinel, owinięty w AppError)
- `model.access_policy.validate_revision.type_immutable.app_error` (i18n / AppError ID)
- `AccessControlPolicyRepository` (interfejs) i jego metoda `Revise`
- `store.ErrAccessControlPolicyTypeImmutable` (sentinel error store'u, zastępuje gołe `errors.New`)

---

## Podsumowanie

Ponowna weryfikacja kodu (nie tylko wcześniejszej destylacji) pokazała, że dwa
z trzech niezmienników ABAC wskazanych wcześniej jako złamane są dziś albo
naprawione (`masked_rule_deleted` jest współdzielone między REST i Plugin API
przez `enforceAccessControlPolicyWriteGuards`), albo z założenia odrębne
(rozbieżność wersji `v0.3`/`v0.5` to dwa różne pod-typy, nie jeden agregat).
Pozostaje jeden, wciąż realny: niemutowalność `AccessControlPolicy.Type` po
utworzeniu. Cztery inne miejsca w kodzie (`Channel.PolicyEnforced`, ochrona
własności typu pluginowego, jednolite 404 dla pluginu, rozróżnienie
no-policy/deny) milcząco zakładają ten niezmiennik, nie weryfikując go
ponownie — co czyni go rdzeniowym. Mimo to jest egzekwowany wyłącznie jako
efekt uboczny jednej implementacji store'u SQL, gołym `errors.New`, osiągalny
z sześciu niezależnych miejsc zapisu w `app`, z czego tylko jedno (ścieżka
pluginu) ma własną, nazwaną barierę. Zaprojektowany agregat-strażnik przenosi
regułę do jawnej metody domenowej `ValidateRevision` uruchamianej przez jedno
repozytorium (`Revise`), które ładuje stan poprzedni i odrzuca nielegalne
przejście, zanim dotrze do SQL — a store zachowuje swój check jako sentinel
error, będący ostatnią linią obrony niezależną od tego, czy przyszły
wywołujący (w tym nieznany nam enterprise PAP) o repozytorium pamięta.
