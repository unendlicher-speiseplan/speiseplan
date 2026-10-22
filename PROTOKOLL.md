# PROTOKOLL: Unendlicher Speiseplan (Übergabe an jede KI und jeden Chat)

Stand dieser ZIP: 2026-10-22 05:35 UTC · Paket: App (Ausspielung). Der Quellcode steht in speiseplan-projekt.zip.

**Zweck dieser Datei:** Ein neuer Chat oder eine andere KI soll nur diese ZIP lesen und ohne Rückfragen weiterarbeiten können. Alles Wichtige steht hier: Vorgaben des Auftraggebers, Architektur, Datenformate, Teststand, bekannte Fallen, offene Aufgaben und ehrliche Grenzen. Die Zahlen in Abschnitt 3 und 4 sind beim Packen automatisch aus dem Quellcode erzeugt und stimmen mit dieser ZIP überein. Alles andere ist von Hand geschrieben. Bei Widerspruch gilt der Quellcode.

---

## 0. Kurzanleitung zum Weiterarbeiten (in dieser Reihenfolge)

1. Diese Datei vollständig lesen, besonders Abschnitt 1 (Vorgaben) und Abschnitt 9 (Arbeitsregeln).
2. Voraussetzung ist nur **Node.js** (ab Version 18, getestet mit 22). Keine npm-Pakete nötig. Für den optionalen Browsertest zusätzlich Python 3 mit Playwright und Chromium.
3. Im Projektordner ausführen: `node build.js`. Das erzeugt `speiseplan.html` (Variante für Claude-Artifacts) und den Ordner `dist/` (eigenständige App als PWA: `index.html`, `sw.js`, `manifest.json`, Icons).
4. Tests ausführen (alle müssen grün sein, Erwartungswerte in Abschnitt 5): `node test_logic2.js`, `node test_logic3.js`, `node smoke2.js`, `node smoke3.js`.
5. Änderung machen, in den Quelldateien (nie in `dist/` oder `speiseplan.html`, die werden überschrieben).
6. Erneut `node build.js` und alle Tests.
7. Mit `node make_zip.js` neu packen. Das Skript baut, testet, erzeugt dieses Protokoll mit frischen Zahlen neu und packt beide ZIPs. **Jede ZIP muss dieses Protokoll enthalten.** Das ist eine Vorgabe des Auftraggebers.
8. Im Änderungsprotokoll (Abschnitt 11, in `PROTOKOLL.template.md`) einen Eintrag ergänzen.

Hinweis: Diese ZIP enthält nur die fertige App. Für Änderungen den Quellcode aus `speiseplan-projekt.zip` verwenden. Auf GitHub genügt es, `index.html` und `sw.js` zu ersetzen; `PROTOKOLL.md` muss nicht hochgeladen werden.

Wer kein Node hat: Das fertige Produkt ist `dist/index.html`. Es läuft in jedem modernen Browser, ohne Server.

---

## 1. Vorgaben des Auftraggebers (verbindlich, nicht ohne Rückfrage ändern)

### 1.1 Projektziel
Ein digitales, deutschsprachiges Speiseplan-System namens **„Unendlicher Speiseplan“** mit Rezeptbibliothek (geplant 240 Rezepte), Wochenplan, Einkaufsliste, Nährwerten, Allergenen und Unverträglichkeiten, Vorrat (Zutaten, Gewürze, Gefrierer), einem Tagebuch im Stil von Yazio und einem Bereich für modulares Meal-Prep im Gefrierer (Souper-Cubes-Prinzip). Die App ist für den Alltag einer einzelnen Person gebaut.

### 1.2 Sprache, Einheiten, Ernährung
- Alles auf **Deutsch**. Metrische Einheiten (g, ml, °C). Flüssigkeiten in Gramm gerechnet (1 ml ≈ 1 g). Ein Ei = 55 g, 1 EL ≈ 15 g, 1 TL ≈ 5 g, 1 Tasse ≈ 240 g.
- **Kalorienbewusst und proteinreich.** Voreingestellte Tagesziele: 1600 kcal, 120 g Protein, 2000 ml Wasser (frei einstellbar). Richtwert für Rezepte: etwa 400 bis 650 kcal und möglichst 30 g Protein oder mehr je Portion.

### 1.3 Geräte
Vorhanden: Airfryer, Kontaktgrill, Multikocher, Sous-vide, Herd, Backofen, Dutch Oven. **Monsieur Cuisine Connect** nur mit nachweislich geprüften Programmdaten (aktuell gibt es keine solchen Daten, deshalb keine Varianten dafür). **Thermomix bekommt keine Priorität.** In `GERAETE` steht Monsieur Cuisine noch als Wert, in der Rezept-Eingabe ist er ausgeblendet.

### 1.4 Feste Ausschlüsse (überall, ohne Ausnahme)
**Süßkartoffel(n), Bataten (Batate), Ananas**, einschließlich aller Schreibweisen und Zusammensetzungen (zum Beispiel „Sweet Potato“, „Ananasstücke“). Das gilt für Zutaten, Rezeptnamen, Schritte, Tags, Tipps, Importe, KI-Antworten und Einkaufslisten. Umgesetzt in `FIXE_AUSSCHLUESSE`, `EXCL_KEYS`, `exclusionHits()`, `recipeExclusionHits()`. Jeder neue Text und jeder Import muss diese Prüfung bestehen. Die Prüfung wird auch für die Bausteine im Meal-Prep-Bereich ausgeführt. Weitere persönliche Ausschlüsse sind unter „Mehr → Allergene, Unverträglichkeiten, Ausschlüsse“ einstellbar (`state.profile.excl`).

### 1.5 Allergene, Unverträglichkeiten, Histamin
- Alle **14 EU-Hauptallergene** (`ALLERGENE`) plus Unverträglichkeiten (`UNVERTR`: Laktose, Histamin, FODMAP, Hülsenfrüchte, Fruktose). Beides wird aus den Zutaten abgeleitet (`derive()`). Fertigprodukte können mehr enthalten, die Packung ist zu prüfen.
- **Histamin nur als Risikoprofil** (niedrig, mittel, erhöht). **Nie „histaminfrei“ behaupten.**

