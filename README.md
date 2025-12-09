# enerkii Price Signal v2

## Dokumentation

---

## 1. Übersicht

### 1.1 Zweck der Applikation

enerkii Price Signal ist ein Benachrichtigungssystem für Strompreise auf dem deutschen Day-Ahead-Markt. Die Applikation ermöglicht es Unternehmen, individuelle Preisschwellwerte zu definieren und automatisierte E-Mail-Benachrichtigungen zu erhalten, wenn diese Schwellwerte am Folgetag über- oder unterschritten werden.

### 1.2 Zielgruppe

- Energieintensive Unternehmen (Produktion, Fertigung)
- Unternehmen mit flexiblen Produktionsprozessen
- Energiemanager und Einkäufer
- Unternehmen, die ihren Strombezug optimieren möchten

### 1.3 Kernnutzen

| Nutzen | Beschreibung |
|--------|--------------|
| **Kostenoptimierung** | Verlagerung energieintensiver Prozesse in Niedrigpreisphasen |
| **Planungssicherheit** | Frühzeitige Information über Hochpreisphasen am Folgetag |
| **Automatisierung** | Keine manuelle Marktbeobachtung erforderlich |
| **Individualisierung** | Persönliche Schwellwerte je nach Unternehmensbedarf |

---

## 2. Funktionsweise

### 2.1 Systemarchitektur

```
┌─────────────────────────────────────────────────────────────────┐
│                         WEBSEITE                                 │
│                   (enerkii.com/strompreisalarm)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  Formular-Component                        │  │
│  │                                                            │  │
│  │  • Kundendaten erfassen                                    │  │
│  │  • Preisschwellwerte konfigurieren                         │  │
│  │  • Historische Preisdaten visualisieren                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         n8n BACKEND                              │
│                                                                  │
│  ┌─────────────────────┐      ┌─────────────────────────────┐   │
│  │ GET /price-history  │      │ POST /signup                │   │
│  │                     │      │                             │   │
│  │ Liefert historische │      │ Speichert Kundenanmeldung   │   │
│  │ Preisdaten für      │      │ und Preisschwellwerte       │   │
│  │ Visualisierung      │      │                             │   │
│  └─────────────────────┘      └─────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SCHEDULER (täglich 14:00 Uhr)                           │    │
│  │                                                          │    │
│  │ 1. Abruf Day-Ahead-Preise von ENTSO-E                   │    │
│  │ 2. Vergleich mit Kundenschwellwerten                    │    │
│  │ 3. Versand personalisierter E-Mail-Benachrichtigungen   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DATENQUELLEN                               │
│                                                                  │
│  ┌─────────────────────┐      ┌─────────────────────────────┐   │
│  │ ENTSO-E API         │      │ PostgreSQL (Supabase)       │   │
│  │                     │      │                             │   │
│  │ Day-Ahead-Preise    │      │ • Historische Preisdaten    │   │
│  │ für DE-LU Markt     │      │ • Kundenstammdaten          │   │
│  └─────────────────────┘      └─────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Datenfluss

#### Anmeldeprozess

1. Kunde öffnet das Formular auf der enerkii-Webseite
2. Historische Preisdaten werden geladen und visualisiert
3. Kunde gibt persönliche Daten ein
4. Kunde konfiguriert individuelle Preisschwellwerte
5. Interaktive Vorschau zeigt, wie oft Benachrichtigungen ausgelöst worden wären
6. Kunde sendet Anmeldung ab
7. Daten werden im Backend gespeichert
8. Kunde erhält ab dem Folgetag automatisierte Benachrichtigungen

#### Täglicher Benachrichtigungsprozess

1. Täglich um 14:00 Uhr werden die Day-Ahead-Preise für den Folgetag abgerufen
2. System vergleicht die 96 Viertelstunden-Preise mit den Kundenschwellwerten
3. Bei Überschreitung: Zeiträume mit hohen Preisen werden identifiziert
4. Bei Unterschreitung: Zeiträume mit niedrigen Preisen werden identifiziert
5. Personalisierte E-Mail wird an jeden betroffenen Kunden versendet
6. E-Mail enthält konkrete Zeitfenster und Handlungsempfehlungen

---

## 3. API-Schnittstellen

### 3.1 GET Price History

**Endpunkt:** `https://enerkii.app.n8n.cloud/webhook/webhook_get_price_history_day_ahead`

**Methode:** GET

**Zweck:** Abruf historischer Day-Ahead-Preisdaten für die Visualisierung im Frontend

**Antwortformat:**

