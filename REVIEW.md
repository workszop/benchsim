# Przegląd `deep_sim` i `co_sim` — wartość demo, wierność faktograficzna, poprawność wizualizacji

Metoda: statyczna lektura obu plików + uruchomienie w headless Chromium (Playwright),
przejście przez wszystkie 4 zakładki × 3 scenariusze w obu aplikacjach, zrzuty ekranu
desktop (1500 px) i mobile (390 px), odczyt wyliczonych wartości z DOM.

## Werdykt

| | `co_sim` | `deep_sim` |
|---|---|---|
| Gotowość na demo | **Tak, z jednym zastrzeżeniem** (ARES) | **Nie** — 4 blokery |
| Panele działające | 4 / 4 | 3 / 4 (QuestEval martwy) |
| Wierność faktograficzna treści PL | dobra (poprawne Dz.U., linki do gov.pl/ISAP) | słaba (błędna podstawa prawna, brak źródeł) |
| Model oceny (silnik) | prostszy, ale spójny | bogatszy (stemming klauzul, 8 wariantów loose) |
| Warstwa AI | brak (celowo) | pełna, dobrze napisana, 1 crash |

Krótko: `co_sim` to lepsze demo, `deep_sim` to lepszy silnik. Docelowo warto połączyć
silnik `deep_sim` z warstwą prezentacji i treścią `co_sim`.

---

## A. Blokery `deep_sim` (widoczne na ekranie w ciągu 30 sekund demo)

### A1. QuestEval nie działa w ogóle — `deep_sim.html:2073`

```js
const target = $("q-target") ? $("#q-target").value : "";
```

Brak `#`. `querySelector("q-target")` szuka elementu o nazwie tagu `q-target` → `null`
→ `target` zawsze `""`. Skutek: **wszystkie trzy scenariusze QuestEval pokazują
identyczny wynik** `Completeness 0%`, `Faithfulness n/d`, `Wynik łączny n/d`, wszystkie
6 wierszy `fail` / „brak informacji" — mimo że tekst docelowy w textarea zawiera każdy
z faktów. Ścieżka wsteczna jest pusta. Testy QA tego nie łapią, bo wołają
`questevalEval()` bezpośrednio, z pominięciem DOM.

```js
const target = $("#q-target")?.value ?? "";
```

### A2. Kolumna „WARUNEK" w IFEval jest pusta — `deep_sim.html:1930`

```js
<span>${t(name)}</span>     // `nameKey` policzone linijkę wyżej i nieużyte
```

`name` nie jest zadeklarowane → rozwiązuje się do `window.name` (`""`) → `t("")` → `""`.
Tabela pokazuje tylko kolorowe kropki bez nazw warunków. Zamienić na `t(nameKey)`.

### A3. Klucz i18n na przycisku scenariusza — `deep_sim.html:1571`

`PRESET_SPECS.ifeval[0].label = "iPresetGood"`, a w `T` jest `iPresetAll`. Pierwszy
pill w IFEval renderuje dosłownie **`iPresetGood`**.

### A4. Wycieki kluczy i18n w interfejsie

| Miejsce | Widoczny tekst | Linia |
|---|---|---|
| stopka | `shortcuts` | 1618 |
| nagłówek obu torów QA | `nonEmpty` | 1987 |
| karta Completeness | `0 / 6 retainedCount` | 2015 |
| każdy badge PASS/FAIL | `pass` / `fail` małymi literami | 1537 |

`t()` zwraca klucz, gdy go nie ma w słowniku — brak `shortcuts`, `nonEmpty`,
`retainedCount`, `pass`, `fail` w `T.pl`/`T.en`. Warto dodać w dev-buildzie asercję,
która porównuje zbiory kluczy `T.pl` i `T.en` i zgłasza użycie klucza spoza słownika.

### A5. „Prompt: PASS" renderowany w kolorze porażki — `deep_sim.html:211, 1945`

`.overall-gate` ma na sztywno `border:var(--c-negative); background:var(--c-negative-soft)`.
`renderIfevalResults` dokłada klasę `pass`/`fail`, ale **nie ma reguły `.overall-gate.pass`**.
Zielony wynik pokazuje się na różowym tle. (`co_sim` obchodzi to inline-style'em w
`co_sim.html:802` — też do posprzątania, ale przynajmniej działa.)

### A6. Etykiety na osi liczbowej ARES nie są pozycjonowane — `deep_sim.html:1852-1854`

