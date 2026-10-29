# PROTOKOLL: Unendlicher Speiseplan (Übergabe an jede KI und jeden Chat)

Stand dieser ZIP: 2026-10-28 22:18 UTC · Paket: App (Ausspielung). Der Quellcode steht in speiseplan-projekt.zip.

**Zweck dieser Datei:** Ein neuer Chat oder eine andere KI soll nur diese ZIP lesen und ohne Rückfragen weiterarbeiten können. Alles Wichtige steht hier: Vorgaben des Auftraggebers, Architektur, Datenformate, Teststand, bekannte Fallen, offene Aufgaben und ehrliche Grenzen. Die Zahlen in Abschnitt 3 und 4 sind beim Packen automatisch aus dem Quellcode erzeugt und stimmen mit dieser ZIP überein. Alles andere ist von Hand geschrieben. Bei Widerspruch gilt der Quellcode.

---

## 0. Kurzanleitung zum Weiterarbeiten (in dieser Reihenfolge)

1. Diese Datei vollständig lesen, besonders Abschnitt 1 (Vorgaben) und Abschnitt 9 (Arbeitsregeln).
2. Voraussetzung ist nur **Node.js** (ab Version 18, getestet mit 22). Keine npm-Pakete nötig. Für den optionalen Browsertest zusätzlich Python 3 mit Playwright und Chromium.
3. Im Projektordner ausführen: `node build.js`. Das erzeugt `speiseplan.html` (Variante für Claude-Artifacts) und den Ordner `dist/` (eigenständige App als PWA: `index.html`, `sw.js`, `manifest.json`, Icons).
4. Tests ausführen (alle müssen grün sein, Erwartungswerte in Abschnitt 5): `node test_logic2.js`, `node test_logic3.js`, `node smoke2.js`, `node smoke3.js`, `node test_saison.js`, `node smoke5.js`, `node test_quellen.js`.
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

### 1.4 Bedingte Ausschlüsse (Stand 28.10.2026)
**Ananas, Süßkartoffel(n) und Bataten (Batate)** sind **erlaubt, aber nicht als Hauptgericht** (Auftraggeber, 28.10.2026; Süßkartoffel und Batate waren zuvor überall ausgeschlossen). Erlaubt sind sie in den Kategorien Frühstück, Snack und Dessert. Gesperrt bleiben sie in Hauptgericht, Suppe/Eintopf, Salat/Bowl und in den Meal-Prep-Bausteinen, einschließlich aller Schreibweisen und Zusammensetzungen (zum Beispiel „Sweet Potato“, „Ananasstücke“) in Zutaten, Rezeptnamen, Schritten, Tags, Tipps, Importen, KI-Antworten. Umgesetzt in `FIXE_BEDINGT`, `EXCL_HAUPT_KEYS` und dem Parameter `haupt` von `exclusionHits()`/`recipeExclusionHits()` (`isHauptRezept()`); `FIXE_AUSSCHLUESSE` und `EXCL_KEYS` sind leer und bleiben für künftige feste Ausschlüsse bestehen. In freien Texten ohne Rezeptkategorie (Gefrierer-Namen, Tagebuch-Schnelleinträge, Einkaufsextras) werden sie nicht blockiert. Auslegung: „nicht als Hauptgericht“ gilt hier auch für Suppe/Eintopf und Salat/Bowl, weil sie als Hauptmahlzeiten zählen; Im Zutatenstamm gibt es Ananas (BLS F501100, bei den neutralen Saison-Zutaten geführt); eine Zutat Süßkartoffel gibt es noch nicht und wäre bei Bedarf mit BLS-Wert nachzutragen. Die Prüfung wird auch für die Bausteine im Meal-Prep-Bereich ausgeführt. Weitere persönliche Ausschlüsse sind unter „Mehr → Allergene, Unverträglichkeiten, Ausschlüsse“ einstellbar (`state.profile.excl`).

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
| Mitgelieferte Rezepte | **258** von 240 geplanten |
| Verteilung nach Gruppe | Geflügel: 52, Rind: 47, Schwein: 36, Fisch: 26, Vegetarisch: 97 |
| Verteilung nach Mahlzeit | Hauptgericht: 124, Suppe/Eintopf: 45, Salat/Bowl: 13, Frühstück: 28, Snack: 4, Dessert: 44 |
| Einfrierbar: ja / eingeschränkt | 123 / 39 |
| Zutaten im Stamm (davon mit Nährwerten) | 208 (207) |
| Gewürze | 47 |
| Bausteine (Meal-Prep) | **71** (Protein: 12, Beilage: 8, Sauce: 23, Gemüse: 5, Booster: 23) |
| Einkaufskategorien | Fleisch & Geflügel, Fisch & Meeresfrüchte, Gemüse, Obst, Milchprodukte, Eier, Tiefkühlprodukte, Trockenprodukte, Konserven, Gewürze & Öle, Sonstiges |
| Geräte-Werte | Airfryer, Kontaktgrill, Multikocher, Sous-vide, Herd, Backofen, Monsieur Cuisine Connect, Dutch Oven |
| Feste Ausschlüsse | keine ganz ausgeschlossenen Zutaten; Ananas, Süßkartoffel und Batate sind erlaubt, aber nicht als Hauptgericht (nur Frühstück, Snack, Dessert) |
| Größe `dist/index.html` | 1203 KB |

---

## 4. Architektur

### 4.1 Prinzip
Eine einzige HTML-Datei mit eingebettetem JavaScript, **zusammengesetzt durch Verkettung** (`build.js`). Es gibt kein Framework, keine Bibliotheken, keine externen Anfragen (Google Fonts nur in der Artifact-Variante). Reine Logik (ohne DOM) ist von der Oberfläche getrennt und in Node testbar.

### 4.2 Reihenfolge der Verkettung (wichtig, alles teilt einen globalen Geltungsbereich)
`logic.js` → `data_zutaten.js` → `data_bls.js` → `data_sousvide.js` → `data_rezepte1.js` bis `data_rezepte5.js` → `SEED2.forEach(r => SEED.push(r))` → `logic2.js` → `data_bausteine.js` → `logic3.js` → `data_saison.js` → `logic4.js` → `look.js` → `app_a.js` → `data_quellen.js` → `app_b.js` → `app_c.js` → `app_d.js` → `app_f.js` → `app_e.js`. Die Reihenfolge steht in `build.js`; neue Dateien dort **und** in `make_zip.js` (Listen `SOURCES`, `TESTS`, `TOOLS`) eintragen.
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
| `data_saison.js` | Saisonkalender als eigene Daten (`SAISON`: 47 Zutaten mit 12-Monats-Muster, `SAISON_NEUTRAL`: bewusst nicht bewertet); unabhängig von jeder Quelldatei | 
| `logic4.js` | Saison-Logik: Rezept-Score je Monat, Monatsübersicht, Einkaufsstatus, Wochenanteil, eigene Korrekturen, Prüfbericht (`runQASaison`) | 
| `look.js` | Darstellung je Gerät: Palettenrechnung mit Kontrastprüfung, Farbmodus, Akzent, Schriftgröße, Dichte, Layout (`lookApply`, `lookPanel`); eigener Speicherschlüssel `speiseplan-look-v1` | 
| `app_a.js` | Zustand (`defaultState`, `mergeState`), Speicher, Hilfen, Ansicht Heute, Hinzufügen-Blatt | 
| `data_quellen.js` | Geprüfte und korrigierte externe Rezeptreferenzen; separate Links ohne Änderung an Rezepten | 
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
| `smoke6.js` | Oberflächentest Ausrüstung (Souper Cubes editierbar), Darstellung (Panel, Spalten, schmale Ansicht) und Dessert (Kategorie, Plan-Platz, Auto-Plan, Tagebuch) | 
| `test_look.js` | Tests Darstellung: Kontrast aller 6 Paletten hell und dunkel, Bereinigung, Anwenden auf ein Dokument, Host-Theme, Speicherung | 
| `test_quellen.js` | Quellen-Link-Mapping, HTML-Sicherheit, Status und Vollständigkeit der Zuordnungen | 
| `pw_prep.py` | Browsertest Prep (Playwright, 400 px, hell/dunkel) | 
| `pw_lib.py` | Browsertest Bibliotheksverwaltung | 
| `make_zip.js` | Dieses Packskript | 
| `mk_prompt.js` | Erzeugt `perplexity-prompt-rezepte.txt` aus den aktuellen Zutatenlisten | 
| `check_recipes.js` | Prüfbericht für Perplexity-Rezepte vor dem Import: `node check_recipes.js datei.json` | 
| `make_icons.py` | Erzeugt alle App-Icons aus `icons/icon-quelle.png` (nach Änderung der Quelle ausführen, danach `node build.js`) | 
| `gesperrte_links.txt` | Links und Titel der für Claude gesperrten Rezeptseiten, mit Prompt zum Kopieren für ChatGPT (auch im Protokoll, Anhang 13) | 
| `QUELLEN_PRUEFBERICHT.md` | Prüfergebnis und offene Originaladressen, mit Änderungsnachweis | 
| `QUELLEN_PRUEFBERICHT.csv` | Jede gelieferte Referenz samt Zuordnung, Link und Prüfergebnis | 
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
- `equip{formen{50,120,125,250,500}, kammern{50,120,125,250,500}}` (Souper-Cubes-Ausrüstung, siehe 4.11), `dessertOn` (Dessert-Platz im Plan und Tagebuch sichtbar, Standard aus), Wochentage haben zusätzlich den Platz `d` (Dessert; alte Stände werden von `mergeState()` ergänzt)
- `saisonOn` (Saison-Anzeige, Standard an), `saisonPlan` (Saison-Vorliebe im Auto-Plan, Standard aus), `saisonOv{id: 12-Zeichen-Muster}` (eigene Korrekturen am Saisonkalender, `saisonCleanOv()` bereinigt beim Laden)

### 4.5 Oberfläche
- Sechs Tabs (`TABS` in `app_d.js`): **Heute** (Tagebuch), **Rezepte**, **Plan** (Wochenplan, Einkauf, Saison), **Vorrat** (Zutaten, Gewürze, Gerichte, Kochen), **Prep** (Start, Katalog, Bauen, Session), **Mehr** (Ziele, Rechner, Profil, Darstellung, Ausrüstung, Rezept-Bibliothek verwalten, Sicherung, Prüfbericht, Status). Auf dem PC (Layout `desktop`) steht die Tab-Leiste als feste Seitenleiste links, der Inhalt ist bis 1180 px breit und wird in Heute, Plan und Mehr in zwei Spalten (`.cols`/`.col`) gelegt; auf dem Handy ist `.cols{display:contents}`, die Ansicht bleibt einspaltig.
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
- `freeze.block` ∈ 50, 120, 125, 250, 500 ml je Portion (120 ml seit 28.10.2026).
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

### 4.11 Ausrüstung (Souper Cubes), Stand 28.10.2026
Die Zahl der Formen und die Kammern je Form sind im Zustand (`state.equip`) editierbar (Mehr → Ausrüstung, Gefrierer-Ansicht, Prep-Formatetabelle über `equipTable()`). Standard: 4 × 500 ml (2 Kammern), 4 × 250 ml (4), 2 × 125 ml (6), 2 × 50 ml (9) = 12 Formen, 54 Kammern, 10,4 L. Seit 28.10.2026 gibt es als fünfte Größe **120 ml** (Wunsch des Auftraggebers, weil der eingetragene Bestand nicht stimmte); Standardbestand 0 Formen mit 6 Kammern je Form als Voreinstellung, die Standardsumme (12 Formen, 54 Kammern, 10,4 L) bleibt unverändert, und für 120 ml erscheint bei 0 Formen keine Warnung. Grenzen: Formen 0 bis 24, Kammern 1 bis 30. `KAMMERN` bleibt die Liste gültiger Blockgrößen und Standardwerte (Rezeptprüfung); die Kapazität kommt aus `kap(ml)` (Formen × Kammern, über `equipNow()`), für Divisionen gilt `kapN(ml) = max(1, kap)`. Bei 0 Formen warnt die Oberfläche, es gibt keine Division durch null. `logic3.js` (Runden, Verbrauch, Prep-Runden) nutzt `kap`/`kapN`. Die Ausrüstung liegt im Zustand und damit in der Sicherung.