```json
[
  {
    "date": "2025-06-12T00:00:00.000Z",
    "min": "-27.90",
    "max": "132.00",
    "avg": "53.86"
  },
  {
    "date": "2025-06-13T00:00:00.000Z",
    "min": "-6.09",
    "max": "111.19",
    "avg": "52.91"
  }
]
```

**Felderbeschreibung:**

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `date` | ISO 8601 Datum | Tag der Preisdaten |
| `min` | String (Dezimal) | Minimaler Preis des Tages in €/MWh |
| `max` | String (Dezimal) | Maximaler Preis des Tages in €/MWh |
| `avg` | String (Dezimal) | Durchschnittspreis des Tages in €/MWh |

**Datenumfang:** Letzte 180 Tage

**Datenquelle:** ENTSO-E Transparency Platform, Marktgebiet DE-LU (EIC: 10Y1001A1001A82H)

---

### 3.2 POST Signup

**Endpunkt:** `https://enerkii.app.n8n.cloud/webhook/signup`

**Methode:** POST

**Content-Type:** `application/json`

**Zweck:** Registrierung eines neuen Kunden für den Preisalarm-Service

**Request Body:**

```json
{
  "vorname": "Max",
  "name": "Mustermann",
  "geschlecht": "Herr",
  "unternehmen": "Muster GmbH",
  "mail": "max@muster.de",
  "telefon": "+49 123 456789",
  "upper_price_signal": 80,
  "lower_price_signal": 10
}
```

**Felderbeschreibung:**

| Feld | Typ | Pflicht | Beschreibung |
|------|-----|---------|--------------|
| `vorname` | String | Ja | Vorname des Ansprechpartners |
| `name` | String | Ja | Nachname des Ansprechpartners |
| `geschlecht` | String | Nein | Anrede (Herr/Frau/Divers) |
| `unternehmen` | String | Nein | Firmenname |
| `mail` | String | Ja | E-Mail-Adresse für Benachrichtigungen |
| `telefon` | String | Nein | Telefonnummer |
| `upper_price_signal` | Number | Ja | Oberer Schwellwert in €/MWh |
| `lower_price_signal` | Number | Ja | Unterer Schwellwert in €/MWh |

**Erfolgreiche Antwort:** HTTP 200 OK

**Schwellwert-Bereiche:**

| Schwellwert | Minimum | Maximum | Standardwert |
|-------------|---------|---------|--------------|
| Oberer Schwellwert | 50 €/MWh | 200 €/MWh | 80 €/MWh |
| Unterer Schwellwert | -20 €/MWh | 50 €/MWh | 10 €/MWh |

---

## 4. Design und Aufbau

### 4.1 Gesamtlayout

Das Formular ist als zweispaltiges Layout konzipiert:

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐ │
│  │                         │  │                             │ │
│  │     FORMULAR            │  │     PREISCHART              │ │
│  │                         │  │                             │ │
│  │  • Kundendaten          │  │  • 180-Tage-Verlauf         │ │
│  │  • Schwellwert-Slider   │  │  • Min/Max/Avg pro Tag      │ │
│  │  • Alert-Statistik      │  │  • Schwellwert-Linien       │ │
│  │  • Anmelde-Button       │  │  • Interaktive Tooltips     │ │
│  │                         │  │                             │ │
│  └─────────────────────────┘  └─────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Responsives Verhalten:** Auf mobilen Geräten (< 768px Bildschirmbreite) wird nur die linke Spalte mit dem Formular angezeigt. Der Chart entfällt zugunsten einer optimierten mobilen Nutzererfahrung.

### 4.2 Farbschema

Das Design verwendet das offizielle enerkii-Farbschema:

| Farbe | Hex-Code | Verwendung |
|-------|----------|------------|
| **enerkii Green** | `#1C754F` | Primärfarbe, Buttons, Chart-Durchschnittslinie |
| **Mid Green** | `#55977B` | Hover-Zustände, sekundäre Elemente |
| **Light Green** | `#8FBAA8` | Hintergründe, Chart-Maximum-Fläche |
| **White** | `#FFFFFF` | Haupthintergrund, Karten |
| **Light Grey** | `#FAFAFA` | Sekundärer Hintergrund, Chart-Bereich |
| **Grey** | `#A4A4A4` | Beschriftungen, Hilfstexte |
| **Dark Grey** | `#4E4E4E` | Formular-Labels, Sekundärtext |
| **Black** | `#222222` | Überschriften, Haupttext |
| **Yellow** | `#FEC240` | Chart-Minimum-Linie, Akzente |
| **Light Yellow** | `#FFE09F` | Chart-Minimum-Fläche |
| **Blue** | `#4E71EE` | Unterer Schwellwert, "Tage unter" Statistik |

