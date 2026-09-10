# Artefakt 3 — Kontekst kontrybutorów: kto wie co

**Repo:** mattermost, HEAD `87168644a4` · **Okno:** 12 miesięcy (2025-09-10 → 2026-09-10)
**Metoda:** wyłącznie `git log` (autorzy, tematy, wzorce commitów). Dane publiczne z historii repo.

---

## 0. Wybór obszaru

**Obszar: potrójna kopia kontraktu REST.**

| Kod | Ścieżka | Rola |
|---|---|---|
| **A** | `server/public/model/` | źródło prawdy (Go) — 204 pliki, `client4.go` = 787 metod |
| **B** | `webapp/platform/types/` + `webapp/platform/client/` | lustro TS — `client4.ts` = 557 metod |
| **C** | `api/v4/source/` | spec OpenAPI — `definitions.yaml` = 5742 linie |

### Dlaczego ten obszar

| Kryterium | Dowód |
|---|---|
| **Centralny** | Artefakt 2: `public/model` ma fan-in 129 i **84% zasięgu tranzytywnego**; `@mattermost/types` ma 2235 importów w webappie — więcej niż `react` (2101) |
| **Centralny (niezależnie)** | Artefakt 1: `public/model ↔ platform/types` = **77% pewności współzmienności**, najwyższa para w repo |
| **Podejrzany** | Artefakt 2: **brak generatora Go→TS**; zmierzona rozjazdka **787 vs 557 metod** |
| **Kontrast** | Ten sam zespół rozwiązał identyczny problem dla wtyczek generatorem + `plugin-checker` w `check-style` |

**Pytanie badawcze:** skoro nic nie pilnuje spójności trzech kopii narzędziowo — czy pilnują jej ludzie?

---

## 1. Punkt wyjścia: `CODEOWNERS` nie obejmuje tego obszaru

Cały plik ma **19 linii**. Kod pokrywają trzy wpisy:

```
/webapp/channels/src/packages/mattermost-redux/src/store/configureStore.ts  @hmhealey
/server/channels/app/authentication.go   @mattermost/product-security
/server/channels/app/authorization.go    @mattermost/product-security
```

Pozostałe 13 wpisów to **dokumentacja** (`docs/main/...` → `@mattermost/release-eng`, `@mattermost/release-managers`).

> **Ani `server/public/model/`, ani `webapp/platform/types/`, ani `api/v4/source/` nie mają
> przypisanego właściciela.** Strony changelogu w dokumentacji mają wymuszony review;
> kontrakt REST, od którego zależy 84% serwera i cały webapp — nie.

---

## 2. Kto pracował przy obszarze (12 miesięcy)

### 2a. Rozkład wg trzech kopii

| # | **A** `public/model` (329 commitów) | **B** `platform/types+client` (172) | **C** `api/v4/source` (89) |
|---|---|---|---|
| 1 | Jesse Hallam — 45 | Harrison Healey — 13 | Nick Misasi — 10 |
| 2 | Ben Schumacher — 40 | Pablo Vélez — 11 | Miguel de la Cruz — 7 |
| 3 | *cursor[bot] — 24* | Scott Bishel — 10 | Harshil Sharma — 7 |
| 4 | Pablo Vélez — 22 | Ibrahim Serdar Acikgoz — 10 | Pablo Vélez — 5 |
| 5 | Devin Binnie — 18 | Devin Binnie — 9 | Jesse Hallam — 5 |
| 6 | Nick Misasi — 16 | Nick Misasi — 8 | Ibrahim S. Acikgoz — 5 |
| 7 | Ibrahim S. Acikgoz — 16 | *cursor[bot] — 6* | Devin Binnie — 5 |
| 8 | Ben Cooke — 16 | Jesse Hallam — 6 | Daniel Espino García — 5 |