### 1.6 Nährwerte und Wahrhaftigkeit
- **Transparent aus den Zutaten berechnet** (`calcRecipe()`), mit Quelle BLS 4.0 (je Zutat in `data_bls.js`, sonst als eigener Packungs- oder Richtwert gekennzeichnet). **Keine erfundenen Werte.**
- Gewürze und kleine Kräutermengen fließen nicht in die Nährwerte ein.
- **Nie behaupten, dass Dateien, Tests oder Ergebnisse existieren, ohne es geprüft zu haben.** Berichte müssen ehrlich sagen, was nicht getestet werden konnte.

### 1.7 Statusbegriffe für Rezepte (festes Vokabular)
`Konzept erstellt` · `Entwurf erstellt` · `Automatisch validiert` · `Praktisch getestet` · `Gesamtvalidiert` · `Fertig`.
**„Fertig“ nur, wenn alle Kriterien erfüllt sind.** Die mitgelieferten Rezepte und Bausteine stehen auf **„Entwurf, automatisch validiert“**. „Praktisch getestet“ setzt nur der Nutzer selbst (nach dem Kochen). Von der KI wurde nie ein Rezept gekocht.

### 1.8 Rezept-Herkunft und Urheberrecht
Rezepte sind **authentische Gerichte aus aller Welt, in eigenen Worten geschrieben**. Keine Texte von Anbietern kopieren, keine Marken-Rezepte nachbauen, keine KI-Phantasiegerichte als „authentisch“ ausgeben. Herkunft in `origin`, Quellen in `src` angeben. Wenn die Herkunft unklar ist, „unklar“ schreiben.

### 1.9 Gefrierer und Souper Cubes
Zwölf Silikonformen in vier Größen, großer Gefrierschrank, Ziel 4 bis 6 Wochen Vorrat. Kammern je Größe und Formen-Runde (`KAMMERN`): **50 ml: 18, 125 ml: 12, 250 ml: 16, 500 ml: 8**. Aufbau: 500 ml 4 Formen mit je 2 Kammern, 250 ml 4 Formen mit je 4, 125 ml 2 Formen mit je 6, 50 ml 2 Formen mit je 9 (`FORMEN_TEXT`). Ziel Gefrierer: 40 bis 60 größere Portionen oder Komponenten (Blöcke ab 250 ml).

### 1.10 Sicherheitsregeln (in der App hinterlegt, bei Änderungen beibehalten)
Kerntemperatur Fleisch, Geflügel und Hack mindestens 70 °C (mindestens 2 Minuten). Reste auf mindestens 70 °C erhitzen. Gekochtes höchstens 2 Stunden bei Raumtemperatur. Kühlschrank unter 7 °C, Gefrierer mindestens −18 °C. Auftauen im Kühlschrank. Aufgetautes nicht erneut einfrieren, außer vorher durcherhitzt. Haltbarkeiten im Gefrierer sind Qualitätsrichtwerte.

### 1.11 Ablage und Ausspielung
Das Produkt läuft **ohne Server**: als eigenständige PWA (GitHub Pages) und als Claude-Artifact. Die Daten liegen nur im Browser des Geräts (Export unter „Mehr → Daten und Sicherung“). Ein **Mehrbenutzer-Serverprojekt** (Docker, Rollen, Mealie-Import und -Export) ist ausdrücklich **zurückgestellt**; es startet erst, wenn der Auftraggeber es erneut beauftragt.

---

## 2. Betrieb und Ausspielung

- **Eigenständige App:** `dist/index.html` plus `sw.js`, `manifest.json`, `icon.svg`, `icon-192.png`, `icon-512.png`. Läuft offline. Gehostet auf GitHub Pages (Konto `unendlicher-speiseplan`, Repository `speiseplan`, Pages aus Branch `main`, Ordner `/ (root)`). Der Auftraggeber bestätigte, dass die Seite läuft.
- **Aktualisieren auf GitHub:** Im Repository die Dateien `index.html` und `sw.js` ersetzen (Hochladen über den Browser, die GitHub-Mobil-App kann keine Dateien hochladen). `manifest.json` und Icons ändern sich selten. Der Cache-Name in `sw.js` wird bei jedem Build neu erzeugt, dadurch holt sich die App die neue Version. Falls die alte Version erscheint: App ganz schließen und neu öffnen.
- **Claude-Artifact:** `speiseplan.html` wird mit dem Artifact-Werkzeug unter demselben Dateipfad erneut veröffentlicht (dann bleibt die URL). Gespeicherte Fähigkeiten: `sample` (mit Bildern), `db`, `user`, `downloads`. Nur dort gehen KI-Vorschläge, der KI-Import und das Speichern im Claude-Konto. In der eigenständigen App fehlen diese Funktionen (die Knöpfe erscheinen dann nicht).
- **Speicher:** `localStorage` unter dem Schlüssel `speiseplan-xxl-v1`. Wenn `window.claude` existiert, zusätzlich Dokument `data/users/<uid>/state` mit `{v, json, saved}`.
- **Service Worker** laufen nicht in Artifacts, nur im gehosteten Standalone-Build.

---

## 3. Inhalt dieses Stands (automatisch erzeugt)

