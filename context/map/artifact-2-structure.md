# Artefakt 2 — Struktura: zależności, entry pointy, cykle i lokalne centra

**Repo:** mattermost, HEAD `87168644a4`
**Pytanie prowadzące:** co realnie zależy od czego — gdzie jest blast radius, gdzie lokalne centra, gdzie cienkie wejścia.
**Wejście:** Artefakt 1 (terytorium) — sprawdzamy, czy gorące obszary z historii pokrywają się z centrami grafu.

---

## 0. Wybór narzędzia — co sprawdzono przed instalacją

### Stan zastany

| Aspekt | Ustalenie |
|---|---|
| Package manager | **npm 12 / Node 24** (`webapp/package.json`: `engines.npm ^11`, `node ^24`) |
| Workspaces | npm workspaces, 7 pakietów: `channels`, `platform/{client,components,eslint-plugin,mattermost-redux,shared,types}` |
| Drugi workspace | `e2e-tests/playwright` — osobny root npm workspaces, linkuje webapp przez `file:` |
| Moduły Go | **dwa**: `github.com/mattermost/mattermost/server/v8` i `github.com/mattermost/mattermost/server/public` (+ 6 pomocniczych: `api/server`, `tools/*`, `docs/site/scripts/*`) |
| Aliasy ścieżek | `channels/tsconfig.json`: `baseUrl: "./src"` (bare importy `components/…`, `utils/…`, `actions/…`), `paths: {"mattermost-redux/*": ["packages/mattermost-redux/src/*"], "@mui/styled-engine": [styled-engine-sc]}` |
| Project references | `channels` → `platform/{client,components,types}`; `platform/mattermost-redux` → `platform/{types,client}` |
| **Istniejące narzędzie do grafu** | **brak** — `grep -riE 'madge\|dependency-cruiser\|skott\|goda\|go-mod-graph'` po `package.json`, `Makefile`, `*.mk`, workflow'ach i `webapp/scripts` nie zwraca nic |
| Dostępne toolchainy | `node v24.19.0`, `npm 12.0.2`; **`go` niedostępne w PATH**; **`webapp/node_modules` nie zainstalowane** |

### Decyzja: analiza statyczna na własnym parserze, bez instalacji

- `madge` / `dependency-cruiser` wymagałyby pełnego `npm install` monorepo (7 workspace'ów,
  lockfile ~ dziesiątki tysięcy linii) — długo, ciężko i **modyfikuje drzewo repo**.
- `go list -deps` / `go mod graph` odpadają — brak toolchainu Go.
- Aliasy w tym repo są **proste i deklaratywne** (`baseUrl` + jeden `paths`), a importy Go
  są jednoznaczne — graf da się odtworzyć wiarygodnie z samego tekstu.

**Zbudowano dwa grafy własnym parserem** (`grep` + `awk`, bez zapisu do repo):

| Graf | Źródło | Węzły | Krawędzie |
|---|---|---:|---:|
| **Go** | 1174 plików `.go` (bez `_test.go`, bez `mocks/`), 7273 importy → 2795 wewnętrznych | 164 pakiety | 677 |
| **TypeScript** | 2799 plików `.ts/.tsx` w `webapp/channels/src` (bez `.test.*`), 22 535 importów | 384 moduły | 2156 |

Granulacja TS: `components/<nazwa>` (depth 2), reszta depth 1, `packages/mattermost-redux` jako jeden węzeł.

---

## 1. Warstwa Go — graf wymuszenie acykliczny, sprzężenie ucieka do runtime

### 1a. Fan-in: kto jest importowany (liczba pakietów)

| Fan-in | Pakiet | Rola |
|---:|---|---|
| **129** | `public/model` | **hub absolutny** — modele, `Client4`, `Config` |
| 74 | `public/shared/mlog` | logger |
| 42 | `public/shared/request` | kontekst żądania |
| 41 | `channels/store` | interfejs warstwy danych |
| 35 | `public/plugin` | kontrakt pluginów |
| 34 | `channels/jobs` | scheduler |
| 18 | `channels/utils` | |
| **16** | `einterfaces` | **inwersja zależności enterprise** |
| 15 | `platform/shared/filestore` | |
| 11 | `channels/app` | |

### 1b. Fan-out: kto zależy od najwięcej rzeczy