**Trzy różne czołówki.** Lider kopii A (Jesse Hallam, 45) jest ósmy w B i piąty w C.
Lider kopii B (Harrison Healey, 13) nie występuje w pierwszej ósemce ani A, ani C.

### 2b. Udział obszaru w całej pracy autora

| Autor | commity w repo (12m) | w strefie kontraktu | udział |
|---|---:|---:|---:|
| Scott Bishel | 23 | 14 | **60%** |
| Ben Schumacher | 92 | 43 | 46% |
| David Krauser | 32 | 15 | 46% |
| Devin Binnie | 50 | 18 | 36% |
| Ibrahim Serdar Acikgoz | 52 | 19 | 36% |
| Pablo Vélez | 79 | 27 | 34% |
| Harshil Sharma | 55 | 16 | 29% |
| Nick Misasi | 79 | 22 | 27% |
| Jesse Hallam | 194 | 50 | 25% |
| Alejandro García Montoro | 60 | 14 | 23% |
| Harrison Healey | 103 | 20 | 19% |

**Nikt nie jest „osobą od kontraktu".** Wszyscy przechodzą przez ten obszar przy okazji
pracy nad funkcjonalnościami. Nawet najwyższy udział (Scott Bishel, 60%) to 14 commitów rocznie.

---

## 3. Tematy powtarzające się u konkretnych osób

Z treści commitów w strefie kontraktu wyłania się siedem wyraźnych specjalizacji:

| Osoba | Powtarzający się temat | Charakterystyczne commity |
|---|---|---|
| **Jesse Hallam** | **konfiguracja i wygaszanie flag** — „graduate X out of Experimental", usuwanie martwych ustawień, rendering React | `Graduate Hardened Mode out of Experimental Features`, `Graduate email notification settings to Site Configuration`, `Remove EnableExperimentalLocales`, `MM-69706: remove the ChannelBookmarks feature flag compatibility shim`, `feat: enable React concurrent rendering by default` |
| **Ben Schumacher** | **Personal Access Tokens, wtyczki, support packet, WebSocket eventy jobów** | `[MM-69561] rotate (regenerate) Personal Access Tokens`, `[MM-69075] Revoke non-compliant PATs`, `[MM-69215] warn users before their PATs expire`, `[MM-70279] plugin upload overwrite review panel`, `[MM-69403] Push WebSocket events on job status changes` |
| **Harshil Sharma** | **content flagging / data spillage** (pionowa funkcjonalność end-to-end) | `Data spillage exposure radius report generation`, `Content flagging file downloads`, `Anonymous URLs`, `Flag post API`, `(1) - Feature post delivery audit setting` |
| **Pablo Vélez** | **ABAC: team membership + channel attributes** | `MM 69100 - Team ABAC Membership`, `MM-70180 — Channel attributes`, `MM-69831 configurable interval for ABAC membership sync schedulers`, `MM-68283 render-time ABAC decisions for file upload/download` |
| **Nick Misasi** | **ABAC/PAP + Recaps + agenty + licencjonowanie** | `ABAC: plugin-keyed resource types, trusted plugin PAP/CEL APIs, AuthZEN-style decision API`, `[MM-67163] Scheduled Recaps`, `[MM-67605] DCR redirect URI allowlist for OAuth`, `Add Default Agent Support` |
| **Devin Binnie** | **Session Attributes + system property fields + edytor polityk** | `Session Attributes MVF - Server-work`, `[MM-70189] operators for CIDR and version checks in the simple policy editor`, `[MM-68648] GetForGroup in the Property System`, `[MM-70250] Remove unused GET bulk reactions endpoint` |
| **Ibrahim Serdar Acikgoz** | **ABAC fazy 2–6 + discoverable private channels + audit logging** | `[MM-69333..69337] Native ABAC user attributes (Phase 2/4/5/6)`, `MM-68763 Discoverable Private Channels`, `MM-69612 opt-in EnableAuditLogging for ABAC` |

### Obserwacja: cztery z siedmiu specjalizacji to ABAC