`<span class="point-label truth">` nie dostaje `style="left:…"`, w przeciwieństwie do
`<span class="point truth">`. Efekt: „Prawda 70.0%", „PPI 72.0%" i „Raw 72.0%" leżą
przyklejone do lewej krawędzi, a znaczniki są przy 72 — dane i podpisy są rozjechane.

### A7. Podwójny procent i nieprzeskalowane liczby

* `uncertainty: "95% CI: {lo}–{hi}%"` (523/668) + `li()` już dokleja `%` → **`68.1%–75.9%%`**.
* `formulaHtml` (1866): `raw = ${r.raw.toFixed(2)}%` → **`raw = 0.72%`** zamiast `72.00%`.
  To samo dla `korekta`. Brakuje `*100`.
* `overallFailExpl` (1909): `{p}` dostaje `r.strictInstr.toFixed(0)` (ułamek 0–1) → panel
  mówi **„0 warunków spełnionych"**, gdy Instruction strict = 36%. Powinna być liczba
  zdanych warunków, nie ułamek.

### A8. Tryb AI QuestEval rzuca wyjątkiem — `deep_sim.html:1219`

```js
: questevalAiPrompt($("#q-source").value, $("#q-target").value);
```

W `renderQuestEval()` tekst źródłowy to statyczny `<p>` bez `id` — `#q-source` nie
istnieje. Linia jest **poza `try`**, więc `TypeError` leci w górę, a panel zostaje
na „AI myśli…". Użyć `MODEL.questeval.source`.

---

## B. Wierność faktograficzna

### B1. Model szumu w ARES odwraca sens scenariusza (oba pliki)
`co_sim.html:682`, `deep_sim.html:1347`

```js
if(random() < noiseRate) judge = 1 - judge;   // symetryczny flip
```

Symetryczny flip na asymetrycznej populacji (`trueRate = 0.70`) systematycznie **zaniża**
średnią judge'a o `noise × (2·trueRate − 1)`. Zmierzone:

| scenariusz | ustawienie | Prawda | Raw judge | Korekta |
|---|---|---|---|---|
| `co_sim` „Próba bazowa" | bias 0, noise 10 | 70% | **64,2%** | +7,5% |
| `co_sim` „Judge z biasem" | bias **+18** (zawyża) | 70% | **67,4%** | **+6,7%** |
| `deep_sim` bias 0, noise 15 | — | 70% | **66,2%** | — |

Czyli: pill mówi „Judge z biasem · **zawyża**", a oś pokazuje judge'a **poniżej** prawdy,
a audyt koryguje **w górę**. Podpis „Zawyżający judge tworzy fałszywie dodatnie oceny,
które wykrywa audyt" jest sprzeczny z tym, co widać. Bazowy (nieskażony) judge ma
największą korektę ze wszystkich trzech scenariuszy.

Poprawka: rozdzielić szum na dwie stawki zachowujące średnią
(`P(0→1) = noise·trueRate`, `P(1→0) = noise·(1−trueRate)`), albo nakładać bias **po**
szumie i kalibrować presety tak, żeby bias dominował.

### B2. Marker „Prawda" to parametr generatora, nie prawda populacji (oba pliki)
`co_sim.html:791` (`result.trueRate`), `deep_sim.html:1852` (`truthRate`)

PPI szacuje **średnią prawdy w wylosowanej puli N**, a nie `p` rozkładu Bernoulliego.
Przy N = 500 i p = 0,70 realizacja to np. 0,72 — i wtedy PPI = 72,0% wygląda jak
„pudło o 2 pp" względem markera 70%, mimo że jest dokładnie trafione. Marker powinien
liczyć `mean(truth[])`.

### B3. Podstawa prawna w `deep_sim` jest błędna — `deep_sim.html:779,783`

```
"ustawa z dnia 5 lipca 2018 r."
"rozporządzenie w sprawie profilu zaufanego z 2021 r."
```

5 lipca 2018 r. to ustawa o krajowym systemie cyberbezpieczeństwa — nie dotyczy profilu
zaufanego. Właściwe akty (i użyte poprawnie w `co_sim`, `co_sim.html:533-534`):
**Dz.U. 2005 nr 64 poz. 565** (ustawa o informatyzacji) oraz **Dz.U. 2020 poz. 1194**
(rozporządzenie MC z 29.06.2020 w sprawie profilu zaufanego i podpisu zaufanego).
Te stringi trafiają do promptu wysyłanego do modelu, do „dobrej" odpowiedzi wzorcowej
i do kolumny dowodów — błąd jest widoczny w trzech miejscach naraz.