| Kennzahl | Wert |
|---|---|
| Mitgelieferte Rezepte | **76** von 240 geplanten |
| Verteilung nach Gruppe | Geflügel: 16, Rind: 13, Schwein: 9, Fisch: 12, Vegetarisch: 26 |
| Verteilung nach Mahlzeit | Hauptgericht: 48, Suppe/Eintopf: 11, Salat/Bowl: 5, Frühstück: 7, Snack: 5 |
| Einfrierbar: ja / eingeschränkt | 29 / 11 |
| Zutaten im Stamm (davon mit Nährwerten) | 166 (165) |
| Gewürze | 42 |
| Bausteine (Meal-Prep) | **35** (Protein: 12, Beilage: 7, Sauce: 6, Gemüse: 5, Booster: 5) |
| Einkaufskategorien | Fleisch & Geflügel, Fisch & Meeresfrüchte, Gemüse, Obst, Milchprodukte, Eier, Tiefkühlprodukte, Trockenprodukte, Konserven, Gewürze & Öle, Sonstiges |
| Geräte-Werte | Airfryer, Kontaktgrill, Multikocher, Sous-vide, Herd, Backofen, Monsieur Cuisine Connect, Dutch Oven |
| Feste Ausschlüsse | Ananas, Süßkartoffel, Batate |
| Größe `dist/index.html` | 391 KB |

---

## 4. Architektur

### 4.1 Prinzip
Eine einzige HTML-Datei mit eingebettetem JavaScript, **zusammengesetzt durch Verkettung** (`build.js`). Es gibt kein Framework, keine Bibliotheken, keine externen Anfragen (Google Fonts nur in der Artifact-Variante). Reine Logik (ohne DOM) ist von der Oberfläche getrennt und in Node testbar.

### 4.2 Reihenfolge der Verkettung (wichtig, alles teilt einen globalen Geltungsbereich)
`logic.js` → `data_zutaten.js` → `data_rezepte1.js` → `data_rezepte2.js` → `data_rezepte3.js` → `SEED2.forEach(r => SEED.push(r))` → `logic2.js` → `data_bausteine.js` → `logic3.js` → `app_a.js` → `app_b.js` → `app_c.js` → `app_d.js` → `app_e.js`.
**`initRuntime()` steht am Ende von `app_e.js` und muss das letzte ausgeführte Statement bleiben**, weil `render()` auf alle Ansichten zugreift.

### 4.3 Dateien
| Datei | Inhalt |
|---|---|
| `logic.js` | Kern: Konstanten (Allergene, Geräte, Kammern), Zutatenstamm (Basis), Nährwert- und Allergenrechnung, Ausschlussprüfung, Rezeptprüfung, KI-Rezept-Normalisierung | 
| `data_zutaten.js` | Zusätzliche Zutaten (insgesamt siehe Abschnitt 3) und Gewürze (`SPICES`, `SMAP`) | 
| `data_bls.js` | Quelle je Zutat: BLS-Code, Modus (bls/naeh/mix/abl/own), früherer Wert (`BLS_REF`) | 
| `data_sousvide.js` | Sous-vide: belegte Zeit-Temperatur-Werte für Geflügel (Anova/USDA) und Regeln (`SOUSVIDE`) | 
| `data_rezepte1.js` | Mitgelieferte Rezepte, Teil 1 (Hilfsfunktion `R()`) | 
| `data_rezepte2.js` | Mitgelieferte Rezepte, Teil 2 | 
| `data_rezepte3.js` | Mitgelieferte Rezepte, Teil 3 | 
| `logic2.js` | Datum, Zutaten-Chargen (FIFO), Einkauf, Tagebuch, Ziele, Fasten, Autoplan, Import/Export | 
| `data_bausteine.js` | Meal-Prep-Bausteine (`MODULE`, `STIL_PASST`, `BAUSTEINE`, `BMAP`) | 
| `logic3.js` | Meal-Prep-Logik: Blockzahl, Prüfung, Mahlzeit bauen, Vorschläge, Session, Einbuchen, Soll/Ist, Schubladen, Etikett | 
| `app_a.js` | Zustand (`defaultState`, `mergeState`), Speicher, Hilfen, Ansicht Heute, Hinzufügen-Blatt | 
| `app_b.js` | Rezepte (Liste, Filter, Detail), Rezept-Editor, KI-Import-Prompt | 
| `app_c.js` | Plan, Einkauf, Vorrat (Zutaten, Gewürze, Gerichte, Kochen), Mehr (inkl. Bibliotheksverwaltung, Status) | 
| `app_d.js` | Tabs, `render`, Blätter, KI-Funktionen, `ACT`/`CHG`/`INP`-Tabellen, Ereignis-Verdrahtung | 
| `app_e.js` | Prep-Bereich (Ansichten und Aktionen); enthält am Ende `initRuntime()` | 
| `head.html` | HTML-Kopf mit allem CSS (Farb-Tokens, Dunkelmodus) | 
| `body.html` | HTML-Gerüst (Kopf, `#view`, `#tabs`, `#sheet`, `#toast`) | 
| `build.js` | Verkettet alles zu `speiseplan.html` (Artifact) und `dist/` (PWA) | 
| `bls_auszug.json` | Auszug der verwendeten BLS-4.0-Einträge (Werte je 100 g, mit Quellenangabe) | 
| `test_logic2.js` | Logiktests (Zutaten, FIFO, Einkauf, Tagebuch, Autoplan, Import) | 
| `test_logic3.js` | Logiktests Meal-Prep (Bausteine, Mahlzeit, Session, Soll/Ist) | 
| `test_bls.js` | Herkunft der Nährwerte: Quelle je Zutat, Abgleich mit `bls_auszug.json`, Energieprüfung | 
| `smoke2.js` | Oberflächentest in Node-VM (App v2) | 
| `smoke3.js` | Oberflächentest Prep und Bibliotheksverwaltung | 
| `pw_prep.py` | Browsertest Prep (Playwright, 400 px, hell/dunkel) | 
| `pw_lib.py` | Browsertest Bibliotheksverwaltung | 
| `make_zip.js` | Dieses Packskript | 
| `mk_prompt.js` | Erzeugt `perplexity-prompt-rezepte.txt` aus den aktuellen Zutatenlisten | 
| `check_recipes.js` | Prüfbericht für Perplexity-Rezepte vor dem Import: `node check_recipes.js datei.json` | 
| `PROTOKOLL.template.md` | Handgeschriebener Teil des Protokolls (Platzhalter `{{…}}` werden beim Packen gefüllt) | 
| `perplexity-prompt-rezepte.txt` | Fertiger Prompt für Perplexity inkl. Importformat und ID-Listen (erzeugt) |
| `PROTOKOLL.md` | Dieses Protokoll (erzeugt) |
| `dist/` | Fertige eigenständige App (erzeugt) |
| `speiseplan.html` | Artifact-Variante (erzeugt) |

