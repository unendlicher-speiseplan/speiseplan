# PROTOKOLL: Unendlicher Speiseplan (Übergabe an jede KI und jeden Chat)

Stand dieser ZIP: 2026-10-28 18:39 UTC · Paket: App (Ausspielung). Der Quellcode steht in speiseplan-projekt.zip.

**Zweck dieser Datei:** Ein neuer Chat oder eine andere KI soll nur diese ZIP lesen und ohne Rückfragen weiterarbeiten können. Alles Wichtige steht hier: Vorgaben des Auftraggebers, Architektur, Datenformate, Teststand, bekannte Fallen, offene Aufgaben und ehrliche Grenzen. Die Zahlen in Abschnitt 3 und 4 sind beim Packen automatisch aus dem Quellcode erzeugt und stimmen mit dieser ZIP überein. Alles andere ist von Hand geschrieben. Bei Widerspruch gilt der Quellcode.

---

## 0. Kurzanleitung zum Weiterarbeiten (in dieser Reihenfolge)

1. Diese Datei vollständig lesen, besonders Abschnitt 1 (Vorgaben) und Abschnitt 9 (Arbeitsregeln).
2. Voraussetzung ist nur **Node.js** (ab Version 18, getestet mit 22). Keine npm-Pakete nötig. Für den optionalen Browsertest zusätzlich Python 3 mit Playwright und Chromium.
3. Im Projektordner ausführen: `node build.js`. Das erzeugt `speiseplan.html` (Variante für Claude-Artifacts) und den Ordner `dist/` (eigenständige App als PWA: `index.html`, `sw.js`, `manifest.json`, Icons).
4. Tests ausführen (alle müssen grün sein, Erwartungswerte in Abschnitt 5): `node test_logic2.js`, `node test_logic3.js`, `node smoke2.js`, `node smoke3.js`, `node test_saison.js`, `node smoke5.js`.
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
| Mitgelieferte Rezepte | **142** von 240 geplanten |
| Verteilung nach Gruppe | Geflügel: 42, Rind: 36, Schwein: 28, Fisch: 23, Vegetarisch: 13 |
| Verteilung nach Mahlzeit | Hauptgericht: 117, Suppe/Eintopf: 22, Salat/Bowl: 3, Frühstück: 0, Snack: 0 |
| Einfrierbar: ja / eingeschränkt | 75 / 31 |
| Zutaten im Stamm (davon mit Nährwerten) | 188 (187) |
| Gewürze | 47 |
| Bausteine (Meal-Prep) | **70** (Protein: 12, Beilage: 8, Sauce: 22, Gemüse: 5, Booster: 23) |
| Einkaufskategorien | Fleisch & Geflügel, Fisch & Meeresfrüchte, Gemüse, Obst, Milchprodukte, Eier, Tiefkühlprodukte, Trockenprodukte, Konserven, Gewürze & Öle, Sonstiges |
| Geräte-Werte | Airfryer, Kontaktgrill, Multikocher, Sous-vide, Herd, Backofen, Monsieur Cuisine Connect, Dutch Oven |
| Feste Ausschlüsse | Ananas, Süßkartoffel, Batate |
| Größe `dist/index.html` | 839 KB |

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
| `data_rezepte4.js` | Perplexity-Charge 1 (9 Rezepte), Charge 2 (9 Sous-vide), Charge 6 (17 recherchierte Hähnchen-Gerichte), Charge 7 und 8 (11 Rind-Gerichte) und Liste ausgemusterter eigener Rezepte (`RETIRED`) | 
| `data_rezepte5.js` | Bibliothek ab r77: mustert beim Start nur die früheren eigenen Rezepte (IDs unter r77) aus; Anleitung zum erneuten Ausschließen aller Nicht-Perplexity-Rezepte im Dateikopf; mit `const FULL_LIBRARY = true;` davor bleibt die volle Bibliothek für Tests | 
| `logic2.js` | Datum, Zutaten-Chargen (FIFO), Einkauf, Tagebuch, Ziele, Fasten, Autoplan, Import/Export | 
| `data_bausteine.js` | Meal-Prep-Bausteine (`MODULE`, `STIL_PASST`, `BAUSTEINE`, `BMAP`) | 
| `logic3.js` | Meal-Prep-Logik: Blockzahl, Prüfung, Mahlzeit bauen, Vorschläge, Session, Einbuchen, Soll/Ist, Schubladen, Etikett | 
| `data_saison.js` | Saisonkalender als eigene Daten (`SAISON`: 45 Zutaten mit 12-Monats-Muster, `SAISON_NEUTRAL`: bewusst nicht bewertet); unabhängig von jeder Quelldatei | 
| `logic4.js` | Saison-Logik: Rezept-Score je Monat, Monatsübersicht, Einkaufsstatus, Wochenanteil, eigene Korrekturen, Prüfbericht (`runQASaison`) | 
| `app_a.js` | Zustand (`defaultState`, `mergeState`), Speicher, Hilfen, Ansicht Heute, Hinzufügen-Blatt | 
| `app_b.js` | Rezepte (Liste, Filter, Detail), Rezept-Editor, KI-Import-Prompt | 
| `app_c.js` | Plan, Einkauf, Vorrat (Zutaten, Gewürze, Gerichte, Kochen), Mehr (inkl. Bibliotheksverwaltung, Status) | 
| `app_d.js` | Tabs, `render`, Blätter, KI-Funktionen, `ACT`/`CHG`/`INP`-Tabellen, Ereignis-Verdrahtung | 
| `app_f.js` | Saison-Oberfläche: Übersicht (Plan → Saison), Heatmap, Abzeichen, Rezeptfilter, Heute-Karte, Einkaufs-Hinweise, Einstellungen | 
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
| `smoke4.js` | Oberflächentest der ausgelieferten Nur-Perplexity-Bibliothek (18 Rezepte, leere Bereiche, alter Stand mit ausgemusterten Verweisen) | 
| `test_saison.js` | Logiktests Saison (Daten, Zeitraumtext, Score, Korrekturen, Monatsübersicht, Auto-Plan mit festen Seeds) | 
| `smoke5.js` | Oberflächentest Saison (alle 12 Monate, Bearbeiten, Filter, Heute, Plan, Einkauf, Einstellungen, Export/Import) | 
| `pw_prep.py` | Browsertest Prep (Playwright, 400 px, hell/dunkel) | 
| `pw_lib.py` | Browsertest Bibliotheksverwaltung | 
| `make_zip.js` | Dieses Packskript | 
| `mk_prompt.js` | Erzeugt `perplexity-prompt-rezepte.txt` aus den aktuellen Zutatenlisten | 
| `check_recipes.js` | Prüfbericht für Perplexity-Rezepte vor dem Import: `node check_recipes.js datei.json` | 
| `make_icons.py` | Erzeugt alle App-Icons aus `icons/icon-quelle.png` (nach Änderung der Quelle ausführen, danach `node build.js`) | 
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
| `node test_logic2.js` | 79 Prüfungen | bestanden |
| `node test_logic3.js` | 263 Prüfungen | bestanden |
| `node test_bls.js` | 9 Prüfungen | bestanden |
| `node smoke2.js` | 153 Prüfungen | bestanden |
| `node smoke3.js` | 91 Prüfungen | bestanden |
| `node smoke4.js` | 55 Prüfungen | bestanden |
| `node test_saison.js` | 57 Prüfungen | bestanden |
| `node smoke5.js` | 85 Prüfungen | bestanden |

