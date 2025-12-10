# YAML Migration Guide – Kritische Punkte

**Zielgruppe:** Developer, die die neue YAML-Struktur verstehen/erweitern möchten

---

## 1. YAML-Struktur auf einen Blick

### Alte Struktur (deprecated)
```yaml
nis2_bsig:
  kategorien:
    bwe: {...}
  groessenschwellen:
    small_enterprise: {...}
  anlage1:
    title: "Anlage 1..."
    sectors: [...]
```

### Neue Struktur (aktiv)
```yaml
# ROOT LEVEL – kein wrapper
override: {...}                    # Behördliche Einstufung
kritis: {...}                      # KRITIS-Betreiber
sonderfaelle: [...]                # QVD, TLD, DNS
telekommunikation: {...}           # Mit Umsatz/Bilanz
anlage1: {...}                     # Sektoren hoher Kritikalität
anlage2: {...}                     # Sonstige kritische Sektoren
abs2_kategorien: {...}             # Wichtige Einrichtungen (§28 Abs. 2)
```

---

## 2. Kritische Änderungen in JavaScript

### ❌ Falsch (alte Struktur)
```javascript
anlagenData = jsyaml.load(yaml);
const anlage1 = anlagenData.nis2_bsig.anlage1;  // ❌ Fehler

regelnData.regeln.forEach(rule => {...});  // ❌ Fehler
```

### ✅ Richtig (neue Struktur)
```javascript
anlagenData = jsyaml.load(yaml);
const anlage1 = anlagenData.anlage1;  // ✅ Direkt

regelnData.rules.forEach(rule => {...});  // ✅ Neuer Array-Name
```

---

## 3. Schwellenwert-Prüfung

### Neue Condition-Typen in regeln.yaml

```yaml
rules:
  - id: beispiel_all_met
    conditions:
      - type: threshold_all_met       # Alle müssen erfüllt sein
        thresholds:
          employees_min: 250
          turnover_min: 50000000      # €
          balance_min: 43000000       # €

  - id: beispiel_any_met
    conditions:
      - type: threshold_any_met       # Mindestens eine erfüllt
        thresholds:
          employees_min: 50
          turnover_min: 10000000

  - id: beispiel_range
    conditions:
      - type: threshold_range        # Bereich
        employees_min: 50
        employees_max: 249

  - id: beispiel_not_met
    conditions:
      - type: threshold_not_met      # Inverse Prüfung
        thresholds:
          employees_min: 50
```

### Wichtig: appState.collected Felder
```javascript
appState.collected = {
    employees: 300,        // Mitarbeiterzahl (optional)
    turnover: 60000000,    // Umsatz in € (optional)
    balance: 50000000,     // Bilanz in € (optional)
    specialCases: [...],
    sectorCategory: 'anlage1'
}
```

---

## 4. Sektor-Struktur: einrichtungsarten

### Alte Struktur (deprecated)
```yaml
anlage1:
  sectors:
    - id: energie
      name: "Energie"
      facility_types:            # ❌ Alt
        - id: strom_erzeugung
          name: "Stromerzeugung"
```

### Neue Struktur (aktiv)
```yaml
anlage1:
  sectors:
    - id: energie
      name: "Energieversorgung (komplett nach Gesetz)"
      einrichtungsarten:         # ✅ Neu (plural!)
        - stromerzeugung         # ✅ Nur IDs, keine Nesting
        - stromnetzbetreiber
        - gasversorgung
        - ölförderung
```

**Warum die Änderung?**
- `einrichtungsarten` ist das korrekte juristische Deutsch
- Keine nested Objekte → einfacher zu iterieren
- Direkt in `getSectorInfoContent()` nutzbar

---

## 5. Regelwerk-Typ-System

### Alte Bedingungen
```yaml
conditions:
  - type: special_case           # Sonderfall-ID
    id: kritis_betreiber
  - type: sector_category        # Sektor-Kategorie
    category: anlage1
  - type: size                   # Größe (simple)
    threshold: medium_enterprise
```