| Fan-out | Pakiet |
|---:|---|
| **72** | `channels/app` |
| 27 | `channels/api4` |
| 23 | `channels/app/platform` |
| 16 | `channels/testlib` |
| 15 | `cmd/mattermost/commands` |
| 12 | `cmd/mmctl/commands`, `enterprise/elasticsearch/*` |
| 11 | `channels/store/sqlstore` |

> **`channels/app`: fan-out 72, fan-in 11.** To klasyczny *god package* — wie o wszystkim,
> prawie nikt nie wie o nim. Zgadza się z Artefaktem 1: 1355 dotknięć, najbardziej
> aktywny katalog serwera.

### 1c. Blast radius (tranzytywne domknięcie wsteczne, ze 164 pakietów)

| Pakiet | Bezpośrednio | Tranzytywnie | Zasięg | Wzmocnienie |
|---|---:|---:|---:|---:|
| **`public/model`** | 129 | **138** | **84%** | 1,07× |
| `public/shared/request` | 42 | 72 | 44% | 1,7× |
| `public/plugin` | 35 | 68 | 41% | 1,9× |
| `channels/store` | 41 | 66 | 40% | 1,6× |
| **`einterfaces`** | 16 | **59** | 36% | **3,7×** |
| `channels/utils` | 18 | 30 | 18% | 1,7× |
| `platform/shared/filestore` | 15 | 26 | 16% | 1,7× |
| `config` | 5 | 18 | 11% | 3,6× |
| `channels/app/platform` | 7 | 17 | 10% | 2,4× |
| `channels/app` | 11 | 15 | 9% | 1,4× |

Dwie obserwacje:

- **`public/model` jest nasycone** (wzmocnienie 1,07) — prawie wszyscy, którzy go używają,
  importują go bezpośrednio. Zmiana tu jest widoczna od razu w 129 miejscach.
- **`einterfaces` i `config` są zdradliwe** (3,7× i 3,6×) — wyglądają na małe (16 i 5
  bezpośrednich importerów), ale ciągną za sobą trzy–cztery razy więcej pakietów.
  To pakiety, gdzie „mała zmiana" myli oceną zasięgu.

### 1d. Cykli nie ma — i to jest właśnie informacja

```
2-cykle w grafie pakietów Go:   0
public/ importuje server/v8:    0 krawędzi  (granica modułów czysta)
```

Go **kompilacyjnie zabrania** cykli między pakietami, więc zero to nie zasługa architektury,
tylko ograniczenie języka. Ciekawe jest **gdzie sprzężenie uciekło**:

#### Ukryty cykl przez rejestr runtime

Statycznie: `enterprise/*` → `channels/app` (13 importów), a `channels/app` → `enterprise` **nie istnieje**.
Faktycznie zależność jest dwukierunkowa, tylko domknięta w czasie wykonania:

```
channels/app/platform/enterprise.go:19  func RegisterElasticsearchInterface(f func(*PlatformService) searchengine.SearchEngineInterface)
channels/app/platform/enterprise.go:37  func RegisterLicenseInterface(f func(*PlatformService) einterfaces.LicenseInterface)
channels/app/platform/service.go:560    ps.SearchEngine.RegisterElasticsearchEngine(elasticsearchInterface(ps))
```

a wpięcie następuje przez **blank import w entry poincie**:

```go
// cmd/mattermost/main.go  (całe 23 linie)
import (
    "os"
    "github.com/mattermost/mattermost/server/v8/cmd/mattermost/commands"
    _ "github.com/mattermost/mattermost/server/v8/channels/app/slashcommands"
    _ "github.com/mattermost/mattermost/server/v8/channels/app/oauthproviders/gitlab"
    _ "github.com/mattermost/mattermost/server/v8/enterprise"   // ← cała gałąź enterprise przez init()
)
```

**Konsekwencja praktyczna:** graf importów nie powie ci, że zmiana sygnatury w
`einterfaces/*.go` (21 plików) zepsuje `enterprise/`. Kompilator też nie — dopóki nie
zbudujesz binarki enterprise. To jedyne miejsce w warstwie Go, gdzie „idź za importami"
zawodzi.

### 1e. Cienkie wejścia vs głębokie centra (Go)