### 4.12 Darstellung (`look.js`), Stand 28.10.2026
Einstellungen **je Gerät** unter dem eigenen Schlüssel `speiseplan-look-v1` (nicht im Zustand, nicht in der Sicherung). Farbmodus auto/hell/dunkel (ein vom Host gesetztes `data-theme` wird gemerkt und beim Zurücksetzen wiederhergestellt), Akzent Petrol (Original, keine Überschreibungen), Wald, Ozean, Beere, Terrakotta, Rose (per HSL erzeugt, Kontrastschleifen nach WCAG, getestet hell und dunkel), Schriftgröße klein/normal/groß/sehr groß über `--fz` (Inline-Schriftgrößen in der App sind `calc(… * var(--fz))`), Dichte normal/kompakt, Layout automatisch (ab 1000 px Breite PC) / Handy / PC. `lookInit()` läuft beim Laden und prüft, ob `document` existiert. Wer auf dem Handy das PC-Layout erzwingt, bekommt es so, wie er es wählt. **Nur in headless Chromium geprüft** (390, 1100, 1440 px, kein Querscrollen), nicht auf echten Geräten.

### 4.13 Dessert, Stand 28.10.2026
Kategorie `Dessert` in `MEALS`, Platz `d` in `SLOTS`, angezeigt nur bei `state.dessertOn` oder wenn belegt (`slotsOn/slotsPlan/slotsDiary`). `slotFits('d')` nur Dessert, `slotFits('s')` Snack und Dessert. Rezeptprüfung für Dessert: 120 bis 350 kcal, mindestens 10 g Eiweiß je Portion (Hauptmahlzeiten weiter 400 bis 650 kcal und mindestens 30 g). Der Auto-Plan reserviert bei `dessert: true` durchschnittliche Dessert-kcal und -Eiweiß in der Suche und wählt danach ein Dessert (höchstens 12 % über dem Ziel, Rückfall auf das kalorienärmste bis 20 %). Standardmäßig aus, damit bestehende Pläne und Tagessummen unverändert bleiben. Gelatine wurde bewusst nicht als Zutat aufgenommen (Rezepte kommen ohne aus).

### 4.9 Gesamtbild: Schichten, Datenfluss, Rollen (Stand 28.10.2026)
**Schichten (von unten nach oben):**
1. *Daten* (nur Konstanten und Listen, keine Logik): `data_zutaten.js` (Zutaten, Gewürze), `data_bls.js` (Quelle der Nährwerte je Zutat), `data_sousvide.js`, `data_rezepte1.js` bis `data_rezepte5.js` (`SEED`, `SEED2`, `RETIRED`), `data_bausteine.js` (Meal-Prep), `data_saison.js` (Saisonkalender).
2. *Logik ohne DOM* (in Node testbar): `logic.js` (Konstanten, Nährwert- und Allergenrechnung, Ausschlüsse, Rezeptprüfung, Normalisierung), `logic2.js` (Datum, Chargen/FIFO, Einkauf, Tagebuch, Ziele, Fasten, Auto-Plan, Import/Export), `logic3.js` (Meal-Prep), `logic4.js` (Saison-Bewertung).
3. *Oberfläche*: `app_a.js` (Zustand, Speicher, Heute), `app_b.js` (Rezepte), `app_c.js` (Plan, Einkauf, Vorrat, Mehr), `app_d.js` (Tabs, `render()`, Blätter, Aktionstabellen `ACT`/`CHG`/`INP`), `app_f.js` (Saison-Oberfläche, nur Funktionsdeklarationen), `app_e.js` (Prep und `initRuntime()` als letzte Anweisung).
4. *Hülle*: `head.html` (CSS), `body.html` (Gerüst), `build.js` → `speiseplan.html` (Artifact-Variante) und `dist/` (PWA: `index.html`, `sw.js`, `manifest.json`, Icons). Der Service Worker cached die Dateien (Cache-Name wird bei jedem Build neu erzeugt); nach einem Upload auf GitHub deshalb hart neu laden.

**Datenfluss:** Klick oder Eingabe → `ACT`/`CHG`/`INP` ändert `state` → `save()` (600 ms Verzögerung) → `flush()` schreibt in `localStorage` (Schlüssel `speiseplan-xxl-v1`) und, nur in der Artifact-Variante mit Konto (`db` und `user`), zusätzlich in das Dokument `data/users/<id>/state` → `render()` baut die Ansicht neu. Beim Start: `loadLocal()` → `mergeState()` (bereinigt und ergänzt alte Stände) → `render()`; danach hat ein vorhandener Konto-Stand Vorrang. Die GitHub-Pages-Variante hat kein `window.claude` und speichert nur im Browser.

**Rollen und Regelebenen.** Die App hat keine Benutzerkonten und keine Rechteverwaltung: ein Browser = ein Nutzer, kein Abgleich zwischen Geräten außer über das Konto der Artifact-Variante oder Export/Import. Das Mehrbenutzer-Serverprojekt mit Rollen (Docker, Mealie) ist zurückgestellt (Abschnitt 1.11).
- *Mahlzeiten-Rolle eines Rezepts* (`meal`): Hauptgericht, Suppe/Eintopf, Salat/Bowl (zusammen `MAIN_MEALS`, zählen für Auto-Plan und Wochenanteile), Frühstück, Snack. Plan-Plätze: Frühstück, Mittag, Abend, Snack (`SLOTS`).
- *Rolle eines Bausteins* (`MODULE`): Protein (250 ml), Beilage (250 ml), Gemüse (250 ml), Sauce (125 ml), Booster (50 ml). Die Art `typ` (Dip, Grillsauce, Marinade, Dressing) und der Stil (`STIL_PASST`) bestimmen, was zusammenpasst. Aus Modulen entstehen Mahlzeiten (`mealBuild`, `suggestMeals`).
- *Lagerorte* (`inv.loc`): Gefrierer, Kühlschrank, Vorratsschrank.
- *Harte Regeln* (blockieren): bedingte Ausschlüsse (Ananas, Süßkartoffel, Bataten außerhalb von Frühstück, Snack und Dessert), gewählte Allergene und Ausschlüsse im Profil, Sicherheitsregeln (z. B. Kerntemperatur 70 °C bei Geflügel und Hackfleisch, Abschnitt 1.10). *Weiche Regeln* (Hinweise und Gewichte): Unverträglichkeiten, Histamin als Risikoprofil, Kalorien- und Eiweißziele, Saison.
- *Herkunft eines Rezepts* (`src`): „Perplexity-Recherche“, „Eigene Fassung (Claude)“ (nach Recherche oder nach allgemeinem Kochwissen, wenn die Quelle gesperrt war), eigene und importierte Rezepte (`user: true`, KI-Import `ai: true`). Dazu das feste Statusvokabular aus Abschnitt 1.7.
- *Beteiligte bei der Entwicklung:* der Auftraggeber entscheidet und liefert Links und Wünsche; Claude baut, prüft und führt dieses Protokoll; Perplexity (und bei gesperrten Links ChatGPT, siehe Anhang 13) liefert recherchierte Rezepte im Importformat.

**Sicherung und Wiederherstellung:**
- Quellcode, Tests und Protokoll: `speiseplan-projekt.zip` (enthält auch die fertige App in `dist/`). Ablegen, wie in Abschnitt 1.11 beschrieben, nach jeder Änderung die neue ZIP.
- Eigene Daten (Plan, Vorrat, Tagebuch, Gefrierer, Einstellungen, eigene Rezepte, Saison-Korrekturen): in der App unter Mehr → Daten und Sicherung exportieren (JSON, `exportState`); wiederherstellen mit dem Import (`importState`, wird bereinigt). Der Browser-Speicher allein ist keine Sicherung.
- Neu aufsetzen: `speiseplan-projekt.zip` entpacken, `node build.js`, `node make_zip.js` (führt alle Tests aus und packt neu), den Inhalt von `speiseplan-app.zip` auf GitHub hochladen.

### 4.10 Saison (Stand 28.10.2026)
`data_saison.js` hält 49 Zutaten mit einem 12-Monats-Muster (F frisch aus Deutschland, G Gewächshaus/Zucht, L Lager, I Import, – keine Frischware; Herkunft k/e, im Kalender mit * für eigene Einschätzung) und `SAISON_NEUTRAL` (bewusst nicht bewertet, mit Begründung). `logic4.js` rechnet den Saison-Score je Rezept und Monat (gewichtetes Mittel mit Wurzel der Grammzahl; frisch 1, Lager 0,75, Gewächshaus 0,5, Import und – 0; nur bei mindestens zwei bewertbaren Zutaten mit mindestens 15 % Mengenanteil; ab 0,8 „Saison“, ab 0,5 „teils“). `app_f.js` zeigt Plan → Saison (Monatswahl, Heatmap, Korrekturmodus), Abzeichen und Filter in den Rezepten, die Karte „Jetzt frisch“ auf Heute, Hinweise in Einkauf und Wochenplan, und Mehr → Saison. Der Auto-Plan nutzt Saison nur mit eingeschalteter Vorliebe (`opts.seasonFn`, weicher Abzug im Punktewert). Tests: `test_saison.js`, `smoke5.js`. Die Daten sind Orientierung und nicht gegen die Originalquellen geprüft.

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
| `node test_logic2.js` | 95 Prüfungen | bestanden |
| `node test_logic3.js` | 266 Prüfungen | bestanden |
| `node test_bls.js` | 9 Prüfungen | bestanden |
| `node smoke2.js` | 153 Prüfungen | bestanden |
| `node smoke3.js` | 91 Prüfungen | bestanden |
| `node smoke4.js` | 57 Prüfungen | bestanden |
| `node test_saison.js` | 57 Prüfungen | bestanden |
| `node smoke5.js` | 85 Prüfungen | bestanden |
| `node smoke6.js` | 81 Prüfungen | bestanden |
| `node test_look.js` | 99 Prüfungen | bestanden |
| `node test_quellen.js` | 21 Prüfungen | bestanden |

Weitere Hinweise zu den Tests:
- `smoke2.js` und `smoke3.js` laden `dist/index.html` in eine Node-VM mit einem kleinen DOM-Ersatz und klicken über die Aktionstabellen. Sie prüfen unter anderem, dass nirgends `undefined`, `NaN`, `[object` oder Ausschluss-Zutaten im HTML stehen.
- Beim Schreiben von Tests als Template-String Rückwärtsstriche in Regex doppelt schreiben (`\\[object`), sonst entstehen Syntaxfehler im Test statt in der App.
- `pw_prep.py` und `pw_lib.py` sind Browsertests (Playwright, Chromium, Fenster 400 px, hell und dunkel) mit Screenshots. Sie prüfen Konsolenfehler und Querscrollen. Chromium-Pfad im Beispiel: `/opt/pw-browsers/chromium`, Start mit `--no-sandbox`. Passe den Pfad an deine Umgebung an.
- Beim Test im echten Browser wurde ein Fehler gefunden, den die Node-Tests nicht sehen: das `hidden`-Attribut wurde von CSS übersteuert.

---

## 6. Ehrliche Grenzen des aktuellen Stands