Pablo Vélez, Nick Misasi, Devin Binnie i Ibrahim Serdar Acikgoz pracują nad różnymi
warstwami tej samej inicjatywy (członkostwo zespołów / atrybuty sesji / property fields /
natywne atrybuty użytkownika). To domyka obserwację z Artefaktu 1 (`abac` = 31 wystąpień
w tematach commitów, klaster 655 dotknięć): **aktywność w strefie kontraktu w tym oknie
jest w dużej mierze produktem ubocznym ABAC-a.**

---

## 4. Kto faktycznie synchronizuje trzy kopie

To sedno tego kroku. 440 commitów dotknęło strefy kontraktu w 12 miesięcy.
Rozkład wg tego, ile kopii ruszyły **w jednym commicie**:

| Kombinacja | Commitów | Udział | Interpretacja |
|---|---:|---:|---|
| **A** (tylko Go) | **212** | **48%** | zmiana modelu bez lustra i bez spec |
| A+B | 87 | 20% | Go + TS, **bez spec** |
| B (tylko TS) | 49 | 11% | praca webappowa lub nadrabianie zaległości |
| **A+B+C** | **41** | **9%** | **pełna synchronizacja** |
| A+C | 24 | 5% | Go + spec, **bez TS** |
| C (tylko spec) | 20 | 5% | nadrabianie dokumentacji |
| B+C | 7 | 2% | |

### Zawężenie do samej powierzchni REST (`client4.go`)

50 commitów w 12 miesięcy dotknęło `client4.go`. Czy zaktualizowały lustro?

| Wzorzec | Commitów | Udział |
|---|---:|---:|
| `client4.go` + `client4.ts` + spec | **17** | **34%** |
| `client4.go` + spec (bez TS) | 14 | 28% |
| **`client4.go` sam** | **15** | **30%** |
| `client4.go` + TS (bez spec) | 4 | 8% |

> **Tylko co trzecia zmiana klienta REST propaguje się do wszystkich trzech reprezentacji.
> Prawie co trzecia nie propaguje się nigdzie.** To mechanizm, który wytwarza zmierzoną
> w Artefakcie 2 różnicę 230 metod — widoczny wprost w historii.

### Kto robi pełny 3-stronny sync (41 commitów ABC)

```
5  Harshil Sharma          2  Scott Bishel            1  Rahim Rahman
4  Pablo Vélez             2  David Krauser           1  Miguel de la Cruz
4  Nick Misasi             2  Daniel Espino García    1  Julien Tant
4  Devin Binnie            1  cursor[bot]             1  Joshua D Schoep
3  Ibrahim S. Acikgoz      1  Guillermo Vayá          1  Jesse Hallam
3  Ben Schumacher          1  Elias Nahum             1  Alejandro García Montoro
3  Ben Cooke
```

**19 różnych osób, rekordzista 5 razy w roku.** Nie ma nikogo, kto robiłby to rutynowo.

### Kto zmienia tylko model Go (212 commitów)

```
38  Jesse Hallam        10  Alejandro García Montoro    8  Devin Binnie
33  Ben Schumacher       9  Pablo Vélez                 8  David Krauser
16  cursor[bot]          8  Nick Misasi                 7  Ben Cooke
                         8  Doug Lauder
```

Jesse Hallam i Ben Schumacher odpowiadają za **33% zmian jednostronnych**.
Kontekst częściowo je usprawiedliwia: Hallam pracuje głównie nad `config.go`
(graduacja flag), gdzie wiele pól nie ma odpowiednika w kliencie REST. Ale ten sam
wzorzec dotyczy commitów, które **dodają powierzchnię API** — patrz §6b.

---

## 5. Rozproszona czy skupiona? — pomiar

**Rozproszona, skrajnie.** Metryka: ile osób pokrywa 50% commitów (bez botów).