| Plik | Linie | Charakter |
|---|---:|---|
| `cmd/mattermost/main.go` | **23** | 🚪 cienkie wejście — 3 blank importy uruchamiają cały serwer |
| `channels/api4/api.go` | 544 | 🚪 tablica routingu: `Root` / `APIRoot` (`api/v4`) / **`APIRoot5` (`api/v5`)** + 764 zarejestrowane endpointy |
| `channels/store/store.go` | 1516 | 🧠 centrum — **63 interfejsy, 1024 metody** |
| `channels/app/server.go` | 2093 | 🧠 bootstrap runtime |
| `public/model/config.go` | 5766 | 🧠 schemat konfiguracji |
| `public/model/client4.go` | **8541** | 🧠 **787 metod** — cała powierzchnia REST w jednym pliku |

`main.go` (23 linie) → `client4.go` (8541 linii) to rozpiętość, o którą chodzi w tym kroku:
najcieńsze wejście prowadzi do najgłębszego centrum w kilku skokach.

---

## 2. Warstwa TypeScript — jeden wielki cykl

### 2a. Fan-in modułów (liczba modułów importujących, z 384)

| Fan-in | Moduł |
|---:|---|
| **234** | `utils` |
| **223** | `packages/mattermost-redux` |
| **187** | `types` |
| 144 | `actions` |
| 114 | `components/widgets` |
| 105 | `selectors` |
| 78 | `components/common` |
| 40 | `components/external_link` |
| 29 | `plugins`, `components/loading_screen` |

### 2b. Fan-in pojedynczych plików (liczba plików z 2799)

| Plików | Moduł | % bazy |
|---:|---|---:|
| **733** | `utils/constants` (2258 linii) | **26%** |
| **695** | `types/store` | 25% |
| 253 | `mattermost-redux/selectors/entities/general` | 9% |
| 200 | `utils/utils` | 7% |
| 172 | `mattermost-redux/selectors/entities/preferences` | 6% |
| 162 | `actions/views/modals` | 6% |
| 142 | `mattermost-redux/client` | 5% |

### 2c. Fan-in pakietów `@mattermost/*` (liczba importów)

| Importów | Pakiet |
|---:|---|
| **2235** | `@mattermost/types` |
| 453 | `@mattermost/shared` |
| 227 | `@mattermost/compass-icons` |
| 106 | `@mattermost/components` |
| **31** | `@mattermost/client` ← patrz §2e |

> `@mattermost/types` (2235) jest importowane **częściej niż `react` (2101)**.
> To najbardziej zależna rzecz w całym webappie.

### 2d. Cykle — pierwszorzędne odkrycie tego kroku

```
krawędzie dwukierunkowe:   164 / 2156  (82 pary modułów)
największy SCC:            192 z 384 modułów  =  50% webappu
```

**Połowa modułów `webapp/channels/src` leży w jednej silnie spójnej składowej** — każdy
z tych 192 modułów jest osiągalny z każdego innego. Tranzytywne domknięcie wsteczne
dowolnego z nich wynosi **299 modułów niezależnie od tego, który wybierzesz**:

```
utils                      → 299        components/post_view      → 299
packages/mattermost-redux  → 299        components/admin_console  → 299
types                      → 299        stores                    → 299
```

**Blast radius jako metryka przestaje działać wewnątrz tej składowej.** Nie da się
powiedzieć „ta zmiana dotyka X modułów" — dotyka całego kłębka.

Moduły uczestniczące w największej liczbie 2-cykli:

| Cykli | Moduł |
|---:|---|
| 16 | `utils` |
| 15 | `actions` |
| 11 | `components/common` |
| 11 | `components/admin_console` |
| 7 | `plugins`, `components/post_view` |
| 5 | `selectors`, `components/widgets`, `components/advanced_text_editor` |
| 4 | `types` |

Przykłady par (pełna lista: 82):

```
actions            <->  utils                       utils           <->  components/at_mention
actions            <->  types                       selectors       <->  components/admin_console
actions            <->  stores                      hooks           <->  components/admin_console
actions            <->  plugins                     components/post_view <-> components/block_renderer
components/common  <->  components/admin_console    components/claim <-> components/login
```

**Wzorzec:** to nie są cykle „przypadkiem między dwoma komponentami". To cykle
**między warstwami** — `actions ↔ utils ↔ types ↔ selectors ↔ stores` tworzą rdzeń
kłębka, a komponenty wpinają się w niego w obie strony. Warstwowość jest zadeklarowana
strukturą katalogów, ale nie egzekwowana.