### 4.4 Zustand (`state`, Version 2, `mergeState()` in `app_a.js` bereinigt und ergänzt alte Stände)
- `profile`: `avoidAllergens[]`, `avoidIntol[]`, `excl[]`, `goals{kcal, protein, water}`, `body{sex, age, h, w, pal, mode, rate}`
- `favs[]` (Rezept-IDs), `week{Mo..So: {f, m, a, s}}` (je Platz `{rid, fz}` oder null), `shopChecked{}`, `shopExtra[]`
- `inv[]`: Gefrierer- und Vorratsposten `{id, name, rid | bid, ml, count, date, months, exp, loc, mod}` mit `loc` ∈ Gefrierer | Kühlschrank | Vorratsschrank. `rid` = Rezept, `bid` = Baustein, `mod` = Modul (Protein, Beilage, Sauce, Gemüse, Booster, Komplett).
- `stock[]`: Zutaten-Chargen `{id, ing, g, bought, exp}` (FIFO-Abbuchung). `spices{id: {own, level}}`, `minStock{}`
- `userRecipes[]`: eigene und importierte Rezepte (`user: true`, KI-Importe `ai: true`)
- `removed[]`: IDs mitgelieferter Rezepte, die der Nutzer aus der Bibliothek entfernt hat (reversibel, siehe `seedLib()`/`lib()`)
- `tested{}`, `cooked{}` (praktisch getestet, Bewertungen, Notizen), `hideConf` (Rezepte mit Profilkonflikt ausblenden)
- `diary{ISO-Datum: {e[], water, w}}` (Einträge: Rezept `kind:'r'`, Zutat `'z'`, Schnell `'q'`), `fast{start, goalH, log[]}`
- `prep`: `{weeks, perWeek, guests, drawers, plan[{k:'b'|'r', id, n}], checked{}, log[]}`

### 4.5 Oberfläche
- Sechs Tabs (`TABS` in `app_d.js`): **Heute** (Tagebuch), **Rezepte**, **Plan** (Wochenplan, Einkauf), **Vorrat** (Zutaten, Gewürze, Gerichte, Kochen), **Prep** (Start, Katalog, Bauen, Session), **Mehr** (Ziele, Rechner, Profil, Rezept-Bibliothek verwalten, Sicherung, Prüfbericht, Status).
- Muster: Ansichten sind Funktionen, die HTML-Strings liefern (`viewHeute`, `viewRezepte`, `viewPlanTab`, `viewVorrat`, `viewPrep`, `viewMehr`). `render()` setzt `#view`. Blätter (Sheets) über `openSheet({type, id})` und `renderSheet()` (Typen: recipe, add, editor, ing, stockadd, baustein).
- **Ereignisse** sind ausschließlich per Delegation verdrahtet: `data-act` → Tabelle `ACT`, `data-chg` → `CHG`, `data-inp` → `INP`. Neue Aktionen in diese Tabellen eintragen.
- **Zweistufige Bestätigung** für riskante Aktionen: `confirmed(key)` und `CONFIRM_ACTS` (Aktionsname → Schlüssel). Zwei verschiedene Aktionen mit demselben Schlüssel bestätigen sich gegenseitig. Das wurde schon einmal fehleranfällig und ist bei `rmseedall`/`rmseedgroup` per `ui.rmGroup` abgesichert.
- Kein `alert`, `confirm` oder `prompt` (Artifact-Regel). Rückmeldungen über `toast()`.
- Design: CSS in `head.html` mit Farb-Tokens in `:root`, Dunkelmodus per `prefers-color-scheme`, Safe-Area-Abstände, Layout bis 400 px ohne Querscrollen. Die Regel `[hidden]{display:none!important}` ist nötig, weil `.sheet{display:flex}` sonst das Attribut `hidden` übersteuert.

### 4.6 Rezeptformat (Bibliothek und Import)
```json
{"id":"r01","name":"…","meal":"Hauptgericht|Suppe/Eintopf|Salat/Bowl|Frühstück|Snack",
 "group":"Geflügel|Rind|Schwein|Fisch|Vegetarisch","cuisine":"…","origin":"…","src":"…",
 "servings":4,"prep":20,"cook":25,"level":"leicht|mittel|anspruchsvoll",
 "devices":["Herd"],"tags":["Meal Prep"],"mp":true,
 "ing":[["haehnchenbrust",600],["zwiebel",150]],"spices":["salz","pfeffer"],
 "steps":["…"],"tips":"…","storage":"…","reheat":"…",
 "freeze":{"ok":"ja|eingeschränkt|nein","block":500,"months":3,"note":"…"}}
```
- `ing`: Paare `[Zutaten-ID, Gramm]` für **alle** Portionen. IDs müssen in `ZMAP` (aus `data_zutaten.js` und `logic.js`) stehen. `["gewuerze", 2..15]` ist ein erlaubter Platzhalter ohne Nährwert. Gewürze gehören in `spices` (IDs aus `SMAP`).
- Es gibt **keine Nährwerte im Rezept**, sie werden aus den Zutaten berechnet. Es gibt keine Zutat „Wasser“ (Flüssigkeit als passende Zutat, zum Beispiel Brühe, oder weglassen).
- `freeze.block` ∈ 50, 125, 250, 500 ml je Portion.
- Import (Rezepte → Neu → „JSON importieren“) akzeptiert ein Objekt, ein Array oder `{recipes:[…]}`. Jedes Rezept läuft durch `normalizeUserRecipe()` und `validateRecipe()`. Geprüft werden Pflichtfelder, Zutaten im Stamm, Gewürze im Stamm, plausible Mengen, Ausschlüsse, Energie-Plausibilität, Portionsgewicht, Zeitrechnung, Dubletten, Ziele, Gefrierangaben, Profilkonflikte. Abgelehnte Rezepte werden mit Grund gemeldet.

