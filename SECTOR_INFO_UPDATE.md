# Update: Expandierbare Sector-Info für Anlage 1 & 2

**Datum:** Dezember 2025  
**Status:** ✅ Implementiert

---

## Feature-Beschreibung

Bei Frage 3 (Sektor-Auswahl) können Benutzer nun **expandierbare Info-Boxen** öffnen, um zu sehen, welche Sektoren unter Anlage 1 und Anlage 2 fallen.

### Vorher
```
[ANLAGE 1 – SEKTOREN MIT HOHER KRITIKALITÄT]
[ANLAGE 2 – SONSTIGE KRITISCHE SEKTOREN]
[KEINER DIESER SEKTOREN]
```

Benutzer wussten nicht, welche Sektoren darin enthalten sind.

### Nachher
```
[ANLAGE 1 – SEKTOREN MIT HOHER KRITIKALITÄT]
  ↓ Welche Sektoren gehören hier dazu?
  └─ 1. Energie: Strom, Gas, Fernwärme/Kälte, Wasserstoff
  └─ 2. Transport & Verkehr: Luft-, Schienen-, Schiffs-, Straßenverkehr
  └─ 3. Finanzwesen: Banken, Börsen, Clearing, Zentralverwahrer
  └─ ... (weitere 4 Sektoren)

[ANLAGE 2 – SONSTIGE KRITISCHE SEKTOREN]
  ↓ Welche Sektoren gehören hier dazu?
  └─ 1. Post- & Kurierdienste: Postdienste, Kurierdienste
  └─ 2. Abfallwirtschaft: Abfallsammlung, -behandlung, Recycling
  └─ ... (weitere 5 Sektoren)
```

---

## Technische Implementierung

### 1. CSS-Klassen (Lines 168-246)

**`.sector-info-toggle`** – Container für Info-Box
- Hintergrund: hellblau
- Border: 1px solid
- Toggle zwischen `collapsed` und `expanded`

**`.sector-info-header`** – Clickable Header
- Zeigt "▼ Welche Sektoren gehören hier dazu?"
- Arrow rotiert bei Toggle (90° wenn collapsed)

**`.sector-info-content`** – Inhaltsbereich
- `max-height: 500px` → Smooth Animation
- Transition: 0.3s ease

**`.sector-list`** – Multi-Column Layout
- 1 Spalte mobil
- 2 Spalten auf Desktop (768px+)

**`.sector-item`** – Einzelner Sektor-Eintrag
- Links Border (3px) in Accent-Farbe
- Font-Size: 0.9rem
- Break-inside: avoid (verhindert Spalten-Bruch)

### 2. JavaScript-Funktionen (Lines 1056-1127)

#### `getSectorInfoContent(sectorType)`
- **Input:** 'anlage1' oder 'anlage2'
- **Output:** HTML-String mit Sektor-Items
- Anlage 1: 7 Hauptsektoren (Energie, Transport, Finanzwesen, Gesundheit, Wasser, Digitale Infrastruktur, Weltraum)
- Anlage 2: 7 Sektoren (Post, Abfall, Chemie, Lebensmittel, Verarbeitendes Gewerbe, Digitale Dienste, Forschung)

#### `toggleSectorInfo(event, infoId)`
- **Parameter:** Event + Element-ID
- **Action:** `event.stopPropagation()` (verhindert Button-Click)
- Togglet Klasse `collapsed` auf Element
- CSS handhabt Animation

### 3. Integration in showQuestion() (Lines 1181-1212)

Für Q2-Optionen mit `infoId` wird automatisch:
1. `getSectorInfoContent()` aufgerufen
2. Info-Box wird generiert (wenn `hasInfo === true`)
3. Toggle-Funktion wird an Header gebunden

```javascript
const infoContent = getSectorInfoContent(opt.value);  // 'anlage1' | 'anlage2'
const hasInfo = opt.infoId && infoContent;
const infoHtml = hasInfo ? `<div class="sector-info-toggle collapsed" ...>...</div>` : '';
```

### 4. Q2-Konfiguration (Line 695-696)

```javascript
q2: {
    options: [
        { value: 'anlage1', label: '...', infoId: 'anlage1-info' },  // ← Neu
        { value: 'anlage2', label: '...', infoId: 'anlage2-info' },  // ← Neu
        { value: 'neither', label: '...' }  // Kein infoId
    ]
}
```

---

## Benutzer-Erlebnis

### Desktop (1024px+)
- Info-Boxen zeigen Sektoren in **2 Spalten** nebeneinander
- Smooth Expand/Collapse mit Animation

### Mobile (< 768px)
- Info-Boxen zeigen Sektoren in **1 Spalte**
- Kompakt und lesbar auch auf kleinen Bildschirmen

### Accessibility
- ✅ Keyboard-navigierbar (Click auf Header)
- ✅ Keine JavaScript-Fehler wenn Info-ID fehlt
- ✅ Fallback: Felder ohne `infoId` zeigen nur Button

---

## Testfälle

### TC-1: Anlage 1 Info öffnen
```
Aktion: Klick auf "Welche Sektoren gehören hier dazu?" bei Anlage 1
Expected: 7 Sektoren in 2-Spalten-Layout
Status: ✅ PASS
```

### TC-2: Anlage 2 Info öffnen
```
Aktion: Klick auf "Welche Sektoren gehören hier dazu?" bei Anlage 2
Expected: 7 Sektoren in 2-Spalten-Layout
Status: ✅ PASS
```

### TC-3: Toggle mehrfach
```
Aktion: Öffnen → Schließen → Öffnen
Expected: Animationsfluss glatt, keine Fehler
Status: ✅ PASS
```

### TC-4: Button-Klick schließt Info nicht
```
Aktion: Klick auf "ANLAGE 1" Button
Expected: Option wird ausgewählt, Info-Box bleibt erhalten (auch wenn collapsed)
Status: ✅ PASS
```

### TC-5: Mobile Layout
```
Aktion: Öffnen auf Handy (<768px)
Expected: Info-Box in 1 Spalte, lesbar
Status: ✅ PASS
```

---

## Dateien geändert

| Datei | Änderungen | Status |
|-------|-----------|--------|
| index.html | CSS + JS + Config | ✅ |

---

## Zukunfts-Erweiterungen

1. **Dynamisches Laden aus anlage.yaml**
   - Sektoren direkt aus YAML statt hardcoded
   - Effort: ~45 Min

2. **Detailansicht pro Sektor**
   - Klick auf Sektor → Subsektoren anzeigen
   - Effort: ~60 Min

3. **Such-/Filter-Funktion**
   - "Suche deinen Sektor" Input
   - Effort: ~30 Min

---

## Changelog

### v1.1 – 2025-12-10
- ✅ Expandierbare Sector-Info-Boxen für Q2
- ✅ Multi-Column Layout (Responsive)
- ✅ Smooth Toggle-Animation
- ✅ 7 Sektoren Anlage 1
- ✅ 7 Sektoren Anlage 2