- **Rezepte:** 258 von 240 geplanten sind mitgeliefert. Status „Entwurf, automatisch validiert“, **von der KI nicht gekocht**. Der Nutzer kann Rezepte entfernen und durch eigene oder importierte ersetzen.
- **Bausteine:** 71 Bausteine als eigene Grundzubereitungen, Status „Entwurf, automatisch validiert“. Ausbeute, Blockanzahl und Küchenzeiten sind Schätzungen. Nichts davon wurde gekocht oder eingefroren.
- **Nährwerte:** 158 von 208 Zutaten direkt nach BLS 4.0 (Stand 28.10.2026) (Quelle und Code je Zutat in `data_bls.js`, Auszug in `bls_auszug.json`, geprüft von `test_bls.js`), die übrigen 50 mit Näherungs-, abgeleiteten oder eigenen Werten (Kennung `m` in `data_bls.js`, je begründet). Das gilt je 100 g der Zutat; die Mahlzeitenwerte sind daraus gerechnet. Marken, Sorte, Reifegrad und Garverluste (Rezeptmengen sind Rohgewichte) lassen Abweichungen im Alltag zu. „Absolut“ ist nicht erreichbar, nachvollziehbar und belegt schon.
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

- **Quellenlinks ergänzt und geprüft (28.10.2026):** Alle 102 URLs aus `gesperrte_links.txt` wurden durch Webabrufe geprüft; fünf übermittelte Adressen gaben 404 zurück und wurden durch passende, überprüfte Zieladressen ersetzt. Die 25 Einträge ohne URL (Abschnitt F) bleiben bewusst ohne erfundene Originaladresse. Neue Datei `data_quellen.js` mit separat gekennzeichneten, anklickbaren Referenzen bei Rezepten und Meal-Prep-Bausteinen, ohne `src`, Zubereitungen, Zutaten, Status oder Nährwerte zu ändern. Detailbericht `QUELLEN_PRUEFBERICHT.md` und vollständige Tabelle `QUELLEN_PRUEFBERICHT.csv`, Tests in `test_quellen.js`. Die verknüpften fremden Rezepte sind **nicht** identisch mit den unabhängigen App-Fassungen und stellen keinen Herkunftsnachweis dar.
- **Zusätzliche Titel-Treffer für Linkliste F (28.10.2026):** Von 25 zuvor nur als Titel vorliegenden Einträgen konnten 22 gleich oder ausreichend ähnlich betitelte, tatsächlich aufgerufene Seiten erneut gefunden und als **separater möglicher Seiten-Treffer** in `data_quellen.js` ergänzt werden. Weder die 22 URLs noch ihre Inhalte dürfen als die im ursprünglichen Chat geteilten Links oder als ursprüngliche Rezeptinhalte ausgegeben werden; die Originaladressen bleiben bei allen 25 unbekannt. Drei Titel (r213, r216, r218) sind zu mehrdeutig und haben keinen Seiten-Treffer. Die App-Fassungen und alle bisherigen Rezeptdaten sind unverändert. Der vollständige Nachweis steht in `QUELLEN_PRUEFBERICHT.md` und `QUELLEN_PRUEFBERICHT.csv`.

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
- **Entscheidung 28.10.2026 (vierte): alle Rezepte wieder in der App.** Auf ausdrücklichen Wunsch („Ich brauche keine Prompts. Ich will meine fertige App mit Rezepten. Bitte alle Rezepte einbauen“) wurde die Beschränkung auf Perplexity-Rezepte zurückgenommen: In `data_rezepte5.js` ist der Zweig entfernt, der Rezepte ohne Perplexity-Quelle ausmusterte (Sicherung: `data_rezepte5.js.bak_perp`, im Dateikopf steht, wie man ihn wieder einschaltet). Ausgemustert bleiben nur r01 bis r76. Jedes Rezept zeigt seine Quelle im Feld `src` ehrlich an: „Perplexity-Recherche“ (18) oder „Eigene Fassung …“ (162). `smoke4.js` prüft jetzt 142 mitgelieferte Rezepte, 76 ausgemusterte und dass jede Quellenangabe mit „Perplexity“ oder „Eigene Fassung“ beginnt.
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
- **Charge 27 (28.10.2026, IDs r219 bis r223) und ein Dip-Baustein: 6 Links von Chefkoch und BBC Good Food:** Grünkohlchips, „Gesundes Bananenbrot“, „Bananenbrot ohne extra Fett und Zucker“, Bacon-Tomaten-Frischkäse-Häppchen, Nutty chicken satay strips (als Rezepte) und Lemon coriander hummus (als Baustein „Hummus mit Zitrone und Koriander“, jetzt 71 Bausteine, 6 Dips). Beide Seiten sind für die Werkzeuge gesperrt (`SITE_BLOCKED`); ein erneuter Abruf von je einem Link wurde mit derselben Meldung abgelehnt. Auf die Bitte, die Sperre aufzuheben, wurde erklärt, dass sie in den Abrufwerkzeugen liegt und nicht von Claude geändert werden kann; sie wird nicht umgangen. Es sind daher eigene Fassungen nach den Titeln und allgemeinem Kochwissen (Hinweis in `src`, beim Baustein im Tipp); die Zutaten der Originale sind unbekannt. Die beiden Bananenbrote sind bewusst verschieden (Haferflocken, Walnüsse und Honig gegenüber Quark, Mehl und Rosinen ohne Fett und Zuckerzusatz). Beide Brote liefern allein nur 17 bis 19 g Eiweiß je Portion; deshalb gehören 125 g Skyr zur Portion und sind eingerechnet (32 g und 30,1 g Eiweiß, 505 und 450 kcal). Das Satay-Gericht hat 568 kcal und 46,1 g Eiweiß und enthält Erdnuss, Soja und Gluten (Hinweis im Tipp). Geprüft mit `check_recipes.js`: 0 Fehler, 2 Warnungen (Grünkohlchips 136 kcal und Häppchen 234 kcal liegen als Snacks unter dem üblichen Bereich, gewollt). Rohdaten: `chargen/recherche_24_chefkoch_bbc.json`, Generator `_gen24.js`, Einbau `_mk_charge24.js`, Baustein `_add_hummus_kor.js` (Sicherung `data_bausteine.js.bak_hummuskor`). Testzähler: volle Bibliothek 218, ausgelieferte 147, 71 Bausteine, `test_logic3.js` 266 Prüfungen.
- **Prüfung der von ChatGPT ergänzten Quellenlinks (28.10.2026):** Der Auftraggeber hat ein Projekt-ZIP zurückgeschickt, in dem ChatGPT `data_quellen.js`, `test_quellen.js` und `QUELLEN_PRUEFBERICHT.md/.csv` ergänzt und `app_b.js`, `app_e.js`, `build.js`, `make_zip.js`, `PROTOKOLL.template.md` angepasst hat. Alle anderen Dateien waren unverändert gegenüber dem vorherigen Stand (Dateivergleich). Claude hat übernommen und geprüft: (1) Zuordnung ID zu Link: alle 127 betroffenen Rezepte und Bausteine existieren, Name und Linkbezeichnung passen bei allen zusammen (Stichprobe nicht nötig, alle Einträge einzeln verglichen); r84 (Tajine) und r91 (Satay) sind ausdrücklich nur thematisch verwandt. (2) 97 der 102 Links aus `gesperrte_links.txt` sind zugeordnet; die übrigen fünf sind die laut ChatGPT nicht erreichbaren und durch andere Adressen ersetzten Links (r219 bis r223). (3) Die Allrecipes-Links der Garnelengerichte r155 bis r159 fehlen in der Linkliste und haben daher keine Referenz. (4) `quellenBlock()` rendert nur `https://`-Adressen, escaped alle Texte und setzt `rel="noopener noreferrer"`. Rezepte, Zutaten, Nährwerte und `src` blieben unverändert. **Nicht geprüft werden konnte, ob die Links tatsächlich erreichbar sind und zum Titel passen:** Chefkoch, BBC Good Food, Allrecipes und Serious Eats sind für Claudes Werkzeuge gesperrt. Die Aussage „aufgerufen und Titel geprüft“ stammt von ChatGPT und ist von Claude nicht nachprüfbar. Die 22 „Titel-Treffer“ (Art `F_TREFFER`) sind ausdrücklich keine Wiederherstellung der Originallinks. Der geänderte BBC-Link bei r223 (`nutty-chicken-sate-strips`) ist ebenfalls ungeprüft.
- **Charge 28 (28.10.2026, IDs r224 bis r233): 10 Frühstücksrezepte** nach Titeln von Chefkoch (Overnight Oats, Bananen-Apfel-Zimt-Porridge, Frühstücks-Muffins, Rührei aus dem Backofen, Haferflockenbrot, Bestes Bircher Müsli, Erdbeer-Bananen-Smoothie; für diese sieben nannte der Auftraggeber nur den Titel, „Direktlink nicht verifiziert“) und BBC Good Food (american-pancakes, healthy-shakshuka, ham-mushroom-spinach-frittata; Links genannt). Beide Seiten sind gesperrt und wurden nicht gelesen; es sind eigene Fassungen nach den Titeln (Hinweis im `src`), die Zutaten der Originale sind unbekannt. Alle zehn liegen bei 326 bis 506 kcal und 23,7 bis 33,9 g Eiweiß je Portion; das Eiweiß kommt aus Skyr, Quark, Hüttenkäse und Ei. Die Pancakes, das Bircher Müsli, der Smoothie und die Oats enthalten dafür bewusst Skyr oder Quark. „Overnight Oats mit Skyr, Zimt und Vanille“ heißt so, weil der alte, ausgemusterte Eintrag r65 bereits „Overnight Oats mit Skyr und Beeren“ hieß (Namens-Eindeutigkeit). Das Rührei wird im Ofen gegart, bis es durchgehend 70 °C hat; Shakshuka und Frittata haben Garhinweise für die Eier. Das Haferflockenbrot ist ohne Hefe und ohne Mehl geschrieben, weil das Original unbekannt ist. Die Garnelen- und Chinagerichte aus früheren Chargen blieben unverändert. Referenzen in `data_quellen.js`: drei BBC-Links als Art `B` (ungeprüft, Hinweis in der Notiz), sieben Chefkoch-Titel als Art `F` ohne Link. Geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Rohdaten: `chargen/recherche_25_fruehstueck.json`, Generator `_gen25.js`, Einbau `_mk_charge25.js`. Testzähler: volle Bibliothek 228, ausgelieferte 157, `test_quellen.js` 21 Prüfungen.
- **Charge 29 (28.10.2026, IDs r234 bis r243): 10 Suppen** nach Titeln von Chefkoch (Käse-Lauch-Suppe mit Hackfleisch und Hühnersuppe mit Hähnchenschenkel mit genannten Links; Pho Ga, Kräftige Rindfleischsuppe, Einfache Gemüsesuppe, Kartoffel-Rindfleisch-Suppe und Roros Grießnockerlsuppe nur mit Titel, „Direktlink nicht verifiziert“) und BBC Good Food (chicken-noodle-soup mit Link; Spiced Carrot & Lentil Soup und Tomato Soup nur mit Titel). Beide Seiten sind gesperrt und wurden nicht gelesen; es sind eigene Fassungen nach den Titeln (Hinweis im `src`), die Zutaten der Originale sind unbekannt. Alle zehn liegen bei 364 bis 562 kcal und 32 bis 42 g Eiweiß je Portion (Suppe/Eintopf zählt als Hauptmahlzeit). Dafür sind einige Fassungen bewusst angepasst: Einfache Gemüsesuppe mit weißen Bohnen und Linsen, Möhren-Linsen-Suppe mit 125 g Skyr je Portion, Tomatensuppe mit weißen Bohnen, Skyr und Käse-Croûtons, Grießnockerlsuppe mit Quark im Teig und Hähnchenstreifen (die klassische Grießnockerlsuppe ist eine leichte Vorspeise; das steht im Tipp). Die Hühnersuppe ist mit entbeinten Keulen und fertiger Brühe gerechnet, weil die Nährwerte der Zutat „Hähnchenkeule ohne Haut und Knochen“ vorliegen; das Original kocht vermutlich mit Knochen. Fleisch und Geflügel haben in den Schritten die 70-°C-Angabe, die Suppen den Aufwärmhinweis (mindestens 70 °C). Neue Zutat **Weizengrieß** (BLS 4.0, C218000 „Weichweizen Grieß“, 355 kcal, 10,7 g Eiweiß, 74 g Kohlenhydrate, 0,9 g Fett je 100 g, Gluten) für die Nockerl; Eintrag in `bls_auszug.json` ergänzt, Sicherungen `*.bak_gries`, Skript `_add_gries.js`. Geprüft mit `check_recipes.js` (0 Fehler, 0 Warnungen). Rohdaten: `chargen/recherche_26_suppen.json`, Generator `_gen26.js`, Einbau `_mk_charge26.js`. Referenzen in `data_quellen.js`: **nachträglich (28.10.2026) hat der Auftraggeber für alle zehn Suppen die Links geschickt**; sie sind als Art `A` (Chefkoch) bzw. `B` (BBC) eingetragen, im `src` der Rezepte steht jetzt „nannte den Link …“ statt „nur den Titel“. Die Links sind von Claude weder aufgerufen noch geprüft (Seiten gesperrt). Das Rezept selbst bleibt eine eigene Fassung nach dem Titel; der Generator `_gen26.js` enthält noch die ältere Quellenformulierung, maßgeblich sind die Daten in `data_rezepte4.js`. Testzähler `test_quellen.js`: 150 Einträge mit Link, 32 ohne. Testzähler: volle Bibliothek 251, ausgelieferte 180, `test_quellen.js` 21 Prüfungen.
- **Charge 30 (28.10.2026, IDs r244 bis r252): 9 Eintöpfe und Suppen** nach zehn übermittelten Links (sieben Chefkoch, drei BBC Good Food). Eigene Fassungen von Claude, nicht von Perplexity: Blumenkohleintopf mit Putenhack und Kartoffeln, Low-Carb-Bauerntopf mit Rinderhack und Weißkohl, Deftige Erbsensuppe mit Kochschinken, Szegediner Gulasch mit Sauerkraut und Kartoffeln, Grüner-Bohnen-Eintopf mit Schweineschulter, Burgunder-Fleischtopf mit Rind, Schwein und Kartoffeln, Kartoffel-Lauch-Suppe mit Hack und Käse, Winter-Rindereintopf mit Käse-Muffin-Haube (BBC „muffin-topped-winter-beef-stew“) und Rind-Stout-Eintopf mit Karotten (BBC „beef-stout-stew-carrots“). **Übersprungen:** BBC „beef-vegetable-casserole“ gibt es bereits als r121 (Link dort schon hinterlegt), es wurde kein zweites Rezept angelegt. Alle Seiten sind für Claudes Werkzeuge gesperrt und wurden nicht gelesen; die Zutaten der Originale sind unbekannt, Mengen und Zeiten sind Richtwerte, nicht getestet. Je Portion 419 bis 508 kcal und 37 bis 45 g Eiweiß (Zielbereich 400 bis 650 kcal, mindestens 30 g Eiweiß). Fleisch wird bei allen Rezepten bis mindestens 70 °C im Kern gegart. Abweichungen vom vermutlichen Original stehen im Tipp: Erbsensuppe mit magerem Kochschinken statt Wurst, Bohnenkraut durch Thymian und Majoran ersetzt (Bohnenkraut ist in der App nicht hinterlegt), Burgunder-Topf gemischt aus Rind und Schwein, um ihn vom bestehenden Boeuf Bourguignon abzugrenzen. Rotwein und Stout enthalten Alkohol, der bei der Garzeit nur teilweise ausgekocht wird; das steht im Tipp. Neue Zutaten aus der BLS 4.0: **Erbsen, getrocknet** (G760400 „Erbse reif“, 311 kcal, 23,0 g Eiweiß, 40,5 g Kohlenhydrate, 1,4 g Fett; Energie mit Ballaststoffanteil geprüft) und **Rotwein, trocken** (P2A3000). Die Links stehen in den Externen Rezeptreferenzen (Art A bzw. B, als „nicht geprüft“ gekennzeichnet) und in `gesperrte_links.txt` (Chefkoch 36, BBC 41). Testzähler: volle Bibliothek 247, ausgelieferte 176, `test_quellen.js` 146 Einträge mit Link, 32 ohne, `test_bls.js` 9 Prüfungen.
- **Charge 31 (28.10.2026, IDs r253 bis r256): 4 weitere Eintöpfe** nach vier BBC-Good-Food-Links: Bohnen-Halloumi-Eintopf mit Paprika und Spinat („bean-halloumi-stew“, Adresse bbcgoodfoodme.com), Geschmorter Rinder-Topf mit Zwiebeln, Paprika und Tomaten („braised-beef-onepot“), Slow-Cooker-Rindereintopf mit Champignons und Pastinaken („slow-cooker-beef-stew“, mit Hinweis zur Mindesttemperatur im Slow Cooker) und Tomaten-Paprika-Bohnentopf mit Ei und Feta („tomato-pepper-bean-one-pot“, Eier bis zum gestockten Eiweiß, für Risikogruppen bis zum festen Eigelb). Eigene Fassungen von Claude, nicht von Perplexity; die Seite ist gesperrt und wurde nicht gelesen, Zutaten der Originale unbekannt. Je Portion 411 bis 543 kcal und 32 bis 43 g Eiweiß. **Übersprungen:** Der Auftraggeber schickte auch die zehn Links aus Charge 30 erneut; sie sind bereits als r244 bis r252 (und r121) in der App. „spicy-root-lentil-casserole“ gibt es schon als r197. Testzähler: volle Bibliothek 251, ausgelieferte 180, `test_quellen.js` 150 Einträge mit Link.
- **Charge 32 und drei Pakete: Ausrüstung, Darstellung, Dessert (28.10.2026):** (1) Souper-Cubes-Ausrüstung editierbar (4.11). (2) Einstellungsbereich Darstellung mit Farbmodus, sechs Akzenten, Schrift, Dichte und PC-Layout mit Seitenleiste und Zwei-Spalten-Ansicht (4.12), neue Datei `look.js`, Tests `test_look.js` (99) und `smoke6.js` (69). (3) Kategorie Dessert mit Plan-Platz, Tagebuch-Platz und Auto-Plan-Unterstützung (4.13) sowie 12 neue Desserts r257 bis r268 (Skyr-Mousse mit Beeren, Schoko-Quark-Creme, Vanillepudding mit Quark, Bratapfel mit Skyr, Beerengrütze mit Vanille-Skyr, Frozen Joghurt mit Beeren, Schoko-Bananen-Eis mit Skyr, Chia-Pudding mit Mango und Skyr, Quarkauflauf mit Birnen, Leichter Kaiserschmarrn mit Skyr, Zitronen-Ricotta-Creme mit Erdbeeren, Schoko-Quark-Muffins mit Haferflocken), alle eigene Fassungen ohne externe Vorlage, Zeiten Richtwerte, nicht praktisch getestet. Neue Zutaten mit BLS-4.0-Werten: Kakao (S711000) und Chiasamen (H480100). Zähler: Bibliothek 263, mitgelieferte Rezepte 192, „Eigene Fassung …“ 174 (vorher 162), Quellen-Links unverändert 150 verlinkt und 160 Schlüssel. Die Einstellungen der Darstellung sind je Gerät und nicht in der Sicherung.
- **Charge 33 (28.10.2026, IDs r269 bis r278): 10 Desserts** nach Titeln von zehn übermittelten Links, davon vier Chefkoch (Dessert mit isländischem Frischkäse und Weintrauben, Himbeerdessert mit Spekulatius, Himbeerkuchen, Birnen in Rotwein an Vanilleeis) und sechs BBC (Best apple crumble, Luscious lemon baked cheesecake, Lemon syllabub, Best-ever chocolate brownies, No-bake strawberry unter bbc.co.uk/food, No-cook chocolate tart). **Alle zehn Seiten waren gesperrt (SITE_BLOCKED), keine wurde gelesen oder umgangen.** Die App-Fassungen sind eigene Rezepte nach dem Titel und allgemeinem Kochwissen, an die Vorgaben angepasst (120 bis 350 kcal, mindestens 10 g Eiweiß je Portion, daher Quark und Skyr statt Sahne-Frischkäse-Mengen und viel Zucker): Skyr-Dessert mit Weintrauben und Mandeln, Himbeer-Spekulatius-Schichtdessert, Himbeer-Quarkkuchen mit Haferflocken, Rotwein-Birnen mit Vanille-Skyr und Vanilleeis, Apfel-Crumble mit Haferflocken und Vanille-Skyr, Zitronen-Käsekuchen mit Quark und Haferboden, Zitronen-Syllabub mit Skyr, Saftige Schoko-Brownies mit Quark, Erdbeer-Cheesecake im Glas ohne Backen, Schoko-Tarte ohne Backen mit Quarkcreme. Sie weichen von den Originalen ab und sind **nicht praktisch getestet**. Die Links hängen als „Übermittelter Link, für Claude gesperrt, nicht gelesen“ an den Rezepten (Art A Chefkoch, Art B BBC) und stehen in `gesperrte_links.txt` (Abschnitte A und B). Fünf neue Zutaten aus BLS 4.0: Himbeeren (F302100), Weintrauben (F310100), Spekulatius (D730600), Zartbitterschokolade (S560000), Vanilleeis (S241000); Himbeeren und Weintrauben stehen auch im Saisonkalender (47 bewertete Zutaten, beide mit eigener Einschätzung, jetzt 10 Einträge mit Herkunft e). Der Rotwein in den Rotwein-Birnen ist auf 150 g begrenzt, weil die Energie-Plausibilität den Alkoholanteil sonst nicht mehr deckt; Syllabub und Rotwein-Birnen enthalten Alkohol (Hinweis im Tipp). Zähler: Bibliothek 273, ausgelieferte Rezepte 202, „Eigene Fassung …“ 184, Quellen-Links 160 verlinkt und 170 Schlüssel.
- **Charge 34 (28.10.2026, IDs r279 bis r283): 5 Kontaktgrill-Gerichte** nach fünf übermittelten Links. **Gelesen** wurden drei Seiten von lecker.de: „Surf & Turf“-Gemüsepuffer, Blitzschneller Portobello-Burger, Gegrilltes Steak mit Bandnudeln und Pfefferrahmsoße. Sie sind in eigenen Worten und für die App angepasst (die Originale liegen bei 771 kcal und 35 g Eiweiß, 592 kcal und 17 g Eiweiß bzw. deutlich größeren Fleisch- und Nudelmengen): Gemüsepuffer vom Kontaktgrill mit Räucherlachs und Schinken (Skyr statt Schmand, ohne Parmaschinken), Pilz-Burger vom Kontaktgrill mit Halloumi und Ajvar-Creme (Halloumi statt Cheddar, Skyr-Creme statt Salatcreme, damit die 30 g Eiweiß erreicht werden), Rumpsteak vom Kontaktgrill mit Vollkornbandnudeln und Pfefferrahmsauce (150 g Steak und 60 g Nudeln roh je Portion). **Nicht gelesen** wurden essen-und-trinken.de (robots.txt verbietet den Abruf; der Link führt zu einem Sammelartikel, gesucht war das Panini mit Tomate und Rucola) und Serious Eats (gesperrt, Cuban Sandwich): beide sind eigene Fassungen nach dem Titel (Panini mit Mozzarella, Tomate, Rucola und Schinken; Cuban Sandwich mit Schweinefilet und Gouda vom Kontaktgrill), die Zutaten der Originale sind unbekannt. Alle fünf sind **nicht praktisch getestet**, Zeiten sind Richtwerte. Die Quellenhinweise tragen Art G (lecker.de, gelesen), H (essen-und-trinken.de) und D (Serious Eats); die beiden nicht gelesenen Links stehen in `gesperrte_links.txt` (Abschnitte D und G). Keine neuen Zutaten. Zähler: Bibliothek 278, ausgelieferte Rezepte 207, „Eigene Fassung …“ 189, Quellen-Links 165 verlinkt und 175 Schlüssel.
- **Charge 35 (28.10.2026, IDs r284 bis r292): 9 Smoothies** (Kategorie Frühstück, je 1 Portion, 283 bis 469 kcal, 21 bis 28 g Eiweiß) nach neun übermittelten Links. **Gelesen** wurden zwei Seiten von lecker.de (Erdbeer-Bananen-Smoothie mit Milch; Erdbeer-Smoothie mit Minze, Soja-Joghurt und Mandelmilch): Erdbeer-Bananen-Smoothie mit Milch und Magerquark (Quark ergänzt, damit das Eiweiß statt 9 g bei 23 g liegt) und Erdbeer-Minze-Smoothie mit Soja und Tofu, vegan (Sojajoghurt gibt es im Zutatenstamm nicht, Tofu übernimmt Eiweiß und Cremigkeit; die Mengenangabe „200 EL“ auf der Originalseite ist offensichtlich ein Fehler und wurde nicht übernommen). **Nicht gelesen** wurden die sieben BBC-Good-Food-Seiten (gesperrt): Two-minute breakfast smoothie, Breakfast smoothie, Strawberry smoothie, Sunshine smoothie, Peanut butter smoothie, Mango banana smoothie, Kale smoothie. Sie sind eigene Fassungen nach dem Titel, die Zutaten der Originale sind unbekannt (Zwei-Minuten-Frühstücks-Smoothie mit Banane und Haferflocken, Frühstücks-Smoothie mit Beeren, Banane und Chia, Erdbeer-Joghurt-Smoothie mit Orange und Skyr, Sonnenschein-Smoothie mit Mango, Orange und Karotte, Erdnuss-Bananen-Smoothie mit Skyr, Mango-Bananen-Smoothie mit Skyr und Limette, Grünkohl-Apfel-Smoothie mit Skyr und Ingwer). Der Zwei-Minuten-Smoothie nutzt Magerquark und Walnüsse, weil er dem vorhandenen Erdbeer-Bananen-Smoothie mit Skyr sonst als Dublette aufgefallen wäre. Alle neun sind **nicht praktisch getestet**; Smoothies sind als Frühstück eingeordnet (nicht Snack), weil sie über 20 g Eiweiß liefern. Keine neuen Zutaten. Zähler: Bibliothek 287, ausgelieferte Rezepte 216, „Eigene Fassung …“ 198, Quellen-Links 174 verlinkt und 184 Schlüssel.
- **Ausschlussregel gelockert (28.10.2026):** Auf Zuruf des Auftraggebers („Ananas ist erlaubt. Nur nicht als Hauptgericht. Das gilt für Süßkartoffeln auch“) sind Ananas, Süßkartoffel und Batate nur noch außerhalb der Hauptmahlzeiten erlaubt (Frühstück, Snack, Dessert). In Hauptgericht, Suppe/Eintopf, Salat/Bowl und den Meal-Prep-Bausteinen bleiben sie gesperrt; ich lege „nicht als Hauptgericht“ so aus, dass Suppe/Eintopf und Salat/Bowl als Hauptmahlzeiten mitzählen (bei anderer Absicht ist es eine Änderung in `isHauptRezept()`/`MAIN_MEALS`). Umsetzung: `FIXE_AUSSCHLUESSE`/`EXCL_KEYS` jetzt leer, `FIXE_BEDINGT`/`EXCL_HAUPT_KEYS` mit den drei Zutaten, Parameter `haupt` in `exclusionHits()`; Hinweis in Prüfbericht, Profil (Mehr), KI-Prompt und Perplexity-Prompt angepasst. Neu: Zutat Ananas (BLS F501100) im Stamm, in der Saison als neutrale Zutat (jetzt 10 neutrale, 47 bewertete). In freien Texten ohne Rezeptkategorie (Gefrierer-Namen, Schnelleinträge, Einkaufsextras, Vorratssuche) greifen nur die eigenen Ausschlüsse des Profils. Tests angepasst: test_logic2 95 Prüfungen (Hauptgericht blockiert, Frühstück/Dessert erlaubt, Salat/Bowl blockiert), die Smoke-Tests prüfen die Abweisung in freien Texten jetzt mit eigenen Profil-Ausschlüssen (Rosenkohl, Okra). Es gibt noch kein Rezept mit Ananas oder Süßkartoffel.
- **Charge 36 (28.10.2026, IDs r293 bis r299): 7 Desserts** nach sieben BBC-Links. **Gelesen** wurden drei Seiten von bbcgoodfoodme.com (die Schwesterseite ist abrufbar, bbcgoodfood.com nicht): Ultimate crème brûlée (Original: 426 ml Sahne und 5 Eigelb für 4 Portionen), Easy cornflake tart (Mürbeteig, Sirup, Marmelade), Chocolate brownie cake (100 g Butter, 250 g Zucker, 100 g Mehl für 8 Portionen). Sie sind in eigenen Worten und für die App angepasst: Crème brûlée mit weniger Sahne (ganze Eier, mehr Milch), Cornflakes-Tarte mit Himbeer-Quark und Haferboden, Brownie-Kuchen mit weißen Bohnen und Quark. **Nicht gelesen** wurden vier Seiten von bbcgoodfood.com (gesperrt): Easy chocolate molten cakes, Ultimate lemon meringue pie, Instant frozen berry yogurt, Chocolate cheesecake. Sie sind eigene Fassungen nach dem Titel, die Zutaten der Originale sind unbekannt: Schoko-Lavakuchen mit Quark (der flüssige Kern ist nicht durchgegart, Hinweis im Tipp), Zitronen-Baiser-Becher mit Skyr (Baiser als Schweizer Meringue auf 70 °C erwärmt), Sofort-Beereneis aus dem Mixer mit Skyr, Schoko-Käsekuchen mit Haferboden. Alle sieben liegen bei 174 bis 244 kcal und 10,5 bis 15 g Eiweiß je Portion und sind **nicht praktisch getestet**. Neue Zutat aus BLS 4.0: Cornflakes (gesüßt, C515000). Zähler: Bibliothek 294, ausgelieferte Rezepte 223, „Eigene Fassung …“ 205, Quellen-Links 181 verlinkt und 191 Schlüssel. Hinweis: Das Rezept r253 (Bohnen-Halloumi-Eintopf, Adresse bbcgoodfoodme.com/recipes/bean-halloumi-stew/) wurde früher als nicht gelesen geführt; die Seite ist abrufbar und könnte nachträglich mit dem Original abgeglichen werden.
- **120-ml-Blockgröße (28.10.2026):** Der Auftraggeber meldete einen falschen Souper-Cubes-Bestand und bat um den 120-ml-Behälter. `BLOCK_ML` ist jetzt 50, 120, 125, 250, 500; `KAMMERN`, `EQUIP_STD` (120 ml: 0 Formen, 6 Kammern), `equipClean`, Gefrier-Validierung (`freeze.block`), Import-Prüfung, Vorrats- und Rezepteditor-Auswahl, Ausrüstungstabelle, Prep-Rundenrechnung (`prepRoundsFor`, `sessionYield`) und Sitzungsübersicht kennen die Größe. Rezepte und Bausteine bleiben bei 125 ml (Soßen); 120 ml ist für eigene Gefrierer-Einträge und als Rezeptblock wählbar. Tests: `smoke6.js` 76 Prüfungen (zehn Eingabefelder, Summen mit 120 ml, mergeState, Rezeptblock 120 gültig und 77 ungültig). Die 125-ml-Zahlen des Auftraggebers wurden nicht geändert; falls der alte 125-ml-Bestand falsch war, ist er unter Mehr → Ausrüstung anzupassen.
- **Smoothie-Schnellfilter (28.10.2026):** In der Rezeptliste gibt es neben den Mahlzeit-Chips den Chip „Smoothie (Anzahl)“. Es ist keine neue Kategorie: `isSmoothie(r)` trifft Rezepte mit `meal === 'Frühstück'` und „Smoothie“ im Namen (`SMOOTHIE_FILTER` in `app_b.js`, Wert in `ui.f.meal`). Die Smoothies bleiben Frühstück und im Frühstück-Filter sowie im Plan enthalten (Prüfwerte unverändert, mindestens 20 g Eiweiß). Aktuell 10 Treffer. Test: `smoke6.js` (81 Prüfungen). Die ältere Datei `smoke.js` gehört nicht zum Lieferumfang.
- **Charge 37 (28.10.2026, IDs r300 bis r314): 15 chinesische Desserts** nach 15 Links. **Gelesen** wurden sechs Seiten von foodnetwork.com (Quellenart `I`): Tangyuan mit Erdnussfüllung (r306), Schwarze Sesam-Süßsuppe (r307), Fa Gao (r308), Hong Dou Tang (r309), Orange-Grand-Marnier-Sauce für Nian Gao (r310), Ba Bao Fan (r311); sie sind in eigenen Worten und für die App angepasst (mehr Eiweiß, weniger Zucker und Fett, Sojadrink statt Wasser, Skyr oder Quark ergänzt; Grand Marnier weggelassen). **Nicht gelesen** wurden neun Seiten: essen-und-trinken.de (robots.txt untersagt automatisches Abrufen): Mangopudding (r300), Glückskekse (r301), Chinesische Pfannkuchen mit Pflaumensauce (r314); bbcgoodfood.com (gesperrt): Lychee-Ginger-Sorbet (r302), Tanghulu (r303); seriouseats.com (gesperrt): Mondkuchen mit Walnuss und roter Bohne (r304), Ingwer-Milchpudding (r305); chefkoch.de (gesperrt): Chinesisches flambiertes Eis (r312, als gebackenes Skyr-Eis im Airfryer, nicht frittiert und nicht flambiert), Gedämpfte Dattelklößchen (r313). Diese neun sind eigene Fassungen nach dem Titel, die Zutaten der Originale sind unbekannt. Alle 15 liegen bei 149 bis 329 kcal und 10,8 bis 19,1 g Eiweiß je Portion, `check_recipes.js` 0 Fehler und 0 Warnungen, **nicht praktisch getestet**. Heikle Punkte stehen in den Tipps: heißer Zucker bei Tanghulu (150 bis 155 °C, Verbrennungsgefahr), Ingwer-Milchpudding gerinnt mit 1,5-%-Milch unsicher, Ei im Airfryer-Eis nur kurz erhitzt. **Rote Bohnen:** Azukibohnen stehen nicht in der BLS; Mondkuchen, Hong Dou Tang und Ba Bao Fan sind mit Kidneybohnen aus der Dose gerechnet und im Tipp als Abwandlung gekennzeichnet. Klebreismehl und Klebreis haben keinen eigenen BLS-Eintrag und werden mit Reismehl (C453000) bzw. Sushireis gerechnet. **Neue Zutaten aus BLS 4.0:** Reismehl (C453000), Datteln getrocknet (F504400), Litschi roh (F515100, in `SAISON_NEUTRAL`, jetzt 11 neutrale), Eiklar (E113100). Die Liste `gesperrte_links.txt` enthält die neun gesperrten Links. Zähler: Bibliothek 309, ausgelieferte Rezepte 238, „Eigene Fassung …“ 220, Quellen-Links 196 verlinkt und 206 Schlüssel, Dessert 44 Rezepte.
- **Charge 38 (28.10.2026, IDs r315 bis r320): 6 Smoothies (Frühstück)** nach sieben BBC-Links (Tropical breakfast smoothie lag doppelt vor). **Gelesen** wurde bbcgoodfoodme.com: Tropical breakfast smoothie (Passionsfrucht, Banane, Mango, Orangensaft; Original für drei Gläser ohne Eiweißquelle, hier auf ein Glas umgerechnet und mit Skyr ergänzt, r316). **Nicht gelesen** (bbcgoodfood.com gesperrt): Avocado & strawberry smoothie (r315), Blueberry smoothie (r317), Kiwi fruit smoothie (r318), Creamy mango & coconut smoothie (r319), Immune-boosting smoothie (r320); eigene Fassungen nach dem Titel, die Zutaten der Originale sind unbekannt. Alle sechs liegen bei 271 bis 370 kcal und 23,2 bis 28,5 g Eiweiß je Glas, `check_recipes.js` 0 Fehler und 0 Warnungen, **nicht praktisch getestet**. Der Tipp zum Immun-Smoothie nennt ausdrücklich keinen Nutzen für die Abwehr. **Neue Zutaten aus BLS 4.0:** Passionsfrucht (F508100), Kiwi (F514100), Heidelbeeren (F304100), Orangensaft (F603600). Saison: Heidelbeeren bewertet (Juni bis Oktober frisch, eigene Einschätzung, jetzt 48 bewertete Zutaten), Kiwi, Passionsfrucht und Orangensaft in `SAISON_NEUTRAL` (jetzt 14 neutrale). Der Schnellfilter „Smoothie“ zeigt 16 Treffer. Zähler: Bibliothek 315, ausgelieferte Rezepte 244, „Eigene Fassung …“ 226, Quellen-Links 203 verlinkt und 212 Schlüssel, Dessert unverändert 44.
- **Charge 39 (28.10.2026, IDs r321 bis r330): 10 Salate (Kategorie Salat/Bowl)** nach zehn Links. **Gelesen** wurde eine Seite von bbcgoodfoodme.com: Halloumi, Karotten & Orangen-Salat (r328; Original für vier Portionen mit 225 g Halloumi und ohne weitere Eiweißquelle, hier für zwei Portionen mit 180 g Halloumi und Kichererbsen, damit mindestens 30 g Eiweiß erreicht werden). **Nicht gelesen** wurden neun Seiten: essen-und-trinken.de (robots.txt untersagt automatisches Abrufen): Salat mit Honig-Senf-Dressing (r321, mit gebratener Hähnchenbrust ergänzt); bbcgoodfood.com (gesperrt): Chicken Caesar salad (r322), Greek salad (r323), Chicken satay salad (r324), Classic potato salad (r325), 10-minute couscous salad (r326), Quinoa salad with grilled halloumi (r327), Epic summer salad (r329, Inhalt des Originals unbekannt, daher ein Sommersalat mit Lachs als eigene Abwandlung), Salade niçoise (r330). Diese neun sind eigene Fassungen nach dem Titel, die Zutaten der Originale sind unbekannt. Alle zehn sind für zwei Portionen gerechnet und liegen bei 433 bis 608 kcal und 31 bis 51,5 g Eiweiß je Portion, `check_recipes.js` 0 Fehler und 0 Warnungen, **nicht praktisch getestet**. Gruppen: Geflügel 3 (r321, r322, r324), Schwein 1 (r325, Kochschinken), Fisch 2 (r329, r330), Vegetarisch 4 (r323, r326, r327, r328). Skyr ersetzt bei Caesar, Griechischem, Couscous- und Quinoa-Salat sowie Kartoffelsalat das klassische Öl- oder Mayonnaise-Dressing. Salate sind nicht einfrierbar (Gefrierblock „nein“). Keine neuen Zutaten. Zähler: Bibliothek 325, ausgelieferte Rezepte 254, „Eigene Fassung …“ 236, Quellen-Links 213 verlinkt und 222 Schlüssel; `gesperrte_links.txt`: BBC 79, essen-und-trinken 5.
- **Charge 40 (28.10.2026, IDs r331 bis r334): 4 neue Rezepte aus einer Zusendung mit 10 Links.** Sechs der zehn Links waren schon umgesetzt: Grünkohlchips (r219), Gesundes Bananenbrot (r220), Bananenbrot ohne extra Fett und Zucker (r221), Bacon-Tomaten-Frischkäse-Häppchen (r222), Nutty chicken satay strips (r223; der gesendete Link „…satay-strips“ war nicht erreichbar, die App verweist auf den verifizierten Link „…sate-strips“, bei Grünkohlchips ebenso auf den verifizierten Chefkoch-Link) und der Hummus-Baustein b71. Es wurden keine Dubletten angelegt und die vorhandenen Verweise nicht verändert. **Neu:** Vegetarische Pilz-Wurstrollen (r331, **gelesen** auf bbcgoodfoodme.com; Original mit fertigem Blätterteig und Kastanien, beides nicht im Zutatenstamm, daher Quark-Öl-Teig ohne Kastanien, Gouda light und Quark für das Eiweiß), Regenbogen-Obstspieße mit Skyr-Vanille-Dip (r332, Snack, enthält Ananas), Crunchy Granola mit Skyr, Beeren und Kirschen (r333, als Frühstück eingeordnet, nicht als Snack), Hähnchen-Nachos (r334, Hauptgericht); die drei letzten von bbcgoodfood.com (gesperrt, nicht gelesen), also eigene Fassungen nach dem Titel. `check_recipes.js` 0 Fehler und 0 Warnungen, **nicht praktisch getestet**. **Neue Zutat aus BLS 4.0:** Süßkirschen (F211100), in der Saison mit „eigene Einschätzung“ (Juni bis August frisch, jetzt 49 bewertete Zutaten). Tests angepasst: Ananas ist in Obstspießen (Snack) erlaubt, die Prüfung „keine festen Ausschlüsse“ blockiert sie nur noch in Hauptmahlzeiten, und die Saison-Prüfung erkennt Süßkartoffel nicht mehr am Teilwort „suess“. Zähler: Bibliothek 329, ausgelieferte Rezepte 258, „Eigene Fassung …“ 240, Quellen-Links 217 verlinkt und 226 Schlüssel; `gesperrte_links.txt`: BBC 82.
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