`co_sim` dodatkowo linkuje źródła scenariusza (gov.pl, ISAP) w nagłówku każdej metody —
to jest dokładnie ta rzecz, której `deep_sim` brakuje.

### B4. `deep_sim` zakazuje prawdziwych metod założenia PZ — `deep_sim.html:775-776`

```js
exclude: ["bankowość elektroniczna|e-dowód|mObywatel", "14 dni|czternaście dni"]
```

Bankowość elektroniczna i e-dowód to **realne, podstawowe** sposoby założenia profilu
zaufanego. Ustawienie ich jako „zabronione" sprawia, że wzorcowa „dobra odpowiedź"
w demo jest merytorycznie niepełna, a widz może wyjść z przekonaniem, że tak nie można.
Mechanikę exclude lepiej pokazać na czymś rzeczywiście niepożądanym (np. „14 dni",
które już tam jest, albo zmyślona opłata).

### B5. Scenariusz „Poprawny wynik z wstępem" (IFEval, `deep_sim`) nie testuje tego, co obiecuje

Podpis: „Wariant loose usuwa wstęp i poprawnie parsuje JSON; strict go nie akceptuje".
Ale `ifevalConstraints()` (1876-1882) włącza tylko `includeTerms`, `excludeTerms`, `citeTerms` —
**`validJson` nigdy nie jest aktywny w UI**. Zmierzony wynik: strict 45%, loose 45%,
oba `FAIL`. Loose niczego nie odzyskuje. Cała historia strict vs loose — jedyny powód,
dla którego ten scenariusz istnieje — nie jest widoczna.

Przy okazji: cała lista `IFEVAL_CONSTRAINTS` (1395-1406, 10 pozycji) i 6 z 7 gałęzi
`ifevalCheck` (`requiredWord`, `forbiddenWord`, `maxWords`, `exactItems`, `noComma`,
`exactEnding`) to martwy kod w UI. `co_sim` w tym punkcie wypada lepiej — jego preset
„Wstęp przed JSON" rzeczywiście rozdziela strict `FAIL` / loose `PASS`.

### B6. Domyślny stan ARES w `deep_sim` niczego nie uczy

Preset `baseline` = `{bias: 0, noise: 0, n: 80}`. Judge jest **idealny**: 0 fałszywie
dodatnich, 0 fałszywie ujemnych, korekta `0.0 pp`, PPI = Raw. To jest ekran, który widz
ogląda po wejściu w zakładkę ARES. Preset `biased` (bias 18, noise 5, n 80) też pokazuje
**`0.0 pp`** — fp = 3 i fn = 3 znoszą się przypadkiem przy tym ziarnie. Sprawdzone:

```
bias18 noise5 n= 30 → korekta -3.33pp
bias18 noise5 n= 60 → korekta  0.00pp
bias18 noise5 n= 80 → korekta  0.00pp   ← preset „Zawyżający judge"
bias18 noise5 n=150 → korekta  0.00pp
bias18 noise5 n=300 → korekta -2.00pp
```

Presety powinny być dobierane pod **efekt widoczny na ekranie**, a nie pod ładne liczby
w konfiguracji — warto dodać test QA typu „preset «biased» daje |korekta| ≥ 3 pp".

### B7. `deep_sim` nie ma stemmera, `co_sim` ma

`deep_sim.normalize()` (843) tylko tokenizuje. Pytanie „Jak **założyć** profil zaufany?"
vs odpowiedź „Profil zaufany **zakłada** się…" → brak dopasowania → Answer relevance 67%
zamiast 100%. `co_sim` ma tablicę `STEMS` (`co_sim.html:596`) i radzi sobie poprawnie.
Dla polskiego to nie kosmetyka, tylko warunek sensowności metryki.

### B8. Katalog modeli AI do weryfikacji — `deep_sim.html:882-903`