### 2e. Cienkie wejścia (TS) — trzy przykłady

| Plik | Linie | Za nim stoi |
|---|---:|---|
| `channels/src/root.tsx` | **26** | 🚪 webpack `entry` — cała aplikacja |
| `channels/src/entry.tsx` | 79 | 🚪 bootstrap |
| `packages/mattermost-redux/src/client/index.ts` | **18** | 🚪 **globalny singleton `new Client4()`** |

```ts
// packages/mattermost-redux/src/client/index.ts — całość istotnej treści
import {Client4 as ClientClass4, ...} from '@mattermost/client';
const Client4 = new ClientClass4();
export {Client4, ...};
```

To wyjaśnia paradoks z §2c: `@mattermost/client` ma tylko **31** bezpośrednich importów,
ale **142 pliki** importują `mattermost-redux/client`. Cały dostęp webappu do serwera
przechodzi przez **jedną, globalną, mutowalną instancję** utworzoną w 18-linijkowym pliku,
fronting dla klasy o **5530 liniach i 557 metodach**.

### 2f. Kontrast: `webapp/platform/*` jest czyste

W przeciwieństwie do `channels/src`, pakiety platformowe mają ścisłą warstwowość
bez ani jednego cyklu:

```
types  (liść — 0 importów wewnętrznych)
  ↑
  ├── client   (importuje wyłącznie @mattermost/types)
  ├── shared   (importuje wyłącznie @mattermost/types)
  └── components (importuje @mattermost/shared, @mattermost/compass-icons)
```

Granica `platform/` ↔ `channels/` jest **jedyną zdrową granicą w webappie**.

### 2g. Anomalia: `platform/mattermost-redux` nie ma własnego kodu

```json
// webapp/platform/mattermost-redux/tsconfig.json
"rootDir": "../../channels/src/packages/mattermost-redux/src",
"include": ["../../channels/src/packages/mattermost-redux/src/**/*"]
```

Publikowalny pakiet npm `mattermost-redux@12.0.0` (z `exports` na `./actions`,
`./selectors`, `./reducers`…) jest **wyłącznie skorupą buildową** nad źródłami leżącymi
wewnątrz aplikacji. Zależność biegnie odwrotnie niż sugeruje układ katalogów:
`platform/` (warstwa niżej) czyta z `channels/` (warstwa wyżej). To jedyne miejsce,
gdzie `platform/` łamie własną warstwowość.

---

## 3. Kontrakty między warstwami

### 3a. Cztery reprezentacje tego samego kontraktu

| # | Reprezentacja | Rozmiar | Egzekwowana? |
|---|---|---|---|
| 1 | `server/public/model/*.go` | 204 pliki, `client4.go` = **787 metod** | źródło prawdy |
| 2 | `webapp/platform/types/src/*.ts` | 63 pliki | ❌ ręcznie |
| 3 | `webapp/platform/client/src/client4.ts` | **557 metod** | ❌ ręcznie |
| 4 | `api/v4/source/*.yaml` (OpenAPI) | `definitions.yaml` = 5742 linie | ❌ ręcznie |

```
grep -riE 'tygo|go2ts|generate.*typescript' server/Makefile webapp/Makefile webapp/scripts
→ (brak wyników)
```

**Nie istnieje generator Go→TS.** Kontrakt jest przepisywany ręcznie trzy razy.

**Zmierzona rozjazdka:** `client4.go` ma 787 metod, `client4.ts` ma 557 — **różnica 230 metod**.
Część to metody administracyjne używane tylko przez `mmctl`, ale rząd wielkości pokazuje,
że mirror nigdy nie był i nie jest pełny.

Dodatkowo — nazwy plików nawet nie odpowiadają sobie strukturalnie. Wspólnych nazw między
`public/model/*.go` (204) a `platform/types/src/*.ts` (63) jest **11**:

```
agents  audits  client4  cloud  compliance  config
content_flagging  limits  product_notices  saml  terms_of_service
```

> To domyka obserwację z Artefaktu 1: `public/model ↔ platform/types` miało **77% pewności
> współzmienności**. Teraz wiadomo dlaczego — bo ktoś **musi** to przepisać ręcznie,
> a graf zależności nigdy o tym nie przypomni.