**logic.js**: `equipClean`, `equipNow`, `kap`, `kapN`, `equipTotals`, `I`, `esc`, `norm`, `round1`, `exclusionHits`, `recipeTexts`, `isHauptRezept`, `recipeExclusionHits`, `calcRecipe`, `derive`, `conflicts`, `mealType`, `slotFits`, `energyCheck`, `ingredientEnergyIssues`, `validateRecipe`, `runQA`, `distribution`, `dayTotals`, `shoppingList`, `fmtAmount`, `addMonths`, `expiry`, `invStats`, `zutatenListeFuerPrompt`, `normalizeAIRecipe`

**data_zutaten.js**: (nur Daten)

**data_bls.js**: `blsStats`, `blsLine`

**data_sousvide.js**: `svMinTemp`, `svTabelleHtml`

**logic2.js**: `isoLocal`, `addDaysIso`, `diffDays`, `fmtDate`, `shelfDays`, `lagerort`, `newLot`, `lotState`, `stockTotals`, `deductFifo`, `recipeNeed`, `coverage`, `spiceCheck`, `whatCanICook`, `expiringSoon`, `lowStock`, `weekNeed`, `shopPlan`, `fmtGrams`, `emptyDay`, `dayOf`, `round1x`, `nutriOfIng`, `entryRecipe`, `entryIng`, `entryQuick`, `diaryTotals`, `macroGoals`, `weekStats`, `streak`, `weightSeries`, `recentFoods`, `calcGoals`, `fastStatus`, `fmtHM`, `mulberry32`, `autoPlan`, `weekSummary`, `normalizeUserRecipe`, `parseRecipeJson`, `exportState`, `importState`, `diaryCsv`