### 4.3 Formular-Bereich (Linke Spalte)

#### Kundendaten-Sektion

**Überschrift:** "Ihre Daten"

**Felder:**

| Feld | Typ | Layout |
|------|-----|--------|
| Vorname / Nachname | Text | Zweispaltig |
| Anrede | Dropdown | Einspaltig |
| Unternehmen | Text | Einspaltig |
| E-Mail / Telefon | Text | Zweispaltig |

**Pflichtfelder** sind mit einem Asterisk (*) gekennzeichnet.

#### Preisschwellwerte-Sektion

**Überschrift:** "Preisschwellwerte"

Diese Sektion ist visuell durch eine horizontale Linie vom Datenbereich getrennt.

**Oberer Schwellwert:**
- Slider von 50 bis 200 €/MWh
- Anzeige in €/MWh und ct/kWh (umgerechnet)
- Akzentfarbe: enerkii Green

**Unterer Schwellwert:**
- Slider von -20 bis 50 €/MWh
- Anzeige in €/MWh und ct/kWh (umgerechnet)
- Akzentfarbe: Blue

#### Alert-Statistik

Zwei nebeneinander angeordnete Statistik-Karten:

**Karte 1 – "Tage über Schwellwert":**
- Große Zahl in enerkii Green
- Rahmen in Light Green
- Zeigt Anzahl der Tage, an denen der obere Schwellwert überschritten wurde

**Karte 2 – "Tage unter Schwellwert":**
- Große Zahl in Blue
- Rahmen in Light Yellow
- Zeigt Anzahl der Tage, an denen der untere Schwellwert unterschritten wurde

**Hinweistext:** "Basierend auf den letzten [X] Tagen"

Die Statistiken aktualisieren sich in Echtzeit beim Verschieben der Schwellwert-Slider.

#### Anmelde-Button

- Volle Breite
- enerkii Green Hintergrund
- Weißer Text: "Kostenlos anmelden"
- Hover-Effekt: Dunkleres Grün
- Deaktiviert während des Sendevorgangs

### 4.4 Chart-Bereich (Rechte Spalte)

#### Header

**Überschrift:** "Day-Ahead Strompreise"

**Untertitel:** "Letzte [X] Tage · DE-LU Marktgebiet"

#### Diagramm

**Typ:** Kombiniertes Flächen- und Liniendiagramm

**Zeitraum:** 180 Tage

**Dargestellte Daten:**

| Element | Farbe | Darstellung |
|---------|-------|-------------|
| Maximum | Light Green (Fläche), Mid Green (Linie) | Gefüllte Fläche, 40% Opazität |
| Minimum | Light Yellow (Fläche), Yellow (Linie) | Gefüllte Fläche, 50% Opazität |
| Durchschnitt | enerkii Green | Durchgezogene Linie, 2px |
| Oberer Schwellwert | enerkii Green | Gestrichelte Linie |
| Unterer Schwellwert | Blue | Gestrichelte Linie |

**Achsen:**

- X-Achse: Datum (Format: TT.MM)
- Y-Achse: Preis in €/MWh (Bereich: -50 bis 200)

**Interaktivität:**

- Tooltip bei Hover zeigt Datum, Minimum, Maximum und Durchschnitt
- Schwellwert-Linien bewegen sich bei Slider-Änderung

#### Legende

Horizontal zentriert unter dem Chart:

- 🟢 Durchschnitt
- 🟢 Maximum (hellere Tönung)
- 🟡 Minimum

#### Hinweistext

"Bewegen Sie die Regler links, um zu sehen wie oft Sie benachrichtigt worden wären."

### 4.5 Erfolgsmeldung

Nach erfolgreicher Anmeldung wird das gesamte Formular durch eine Erfolgsmeldung ersetzt:

```
┌────────────────────────────────────────┐
│                                        │
│              ✓ (grüner Kreis)          │
│                                        │
│       Anmeldung erfolgreich!           │
│                                        │
│   Sie erhalten ab morgen Ihre          │
│   personalisierten Strompreis-         │
│   Signale per E-Mail.                  │
│                                        │
└────────────────────────────────────────┘
```

### 4.6 Ladezustände

**Initiales Laden:**
- Während die Preisdaten geladen werden, erscheint ein animierter Spinner
- Text: "Lade Preisdaten..."

**Formular-Versand:**
- Button-Text ändert sich zu "Wird gesendet..."
- Button ist deaktiviert (nicht klickbar)
- Cursor zeigt "not-allowed"

---

## 5. E-Mail-Benachrichtigungen

### 5.1 Auslöser

Eine E-Mail wird versendet, wenn für den Folgetag mindestens eine der folgenden Bedingungen erfüllt ist:

1. Mindestens eine Viertelstunde überschreitet den oberen Schwellwert
2. Mindestens eine Viertelstunde unterschreitet den unteren Schwellwert

### 5.2 Versandzeitpunkt

Täglich um ca. 14:00 Uhr MEZ/MESZ (nach Veröffentlichung der Day-Ahead-Preise durch ENTSO-E)

### 5.3 E-Mail-Inhalt

**Betreff:** `⚡ Strompreis-Signal für [Datum]: [X] Stunden über [Y] ct/kWh, [Z] Stunden unter [W] ct/kWh`

**Inhalt:**

```
Guten Tag [Anrede] [Nachname],

für morgen ([Datum]) haben wir relevante Strompreisbewegungen 
für [Unternehmen] identifiziert:

🔴 HOHE PREISE ([X] Stunden über [Y] ct/kWh):
   • [Uhrzeit] Uhr: [Preis] ct/kWh
   • [Uhrzeit] Uhr: [Preis] ct/kWh
   ...

   → Empfehlung: Stromintensive Prozesse wenn möglich verschieben.

🟢 NIEDRIGE PREISE ([X] Stunden unter [Y] ct/kWh):
   • [Uhrzeit] Uhr: [Preis] ct/kWh
   • [Uhrzeit] Uhr: [Preis] ct/kWh
   ...

   → Empfehlung: Idealer Zeitraum für energieintensive Tätigkeiten.

Bei Fragen stehen wir Ihnen gerne zur Verfügung.

Mit freundlichen Grüßen
Ihr enerkii-Team

--
enerkii GmbH
www.enerkii.de
```

### 5.4 Preisumrechnung

In den E-Mails werden Preise in **ct/kWh** angegeben (Umrechnung: €/MWh ÷ 10), da diese Einheit für Endkunden verständlicher ist.

Zeitangaben werden von Viertelstunden in **Stunden** aggregiert (z.B. "1,5 Stunden" statt "6 Viertelstunden").

---

## 6. Datengrundlage

### 6.1 ENTSO-E Transparency Platform

**Datenquelle:** European Network of Transmission System Operators for Electricity

**Dokumenttyp:** A44 (Day-Ahead Prices)

**Marktgebiet:** DE-LU (Deutschland-Luxemburg)
- EIC-Code: `10Y1001A1001A82H`

**Zeitauflösung:** 15 Minuten (96 Werte pro Tag)

**Zeitzone:** UTC (Umrechnung auf Europe/Berlin für Anzeige und E-Mails)

### 6.2 Preischarakteristik

Day-Ahead-Preise können:
- Negativ werden (bei Überproduktion erneuerbarer Energien)
- Stark schwanken (zwischen -50 und 500+ €/MWh)
- Saisonale Muster aufweisen (höher im Winter, niedriger im Sommer)
- Tageszeitliche Muster aufweisen (höher morgens und abends)

---

## 7. Limitierungen

### 7.1 Funktionale Einschränkungen

- Maximal eine E-Mail pro Kunde pro Tag
- Keine SMS- oder Push-Benachrichtigungen
- Keine Änderung der Schwellwerte nach Anmeldung (Neuanmeldung erforderlich)

### 7.2 Dateneinschränkungen

- Preisdaten nur für DE-LU Marktgebiet verfügbar
- Historische Daten auf 180 Tage begrenzt
- Day-Ahead-Preise sind Großhandelspreise (ohne Netzentgelte, Steuern, etc.)

---

## 8. Glossar

| Begriff | Definition |
|---------|------------|
| **Day-Ahead-Markt** | Großhandelsmarkt, auf dem Strom für den Folgetag gehandelt wird |
| **ENTSO-E** | Europäischer Verband der Übertragungsnetzbetreiber |
| **EIC-Code** | Energy Identification Code, europaweit eindeutige Kennung |
| **DE-LU** | Deutsch-Luxemburgische Gebotszone im europäischen Strommarkt |
| **€/MWh** | Euro pro Megawattstunde (Großhandelseinheit) |
| **ct/kWh** | Cent pro Kilowattstunde (Endkundeneinheit, €/MWh ÷ 10) |
| **Schwellwert** | Individuell definierter Preisgrenzwert für Benachrichtigungen |

---

## 9. Versionierung

| Version | Datum | Änderungen |
|---------|-------|------------|
| v1.0 | Dezember 2025 | Initiale Version mit Basis-Funktionalität |
| v2.0 | Dezember 2025 | Neue API-Struktur, interaktive Visualisierung, verbessertes Design |

---

*Dokumentation erstellt: Dezember 2025*
*enerkii GmbH*