### 4.7 Baustein-Format (Meal-Prep, `data_bausteine.js`)
Ein Baustein ist eine **Charge einer Grundzubereitung**, die in mehrere gleich große Blöcke eingefroren wird.
```js
B({ name, mod:'Protein|Beilage|Sauce|Gemüse|Booster', group, style, cuisine, ml:250,
    cooked:1250, dens:1, prep, cook, ing:[[id,g]], spices:[…], steps:[…], tips, months,
    devices?, thaw?, reheat? })
```
- `n = max(2, round(cooked / dens / ml))` Blöcke je Charge (in `logic3.js` berechnet), `servings = n`. Dadurch liefert `calcRecipe(b).pp` direkt die Werte **je Block**.
- **`cooked` (Gewicht nach dem Garen) und `dens` sind Schätzwerte (Richtwerte), keine Messungen.** `validateBaustein()` prüft die Plausibilität über den Füllgrad (0,9 bis 1,12).
- Modul-Blockgrößen: Protein 250 ml, Beilage 250 ml, Sauce 125 ml, Gemüse 250 oder 500 ml (Suppen), Booster 50 ml.
- `STIL_PASST` liefert nur **Warnungen** bei unpassenden Geschmacksrichtungen, nie harte Fehler.
- Komplettgerichte im Prep-Bereich stammen aus den einfrierbaren Bibliotheksrezepten (`freeze.ok` ≠ nein, Block 50 bis 500 ml).

### 4.8 Wichtige Logik-Bausteine (Namen zum Auffinden)
- Nährwerte und Prüfung: `calcRecipe`, `derive`, `conflicts`, `energyCheck`, `validateRecipe`, `runQA`, `exclusionHits`.
- Vorrat und Einkauf: `newLot`, `lotState`, `stockTotals`, `deductFifo`, `recipeNeed`, `coverage`, `whatCanICook`, `expiringSoon`, `lowStock`, `weekNeed`, `shopPlan`, `fmtGrams`.
- Tagebuch und Ziele: `entryRecipe`, `entryIng`, `entryQuick`, `diaryTotals`, `calcGoals`, `fastStatus`, `autoPlan`, `weekSummary`.
- Import und Sicherung: `normalizeUserRecipe`, `parseRecipeJson`, `exportState`, `importState`, `diaryCsv`.
- Meal-Prep: `validateBaustein`, `runQABaukasten`, `cubePP`, `mealBuild`, `suggestMeals`, `sessionYield`, `sessionNeeds`, `prepShopping`, `sessionTimeline`, `bookSession`, `sollBestand`, `istBestand`, `prepRoundsFor`, `etikett`.
- Rezeptverwaltung: `seedLib`, `lib`, `byIdMap` (enthält auch Bausteine), `libMap` (nur Rezepte), `dropRecipeRefs`.
- Vollständiges Verzeichnis der Funktionen je Datei: Abschnitt 12.

---

## 5. Tests (Erwartungswerte beim Stand dieser ZIP, automatisch erzeugt)

| Befehl | Ergebnis | Status |
|---|---|---|
| `node test_logic2.js` | 78 Prüfungen | bestanden |
| `node test_logic3.js` | 141 Prüfungen | bestanden |
| `node test_bls.js` | 9 Prüfungen | bestanden |
| `node smoke2.js` | 153 Prüfungen | bestanden |
| `node smoke3.js` | 86 Prüfungen | bestanden |

Weitere Hinweise zu den Tests:
- `smoke2.js` und `smoke3.js` laden `dist/index.html` in eine Node-VM mit einem kleinen DOM-Ersatz und klicken über die Aktionstabellen. Sie prüfen unter anderem, dass nirgends `undefined`, `NaN`, `[object` oder Ausschluss-Zutaten im HTML stehen.
- Beim Schreiben von Tests als Template-String Rückwärtsstriche in Regex doppelt schreiben (`\\[object`), sonst entstehen Syntaxfehler im Test statt in der App.
- `pw_prep.py` und `pw_lib.py` sind Browsertests (Playwright, Chromium, Fenster 400 px, hell und dunkel) mit Screenshots. Sie prüfen Konsolenfehler und Querscrollen. Chromium-Pfad im Beispiel: `/opt/pw-browsers/chromium`, Start mit `--no-sandbox`. Passe den Pfad an deine Umgebung an.
- Beim Test im echten Browser wurde ein Fehler gefunden, den die Node-Tests nicht sehen: das `hidden`-Attribut wurde von CSS übersteuert.

---

## 6. Ehrliche Grenzen des aktuellen Stands