**data_bausteine.js**: `B`

**logic3.js**: `cubePP`, `bausteinAllergene`, `validateBaustein`, `runQABaukasten`, `stylePasst`, `mealBuild`, `mealName`, `suggestMeals`, `planItemInfo`, `sessionYield`, `sessionNeeds`, `prepShopping`, `sessionTimeline`, `bookSession`, `addMonthsStr`, `sollBestand`, `istBestand`, `prepRoundsFor`, `etikett`

**data_saison.js**: `S`

**logic4.js**: `saisonPattern`, `saisonYearRound`, `saisonRated`, `saisonCode`, `saisonScore`, `saisonMonths`, `saisonRangeText`, `saisonBest`, `saisonLevel`, `saisonMonthSummary`, `saisonTop`, `saisonWeek`, `saisonShopState`, `saisonCycle`, `saisonCleanOv`, `runQASaison`

**look.js**: `lookClean`, `lookHsl`, `lookHex`, `lookLum`, `lookContrast`, `lookHexRgb`, `lookPalette`, `lookLoad`, `lookSave`, `lookWidth`, `lookLayoutNow`, `lookSchemeNow`, `lookApply`, `lookSet`, `lookReset`, `lookInit`, `lookPanel`

**app_a.js**: `emptyWeek`, `defaultPrep`, `defaultState`, `mergeState`, `loadLocal`, `save`, `flush`, `initRuntime`, `toast`, `copyText`, `saveFile`, `statusOf`, `mealForSlot`, `slotsOn`, `slotsPlan`, `slotsDiary`, `goalsNow`, `dayRec`, `dayW`, `ringSvg`, `macroBar`, `weekBars`, `weightChart`, `entryRow`, `fastCard`, `viewHeute`, `addListHtml`, `addSheet`

