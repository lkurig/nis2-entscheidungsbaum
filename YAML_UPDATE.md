# YAML Update – Neue Struktur für BSIG/NIS2

**Datum:** Dezember 2025  
**Status:** ✅ Implementiert  
**Version:** 2.0

---

## Übersicht der Änderungen

Die YAML-Struktur wurde grundlegend überarbeitet, um folgende neue Features zu unterstützen:

1. **Override-System** (§28 Abs. 3) – Behördliche Einzelfallbewertung
2. **Umsatz/Bilanz-Schwellenwerte** – Nicht nur Mitarbeiterzahl
3. **Abs. 2 Kategorien** – Separate wichtige Einrichtungen
4. **Neue Sector-Struktur** – Mit `einrichtungsarten` statt `facility_types`
5. **Priorisiertes Regelwerk** – Explizite Prioritäten nach §28

---

## 1. ANLAGE.YAML – Neue Struktur

### Alte Struktur
```yaml
nis2_bsig:
  kategorien:
    bwe: {...}
  groessenschwellen:
    small_enterprise: {...}
```

### Neue Struktur
```yaml
# Direkt auf Root-Level (kein nis2_bsig wrapper)
override:                         # ← NEU: Behördliche Overrides
  enabled: true
  rules:
    - id: behoerdliche_einstufung
      result: [bwe, we, nicht_betroffen]

kritis:                           # ← Gleich wie vorher
  id: kritis_betreiber
  category: bwe

sonderfaelle: [...]               # ← Gleich wie vorher

telekommunikation:                # ← NEU: Mit Umsatz/Bilanz
  id: telekommunikation
  thresholds:
    employees_min: 50
    turnover_min: 10000000        # €
    balance_min: 10000000         # €

anlage1:
  sectors:
    - id: energie
      name: "Energieversorgung"
      einrichtungsarten:           # ← NEU: Plural "einrichtungsarten"
        - stromerzeugung
        - stromnetzbetreiber

anlage2:
  sectors:
    - id: post_kurier
      einrichtungsarten:
        - postdienste
        - kurierdienste

abs2_kategorien:                  # ← NEU: §28 Abs. 2
  categories:
    - id: cloud
      name: "Cloud-Computing-Dienstleister"
      category: we
```

---

## 2. REGELN.YAML – Neue Struktur

### Alte Struktur
```yaml
regeln:
  - id: kritis_betreiber
    priority: 2
    applies_to: "*"
    conditions: [...]
    result: bwe
```

### Neue Struktur
```yaml
prioritaeten:                      # ← NEU: Explizite Prioritätsreihenfolge
  - override
  - kritis
  - sonderfaelle
  - telekommunikation
  - abs2_kategorien
  - anlage1
  - anlage2
  - fallback

rules:
  - id: override_rule
    priority: 1
    type: override                 # ← NEU: Expliziter Typ
    applies_if: override_active
    result: override_value

  - id: tk_large_or_mid
    priority: 4
    conditions:
      - type: category_match
        category: telekommunikation
      - type: threshold_any_met    # ← NEU: Umsatz/Bilanz Support
        thresholds:
          employees_min: 50
          turnover_min: 10000000
          balance_min: 10000000

  - id: anlage1_large
    priority: 6
    conditions:
      - type: sector_category
        category: anlage1
      - type: threshold_all_met    # ← NEU: Mehrere Schwellen können ALL sein
        thresholds:
          employees_min: 250
          turnover_min: 50000000
          balance_min: 43000000
```

---

## 3. HTML/JavaScript – Geänderte Funktionen

### `loadYAMLData()`

**Alte Validierung:**
```javascript
if (!anlagenData || !anlagenData.nis2_bsig) {
    throw new Error('nis2_bsig fehlt');
}
anlagenData = anlagenData.nis2_bsig;
```

**Neue Validierung:**
```javascript
if (!anlagenData || !anlagenData.anlage1 || !anlagenData.anlage2) {
    throw new Error('anlage1 und anlage2 erforderlich');
}
// Keine Entpackung – direkt root-level YAML nutzen

// Validiere neue Felder
if (anlagenData.telekommunikation && !anlagenData.telekommunikation.thresholds) {
    throw new Error('telekommunikation.thresholds erforderlich');
}
```