| Obszar | Autorzy | Commity | top1 | top3 | top5 | Bus factor (50%) |
|---|---:|---:|---:|---:|---:|---:|
| **A** `public/model` | 44 | 329 | 13% | 32% | 42% | **7** |
| **B** `platform/types+client` | 33 | 172 | 10% | 28% | 40% | **7** |
| **C** `api/v4/source` | 28 | 89 | 11% | 26% | 38% | **8** |
| ↳ `client4.go` | 20 | 49 | 18% | 38% | 51% | 5 |
| ↳ `client4.ts` | 24 | 68 | 14% | 35% | 52% | 5 |
| ↳ `config.go` | 23 | 75 | 21% | 42% | 57% | 4 |
| **REF:** `app/access_control.go` | **9** | 37 | **35%** | **75%** | **89%** | **2** |
| **REF:** `store/store.go` | 25 | 83 | 18% | 37% | 50% | 5 |

**W strefie kontraktu pracowało 49 z 119 kontrybutorów repo (41%) w ciągu roku.**

### Dwa przeciwstawne profile ryzyka w jednym repo

| | Kontrakt REST | Klaster ABAC (`access_control.go`) |
|---|---|---|
| Autorzy (12m) | 44 / 33 / 28 | **9** |
| top3 | 26–32% | **75%** |
| Bus factor | 7–8 | **2** |
| Typ ryzyka | 🔴 **brak spójności** — 41% repo przechodzi tędy, nikt nie odpowiada za całość | 🔴 **bus factor** — wiedza w dwóch głowach |
| Objaw | 787 vs 557 metod, 30% zmian bez propagacji | brak zapasowego eksperta |
| Lekarstwo | narzędzie (generator/checker), nie osoba | dokumentacja i rozszerzenie zespołu |

> To nie jest obszar z bus factorem. To obszar bez właściciela — a to inny,
> trudniejszy problem: **każdy go dotyka, nikt go nie pilnuje.**

---

## 6. Co przeczytać przed zmianą

### 6a. Wzorce do naśladowania — pełny 3-stronny sync

Siedemnaście commitów w 12 miesięcy zaktualizowało `client4.go` + `client4.ts` + spec.
To jedyne dostępne przykłady „jak to się robi kompletnie":

| Data | Autor | PR |
|---|---|---|
| 2026-09-07 | Harshil Sharma | `#37882` Feature post delivery audit setting |
| 2026-08-29 | Pablo Vélez | `#36820` MM-68283 render-time ABAC decisions for file upload/download |
| 2026-08-19 | Harshil Sharma | `#37809` Data spillage exposure radius report generation |
| 2026-07-17 | Nick Misasi | `#37458` admin-locked profile fields, pre-provisioned names on invites |
| 2026-07-10 | Ben Schumacher | `#37295` MM-69561 rotate (regenerate) Personal Access Tokens |
| 2026-07-02 | Ben Schumacher | `#37030` MM-69075 Revoke non-compliant PATs |
| 2026-06-01 | Ben Schumacher | `#34877` MM-67113 license preview/diff view |
| 2026-04-29 | David Krauser | `#36250` MM-68464 system object type for property fields and values |
| 2025-12-11 | Ibrahim S. Acikgoz | `#34703` MM-61758 Burn on read |
| 2025-11-20 | Rahim Rahman | `#34264` Magic link (passwordless) authentication for guests |
| 2025-10-02 | Harshil Sharma | `#33765` Flag post API |

**Harshil Sharma ma 5 z 17** — najbardziej konsekwentna osoba w tym obszarze.
Ich PR-y (`#33765`, `#37809`, `#37882`) są najlepszym dostępnym wzorcem.

### 6b. Decyzje architektoniczne o `Client4` — przeczytać obowiązkowo

