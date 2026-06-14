# TED-Ausschreibungs-Monitoring (UiPath)

Umsetzung des RoboPath „TED-Ausschreibungs-Monitoring" für die fiktive
**Bavaria Tactical Trading GmbH** (HM, Informationssysteme 2).

UiPath-**Studio**-Projekt (VisualBasic, Target Framework **Windows**, getestet mit Studio **2026.0**).
`Main.xaml` ist ein **Flowchart**: 5 Datensystem-Schritte (Excel → Browser/TED → Dedup → Excel → Kalender/Mail),
ein **Human-in-the-Loop**-Gate (Go/No-Go je Treffer), zwei Entscheidungen, eine 24/7-Schleife und ein globaler Try/Catch.
Im Auslieferungszustand läuft alles **out of the box im Demo-Modus** — ohne Browser-Extension, ohne Orchestrator, ohne Mail-Konto.

## ⚡ Schnellstart (zum Verteilen / Weitergeben)
**Klonen → Projektordner in Studio öffnen → Run.** Mehr nicht.
1. Repo klonen (oder ZIP entpacken).
2. UiPath Studio (2023.4+ / 2026.0) → **Open** → den **Projektordner** wählen (die `project.json`).
   Studio bietet evtl. an, das Projekt auf deine Version zu aktualisieren → bestätigen. Die Abhängigkeiten
   werden automatisch aus dem offiziellen UiPath-Feed wiederhergestellt.
3. **Run / Debug (F5).** Der Bot läuft komplett durch: liest die Suchkriterien (Excel), spielt im Demo-Modus
   2 Beispiel-Treffer ein, dedupliziert gegen die Gesehen-Liste, schreibt `.ics`-Kalendereinträge nach `Output/`,
   protokolliert jeden Schritt und schleift. Nach dem ersten Durchlauf **Stop** drücken (der Flow schleift bewusst).