**data_quellen.js**: `quellenBlock`

**app_b.js**: `cuisines`, `isSmoothie`, `filtered`, `activeFilterCount`, `recipeCard`, `renderList`, `distBlock`, `viewRezepte`, `nutriBlock`, `checksBlock`, `ingredientRows`, `spiceChips`, `recipeText`, `recipeSheet`, `newEd`, `edFrom`, `edToRecipe`, `edResultsHtml`, `editorSheet`, `kiImportPrompt`

**app_c.js**: `seg`, `slotOptions`, `goalBar`, `viewPlanTab`, `dessertPlanSwitch`, `viewPlan`, `shopData`, `shopKey`, `shopText`, `viewShop`, `viewVorrat`, `expBadge`, `viewStock`, `usageCount`, `viewSpices`, `viewFreezer`, `viewKochen`, `stockResultsHtml`, `stockAddSheet`, `ingSheet`, `openAttr`, `calcPanel`, `infoPanel`, `backupPanel`, `profilePanel`, `statusPanel`, `libPanel`, `equipTable`, `equipPanel`, `viewMehr`

**app_d.js**: `render`, `renderSheet`, `openSheet`, `aiErr`, `exclText`, `profileText`, `runPlanKI`, `runVorratKI`, `runImportKI`, `confirmed`, `dropRecipeRefs`, `logEntry`, `invPortion`, `eatInv`, `markCooked`, `buyGrams`, `addMonthsIso`, `fillStep`, `recomputeEntry`, `newInvForm`, `runAuto`

**app_f.js**: `saisonOn`, `saisonOvNow`, `saisonRealMonth`, `saisonMonth`, `saisonName`, `saisonUsable`, `saisonBadge`, `saisonOptionTag`, `saisonStrip`, `saisonStateBadge`, `saisonRecipePanel`, `saisonTag`, `saisonShopNote`, `saisonWeekLine`, `saisonPlanSwitch`, `saisonPlanFn`, `saisonPicks`, `saisonFMonths`, `saisonRankSort`, `saisonUsage`, `saisonHeute`, `saisonHeat`, `viewSaison`, `saisonSettings`

**app_e.js**: `prepSt`, `prepAllergBadges`, `cubeLine`, `isUsable`, `viewPrep`, `prepStart`, `prepKatalog`, `bausteinSheet`, `prepBauen`, `prepFreezable`, `prepSession`, `prepToast`

---

## 13. Anhang: gesperrte Rezept-Links (zum Kopieren für ChatGPT)
Diese Liste steht auch als eigene Datei `gesperrte_links.txt` im Projektpaket. Die Abruf-Werkzeuge von Claude blockieren Chefkoch, BBC Good Food, Allrecipes und Serious Eats (`SITE_BLOCKED`); die Seiten wurden nicht gelesen und die Sperre wird nicht umgangen. Die zugehörigen Rezepte in der App sind „Eigene Fassung“. Abschnitt F nennt nur Titel, weil die Links nicht mehr vorliegen.