Weitere Hinweise zu den Tests:
- `smoke2.js` und `smoke3.js` laden `dist/index.html` in eine Node-VM mit einem kleinen DOM-Ersatz und klicken über die Aktionstabellen. Sie prüfen unter anderem, dass nirgends `undefined`, `NaN`, `[object` oder Ausschluss-Zutaten im HTML stehen.
- Beim Schreiben von Tests als Template-String Rückwärtsstriche in Regex doppelt schreiben (`\\[object`), sonst entstehen Syntaxfehler im Test statt in der App.
- `pw_prep.py` und `pw_lib.py` sind Browsertests (Playwright, Chromium, Fenster 400 px, hell und dunkel) mit Screenshots. Sie prüfen Konsolenfehler und Querscrollen. Chromium-Pfad im Beispiel: `/opt/pw-browsers/chromium`, Start mit `--no-sandbox`. Passe den Pfad an deine Umgebung an.
- Beim Test im echten Browser wurde ein Fehler gefunden, den die Node-Tests nicht sehen: das `hidden`-Attribut wurde von CSS übersteuert.

---

## 6. Ehrliche Grenzen des aktuellen Stands

- **Rezepte:** 142 von 240 geplanten sind mitgeliefert. Status „Entwurf, automatisch validiert“, **von der KI nicht gekocht**. Der Nutzer kann Rezepte entfernen und durch eigene oder importierte ersetzen.
- **Bausteine:** 70 Bausteine als eigene Grundzubereitungen, Status „Entwurf, automatisch validiert“. Ausbeute, Blockanzahl und Küchenzeiten sind Schätzungen. Nichts davon wurde gekocht oder eingefroren.
- **Nährwerte:** 148 von 166 Zutaten nach BLS 4.0 (Quelle und Code je Zutat in `data_bls.js`, Auszug in `bls_auszug.json`, geprüft von `test_bls.js`), 18 Zutaten ohne BLS-Eintrag mit eigenem Packungs- oder Richtwert (begründet). Das gilt je 100 g der Zutat; die Mahlzeitenwerte sind daraus gerechnet. Marken, Sorte, Reifegrad und Garverluste (Rezeptmengen sind Rohgewichte) lassen Abweichungen im Alltag zu. „Absolut“ ist nicht erreichbar, nachvollziehbar und belegt schon.
- **Nicht testbar durch die KI:** Funktionen, die nur im Claude-Artifact laufen (KI-Vorschläge, KI-Import, Speichern im Claude-Konto, Downloads über `downloads`), sowie die Installation der App als PWA auf einem echten Handy. Der Nutzer bestätigte, dass die gehostete App läuft.
- **Autoplan:** rechnet je Platz mit einer Portion. Gut für etwa 1500 bis 1900 kcal. Proteinziele über etwa 130 g werden nur an einem Teil der Tage erreicht.
- **Nicht enthalten:** Monsieur-Cuisine-Varianten, Mealie-Austausch (nur mit echtem Export testbar), PDF-Rezeptkarten und Rezeptbuch, Barcode-Scan und Kamera, Mehrbenutzer-Server.
- **Daten liegen nur im Gerät.** Ohne Sicherung gehen sie bei gelöschten Browserdaten verloren.

---

## 7. Offene Aufgaben (Reihenfolge nach Rückmeldung des Auftraggebers)

1. **Perplexity-Rezeptimport:** Die App enthält die 18 Perplexity-Rezepte (r77 bis r94), die 65 nach Online-Recherche geschriebenen Rezepte (r95 bis r159) und 18 weitere Rezepte ohne abrufbare Quelle (r160 bis r177); die früheren eigenen Rezepte r01 bis r76 sind per `data_rezepte5.js` ausgemustert (Entscheidung vom 28.10.2026, siehe Eintrag „vierte“). Neue Rezepte bekommen IDs ab r160 und werden am Ende von `data_rezepte4.js` angehängt. Ohne Rezepte für Frühstück, Snack, Salat/Bowl, Fisch und Vegetarisch erreicht der Wochenplan den Kalorienkorridor nicht; diese Bereiche müssen über Perplexity nachgeliefert werden. Dafür gibt es `perplexity-prompt-rezepte.txt` (erzeugt von `mk_prompt.js` mit den aktuellen Zutaten- und Gewürzlisten) und die Bibliotheksverwaltung. Geliefertes JSON bitte vor dem Import gegen die Regeln aus Abschnitt 1 prüfen.
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

- **Rezept-IDs sind positionsabhängig:** `R()` vergibt IDs nach Aufrufreihenfolge. Niemals einen `R({...})`-Aufruf aus den Quelldateien löschen, sonst verschieben sich alle folgenden IDs und gespeicherte Favoriten, Pläne und Gefrierer-Einträge zeigen auf falsche Rezepte. Rezepte stattdessen in `RETIRED` (`data_rezepte4.js` oder spätere Charge) eintragen. Neue Chargen in neue Dateien, `build.js`, `make_zip.js`, `mk_prompt.js`, `check_recipes.js` und die Tests um die Datei erweitern.

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