### 3b. Kontrast: kontrakt pluginów **jest** maszynowo egzekwowany

| Mechanizm | Plik / cel |
|---|---|
| Generowana warstwa RPC | `public/plugin/client_rpc_generated.go` |
| Generowana warstwa timerów | `public/plugin/api_timer_layer_generated.go` |
| Generator | `make pluginapi` (`Makefile:439`) — „Generates api and hooks glue code for plugins" |
| Weryfikator | `make plugin-checker` (`Makefile:312`), wpięty w **`check-style`** (`Makefile:489`) |

Ta sama organizacja rozwiązała ten sam problem (kontrakt przez granicę procesu)
generatorem + checkerem w CI dla pluginów, a ręcznym przepisywaniem dla webappu.
Ta asymetria jest najczystszym sygnałem strukturalnym w repo.

### 3c. Granica modułów Go — twarda i czysta

`server/public` to **osobny moduł Go**. Kompilator fizycznie uniemożliwia mu import
`server/v8`. Zweryfikowane: **0 krawędzi `public/ → v8/`**. To jedyna granica
w repozytorium egzekwowana narzędziowo, a nie konwencją.

### 3d. Kto konsumuje `@mattermost/types` i `@mattermost/client`

```
webapp/channels          2235 importów @mattermost/types
webapp/platform/shared      ~6
webapp/platform/components  ~2
e2e-tests/playwright       109 @mattermost/types  +  57 @mattermost/client
```

`e2e-tests/playwright/package.json` linkuje je przez protokół `file:`:

```json
"@mattermost/client": "file:../../webapp/platform/client",
"@mattermost/types":  "file:../../webapp/platform/types"
```

**E2E to czwarty konsument kontraktu, nie tylko testy.** Zmiana w `platform/types`
łamie kompilację `e2e-tests/` — co tłumaczy współzmienność `playwright/lib ↔ public/model`
(42 wspólne commity) zaobserwowaną w Artefakcie 1.

---

## 4. Krawędzie runtime — czego graf importów nie widzi

To najważniejsza część dla oceny blast radius, bo są to zależności **realne, ale niewidoczne
dla kompilatora i dla każdego narzędzia do grafu**.

### 4a. Module Federation — `webapp/channels/src` eksportuje publiczne API do wtyczek

```js
// webapp/channels/webpack.config.js
moduleFederationPluginOptions.exposes = {
    './app':      'components/app',
    './store':    'stores/redux_store',
    './styles':   './src/sass/styles.scss',
    './registry': 'module_registry',
};
```

Cztery wewnętrzne moduły `channels/src` są **kontraktem publicznym dla zewnętrznych
bundle'i pluginów i produktów**. Zmiana kształtu `stores/redux_store` albo `module_registry`
zepsuje wtyczki third-party **w runtime, po stronie klienta, bez żadnego sygnału w CI**.

### 4b. Współdzielone zależności — dwa reżimy wersjonowania

```js
makeSharedModules(['@mattermost/client', '@mattermost/types', 'luxon'], false)
//   singleton: false, strictVersion: false  → dozwolony skew wersji

makeSharedModules(['react','react-dom','react-redux','react-router-dom',
                   'react-intl','react-bootstrap','styled-components',
                   'history','@hello-pangea/dnd'], true)
//   singleton: true, strictVersion: true    → build/runtime error przy niezgodności
```

**Kontrakt danych (`@mattermost/types`, `@mattermost/client`) jest w reżimie luźnym.**
Plugin z własną, starszą kopią typów załaduje ją bez ostrzeżenia. Reżim ostry zarezerwowano
dla Reacta — czyli dla ryzyka technicznego, a nie dla ryzyka kontraktowego.

### 4c. Zdalne kontenery produktowe

```js
remotes[product.name] = `${product.name}@[window.basename]/static/products/${product.name}/remote_entry.js`
```

Produkty ładowane przez `remote_entry.js` z runtime'owym podstawieniem `window.basename`
(`ExternalTemplateRemotesPlugin`). Lista `products` jest obecnie pusta w konfiguracji,
ale mechanizm jest aktywny.

### 4d. Zestawienie: gdzie „idź za importami" zawodzi

