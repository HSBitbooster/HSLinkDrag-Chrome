# HSLinkDrag

Chrome-Erweiterung: Links, Bilder und markierten Text per **Click & Drag** in einem
neuen Tab öffnen. Die Zugrichtung (links/rechts) bestimmt, ob der Tab im Vorder- oder
Hintergrund geöffnet wird.

Firefox-Version mit gleicher Versionsnummer und gleichem Funktionsstand:
[HSLinkDrag-Firefox](https://github.com/HSBitbooster/HSLinkDrag-Firefox), Installation über [addons.mozilla.org](https://addons.mozilla.org/de/firefox/addon/hslinkdrag/).

## Installation

Aus dem Chrome Web Store: [HSLinkDrag](https://chromewebstore.google.com/detail/hslinkdrag/pilepemcdicgnpdbilcoiieediljfkne) → „Zu Chrome hinzufügen“.

## Installation (entpackt, zum Entwickeln)

1. In Chrome `chrome://extensions` öffnen.
2. Oben rechts den **Entwicklermodus** einschalten.
3. **„Entpackte Erweiterung laden“** → diesen Ordner (`HSLinkDrag`) auswählen.
4. Optional: In den Details der Erweiterung **„Zugriff auf Datei-URLs zulassen“**
   aktivieren, damit sie auch auf lokalen Dateien (`file://`) funktioniert.

Nach Codeänderungen auf der Erweiterungskarte auf ⟳ klicken **und offene Tabs neu laden**
(bereits offene Seiten behalten sonst das alte Content-Script).

Bereits beim Installieren offene Tabs müssen einmal neu geladen werden, bevor die
Geste dort funktioniert.

## Bedienung

| Aktion | Ergebnis (Standardeinstellung) |
|---|---|
| Link/Bild/markierten Text nach **rechts** ziehen | Neuer Tab im **Vordergrund** |
| … nach **links** ziehen | Neuer Tab im **Hintergrund** |
| Verlinktes Bild oder SVG ziehen, beim Loslassen **SHIFT** halten | Bild statt Link öffnen |
| Markierte Internetadresse ziehen | Adresse öffnen statt suchen |
| E-Mail-Link oder markierte E-Mail-Adresse ziehen | E-Mail-Programm öffnet sich |
| **ESC** während des Ziehens | Abbruch |
| Klick auf das Symbol in der Symbolleiste | Erweiterung an/aus |
| Rechtsklick auf das Symbol → **Optionen** | Einstellungen |

## Projektstruktur

```
manifest.json            Erweiterungs-Manifest (MV3)
src/background.js        Service-Worker: Tabs öffnen, E-Mail, Toolbar-Schalter
src/content/content.js   Geste erkennen, Hinweis am Mauszeiger
src/shared/settings.js   Standardwerte, Ausschlussliste
src/shared/detect.js     Erkennung URL / E-Mail / Suche, Such-URLs
options/                 Optionsseite
viewer/                  Anzeigeseite für per Drag geöffnete SVG-Grafiken
_locales/de, _locales/en Texte Deutsch/Englisch
icons/                   Icons (logo-original.png ist die Quelle)
test/                    Unit-Tests und manuelle Testseite
scripts/                 ZIP-Paket, Store-Bilder, Testserver, Versionsabgleich mit Firefox
store/                   Chrome-Web-Store-Texte, Datenschutzerklärung, Bilder
docs/KONZEPT.md          Konzept, Entscheidungen, Status
docs/TESTPLAN.md         Manuelle Test-Checkliste
```

## Tests

Unit-Tests der Erkennung (Node ≥ 18, keine Abhängigkeiten):

```bash
node --test test/
```

Manuelle Tests: `test/testseite.html` über den Testserver öffnen (ohne Cache, damit nach
Änderungen keine alten Testdateien erscheinen) und `docs/TESTPLAN.md` abarbeiten:

```bash
node scripts/serve.js
```

Danach im Browser `http://localhost:8765/test/testseite.html` aufrufen.

## Icons neu erzeugen

```bash
cd icons && for s in 16 32 48; do convert logo-original.png -filter Lanczos -resize ${s}x${s} icon-$s.png; convert logo-original.png -filter Lanczos -resize ${s}x${s} -colorspace Gray -channel A -evaluate multiply 0.6 +channel icon-$s-off.png; done
```

```bash
cd icons && convert logo-original.png -filter Lanczos -resize 96x96 -background none -gravity center -extent 128x128 icon-128.png
```

Das 128er-Icon hat nach den Store-Richtlinien 16 px transparenten Rand (Grafik 96 × 96).

## Chrome Web Store

ZIP-Paket bauen (prüft vorher, dass die Firefox-Version dieselbe Versionsnummer hat, und führt die Unit-Tests aus) → `dist/HSLinkDrag-<version>.zip`:

```bash
scripts/build-zip.sh
```

Store-Bilder neu erzeugen (headless Chrome) → `store/images/`:

```bash
scripts/store-images.sh
```

Alle Texte zum Kopieren ins Entwickler-Dashboard: `store/STORE-EINTRAG.md`,
Datenschutzerklärung: `store/DATENSCHUTZ.md`.

## Lizenz

MIT – siehe [LICENSE](LICENSE). Nutzung, Änderung und Weitergabe erlaubt, solange der
Urheberhinweis erhalten bleibt; keine Gewährleistung.