| Data | Autor | PR | Dlaczego ważne |
|---|---|---|---|
| 2025-10-07 | Ben Schumacher | `#31805` **MM-64633 Rewrite Go client using Generics** | przepisanie całego klienta Go na generyki — zmienia sposób pisania każdej nowej metody |
| 2026-01-29 | Alejandro García Montoro | `#34499` **MM-65970 New way to build routes in Client4** | nowa konwencja budowania tras; starsze przykłady w kodzie są nieaktualne |
| 2026-01-20 | Alejandro García Montoro | `#34951` MM-67119 Remove unused `Channel.Etag` | precedens usuwania pola z modelu publicznego |
| 2026-06-30 | SomakD | `#37045` `GetUsersNotInChannelWithOptions` — panic przy `options == nil` | **edge case:** dostęp do `options.Etag` poza strażą `if options != nil`; klasa błędu do sprawdzenia w podobnych metodach |

### 6c. Fala breaking changes v12.0 — kontekst wydania

Obszar jest w trakcie skoordynowanego usuwania przestarzałych elementów. Przed dodaniem
czegokolwiek do kontraktu warto wiedzieć, co właśnie z niego wypada:

| Data | Autor | PR |
|---|---|---|
| 2026-09-10 | Amy Blais | `#38382` Update v12.0 and Mobile v2.45 deprecation notices |
| 2026-08-31 | Devin Binnie | `#38174` MM-70250 Remove unused GET bulk reactions endpoint |
| 2026-08-27 | Alejandro García Montoro | `#37743` MM-70018 Remove unused config fields for v12 |
| 2026-08-18 | Scott Bishel | `#37759` MM-68396 Remove deprecated dialog date/datetime fields for v12.0 |
| 2026-08-18 | Felipe Martin | `#37999` Remove deprecated built-in Slack import API and CLI |
| 2026-08-17 | Ben Schumacher | `#37163` MM-67868 Remove deprecated Slack compatibility type aliases |
| 2026-08-17 | Ben Schumacher | `#37167` MM-67157 Remove `format` parameter requirement from client license endpoint |
| 2026-09-02 | Jesse Hallam | `#38291` MM-69706 Remove the ChannelBookmarks feature flag compatibility shim |

### 6d. Edge case'y i decyzje warte lektury

| PR | Autor | Rzecz |
|---|---|---|
| `#37505` | cursor[bot] | MM-66243 — pomijanie `last_viewed_at`/`last_update_at` zamiast zwracania `-1` dla innych użytkowników; **zmiana kształtu odpowiedzi API bez zmiany wersji** |
| `#38054` | cursor[bot] | MM-69945 — ujednolicenie walidacji typu posta między ścieżką post i scheduled post (dwie ścieżki rozjechały się) |
| `#38296` | cursor[bot] | MM-69882 — odsprzęgnięcie pin/unpin od `ServiceSettings.PostEditTimeLimit`; commit zawiera zabezpieczenie `PostPatch.IsEmpty` z asercją liczby pól, „so a new field cannot slip past unnoticed" — wzorzec obrony przed cichym rozjazdem |
| `#37917` | cursor[bot] | MM-70213 — dopuszczenie własnych schematów URI w walidacji DCR redirect URI |
| `#37509` | Nick Misasi | ABAC: plugin-keyed resource types + AuthZEN-style decision API — nowa powierzchnia publiczna dla wtyczek |
| `#33885` | Jesse Hallam | MM-64395 — usunięcie nieużywanego `searchArchivedChannelsForTeam`; wzorzec pełnego usunięcia endpointu |

### 6e. Martwa gałąź: `api/v5`

Artefakt 2 wykrył w `channels/api4/api.go` zarejestrowany subrouter `APIRoot5` (`api/v5`).
Historia pokazuje, że wszedł z commitem **`#22553` „Mono repo -> Master" (2023-03-22, Doug Lauder)**,
a `model.APIURLSuffixV5` z **`#23345` „Expose public/ API as submodule" (2023-05-10, Jesse Hallam)**.
W `api.go` są **2 odwołania do `APIRoot5`** i **żadnej aktywności w oknie 12 miesięcy**.
Traktować jako placeholder, nie jako ścieżkę migracji.