| Krawędź | Widoczna statycznie? | Co ją domyka |
|---|---|---|
| `channels/app/platform` → `enterprise` | ❌ | `RegisterXInterface()` + blank import w `main.go` |
| `channels/api4` → handler dla trasy | ❌ | rejestracja przez `mux.Router` w `api.go` |
| `stores/redux_store` → wtyczka third-party | ❌ | Module Federation `exposes` |
| `public/model` → `platform/types` | ❌ | ręczne przepisanie (brak generatora) |
| `public/model` → `api/v4/source/*.yaml` | ❌ | ręczne przepisanie |
| `store.go` → `timerlayer`/`retrylayer`/`mocks` | ❌ | `make store-layers` (Artefakt 1 §0a) |
| `public/plugin` → glue RPC | ✅ pośrednio | `make pluginapi` + `plugin-checker` w `check-style` |

---

## 5. Odpowiedzi na pytania kroku

### Gdzie widać kontrakty między warstwami

Pięć granic, o bardzo różnej jakości egzekwowania:

| Granica | Egzekwowanie | Ocena |
|---|---|---|
| `server/public` ↔ `server/v8` | moduł Go — kompilator | 🟢 twarda, 0 naruszeń |
| `webapp/platform/*` ↔ `webapp/channels` | project references + `package.json` | 🟢 czysta, acykliczna |
| `public/plugin` ↔ wtyczki Go | generator + `plugin-checker` w CI | 🟢 maszynowa |
| **`public/model` ↔ `platform/types` ↔ `api/v4`** | **konwencja i pamięć autora** | 🔴 **230 metod różnicy** |
| `channels/src` ↔ bundle'e wtyczek JS | Module Federation, `strictVersion:false` | 🔴 niewidoczna, luźna |

### Cienkie wejścia vs głębokie centra

| 🚪 Cienkie wejścia | Linie | 🧠 Głębokie centra | Linie / rozmiar |
|---|---:|---|---|
| `cmd/mattermost/main.go` | 23 | `public/model/client4.go` | 8541 / 787 metod |
| `channels/src/root.tsx` | 26 | `public/model/config.go` | 5766 |
| `packages/mattermost-redux/src/client/index.ts` | 18 | `platform/client/src/client4.ts` | 5530 / 557 metod |
| `channels/src/entry.tsx` | 79 | `channels/app/server.go` | 2093 |
| `channels/api4/api.go` (routing) | 544 / 764 endpointy | `utils/constants.tsx` | 2258 / fan-in 733 |
| — | | `channels/store/store.go` | 1516 / 63 interfejsy, 1024 metody |

### Czy graf pokazuje cykle albo podejrzane zależności

- **Go: 0 cykli** — wymuszone przez kompilator. Sprzężenie uciekło do **rejestru runtime**
  (`einterfaces` + blank importy) i tam jest niewidoczne.
- **TypeScript: 82 pary dwukierunkowe, SCC = 192 z 384 modułów (50%)**. Cykle biegną
  **między warstwami** (`actions ↔ utils ↔ types ↔ selectors ↔ stores`), nie tylko
  między komponentami.
- **Podejrzane:** `platform/mattermost-redux` (pakiet publikowalny bez własnego kodu,
  czytający z `channels/src`) oraz `@mattermost/types` w Module Federation z
  `strictVersion: false`.

### Kto zależy od danego modułu i co może pęknąć

| Zmienisz… | Statycznie pęknie | Może pęknąć cicho |
|---|---|---|
| `public/model/*.go` | 129 pakietów Go (84% repo tranzytywnie) | `platform/types`, `client4.ts`, `api/v4/*.yaml`, wtyczki JS ze starą kopią typów |
| `channels/store/store.go` | 41 pakietów | generowane `timerlayer`/`retrylayer`/`mocks` — do regeneracji `make store-layers` |
| `einterfaces/*.go` | 16 pakietów bezpośrednio | **59 tranzytywnie**, w tym całe `enterprise/` — wykryjesz dopiero przy buildzie enterprise |
| `platform/types/src/*.ts` | `channels` (2235 importów), `platform/{client,shared}` | **`e2e-tests/playwright`** (109 importów przez `file:`) |
| `utils/constants.tsx` | 733 pliki (26% webappu) | wszystko w SCC-192 |
| `stores/redux_store` | kilka plików wewnątrz `channels` | **wtyczki third-party** przez Module Federation — brak sygnału w CI |
| `channels/app/*` | 11 pakietów (fan-in niski!) | myląco mało — `app` ma fan-out 72, to on się psuje od innych |
| `platform/client/src/client4.ts` | 31 bezpośrednich importów | **142 pliki** przez singleton `mattermost-redux/client` |