- **Perplexity-Charge 1 integriert, neues App-Icon:** Neue Datei `data_rezepte4.js` mit 9 geprüften Perplexity-Rezepten (IDs r77 bis r85; Rohdaten und geprüfte Fassung in `chargen/`). Fünf gleichnamige eigene Rezepte wurden ausgemustert (`RETIRED`: r02 Rindergulasch, r03 Chili, r36 Butter Chicken, r43 Thai-Curry, r55 Tajine). Die Rezeptliste hat jetzt 80 Einträge. `mergeState()` bereinigt Favoriten, Wochenplan, Prep-Plan und Entfernt-Liste; Gefrierer-Einträge bleiben als Name stehen. Weitere eigene Rezepte werden jeweils ausgemustert, wenn ein Perplexity-Rezept dasselbe Gericht ersetzt (Wunsch des Auftraggebers: eigene Rezepte löschen, sobald Perplexity-Rezepte integriert sind). Neues Icon aus `icons/icon-quelle.png` (Bild des Auftraggebers), erzeugt mit `make_icons.py`: `icon-192`, `icon-512`, `icon-maskable-512`, `apple-touch-180`, `favicon-48`; Manifest und Offline-Cache angepasst, `icon.svg` entfällt. Beim Hochladen auf GitHub alle Dateien aus `speiseplan-app.zip` hochladen (neue Icons!).
- **Charge 8: 4 weitere Rind-Gerichte (28.10.2026, IDs r119 bis r122):** Von vier Links war nur Food Network lesbar (Chefkoch und BBC Good Food gesperrt). Eigene Fassungen nach Recherche bei anderen Quellen: Rinderschmorbraten in Rotweinsoße, Gulasch nach Omas Art (deutsche Art, nicht der ungarische Gulyás r77), Rind-Gemüse-Schmortopf nach britischer Art, Rinderfilet in Salzkruste (Rohdaten `chargen/recherche_05_rind.json`; nicht von Perplexity). **Salzkruste:** das einzige Rezept mit rosa Kerntemperatur (52 bis 54 °C herausnehmen, Ruhen auf 58 bis 60 °C); im Text steht der Hinweis, dass Schwangere, Kleinkinder, Ältere und Immungeschwächte auf mindestens 70 °C durchgaren sollen. Die Zeiten sind aus einem 3-kg-Rezept auf 800 g hochgerechnet und nur Richtwerte. Einfrieren: nein. Die Kruste und der Wein sind nicht in den Nährwerten. Bibliothek jetzt 46 Rezepte. `level` erlaubt nur `leicht | mittel | anspruchsvoll`.
- **Charge 9: 8 weitere Rind-Gerichte (28.10.2026, IDs r123 bis r130):** Acht Allrecipes-Links (Brisket, Corned-Beef-Brisket, Short Ribs, Beef Wellington, Chili, Enchiladas, Rinderfilet-Braten, Stir-Fry). Allrecipes ist für meine Werkzeuge gesperrt, daher eigene Fassungen nach Recherche bei anderen Quellen (Küchengötter, patient.info, Well Plated, Ohio State University, Der Düsseldorfer, LECKER), nicht von Perplexity; Rohdaten `chargen/recherche_06_rind.json`. **Ersatzzutaten in den Nährwerten:** Rinderbrust, Pökelrinderbrust und Short Ribs sind als Rindergulasch gerechnet (die echten Stücke sind fetter, bei Pökelware ist der Salzgehalt viel höher); Rinderfilet als Hüftsteak. **Wellington:** Blitzteig aus Mehl, Butter und Quark statt fertigem Blätterteig, damit die Nährwerte berechnet werden können. **Rosa gegart (wie die Salzkruste):** Filet Wellington und Rinderfilet-Braten, Herausnehmen bei 52 bis 54 °C, Ruhen auf etwa 56 bis 58 °C; im Text steht der Hinweis auf 70 °C für Risikogruppen. Garzeiten stammen aus den Quellen und sind nicht von mir erprobt. Einfrieren: ja bei Brisket, Short Ribs und Chili; eingeschränkt bei Pökelrinderbrust, Enchiladas und Pfanne; nein bei Wellington und Filet-Braten. Bibliothek jetzt 54 Rezepte.
- **Charge 10: 3 weitere Rind-Gerichte (28.10.2026, IDs r131 bis r133):** Drei Links (Allrecipes Beef Lo Mein, Allrecipes South Asian-Style Ground Beef Keema, Chefkoch Rinderrouladen klassisch). Allrecipes und Chefkoch sind gesperrt, daher eigene Fassungen nach Recherche (Southern Plate, Tea for Turmeric, LECKER), nicht von Perplexity; Rohdaten `chargen/recherche_07_rind.json`. **Keema:** Das Aloo Keema (r118) war schon da, die neue Fassung ist die Erbsenvariante Keema Matar mit Joghurt und Tomate, also kein Duplikat. **Nicht in den Nährwerten:** Hoisinsauce und Sesamöl (Lo Mein, durch Sesam und Sojasauce ersetzt), Gewürzgurken (Rouladen); Eiernudeln als Hartweizennudeln gerechnet. Einfrieren: ja bei Keema und Rouladen, eingeschränkt bei Lo Mein. Bibliothek jetzt 57 Rezepte.
- **Charge 11: 3 weitere Rind-Gerichte (28.10.2026, IDs r134 bis r136):** Drei Allrecipes-Links (Empanadas, Beef and Broccoli, Beef and Guinness Stew). Allrecipes ist gesperrt, daher eigene Fassungen nach Recherche (LECKER, America's Test Kitchen, Primavera Kitchen), nicht von Perplexity; Rohdaten `chargen/recherche_08_rind.json`. **Nicht in den Nährwerten:** Stout (330 ml, etwa 25 bis 30 kcal je Portion, enthält Gluten und Alkohol) und die Austernsauce (Weichtiere, im Stamm nicht vorhanden; durch Sojasauce ersetzt). Worcestershiresauce (enthält Fisch) ist durch Sojasauce und Essig ersetzt. Die Quelle gart den Eintopf nur 30 Minuten, für Schulterfleisch sind 2,5 Stunden angesetzt. Rindfleisch mit Brokkoli ähnelt der Rindfleisch-Gemüse-Pfanne (r130), hat aber eine andere Sauce. **Korrektur:** Im Tipp der geschmorten Rinderbrust (r123) stand fälschlich Zwiebelsuppenpulver als Original-Zutat; die Quelle nutzt Ketchup und BBQ-Sauce, der Text ist angepasst. Bibliothek jetzt 60 Rezepte.
- **Charge 12: 6 Schweine-Gerichte (28.10.2026, IDs r137 bis r142):** Sechs Allrecipes-Links (Pork Carnitas, Baked Pork Chops I, Asian Pork Tenderloin, The Best Easy Pork Tenderloin, Chinese Barbeque Pork Char Siu, Roast Adobo Pork Loin). Allrecipes ist gesperrt, daher eigene Fassungen nach Recherche (Made In, strongrfastr, Der Standard, LECKER, Chili Pepper Madness, The Woks of Life), nicht von Perplexity; Rohdaten `chargen/recherche_09_schwein.json`. Die Gruppe Schwein hat damit 8 Rezepte. **Abweichungen:** Die Adobo-Fassung ist die geschmorte Variante nach Woks of Life, kein Lendenbraten aus dem Ofen (die Allrecipes-Vorlage war nicht lesbar). Das Asia-Filet ist ein Pfannengericht nach Der Standard, kein Ofenbraten. Die Koteletts sind als Schweineschulter ohne Knochen gerechnet. **Nicht in den Nährwerten:** Sherry, Hoisinsauce und Reiswein (weggelassen oder durch Sojasauce und Honig ersetzt). **Kerntemperatur:** In den Quellen gilt für Schwein 63 °C (145 °F), in der App gilt wie bei allen Fleischrezepten mindestens 70 °C. Garzeiten stammen aus den Quellen und sind nicht von mir erprobt. Einfrieren: ja bei Carnitas, Char Siu und Adobo; eingeschränkt bei den übrigen. Bibliothek jetzt 66 Rezepte.
- **Charge 13: 2 weitere Schweine-Gerichte (28.10.2026, IDs r143 und r144):** Zwei BBC-Good-Food-Links (Thai Pork Peanut Curry, Mustardy Pork and Apples). BBC Good Food ist für meine Werkzeuge gesperrt (auch die Spiegelseite bbcgoodfoodme.com habe ich nicht benutzt), daher eigene Fassungen nach Recherche (Ocado, Christine's Recipes, Aldi Süd, LECKER), nicht von Perplexity; Rohdaten `chargen/recherche_10_schwein.json`. **Thai-Curry:** Currypaste enthält oft Garnelenpaste (Krebstiere), Erdnussmus ist Hauptallergen; Baby-Mais ist als Dosenmais, Jasminreis als Basmatireis gerechnet; weniger Kokosmilch als die Quelle, dafür Brühe. **Apfel-Senf-Medaillons:** Apfelsaft der Quelle durch Gemüsebrühe ersetzt, Kochsahne statt Schlagsahne, Feldsalat als Blattsalat gerechnet; nicht einfrierbar. Die Gruppe Schwein hat damit 10 Rezepte. Bibliothek jetzt 68 Rezepte.
- **Charge 14: 10 Fisch-Gerichte (28.10.2026, IDs r145 bis r154):** Zehn Allrecipes-Links (Fish Tacos, Super Simple Salmon, Cedar Planked Salmon, Salmon with Dill, Baked Salmon II, Easy Baked Tilapia, Indian Fish Curry, Grilled Cod, Parmesan Crusted Tilapia, Best Tuna Casserole). Allrecipes ist gesperrt, daher eigene Fassungen nach Recherche (Küchengötter, LECKER, Aldi Süd), nicht von Perplexity; Rohdaten `chargen/recherche_11_fisch.json`. Es sind die ersten Rezepte der Gruppe Fisch. **Zuordnung statt Kopie:** Die zehn Gerichte sind so gewählt, dass jedes eine eigene Zubereitung hat: Fisch-Tacos (Pfanne statt Frittieren), Zitronen-Knoblauch-Lachs, Lachs mit Soja-Honig-Glasur vom Brett, Lachs mit Zitronen-Dill-Sauce, Lachs mit Kräuter-Senf-Kruste, Seelachs auf Tomaten (statt Tilapia), Fischcurry, Kabeljau vom Kontaktgrill (statt Grill), Seelachs in Parmesankruste (statt Tilapia), Thunfisch-Nudelauflauf. Tilapia ist nicht im Stamm und durch Seelachs bzw. Kabeljau ersetzt. **Garpunkt Fisch:** Kern mindestens 63 °C (undurchsichtig, zerfällt in Lamellen); in jedem Rezept steht der Hinweis, dass Schwangere, Kleinkinder, Ältere und Immungeschwächte bis 70 °C durchgaren sollen. Aufwärmen immer auf mindestens 70 °C. **Nicht in den Nährwerten:** Wein (durch Brühe ersetzt), Curryblätter, Chipotle (durch Rauchpaprika ersetzt), Zitronenschale. Mais-Tortillas der Quelle sind Weizentortillas. Einfrieren: nein bei allen gegarten Fischen; nur der Thunfisch-Nudelauflauf ist einfrierbar, das Fischcurry eingeschränkt (Sauce einfrieren, Fisch frisch einlegen). Garzeiten stammen aus den Quellen und sind nicht von mir erprobt. Bibliothek jetzt 78 Rezepte.
- **Charge 15: 5 Garnelen-Gerichte (28.10.2026, IDs r155 bis r159):** Fünf Allrecipes-Links (Shrimp Scampi, Indian Shrimp Curry, Good New Orleans Creole Gumbo, Peppered Shrimp Alfredo, Chipotle Shrimp Tacos). Allrecipes ist gesperrt, daher eigene Fassungen nach Recherche (LECKER, Küchengötter, Lidl Kochen, Fed by Sab, ASC), nicht von Perplexity; Rohdaten `chargen/recherche_12_garnelen.json`. Sie stehen in der Gruppe Fisch (die App hat keine eigene Gruppe für Meeresfrüchte), die Gruppe hat damit 15 Rezepte. **Garpunkt Garnelen:** durchgehend undurchsichtig, rosa und fest, Kern mindestens 63 °C; in jedem Rezept steht der Hinweis auf 70 °C für Risikogruppen und auf das Allergen Krebstiere. **Nicht in den Nährwerten:** Weißwein (Scampi, weggelassen), Okra und Andouille (Gumbo; Wurst als Chorizo gerechnet), Tamarinde und Curryblätter (Curry), Chipotle (durch Rauchpaprika und Chilipulver ersetzt). Mais-Tortillas der Quelle sind Weizentortillas. Einfrieren: nein bei Scampi, Alfredo und Tacos, eingeschränkt bei Curry und Gumbo (Sauce einfrieren, Garnelen frisch zugeben). Die Gumbo-Quelle von Allrecipes war nicht lesbar, die Rezeptstruktur folgt der Lidl-Fassung. Bibliothek jetzt 83 Rezepte, nächste freie ID r160.
- **Entscheidung 28.10.2026 (zweite): recherchierte Rezepte ausgemustert.** Auf die Frage, woher die Nicht-Perplexity-Rezepte stammen: Die 65 Rezepte r95 bis r159 (Hähnchen, Rind, Schwein, Fisch, Garnelen) hat Claude selbst geschrieben, aufgebaut nach echten Online-Rezepten (Quellen im Feld `src`), aber mit eigenem Text, eigenen Mengen, Ersatzzutaten und teils hochgerechneten Garzeiten. Das ist KI-geschriebener Inhalt. Nach der Vorgabe des Auftraggebers (KI-generierte Rezepte sofort entfernen) werden sie beim Start ausgemustert; ausgeliefert werden nur Rezepte, deren `src` mit "Perplexity" beginnt (r77 bis r94, 18 Rezepte). Die Rezepte bleiben im Quellcode (`data_rezepte4.js`, Rohdaten in `chargen/`), damit die Entscheidung umkehrbar ist: in `data_rezepte5.js` den Zweig `else if` entfernen. Neue Rezepte bekommen IDs ab r160. `smoke4.js` prüft jetzt 18 Rezepte, 141 ausgemusterte und dass jede mitgelieferte Quelle Perplexity ist. Die 35 Bausteine sind weiter enthalten (keine Rezepte, aber auch nicht von Perplexity).
- **Entscheidung 28.10.2026 (dritte): Allrecipes für alle Perplexity-Suchen ausgeschlossen.** Auf Wunsch des Auftraggebers enthält der Prompt (`mk_prompt.js`, `perplexity-prompt-rezepte.txt`) jetzt die Regel, allrecipes.com und Ableger nicht als Quelle zu nutzen und nicht in `src` zu nennen. Hintergrund: Allrecipes ist für Claudes Werkzeuge gesperrt. Die ausgefüllten Prompts für Meeresfrüchte und Vegetarisch wurden entsprechend angepasst.
- **Entscheidung 28.10.2026 (vierte): alle Rezepte wieder in der App.** Auf ausdrücklichen Wunsch („Ich brauche keine Prompts. Ich will meine fertige App mit Rezepten. Bitte alle Rezepte einbauen“) wurde die Beschränkung auf Perplexity-Rezepte zurückgenommen: In `data_rezepte5.js` ist der Zweig entfernt, der Rezepte ohne Perplexity-Quelle ausmusterte (Sicherung: `data_rezepte5.js.bak_perp`, im Dateikopf steht, wie man ihn wieder einschaltet). Ausgemustert bleiben nur r01 bis r76. Jedes Rezept zeigt seine Quelle im Feld `src` ehrlich an: „Perplexity-Recherche“ (18) oder „Eigene Fassung …“ (124). `smoke4.js` prüft jetzt 142 mitgelieferte Rezepte, 76 ausgemusterte und dass jede Quellenangabe mit „Perplexity“ oder „Eigene Fassung“ beginnt.
- **Charge 16 (28.10.2026, IDs r160 bis r177): 18 Rezepte ohne abrufbare Quelle.** Für Vegetarisch (7), Airfryer (6) und BBC-Meeresfrüchte (5) liegen die Links des Auftraggebers vor, die Seiten waren für Claudes Werkzeuge gesperrt und wurden nicht gelesen. Die Rezepte sind deshalb **eigene Fassungen nach allgemeinem Kochwissen**, nicht von Perplexity und nicht nach der Originalseite; Zeiten und Temperaturen sind Richtwerte und nicht getestet (das steht in `src` und in den Tipps). Geprüft mit `check_recipes.js` (0 Fehler, 1 Warnung: Korma nur 19 g Eiweiß). Bewusst **nicht** umgesetzt: Schweinebauch aus dem Airfryer, weil Schweinebauch nicht in der Nährwert-Datenbank steht und die Werte nicht erfunden werden dürfen. Zutaten, die nur ersatzweise gerechnet sind, stehen im Feld „unmapped“: Lasagneplatten (als Nudeln), Paella-Reis (als Risottoreis), ganzes Hähnchen (als Keule und Brust ohne Haut und Knochen). Rohdaten: `chargen/recherche_13_luecken.json`, Generator `chargen/_gen13.js`, Einbau `_mk_charge13.js`. Testzähler: volle Bibliothek 172 Rezepte, ausgelieferte 101.
- **Charge 17 (28.10.2026, ID r178): Halloumi aus dem Airfryer.** Der Link von BBC Good Food war gesperrt, deshalb eigene Fassung nach allgemeinem Kochwissen (Quelle und Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Die zweite Zusendung, eine Chefkoch-Suchseite „Airfryer“ (https://www.chefkoch.de/rs/s0/airfryer/Rezepte.html), enthält viele Rezepte und war ebenfalls gesperrt; sie wurde nicht umgesetzt, weil unklar ist, welche Rezepte gemeint sind. Rohdaten: `chargen/recherche_14_halloumi.json`. Testzähler: volle Bibliothek 173, ausgelieferte 102.
- **Charge 18 (28.10.2026, ID r179): Indisches Mango-Hähnchen-Curry.** Das Original auf Chefkoch ist für den Monsieur Cuisine; die Seite war gesperrt und wurde nicht gelesen. Da für den Monsieur Cuisine keine verifizierten Daten vorliegen, ist es eine eigene Fassung nach allgemeinem Kochwissen für Herd und Multikocher (Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Rohdaten: `chargen/recherche_15_mango.json`. Testzähler: volle Bibliothek 174, ausgelieferte 103. Der Planer-Test „Autoplan nutzt Gefrierer-Chili“ in test_logic2.js prüft jetzt die Seeds 5 bis 9 statt nur Seed 5, weil die Auswahl vom Rezeptbestand abhängt (Ergebnis 0/1/1/1/0 Blöcke, Vorrats-Bonus wirkt, höchstens 3 Blöcke).
- **Charge 19 (28.10.2026, IDs r180 bis r188): 9 Gerichte von monsieur-cuisine.com.** Die Seiten waren erreichbar, lieferten aber nur Titel und Beschreibung, keinen Rezepttext und keine Geräteeinstellungen. Es sind deshalb eigene Fassungen nach allgemeinem Kochwissen für Herd und Multikocher; es gibt keine Monsieur-Cuisine-Einstellungen, weil dafür keine verifizierten Daten vorliegen (Hinweis in `src`). Gerichte: Hähnchen mit Gemüse in Rahmsauce, Paprika-Sahne-Hähnchen, Hähnchenbrust mit Penne in Tomatenrahmsauce, Hähnchen-Kokos-Curry, Lachs mit Dillsauce, Rindersteaks in Tomatensauce, One-Pot-Hackfleischtopf, Spaghetti Bolognese, Kichererbsencurry. Geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Rohdaten: `chargen/recherche_16_monsieur.json`. Testzähler: volle Bibliothek 183, ausgelieferte 112.
- **Charge 20 (28.10.2026, IDs r189 bis r193): 5 Sous-vide-Gerichte.** Die Links (Chefkoch, Serious Eats) waren gesperrt und wurden nicht gelesen; eigene Fassungen nach allgemeinem Kochwissen (Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen): Schweinebraten (Schulter), Pulled Pork, Rindersteak, Paprika-Bowl, glasierte Möhren mit Quinoa. Die Gemüsegerichte sind um Eiweißbeilagen ergänzt (Ei, Kichererbsen, Edamame, Skyr), damit Mahlzeiten daraus werden. **Bewusst nicht umgesetzt:** Ribeye dry-aged, Heilbutt und Roastbeef dry-aged sous-vide, weil die Originale bei Rind und Fisch Temperaturen deutlich unter 70 °C nutzen, die App aber für Schwein, Rind und Fisch ein Bad von mindestens 70 °C verlangt (BfR-Regel in `data_sousvide.js`), und weil Heilbutt nicht in der Nährwert-Datenbank steht. Das Rindersteak ist deshalb eine durchgegarte Variante bei 70 °C und im Rezept als solche beschrieben. Garzeiten (12 bis 24 Stunden bei Schulter, 45 bis 60 Minuten bei Gemüse) sind Richtwerte, nicht getestet. Rohdaten: `chargen/recherche_17_sousvide.json`. Testzähler: volle Bibliothek 188, ausgelieferte 117.
- **Charge 21 (28.10.2026, IDs r194 bis r197): 4 Gerichte von BBC Good Food (Pot Roast Chicken, zwei Wurst-Bohnen-Eintöpfe, Wurzelgemüse-Linsen-Eintopf).** Die Seiten waren gesperrt; eigene Fassungen nach allgemeinem Kochwissen (Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Ersatz für fehlende Zutaten steht im Feld „unmapped“: Bratwürstchen sind nicht in der Nährwert-Datenbank, deshalb Chorizo (Eintopf 1) und selbst gewürzte Hackbällchen (Eintopf 2); das ganze Hähnchen ist als Keule und Brust ohne Haut gerechnet. Rohdaten: `chargen/recherche_18_eintoepfe.json`. Testzähler: volle Bibliothek 192, ausgelieferte 121.
- **Charge 22 (28.10.2026, IDs r198 bis r203): 6 Schweinegerichte von BBC Good Food** (Medaillons, Schulterbraten, Stroganoff, Schmorgericht mit Pflaumen, Pulled Pork, chinesisch geschmort mit Pflaumen). Die Seiten waren gesperrt; eigene Fassungen nach allgemeinem Kochwissen (Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Pflaumen und Reiswein stehen nicht in der Nährwert-Datenbank: Pflaumen sind als Apfel gerechnet, Reiswein ist weggelassen (Feld „unmapped“ und Tipp im Rezept). Rohdaten: `chargen/recherche_19_schwein.json`. Testzähler: volle Bibliothek 198, ausgelieferte 127; Bausteine: 48.
- **Dips, Grillsaucen und Marinaden (28.10.2026): 13 neue Bausteine (jetzt 48).** Auf Wunsch des Auftraggebers („es fehlt eine Kategorie Sauce, Dips und fertige Marinaden“) als Erweiterung der Bausteine statt als eigene Rezeptkategorie. Neues Feld `typ` in `data_bausteine.js`: 5 Dips im Modul Sauce (Hummus, Baba Ghanoush, Weiße-Bohnen-Dip, Rote-Bete-Hummus, gekochte Tomaten-Salsa), 2 Grillsaucen im Modul Sauce (BBQ-Sauce, zuckerarmes Ketchup), 6 Marinaden im Modul Booster mit 50-ml-Würfeln (Zitronen-Kräuter, Joghurt-Tandoori, Soja-Honig, Rauchpaprika, Limetten-Chili, Senf-Honig). Im Katalog gibt es einen zusätzlichen Filter „Art“ (`pktyp`). Die automatischen Mahlzeit-Vorschläge (`suggestMeals`) nehmen nur Bausteine ohne `typ`, damit Dips und Marinaden nicht ungefragt in Vorschläge geraten; im Baukasten sind sie wählbar. Alle 13 bestehen die automatische Prüfung (0 Fehler, 0 Warnungen). **Bewusst nicht aufgenommen:** Tzatziki und andere Joghurt-Kräuter-Dips, weil sie sich nicht einfrieren lassen, alle Bausteine aber Gefrierer-Blöcke sind. Hinweise: Marinaden, die mit rohem Fleisch oder Fisch in Kontakt waren, müssen weggeworfen oder 3 Minuten gekocht werden (steht in jedem Eintrag); die Nährwerte der Marinaden gelten für den ganzen Würfel, auf dem Teller landet meist weniger. Es sind eigene Grundzubereitungen nach allgemeinem Kochwissen, nicht getestet. Tests: 48 Bausteine, Filter nach Art, Vorschläge ohne Dips/Marinaden (`test_logic3.js` 197, `smoke3.js` 90).
- **Charge 23 (28.10.2026, IDs r204 bis r207): 4 weitere Schweinegerichte von BBC Good Food** (Sichuan-Art mit Brokkoli und Cashews, Pastinaken-Cobbler, langsam geschmorte Schulter mit Apfel und Pastinaken, grüne Bohnen in Austernsauce-Art). Die ersten sechs Links der Zusendung waren schon in Charge 22. Seiten gesperrt, eigene Fassungen nach allgemeinem Kochwissen (Hinweis in `src`), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Zum Zeitpunkt der Charge nicht in der Nährwert-Datenbank und deshalb ersetzt oder weggelassen: Sichuanpfeffer, Chilibohnenpaste, Reiswein, Cider, Austernsauce (durch Sojasauce und Honig), Backpulver. Cider, Backpulver, Pflaumen und Sichuanpfeffer sind inzwischen ergänzt (siehe unten), Reiswein, Chilibohnenpaste und Austernsauce fehlen weiter. Rohdaten: `chargen/recherche_20_schwein2.json`. Testzähler: volle Bibliothek 202, ausgelieferte 131.
- **Fehlende Zutaten ergänzt (28.10.2026), 9 neue Einträge in `data_zutaten.js`, `data_bls.js` und `bls_auszug.json`:** Pflaume/Zwetschge (F220100), Schweinebauch (U642100), Heilbuttfilet (T306100), Bratwurst (W222600, Rostbratwurst), Okra (G530100), Apfelwein/Cider (P353100), Rotwein trocken (P2A3000), Weißwein trocken (P210000), Backpulver (R421100), dazu das Gewürz Sichuanpfeffer (ohne Nährwert). Alle Werte stammen aus der BLS 4.0; Ballaststoff-, Alkohol- und Säureanteile für die Energieprüfung kommen aus derselben Tabelle, `test_bls.js` ist grün (9 Prüfungen). Sicherungen: `*.bak_fehlend`. **Nicht aufgenommen, weil nicht in der BLS und kein belegter Wert vorliegt:** Reiswein (Shaoxing), Austernsauce, Hoisin-Sauce, Mirin, Marsala, Stout, Tamarinde, Tilapia, Maistortilla; dafür braucht es die Packungsangabe. Die Rezepte mit diesen Zutaten behalten ihre Hinweise („fehlt in der Nährwert-Datenbank“).
- **Rezepte auf die neuen Zutaten umgestellt (28.10.2026):** r201 und r203 rechnen jetzt mit Pflaumen statt Apfel; r206 mit Cider (250 g Apfelwein, Brühe 400 auf 200 g, Essig 30 auf 15 g, Schritt 3 ergänzt); r205 mit 5 g Backpulver; r157 (Gumbo) mit 150 g Okra (Schritt 4 ergänzt); r107 mit 150 g Weißwein; r113 und r119 mit je 400 g Rotwein, r116 mit 150 g, r121 mit 125 g. Der Wein ist **voll eingerechnet** (Alkohol verdampft beim Kochen nur teilweise, der Anteil hängt von Garzeit und Gefäß ab, deshalb keine Abschläge geschätzt). Folge: r113 694, r119 657 und r116 667 kcal je Portion liegen jetzt über dem Zielband von 650 kcal. Der frühere Tipp „400 ml Rotwein ergeben etwa 40 kcal je Portion“ war falsch (richtig sind 70 kcal) und wurde korrigiert. In r195 und r196 verweisen die Tipps jetzt auf Bratwurst (178 kcal je 100 g), gerechnet sind weiter Chorizo und Hack.
- **Charge 24 (28.10.2026, ID r208): Schweinebauch aus dem Airfryer** (Kartoffeln, Apfel-Kohlsalat mit Skyr). Link war gesperrt, eigene Fassung nach allgemeinem Kochwissen (Hinweis in `src`), 570 kcal und 30,3 g Eiweiß je Portion, nach Rohgewicht gerechnet (ein Teil des Fettes bleibt im Gerät), geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Garzeiten sind Richtwerte, nicht getestet. Rohdaten: `chargen/recherche_21_schweinebauch.json`. Testzähler: volle Bibliothek 203, ausgelieferte 132.
- **Charge 25 (28.10.2026, IDs r209 bis r214): 6 Chinagerichte von Chefkoch** (gebratene Nudeln mit Hähnchen, Peking-Suppe, Gong Bao Chicken, Kung-Pao-Huhn mit Hoisin, Bratnudeln mit Tofu, Jiaozi). Chefkoch ist für die Werkzeuge gesperrt (`SITE_BLOCKED`), die Seiten wurden nicht gelesen und nicht umgangen; es sind eigene Fassungen nach den Titeln und allgemeinem Kochwissen (Hinweis in `src`). Die Zutaten der Originale sind daher unbekannt, die Rezepte können davon abweichen. Gong Bao und „Kung Pao 2“ sind bewusst zwei verschiedene Ansätze (scharf mit Sichuanpfeffer und Sellerie gegen süßer mit Paprika, Zucchini und Hoisin); das ältere r102 bleibt daneben bestehen. Die Bratnudeln sind als vegetarische Tofu-Variante geschrieben, weil das Original unbekannt ist. Jiaozi sind einfrierbar (roh vorfrieren). Geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen); Jiaozi liegen mit 29,2 g Eiweiß knapp unter dem Ziel von 30 g. Reiswein, Austernsauce und Hoisin sind mit den neu ergänzten Zutaten gerechnet. Rohdaten: `chargen/recherche_22_chefkoch_china.json`. Testzähler: volle Bibliothek 209, ausgelieferte 138.
- **Alkalische Nudeln (碱水面) als Beilagen-Baustein (28.10.2026, jetzt 57 Bausteine):** Rezept aus der Zusendung des Auftraggebers (Mehl, gebackenes Soda, kaltes Wasser, Salz). Als Beilage mit Gefrierwürfeln (250 ml) angelegt, doppelter Ansatz (600 g Mehl, etwa 800 g Teig), Kochzeit auf 1 bis 2 Minuten gekürzt, damit sie beim Aufwärmen nicht weich werden. **Die Ausbeute nach dem Kochen (1040 g) ist geschätzt, nicht gewogen**, und bestimmt Würfelzahl und Füllgrad. Neues Gewürz „Backsoda / Natron (gebacken)“ ohne Nährwert. Sicherheitshinweis zum stark alkalischen gebackenen Soda im ersten Schritt ergänzt (Augen, Haut, Kinder, Aluminium). Haltbarkeit im Würfel 1 Monat nach Angabe des Auftraggebers. Sicherungen: `*.bak_laugennudeln`. Testzähler: 57 Bausteine.
- **Charge 26 (28.10.2026, IDs r215 bis r218): 4 weitere Chinagerichte von Chefkoch** (Rindfleisch mit Paprika und Zwiebeln, Nudelpfanne mit Schweinefilet, Blumenkohl-Hähnchen-Pfanne, gebratene Nudeln mit Garnelen). Der Auftraggeber schickte die ersten sechs Links erneut mit vier neuen; die sechs bekannten sind in Charge 25 und wurden nicht doppelt gebaut. Chefkoch ist gesperrt (`SITE_BLOCKED`), eigene Fassungen nach den Titeln (Hinweis in `src`); die Zutaten der Originale sind unbekannt. Damit die vier Nudelgerichte (Charge 25 und 26) nicht gleich sind, haben sie verschiedene Hauptzutaten (Hähnchen, Tofu, Schwein, Garnelen). Die Blumenkohl-Pfanne ist ohne Kenntnis des Originals eine Curry-Sahne-Fassung. Geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Rohdaten: `chargen/recherche_23_chefkoch_china2.json`. Testzähler: volle Bibliothek 213, ausgelieferte 142.
- **9 Saucen als Bausteine (28.10.2026, jetzt 66 Bausteine, davon 22 im Modul Sauce):** Auf Zusendung von 9 BBC-Good-Food-Links (Pizzasauce, Käsesauce, weiße Sauce, süß-saure Sauce, Tomaten-Basilikum, Pilzsauce, Pfeffersauce, Brotsauce, Arrabbiata). Die Seiten sind gesperrt (`SITE_BLOCKED`), nicht gelesen und nicht umgangen: eigene Grundzubereitungen nach allgemeinem Kochwissen, das steht im `src` jedes Bausteins. Als Saucen-Würfel (125 ml) ohne Art-Kennzeichnung, damit sie in den Gericht-Vorschlägen auftauchen. Die süß-saure Sauce kommt ohne Ananas aus (feste Ausschlussregel). Die Brotsauce rechnet mit Semmelbröseln statt frischem Weißbrot, weil dieses nicht in der Datenbank steht. Milchbasierte Saucen sind mit Mehl oder Stärke gebunden und tauen mit Rühren wieder glatt, können sich aber leicht trennen. Mengen und Zeiten sind Richtwerte, nicht getestet. Generator: `_add_saucen.js`, Sicherung `data_bausteine.js.bak_saucen`. Testzähler: 66 Bausteine, Filter Sauce 22 Karten.
- **3 Dressings und ein Grünkohl-Pesto als Bausteine (28.10.2026, jetzt 70 Bausteine):** Aus einer Zusendung mit drei Chefkoch-Dressings (Joghurt ohne Sahne, Schmand-Kräuter, Ahornsirup-Senf), einem BBC-Link-Block (die sechs bekannten Saucen waren schon da, nicht doppelt gebaut) und dem Kale-Pesto von Good Food Middle East. Chefkoch und BBC sind gesperrt: die drei Dressings sind eigene Fassungen nach den Titeln. Das **Kale-Pesto war lesbar**: Mengen von dort übernommen (Pinienkerne 85 g, Parmesan 85 g, Knoblauch, Öl 150 ml, Grünkohl 85 g, Zitrone), Text in eigenen Worten, Quelle im `src` genannt. Neue Art „Dressing“ (Booster-Würfel, 50 ml) mit Filter im Katalog; neue Zutaten Grünkohl (BLS G322100) und Ahornsirup (BLS S151100). Die beiden Milch-Dressings frieren schlecht (trennen sich, können körnig bleiben): trotzdem als Würfel angelegt, mit Hinweis im Tipp und nur 1 Monat Haltbarkeit; frisch halten sie etwa 3 Tage. Generator `_add_dressings.js`, Sicherungen `*.bak_dressings`. Testzähler: 70 Bausteine, 3 Dressings.
- **Saisonkalender als eigene Übersicht (28.10.2026):** Auf Wunsch des Auftraggebers („Baue die Übersicht komplett nach deinem Ermessen in die App ein. Mache es unabhängig von einer Quelldatei“). Grundlage war die hochgeladene Tabelle „Saisonkalender Gemüse und Obst“, die zuerst analysiert, bewertet und um Lücken ergänzt wurde. Die App enthält die Daten jetzt fest in `data_saison.js` (45 Zutaten mit einem 12-Monats-Muster: F frisch aus Deutschland, G Gewächshaus oder Zucht, L Lagerware, I Import, – keine Frischware; Herkunft k = aus der Tabelle, e = eigene Einschätzung, im Kalender mit * markiert) und braucht keine Quelldatei. Neun Zutaten sind bewusst neutral (Banane, Avocado, Orange, Mango, Okra, Limetten- und Zitronensaft, Rote Bete vorgegart, Trüffel), jeweils mit Begründung. Süßkartoffel und Ananas bleiben ausgeschlossen und kommen nicht vor. **Wo in der App:** Plan → Saison (Monatswahl, was jetzt frisch ist, Rezeptvorschläge, Jahresübersicht als Heatmap mit Korrekturmodus), Rezeptliste (Abzeichen „Saison“ / „teils“, Filter „Saisonal im Monat ab 80 %“, Sortierung zuerst saisonale Rezepte), Rezeptdetail (Tabelle je Zutat), Karte „Jetzt frisch“ auf Heute, Hinweise im Einkauf und in der Wochenplanung, Mehr → Saison (Ein/Aus, Auto-Plan-Vorliebe, Korrekturen zurücksetzen). **Rechenweg:** Der Saison-Score eines Rezepts ist das gewichtete Mittel der bewertbaren Zutaten (Gewicht = Wurzel der Grammzahl; frisch 1, Lager 0,75, Gewächshaus 0,5, Import und keine Frischware 0). Er gilt nur bei mindestens zwei bewertbaren Zutaten, die mindestens 15 % der Zutatenmenge ausmachen; ganzjährig gleich bewertete Zutaten zählen nicht. Ab 0,8 gilt „Saison“, ab 0,5 „teils“. **Eigene Korrekturen** (`saisonOv`) überstimmen die mitgelieferten Muster, werden im Stand gespeichert, bereinigt und mit exportiert. **Auto-Plan:** Die Saison-Vorliebe ist ein weicher Abzug im Punktewert und standardmäßig aus; ohne sie bleibt der Auto-Plan unverändert. **Grenzen:** Die Daten sind eine Orientierung und nicht gegen die Originalquellen geprüft (die Hessen-PDF war nicht erreichbar, im WWF-PDF gehen die Markierungen beim Textauszug verloren); eigene Einschätzungen sind mit * gekennzeichnet; Saison hängt vom Standort und Jahr ab, deshalb gibt es den Korrekturmodus. Bausteine zeigen keine Saison. Tests: `test_saison.js` (57 Prüfungen) und `smoke5.js` (85 Prüfungen); der Prüfbericht in der App (Mehr → Prüfen) enthält sieben Saison-Prüfungen.
- **Zutaten-Recherche für Lücken der BLS (28.10.2026), 9 weitere Einträge:** Stout (BLS P177000) und Marsala (Näherung mit BLS Sherry medium dry P462000) stammen aus der BLS. Außerhalb der BLS, als eigene Werte mit Quelle im Hinweistext: Austernsauce (USDA SR Legacy 174529: 51 kcal), Hoisinsauce (USDA, NDB 6175: 220 kcal), Tilapia (USDA 175176: 96 kcal), Tamarinde (USDA 167763: 239 kcal), Maistortilla (USDA 175036: 218 kcal), Reiswein (Näherung mit Sake nach USDA 167723: 134 kcal) und Mirin (Packungsangabe eines Händlers, 236 kcal je 100 g; ein zweiter Händler nennt 239 kcal je 100 ml). Die USDA-Werte wurden von den Datenseiten der Tabelle abgelesen, nicht gegen die Originaldatenbank geprüft. **Verworfen:** Reiswein-Werte der ReciPal-Seite (stammen aus Wikipedia, 0 g Kohlenhydrate, unbrauchbar); Chilibohnenpaste (Doubanjiang): zwei Aggregatorseiten nennen widersprüchliche Makros (92 kcal mit 8 g Eiweiß gegen 3 g Eiweiß), keine Herstellerangabe, daher nicht aufgenommen. Allergene sind bei Austernsauce, Hoisin, Reiswein und Stout vorsorglich gesetzt (Weichtiere, Soja, Weizen bzw. Gerste), weil die Zusammensetzung je Marke schwankt. `test_bls.js` ist grün (9 Prüfungen). Sicherungen: `*.bak_recherche`.
- **8 Öl-Marinaden als Bausteine (28.10.2026, jetzt 56 Bausteine, davon 14 Marinaden):** Aus der Zusendung des Auftraggebers (Textdatei mit acht Rezepten): Premium Trüffel, Schwarzer Knoblauch, Buttrige Kräuter, Fresh Lemon, Scharfe Schote, Smoky Whiskey, Sweet Bacon, Alter Pfeffer. Alle als Booster-Würfel (50 ml, 4 bis 5 Würfel je Ansatz, Typ „Marinade“), nur zum Einfrieren gedacht. Neue Zutaten dafür: Whisky (BLS P733000, für den Bourbon) und Trüffel roh (BLS K702100); neue Gewürze Estragon, weißer und grüner Pfeffer. Als Näherung gerechnet und im Tipp genannt: Trüffelöl wie Rapsöl, schwarzer Knoblauch wie rohe Knoblauchzehe, brauner Zucker wie Zucker. Weggelassen (zu geringe Mengen): Zitronenschale, Hefeextrakt, Tomatenpulver, Flüssigrauch. **Abweichung von der Vorlage:** Die Vorlage nennt 1 bis 2 Wochen Kühlschrank für Marinaden mit frischem Knoblauch, Kräutern, Zitrone oder Chili in Öl; wegen des Botulismus-Risikos steht in den Tipps stattdessen „sofort einfrieren, im Kühlschrank höchstens wenige Tage“. Die Nährwerte gelten für den ganzen Würfel (335 bis 446 kcal); ein Teil des Öls bleibt beim Braten zurück. Generator: `_add_marinaden.js`, Sicherungen `*.bak_marinaden`. Testzähler: 56 Bausteine, 14 Marinaden.
- **Rezepte auf die recherchierten Zutaten umgestellt (28.10.2026):** r98, r135 und r207 (Austernsauce 30 bis 40 g, Sojasauce entsprechend gekürzt; r207 heißt jetzt „Schwein mit grünen Bohnen in Austernsauce“), r99 (Mirin 45 g, Honig gekürzt), r102 und r203 (Reiswein 5 bzw. 30 g), r105 (Marsala 100 g), r111 und r131 (Hoisinsauce 30 g, Honig gekürzt), r136 (Stout 330 g, Brühe 600 auf 450 g), r115 (Maistortilla statt Weizentortilla, 462 kcal, 39,8 g Eiweiß), r204 (Sichuanpfeffer als Gewürz in Schritt 4). Alkohol ist voll eingerechnet.
- **Charge 7: 7 Rind-Gerichte (28.10.2026, IDs r112 bis r118):** Aus 7 weiteren Allrecipes-Links (Seite gesperrt) wurden wie bei Charge 6 eigene Fassungen nach Recherche bei anderen Quellen geschrieben (Boeuf Stroganoff, Boeuf Bourguignon, Bulgogi, Carne-Asada-Tacos, Lasagne al forno, American Goulash, Aloo Keema; Rohdaten `chargen/recherche_04_rind.json`). Herkunft in `src` ehrlich angegeben, **nicht von Perplexity**. „Classic Goulash“ wurde als amerikanischer Hackfleisch-Nudel-Topf umgesetzt (das ungarische Gulasch gibt es als r77 und r94). Wein, Sesamöl, Gewürzgurken sind nicht in den Nährwerten (in `tips` benannt). 509 bis 641 kcal je Portion, 38 bis 57 g Eiweiß. Bibliothek jetzt 42 Rezepte. `test_logic2.js` prüft Dubletten jetzt getrennt für die ausgelieferten Rezepte (ab r77) und die als Testdaten behaltenen alten (die alte Lasagne r29 ähnelt der neuen r116).
- **Charge 6: 17 Hähnchen-Gerichte (28.10.2026, IDs r95 bis r111):** Der Auftraggeber schickte 19 Allrecipes-Links. Die Seite war für die Werkzeuge gesperrt, die Rezepte wurden deshalb **nicht** übernommen. Auf ausdrückliche Bitte („Bitte einbauen“) schrieb Claude eigene Fassungen der Gerichte nach Recherche bei anderen Quellen (LECKER, Saveur, NPR, Rouxbe, Italy Abroad u. a.; je Rezept im Feld `src` genannt, Rohdaten `chargen/recherche_03_haehnchen.json`). **Diese 17 Rezepte stammen nicht von Perplexity**, obwohl der Auftraggeber sonst nur Perplexity-Rezepte wollte; die Herkunft steht ehrlich in `src`. Weggelassen: Tajine (schon r84) und Saté (schon r91). Nicht abbildbare Zutaten (Mirin, Marsala, Weißwein, Austernsauce, Hoisin, Sichuanpfeffer, Shaoxing-Reiswein, Scotch-Bonnet) sind in `tips` benannt und nicht in den Nährwerten. Alle liegen laut `check_recipes.js` bei 526 bis 645 kcal je Portion. Bibliothek jetzt 35 Rezepte. Als Nächstes wollte der Auftraggeber Rind-Gerichte.
- **Nur-Perplexity-Bibliothek (28.10.2026):** Auf Wunsch des Auftraggebers sind alle eigenen mitgelieferten Rezepte (r01 bis r76, 71 noch vorhandene) ausgemustert; die App liefert nur noch die Rezepte ab r77 (18 Perplexity-Rezepte r77 bis r94 plus die 17 recherchierten Hähnchen-Gerichte r95 bis r111). Umsetzung in der neuen Datei `data_rezepte5.js` (setzt beim Start `RETIRED`, entfernt die Rezepte aus `SEED`; `mergeState()` bereinigt Plan, Favoriten, Prep; Gefrierer-Einträge behalten ihren Namen). Die Tests laufen weiter gegen die volle Bibliothek (`const FULL_LIBRARY = true;` wird vorangestellt), neu ist `smoke4.js` für die ausgelieferte Fassung. Folge: kein Frühstück, Snack, Salat/Bowl, Fisch oder Vegetarisch mehr; der Autoplan füllt 14 Plätze, aber keinen Tag im Kalorienkorridor. Rückgängig: `data_rezepte5.js` aus `build.js` und `make_zip.js` nehmen. Nutzereigene Rezepte (`userRecipes`) sind nicht betroffen.
- **Perplexity-Charge 2 integriert (Sous-vide):** 9 geprüfte Rezepte (IDs r86 bis r94: Hähnchenbrust, Hähnchenkeulen-Confit, Putenbrust, Putenhackbällchen, Hähnchen-Fajitas, Hähnchen-Saté, Königsberger Klopse mit Putenhack, Pulled-Beef-Tacos, Sous-vide-Gulasch) am Ende von `data_rezepte4.js` angehängt; Rohdaten und geprüfte Fassung in `chargen/perplexity_02_*`. Keine Rezepte ausgemustert, weil es sich um andere Zubereitungen als die eigenen Rezepte handelt (Rinder-Fajitas, Klopse mit Kalb/Schwein, Pulled Pork aus dem Dutch Oven). Das zehnte Rezept der Charge (Sous-vide-Pulled-Pork) fehlt, weil die Eingabe bei 20.000 Zeichen abgeschnitten war, und muss erneut geschickt werden. Bibliothek jetzt 89 Rezepte. Der Sous-vide-Test in `test_logic3.js` liest die Badtemperatur jetzt aus dem Satz mit "Wasserbad", egal in welcher Reihenfolge; die Rezepttexte blieben unverändert.
- **Perplexity-Prompt auf Teilausgaben umgestellt:** `mk_prompt.js` verlangt jetzt höchstens 8 Rezepte pro Antwort (Felder ANZAHL, VON, BIS) und kompaktes JSON ohne Einrückung, damit die Antwort unter der 20.000-Zeichen-Grenze des Chats bleibt. Das Beispiel im Prompt ist ebenfalls kompakt und besteht weiterhin die Prüfung.
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

**data_saison.js**: `S`

**logic4.js**: `saisonPattern`, `saisonYearRound`, `saisonRated`, `saisonCode`, `saisonScore`, `saisonMonths`, `saisonRangeText`, `saisonBest`, `saisonLevel`, `saisonMonthSummary`, `saisonTop`, `saisonWeek`, `saisonShopState`, `saisonCycle`, `saisonCleanOv`, `runQASaison`

**app_a.js**: `emptyWeek`, `defaultPrep`, `defaultState`, `mergeState`, `loadLocal`, `save`, `flush`, `initRuntime`, `toast`, `copyText`, `saveFile`, `statusOf`, `mealForSlot`, `goalsNow`, `dayRec`, `dayW`, `ringSvg`, `macroBar`, `weekBars`, `weightChart`, `entryRow`, `fastCard`, `viewHeute`, `addListHtml`, `addSheet`

**app_b.js**: `cuisines`, `filtered`, `activeFilterCount`, `recipeCard`, `renderList`, `distBlock`, `viewRezepte`, `nutriBlock`, `checksBlock`, `ingredientRows`, `spiceChips`, `recipeText`, `recipeSheet`, `newEd`, `edFrom`, `edToRecipe`, `edResultsHtml`, `editorSheet`, `kiImportPrompt`

**app_c.js**: `seg`, `slotOptions`, `goalBar`, `viewPlanTab`, `viewPlan`, `shopData`, `shopKey`, `shopText`, `viewShop`, `viewVorrat`, `expBadge`, `viewStock`, `usageCount`, `viewSpices`, `viewFreezer`, `viewKochen`, `stockResultsHtml`, `stockAddSheet`, `ingSheet`, `openAttr`, `calcPanel`, `infoPanel`, `backupPanel`, `profilePanel`, `statusPanel`, `libPanel`, `viewMehr`

**app_d.js**: `render`, `renderSheet`, `openSheet`, `aiErr`, `exclText`, `profileText`, `runPlanKI`, `runVorratKI`, `runImportKI`, `confirmed`, `dropRecipeRefs`, `logEntry`, `invPortion`, `eatInv`, `markCooked`, `buyGrams`, `addMonthsIso`, `fillStep`, `recomputeEntry`, `newInvForm`, `runAuto`

**app_f.js**: `saisonOn`, `saisonOvNow`, `saisonRealMonth`, `saisonMonth`, `saisonName`, `saisonUsable`, `saisonBadge`, `saisonOptionTag`, `saisonStrip`, `saisonStateBadge`, `saisonRecipePanel`, `saisonTag`, `saisonShopNote`, `saisonWeekLine`, `saisonPlanSwitch`, `saisonPlanFn`, `saisonPicks`, `saisonFMonths`, `saisonRankSort`, `saisonUsage`, `saisonHeute`, `saisonHeat`, `viewSaison`, `saisonSettings`

**app_e.js**: `prepSt`, `prepAllergBadges`, `cubeLine`, `isUsable`, `viewPrep`, `prepStart`, `prepKatalog`, `bausteinSheet`, `prepBauen`, `prepFreezable`, `prepSession`, `prepToast`