- **Rezepte:** 76 von 240 geplanten sind mitgeliefert. Status „Entwurf, automatisch validiert“, **von der KI nicht gekocht**. Der Nutzer kann Rezepte entfernen und durch eigene oder importierte ersetzen.
- **Bausteine:** 35 Bausteine als eigene Grundzubereitungen, Status „Entwurf, automatisch validiert“. Ausbeute, Blockanzahl und Küchenzeiten sind Schätzungen. Nichts davon wurde gekocht oder eingefroren.
- **Nährwerte:** 148 von 166 Zutaten nach BLS 4.0 (Quelle und Code je Zutat in `data_bls.js`, Auszug in `bls_auszug.json`, geprüft von `test_bls.js`), 18 Zutaten ohne BLS-Eintrag mit eigenem Packungs- oder Richtwert (begründet). Das gilt je 100 g der Zutat; die Mahlzeitenwerte sind daraus gerechnet. Marken, Sorte, Reifegrad und Garverluste (Rezeptmengen sind Rohgewichte) lassen Abweichungen im Alltag zu. „Absolut“ ist nicht erreichbar, nachvollziehbar und belegt schon.
- **Nicht testbar durch die KI:** Funktionen, die nur im Claude-Artifact laufen (KI-Vorschläge, KI-Import, Speichern im Claude-Konto, Downloads über `downloads`), sowie die Installation der App als PWA auf einem echten Handy. Der Nutzer bestätigte, dass die gehostete App läuft.
- **Autoplan:** rechnet je Platz mit einer Portion. Gut für etwa 1500 bis 1900 kcal. Proteinziele über etwa 130 g werden nur an einem Teil der Tage erreicht.
- **Nicht enthalten:** Monsieur-Cuisine-Varianten, Mealie-Austausch (nur mit echtem Export testbar), PDF-Rezeptkarten und Rezeptbuch, Barcode-Scan und Kamera, Mehrbenutzer-Server.
- **Daten liegen nur im Gerät.** Ohne Sicherung gehen sie bei gelöschten Browserdaten verloren.

---

## 7. Offene Aufgaben (Reihenfolge nach Rückmeldung des Auftraggebers)

1. **Perplexity-Rezeptimport:** Der Auftraggeber will die mitgelieferten Rezepte nach und nach durch Rezepte aus Perplexity ersetzen. Dafür gibt es `perplexity-prompt-rezepte.txt` (erzeugt von `mk_prompt.js` mit den aktuellen Zutaten- und Gewürzlisten) und die Bibliotheksverwaltung. Geliefertes JSON bitte vor dem Import gegen die Regeln aus Abschnitt 1 prüfen.
2. Optional: ein eigener Perplexity-Prompt für **Bausteine** im Baustein-Format (nur wenn der Auftraggeber es will; die Bausteine wurden bewusst nicht entfernt).
3. Rezeptbibliothek Richtung 240 ausbauen (Verteilung bei 120 Hauptmahlzeiten: 36 Geflügel, 30 Rind, 18 Schwein, 18 Fisch, 18 vegetarisch; Frühstück und Snacks zählen nicht mit).
4. Rückmeldungen aus dem ersten echten Kochen und Einfrieren einarbeiten (Blockzahlen, Garzeiten, Garverlust).
5. Später, nur auf Auftrag: PDF-Rezeptkarten und Rezeptbuch, Mealie-Austausch, Mehrbenutzer-Server (Docker, Rollen, Mealie-Import und -Export).

---

## 8. Entscheidungen und Begründungen (Auszug)

- **Eine Datei, keine Abhängigkeiten:** läuft überall, offline, leicht zu sichern und zu hosten.
- **Nährwerte nie im Rezept speichern:** immer aus den Zutaten rechnen, so sind sie nachvollziehbar und bleiben bei Änderungen richtig.
- **Entfernen statt Löschen für mitgelieferte Rezepte:** `state.removed` blendet nur aus, „Wiederherstellen“ holt alles zurück.
- **Blockberechnung der Bausteine über Gewicht und Dichte:** erlaubt Nährwerte je Block, ohne Messwerte zu erfinden. Deshalb offen als Richtwert gekennzeichnet.
- **Stil-Verträglichkeit nur als Warnung:** der Nutzer entscheidet, was ihm schmeckt.
- **Zeitangaben der Session sind Richtwerte:** Küchenzeit = Vorbereitung + längste Garzeit + 60 Minuten Abkühlen und Füllen.
- **Runden-Rechnung:** Runden = max über die Größen von ceil(Blöcke / Kammern). Eine Runde braucht etwa 8 bis 12 Stunden zum Durchfrieren.
- **Zweistufige Bestätigung** statt Browser-Dialogen.

---

## 9. Arbeitsregeln für jede KI, die hier weiterarbeitet

1. **Deutsch antworten.** Kurz und konkret. Am Ende: was entstanden ist, welche Datei, ein sinnvoller nächster Schritt.
2. **Ehrlich berichten.** Nur behaupten, was geprüft wurde. Nicht Getestetes ausdrücklich nennen.
3. **Vorgaben aus Abschnitt 1 nie stillschweigend ändern.** Bei Konflikt nachfragen.
4. **Nie ohne Tests ausliefern.** `build.js`, alle Tests, bei Oberflächenänderungen zusätzlich ein Browsertest bei 400 px in Hell und Dunkel.
5. **Keine neuen Ausschluss-Zutaten, keine erfundenen Nährwerte, keine kopierten Rezepttexte.**
6. **Bei Änderungen an Daten (Zutaten, Gewürze) den Perplexity-Prompt neu erzeugen** (`node mk_prompt.js`, macht auch `make_zip.js`).
7. **Jede ausgelieferte ZIP enthält dieses Protokoll** (`node make_zip.js`). Änderungsprotokoll fortschreiben, Zahlen werden automatisch aktualisiert.
8. **Nutzer-Anweisungen zur Arbeitsweise gelten dauerhaft**, wenn sie so formuliert sind (zum Beispiel diese Protokoll-Pflicht).
9. Dem Nutzer nach einer neuen Version sagen, welche Dateien er auf GitHub ersetzen muss (`index.html` und `sw.js`).

---

## 10. Bekannte Fallen (aus der Entwicklung)

- Eingefügter Text im Chat wird bei etwa 20.000 Zeichen abgeschnitten. Perplexity-Ausgaben deshalb in Chargen von höchstens 8 Rezepten holen oder als Datei anhängen.