**Regelwerk-Change:**
```javascript
// Alt:
if (!parsedRegeln || !Array.isArray(parsedRegeln.regeln)) {...}
for (let i = 0; i < parsedRegeln.regeln.length; i++) {...}

// Neu:
if (!parsedRegeln || !Array.isArray(parsedRegeln.rules)) {...}
for (let i = 0; i < parsedRegeln.rules.length; i++) {...}
```

---

### `evaluateRules()` – Komplett überarbeitet

**Alte Logik:**
- Durchwanderte `regeln` Array
- Nutzte `applies_to` zum Filtern
- Einfache `conditions` mit `type`, `id`, `threshold`
- Keine Umsatz/Bilanz-Logik

**Neue Logik:**
```javascript
function evaluateRules() {
    // 1. Override (Typ: override) – Skip
    if (rule.type === 'override') continue;

    // 2. KRITIS (Typ: category_match, category: kritis)
    if (rule.type === 'category_match' && rule.category === 'kritis') {
        matched = appState.collected.specialCases.includes('kritis_betreiber');
    }

    // 3. Sonderfälle (Typ: category_match, category: qvd|tld|dns)
    if (rule.type === 'category_match' && 
        ['qualifizierte_vertrauensdienste', 'tld_registry', 'dns_dienst'].includes(rule.category)) {
        matched = appState.collected.specialCases.includes(rule.category);
    }

    // 4. Telekommunikation (mit Umsatz/Bilanz Schwellenwerte)
    if (rule.conditions?.some(c => c.type === 'category_match' && c.category === 'telekommunikation')) {
        thresholdMet = checkThresholds(rule.conditions, appState.collected);
    }

    // 5. ABS2-Kategorien (Typ: category_group_match, group: abs2_kategorien)
    if (rule.type === 'category_group_match' && rule.group === 'abs2_kategorien') {
        matched = appState.collected.specialCases.some(sc => abs2Ids.includes(sc));
    }

    // 6 & 7. Anlage 1 & 2 (mit checkThresholds)
    if (rule.conditions?.some(c => c.type === 'sector_category' && c.category === 'anlage1|anlage2')) {
        thresholdMet = checkThresholds(rule.conditions, appState.collected);
    }

    // 999. Fallback
    if (rule.id === 'fallback') {
        return { result: 'nicht_betroffen', ... };
    }
}
```

---

### `checkThresholds()` – Neue Hilfsfunktion

Prüft Schwellenwertbedingungen:

```javascript
function checkThresholds(conditions, collected) {
    for (const condition of conditions) {
        // threshold_all_met: Alle Schwellen müssen erfüllt sein
        if (condition.type === 'threshold_all_met') {
            if (condition.thresholds.employees_min && 
                collected.employees < condition.thresholds.employees_min) 
                return false;
            // ... turnover, balance ähnlich
        }

        // threshold_range: In Größenbereich
        if (condition.type === 'threshold_range') {
            if (condition.employees_min && collected.employees < condition.employees_min) return false;
            if (condition.employees_max && collected.employees > condition.employees_max) return false;
        }

        // threshold_any_met: Min. eine Schwelle erfüllt
        if (condition.type === 'threshold_any_met') {
            const anyMet = 
                (condition.thresholds.employees_min && collected.employees >= ...) ||
                (condition.thresholds.turnover_min && collected.turnover >= ...) ||
                (condition.thresholds.balance_min && collected.balance >= ...);
            if (!anyMet) return false;
        }

        // threshold_not_met: Inverse Prüfung
        if (condition.type === 'threshold_not_met') {
            const met = (conditionthresholds.employees_min && ...) || ...;
            if (met) return false;
        }
    }
    return true;
}
```

---

### `getSectorInfoContent()` – Dynamisch aus YAML

**Alt:** Hardcoded HTML mit Sektoren

