# HSLinkDrag – Konzept

| | |
|---|---|
| **Status** | **[Chrome Web Store](https://chromewebstore.google.com/detail/hslinkdrag/pilepemcdicgnpdbilcoiieediljfkne):** **1.0.4 veröffentlicht** (2026-11-05). **GitHub: Version 1.0.4 – fertig und getestet** (gleicher Stand wie Firefox 1.0.4, Tag `v1.0.4`). Store und GitHub haben denselben Stand |
| **Stand** | 2026-11-04 |
| **Name** | HSLinkDrag |
| **Store-ID** | `pilepemcdicgnpdbilcoiieediljfkne` |
| **Plattform** | Google Chrome, aktuelle Version (Manifest V3). Entwicklung unter Linux. Schwesterprojekt für Firefox: `../../FIREFOX_AddOns/HSLinkDrag` (github.com/HSBitbooster/HSLinkDrag-Firefox, [addons.mozilla.org](https://addons.mozilla.org/firefox/addon/hslinkdrag/)) |
| **Versionen** | Chrome- und Firefox-Version haben **immer dieselbe Versionsnummer und denselben Funktionsstand** (Abschnitt 11) |
| **Logo** | `icons/logo-original.png` (295 × 295 px, vom Nutzer geliefert) – daraus werden die Icon-Größen 16/32/48/128 erzeugt |

---

## 1. Ziel

Eine Chrome-Erweiterung, die Links, Bilder und markierten Text per **Click & Drag**
(Element anklicken, Maus mit gedrückter Taste ein Stück ziehen, loslassen) in einem
neuen Tab öffnet. Die **Zugrichtung** (links/rechts) entscheidet, ob der Tab im
Vorder- oder Hintergrund geöffnet wird. Verhalten und Tab-Position sind einstellbar.

## 2. Anforderungen (aus dem Auftrag)

| Nr. | Anforderung |
|---|---|
| A1 | Link per Click & Drag in neuem Tab öffnen. |
| A2 | Zugrichtung (links/rechts) bestimmt Vordergrund/Hintergrund – **einstellbar**. |
| A3 | Position des neuen Tabs einstellbar: rechts vom Ursprungstab, links davon, am Ende aller Tabs. |
| A4 | Bild **ohne** Link per Click & Drag → Bild in neuem Tab öffnen (gleiche Regeln). |
| A5 | Markierter Text per Click & Drag → Suche in neuem Tab (Google, DuckDuckGo, Bing einstellbar). |
| A6 | Ist der markierte Text eine gültige Internetadresse → Adresse öffnen statt suchen (gleiche Regeln). |
| A7 | Ist der Link oder der markierte Text eine E-Mail-Adresse → Standard-E-Mail-Client öffnen. |
| A8 | Link auf einem Bild: SHIFT entscheidet, ob Link oder Bild geöffnet wird – **einstellbar**. |
| A9 | Optionsseite kompakt, aber mit verständlichem Erklärungstext zu jeder Option/Optionsgruppe. |
| A10 | ESC während des Ziehens bricht die Geste ab – es wird nichts geöffnet. |

## 3. Begriffe

- **Geste / Drag:** Maustaste auf einem Element drücken, ziehen, loslassen.
- **Ursprungstab:** Der Tab, in dem die Geste ausgeführt wurde.
- **Ziel:** Das, was geöffnet wird (Link-URL, Bild-URL, Such-URL, `mailto:`).
- **Mindestdistanz:** Horizontale Strecke in Pixeln, ab der eine Geste zählt.

## 4. Verhalten im Detail

### 4.1 Geste erkennen

Verwendet werden die **nativen Drag-&-Drop-Ereignisse** des Browsers (`dragstart`,
`dragover`, `drop`, `dragend`). Links, Bilder und markierter Text sind im Browser
ohnehin „ziehbar“ – die Erweiterung hängt sich nur daran.

Ablauf:

1. `dragstart`: Startposition merken, Ziel bestimmen (siehe 4.2). Gibt es kein
   passendes Ziel → Erweiterung tut nichts.
2. `dragover` (auf dem Dokument): Drop erlauben (`preventDefault`), aktuelle Position
   und SHIFT-Zustand merken.
3. `drop`: Standardverhalten der Seite verhindern, horizontale Distanz `dx`
   berechnen, Richtung bestimmen, Auftrag an den Hintergrund-Service-Worker senden.
4. `dragend`: Aufräumen (Overlay entfernen, Zustand zurücksetzen). Kam vorher
   kein `drop` (Drop außerhalb des Fensters oder Abbruch per ESC) → **nichts öffnen**.

**Abbruch mit ESC:** Drückt der Nutzer während des Ziehens ESC, wird die Geste
abgebrochen – es wird nichts geöffnet, das Overlay verschwindet. Technisch bricht
Chrome einen nativen Drag bei ESC selbst ab: Es folgt kein `drop`, nur `dragend`
(mit `dropEffect = "none"`). Tastaturereignisse erreichen die Seite während eines
Drags nicht, daher wird ESC nicht per `keydown` abgefangen, sondern über das
fehlende `drop` erkannt. Muss im Testdurchlauf ausdrücklich geprüft werden.

Begründung gegenüber eigener Maus-Logik (`mousedown`/`mousemove`/`mouseup`):
Native Drags stören die normale Textmarkierung nicht, funktionieren mit allen
Link-/Bildarten und erfordern kein Unterdrücken von Klicks. Außerdem bleibt das
Ziehen von Links in die Lesezeichenleiste oder andere Programme möglich.

**Ausnahme – Maus-Geste** (seit 1.0.1/1.0.2) für drei Fälle:

- **Verlinkte Bilder und verlinkte SVG-Grafiken:** Chrome meldet unter Linux die
  SHIFT-Taste während eines nativen Drags nicht (im Test immer `false`). Darum wird hier
  der native Drag bei `dragstart` abgebrochen und die Geste über Mausereignisse
  weiterverfolgt; dort kommen SHIFT (`mousemove`/`mouseup`, `keydown`/`keyup`) und ESC
  (`keydown`) zuverlässig an. Nachteil: Diese lassen sich bei aktiver Erweiterung nicht
  mehr nativ in andere Programme ziehen.
- **SVG-Grafiken und SVG-Links:** sind in Chrome nicht nativ ziehbar (kein `dragstart`).
  Die Geste beginnt bei `mousedown` auf einer SVG-Grafik; Textmarkierung wird
  währenddessen per `selectstart` unterdrückt. Nicht in Bedienelementen (Button,
  `role="button"`, Label, Formularfelder), und nicht, wenn die Seite den `mousedown`
  selbst behandelt (`defaultPrevented`/`stopPropagation`, z. B. verschiebbare Diagramme).
- **Alles in eingebetteten Frames (iframes)** (seit 1.0.2): Chrome leitet Mausereignisse
  während eines Klicks an den Frame weiter, in dem er begann – auch wenn der Zeiger den
  Frame verlässt. Damit funktioniert das Ziehen über den Frame-Rand hinaus mit korrekter
  Richtung, auch bei fremdem Ursprung. Der native Weg ging dafür nicht: Den Drop
  außerhalb eines Frames mit fremdem Ursprung sperrt Chrome, und `screenX` in
  Drag-Ereignissen war unter Linux unbrauchbar (1.0.1 öffnete immer im Hintergrund).
  Nicht unterstützt: von der Seite **in** einen Frame hinein ziehen.

Nach einer Maus-Geste wird der folgende `click` unterdrückt, damit der Link nicht
zusätzlich im aktuellen Tab öffnet.

**Seite bricht den nativen Drag ab** (seit 1.0.4): Bricht die Seite das Ziehen eines Links
oder Bildes selbst ab (`dragstart` mit `preventDefault`, z. B. Googles Reiter „Bilder“,
„Videos“), geht die Geste als Maus-Geste weiter. Schutz für Seiten mit eigenem Ziehen
(Karussell, Schieberegler, sortierbare Listen): Verschiebt die Seite das gezogene Element
um mehr als 3 px, bricht HSLinkDrag die Geste ab und öffnet nichts.

Startet Chrome nach dem `mousedown` auf einer SVG-Grafik trotzdem einen nativen Drag
(SVG in einem HTML-Link: Chrome meldet den Link als Startpunkt, nicht die Grafik), wird
dieser abgebrochen und die bereits laufende Maus-Geste behält Vorrang (seit 1.0.3).

**Hinweis außerhalb eines Frames** (seit 1.0.3): Verlässt der Zeiger während einer
Geste den Frame, schickt der Frame Text und Position per `postMessage` an den
übergeordneten Frame. Dieser findet das Frame-Element über `event.source`, rechnet die
Koordinaten um (Position, Rahmen, Innenabstand, Zoom) und zeichnet den Hinweis selbst –
bei verschachtelten Frames wird weitergereicht. So bleibt der Hinweis nicht am
Frame-Rand hängen.

**Richtung:** Nur das Vorzeichen von `dx` zählt (rechts = positiv, links = negativ).
Die vertikale Bewegung wird ignoriert (oben/unten später ggf. als eigene Aktionen;
Datenmodell ist dafür erweiterbar). Ist `|dx|` kleiner als die Mindestdistanz
(Standard 20 px, einstellbar), passiert nichts.

**Rückmeldung beim Ziehen:** Ein kleines Overlay am Mauszeiger zeigt, was beim
Loslassen passiert (z. B. „→ Vordergrund“, „← Hintergrund“, „✉ E-Mail“, „—“ unter
Mindestdistanz). Abschaltbar.

**Zieh-Bild des Browsers** (seit 1.0.4): Chrome zeigt beim Ziehen eines Links einen Kasten
mit Linktext und Adresse (bei Bildern eine Vorschau), der den Hinweis überdeckt. Ist der
Hinweis eingeschaltet, ersetzt HSLinkDrag dieses Zieh-Bild per
`dataTransfer.setDragImage()` durch ein durchsichtiges 1 × 1-Bild. Bei markiertem Text
bleibt das Zieh-Bild (überdeckt den Hinweis nicht); bei ausgeschaltetem Hinweis ebenso.

**Ein/Aus & Ausschlussliste:** Die Erweiterung tut nichts, wenn sie über den
Toolbar-Button global ausgeschaltet ist oder die Domain der Seite in der
Ausschlussliste steht (Subdomains eingeschlossen).

**Markierter Text:** Gilt nur, wenn die Maustaste innerhalb der Markierung gedrückt wurde;
sonst gilt das Element unter dem Mauszeiger (z. B. der Link), auch wenn anderswo oder im
selben Link Text markiert ist (seit 1.0.4, aus dem Firefox-Test F7 übernommen).

### 4.2 Zielbestimmung (Prioritätsreihenfolge)

| Prio | Situation beim Drag-Start | Ergebnis |
|---|---|---|
| 0 | Start in Eingabefeld, Textarea oder editierbarem Bereich | **Nicht eingreifen** (Seite verhält sich normal) |
| 0 | Drop auf Eingabefeld/editierbaren Bereich | **Nicht eingreifen** (z. B. Text in Suchfeld ziehen bleibt möglich) |
| 1 | Markierter Text wird gezogen | Text-Logik (4.7) |
| 2 | Link mit `mailto:` | E-Mail-Client (4.8) |
| 3 | Link, der ein Bild enthält | Link oder Bild je nach SHIFT-Einstellung (4.6) |
| 4 | Normaler Link (`http`, `https`, `ftp`, `file`) | Link öffnen |
| 5 | Bild ohne Link | Bild öffnen (4.5) |
| – | `javascript:`-Links, leere/`#`-Links, sonstiges | Nichts tun |

### 4.3 Richtung → Vordergrund/Hintergrund

Einstellung je Richtung, Auswahl: **Vordergrund · Hintergrund · Deaktiviert**.
Standard: rechts = Vordergrund, links = Hintergrund.

### 4.4 Tab-Position

Auswahl: **Rechts vom Ursprungstab · Links vom Ursprungstab · Am Ende**.

Bei mehreren hintereinander geöffneten Tabs (entspricht Chrome-Verhalten
bei Mittelklick): Die Tabs reihen sich **in Öffnungsreihenfolge** rechts vom
Ursprungstab ein (1, 2, 3 …), statt sich jeweils direkt neben den Ursprungstab zu
drängen (3, 2, 1). Umsetzung über `openerTabId` und Merken des zuletzt geöffneten
Index je Ursprungstab; Zurücksetzen, sobald der Nutzer den Tab wechselt.

Neue Tabs werden immer im **selben Fenster** (und in einer evtl. vorhandenen
Tab-Gruppe des Ursprungstabs) geöffnet.

### 4.5 Bilder

- Verwendet wird die tatsächlich angezeigte Bildquelle (`img.currentSrc`, berücksichtigt
  `srcset`/`<picture>`).
- `data:`- und `blob:`-Bilder: werden in Version 1 ignoriert (Chrome erlaubt das
  direkte Öffnen in einem Tab nur eingeschränkt).
- CSS-Hintergrundbilder: nicht unterstützt.
- **Inline-SVG-Grafiken** (`<svg>` im Seitenquelltext, seit 1.0.2) werden wie Bilder
  behandelt, auch mit Link und SHIFT. Da sie keine eigene Adresse haben, wird die Grafik
  in eine eigenständige SVG-Datei umgewandelt (Farben/Schrift aus dem Seiten-CSS werden
  übernommen, `<use href="#…">`-Verweise auf Symbol-Sprites eingebettet, angezeigte Größe
  übernommen), in `chrome.storage.session` abgelegt (die neuesten 30, bis zum
  Browser-Neustart) und in der erweiterungseigenen Anzeigeseite `viewer/svg.html`
  geöffnet: dunkler Hintergrund, ans Fenster angepasst, Klick = Originalgröße,
  Knopf „Als SVG-Datei speichern“. Angezeigt wird per `<img>`, Scripte in der SVG laufen
  daher nicht. SVGs, die per `<img src="….svg">` eingebunden sind, sind normale Bilder.
- Grenzen der Umwandlung: externe Sprites (`<use href="datei.svg#id">`), Schriften und
  Bilder, die die SVG nachlädt, fehlen in der geöffneten Datei.

### 4.6 Link auf einem Bild + SHIFT

Einstellung: **„Ohne SHIFT öffnen:“ Link · Bild** (mit SHIFT jeweils das andere).
Standard: ohne SHIFT = Link, mit SHIFT = Bild.
Der SHIFT-Zustand wird **beim Loslassen** (`drop`) ausgewertet.

### 4.7 Markierter Text

Der Text wird getrimmt; Zeilenumbrüche werden zu Leerzeichen. Danach in dieser
Reihenfolge prüfen:

1. **E-Mail-Adresse** (`name@domain.tld`, optional mit `mailto:`) → 4.8
2. **Internetadresse** → öffnen. Erkannt werden:
   - mit Schema: `http://`, `https://`, `ftp://`
   - mit `www.` am Anfang (→ `https://` wird ergänzt)
   - `domain.tld[/pfad]` ohne Leerzeichen, wenn die Endung eine bekannte
     Top-Level-Domain ist (→ `https://` wird ergänzt)
   - IPv4-Adressen (optional mit Port/Pfad) und `localhost[:port]` (→ `http://` wird ergänzt)
   - umschließende Klammern/Anführungszeichen und abschließende Satzzeichen
     (`.`, `,`, `;`, `)`) werden entfernt
3. **Sonst** → Suche mit der eingestellten Suchmaschine:
   - Google: `https://www.google.com/search?q=…`
   - DuckDuckGo: `https://duckduckgo.com/?q=…`
   - Bing: `https://www.bing.com/search?q=…`
   - Chrome-Standard: die in Chrome eingestellte Suchmaschine (`chrome.search.query`
     im zuvor angelegten Tab). Keine eigene Such-URL.

### 4.8 E-Mail

`mailto:`-Links und als E-Mail erkannter Text öffnen den Standard-E-Mail-Client.
Es wird **kein** neuer Tab erzeugt (sonst bliebe ein leerer Tab zurück); die
Zugrichtung spielt hier keine Rolle. Technisch wird der Ursprungstab kurz auf die
`mailto:`-Adresse gesetzt – Chrome reicht das an das Betriebssystem weiter, die
Seite selbst bleibt erhalten.

Hinweis Linux: Chrome fragt beim ersten Mal ggf., ob `xdg-open` gestartet werden darf.

### 4.9 Wo die Erweiterung grundsätzlich nicht funktioniert

Technisch bedingt (Chrome lässt dort keine Erweiterungen zu): `chrome://`-Seiten,
Chrome Web Store, neue-Tab-Seite, eingebauter PDF-Viewer, andere Erweiterungsseiten.

## 5. Technische Architektur

- **Manifest V3**, reines JavaScript, **kein Build-Schritt**.
- **Berechtigungen:** `storage` (Einstellungen), `search` (Chrome-Standardsuche);
  Content-Script für `<all_urls>` (Chrome zeigt dafür „Kann alle Daten auf allen
  Websites lesen und ändern“ – bei dieser Art Erweiterung unvermeidbar).
- **Verteilung:** zunächst privat als entpackte Erweiterung; von Anfang an
  Web-Store-tauglich gebaut (minimale Berechtigungen, keine Fremd-Bibliotheken,
  kein nachgeladener Code, keine Datenübertragung).
- **Sprachen:** Deutsch und Englisch über `_locales/de` und `_locales/en`
  (`chrome.i18n`), Standardsprache Englisch, Deutsch bei deutscher Browsersprache.
- **Toolbar-Button:** Klick schaltet die Erweiterung global an/aus; Zustand wird
  am Icon angezeigt (Badge „AUS“ bzw. graues Icon). Optionsseite über Rechtsklick
  → „Optionen“.

| Komponente | Datei (geplant) | Aufgabe |
|---|---|---|
| Manifest | `manifest.json` | Deklaration, Berechtigungen |
| Content-Script | `src/content/content.js` | Geste erkennen, Ziel bestimmen, Nachricht senden |
| Erkennung | `src/shared/detect.js` | URL-/E-Mail-Erkennung, Such-URL bauen (gut testbar) |
| Einstellungen | `src/shared/settings.js` | Standardwerte, Laden/Speichern (`chrome.storage.sync`) |
| Service-Worker | `src/background.js` | Tab erzeugen (`chrome.tabs.create` mit `index`, `active`, `openerTabId`), `mailto:`, SVG-Grafiken ablegen |
| Optionsseite | `options/options.html/.css/.js` | Einstellungen |
| SVG-Anzeige | `viewer/svg.html/.css/.js` | Zeigt per Drag geöffnete Inline-SVG-Grafiken |
| Icons | `icons/` | 16/32/48/128 px |
| Tests | `test/testseite.html`, `test/detect.test.js` | Manuelle Testseite, Unit-Tests der Erkennung (Node, ohne Abhängigkeiten) |

Content-Script läuft in **allen Frames** (`all_frames: true`), damit es auch in
eingebetteten Seiten funktioniert. Einstellungen werden im Content-Script gecacht
und bei Änderung (`chrome.storage.onChanged`) aktualisiert.

## 6. Einstellungen

### 6.1 Datenmodell

Gespeichert in `chrome.storage.sync` (geräteübergreifend synchronisiert).

```js
{
  enabled:        true,          // globaler Ein/Aus-Schalter (Toolbar-Button)
  dirRight:       "foreground",  // "foreground" | "background" | "off"
  dirLeft:        "background",
  tabPosition:    "right",       // "right" | "left" | "end"
  imageLinkShift: "link",        // was OHNE Shift geöffnet wird: "link" | "image"
  searchEngine:   "google",      // "google" | "duckduckgo" | "bing" | "default"
  minDistance:    20,            // px
  showOverlay:    true,          // Hinweis beim Ziehen anzeigen
  excludedDomains: []            // z. B. ["trello.com", "drive.google.com"]
}
```

### 6.2 Skizze der Optionsseite

```
HSLinkDrag – Einstellungen
─────────────────────────────────────────────────────────────
Zugrichtung
  Nach rechts ziehen:  [Im Vordergrund öffnen ▾]
  Nach links ziehen:   [Im Hintergrund öffnen ▾]
  ⓘ Bestimmt, ob der neue Tab sofort angezeigt wird (Vordergrund)
    oder unauffällig im Hintergrund geladen wird.
─────────────────────────────────────────────────────────────
Position des neuen Tabs
  (•) Rechts vom aktuellen Tab  ( ) Links  ( ) Am Ende
  ⓘ Wo der neue Tab in der Tableiste eingefügt wird …
─────────────────────────────────────────────────────────────
Verlinkte Bilder
  Ohne SHIFT öffnen: (•) Link  ( ) Bild
  ⓘ Liegt ein Link auf einem Bild, entscheidet die SHIFT-Taste …
─────────────────────────────────────────────────────────────
Markierter Text
  Suchmaschine: [Google ▾]
  ⓘ Markierten Text ziehen startet eine Suche. Ist der Text eine
    Internetadresse, wird sie direkt geöffnet; eine E-Mail-Adresse
    öffnet das E-Mail-Programm.
─────────────────────────────────────────────────────────────
Erweitert
  Mindestdistanz: [20] px
  ⓘ So weit muss die Maus mindestens zur Seite bewegt werden …
  [✓] Hinweis beim Ziehen anzeigen
  ⓘ Zeigt am Mauszeiger, was beim Loslassen passiert …
─────────────────────────────────────────────────────────────
Ausgeschlossene Websites
  [ trello.com                        ]
  [ drive.google.com                  ]
  ⓘ Eine Domain pro Zeile. Auf diesen Seiten (inkl. Subdomains)
    bleibt HSLinkDrag inaktiv – nützlich bei Seiten mit eigenem
    Drag & Drop.
```

Speichern erfolgt sofort bei Änderung (kein „Speichern“-Knopf), mit kurzer
Bestätigung „Gespeichert“.

## 7. Bekannte Grenzen / Risiken

- Seiten mit eigener Drag-&-Drop-Logik (z. B. Trello, Google Drive, Kanban-Boards)
  können kollidieren → Ausschlussliste (F10).
- Seiten, die Ziehen per `draggable="false"` oder `user-drag: none` unterbinden,
  lösen kein `dragstart` aus → dort funktioniert die Geste nicht. Bricht eine Seite den
  `dragstart` dagegen per Script ab, greift der Rückfallweg (4.1).
- Seiten mit eigenem Ziehen, die das Element dabei **nicht** verschieben, könnten mit dem
  Rückfallweg kollidieren → Ausschlussliste. Steht ein Karussell schon am Ende, bewegt es
  sich nicht mehr, und der Link öffnet (im Firefox-Test T36 vom Nutzer akzeptiert).
- Mit eingeschaltetem Hinweis fehlt beim Ziehen von Links und Bildern das Zieh-Bild des
  Browsers – auch beim Ziehen in die Lesezeichenleiste oder andere Programme.
- Grafiken mit `pointer-events: none` in einem Link (z. B. das YouTube-Logo) sieht HSLinkDrag nicht:
  Der Link öffnet normal, mit SHIFT lässt sich die Grafik aber nicht öffnen. Vom Nutzer so belassen.
- `drop` außerhalb des Browserfensters wird nicht ausgewertet.
- Von der Seite in einen eingebetteten Frame hineinziehen öffnet nichts.
- Verlinkte Bilder/SVGs und alles in Frames können bei aktiver Erweiterung nicht nativ in andere Programme
  oder die Lesezeichenleiste gezogen werden (Maus-Geste, siehe 4.1).
- Elemente mit `draggable="true"` (außer Links/Bilder) gelten als seiteneigenes
  Drag & Drop – dort greift die Erweiterung nicht ein.
- Tabs, die schon vor der Installation offen waren, brauchen einmal Neuladen.
- Die bloße Endung entscheidet bei Adressen ohne `www.`/Schema: Endungen, die zugleich
  gängige Dateiendungen sind (`.md`, `.py`, `.rs`, `.sh`, `.so`, `.zip`, `.mov`), stehen
  bewusst nicht in der TLD-Liste (`src/shared/detect.js`).

## 8. Geklärte Fragen

Alle Fragen am 2026-11-03 wie vorgeschlagen entschieden. Aktuell keine offenen Fragen.

| Nr. | Frage | Entscheidung |
|---|---|---|
| F1 | Pro Richtung auch „Deaktiviert“ wählbar? | Ja |
| F2 | Vertikale Richtungen (oben/unten) jetzt oder später nutzen? | Nein, ignorieren; Datenmodell erweiterbar halten |
| F3 | Mehrere Tabs hintereinander: Reihenfolge 1,2,3 (wie Chrome) oder 3,2,1? | 1,2,3 |
| F4 | Mindestdistanz einstellbar? | Ja, Standard 20 px |
| F5 | URL-Erkennung: auch `domain.tld` ohne `www.`/Schema? IP-Adressen, `localhost:port`? | Ja (mit TLD-Liste), IP ja, localhost ja |
| F6 | Zusätzlich „Browser-Standardsuchmaschine“ oder eigene Such-URL? | Optional als 4. Option „Chrome-Standard“ |
| F7 | CSS-Hintergrundbilder / SVG auch öffnen? | CSS-Hintergründe nein; Inline-SVG ja (seit 1.0.2, auf Nutzerwunsch) |
| F8 | `data:`/`blob:`-Bilder? | Ignorieren in Version 1 |
| F9 | Visuelle Rückmeldung während des Ziehens (z. B. Hinweis „→ Vordergrund“)? | Kleines Overlay, abschaltbar |
| F10 | Ausschlussliste für Websites? | Ja, einfache Domainliste |
| F11 | Toolbar-Button zum schnellen Ein-/Ausschalten? | Ja, Klick schaltet global an/aus |
| F12 | Sprache der Oberfläche: nur Deutsch oder Deutsch + Englisch? | Deutsch + Englisch (`_locales`) |
| F13 | Verteilung: privat (entpackt laden) oder Chrome Web Store? | Erst privat, Web-Store-tauglich bauen |
| F14 | Drop außerhalb des Fensters / auf Eingabefeld → nichts tun? | Ja |
| F15 | Einstellungen geräteübergreifend synchronisieren (`storage.sync`)? | Ja |
| F16 | Gezogener Link mit markiertem Text darin – Link oder Text? | Link hat Vorrang, wenn der Drag auf dem Link startet und nicht in einer Markierung |

## 9. Entscheidungsprotokoll

| Datum | Entscheidung |
|---|---|
| 2026-11-03 | Projekt angelegt, Konzeptentwurf erstellt. |
| 2026-11-03 | F1–F16 wie vorgeschlagen angenommen (siehe Abschnitt 8); Konzeptabschnitte entsprechend angepasst. |
| 2026-11-03 | Name bleibt „HSLinkDrag“. Logo: vom Nutzer geliefertes Bild, abgelegt als `icons/logo-original.png`. |
| 2026-11-03 | Zielplattform: nur Chrome, aktuelle Version. Keine Unterstützung anderer Browser (Edge, Brave, Vivaldi …). |
| 2026-11-03 | Neue Anforderung A10: ESC während des Ziehens bricht die Geste ab (siehe 4.1). |
| 2026-11-03 | Freigabe „starten“; Version 1.0.0 umgesetzt. Umsetzungsdetails: E-Mail braucht ebenfalls die Mindestdistanz (Richtungseinstellung egal); kuratierte TLD-Liste; Mindestdistanz 5–300 px; Ausschlussliste prüft die Domain der Seite in der Adresszeile (auch in iframes). |
| 2026-11-03 | Nutzertest 1.0.0: SVG-Link öffnet nicht, SHIFT bei verlinkten Bildern wirkungslos (Linux), Wunsch nach Ziehen über Frame-Grenzen. → 1.0.1: Maus-Geste für verlinkte Bilder und SVG-Links, frameübergreifende Abstimmung über den Service-Worker (siehe 4.1). Testseite um mailto-, Lokal-, Frame- und Diagnosefälle erweitert. |
| 2026-11-03 | Nutzertest 1.0.1: SHIFT bei Bildlinks und SVG-Links ok; über den Frame-Rand bei gleichem Ursprung immer Hintergrund, bei fremdem Ursprung gar nicht. → 1.0.2: Gesten in Frames als Maus-Geste (Abstimmung über den Service-Worker entfernt); Ziehen in einen Frame hinein nicht mehr unterstützt. Neu auf Wunsch: Inline-SVG-Grafiken wie Bilder öffnen, inkl. Link + SHIFT (Anzeigeseite `viewer/svg.html`). |
| 2026-11-03 | Nutzertest 1.0.2: T25, T27, T28, T30–T32 ok. Hinweis bleibt am Frame-Rand hängen; T29 (SVG im HTML-Link) ohne SHIFT-Erkennung. → 1.0.3: laufende Maus-Geste hat Vorrang vor dem nativen Link-Drag; Hinweis wird außerhalb des Frames vom übergeordneten Frame gezeichnet. Test-Frames enthalten jetzt auch SVG, SVG im Link und SVG-Link. |
| 2026-11-03 | Nutzertest 1.0.3: T29, T33, T34 bestanden; Grundtests T1–T17, T19–T21, T24 ebenfalls bestanden. Version 1.0.3 abgeschlossen. |
| 2026-11-04 | Chrome-Web-Store-Paket vorbereitet: `scripts/build-zip.sh` (ZIP nur mit Laufzeitdateien), Store-Icon 128 px mit 16 px Rand (Store-Richtlinie, gilt auch im Manifest), Texte und Berechtigungsbegründungen de/en (`store/STORE-EINTRAG.md`), Datenschutzerklärung (`store/DATENSCHUTZ.md`), Screenshots 1280 × 800 und Werbekachel 440 × 280 de/en aus `store/scenes/` per `scripts/store-images.sh`. |
| 2026-11-04 | Version 1.0.3 im Chrome Web Store zur Prüfung eingereicht. Datenschutzerklärung öffentlich unter github.com/HSBitbooster/HSLinkDrag-Datenschutz. Dashboard-Hinweis „Erweiterte Hostberechtigungen“ bestätigt mit „Prüfen lassen“: `activeTab` oder feste Websites passen nicht, weil die Geste auf jeder Seite ohne vorherigen Klick bereitstehen muss. |
| 2026-11-04 | Firefox-Umsetzung in `../../FIREFOX_AddOns/HSLinkDrag` (github.com/HSBitbooster/HSLinkDrag-Firefox). Dieses Repository heißt jetzt `HSBitbooster/HSLinkDrag-Chrome`. |
| 2026-11-04 | Abgleich mit Firefox, beide als **1.0.4**: Rückfallweg, wenn die Seite das Ziehen abbricht (Googles Reiter), mit Karussell-Schutz; Markierter Text zählt nur bei Druck in der Markierung; Zieh-Bild des Browsers wird bei eingeschaltetem Hinweis ausgeblendet (`setDragImage`, in beiden Browsern). Die Live-Sprachumschaltung der Firefox-Version entfällt, weil Chrome die Sprache nur mit Neustart wechselt. Testseite/Testserver (ohne Cache, `scripts/serve.js`) wie bei Firefox. Logik-Test in headless Chrome mit simulierten APIs 6/6. |
| 2026-11-04 | Versionsregel: Chrome und Firefox immer gleiche Version und gleicher Stand. `scripts/check-version.js` bricht den Paketbau ab, wenn die Versionen abweichen; Git-Tag `v<version>` in beiden Repositorys. Chrome Web Store: 1.0.4 erst nach Freigabe von 1.0.3 einreichen. |
| 2026-11-04 | Manueller Test 1.0.4 (Runde 5): T35–T39 bestanden. Tag `v1.0.4` in beiden Repositorys. |
| 2026-11-05 | Chrome Web Store: 1.0.3 freigegeben und veröffentlicht, Eintrag per direktem Link erreichbar. Die Store-Suche findet ihn noch nicht. |
| 2026-11-05 | Store-Suche findet HSLinkDrag jetzt (Sichtbarkeit öffentlich, alle Regionen). Update auf 1.0.4 im Chrome Web Store eingereicht, Prüfung läuft. |
| 2026-11-05 | Version 1.0.4 im Chrome Web Store veröffentlicht, auf der öffentlichen Store-Seite geprüft. Öffentliche Kontaktadresse ist jetzt `hslinkdrag@web.de`. |
| 2026-11-05 | Marquee-Bild 1400 × 560 de/en erstellt (`store/scenes/marquee.html`, `scripts/store-images.sh`) als Vorbereitung für einen späteren Antrag auf das „Empfohlen“-Abzeichen. Antrag erst mit etwas mehr Nutzern und Bewertungen. |
| 2026-11-05 | Marquee-Bild englisch im Chrome Web Store hinterlegt (es gibt nur eines für alle Sprachen), erneute Prüfung des Eintrags eingereicht. |
| 2026-11-07 | Firefox-Version 1.0.4 auf addons.mozilla.org veröffentlicht; beide Browser sind in den Stores auf demselben Stand. |
| 2026-11-07 | Grafik mit `pointer-events: none` im Link (YouTube-Logo): SHIFT öffnet sie nicht. Eine Lösung per Suche an der Mausposition wäre möglich; der Nutzer lässt es bewusst so und nimmt dafür keine Version 1.0.5 samt neuer Store-Prüfungen in Kauf. |

## 10. Umsetzungsplan (Schritte 1–8 für Version 1.0.0 erledigt)

1. Grundgerüst: Manifest, Service-Worker, Content-Script, Icons (aus `icons/logo-original.png`
   per ImageMagick auf 16/32/48/128 px skaliert, plus graue Variante für „aus“);
   Link → neuer Tab rechts.
2. Richtung, Mindestdistanz, Tab-Position (inkl. Reihenfolge-Logik).
3. Bilder, Link-auf-Bild mit SHIFT.
4. Markierter Text: E-Mail-/URL-Erkennung, Suche (`detect.js` + Unit-Tests).
5. E-Mail-Client.
6. Optionsseite mit Erklärtexten.
7. Zusatzfunktionen gemäß offenen Fragen (Overlay, Ausschlussliste, Toolbar-Schalter, i18n).
8. Testseite + manueller Testdurchlauf, README mit Installationsanleitung.

## 11. Versionen Chrome ↔ Firefox

- Chrome- und Firefox-Version haben **immer dieselbe Versionsnummer und denselben
  Funktionsstand** (Vorgabe des Nutzers). Unterschiede nur dort, wo der Browser es verlangt
  (Firefox: Live-Sprachwechsel, Menüeintrag „Einstellungen“, Website-Zugriff, Tab „Am Ende“ –
  siehe `docs/KONZEPT.md` Abschnitt 3 im Firefox-Projekt).
- `scripts/check-version.js` vergleicht beim Paketbau (`scripts/build-zip.sh`) die
  Versionsnummer mit dem Firefox-Projekt im Nachbarordner und bricht bei Abweichung ab.
- Jede Version bekommt in **beiden** Repositorys den Git-Tag `v<version>` (z. B. `v1.0.4`).
- Funktionsänderungen werden immer in beiden Projekten umgesetzt und getestet.
- Store-Stand kann vorübergehend abweichen (Prüfung im Chrome Web Store dauert); der Stand
  auf GitHub ist immer gleich.