`gemini-3.6-flash`, `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `claude-opus-4-8`.
Identyfikatory Claude w bieżącej linii to `claude-opus-5`, `claude-sonnet-5`,
`claude-haiku-4-5-20251001` — `claude-opus-4-8` nie pasuje. Pozostałe wymagają
sprawdzenia w dokumentacji dostawców. Zły identyfikator = `404` w trakcie demo na żywo.

### B9. Rozjazd `benches.md` ↔ aplikacje

`benches.md` zawiera konkretne liczby, których żadna z aplikacji nie pokazuje, a które
są najmocniejszym materiałem edukacyjnym:

* IFEval: **541 promptów, 25 typów instrukcji** (brak w obu appkach)
* ARES: **DeBERTa-v3-Large ×3**, próg **~150 etykiet**, w eksperymentach głównych **300**
  — a suwak `n` w `deep_sim` startuje od 10 i domyślnie stoi na 60, czyli **poniżej progu
  użyteczności opisanego we własnej dokumentacji**. Warto oznaczyć strefę `n < 150` na
  suwaku jako „poniżej progu ARES".
* QuestEval: PLainBench używa **5 pytań na tekst** — `deep_sim` ma 6 faktów źródłowych,
  `co_sim` 5. Drobiazg, ale łatwo zsynchronizować.

---

## C. Poprawność reprezentacji wizualnej

### C1. `deep_sim` RAGAS: fragment „zbędny", który wspiera twierdzenie — `deep_sim.html:1306`

```js
const useful = overlap(cw, qWords).length > 0;   // `connected` policzone i nieużyte
```

Na zrzucie: **Fragment 3 ma badge „zbędny"**, a Twierdzenie 3 wskazuje go jako swoje
źródło (`Fragment 3` w evidence-ref) i biegnie do niego strzałka. Widz widzi dwie
sprzeczne informacje o tym samym obiekcie. `co_sim` liczy to poprawnie:
`useful: claimIds.length > 0 || questionTerms.length >= 2` (`co_sim.html:658`).
Wystarczy `const useful = overlap(cw, qWords).length > 0 || connected;`.

### C2. Strzałki w mapie dowodów są dekoracją, nie danymi

`deep_sim.html:1785`: `r.claims.filter(c=>c.supported).map(()=>'<div class="connector"></div>')`
— renderuje N identycznych strzałek rozłożonych równomiernie. Sugerują mapowanie 1:1
fragment→twierdzenie, którego nie ma (Twierdzenie 1 cytuje Fragment 1 **i** 2).
`co_sim.html:777` robi to samo tekstowo: `"F → C1"` — bez numeru fragmentu, więc
też nie niesie informacji. Albo narysować prawdziwe krzywe SVG z `claim.fragments`,
albo zrezygnować i zostawić same chipy `Fragment n` (które są poprawne).

### C3. Szablony `{n} / {n}` — mianownik nigdy się nie renderuje

`deep_sim.html:508,509,551`:

```js
claimsSupported: "{n} / {n} twierdzeń wspieranych",
conceptsCovered: "{n} / {n} pojęć pokrytych",
retainedLabel:   "{n} / {n} zachowanych",
groundedLabel:   "{n} / {n} ugruntowanych",
```

Wołane z `{n: …, total: …}` — `{total}` nie ma w szablonie, `{n}` podstawia się dwa razy.
Widoczny efekt na zrzucie: **„Answer relevance 67% — 2 / 2 pojęć pokrytych"**. Każdy
mianownik równy licznikowi, przy dowolnym wyniku. `fragmentsUseful` używa `{ok}/{n}`
i jest jedynym poprawnym z czwórki.

### C4. Próbka kropek ARES zawyża udział audytu — `deep_sim.html:1372`

`every = floor(N / 120)`, potem `for(i = 0; i < N; i += every)` — próbkuje **co czwarty
indeks**, a `labeled` to prefiks przetasowanej listy, więc audytowane trafiają do próbki
nieproporcjonalnie. Na zrzucie: `n = 80 / N = 500` (16%), a różową obwódkę ma ~1/3 kropek.
`co_sim` (`co_sim.html:697`) próbkuje analogicznie, ale przy `n = 200/500` proporcja
przypadkiem się zgadza. Poprawnie: próbkować losowo z tym samym ziarnem i skalować liczbę
audytowanych do `round(120 · n/N)`.

### C5. `co_sim` QuestEval pokazuje wzorce matchera zamiast odpowiedzi — `co_sim.html:806-808`

Kolumny „ODPOWIEDŹ ŹRÓDŁA" / „ODPOWIEDŹ CELU" renderują `fact.sourceEvidence`, czyli
wyjście `factEvidence()`: **`potwierdzic tozsamosc · potwierdzenie tozsamosci`**,
`urzad gminy · urzad skarbowy`, `bezplatna · bezplatnie` — bez polskich znaków, z listą
akceptowanych wariantów. To nie jest odpowiedź na pytanie QA, to wnętrze matchera.

Ironia: `QUEST_FACTS` **ma** ładne pole `sourceAnswer` („punkt potwierdzający · urząd
gminy · urząd skarbowy", `co_sim.html:540-545`) — i nigdzie go nie renderuje.
`deep_sim` używa tu `f.gold` i wygląda znacznie lepiej.

### C6. Oś liczbowa ARES: trzy markery w odległości 2–3 pp

W obu aplikacjach Prawda / Raw / PPI zlewają się w jedną plamę na osi 0–100%.
Sugestia: oś przyciąć do zakresu `[min−10 pp, max+10 pp]` albo dodać nad nią
mini-wykres „raw → +korekta → PPI" jako trzy etykietowane słupki. Sam mechanizm PPI —
najważniejsza rzecz w tej zakładce — jest dziś nieczytelny.

### C7. Potrójne powtórzenie tej samej dyskusji

Na jednym ekranie `co_sim` napis „Tryb edukacyjny" pojawia się jako (1) eyebrow nad
tytułem, (2) tytuł żółtej karty po prawej, (3) treść żółtego banera `proxy-banner`
w kanwie — z **identycznym** tekstem w (2) i (3). Do jednego miejsca.

### C8. Drobiazgi wizualne

* `deep_sim` prompt IFEval: `…z 2021 r..` — podwójna kropka (`buildIfevalPrompt`, 1883).
* `deep_sim` suwak „Ukryta proporcja pozytywna" ma pusty wiersz etykiety (`sliderRow("", …)`, 1804)
  i opis `populationHelp` zduplikowany z sąsiedniej grupy.
* `deep_sim` `renderAiChip` (1233) robi `el.textContent = …`, kasując `<span class="model-dot">` —
  zielona kropka statusu znika po pierwszym renderze.
* `co_sim` chip „AI · optional" otwiera dialog „Ta wersja nie przechowuje kluczy API" —
  wygląda jak wybór modelu, a nie robi nic. Albo usunąć, albo opisać jako „tryb lokalny".
* Oba: Google Fonts z CDN. W środowisku bez sieci (a takie bywają sale demo) typografia
  marki znika. Dla demo warto osadzić woff2 w base64 albo zrezygnować z Raleway/Geist Mono.

---

## D. Optymalizacja, sprzątanie, refaktor

### D1. Duplikacja między plikami — największa pozycja

182 z 302 linii CSS `deep_sim` jest **bajt w bajt identycznych** z `co_sim` (60%).
Zduplikowane też: `escapeHtml`/`esc`, `mulberry32`, `variance`, `overlap`, silnik ARES,
strict/loose IFEval, matcher QuestEval, rusztowanie i18n, `method-shell` / `preset-bar` /
`log-details`. Razem 217 KB HTML na dwa warianty tego samego demo.

Propozycja: `shared/tokens.css`, `shared/engines.js` (4 czyste funkcje `*Eval`),
`shared/i18n.js`, `shared/shell.js`. Zostają dwa cienkie pliki różniące się warstwą
prezentacji i fixture'ami. Wtedy poprawka B1 (model szumu) czy C3 (`{n}/{n}`) robi się
raz, a nie dwa razy — a dziś ten sam błąd siedzi w obu plikach niezależnie.

### D2. Martwy kod do usunięcia

`deep_sim`:
* `BENCH_STAGES` (398–425, 28 linii) — nigdy nie użyte; razem z nim ~16 kluczy i18n
  `stQuestion`…`stFidelity` × 2 języki.
* `<script src="…lucide.min.js">` (12) — `createIcons()` nie jest wołane **nigdy**.
  Ładuje ~40 KB przez sieć i wywala się z `ERR_TUNNEL_CONNECTION_FAILED` w konsoli
  przy każdym otwarciu. Do wywalenia razem z `BENCH_STAGES`.
* `emptyHtml()` (2020), `ciLeft()` (1862, przesłonięta lokalnym `const` w tej samej funkcji).
* `IFEVAL_CONSTRAINTS` + 6 gałęzi `ifevalCheck` — patrz B5.
* Nieużywane klucze i18n: `rProxy`, `aPresetBaseNote`, `iPresetAll` (osierocony przez A3).

`co_sim`:
* `qaRow(fact, direction)` — parametr `direction` nieużywany; `qaLane` liczy go i przekazuje.
* `clone()` (621) obok `structuredClone` (562) — dwie drogi do tego samego.
* `STOP_WORDS` (595): `"…oraz można można"` — wpisy z polskimi znakami nigdy się nie
  dopasują, bo `tokens()` normalizuje przed sprawdzeniem. Działa tylko wariant `mozna`.
  Do tego duplikat i mieszanka PL/EN w jednym zbiorze.

### D3. Zbędna praca przy każdym renderze

* `co_sim.html:869` — `renderQa()` woła `runQaTests()` **bezwarunkowo**, po czym
  odrzuca wynik, gdy panel jest schowany. A `renderMethod()` woła `renderQa()` zawsze.
  Każde naciśnięcie presetu / klawisza uruchamia 9 testów, w tym pełną symulację ARES
  na 100 wierszach. Przenieść wywołanie za `if(!state.showQa) return;`.
* `deep_sim.html:2080` — `evaluate()` woła `renderShell()`, które przepisuje `#tabs`
  i `#footer-meta` przy **każdym znaku** wpisanym w textarea (listener `input` → `evaluate(false)`).
  Do tego `runProcess()` przepisuje terminal. Wystarczy `updateResults()`.