> **Wichtig:** Den **Projektordner** öffnen, in dem `project.json` und der Ordner `Data\` direkt nebeneinander liegen —
> **nicht** einen Eltern- oder doppelt verschachtelten Ordner. Die Datei-Pfade sind projektrelativ
> (`Data\suchkriterien.xlsx`) und funktionieren so auf jedem Rechner, solange die Projektstruktur stimmt.

### Demo-Modus vs. Echt-Betrieb (Stellschrauben im Flowchart-Variablen-Panel)
| Variable / Schritt | Demo (Standard) | Echt-Betrieb |
|---|---|---|
| `in_DemoFallback` | `True` → 2 Beispiel-Treffer, **kein** Browser nötig | `False` → echte TED-Suche im Browser. Dafür **UiPath-Browser-Extension** installieren (*Studio → Tools*); aufgenommene Selektoren ggf. gegen das aktuelle `ted.europa.eu` neu aufnehmen. |
| `PollIntervalMinutes` | `0` (Loop sofort, zum Vorführen) | z. B. `30` (alle 30 Min pollen) |
| Mail (Schritt 8) / `in_MailTo` | Mail wird versucht, scheitert **leise** (nur Warnung im Log), wenn keine Verbindung da ist | eigene **Office-365-** oder **SMTP-Verbindung** in Studio einrichten und `in_MailTo` setzen |
| Go/No-Go (Schritt 6) | **Demo-Auto-Approve** (Bot gibt automatisch frei) | echte Mensch-Entscheidung; `Forms/GoNoGo.json` ist als Freigabe-Formular (Buttons Freigeben/Ablehnen) vorbereitet |

## Inhalt
```
TED_Ausschreibungs_Monitoring/
├── project.json              UiPath-Projektdatei (Long-Running: supportsPersistence = true)
├── Main.xaml                 Flowchart (Daten-Schritte + Go/No-Go-Gate + 2 Entscheidungen + Schleife + Try/Catch)
├── Data/
│   ├── suchkriterien.xlsx    Eingabe: CPV-Codes, Land, Schwellenwert, Prioritaet, Aktiv-Flag
│   └── seen.xlsx             Gesehen-Liste (Dedup-Log), wird vom Bot befüllt
├── Forms/
│   └── GoNoGo.json           Freigabe-Formular (Buttons Freigeben/Ablehnen)
└── README.md
```

## Abhängigkeiten
Werden beim Öffnen automatisch von Studio aus dem offiziellen UiPath-Feed wiederhergestellt:
- `UiPath.Excel.Activities` — Schritte 1 & 5 (`suchkriterien.xlsx` / `seen.xlsx`, klassische Workbook-Aktivitäten)
- `UiPath.UIAutomation.Activities` — Schritte 2–4 (TED im Browser; nur im Echt-Betrieb aktiv)
- `UiPath.Mail.Activities` + `UiPath.MicrosoftOffice365.Activities` — Schritt 8 (Mail)
- `UiPath.System.Activities` — Kern (Flow, Logging, `.ics`-Datei schreiben)

> Hinweis StudioX: Das ist ein **Studio**-Projekt (klassische Aktivitäten + Flowchart) und wird in derselben
> UiPath-Studio-App geöffnet. Logik, Variablen und CPV-Mechanik sind identisch zu einem StudioX-Nachbau.

## Ablauf (Soll)
1. **Suchkriterien lesen** → aktive CPV-Zeilen filtern (`Aktiv = Ja`).
2.–4. **TED durchsuchen** (Browser, je CPV) → Treffer extrahieren → gegen `seen.xlsx` **deduplizieren**
   (im Demo-Modus durch 2 Beispiel-Treffer ersetzt, damit es ohne Live-TED durchläuft).
5. **Gesehen-Liste** (`seen.xlsx`) um die neuen Notice-IDs ergänzen.
6. **Go/No-Go** je neuem Treffer (Demo-Auto-Approve bzw. echte Freigabe über `GoNoGo.json`).
7. Pro freigegebenem Treffer ein **`.ics`-Kalendereintrag** (VEVENT mit Abgabefrist + 2-Tage-Alarm) nach `Output/`.
8. **Freigabe-Mail** (HTML-Tabelle der freigegebenen Treffer) — sofern Mail-Verbindung konfiguriert, sonst Warnung.
9. **Logging** der Lauf-Zusammenfassung; abgelehnte Treffer separat (9b).
→ Warte-/Poll-Intervall → zurück zu Schritt 1. Globaler Try/Catch eskaliert bei Crash (Log + Crash-Mail).

## Suchkriterien (`Data/suchkriterien.xlsx`)
Die CPV-Codes sind auf das Sortiment der Bavaria Tactical Trading GmbH zugeschnitten –
**Surplus military equipment, Munition und taktische Ausruestung**. Abgedeckt sind vier Bloecke:

- **Waffen & Munition** (35300000 Waffen/Munition, 35320000 Schusswaffen, 35330000 Munition, 35340000 Teile, 35310000 sonstige Waffen)
- **Persoenliche / taktische Ausruestung & Koerperschutz** (35800000/35810000 Ausruestung, 35815000/35815100 ballistischer Schutz & kugelsichere Westen, 35813000 Helme, 35811300/35812000 Uniformen)
- **Militaerfahrzeuge** (35400000 Fahrzeuge, 35420000 Teile, 35410000 gepanzert)
- **Taktische Bekleidung & Schuhe** (35113400 Schutzkleidung, 18100000 Spezialarbeitskleidung, 18800000 Einsatzstiefel)

Drei Spalten steuern die Logik des Bots und sind die Stellschrauben fuer die Auswertung:

| Spalte | Wirkung |
|---|---|
| `Aktiv` (Ja/Nein) | Nur `Ja`-Zeilen werden gescannt. `Nein` = bewusst ausserhalb Sortiment (z. B. gepanzerte Fahrzeuge zu gross, Gasmasken/See-Munition off-topic) – zeigt den Filter. |
| `Min_Auftragswert_EUR` | Untergrenze als Rauschfilter; Treffer darunter werden verworfen. |
| `Prioritaet` (Hoch/Mittel/Niedrig) | Sortiert bzw. hebt Treffer in der Mail hervor (Kerngeschaeft Waffen/Munition/Koerperschutz = `Hoch`). |

`Land`: `DE`/`AT` filtern auf das Auftraggeber-Land, `EU` = EU-weit (kein Laenderfilter).
Die Werte sind plausible Beispiele und in der xlsx frei anpassbar.

> **CPV-Stand:** Aktuell ist weiterhin **CPV 2008** (Verordnung (EG) Nr. 213/2008) – das ist die
> gueltige Version fuer TED-Suche und Notice-Ausfuellung; eine neuere CPV-Fassung ist (Stand 2026)
> nicht veroeffentlicht.

## Hinweise zum Echt-Betrieb (TED-Browser, `in_DemoFallback = False`)
- **UiPath-Browser-Extension** installieren (Studio → Tools), sonst lässt sich die Seite nicht ansteuern.
- TED ist eine **dynamische Web-App** → Selektoren live aufnehmen, mit *Check App State* auf geladene Elemente
  warten statt mit festen Delays. Cookie-Banner als optionalen Klick behandeln.
- Treffer sind **Karten/Listeneinträge** → den *Daten-extrahieren*-Assistenten auf die ersten Trefferkarten
  anwenden; Paginierung aktivieren.
- Steht die **Abgabefrist** nicht in der Trefferliste, pro Treffer kurz die Detailseite öffnen und sie auslesen
  (wird für den `.ics`-Eintrag in Schritt 7 gebraucht).
- Scheitert der Browser-Schritt (Extension fehlt / Selektor veraltet), fängt ein Try/Catch das ab und der Lauf
  bricht **nicht** ab — bei `in_DemoFallback = True` werden ersatzweise die Beispiel-Treffer eingespielt.

## Human-in-the-Loop / Action Center (Hinweis)
Das echte **UiPath Action Center** (Create Form Task / Wait for Form Task and Resume, Paket
`UiPath.Persistence.Activities`) ist mit diesem **Windows-Target-Projekt nicht kompatibel** und würde zudem eine
laufende Orchestrator-Verbindung voraussetzen. Daher ist die Freigabe als robuster **Demo-Auto-Approve**
modelliert, der lokal und zuverlässig durchläuft. Das `Forms/GoNoGo.json` ist als echtes Freigabe-Formular
vorbereitet und kann bei Bedarf über die lokale Aktivität **„Show Form"** (`UiPath.Form.Activities`) eingebunden
werden, um die Entscheidung attended (lokal, ohne Orchestrator) durch einen Menschen treffen zu lassen.
