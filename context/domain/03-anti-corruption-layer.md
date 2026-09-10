---
title: Anti-Corruption Layer — go-opengraph w modelu link-preview
created: 2026-09-10
type: refactor-plan
---

# Plan refaktoru: warstwa antykorupcyjna dla `github.com/dyatlov/go-opengraph`

> **To jest PLAN.** Kod produkcyjny nie został zmodyfikowany. Wszystkie cytaty
> `plik:linia` pochodzą z bezpośredniego czytania kodu na HEAD `5ce50eaa5c`.
> Dokument kontynuuje linię [01-domain-distillation.md](01-domain-distillation.md)
> i [02-invariant-aggregate-refactor.md](02-invariant-aggregate-refactor.md), ale
> dotyczy innej osi: nie niezmiennika agregatu, lecz **przecieku zależności
> zewnętrznej przez granice warstw**.

---

## KROK 0 — Kontekst

**Brak dokumentów wymagań.** Jak w [01-domain-distillation.md §KROK 0](01-domain-distillation.md):
w repo nie ma `context/foundation/prd.md` ani `tech-stack.md`. Stack (z
[README.md](../../README.md#L3) i [repo-map.md §1](../map/repo-map.md)): Go +
React, warstwy serwera `server/channels/store` → `server/channels/app` →
`server/channels/api4` nad współdzielonym modelem `server/public/model`,
lustrzanym ręcznie w `webapp/platform/types`.

**Kluczowy fakt strukturalny:** `server/public` to **osobny moduł Go**
([server/public/go.mod:1](../../server/public/go.mod#L1) —
`module github.com/mattermost/mattermost/server/public`), publikowany jako SDK dla
wtyczek. `server/public/model` ma fan-in 129 pakietów i zasięg tranzytywny 84%
([repo-map.md §1 pkt 1](../map/repo-map.md)), a jego lustro `@mattermost/types` ma
w webappie więcej importów niż `react`. Każda zależność zewnętrzna, którą
`server/public/model` wciąga, jest kompilowana do **każdej wtyczki** i do grafu
zależności całego repo.

**Deklaracje o wymienialności.** Brak PRD, więc nie ma wprost spisanej intencji
„to ma być wymienialne". Jest natomiast **twardy sygnał w manifeście modułu**:
[server/public/go.mod:71-76](../../server/public/go.mod#L71) zawiera blok
`exclude`, a w nim starą ścieżkę modułową tej właśnie biblioteki
(`github.com/dyatlov/go-opengraph v0.0.0-20210112100619-dae8665a5b09`). Projekt
musiał **aktywnie wykluczyć** przestarzały wariant biblioteki, bo `go-opengraph`
zmieniło ścieżkę modułową (z `dyatlov/go-opengraph` na
`dyatlov/go-opengraph/opengraph`) — to samo zjawisko, które komentarz nad blokiem
opisuje dla `willf/bitset`. Wersja używana dziś to pseudo-wersja przypięta do
commita z **24 maja 2022** ([server/go.mod:22](../../server/go.mod#L22),
[server/public/go.mod:7](../../server/public/go.mod#L7)) — ponad 3 lata bez
aktualizacji.

---

## KROK 1 — Zidentyfikowane przeciekające zależności

Przeszukano importy zewnętrzne w `server/public/model/*.go` (manifest: łącznie ~30
zewnętrznych pakietów). Trzy zależności przeciekają przez granice warstw w sposób
domenowo istotny:

### Oś A — `github.com/dyatlov/go-opengraph` (link preview / OpenGraph)

Typ biblioteki `opengraph.OpenGraph` (oraz `.../types/image.Image`) jest
**jednocześnie**: typem w polu domenowym, formatem persystencji (kolumna DB),
kontraktem wire (odpowiedź REST osadzona w `Post`), typem cache w pamięci oraz —
ręcznie odtworzony — typem w webappie.

**Pliki, które dziś „znają" tę zależność:**

| Warstwa | Plik:linia | Co robi |
|---|---|---|
| model | [link_metadata.go:17-18](../../server/public/model/link_metadata.go#L17) | import `opengraph` + `opengraph/types/image` |
| model | [link_metadata.go:41-46](../../server/public/model/link_metadata.go#L41) | `LinkMetadata.Data any` — komentarz deklaruje, że pole trzyma `*opengraph.OpenGraph` |
| model | [link_metadata.go:71-88](../../server/public/model/link_metadata.go#L71) | `TruncateOpenGraph(*opengraph.OpenGraph) *opengraph.OpenGraph` — operacja domenowa na typie biblioteki |
| model | [link_metadata.go:91-112](../../server/public/model/link_metadata.go#L91) | `FilterSVGImages([]*image.Image) []*image.Image` — operacja na typie biblioteki |
| model | [link_metadata.go:156-163](../../server/public/model/link_metadata.go#L156) | `IsValid()` robi `o.Data.(*opengraph.OpenGraph)` (type assertion) |
| model | [link_metadata.go:197-202](../../server/public/model/link_metadata.go#L197) | `DeserializeDataToConcreteType()` — `json.Unmarshal` do `&opengraph.OpenGraph{}` (**rekonstrukcja #1**) |
| model | [post_embed.go:23-24](../../server/public/model/post_embed.go#L23) | `PostEmbed.Data any` — „Only used for OpenGraph embeds" |
| model | [post_metadata.go:11-13](../../server/public/model/post_metadata.go#L11) | `PostMetadata.Embeds` — dokumentacja pola odnosi się do OpenGraph |
| app | [opengraph.go:11-12](../../server/channels/app/opengraph.go#L11) | import |
| app | [opengraph.go:54-79](../../server/channels/app/opengraph.go#L54) | `parseOpenGraphMetadata` → `opengraph.NewOpenGraph()`, `og.ProcessHTML(body)` (**parser HTML**) |
| app | [opengraph.go:88-131](../../server/channels/app/opengraph.go#L88) | `makeOpenGraphURLsAbsolute(*opengraph.OpenGraph, string)` — iteruje `og.Images/Audios/Videos` |
| app | [opengraph.go:133-146](../../server/channels/app/opengraph.go#L133) | `openGraphDataWithProxyAddedToImageURLs` — mutuje `ogdata.Images` |
| app | [opengraph.go:148-157](../../server/channels/app/opengraph.go#L148) | `filterSVGImagesFromOpenGraph` — deleguje do `model.FilterSVGImages` |
| app | [opengraph.go:159-162](../../server/channels/app/opengraph.go#L159) | `openGraphDecodeHTMLEntities` — `html.UnescapeString(og.Title/Description)` |
| app | [opengraph.go:164-190](../../server/channels/app/opengraph.go#L164) | `parseOpenGraphFromOEmbed` — buduje `&opengraph.OpenGraph{...}` z odpowiedzi oEmbed (**rekonstrukcja #2 / ad-hoc adapter, który już istnieje**) |
| app | [post_metadata.go:19](../../server/channels/app/post_metadata.go#L19), [:34](../../server/channels/app/post_metadata.go#L34) | import; pole `OpenGraph *opengraph.OpenGraph` w strukturze cache `linkMetadataCache` |
| app | [post_metadata.go:579-583](../../server/channels/app/post_metadata.go#L579) | `getEmbedForPost` ustawia `PostEmbed{Data: og}` — **surowy wskaźnik biblioteki trafia na wire** |
| app | [post_metadata.go:632](../../server/channels/app/post_metadata.go#L632) | `embed.Data.(*opengraph.OpenGraph)` (type assertion #2) |
| app | [post_metadata.go:847-1164](../../server/channels/app/post_metadata.go#L847) | 12 wystąpień `opengraph.*`: sygnatury `getLinkMetadata*`, `getLinkMetadataFromCache/Database`, `saveLinkMetadataToDatabase`, `cacheLinkMetadata`, `parseLinkMetadata`; type switch `case *opengraph.OpenGraph:` ([:1084](../../server/channels/app/post_metadata.go#L1084)); `model.TruncateOpenGraph` ([:864](../../server/channels/app/post_metadata.go#L864), [:873](../../server/channels/app/post_metadata.go#L873), [:1022](../../server/channels/app/post_metadata.go#L1022)) |
| store (sql) | [link_metadata_store.go:49](../../server/channels/store/sqlstore/link_metadata_store.go#L49) | `json.Marshal(metadata.Data)` — struktura biblioteki serializowana **do kolumny DB `Data`** |
| store (sql) | [link_metadata_store.go:93](../../server/channels/store/sqlstore/link_metadata_store.go#L93) | przy odczycie woła `metadata.DeserializeDataToConcreteType()` |
| store (test) | [storetest/link_metadata_store.go:11-12](../../server/channels/store/storetest/link_metadata_store.go#L11), [:208-235](../../server/channels/store/storetest/link_metadata_store.go#L208) | testy kontraktu store konstruują `&opengraph.OpenGraph{}` i robią `require.IsType(t, &opengraph.OpenGraph{}, received.Data)` |
| model (test) | [link_metadata_test.go:58-346](../../server/public/model/link_metadata_test.go#L58) | ~8 wystąpień `opengraph.OpenGraph{}` |
| app (test) | [opengraph_test.go](../../server/channels/app/opengraph_test.go), [post_metadata_test.go](../../server/channels/app/post_metadata_test.go) | testy na typie biblioteki |
| webapp (typy) | [posts.ts:176-190](../../webapp/platform/types/src/posts.ts#L176) | **ręcznie odtworzony** `OpenGraphMetadata` / `OpenGraphMetadataImage` — **4. kopia kształtu** |
| webapp (typy) | [posts.ts:41](../../webapp/platform/types/src/posts.ts#L41), [:46](../../webapp/platform/types/src/posts.ts#L46), [:164](../../webapp/platform/types/src/posts.ts#L164) | `PostEmbedType` z `'opengraph'`; `PostEmbed.data`; `openGraph` w `PostsState` |
| webapp (redux) | [reducers/entities/posts.ts:62-70](../../webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts#L62) | usuwa `data` z embeda `opengraph` po skopiowaniu |
| webapp (redux) | [reducers/entities/posts.ts:1534-1587](../../webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts#L1534) | reducer `openGraph` — zapisuje **surowe `embed.data`** do stanu, bez mapowania |
| webapp (redux) | [selectors/entities/posts.ts:101-107](../../webapp/channels/src/packages/mattermost-redux/src/selectors/entities/posts.ts#L101) | `getOpenGraphMetadata`, `getOpenGraphMetadataForUrl` |
| webapp (UI) | [post_attachment_opengraph.tsx:10-14](../../webapp/channels/src/components/post_view/post_attachment_opengraph/post_attachment_opengraph.tsx#L10), [:55](../../webapp/channels/src/components/post_view/post_attachment_opengraph/post_attachment_opengraph.tsx#L55) | `getBestImage(openGraphData?: OpenGraphMetadata, ...)` |
| webapp (UI) | [youtube_video.tsx](../../webapp/channels/src/components/youtube_video/youtube_video.tsx) | konsumuje `OpenGraphMetadata` |

### Oś B — `github.com/gorilla/websocket`

| Warstwa | Plik:linia | Co robi |
|---|---|---|
| model | [websocket_client.go:16](../../server/public/model/websocket_client.go#L16), [:48](../../server/public/model/websocket_client.go#L48) | eksportowany `WebSocketClient.Conn *websocket.Conn`; sygnatury `New*WithDialer(dialer *websocket.Dialer, ...)` ([:68-138](../../server/public/model/websocket_client.go#L68)) — **typ biblioteki w publicznym API pakietu domenowego** |
| api4 | [websocket.go](../../server/channels/api4/websocket.go), [apitestlib.go](../../server/channels/api4/apitestlib.go) | upgrade połączenia |
| app / platform | [platform/web_conn.go](../../server/channels/app/platform/web_conn.go), [app/webhub_fuzz.go](../../server/channels/app/webhub_fuzz.go) | pętla read/write |

Ten sam SDK po obu stronach granicy klient/serwer (`model` dostarcza klienta WS,
`app/platform` obsługuje serwer WS). Ale: `gorilla/websocket` to de-facto standard
Go, ryzyko wymiany bliskie zeru; `websocket_client.go` to raczej **kod w złej
warstwie** (klient integracyjny w `model`) niż korupcja domeny.

### Oś C — biblioteki semver (trzy naraz)

| Biblioteka | Gdzie w `public/model` | Call-sites |
|---|---|---|
| `golang.org/x/mod/semver` | [access_policy.go:13](../../server/public/model/access_policy.go#L13) | 5× `semver.IsValid(p.Version)` ([:323](../../server/public/model/access_policy.go#L323), [:370](../../server/public/model/access_policy.go#L370), [:409](../../server/public/model/access_policy.go#L409), [:497](../../server/public/model/access_policy.go#L497), [:616](../../server/public/model/access_policy.go#L616)) |
| `github.com/Masterminds/semver/v3` | [config.go:23](../../server/public/model/config.go#L23), [manifest.go:14](../../server/public/model/manifest.go#L14), [metrics.go:10](../../server/public/model/metrics.go#L10), [packet_metadata.go:9](../../server/public/model/packet_metadata.go#L9), [plugin_install.go:6](../../server/public/model/plugin_install.go#L6) | ~12× `StrictNewVersion` / `NewVersion` / `MustParse` |
| `github.com/blang/semver` | `server/channels/app` | (poza modelem) |

Realna duplikacja rekonstrukcji „wersji" trzema różnymi bibliotekami w jednym
pakiecie domenowym, bez value objectu `Version`. Ale: koszt wymiany każdej z osobna
niski, wartość niska (infrastruktura, nie różnicująca logika), brak deklaracji
wymienialności.

---

## KROK 2 — Klasyfikacja i wybór #1

| Oś | (a) warstwy/pliki dotknięte | (b) ryzyko/koszt wymiany dziś | (c) deklaracja wymienialności / rozjazd intencja-kod |
|---|---|---|---|
| **A — go-opengraph** | **6 warstw**: model, app, store (sql + kontrakt), wire REST, cache, webapp (types + redux + UI). ~28 plików produkcyjnych + testy. | **Wysoki.** Typ jest formatem persystencji (dane w kolumnie DB), formatem wire i typem cache jednocześnie — wymiana biblioteki zmienia kształt danych w 3 miejscach naraz, bez migracji. Biblioteka nieutrzymywana (pin 2022), **HTML-scraper** (`ProcessHTML`) w grafie zależności modułu SDK dla wtyczek. | **Rozjazd.** `server/public/go.mod` musi trzymać blok `exclude` dla starej ścieżki modułowej ([:71-76](../../server/public/go.mod#L71)) — projekt już dziś płaci koszt niestabilności tej zależności. `LinkMetadata.Data`/`PostEmbed.Data` są typu `any` „bo nie chcemy typu biblioteki w polu" — ale komentarz i tak deklaruje `*opengraph.OpenGraph`, a `IsValid()` to wymusza. Intencja („`any`, żeby odseparować") jest, egzekwowanie jej nie. |
| B — gorilla/websocket | 3 warstwy (model, api4, app/platform) | Niski — standard bez alternatywy pod presją. | Brak. To bardziej „kod w złej warstwie". |
| C — semver ×3 | model (7 plików) + app | Niski per biblioteka; trójka bibliotek to koszt spójności, nie wymiany. | Brak. |

**Wybór: Oś A — `github.com/dyatlov/go-opengraph`.**

Uzasadnienie: to jedyny przeciek, który spełnia wszystkie trzy kryteria naraz —
najszerszy zasięg przez warstwy (w tym trzy **różne** kontrakty: DB, wire, cache),
najwyższe ryzyko wymiany (nieutrzymywana biblioteka-scraper w module SDK, której
JSON jest formatem danych na dysku), i udokumentowany rozjazd intencja-vs-kod
(pole `any` „dla separacji", której nie ma; `exclude` w manifeście jako dowód, że
niestabilność już boli). Dodatkowo: **adapter konwersji już istnieje ad hoc** —
`parseOpenGraphFromOEmbed` ([opengraph.go:164-190](../../server/channels/app/opengraph.go#L164))
buduje `opengraph.OpenGraph` z zupełnie innego formatu (oEmbed). To dowód, że
kod chce mieć swój wewnętrzny typ „link preview" — po prostu użył do tej roli
typu biblioteki.

---

## KROK 3 — Diagnoza

### 3.1 Duplikacja operacji domenowych na typie biblioteki

Pięć operacji, które są **regułami domenowymi** o tym, jak Mattermost traktuje
metadane linku, jest dziś rozsmarowanych po `model` i `app` i każda bierze/zwraca
`*opengraph.OpenGraph`:

| Operacja domenowa | Gdzie | Cytat |
|---|---|---|
| „skróć zbyt długie teksty, wytnij nadmiarowe pola, ogranicz liczbę obrazów do 5" | model | [link_metadata.go:71-88](../../server/public/model/link_metadata.go#L71) `TruncateOpenGraph` — zeruje `Article/Book/Profile/Determiner/Locale/LocalesAlternate/Audios/Videos` |
| „usuń obrazy SVG (MM-67372)" | model + app | [link_metadata.go:91-112](../../server/public/model/link_metadata.go#L91) `FilterSVGImages` + [opengraph.go:148-157](../../server/channels/app/opengraph.go#L148) `filterSVGImagesFromOpenGraph` |
| „URL-e względne → bezwzględne względem URL strony" | app | [opengraph.go:88-131](../../server/channels/app/opengraph.go#L88) `makeOpenGraphURLsAbsolute` |
| „przepuść URL-e obrazów przez image proxy" | app | [opengraph.go:133-146](../../server/channels/app/opengraph.go#L133) |
| „odkoduj encje HTML w tytule/opisie" | app | [opengraph.go:159-162](../../server/channels/app/opengraph.go#L159) |
| „metadane bez tytułu I bez URL to brak metadanych" | app | [post_metadata.go:1155-1164](../../server/channels/app/post_metadata.go#L1155) — komentarz: *„The OpenGraph library and Go HTML library don't error for malformed input, so check that at least one of these required fields exists"* |

`TruncateOpenGraph` faktycznie mówi: „domena Mattermost interesuje się tylko
`Title/Description/SiteName/URL/Images`" — reszta pól biblioteki jest natychmiast
zerowana. Mimo to reszta pól podróżuje przez cały system (DB, wire, redux) jako
martwy balast, bo typ jest typem biblioteki, nie okrojonym typem domenowym.

### 3.2 Rekonstrukcja typu w wielu miejscach

- **#1** — [link_metadata.go:197-202](../../server/public/model/link_metadata.go#L197):
  `json.Unmarshal(b, &og)` do `&opengraph.OpenGraph{}` przy odczycie z DB.
- **#2** — [opengraph.go:171-186](../../server/channels/app/opengraph.go#L171):
  `parseOpenGraphFromOEmbed` ręcznie składa `&opengraph.OpenGraph{Type, Title, URL}`
  + `&ogImage.Image{...}` z odpowiedzi oEmbed.
- **#3** — webapp: [reducers/entities/posts.ts:1579-1587](../../webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts#L1579)
  rzutuje `embed.data` na `OpenGraphMetadata` i zapisuje do stanu bez walidacji ani mapowania.

### 3.3 Przeciek przez granice — trzy kontrakty, jeden typ biblioteki

```
opengraph.OpenGraph (struct biblioteki, tagi JSON kontrolowane przez autora biblioteki)
        │
        ├── json.Marshal → kolumna DB  LinkMetadata.Data   [link_metadata_store.go:49]
        ├── json.Marshal → odpowiedź REST  Post.metadata.embeds[].data   [post_metadata.go:579-583]
        └── pole cache  linkMetadataCache.OpenGraph   [post_metadata.go:34]
```

Groźne konkretnie:

1. **Biblioteka-scraper w module SDK.** `server/public/model` (moduł
   `server/public`, fan-in 129) importuje `go-opengraph`. `ProcessHTML`
   ([opengraph.go:59](../../server/channels/app/opengraph.go#L59)) to parser HTML —
   choć wołany w `app`, sama biblioteka jest bezpośrednią zależnością manifestu
   `server/public/go.mod` ([:7](../../server/public/go.mod#L7)), więc **każda
   wtyczka** importująca `.../server/public/model` kompiluje ją do siebie.
2. **JSON biblioteki = schemat na dysku.** Kolumna `LinkMetadata.Data` zawiera
   `json.Marshal` structa biblioteki. Aktualizacja biblioteki, która zmieni tag
   JSON albo typ pola (`image.Image.Width` jest `uint64`), zmienia format danych,
   które już leżą w bazie u klientów — bez ścieżki migracji.
3. **Rozjazd webapp.** [posts.ts:176-190](../../webapp/platform/types/src/posts.ts#L176)
   odtwarza 6 pól (`type/title/description/site_name/url/images`); struct
   biblioteki niesie ich ~14 (dochodzą `Audios/Videos/Article/Book/Profile/
   Determiner/Locale/LocalesAlternate`). To 4. ręcznie utrzymywana kopia kształtu —
   dokładnie ten wzorzec, który [01-domain-distillation.md §KROK 4](01-domain-distillation.md)
   opisał dla `AccessControlPolicy`.

### 3.4 Deklaracja wymienialności vs kod

[server/public/go.mod:68-76](../../server/public/go.mod#L68):

```
// They changed the module path from github.com/willf/bitset to
// github.com/bits-and-blooms/bitset and a couple of dependent repos are yet
// to update their module paths.
exclude (
    github.com/RoaringBitmap/roaring v0.7.0
    github.com/RoaringBitmap/roaring v0.7.1
    github.com/dyatlov/go-opengraph v0.0.0-20210112100619-dae8665a5b09
    github.com/willf/bitset v1.2.0
)
```

`go-opengraph` przeszło tę samą zmianę ścieżki modułowej co `willf/bitset`
(`dyatlov/go-opengraph` → `dyatlov/go-opengraph/opengraph`). Projekt **już płaci
koszt** niestabilności tej zależności na poziomie manifestu, a mimo to jej typ
siedzi w polu domenowym, w kolumnie DB i w kontrakcie wire. Intencja separacji
jest widoczna (`Data any`, a nie `Data *opengraph.OpenGraph`), lecz kod ją omija.

---

## KROK 4 — Projekt warstwy antykorupcyjnej

### 4.1 Domenowy value object — `model.LinkPreview`

Jedyne miejsce w całym repo, które zna kształt „metadanych linku". Pola **własne,
jawne, z własnymi tagami JSON** — to one stają się kontraktem wire i DB, niezależnie
od biblioteki. Celowo pomija pola, które `TruncateOpenGraph` i tak natychmiast
zeruje — decyzja „domena interesuje się tylko tym podzbiorem" zostaje **zakodowana
w typie**, nie w funkcji czyszczącej wołanej po fakcie.

```go
// server/public/model/link_preview.go   (NOWY plik; ZERO importu go-opengraph)

// LinkPreview is Mattermost's own representation of the metadata extracted from
// a link in a message. Its JSON shape is the persistence and wire contract for
// LinkMetadata.Data and PostEmbed.Data — deliberately independent of whichever
// library scraped it.
type LinkPreview struct {
    Type        string             `json:"type,omitempty"`
    Title       string             `json:"title,omitempty"`
    Description  string             `json:"description,omitempty"`
    SiteName    string             `json:"site_name,omitempty"`
    URL         string             `json:"url,omitempty"`
    Images      []*LinkPreviewImage `json:"images,omitempty"`
}

type LinkPreviewImage struct {
    URL       string `json:"url,omitempty"`
    SecureURL string `json:"secure_url,omitempty"`
    Type      string `json:"type,omitempty"`
    Width     int    `json:"width,omitempty"`  // int, nie uint64 z image.Image
    Height    int    `json:"height,omitempty"`
}

// --- operacje domenowe (przeniesione z rozsypki model+app na typ) ---

// Truncate: zastępuje model.TruncateOpenGraph.
func (p *LinkPreview) Truncate() {
    p.Title = truncateText(p.Title)
    p.Description = truncateText(p.Description)
    p.SiteName = truncateText(p.SiteName)
    if len(p.Images) > LinkMetadataMaxImages {
        p.Images = p.Images[:LinkMetadataMaxImages]
    }
}

// FilterSVGImages: zastępuje model.FilterSVGImages + app.filterSVGImagesFromOpenGraph. (MM-67372)
func (p *LinkPreview) FilterSVGImages() { /* logika z link_metadata.go:91-112, ale na []*LinkPreviewImage */ }

// MakeURLsAbsolute: zastępuje app.makeOpenGraphURLsAbsolute.
func (p *LinkPreview) MakeURLsAbsolute(pageURL string) { /* logika z opengraph.go:88-131 */ }

// ProxyImageURLs: zastępuje app.openGraphDataWithProxyAddedToImageURLs.
func (p *LinkPreview) ProxyImageURLs(toProxyURL func(string) string) { /* opengraph.go:133-146 */ }

// DecodeHTMLEntities: zastępuje app.openGraphDecodeHTMLEntities.
func (p *LinkPreview) DecodeHTMLEntities() {
    p.Title = html.UnescapeString(p.Title)
    p.Description = html.UnescapeString(p.Description)
}

// IsMeaningful koduje regułę z post_metadata.go:1155-1164.
func (p *LinkPreview) IsMeaningful() bool { return p.Title != "" || p.URL != "" }

func (p *LinkPreview) IsValid() *AppError { /* obecne sprawdzenia z LinkMetadata.IsValid dot. gałęzi opengraph */ }
```

Zmiana w istniejącym `LinkMetadata`:

```go
// link_metadata.go — po refaktorze pole Data trzyma WYŁĄCZNIE typy model.*:
//   *model.PostImage   |  *model.LinkPreview  |  nil
// IsValid() i DeserializeDataToConcreteType() asertują/rekonstruują *model.LinkPreview,
// nigdy *opengraph.OpenGraph. import go-opengraph znika z pakietu model.
```

### 4.2 Wąski port + adapter

```go
// server/channels/app/linkpreview/extractor.go   (port — interfejs domenowy)

type Extractor interface {
    // FromHTML parsuje ciało dokumentu HTML w domenowy LinkPreview.
    FromHTML(body io.Reader, pageURL, contentType string) (*model.LinkPreview, error)

    // FromOEmbed konwertuje ciało odpowiedzi oEmbed w domenowy LinkPreview.
    FromOEmbed(body io.Reader, pageURL string) (*model.LinkPreview, error)
}
```

```go
// server/channels/app/linkpreview/opengraph/adapter.go
//   JEDYNY plik w całym repo, który importuje github.com/dyatlov/go-opengraph.

package opengraph

import oglib "github.com/dyatlov/go-opengraph/opengraph"

type Adapter struct{}

func (Adapter) FromHTML(body io.Reader, pageURL, contentType string) (*model.LinkPreview, error) {
    og := oglib.NewOpenGraph()
    if err := og.ProcessHTML(forceUTF8(body, contentType)); err != nil {
        return nil, err // parser HTML nigdy nie opuszcza tego pakietu
    }
    return toDomain(og, pageURL), nil
}

func (Adapter) FromOEmbed(body io.Reader, pageURL string) (*model.LinkPreview, error) {
    resp, err := oembed.ResponseFromJSON(body)
    if err != nil { return nil, err }
    return &model.LinkPreview{
        Type:  "opengraph",
        Title: resp.Title,
        URL:   pageURL,
        Images: imagesFromOEmbed(resp),
    }, nil
}

// toDomain — JEDYNE miejsce mapujące typ biblioteki → typ domenowy.
func toDomain(og *oglib.OpenGraph, pageURL string) *model.LinkPreview {
    p := &model.LinkPreview{
        Type: og.Type, Title: og.Title, Description: og.Description,
        SiteName: og.SiteName, URL: og.URL,
    }
    for _, img := range og.Images {
        p.Images = append(p.Images, &model.LinkPreviewImage{
            URL: img.URL, SecureURL: img.SecureURL, Type: img.Type,
            Width:  clampUint64ToInt(img.Width),   // decyzja: overflow/ujemne → clamp
            Height: clampUint64ToInt(img.Height),
        })
    }
    if p.URL != "" { p.URL = pageURL } // reguła z opengraph.go:74-76
    return p
}
```

Reszta kodu (`model`, `post_metadata.go`, store, api4, webapp) zna **tylko**
`*model.LinkPreview` i interfejs `Extractor`. `App` trzyma `Extractor` jako pole
(wstrzyknięte przy budowie serwera), domyślnie `opengraph.Adapter{}`.

### 4.3 Rozstrzygnięcie otwartych pytań zależnych od kontraktu biblioteki

| Pytanie | Rozstrzygnięcie (gdzie zakodować) |
|---|---|
| `image.Image.Width/Height` to `uint64` — co przy wartości > `MaxInt` lub śmieciowej? | `clampUint64ToInt` w `toDomain()` (adapter), **nie** w API. Domena zna tylko `int`. |
| `ProcessHTML` nie zwraca błędu przy śmieciowym wejściu ([post_metadata.go:1157-1158](../../server/channels/app/post_metadata.go#L1157)) | `LinkPreview.IsMeaningful()` (metoda VO) + wywołanie w `app` na wyniku adaptera; adapter zwraca zawsze niepusty `*LinkPreview` albo `error`. |
| oEmbed vs OpenGraph — dwa źródła, jeden typ | Oba idą przez `Extractor` → oba produkują `*model.LinkPreview`. Ad-hoc `parseOpenGraphFromOEmbed` znika. |
| Pola `Audios/Videos/Article/Book/...` | Nie mapowane — decyzja `TruncateOpenGraph` (że domena ich nie chce) staje się kształtem typu. Jeśli kiedyś będą potrzebne — dodaje się pole do `model.LinkPreview` i linię w `toDomain()`. |

---

## KROK 5 — Dowód izolacji + before/after

### 5.1 Lista: wymiana biblioteki dotyka tylko adaptera

Po refaktorze, żeby wymienić `go-opengraph` na inną bibliotekę (albo własny parser),
zmienia się **wyłącznie**:

- `server/channels/app/linkpreview/opengraph/adapter.go` (lub nowy pakiet-adapter),
- linia `require` w `server/go.mod` i `server/public/go.mod` — a właściwie
  **`server/public/go.mod` przestaje w ogóle wymagać tej biblioteki** (model już jej
  nie importuje), więc blok `exclude` ([:71-76](../../server/public/go.mod#L71))
  też można usunąć.

**Nie zmienia się:** schemat tabeli `LinkMetadata` (kolumna `Data` trzyma teraz
JSON `model.LinkPreview`, stabilny), kontrakt REST (`Post.metadata.embeds[].data`),
`webapp/platform/types/src/posts.ts`, żaden komponent UI, żadna wtyczka importująca
`server/public/model`.

### 5.2 Before/after per miejsce

| Miejsce | Dziś | Po refaktorze |
|---|---|---|
| `model/link_metadata.go` import | `dyatlov/go-opengraph/opengraph` + `.../image` | **usunięte** |
| `LinkMetadata.Data` | `any`, komentarz „= `*opengraph.OpenGraph`" | `any`, komentarz „= `*model.PostImage` \| `*model.LinkPreview`" |
| `model.TruncateOpenGraph(*opengraph.OpenGraph)` | funkcja pakietowa na typie biblioteki | metoda `(*model.LinkPreview).Truncate()` |
| `model.FilterSVGImages([]*image.Image)` | funkcja na typie biblioteki | metoda `(*model.LinkPreview).FilterSVGImages()` |
| `LinkMetadata.IsValid()` [:156-163](../../server/public/model/link_metadata.go#L156) | `o.Data.(*opengraph.OpenGraph)` | `o.Data.(*model.LinkPreview)` |
| `DeserializeDataToConcreteType()` [:197-202](../../server/public/model/link_metadata.go#L197) | `json.Unmarshal` do `&opengraph.OpenGraph{}` | `json.Unmarshal` do `&model.LinkPreview{}` |
| `app/opengraph.go` cały plik | 190 linii mieszające scraping + operacje domenowe + proxy | zostaje tylko `GetOpenGraphMetadata` (cache + HTTP); parsowanie → adapter; operacje domenowe → metody VO |
| `app.parseOpenGraphFromOEmbed` [:164-190](../../server/channels/app/opengraph.go#L164) | ręczna budowa `opengraph.OpenGraph` | `Extractor.FromOEmbed(...) (*model.LinkPreview, error)` |
| `app/post_metadata.go` — 12× `opengraph.*` | sygnatury i type-switch na typie biblioteki | `*model.LinkPreview` wszędzie |
| `getEmbedForPost` [:579-583](../../server/channels/app/post_metadata.go#L579) | `PostEmbed{Data: og /* *opengraph.OpenGraph */}` | `PostEmbed{Data: preview /* *model.LinkPreview */}` |
| `linkMetadataCache.OpenGraph` [:34](../../server/channels/app/post_metadata.go#L34) | `*opengraph.OpenGraph` | `*model.LinkPreview` |
| `sqlstore/link_metadata_store.go:49,93` | `json.Marshal` structa biblioteki | `json.Marshal` `model.LinkPreview` (kod bez zmian, inny typ pod spodem) |
| `storetest/link_metadata_store.go` | `require.IsType(t, &opengraph.OpenGraph{}, ...)` | `require.IsType(t, &model.LinkPreview{}, ...)` |
| webapp `OpenGraphMetadata` [posts.ts:176-190](../../webapp/platform/types/src/posts.ts#L176) | ręczna kopia kształtu structa biblioteki | **bez zmian dla webappa** — ale teraz jest lustrem `model.LinkPreview` (typu, który *my* kontrolujemy), więc rozjazd jest naprawialny generatorem/checkerem jak w [repo-map.md §5.1](../map/repo-map.md) |
| webapp redux/UI | konsumuje surowe `embed.data` | bez zmian (kształt JSON stabilny) |

### 5.3 UI dostaje dane domenowe, nie surowy obiekt biblioteki

Dziś: [reducers/entities/posts.ts:1579-1587](../../webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts#L1579)
zapisuje `embed.data` — dosłownie `json.Marshal(*opengraph.OpenGraph)` z serwera —
do stanu redux, a `post_attachment_opengraph.tsx` czyta z niego pola.
Po refaktorze ten sam JSON pochodzi z `json.Marshal(*model.LinkPreview)`: te same
klucze, ale kształt jest **kontraktem Mattermost**, nie strukturą, której tagi
ustala autor zewnętrznej biblioteki. Webapp nie musi się zmienić — zmienia się to,
kto jest właścicielem kontraktu.

---

## KROK 6 — Weryfikacja i plan faz

### 6.1 Kryterium sukcesu (grep)

```
$ grep -rl 'dyatlov/go-opengraph' --include='*.go' server/
server/channels/app/linkpreview/opengraph/adapter.go
server/channels/app/linkpreview/opengraph/adapter_test.go
```

Nic więcej. Dziś ta sama komenda zwraca **7 plików produkcyjnych + 4 testowe**
w 3 modułach (`server`, `server/public`) i 4 pakietach (`model`, `app`,
`store/sqlstore`, `store/storetest`).

| Plik | Dziś zna `go-opengraph` | Po refaktorze |
|---|---|---|
| `server/public/model/link_metadata.go` | ✅ | ❌ |
| `server/public/model/link_metadata_test.go` | ✅ | ❌ |
| `server/channels/app/opengraph.go` | ✅ | ❌ (import znika; parsowanie w adapterze) |
| `server/channels/app/opengraph_test.go` | ✅ | ❌ |
| `server/channels/app/post_metadata.go` | ✅ | ❌ |
| `server/channels/app/post_metadata_test.go` | ✅ | ❌ |
| `server/channels/store/storetest/link_metadata_store.go` | ✅ | ❌ |
| `server/channels/app/linkpreview/opengraph/adapter.go` | — | ✅ (jedyny) |
| `server/public/go.mod` — `require` | ✅ | ❌ (można też usunąć `exclude`) |
| `server/go.mod` — `require` | ✅ | ✅ (przeniesione do sekcji zależności `channels`, nie SDK) |

### 6.2 Plan faz (konwencja: `go test`, brak dedykowanego runnera TDD — [server/AGENTS.md](../../server/AGENTS.md); test-first dla nowej logiki domenowej, mechaniczna podmiana dla reszty)

1. **Faza 1 — test-first: `model.LinkPreview` + metody.**
   Nowy `server/public/model/link_preview.go` i `link_preview_test.go`. Przenieś
   logikę z `TruncateOpenGraph`, `FilterSVGImages` (+ testy z
   `link_metadata_test.go`) na metody VO. Model nadal importuje `go-opengraph`
   (stare funkcje zostają na razie obok). Zielone testy przed dalej.

2. **Faza 2 — test-first: port + adapter.**
   `server/channels/app/linkpreview/`: interfejs `Extractor` +
   `opengraph.Adapter` z `toDomain()`. Testy adaptera: fixture HTML → oczekiwany
   `*model.LinkPreview`; fixture oEmbed → `*model.LinkPreview`; przypadki
   `uint64` overflow, śmieciowy HTML (`IsMeaningful()==false`), URL względne.

3. **Faza 3 — podmiana w `app`.**
   `App` dostaje pole `linkPreview linkpreview.Extractor`. `getLinkMetadataForURL`,
   `getLinkMetadataFromOEmbed`, `parseLinkMetadata` i cache przechodzą na
   `*model.LinkPreview`. `app/opengraph.go` traci wszystko poza
   `GetOpenGraphMetadata`. Usuń `parseOpenGraphFromOEmbed`. Zabezpieczenie:
   istniejące `post_metadata_test.go`, `opengraph_test.go` (przepisane na nowy typ)
   muszą przejść.

4. **Faza 4 — podmiana w `model` + store.**
   `LinkMetadata.Data` gałąź opengraph → `*model.LinkPreview`.
   `DeserializeDataToConcreteType` i `IsValid` → `*model.LinkPreview`.
   Usuń `TruncateOpenGraph`/`FilterSVGImages` (stare sygnatury) i import
   `go-opengraph` z `link_metadata.go`. `storetest/link_metadata_store.go` →
   `model.LinkPreview`. Usuń `require` z `server/public/go.mod` (+ `exclude`).
   **Migracja danych:** niepotrzebna — klucze JSON `model.LinkPreview` = klucze
   dzisiejszego `json.Marshal(*opengraph.OpenGraph)` dla 5 zachowanych pól; stare
   wiersze z nadmiarowymi polami (`article` itd.) odczytają się z ignorowaniem
   nieznanych kluczy (domyślne zachowanie `encoding/json`).

5. **Faza 5 — porządek webapp (opcjonalnie, osobny change).**
   `OpenGraphMetadata` w `posts.ts` jest teraz lustrem `model.LinkPreview` —
   kandydat do objęcia tym samym checkerem kontraktu co Plugin API
   ([repo-map.md §5.1](../map/repo-map.md)).

### 6.3 Ryzyka planu

- `server/public/go.mod` to moduł SDK — usunięcie `require` to zmiana widoczna dla
  wtyczek (znika tranzytywna zależność). To **poprawa**, ale wymaga wpisu w
  changelogu SDK.
- `image.Image.Width uint64 → int`: teoretyczna utrata zakresu dla obrazów
  > 2^31 px. Praktycznie niemożliwe; `clampUint64ToInt` czyni to jawnym.
- `enterprise/` — szybki grep nie wykazał użycia `go-opengraph` w
  `server/enterprise` ani w `einterfaces`, ale build enterprise należy
  zweryfikować osobno (krawędź runtime, [repo-map.md §5.3](../map/repo-map.md)).

---

## Podsumowanie

Spośród trzech przeciekających zależności zewnętrznych w `server/public/model`
(`go-opengraph`, `gorilla/websocket`, trójka bibliotek semver) najgorszym
przeciekiem jest **`github.com/dyatlov/go-opengraph`**: jej typ `opengraph.OpenGraph`
jest jednocześnie polem domenowym (`LinkMetadata.Data`, `PostEmbed.Data` — pola
`any`, które „miały" separować, ale komentarz i `IsValid()` i tak wymuszają typ
biblioteki), formatem persystencji (kolumna DB `LinkMetadata.Data` to
`json.Marshal` structa biblioteki), kontraktem wire (osadzony w `Post.metadata`)
oraz — ręcznie odtworzonym jako 4. kopia kształtu — typem `OpenGraphMetadata` w
webappie. Biblioteka jest nieutrzymywana (pin do commita z 2022), przeszła zmianę
ścieżki modułowej, która zmusiła projekt do trzymania bloku `exclude` w manifeście
modułu SDK, i jest parserem HTML w grafie zależności pakietu o fan-in 129
importowanego przez każdą wtyczkę. Pięć operacji domenowych (skracanie, filtr SVG,
absolutyzacja URL, image proxy, dekodowanie encji) jest rozsmarowanych po `model`
i `app`, każda na typie biblioteki; adapter konwersji z oEmbed już istnieje ad hoc,
co dowodzi, że kod chce własnego typu „link preview". Projekt ACL: value object
`model.LinkPreview` z własnym kształtem JSON (kontrakt DB i wire) i przeniesionymi
na niego operacjami domenowymi, wąski port `linkpreview.Extractor` i jeden adapter
`opengraph.Adapter` — jedyny plik importujący bibliotekę. Po refaktorze `grep`
po nazwie pakietu zwraca wyłącznie katalog adaptera, wymiana biblioteki nie dotyka
tabel/API/UI, a moduł SDK przestaje w ogóle od niej zależeć.