---

## 6. Wnioski

1. **Repo ma dwie różne architektury.** Warstwa Go: acykliczna, warstwowa, z twardą
   granicą modułów. Warstwa TS `channels/src`: 50% modułów w jednym cyklu.
   Nie da się do nich stosować tych samych reguł oceny ryzyka.

2. **`public/model` to strukturalne centrum całego repozytorium**, potwierdzone niezależnie
   od Artefaktu 1: 129 bezpośrednich importerów, 84% zasięgu tranzytywnego, plus
   2235 importów jego TS-owego lustra. Artefakt 1 pokazał je jako oś współzmienności —
   graf pokazuje je jako oś zależności. Zgodność obu metod.

3. **Najostrzejsze ryzyko to nie cykl, tylko brak generatora Go→TS.**
   Kontrakt REST istnieje w czterech ręcznie utrzymywanych kopiach, mierzalnie
   rozjechanych (787 vs 557 metod). Jednocześnie ta sama organizacja rozwiązała
   identyczny problem dla pluginów generatorem i checkerem w `check-style`.
   To luka narzędziowa, nie luka wiedzy.

4. **Fan-in mylnie ocenia dwa pakiety.** `einterfaces` (16 → 59, ×3,7) i `config`
   (5 → 18, ×3,6) wyglądają na peryferia, a są wzmacniaczami. `channels/app` odwrotnie:
   fan-in 11 sugeruje bezpieczeństwo, a fan-out 72 oznacza, że pęka od wszystkiego innego.

5. **Blast radius w `webapp/channels/src` jest nieobliczalny wewnątrz SCC-192.**
   Praktyczna reguła: dla zmian w `utils`, `actions`, `types`, `selectors`, `stores`
   i `components/{common,widgets,admin_console}` zakładaj zasięg globalny w webappie
   i opieraj się na testach, nie na analizie zależności.

6. **Cztery moduły `channels/src` są de facto publicznym API** (`components/app`,
   `stores/redux_store`, `module_registry`, `sass/styles.scss`). Nie są oznaczone jako
   takie ani w kodzie, ani w strukturze katalogów — tylko w `webpack.config.js`.

7. **Wejście do Artefaktu 3:** obszar do analizy kontrybutorów wybrać tam, gdzie
   nakładają się wysoka aktywność (Artefakt 1) i strukturalna krytyczność (Artefakt 2) —
   czyli **`server/public/model` + `webapp/platform/types` + `api/v4/source`**
   (kto pilnuje trzech kopii kontraktu?) albo **klaster ABAC**, który przecina wszystkie
   warstwy naraz.

---

## 7. Ograniczenia tej analizy

- **Graf zbudowany parserem tekstowym, nie kompilatorem.** Brak `go` w PATH i brak
  `node_modules` uniemożliwiły weryfikację przez `go list -deps` / `madge`. Importy
  dynamiczne (`await import(...)`), re-eksporty przez `index.ts` i importy typów
  wymazywane przy kompilacji nie są rozróżniane.
- **Wykluczono `_test.go` i `*.test.*`** — testy dodałyby krawędzie (np. `app ↔ testlib`,
  które w Artefakcie 1 miało 100% współzmienności).
- **Granulacja bucketów TS jest arbitralna** (`components/*` depth 2, reszta depth 1).
  Artefakt: plik `components/app.tsx` i katalog `components/app/` liczą się jako dwa
  różne węzły — dotyczy kilku par w §2, nie wpływa na rozmiar SCC.
- **Nie zmierzono siły krawędzi.** Krawędź „1 import" waży tyle samo co „40 importów";
  fan-in liczy moduły, nie wywołania.
- `webapp/platform/{components,shared,eslint-plugin}` przeanalizowano tylko pod kątem
  importów `@mattermost/*`, bez pełnego grafu wewnętrznego.
- **Graf nie mówi, kto to utrzymuje** — to wejście do Artefaktu 3.