- CSS-Regel `.sheet{display:flex}` übersteuert `hidden` → `[hidden]{display:none!important}` ist Pflicht.
- Reihenfolge der Verkettung nicht ändern, `initRuntime()` zuletzt.
- Im Test-DOM-Ersatz gibt es kein echtes Layout. Layoutfehler zeigt nur der Browsertest.
- Falsche Zutaten-IDs sind die häufigste Ursache für abgelehnte Rezepte (zum Beispiel `linsen` statt `linsen_rot`, `karotte` statt `moehre`, `kichererbsen` statt `kichererbsen_dose`). Immer in `ZMAP` nachsehen.
- `localStorage` und Dienste können im Artifact fehlen oder Fehler werfen: immer in `try/catch`.
- `state.removed` muss in jeder Liste über `lib()` laufen, nicht direkt über `SEED`.
- Der Autoplan füllt nur leere Plätze (außer „alles neu“).

---

## 11. Änderungsprotokoll

- **Sous-vide mit belegten Werten (`data_sousvide.js`):** Tabelle für Hähnchen und Pute (Anova nach USDA, 7-log Salmonellen, Haltezeit ab Erreichen der Kerntemperatur, nur Zeilen ab 63 °C) samt Regeln (Aufheizzuschlag, Listerien dreifache Zeit, nichts unter 60 °C, Risikogruppen) im Sicherheitsbereich unter „Mehr“. Für Schwein, Rind, Fisch und Hack gibt es bewusst keine Tabelle: dort gilt die BfR-Regel, im Rezept r18 (Schweinefilet) deshalb jetzt Bad 70 °C statt 63 °C. Test in `test_logic3.js`: jedes Sous-vide-Rezept hat Badtemperatur ≥ Mindestwert der Gruppe. Offen: Tabellen für Rind/Schwein/Fisch nur mit vollständig geprüfter Originalquelle (z. B. Baldwin, Buch „Sous Vide for the Home Cook“) ergänzen.
- **Rezeptprüfung vor dem Import (`check_recipes.js`):** `node check_recipes.js datei.json` liest Perplexity-Ausgaben im Importformat (auch abgeschnittene bis zum letzten vollständigen Rezept) und meldet je Rezept Nährwerte pro Portion, Pflicht- und Dublettenprüfung, unbekannte IDs, fehlende 70-°C-Angabe bei Fleisch, Gefrierblock gegen Portionsgröße und nicht abgebildete Zutaten. Erste Charge: 9 Rezepte gelesen (Eingabe nach 20.000 Zeichen abgeschnitten, Rezept 10 Pho Ga fehlt), 1 Dublette (Chili con Carne), bei 6 Rezepten kleine Korrekturen (Kochzeiten, Dosenbohnen erst spät, Gefriernotiz, Marinierzeit), Ergebnis `import_geprueft_01.json` mit 8 Rezepten, 0 Fehler, 0 Warnungen über den echten Importpfad (`parseRecipeJson`, `validateRecipe`).
- **Prüfung der Werte (Stand dieser Fassung):**
  - Interne Prüfung der Zutaten (Atwater-Konsistenz kcal gegen Makros, Plausibilität): keine Auffälligkeiten.
  - Prüfung der Gefrierblock-Angaben der Bibliotheksrezepte gegen geschätzte Portionsvolumen (Garverlust 10 bis 30 %, Dichte 0,5 bis 1,0 g/ml, nur der eingefrorene Anteil). Korrigiert wurden nur Blockgröße und Hinweistext, nie Nährwerte: Bolognese 250 auf 500 ml, Congee 250 auf 500 ml (dicker einkochen), Schweinebraten 500 auf 250 ml, Hühnersuppe und Minestrone mit Einkoch-Hinweis, Hummus mit Hinweis 1 bis 2 Blöcke. Alle Volumen sind Schätzungen.
  - Sicherheitswerte gegen BfR-Küchenhygiene geprüft: 70 °C an allen Stellen für 2 Minuten, Kühlschrank 2 bis 7 °C, Abkühlen auf unter 7 °C, Warmhalten ab 60 °C; Tiefkühlen bei −18 °C oder kälter, Auftauen im Kühlschrank, Hack in 3 bis 4 Monaten verbrauchen (Bausteine mit Hack: 3 Monate). Die Regel „höchstens 2 Stunden bei Raumtemperatur“ ist ein verbreiteter Richtwert, auf der BfR-Seite nicht wörtlich genannt.
  - **BLS-Abgleich erledigt:** Datei `BLS_4_0_Daten_2025_DE.xlsx` (Max Rubner-Institut, Download über blsdb.de) wurde Zutat für Zutat zugeordnet. 124 Werte 1:1 übernommen, 20 als Näherung (ähnlicher BLS-Eintrag, z. B. Basmati = Reis poliert), 1 Mittel (Beerenmischung), 3 abgeleitet (Brühen aus Pulver, 2 g je 100 ml), 18 eigene Werte ohne BLS-Gegenstück (z. B. Kochsahne 15 %, Paneer, Harissa, Chorizo, mageres Rinderhack). Rezept-Kalorien pro Portion verschoben sich im Mittel um 5 %, bei 12 Rezepten um 10 bis 19 % (größte Wirkung: Edamame, Hüttenkäse, Forelle, Kartoffelsuppe).
  - **Wichtig für Folgearbeit:** Die BLS rechnet Energie inklusive Ballaststoffen (2 kcal/g), organischen Säuren (3 kcal/g) und Alkohol (7 kcal/g), und „Kohlenhydrate“ sind die verfügbaren (ohne Ballaststoffe). Deshalb ist bei BLS-Zutaten kcal ≠ 4·Eiweiß + 4·KH + 9·Fett. Das Feld `bls.e` in `data_bls.js` trägt den Differenzanteil; `ingredientEnergyIssues()` rechnet ihn ein. Neue Zutaten aus der BLS immer mit `e` anlegen.
  - Neue Zutaten: Wert aus der BLS holen, in `logic.js` oder `data_zutaten.js` eintragen, Eintrag in `data_bls.js` (Code, Modus, Hinweis, `e`) und Code in `bls_auszug.json` ergänzen, sonst schlägt `test_bls.js` an.
  - Nach wie vor offen: Sous-vide-Tabellen für Rind/Schwein/Fisch, Gar- und Ausbeutefaktoren der Bausteine (Schätzungen bis zum ersten Kochen und Wiegen).