```
GESPERRTE REZEPT-LINKS FÜR CHATGPT (Stand 28.10.2026)

Diese Links stammen aus dem Chatverlauf (die meisten hat der Auftraggeber geschickt; einzelne können aus Suchergebnissen stammen). Die Abruf-Werkzeuge von Claude haben die Seiten blockiert (SITE_BLOCKED), sie wurden nicht gelesen und die Sperre wurde nicht umgangen. In der App stehen dazu "Eigene Fassungen" nach den Titeln; die Zutaten und Schritte der Originale sind unbekannt.

=== PROMPT ZUM KOPIEREN (vor die Links setzen) ===
Öffne jeden der folgenden Rezept-Links. Gib pro Rezept in eigenen Worten (nicht wörtlich abschreiben) aus: Titel, Portionen, Zutaten mit Mengen in Gramm bzw. Milliliter (metrisch), Zubereitungsschritte mit Zeiten und Temperaturen, Garzeit und Gerät (Backofen, Airfryer, Herd, Multikocher, Sous-vide, Kontaktgrill oder Dutch Oven), Lagerung und ob einfrierbar. Keine Süßkartoffel, Bataten und Ananas verwenden: wenn das Original sie enthält, schreibe es dazu und nenne einen Ersatz. Wenn du einen Link nicht öffnen kannst, schreibe das deutlich dazu und erfinde nichts. Das Importformat (JSON) steht in der Datei perplexity-prompt-rezepte.txt im Projektpaket.
=== ENDE PROMPT ===

--- A. Chefkoch, Link liegt vor (42) ---
https://www.chefkoch.de/rezepte/1594721266741395/Gong-Bao-Chicken.html
https://www.chefkoch.de/rezepte/1617001268903549/Himbeerkuchen.html
https://www.chefkoch.de/rezepte/1674221275400259/Roros-Griessnockerlsuppe.html
https://www.chefkoch.de/rezepte/1708381279716392/Gruene-Bohnen-Eintopf.html
https://www.chefkoch.de/rezepte/1759701285679844/Kartoffel-Rindfleisch-Suppe.html
https://www.chefkoch.de/rezepte/1772141287238724/Bacon-Tomaten-Frischkaesehaeppchen.html
https://www.chefkoch.de/rezepte/1816341294440474/Himbeerdessert-mit-Spekulatius.html
https://www.chefkoch.de/rezepte/1831471296897945/Schmand-Kraeuter-Dressing.html
https://www.chefkoch.de/rezepte/1891401307954679/Gruenkohlchips.html
https://www.chefkoch.de/rezepte/2017081326788424/Huehnersuppe-mit-Haehnchenschenkel.html
https://www.chefkoch.de/rezepte/2055811332403165/Kraeftige-Rindfleischsuppe.html
https://www.chefkoch.de/rezepte/2133281343053838/Rinderrouladen-klassisch.html
https://www.chefkoch.de/rezepte/2147311345010037/Einfache-Gemuesesuppe.html
https://www.chefkoch.de/rezepte/2286151364553555/Salat-Dressing-mit-Joghurt-ohne-Sahne-fuer-alle-Salate-geeignet.html
https://www.chefkoch.de/rezepte/2386241377981729/Kung-Pao-Huhn-2.html
https://www.chefkoch.de/rezepte/240791097407792/Peking-Suppe.html
https://www.chefkoch.de/rezepte/2465451388345414/Hackfleisch-Kaese-Lauch-Suppe-mit-Kartoffeln.html
https://www.chefkoch.de/rezepte/2479671390227833/Blumenkohleintopf.html
https://www.chefkoch.de/rezepte/253651100543918/Blumenkohl-Haehnchen-Pfanne.html
https://www.chefkoch.de/rezepte/2733891425739452/Bananenbrot-ohne-extra-Fett-und-Zucker.html
https://www.chefkoch.de/rezepte/2848091436834550/Chinesisch-gebratene-Nudeln-mit-Huehnchenfleisch-Ei-und-Gemuese.html
https://www.chefkoch.de/rezepte/2898441442181393/Ahornsirup-Senfdressing.html
https://www.chefkoch.de/rezepte/3033411456059781/Low-Carb-Bauerntopf.html
https://www.chefkoch.de/rezepte/3043751457044725/Chinesische-Teigtaschen-Jiaozi-Dumplings-Pot-Sticker-Version.html
https://www.chefkoch.de/rezepte/3114051464352730/Indisches-Mango-Chicken-Curry-fuer-den-Monsieur-Cuisine.html
https://www.chefkoch.de/rezepte/3196001475688526/Dessert-mit-islaendischem-Frischkaese-und-Weintrauben.html
https://www.chefkoch.de/rezepte/320561113912263/Szegediner-Gulasch.html
https://www.chefkoch.de/rezepte/3214781339456653/Bacon-Tomaten-Frischkaesehaeppchen.html
https://www.chefkoch.de/rezepte/3214781339456653/Bananenbrot-ohne-extra-Fett-und-Zucker.html
https://www.chefkoch.de/rezepte/3214781339456653/Gesundes-Bananenbrot.html
https://www.chefkoch.de/rezepte/3228981480357511/Pho-Ga.html
https://www.chefkoch.de/rezepte/3393031505036823/Gesundes-Bananenbrot.html
https://www.chefkoch.de/rezepte/3422781339869753/Gruenkohlchips.html
https://www.chefkoch.de/rezepte/3459451515356321/Paprikagemuese-Sous-Vide.html
https://www.chefkoch.de/rezepte/4000651613940765/Schweinebraten-sous-vide.html
https://www.chefkoch.de/rezepte/4340011730639098/Rindfleisch-mit-Paprika-und-Zwiebeln-nach-chinesischer-Art.html
https://www.chefkoch.de/rezepte/588501158230051/Burgunder-Fleischtopf.html
https://www.chefkoch.de/rezepte/761551178884059/Deftige-Erbsensuppe.html
https://www.chefkoch.de/rezepte/817681186299057/Birnen-in-Rotwein-an-Vanilleeis.html
https://www.chefkoch.de/rezepte/982031203667502/Kaese-Lauch-Suppe-mit-Hackfleisch.html
https://www.chefkoch.de/rezepte/438371135866566/Chinesisches-flambiertes-Eis.html
https://www.chefkoch.de/rezepte/654001166948846/Gedaempfte-Dattelkloesschen.html

--- B. BBC Good Food / BBC Food, Link liegt vor (82) ---
https://www.bbc.co.uk/food/recipes/no-bake_strawberry_30276
https://www.bbcgoodfood.com/recipes/20minute-seafood-pasta
https://www.bbcgoodfood.com/recipes/air-fryer-chicken-breasts
https://www.bbcgoodfood.com/recipes/air-fryer-chicken-nuggets
https://www.bbcgoodfood.com/recipes/air-fryer-chicken-thighs
https://www.bbcgoodfood.com/recipes/air-fryer-halloumi
https://www.bbcgoodfood.com/recipes/air-fryer-pork-belly
https://www.bbcgoodfood.com/recipes/air-fryer-pork-joint
https://www.bbcgoodfood.com/recipes/air-fryer-roast-chicken
https://www.bbcgoodfood.com/recipes/air-fryer-salmon
https://www.bbcgoodfood.com/recipes/american-pancakes
https://www.bbcgoodfood.com/recipes/arrabbiata-sauce
https://www.bbcgoodfood.com/recipes/beef-stout-stew-carrots
https://www.bbcgoodfood.com/recipes/beef-vegetable-casserole
https://www.bbcgoodfood.com/recipes/best-apple-crumble
https://www.bbcgoodfood.com/recipes/best-ever-chocolate-brownies-recipe
https://www.bbcgoodfood.com/recipes/braised-beef-onepot
https://www.bbcgoodfood.com/recipes/braised-pork-plums
https://www.bbcgoodfood.com/recipes/breakfast-smoothie
https://www.bbcgoodfood.com/recipes/cheese-sauce
https://www.bbcgoodfood.com/recipes/chicken-noodle-soup
https://www.bbcgoodfood.com/recipes/chinese-braised-pork-plums
https://www.bbcgoodfood.com/recipes/chocolate-cheesecake
https://www.bbcgoodfood.com/recipes/easiest-ever-paella
https://www.bbcgoodfood.com/recipes/easy-chocolate-molten-cakes
https://www.bbcgoodfood.com/recipes/easy-roast-pork-shoulder
https://www.bbcgoodfood.com/recipes/ham-mushroom-spinach-frittata
https://www.bbcgoodfood.com/recipes/healthy-shakshuka
https://www.bbcgoodfood.com/recipes/instant-frozen-berry-yogurt
https://www.bbcgoodfood.com/recipes/kale-smoothie
https://www.bbcgoodfood.com/recipes/lemon-coriander-hummus
https://www.bbcgoodfood.com/recipes/lemon-syllabub
https://www.bbcgoodfood.com/recipes/luscious-lemon-baked-cheesecake
https://www.bbcgoodfood.com/recipes/mango-banana-smoothie
https://www.bbcgoodfood.com/recipes/muffin-topped-winter-beef-stew
https://www.bbcgoodfood.com/recipes/mushroom-sauce
https://www.bbcgoodfood.com/recipes/mustardy-pork-apples
https://www.bbcgoodfood.com/recipes/no-cook-chocolate-tart
https://www.bbcgoodfood.com/recipes/nutty-chicken-satay-strips
https://www.bbcgoodfood.com/recipes/nutty-chicken-sate-strips
https://www.bbcgoodfood.com/recipes/peanut-butter-smoothie
https://www.bbcgoodfood.com/recipes/peppercorn-sauce
https://www.bbcgoodfood.com/recipes/pizza-sauce
https://www.bbcgoodfood.com/recipes/pork-green-bean-oyster-stir-fry
https://www.bbcgoodfood.com/recipes/pork-medallions
https://www.bbcgoodfood.com/recipes/pork-parsnip-cobbler
https://www.bbcgoodfood.com/recipes/prawn-tikka-masala
https://www.bbcgoodfood.com/recipes/simple-seafood-chowder
https://www.bbcgoodfood.com/recipes/slow-cooker-beef-stew
https://www.bbcgoodfood.com/recipes/spiced-carrot-lentil-soup
https://www.bbcgoodfood.com/recipes/spicy-root-lentil-casserole
https://www.bbcgoodfood.com/recipes/strawberry-smoothie
https://www.bbcgoodfood.com/recipes/sunshine-smoothie
https://www.bbcgoodfood.com/recipes/sweet-sour-sauce
https://www.bbcgoodfood.com/recipes/thai-pork-peanut-curry
https://www.bbcgoodfood.com/recipes/tomato-basil-sauce
https://www.bbcgoodfood.com/recipes/tomato-pepper-bean-one-pot
https://www.bbcgoodfood.com/recipes/tomato-soup
https://www.bbcgoodfood.com/recipes/two-minute-breakfast-smoothie
https://www.bbcgoodfood.com/recipes/ultimate-lemon-meringue-pie
https://www.bbcgoodfood.com/recipes/white-sauce
https://www.bbcgoodfoodme.com/recipes/bean-halloumi-stew/
https://www.bbcgoodfoodme.com/recipes/chilli-prawn-linguine/
https://www.bbcgoodfood.com/recipes/lychee-ginger-sorbet
https://www.bbcgoodfood.com/recipes/tanghulu
https://www.bbcgoodfood.com/recipes/avocado-strawberry-smoothie
https://www.bbcgoodfood.com/recipes/tropical-breakfast-smoothie
https://www.bbcgoodfood.com/recipes/blueberry-smoothie-recipe
https://www.bbcgoodfood.com/recipes/kiwi-fruit-smoothie
https://www.bbcgoodfood.com/recipes/creamy-mango-coconut-smoothie
https://www.bbcgoodfood.com/recipes/immune-boosting-smoothie
https://www.bbcgoodfood.com/recipes/chicken-caesar-salad
https://www.bbcgoodfood.com/recipes/greek-salad
https://www.bbcgoodfood.com/recipes/chicken-satay-salad
https://www.bbcgoodfood.com/recipes/classic-potato-salad
https://www.bbcgoodfood.com/recipes/10minute-couscous-salad
https://www.bbcgoodfood.com/recipes/quinoa-salad-grilled-halloumi
https://www.bbcgoodfood.com/recipes/epic-summer-salad
https://www.bbcgoodfood.com/recipes/salade-nicoise
https://www.bbcgoodfood.com/recipes/rainbow-fruit-skewers
https://www.bbcgoodfood.com/recipes/crunchy-granola-berries-cherries
https://www.bbcgoodfood.com/recipes/homemade-chicken-nachos

--- C. Allrecipes, Link liegt vor (62) ---
https://www.allrecipes.com/recipe/100606/beef-bulgogi/
https://www.allrecipes.com/recipe/109297/cedar-planked-salmon/
https://www.allrecipes.com/recipe/11786/hearty-vegetable-lasagna/
https://www.allrecipes.com/recipe/128699/famous-chicken-adobo/
https://www.allrecipes.com/recipe/16167/beef-bourguignon-i/
https://www.allrecipes.com/recipe/162690/simple-beef-short-ribs/
https://www.allrecipes.com/recipe/16276/chili-i/
https://www.allrecipes.com/recipe/16348/baked-pork-chops-i/
https://www.allrecipes.com/recipe/16899/beef-wellington/
https://www.allrecipes.com/recipe/17219/best-tuna-casserole/
https://www.allrecipes.com/recipe/18805/old-fashioned-mac-and-cheese/
https://www.allrecipes.com/recipe/189058/super-simple-salmon/
https://www.allrecipes.com/recipe/191156/restaurant-style-beef-and-broccoli/
https://www.allrecipes.com/recipe/213238/indian-fish-curry/
https://www.allrecipes.com/recipe/213268/classic-goulash/
https://www.allrecipes.com/recipe/215231/empanadas-beef-turnovers/
https://www.allrecipes.com/recipe/21555/salmon-with-dill/
https://www.allrecipes.com/recipe/219048/pork-carnitas/
https://www.allrecipes.com/recipe/219758/ellens-chicken-cacciatore/
https://www.allrecipes.com/recipe/221962/roast-beef-tenderloin/
https://www.allrecipes.com/recipe/222202/grilled-cod/
https://www.allrecipes.com/recipe/222978/roast-adobo-pork-loin/
https://www.allrecipes.com/recipe/223042/chicken-parmesan/
https://www.allrecipes.com/recipe/228056/parmesan-crusted-tilapia-fillets/
https://www.allrecipes.com/recipe/228823/quick-beef-stir-fry/
https://www.allrecipes.com/recipe/229123/best-beef-enchiladas/
https://www.allrecipes.com/recipe/231030/braised-corned-beef-brisket/
https://www.allrecipes.com/recipe/231381/beef-lo-mein/
https://www.allrecipes.com/recipe/234534/beef-and-guinness-stew/
https://www.allrecipes.com/recipe/23600/worlds-best-lasagna/
https://www.allrecipes.com/recipe/239854/south-asian-style-ground-beef-keema/
https://www.allrecipes.com/recipe/242352/greek-lemon-chicken-and-potatoes/
https://www.allrecipes.com/recipe/24712/ginger-veggie-stir-fry/
https://www.allrecipes.com/recipe/25202/beef-stroganoff-iii/
https://www.allrecipes.com/recipe/25309/seans-falafel-and-cucumber-sauce/
https://www.allrecipes.com/recipe/25321/eggplant-parmesan-ii/
https://www.allrecipes.com/recipe/255038/sheet-pan-chicken-fajitas/
https://www.allrecipes.com/recipe/257938/spicy-thai-basil-chicken-pad-krapow-gai/
https://www.allrecipes.com/recipe/258494/chinese-barbeque-pork-char-siu/
https://www.allrecipes.com/recipe/267015/pakistani-ground-beef-curry/
https://www.allrecipes.com/recipe/282124/honey-garlic-chicken-thighs/
https://www.allrecipes.com/recipe/34746/baked-salmon-ii/
https://www.allrecipes.com/recipe/36875/indonesian-satay/
https://www.allrecipes.com/recipe/45736/chicken-tikka-masala/
https://www.allrecipes.com/recipe/53729/fish-tacos/
https://www.allrecipes.com/recipe/57264/simply-the-easiest-beef-brisket/
https://www.allrecipes.com/recipe/60598/vegetarian-korma/
https://www.allrecipes.com/recipe/69633/stovetop-moroccan-tagine/
https://www.allrecipes.com/recipe/70163/easy-baked-tilapia/
https://www.allrecipes.com/recipe/70935/taqueria-style-tacos-carne-asada/
https://www.allrecipes.com/recipe/72508/the-best-vegetarian-chili-in-the-world/
https://www.allrecipes.com/recipe/83117/asian-pork-tenderloin/
https://www.allrecipes.com/recipe/84137/easy-paella/
https://www.allrecipes.com/recipe/8667/jays-jerk-chicken/
https://www.allrecipes.com/recipe/8669/chicken-cordon-bleu-ii/
https://www.allrecipes.com/recipe/8778/cajun-chicken-pasta/
https://www.allrecipes.com/recipe/8887/chicken-marsala/
https://www.allrecipes.com/recipe/8941/slow-cooker-chicken-and-dumplings/
https://www.allrecipes.com/recipe/9023/baked-teriyaki-chicken/
https://www.allrecipes.com/recipe/9027/kung-pao-chicken/
https://www.allrecipes.com/recipe/91499/general-tsaos-chicken-ii/
https://www.allrecipes.com/the-best-easy-pork-tenderloin-recipe-7485158

--- D. Serious Eats, Link liegt vor (3) ---
https://www.seriouseats.com/cuban-sandwich-recipe
https://www.seriouseats.com/sweet-chinese-mooncakes-with-walnut-red-bean
https://www.seriouseats.com/chinese-ginger-milk-curd-recipe-11868260

--- G. essen-und-trinken.de, laut robots.txt nicht abrufbar (Panini-Sammelartikel „Kontaktgrill-Test“, drei Dessert-Rezepte und ein Salat) (5) ---
https://www.essen-und-trinken.de/vergleich/kontaktgrill-test/
https://www.essen-und-trinken.de/rezepte/31493-rzpt-mangopudding
https://www.essen-und-trinken.de/rezepte/glueckskekse-12086190.html
https://www.essen-und-trinken.de/rezepte/53951-rzpt-chinesische-pfannkuchen-mit-pflaumensauce
https://www.essen-und-trinken.de/rezepte/salat-mit-honig-senf-dressing-12066152.html

--- E. BBC Good Food / Serious Eats, Pfad aus dem Quellenvermerk der App rekonstruiert (bitte beim Öffnen prüfen) (10) ---
https://www.bbcgoodfood.com/recipes/pork-stroganoff
https://www.bbcgoodfood.com/recipes/pot-roast-chicken-stock
https://www.bbcgoodfood.com/recipes/pulled-pork
https://www.bbcgoodfood.com/recipes/sausage-bean-casserole
https://www.bbcgoodfood.com/recipes/sausage-white-bean-casserole
https://www.bbcgoodfood.com/recipes/sichuan-pork-broccoli-cashew-stir-fry
https://www.bbcgoodfood.com/recipes/slow-braised-pork-shoulder-cider-parsnips
https://www.seriouseats.com/sous-vide-barbecue-pulled-pork-shoulder-recipe
https://www.seriouseats.com/sous-vide-glazed-carrots-recipe
https://www.seriouseats.com/sous-vide-steaks-recipe

--- F. Gesperrt, aber nur der Titel ist gespeichert (Link nicht mehr vorhanden, bitte im Chat des Auftraggebers suchen) ---
Chefkoch | Indisches Mango Chicken Curry für den Monsieur Cuisine  (in der App: r179 Indisches Mango-Hähnchen-Curry mit Reis)
Chefkoch | Schweinebraten sous vide  (in der App: r189 Sous-vide-Schweinebraten mit Kartoffeln und Sauerkraut)
Chefkoch | Paprikagemüse Sous-Vide  (in der App: r192 Sous-vide-Paprika-Bowl mit Kichererbsen und Feta)
Chefkoch | Chinesisch gebratene Nudeln mit Hühnchenfleisch, Ei und Gemüse  (in der App: r209 Chinesisch gebratene Nudeln mit Hähnchen, Ei und Gemüse)
Chefkoch | Peking-Suppe  (in der App: r210 Peking-Suppe (scharf-sauer) mit Hähnchen, Tofu und Ei)
Chefkoch | Gong Bao Chicken  (in der App: r211 Gong Bao Chicken (Sichuan-Art) mit Erdnüssen und Sellerie)
Chefkoch | Kung Pao Huhn 2  (in der App: r212 Kung-Pao-Huhn mit Paprika und Zucchini (Variante mit Hoisin))
Chefkoch | Chinesische Bratnudeln  (in der App: r213 Chinesische Bratnudeln mit Tofu, Ei und Gemüse)
Chefkoch | Chinesische Teigtaschen Jiaozi Dumplings Pot-Sticker-Version  (in der App: r214 Jiaozi (Pot Sticker): gebratene Teigtaschen mit Schwein, Pute und Kohl)
Chefkoch | Rindfleisch mit Paprika und Zwiebeln nach chinesischer Art  (in der App: r215 Rindfleisch mit Paprika und Zwiebeln nach chinesischer Art)
Chefkoch | Chinesische Nudelpfanne  (in der App: r216 Chinesische Nudelpfanne mit Schweinefilet und Kohl)
Chefkoch | Blumenkohl-Hähnchen-Pfanne  (in der App: r217 Blumenkohl-Hähnchen-Pfanne mit Curry und Reis)
Chefkoch | Gebratene chinesische Nudeln  (in der App: r218 Gebratene chinesische Nudeln mit Garnelen, Ei und Chinakohl)
Chefkoch | Overnight Oats; Direktlink nicht verifiziert  (in der App: r224 Overnight Oats mit Skyr, Zimt und Vanille)
Chefkoch | Bananen-Apfel-Zimt-Porridge; Direktlink nicht verifiziert  (in der App: r225 Bananen-Apfel-Zimt-Porridge)
Chefkoch | Frühstücks-Muffins; Direktlink nicht verifiziert  (in der App: r226 Frühstücks-Muffins mit Quark, Haferflocken und Beeren)
Chefkoch | Rührei aus dem Backofen; Direktlink nicht verifiziert  (in der App: r227 Rührei aus dem Backofen mit Hüttenkäse und Schnittlauch)
Chefkoch | Haferflockenbrot; Direktlink nicht verifiziert  (in der App: r228 Haferflockenbrot mit Quark und Sonnenblumenkernen)
Chefkoch | Bestes Bircher Müsli; Direktlink nicht verifiziert  (in der App: r229 Bircher Müsli mit Apfel, Mandeln und Skyr)
Chefkoch | Erdbeer-Bananen-Smoothie; Direktlink nicht verifiziert  (in der App: r230 Erdbeer-Bananen-Smoothie mit Skyr)
Chefkoch | Joghurt-Dressing ohne Sahne  (in der App: Baustein b67)
Chefkoch | Schmand-Kräuter-Dressing  (in der App: Baustein b68)
Chefkoch | Ahornsirup-Senf-Dressing  (in der App: Baustein b69)
BBC Good Food | Pizzasauce  (in der App: Baustein b58)
BBC Good Food | Käsesauce  (in der App: Baustein b59)
BBC Good Food | Weiße Sauce  (in der App: Baustein b60)
BBC Good Food | Süß-saure Sauce  (in der App: Baustein b61)
BBC Good Food | Tomaten-Basilikum-Sauce  (in der App: Baustein b62)
BBC Good Food | Pilzsauce  (in der App: Baustein b63)
BBC Good Food | Pfeffersauce  (in der App: Baustein b64)
BBC Good Food | Brotsauce  (in der App: Baustein b65)
BBC Good Food | Arrabbiata  (in der App: Baustein b66)
```