### Neue Bedingungen
```yaml
conditions:
  - type: category_match         # Kategorie-Match
    category: kritis_betreiber   # oder: telekommunikation
  - type: sector_category        # Sektor (wie vorher)
    category: anlage1
  - type: threshold_all_met      # Mehrere Schwellen
    thresholds:
      employees_min: 250
      turnover_min: 50000000
      balance_min: 43000000
```

**Mapping für evaluateRules():**
```javascript
// type: override → Skip (nicht auto-evaluieren)
if (rule.type === 'override') continue;

// type: category_match → Sonderfall/KRITIS
if (rule.type === 'category_match') {
    matched = appState.collected.specialCases.includes(rule.category);
}

// type: category_group_match → ABS2-Kategorien
if (rule.type === 'category_group_match') {
    const ids = anlagenData.abs2_kategorien.categories.map(c => c.id);
    matched = appState.collected.specialCases.some(sc => ids.includes(sc));
}

// Thresholds → checkThresholds() Hilfsfunktion
let thresholdMet = checkThresholds(rule.conditions, appState.collected);
```

---

## 6. Abs2_kategorien – Wichtige Einrichtungen

### Neue Sektion in anlage.yaml
```yaml
abs2_kategorien:                        # ← NEU
  legal_ref: "§28 Abs. 2 BSIG"
  description: "Wichtige Einrichtungen..."
  categories:
    - id: cloud
      name: "Cloud-Computing-Dienstleister"
      category: we                      # Immer wE, nie bwE

    - id: managed_security_service_provider
      name: "Managed Security Service Provider"
      category: we
```

### Regel in regeln.yaml
```yaml
  - id: abs2_wichtige_einrichtung
    priority: 5
    type: category_group_match          # ← Neuer Typ
    group: abs2_kategorien              # ← Sektion-Referenz
    result: we
    legal_ref: "§28 Abs. 2"
```

### Evaluierung in JavaScript
```javascript
if (rule.type === 'category_group_match' && rule.group === 'abs2_kategorien') {
    const abs2Ids = anlagenData.abs2_kategorien.categories.map(c => c.id);
    matched = appState.collected.specialCases.some(sc => abs2Ids.includes(sc));
    if (matched) {
        return { result: rule.result, appliedRule: rule.id, ... };
    }
}
```

---

## 7. Override-System (§28 Abs. 3)

### Struktur in anlage.yaml
```yaml
override:
  enabled: true                         # Aktiviert?
  rules:
    - id: behoerdliche_einstufung
      result: [bwe, we, nicht_betroffen]  # Mögliche Ergebnisse
      reason_required: true              # Begründung erforderlich?
      legal_ref: "§28 Abs. 3 BSIG"
```

### Regel in regeln.yaml
```yaml
  - id: override_rule
    priority: 1                         # Höchste Priorität!
    type: override
    applies_if: override_active         # Bedingung
    result: override_value              # Wird durch Benutzer gesetzt
```

### Implementierung (TODO)
```javascript
// Override wird derzeit skipped
if (rule.type === 'override') {
    // TODO: Benutzer-Input für override_value
    // TODO: UI für Begründung (reason_required)
    continue;
}
```

---

## 8. Prioritätsreihenfolge

**Wichtig:** Die `prioritaeten` Liste in regeln.yaml definiert die logische Reihenfolge:

```yaml
prioritaeten:
  1. override         # Behördliche Einstufung (höchste)
  2. kritis           # KRITIS-Betreiber
  3. sonderfaelle     # QVD, TLD, DNS
  4. telekommunikation # Mit Umsatz/Bilanz
  5. abs2_kategorien  # Wichtige Einrichtungen
  6. anlage1          # Sektoren hoher Kritikalität
  7. anlage2          # Sonstige kritische Sektoren
  999. fallback       # Standard: nicht_betroffen
```