- **Stand Meal-Prep + Bibliotheksverwaltung (diese Fassung):**
  - Neuer Tab **Prep** mit Start (Soll/Ist, Startprogramm mit 4 Sessions, Schubladenplan, Sicherheit, Etikett), Katalog der Bausteine, Mahlzeit bauen (mit Ins-Tagebuch), Session planen (Chargen, Runden, Einkaufsliste, Ablauf, Einbuchen in den Gefrierer mit Abbuchung vom Vorrat).
  - Bausteine als Gefrierer-Bestand (`bid`, `mod`), im Tagebuch eintragbar.
  - **Rezept-Bibliothek verwalten:** mitgelieferte Rezepte einzeln, gruppenweise oder komplett entfernen, wiederherstellen, Filter „Herkunft des Rezepts“.
  - `perplexity-prompt-rezepte.txt` und `mk_prompt.js` für den Rezeptimport.
  - Dieses Protokoll und `make_zip.js` (jede ZIP enthält das Protokoll).
- **Stand App v2:** vollständige App mit 76 Rezepten, Wochenplan, Einkauf, Vorrat (Zutaten-Chargen mit FIFO, Gewürze, Gefrierer), Tagebuch, Fasten, Kalorienrechner, Sicherung/Import, Prüfbericht, PWA-Build und Hosting auf GitHub Pages.
- **Stand v1:** Rezeptbibliothek, Zutatenstamm, Nährwert- und Allergenrechnung, Prüffunktionen.

*(Beim Weiterarbeiten oben einen neuen Eintrag ergänzen und in `PROTOKOLL.template.md` speichern.)*

---

## 12. Funktionsverzeichnis (automatisch erzeugt)

**logic.js**: `I`, `esc`, `norm`, `round1`, `exclusionHits`, `recipeTexts`, `recipeExclusionHits`, `calcRecipe`, `derive`, `conflicts`, `mealType`, `slotFits`, `energyCheck`, `ingredientEnergyIssues`, `validateRecipe`, `runQA`, `distribution`, `dayTotals`, `shoppingList`, `fmtAmount`, `addMonths`, `expiry`, `invStats`, `zutatenListeFuerPrompt`, `normalizeAIRecipe`

**data_zutaten.js**: (nur Daten)

**data_bls.js**: `blsStats`, `blsLine`

**data_sousvide.js**: `svMinTemp`, `svTabelleHtml`

**logic2.js**: `isoLocal`, `addDaysIso`, `diffDays`, `fmtDate`, `shelfDays`, `lagerort`, `newLot`, `lotState`, `stockTotals`, `deductFifo`, `recipeNeed`, `coverage`, `spiceCheck`, `whatCanICook`, `expiringSoon`, `lowStock`, `weekNeed`, `shopPlan`, `fmtGrams`, `emptyDay`, `dayOf`, `round1x`, `nutriOfIng`, `entryRecipe`, `entryIng`, `entryQuick`, `diaryTotals`, `macroGoals`, `weekStats`, `streak`, `weightSeries`, `recentFoods`, `calcGoals`, `fastStatus`, `fmtHM`, `mulberry32`, `autoPlan`, `weekSummary`, `normalizeUserRecipe`, `parseRecipeJson`, `exportState`, `importState`, `diaryCsv`

**data_bausteine.js**: `B`

**logic3.js**: `cubePP`, `bausteinAllergene`, `validateBaustein`, `runQABaukasten`, `stylePasst`, `mealBuild`, `mealName`, `suggestMeals`, `planItemInfo`, `sessionYield`, `sessionNeeds`, `prepShopping`, `sessionTimeline`, `bookSession`, `addMonthsStr`, `sollBestand`, `istBestand`, `prepRoundsFor`, `etikett`

**app_a.js**: `emptyWeek`, `defaultPrep`, `defaultState`, `mergeState`, `loadLocal`, `save`, `flush`, `initRuntime`, `toast`, `copyText`, `saveFile`, `statusOf`, `mealForSlot`, `goalsNow`, `dayRec`, `dayW`, `ringSvg`, `macroBar`, `weekBars`, `weightChart`, `entryRow`, `fastCard`, `viewHeute`, `addListHtml`, `addSheet`

**app_b.js**: `cuisines`, `filtered`, `activeFilterCount`, `recipeCard`, `renderList`, `distBlock`, `viewRezepte`, `nutriBlock`, `checksBlock`, `ingredientRows`, `spiceChips`, `recipeText`, `recipeSheet`, `newEd`, `edFrom`, `edToRecipe`, `edResultsHtml`, `editorSheet`, `kiImportPrompt`

**app_c.js**: `seg`, `slotOptions`, `goalBar`, `viewPlanTab`, `viewPlan`, `shopData`, `shopKey`, `shopText`, `viewShop`, `viewVorrat`, `expBadge`, `viewStock`, `usageCount`, `viewSpices`, `viewFreezer`, `viewKochen`, `stockResultsHtml`, `stockAddSheet`, `ingSheet`, `openAttr`, `calcPanel`, `infoPanel`, `backupPanel`, `profilePanel`, `statusPanel`, `libPanel`, `viewMehr`

**app_d.js**: `render`, `renderSheet`, `openSheet`, `aiErr`, `exclText`, `profileText`, `runPlanKI`, `runVorratKI`, `runImportKI`, `confirmed`, `dropRecipeRefs`, `logEntry`, `invPortion`, `eatInv`, `markCooked`, `buyGrams`, `addMonthsIso`, `fillStep`, `recomputeEntry`, `newInvForm`, `runAuto`

**app_e.js**: `prepSt`, `prepAllergBadges`, `cubeLine`, `isUsable`, `viewPrep`, `prepStart`, `prepKatalog`, `bausteinSheet`, `prepBauen`, `prepFreezable`, `prepSession`, `prepToast`
