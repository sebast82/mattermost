---
title: Domain Distillation — mattermost
created: 2026-09-10
type: domain-distillation
---

# Destylacja domeny — mattermost

## KROK 0 — Kontekst i ograniczenia

**Brak dokumentów wymagań.** W repo nie istnieje `context/foundation/prd.md` ani
`context/foundation/tech-stack.md` — `context/foundation/` zawiera wyłącznie
[README.md](../foundation/README.md), czyli konwencję pisania foundation docs, nie
treść wymagań. **Ograniczenie:** poniższa destylacja opiera się na (1) opisie
produktu z [README.md](../../README.md#L3), (2) analizie historii gita i grafu
importów w [context/map/repo-map.md](../map/repo-map.md) i
[artifact-1-territory.md](../map/artifact-1-territory.md), (3) istniejącym,
głębokim badaniu jednego przepływu domenowego w
[context/changes/abac-work-in-progress/research.md](../changes/abac-work-in-progress/research.md),
oraz (4) bezpośrednim czytaniu kodu domenowego w `server/public/model`. Nie ma
dokumentu wizji/success criteria — klasyfikacja Core/Supporting/Generic w KROKU 2
opiera się na opisie produktu z README i na tym, gdzie realnie koncentruje się
praca zespołu (mapa repo), nie na formalnie spisanych celach biznesowych.

**Stack i struktura.** Z [README.md](../../README.md#L3): *"Mattermost is an open
core, self-hosted collaboration platform that offers chat, workflow automation,
voice calling, screen sharing, and AI integration (...) written in Go and React,
runs as a single Linux binary, and relies on PostgreSQL."* Logika biznesowa żyje
w trzech warstwach Go (`server/channels/store` → `server/channels/app` →
`server/channels/api4`) nad współdzielonym modelem domenowym
`server/public/model`, lustrzanym ręcznie w `webapp/platform/types` po stronie
TypeScript ([repo-map.md §1](../map/repo-map.md)). Enterprise (`server/enterprise/`)
dopina się runtime'owo przez `einterfaces/*` — commercial-core split, gdzie część
kodu domenowego (PDP/PAP dla ABAC) **nie istnieje w tym repozytorium**
([research.md §2.1](../changes/abac-work-in-progress/research.md)).

---

## KROK 1 — Ubiquitous Language

| Pojęcie | Definicja | Cytat źródłowy | Kod |
|---|---|---|---|
| **Channel** | Jednostka konwersacji; typy: otwarty, prywatny, DM, grupa, „space”, oraz „board” (otwarty/prywatny) | *"ChannelTypeOpen/Private/Direct/Group/Space/OpenBoard/PrivateBoard"* | [channel.go:29-35](../../server/public/model/channel.go#L29) |
| **Board (kanał typu BO/BP)** | Wariant kanału z odrębną walidacją: wymaga `TeamId` i niepustego `DisplayName` | *"IsValidBoard performs the input-validation checks specific to board channels"* | [channel.go:397-399](../../server/public/model/channel.go#L397) |
| **Post** | Wiadomość w kanale; niezmienniki: `UserId`/`ChannelId` muszą być poprawnym ID, długość `Message` ≤ limit | `Post.IsValid(maxPostSize int)` | [post.go:500](../../server/public/model/post.go#L500) |
| **Team** | Grupa nadrzędna wobec kanałów — kontekst przypisania polityk ABAC (`scope="team"`) | *"Scope string \`json:"scope,omitempty"\` // "" (system) or "team""* | [access_policy.go:212](../../server/public/model/access_policy.go#L212) |
| **User** | Podmiot działań; nośnik atrybutów (CPA) używanych w wyrażeniach ABAC | `User.IsValid()` | [user.go:388](../../server/public/model/user.go#L388) |
| **Permission** | Nazwana zdolność działania w danym zakresie (`system_scope`/`team_scope`/`channel_scope`/...) | `PermissionScopeSystem/Team/Channel/Group/Playbook/Run` | [permission.go:11-16](../../server/public/model/permission.go#L11) |
| **Role / Scheme** | Zestaw uprawnień przypisany do podmiotu; Scheme nadaje zestaw ról per zakres (team/channel/playbook/run) | `SchemeScopeTeam/Channel/Playbook/Run` | [scheme.go:16-19](../../server/public/model/scheme.go#L16) |
| **AccessControlPolicy (ABAC)** | Polityka kontroli dostępu wyrażona jako reguły CEL nad atrybutami; typy: `parent` (nadrzędna), `channel`, `team`, `permission` | *"ABAC (...) pozwala administratorowi zdefiniować politykę dostępu jako wyrażenie CEL nad atrybutami użytkownika (...) i przypiąć ją do kanałów lub zespołów"* | [research.md §1.1](../changes/abac-work-in-progress/research.md); typy: [access_policy.go:17-20](../../server/public/model/access_policy.go#L17) |
| **AccessControlPolicyRule** | Pojedyncza reguła polityki: `Actions`, wyrażenie CEL, opcjonalne `Name`/`Role` (rola kanałowa dla reguł v0.4) | *"Role is the channel-scoped role this rule applies to (...) for v0.4 permission rules"* | [access_policy.go:216-224](../../server/public/model/access_policy.go#L216) |
| **Inherit (dziedziczenie polityki)** | Przypisanie polityki nadrzędnej (`parent`) do kanału tworzy/aktualizuje politykę-dziecko, dopisując `parent.ID` do `Imports` | `child.Inherit(parent)` | [access_policy.go:657](../../server/public/model/access_policy.go#L657) via [research.md §1.4 pkt 8](../changes/abac-work-in-progress/research.md) |
| **Masking (maskowanie atrybutów)** | Ukrywanie wartości literalnych w wyrażeniach CEL przed wywołującym bez uprawnień do ich odczytu; domyślnie **włączone** | flaga `AttributeValueMasking` domyślnie `true` | [feature_flags.go:181](../../server/public/model/feature_flags.go#L181) via [research.md §1.7](../changes/abac-work-in-progress/research.md) |
| **Self-inclusion** | Niezmiennik: administrator zapisujący politykę nie może samego siebie z niej wykluczyć | `checkSelfInclusion` → `QueryUsersForExpression` | [access_control.go:474-493](../../server/channels/app/access_control.go#L474) via [research.md §1.3 pkt 25](../changes/abac-work-in-progress/research.md) |
| **PAP / PDP (Policy Administration/Decision Point)** | Szew enterprise: interfejsy definiujące zapis i ocenę polityk; **brak implementacji w tym repo** | *"AccessControlServiceInterface to czysty szew — 20 metod bez implementacji"* | [einterfaces/pap.go:14](../../server/einterfaces/pap.go#L14), [pdp.go:14](../../server/einterfaces/pdp.go#L14) via [research.md §2.1](../changes/abac-work-in-progress/research.md) |
| **PropertyField (PSAv2 / CPA)** | Definicja atrybutu niestandardowego (Custom Profile Attribute), z typem, poziomem docelowym (system/team/channel) i poziomem uprawnień do operowania na nim | *"PropertyFieldTargetLevelSystem/Team/Channel"*, `PermissionLevelNone/Sysadmin/Member/Admin` | [property_field.go:35-46](../../server/public/model/property_field.go#L35) |
| **PropertyValue** | Wartość atrybutu przypisana do konkretnego obiektu (post/kanał/user/...) | `PropertyFieldObjectTypePost/Channel/User/Template/Session` | [property_field.go:49-53](../../server/public/model/property_field.go#L49) |
| **Grupa property `access_control`** | Grupa właściwości (dawniej `custom_profile_attributes`), którą rozwiązuje resolver maskowania; migrowana pod nową nazwą | *"przemianowuje grupę property `custom_profile_attributes` → `access_control`"* | migracja `000176_migrate_cpa_to_access_control` via [research.md §1.8](../changes/abac-work-in-progress/research.md) |
| **Recap** | Wygenerowane przez AI podsumowanie wiadomości w wybranych kanałach dla użytkownika | struct `Recap{UserId, Title, Channels []*RecapChannel, ...}` | [recap.go:6-19](../../server/public/model/recap.go#L6) |
| **Content Flagging (spillage report)** | Proces zgłaszania/oceny nieodpowiedniej treści posta przez recenzenta z workflow status (Pending/Assigned/Removed/Retained) | `ContentFlaggingStatusPending/Assigned/Removed/Retained` | [content_flagging.go:22-27](../../server/public/model/content_flagging.go#L22) |
| **Plugin** | Rozszerzenie zewnętrzne; może posiadać własne, przestrzenne policy types (`<pluginID>:<resourceType>`) | *"PluginAccessControlPolicyTypeSeparator separates the owning plugin ID from the resource-type segment"* | [access_policy.go:24-28](../../server/public/model/access_policy.go#L24) |
| **Client4** | Kontrakt REST utrzymywany ręcznie w trzech (a dla ABAC w czterech) kopiach: Go, TS, OpenAPI YAML, plus wewnętrzna struktura store | *"kontrakt REST istnieje w 3 ręcznie pisanych kopiach i jest mierzalnie rozjechany"* | [repo-map.md §1 pkt 2](../map/repo-map.md); czwarta kopia: [research.md §2.2](../changes/abac-work-in-progress/research.md) |

---

## KROK 2 — Klasyfikacja subdomen: Core / Supporting / Generic

| Obszar / pojęcie | Kategoria | Uzasadnienie |
|---|---|---|
| **Channel, Post, Team, User, realtime (WebSocket)** | **Core** | To jest sam produkt: *"chat, workflow automation, voice calling, screen sharing"* ([README.md](../../README.md#L3)). Bez modelu konwersacji nie ma Mattermost. Potwierdzone strukturalnie: `public/model` to hub repo — fan-in 129 pakietów, zasięg tranzytywny 84% ([repo-map.md §1 pkt 1](../map/repo-map.md)). |
| **Access Control / ABAC (AccessControlPolicy, PDP/PAP, masking)** | **Core** (różnicująca funkcja produktu) | Najgorętszy obszar produktowy okna analizy: 655 dotknięć / 125 commitów, migracje schematu do ostatnich tygodni, przekrój store→app→api4→admin UI→E2E ([repo-map.md §1 pkt 4](../map/repo-map.md), [artifact-1-territory.md §1d](../map/artifact-1-territory.md)). To inwestycja różnicująca produkt (enterprise-grade governance), nie utrzymanie istniejącej funkcji — stąd Core, mimo że silnik decyzyjny (PDP) żyje poza tym repo. |
| **Property Fields / Custom Profile Attributes (PSAv2)** | **Supporting** | Wspiera ABAC (nośnik atrybutów do wyrażeń CEL) oraz profile użytkownika, ale samodzielnie nie jest różnicującą propozycją wartości — to infrastruktura danych, na której core (ABAC) się opiera. |
| **Permissions / Roles / Schemes (RBAC)** | **Supporting** | Klasyczny, dojrzały mechanizm uprawnień pod produktem — niezbędny, ale nie jest dziś obszarem inwestycji (brak w top hot-spots poza `access_control.go`, który jest ABAC, nie RBAC). |
| **Admin Console / System Console UI** | **Supporting** | #2 najgorętszy obszar (1040 dotknięć, ×1.9 kw/kw, [artifact-1-territory.md §1a,§1c](../map/artifact-1-territory.md)) — ale to warstwa konfiguracji/operacji nad core, nie sam produkt rozmowy. |
| **Recap (AI summary), Content Flagging, AI integration** | **Supporting** (rozwijające różnicowanie) | Nowe funkcje nadbudowane nad core (kanały/posty); wskazane w README jako część oferty (*"AI integration"*), ale wciąż nadbudowa nad rozmową, nie jej fundament. |
| **REST kontrakt (client4.go / client4.ts / OpenAPI YAML)** | **Generic** (ale krytyczny operacyjnie) | To infrastruktura integracji, nie logika biznesowa — mechanizm ten sam niezależnie od domeny. Sklasyfikowany Generic pomimo wysokiego ryzyka, bo jego wartość nie różnicuje produktu; jego **stan** (rozjazd) jest za to najpilniejszym ryzykiem technicznym (patrz KROK 5). |
| **Auth (session/oauth/saml/ldap), audit, compliance** | **Generic** | Standardowa funkcjonalność enterprise-collaboration, nieróżnicująca; jedyny zespołowy wpis w CODEOWNERS to `@mattermost/product-security` dla `authentication.go`/`authorization.go` ([CODEOWNERS:2-3](../../CODEOWNERS#L2)), co potwierdza traktowanie jako współdzielonej infrastruktury bezpieczeństwa, nie osobnej domeny biznesowej. |
| **Plugin API / integracje / webhooki** | **Generic** | Mechanizm rozszerzalności — wspiera każdą domenę jednakowo; ma już narzędzia dojrzałości (generator + `plugin-checker` w CI, [repo-map.md §5.1](../map/repo-map.md)), w przeciwieństwie do kontraktu REST. |

---

## KROK 3 — Kandydaci na agregaty i ich niezmienniki

| Kandydat na agregat | Niezmiennik | Cytat/dowód | Status egzekwowania |
|---|---|---|---|
| **AccessControlPolicy** | Polityka typu `parent` musi mieć niepuste `Rules` i nie może mieć `Imports` | `if len(p.Rules) == 0 { ... } if len(p.Imports) > 0 { ... }` | [access_policy.go:303-309](../../server/public/model/access_policy.go#L303) — **egzekwowane** w `IsValid()`, per-wersja (v0.1-v0.5 różne zestawy reguł) |
| **AccessControlPolicy** | Administrator zapisujący politykę nie może wykluczyć samego siebie z dostępu (self-inclusion) | `checkSelfInclusion` po zapisie reguł | [access_control.go:462-493](../../server/channels/app/access_control.go#L462) — **egzekwowane**, ale niezależnie od flagi maskowania i tylko na ścieżce REST |
| **AccessControlPolicy** | Reguła niosąca ukrytą (zamaskowaną) wartość nie może zostać usunięta przez wywołującego, który jej nie widzi (`masked_rule_deleted`) | *"To jest zabezpieczenie przed poszerzeniem dostępu przez skasowanie reguły, której się nie widzi"* | [access_control.go:317-347](../../server/channels/app/access_control.go#L317) — **egzekwowane na ścieżce REST (`mergeFromStore=true`), ale POMIJANE na ścieżce Plugin API** (`mergeFromStore=false`, [plugin_access_control.go:225](../../server/channels/app/plugin_access_control.go#L225)) — **niezmiennik złamany na jednej z dwóch ścieżek zapisu** |
| **AccessControlPolicy** | Typ polityki (`Type`) jest niezmienny po utworzeniu | zmiana `Type` → twardy błąd na poziomie store | [access_control_policy_store.go:216-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L216) — **egzekwowane wyłącznie w store**, nie w `model.IsValid()` — niezmiennik domenowy żyje w warstwie persystencji, nie w agregacie |
| **AccessControlPolicy** | Wersja schematu reguł (`Version`) jest ustalana przez serwer, nie przez klienta | server nadpisuje `policy.Version` | [access_control.go:150](../../server/channels/app/access_control.go#L150) (`v0.3`) vs [plugin_access_control.go:202](../../server/channels/app/plugin_access_control.go#L202) (`v0.5`) — **egzekwowane, ale niespójnie**: dwie ścieżki zapisu wymuszają dwie różne wersje tego samego typu domenowego |
| **Channel** | Baner kanału (`BannerInfo`) może być włączony wyłącznie dla kanałów typu Open/Private | `if o.Type != ChannelTypeOpen && o.Type != ChannelTypePrivate { ... }` | [channel.go:361-364](../../server/public/model/channel.go#L361) — **egzekwowane** w `Channel.IsValid()` |
| **Channel** | Kanał `GroupConstrained` musi wspierać synchronizację grup | `if o.IsGroupConstrained() && !o.SupportsGroupSync()` | [channel.go:387-389](../../server/public/model/channel.go#L387) — **egzekwowane** |
| **Post** | Długość wiadomości nie może przekroczyć `maxPostSize` (parametr configu, nie stała) | `utf8.RuneCountInString(o.Message) > maxPostSize` | [post.go:528-530](../../server/public/model/post.go#L528) — **egzekwowane** |
| **PropertyField** | `PermissionLevel` musi być jedną z 4 wartości (`none/sysadmin/member/admin`); `admin` rozwiązuje się kontekstowo do roli celu (sysadmin/team admin/channel admin) | *"PermissionLevelAdmin resolves to the admin of the field's target (...) documented at hasPropertyFieldPermissionLevel in the app package"* | [property_field.go:41-44](../../server/public/model/property_field.go#L41) — **deklarowane w modelu, egzekwowanie przeniesione do funkcji w warstwie `app`** (adnotacja wprost w komentarzu — model nie jest samowystarczalny) |

---

## KROK 4 — Rozjazdy MODEL vs KOD

Poniższa lista pochodzi głównie z istniejącego badania end-to-end przepływu
zapisu ABAC ([research.md](../changes/abac-work-in-progress/research.md)), które
przeszło pomiar strukturalny (ast-grep) — najbardziej zweryfikowany materiał w
repo na temat rozjazdów model/kod.

| Dokument/model mówi X | Kod robi Y | Dowód |
|---|---|---|
| Kontrakt REST ma być spójny w 3 kopiach (Go/TS/YAML) | Dla `AccessControlPolicy` istnieje **czwarta** kopia — `accessControlPolicyV0_1` w sqlstore, o mylącej nazwie (zawiera bieżący kształt, nie v0.1) | [access_control_policy_store.go:26-32](../../server/channels/store/sqlstore/access_control_policy_store.go#L26) via [research.md §2.2](../changes/abac-work-in-progress/research.md) |
| Model Go (`access_policy.go:196`) definiuje pole `create_at` | TS deklaruje `created_at?: number` — pole, którego serwer nigdy nie wysyła; webapp odczytuje je i zawsze trafia w fallback `Date.now()` | Go: [access_policy.go:201](../../server/public/model/access_policy.go#L201); TS: [access_control.ts:11](../../webapp/platform/types/src/access_control.ts#L11); odczyt: [channel_details.tsx:788,852](../../webapp/channels/src/components/admin_console/team_channel_settings/channel/details/channel_details.tsx#L788) |
| TS deklaruje `props: Record<string, unknown[]>` | Go zapisuje skalary (`child_ids` tablica, ale `channel_count`/`team_count` to `int`, nie tablica) — webapp obchodzi to podwójnym rzutowaniem `as unknown as` w 5 miejscach produkcyjnych | Go: [access_control.go:2101-2103](../../server/channels/app/access_control.go#L2101); TS obejścia: [policies.tsx:153-154](../../webapp/channels/src/components/admin_console/access_control/policies.tsx#L153), [policy_details.tsx:218-220](../../webapp/channels/src/components/admin_console/access_control/policy_details/policy_details.tsx#L218) |
| Spec OpenAPI (`definitions.yaml`) ma dokumentować `AccessControlPolicy` | YAML pomija 10 realnych pól (w tym `revision`, `version`, `roles`, `imports`, `rules`, `scope`, `scope_id`) i dodaje 6 pól-widm, które nie istnieją w kodzie (`display_name`, `description`, `expression`, `is_active`, `update_at`, `delete_at`); **nie ma w ogóle** schematu `AccessControlPolicyRule` | [definitions.yaml:4771](../../api/v4/source/definitions.yaml#L4771) via [research.md §2.2](../changes/abac-work-in-progress/research.md) |
| `client4.go` ma być pełnym klientem Go dla API ABAC | Brakuje mu 4 z 17 endpointów ABAC (`cel/simulate_users`, `cel/validate_requester`, `cel/autocomplete/fields`, `cel/visual_ast`) — testy integracyjne Go nie mogą przejść tymi ścieżkami | [client4.go:8372-8507](../../server/public/model/client4.go#L8372) vs [access_control.go:48-67](../../server/channels/api4/access_control.go#L48) via [research.md §2.2](../changes/abac-work-in-progress/research.md) |
| Komentarz w kodzie: *"ABAC is gated at route registration; only check masking here"* | Rejestracja trasy (`InitAccessControlPolicy()`) nie ma **żadnej** bramki — jedyną realną bramką jest `acs == nil → 501` | [access_control.go:509](../../server/channels/app/access_control.go#L509) (komentarz nieaktualny) vs [api4/api.go:414](../../server/channels/api4/api.go#L414) via [research.md §1.7](../changes/abac-work-in-progress/research.md) |
| `EnableAttributeBasedAccessControl` (config) sugeruje globalny wyłącznik ABAC | Ustawienie **nigdy nie jest sprawdzane** na ścieżce zapisu polityki — polityka i tak zostaje utworzona przy buildzie enterprise | [config.go:4052,4066](../../server/public/model/config.go#L4052) via [research.md §1.7](../changes/abac-work-in-progress/research.md) |
| `TeamMembershipAccessControlEnabled()` opisywana jako jedyna bramka licencyjna ścieżki zapisu | Ścieżka Plugin API ma **własną, odrębną** bramę licencyjną — więc „jedyne” dotyczy tylko ścieżek REST | [team.go:932](../../server/channels/app/team.go#L932) vs [plugin_access_control.go:44](../../server/channels/app/plugin_access_control.go#L44) via [research.md §1.4 pkt 4](../changes/abac-work-in-progress/research.md) |
| `Channel.PolicyEnforced` wygląda jak stan przechowywany na encji | To pole jest **liczone przy każdym odczycie** przez `EXISTS (SELECT 1 FROM AccessControlPolicies …)`, nie jest kolumną | [channel_store.go:191](../../server/channels/store/sqlstore/channel_store.go#L191) via [research.md §1.4](../changes/abac-work-in-progress/research.md) |
| CODEOWNERS ma pilnować krytycznych granic modelu domenowego | `CODEOWNERS` ma 19 linii i **nie pokrywa** `public/model`, `platform/types` ani `api/v4/source` — powierzchnie o najwyższym blast radius w repo nie mają właściciela | [repo-map.md §1 pkt 5](../map/repo-map.md), [CODEOWNERS](../../CODEOWNERS) |

---

## KROK 5 — Ranking refaktoru

Ranking wg wartości (jak rdzeniowy jest niezmiennik dla Core subdomeny ABAC) i
ryzyka (jak słabo jest dziś egzekwowany / jak szeroki jest blast radius cichej
awarii):

1. **#1 do refaktoru: guard `masked_rule_deleted` musi obowiązywać na obu ścieżkach zapisu (REST i Plugin API).**
   Wartość: to jedyny niezmiennik chroniący przed *rozszerzeniem* dostępu przez
   ukrytą regułę — najbardziej bezpośrednio powiązany z security promise ABAC
   (subdomena Core). Ryzyko: dziś jest **całkowicie pomijany** na ścieżce Plugin
   API (`mergeFromStore=false`, [plugin_access_control.go:225](../../server/channels/app/plugin_access_control.go#L225)),
   co oznacza, że plugin może po cichu usunąć regułę niosącą zamaskowaną
   wartość bez wyzwolenia strażnika 403. To nie dług dokumentacyjny — to luka
   w egzekwowaniu niezmiennika bezpieczeństwa w core subdomenie.

2. **#2: ujednolicić wymuszaną wersję (`Version`) polityki w jednym miejscu.**
   Dziś trzy miejsca w `access_control.go` wymuszają `v0.3`, a
   `plugin_access_control.go` niezależnie wymusza `v0.5` — ten sam agregat ma
   dwie rozbieżne, twardo zakodowane reguły „która wersja obowiązuje”, co czyni
   każdą przyszłą migrację wersji (v0.6+) miejscem podwójnej, łatwej do
   przeoczenia zmiany.

3. **#3: przenieść niezmiennik „`Type` jest niemutowalny po utworzeniu” z warstwy store do `AccessControlPolicy.IsValid()`.**
   Obecnie niezmiennik istnieje wyłącznie jako check w SQL-store
   ([access_control_policy_store.go:216-219](../../server/channels/store/sqlstore/access_control_policy_store.go#L216)) — każda alternatywna ścieżka
   zapisu (a wiemy, że istnieją co najmniej dwie: REST i Plugin API, plus
   bezpośredni zapis w `ReconcilePolicyTeamScope`, [team_access_control.go:279](../../server/channels/app/team_access_control.go#L279))
   musi przejść przez store, by niezmiennik zadziałał. Przeniesienie do modelu
   uczyniłoby go niezależnym od ścieżki wejścia.

4. **#4 (niżej priorytetowe, bo Generic, nie Core): domknąć kontrakt REST generatorem lub checkerem w CI**,
   tak jak już istnieje dla Plugin API (`make pluginapi` + `plugin-checker`,
   [repo-map.md §5.1](../map/repo-map.md)). Wartość biznesowa niższa (infrastruktura,
   nie różnicująca logika), ale rozjazd jest największy ilościowo (787 vs 557
   metod w całym repo) i systemowo nierozwiązany od 12 miesięcy.

---

## Podsumowanie

W repo nie ma spisanego PRD ani tech-stacku — destylacja opiera się na README,
istniejącej mapie repo (historia gita + graf importów) i jednym już wykonanym,
zweryfikowanym strukturalnie badaniu przepływu ABAC, uzupełnionym bezpośrednim
czytaniem `server/public/model`. Core domeny to klasyczna konwersacja
(Channel/Post/Team/User) oraz — jako różnicująca inwestycja bieżącego okna —
Attribute-Based Access Control (`AccessControlPolicy`, reguły CEL, maskowanie
atrybutów), którego silnik decyzyjny (PDP) celowo żyje poza tym repozytorium
jako szew enterprise. Property Fields (CPA/PSAv2) i RBAC (Permission/Role/Scheme)
to Supporting — infrastruktura, na której ABAC się opiera. Kontrakt REST
(Client4 w Go/TS + OpenAPI YAML) jest Generic, ale najbardziej rozjechaną
powierzchnią w repo — dla ABAC ma nie trzy, a cztery ręcznie utrzymywane kopie,
z których żadna nie zgadza się z pozostałymi co do kształtu `AccessControlPolicy`.
Najcenniejszy wniosek: sam niezmiennik bezpieczeństwa ABAC
(`masked_rule_deleted` — zakaz cichego usuwania reguł niosących ukryte wartości)
jest egzekwowany tylko na jednej z dwóch istniejących ścieżek zapisu, co czyni go
najpilniejszym kandydatem do refaktoru — nie dlatego, że brakuje dokumentacji,
ale dlatego, że sam agregat domenowy nie broni własnego niezmiennika niezależnie
od tego, kto go wywołuje.