---

## 7. Dwie zmiany w składzie, które trzeba znać

### 7a. `cursor[bot]` — skokowy wzrost udziału agenta AI

| Okres | Commity w strefie kontraktu |
|---|---:|
| miesiące 12–3 wstecz | **1** |
| ostatnie 3 miesiące | **23** |

W całym repo: **135 commitów w 12 miesięcy**, z czego 24 w strefie kontraktu.
W ostatnim kwartale to **drugi co do wielkości „kontrybutor" tego obszaru**, zaraz za
Jesse Hallam (28).

Metadane autorstwa tych commitów:

```
Author:    cursor[bot] <206951365+cursor[bot]@users.noreply.github.com>
Committer: GitHub <noreply@github.com>
Trailery:  Co-authored-by: mattermost-code <matty-code@mattermost.com>   (480×)
           Co-authored-by: Cursor Agent <cursoragent@cursor.com>          (119×)
```

Konto `mattermost-code` jest współautorem 480 commitów — to konto serwisowe, nie osoba.
Imienni współautorzy pojawiają się rzadko (Maria A Nunez 9, Devin Binnie 8,
Miguel de la Cruz 5, Nick Misasi 4).

> **Praktyczna konsekwencja:** dla rosnącej części zmian w tym obszarze `git log` nie wskaże
> człowieka, którego można zapytać o kontekst. Kto realnie prowadził te PR-y, wynika
> z review na GitHubie, nie z historii gita. *(To wniosek z metadanych commitów —
> nie znam wewnętrznego procesu Mattermosta.)*

### 7b. Wyraźne wygaszenia aktywności (kwartał do kwartału)

| Osoba | mies. 12–3 | ostatnie 3 mies. |
|---|---:|---:|
| Jesse Hallam | 22 | **28** ▲ |
| cursor[bot] | 1 | **23** ▲▲ |
| Pablo Vélez | 15 | 12 |
| Ben Schumacher | 31 | **12** ▼ |
| Nick Misasi | 16 | **6** ▼ |
| Harrison Healey | 16 | **4** ▼ |
| Harshil Sharma | 14 | **2** ▼▼ |
| Miguel de la Cruz | 12 | **2** ▼▼ |
| Maria A Nunez | 10 | **2** ▼▼ |
| David Krauser | 12 | **3** ▼ |

Uwaga na kombinację: **Harshil Sharma to autor 5 z 17 pełnych synchronizacji kontraktu,
a jego aktywność spadła z 14 do 2 commitów kwartalnie.** Osoba z najlepszym wzorcem
pracy w tym obszarze jest w nim obecnie najmniej aktywna.

Spadki mogą oznaczać rotację, zmianę zespołu albo po prostu inny projekt — historia gita
tego nie rozstrzyga.

---

## 8. Wnioski

1. **Obszar nie ma właściciela — i to jest przyczyna, nie objaw.**
   `CODEOWNERS` (19 linii) pokrywa strony changelogu w dokumentacji, ale nie pokrywa
   kontraktu REST, od którego zależy 84% pakietów serwera i cały webapp.

2. **Wiedza jest skrajnie rozproszona.** 49 z 119 kontrybutorów repo (41%) dotknęło
   tego obszaru w ciągu roku; top3 to zaledwie 26–32% commitów; bus factor 7–8.
   To odwrotność typowego ryzyka — nie brakuje ludzi, brakuje **jednego punktu odpowiedzialności**.

3. **Rozjazd kontraktu jest widoczny wprost w historii, nie tylko w kodzie.**
   48% zmian w strefie dotyka tylko modelu Go; z 50 zmian `client4.go` tylko **34%**
   propaguje się do wszystkich trzech kopii, a **30% nie propaguje się nigdzie**.
   Różnica 787 vs 557 metod z Artefaktu 2 to zakumulowany efekt tego wzorca.