### D4. Kontrakt DOM niestabilny względem języka

`co_sim.html:762`: `data-metric="${escapeHtml(label)}"` — atrybut maszynowy dostaje
**przetłumaczoną** etykietę, więc każdy selektor testowy pęka po przełączeniu PL/EN.
`deep_sim` używa stałych kluczy (`data-metric="faithfulness"`) — to jest właściwe podejście.

### D5. Skróty klawiszowe

* `co_sim.html:937`: `matches("input,textarea,select,button")` — po kliknięciu w preset
  fokus zostaje na `<button>`, więc `1`–`4` i `R` przestają działać, mimo że stopka je
  reklamuje. `<button>` nie jest polem edycji — wyjąć z tego selektora.
* `deep_sim.html:2414`: `else if(e.key === "r") applyPreset("good")` — ARES i QuestEval
  nie mają presetu `good`, więc `R` jest w nich no-opem. Zrobić cykl po `PRESET_SPECS`,
  jak w `co_sim`.
* `deep_sim`: `Enter` poza polem edycji uruchamia `evaluate()`; po kliknięciu przycisku
  fokus jest na nim i `Enter` odpala akcję dwa razy.

### D6. Spójność statystyczna

`co_sim.variance` dzieli przez `n−1` (próbkowa), `deep_sim.variance` przez `n`
(populacyjna). Te same wzory, dwa wyniki. Przy wspólnym module (D1) — jedna definicja,
udokumentowana.