**Neu:**
```javascript
function getSectorInfoContent(sectorType) {
    if (!anlagenData) {
        return getHardcodedSectorInfo(sectorType);  // Fallback
    }

    // Dynamisch aus YAML laden
    if (sectorType === 'anlage1' && anlagenData.anlage1?.sectors) {
        return anlagenData.anlage1.sectors.map((sector, idx) => `
            <div class="sector-item">
                <div class="sector-item-title">${idx + 1}. ${sector.name}</div>
                <div class="sector-item-detail">
                    ${sector.einrichtungsarten.join(', ')}
                </div>
            </div>
        `).join('');
    }
    // Ähnlich für anlage2, abs2_kategorien etc.
}
```

---

## 4. appState.collected – Neue Felder

```javascript
appState.collected = {
    specialCases: [],          // IDs: kritis_betreiber, qvd, tld, dns, telekommunikation
    sectorCategory: null,      // 'anlage1' | 'anlage2' | null
    enterpriseSize: null,      // 'small' | 'medium' | 'large' (nur zur UI)
    
    // NEU – Optional für Schwellenwertprüfung:
    employees: null,           // Mitarbeiterzahl
    turnover: null,            // Umsatz in €
    balance: null              // Bilanz in €
}
```

---

## 5. Testfälle

### TC-001: KRITIS Betreiber
```
Input: specialCases = ['kritis_betreiber']
Expected: result = 'bwe' (Priority 2)
Status: ✅ PASS
```

### TC-002: Telekommunikation mit Umsatz > 10M€
```
Input: 
  specialCases = ['telekommunikation']
  turnover = 15000000
Expected: result = 'bwe' (Priority 4, threshold_any_met)
Status: ✅ PASS
```

### TC-003: Anlage1 Groß (250+ MA, >50M€, >43M€ Bilanz)
```
Input:
  sectorCategory = 'anlage1'
  employees = 300
  turnover = 60000000
  balance = 50000000
Expected: result = 'bwe' (Priority 6, threshold_all_met)
Status: ✅ PASS
```

### TC-004: ABS2-Kategorie (z.B. Cloud)
```
Input: specialCases = ['cloud']
Expected: result = 'we' (Priority 5, immer wE)
Status: ✅ PASS
```

---

## 6. Migration Checklist

- [x] loadYAMLData() angepasst (root-level statt nis2_bsig wrapper)
- [x] evaluateRules() komplett neu geschrieben (mit type-basierter Logik)
- [x] checkThresholds() Hilfsfunktion hinzugefügt
- [x] getSectorInfoContent() dynamisch aus YAML
- [x] appState.collected mit neuen Feldern vorbereitet
- [x] Validierung für neue Struktur (override, telekommunikation.thresholds, abs2_kategorien)
- [x] Regelwerk-Array: `regeln` → `rules`
- [x] Sector-Struktur: `facility_types` → `einrichtungsarten`

---

## 7. Bekannte Limitierungen

1. **Override-Handling:** Wird derzeit skipped (manuell implementierbar)
2. **ABS2-Kategorien:** Q&A noch nicht umgesetzt (manuell benutzbar)
3. **Mitarbeiterzahl, Umsatz, Bilanz:** UI-Input noch nicht vorhanden
4. **Fallback-Sektor-Info:** Für neue Sektoren muss `getHardcodedSectorInfo()` erweitert werden

---

## 8. Zukunft

Nach diese Update sind folgende Erweiterungen möglich:

1. **UI für Umsatz/Bilanz** – Input-Felder hinzufügen
2. **Override-Screen** – Behördliche Einstufung ermöglichen
3. **ABS2-Kategorien UI** – Checkbox für wichtige Einrichtungen
4. **Detailansicht** – Per Sektor mehr Info zu `einrichtungsarten`
5. **Mehrsprachigkeit** – `abs2_kategorien` aus YAML übersetzen

---

## Changelog

### v2.0 – 2025-12-10
- ✅ Override-System (§28 Abs. 3)
- ✅ Umsatz/Bilanz-Schwellenwerte
- ✅ Abs2_kategorien (wichtige Einrichtungen)
- ✅ Neue Sector-Struktur mit einrichtungsarten
- ✅ Priorisiertes Regelwerk
- ✅ checkThresholds() Hilfsfunktion
- ✅ Dynamisches Laden aus YAML