**In JavaScript:**
```javascript
const sortedRules = [...regelnData.rules].sort((a, b) =>
    (a.priority || 999) - (b.priority || 999)
);
```

Die `priority` im `rules` Array bestimmt die Reihenfolge! (nicht die `prioritaeten` Liste)

---

## 9. Checklist: Neue Felder hinzufügen

### Um einen neuen Sektor zu anlage1 hinzuzufügen:

1. **anlage.yaml:** Sektor in `anlage1.sectors` hinzufügen
   ```yaml
   - id: neue_sektor
     name: "Neue Sektor"
     einrichtungsarten:
       - einrichtungsart1
       - einrichtungsart2
   ```

2. **regeln.yaml:** Regel(n) hinzufügen (falls nötig)
   ```yaml
   - id: neue_sektor_large
     priority: 6
     conditions:
       - type: sector_category
         category: anlage1
       - type: threshold_all_met
         thresholds:
           employees_min: 250
     result: bwe
   ```

3. **index.html:** Normalerweise keine Änderung nötig!
   - `getSectorInfoContent()` lädt dynamisch aus YAML
   - `evaluateRules()` funktioniert generisch

4. **Test:** Neue Einrichtungsart testen
   ```javascript
   // Browser Console
   anlagenData.anlage1.sectors[X].einrichtungsarten
   ```

---

## 10. Fehlerbehebung

### Fehler: "anlage.yaml: Erforderliche Sektionen 'anlage1' und 'anlage2' fehlen"
**Ursache:** YAML hat noch alten `nis2_bsig` wrapper  
**Lösung:** Root-level Struktur verwenden

### Fehler: "regeln.yaml: Erforderliche Struktur 'rules' (Array) fehlt"
**Ursache:** Array hieß `regeln`, nicht `rules`  
**Lösung:** Zu `rules:` umbenennen

### Fehler: "threshold_all_met aber employees ist undefined"
**Ursache:** appState.collected hat keine Felder für Mitarbeiterzahl/Umsatz  
**Lösung:** Neue Input-Felder in UI hinzufügen + appState.collected initialisieren

### Fehler: "getSectorInfoContent() zeigt alte Einträge"
**Ursache:** Fallback zu `getHardcodedSectorInfo()` wird verwendet  
**Lösung:** `anlagenData` vor `showQuestion()` prüfen

---

## 11. Code-Patterns

### Pattern 1: Sektor-Info dynamisch laden
```javascript
function getSectorInfoContent(sectorType) {
    if (!anlagenData || !anlagenData[sectorType]) {
        return getHardcodedSectorInfo(sectorType);
    }
    
    return anlagenData[sectorType].sectors
        .map((sector, idx) => `<div>...</div>`)
        .join('');
}
```

### Pattern 2: Schwellenwerte prüfen
```javascript
function checkThresholds(conditions, collected) {
    for (const cond of conditions) {
        if (cond.type === 'threshold_all_met') {
            const allMet = 
                (!cond.thresholds.employees_min || collected.employees >= cond.thresholds.employees_min) &&
                (!cond.thresholds.turnover_min || collected.turnover >= cond.thresholds.turnover_min) &&
                (!cond.thresholds.balance_min || collected.balance >= cond.thresholds.balance_min);
            if (!allMet) return false;
        }
    }
    return true;
}
```

### Pattern 3: ABS2-Kategorien iterieren
```javascript
if (anlagenData.abs2_kategorien?.categories) {
    for (const category of anlagenData.abs2_kategorien.categories) {
        console.log(category.id, category.name, category.category);
    }
}
```

---

## Version History

| Version | Datum | Änderung |
|---------|-------|----------|
| 1.0 | <2025 | Initial (mit nis2_bsig wrapper) |
| 2.0 | 2025-12 | Override, Umsatz/Bilanz, abs2_kategorien, neue Struktur |