### D7. Bezpieczeństwo / prywatność (kontekst demo)

Escaping jest zrobiony konsekwentnie (`esc`/`escapeHtml` na każdej wstawce treści) —
w porządku. Warstwa transportu AI w `deep_sim` (1039–1084) jest napisana bardzo dobrze:
bounded timeout, brak retry na POST (ryzyko podwójnego rozliczenia), poprawny parser SSE
z obsługą `remainder`, rozpoznawanie `stop_reason`. To najlepszy fragment kodu w obu
plikach — warto go wyciągnąć jako moduł niezależnie od reszty refaktoru.

Jedna uwaga produktowa: klucze API leżą w `localStorage` w czystym tekście i są wysyłane
z przeglądarki bezpośrednio do dostawcy. Jest to uczciwie opisane w `aiKeyHelp`, ale przy
demo na cudzym sprzęcie warto dodać przycisk „wyczyść klucz" i nie zapisywać klucza,
dopóki użytkownik świadomie nie zaznaczy „zapamiętaj".

---

## Kolejność prac (proponowana)

1. **A1–A8** — poprawki jednoliniowe, przywracają `deep_sim` do stanu pokazywalnego. ~1 h.
2. **B1 + B6** — model szumu i presety ARES; bez tego zakładka ARES uczy rzeczy nieprawdziwej
   w **obu** aplikacjach.
3. **B3 + B4** — treść merytoryczna `deep_sim` (podstawa prawna, lista exclude) plus
   przeniesienie linków do źródeł z `co_sim`.
4. **C1, C3, C5** — sprzeczności między liczbą a etykietą; tanie, a bardzo widoczne.
5. **D3 + D2** — wydajność i martwy kod.
6. **D1** — wspólny moduł; dopiero po ustabilizowaniu semantyki.
7. **B9** — dociągnięcie liczb z `benches.md` do interfejsu (541 promptów, próg 150 etykiet).