4. **Trzy kopie mają trzy różne czołówki autorów.** Lider `public/model` jest ósmy
   w `platform/types`; lider `platform/types` nie występuje w czołówce ani modelu, ani spec.
   Nikt nie ogląda całości kontraktu regularnie.

5. **Repo zawiera dowód, że organizacja umie ten problem rozwiązać.** Kontrakt wtyczek
   ma generator (`make pluginapi`) i weryfikator (`plugin-checker` wpięty w `check-style`).
   Dokumentacja ma nazwaną, numerowaną kampanię uzgadniania rozjazdu (Eva Sarafianou,
   `docs(P13)`/`P14`/`P15` — „reconcile authored docs drift"). **Kontrakt REST nie ma
   ani generatora, ani kampanii — w 12 miesiącach zero commitów mówiących „sync spec"
   albo „reconcile client4".**

6. **Aktywność w tym obszarze jest w dużej mierze produktem ubocznym ABAC-a.**
   Cztery z siedmiu głównych specjalizacji (Vélez, Misasi, Binnie, Acikgoz) to różne
   warstwy tej samej inicjatywy. Kiedy ABAC się domknie, ruch w strefie kontraktu spadnie —
   i wtedy nikt nie będzie miał świeżego kontekstu.

7. **Dwa przeciwstawne profile ryzyka w jednym repo.** Kontrakt: 44 autorów, top3 32%
   → ryzyko niespójności. ABAC (`access_control.go`): 9 autorów, top3 **75%**, bus factor **2**
   → ryzyko utraty wiedzy. Wymagają odwrotnych działań: pierwszy narzędzia, drugi ludzi.

8. **Jeżeli szukasz jednej osoby do konsultacji przed zmianą kontraktu:**
   dla powierzchni REST — Ben Schumacher (46% pracy w tej strefie, autor przepisania
   klienta na generyki, 3 pełne synchronizacje); dla dyscypliny propagacji — Harshil Sharma
   (5 z 17 pełnych synchronizacji, choć obecnie mało aktywny); dla konfiguracji
   i cyklu życia flag — Jesse Hallam (50 commitów, najbardziej aktywny obecnie).

---

## 9. Ograniczenia tej analizy

- **Historia gita nie pokazuje review.** Repo scala PR-y przez GitHuba (`Committer: GitHub`),
  więc recenzenci — często najlepsze źródło wiedzy o obszarze — są niewidoczni.
  Pełny obraz wymaga danych z GitHub API (`gh pr list --json reviews`), nieużytych tutaj.
- **Liczba commitów ≠ wiedza.** Osoba z jednym głębokim PR-em architektonicznym może
  wiedzieć więcej niż osoba z trzydziestoma drobnymi zmianami.
- **Squash merge zniekształca liczniki.** Duża funkcjonalność scalona jako jeden commit
  waży tyle samo co literówka.
- **Nazwiska pochodzą z pola `%an`** — ta sama osoba może występować pod kilkoma zapisami
  (widoczne np. `Maria A Nunez` / `maria.nunez` w trailerach). Nie przeprowadzono deduplikacji.
- **Podział na commity „jednostronne" nie odróżnia zaniedbania od poprawnej decyzji.**
  Część zmian tylko w `public/model` (np. `Upgrade to Go 1.26.2`, poprawka nil-pointera,
  przepisanie na generyki) nie dotyczy kontraktu i słusznie nie ma lustra. Rząd wielkości
  (30% zmian `client4.go` bez żadnej propagacji) pozostaje jednak informatywny.
- **Aktywność `cursor[bot]` zinterpretowano wyłącznie z metadanych commitów.** Nie wiem,
  jak wygląda proces prowadzenia i review tych PR-ów w Mattermoście.
- **Spadki aktywności nie są dowodem odejścia** — mogą oznaczać urlop, zmianę zespołu
  albo pracę w innym repo organizacji.
