# YAML Quick Reference – Neue Struktur 2.0

---

## Top-Level Keys (anlage.yaml)

```yaml
override              # Behördliche Einstufung (§28 Abs. 3)
kritis                # KRITIS-Betreiber (§28 Abs. 1 Nr. 1)
sonderfaelle: []      # QVD, TLD, DNS (§28 Abs. 1 Nr. 2-4)
telekommunikation     # TK-Netze mit Umsatz/Bilanz (§28 Abs. 1 Nr. 5)
anlage1               # Sektoren hoher Kritikalität (§28 Abs. 1 Nr. 6)
anlage2               # Sonstige kritische Sektoren (§28 Abs. 2)
abs2_kategorien       # Wichtige Einrichtungen (§28 Abs. 2)
```

---

## Schwellenwerte

### Telekommunikation
```yaml
telekommunikation:
  thresholds:
    employees_min: 50
    turnover_min: 10000000      # €
    balance_min: 10000000       # €
```

### Anlage 1
```yaml
anlage1:
  size_rules:
    - condition: large_enterprise
      employees_min: 250
      turnover_min: 50000000     # €
      balance_min: 43000000      # €
      result: bwe
    - condition: medium_enterprise
      employees_min: 50
      employees_max: 249
      result: we
    - condition: small_enterprise
      employees_max: 49
      result: nicht_betroffen
```

### Anlage 2
```yaml
anlage2:
  size_rules:
    - condition: medium_or_large
      employees_min: 50
      result: we
    - condition: small
      employees_max: 49
      result: nicht_betroffen
```

---

## Sektor-Struktur

### Format
```yaml
sectors:
  - id: sektor_id
    name: "Lesbare Bezeichnung"
    einrichtungsarten:
      - einrichtungsart1
      - einrichtungsart2
```

### Beispiele

**Anlage 1 – Energie:**
```yaml
- id: energie
  name: "Energieversorgung (komplett nach Gesetz)"
  einrichtungsarten:
    - stromerzeugung
    - stromnetzbetreiber
    - gasversorgung
    - ölförderung
    - fernwärme
    - wasserstoff
```

**Anlage 2 – Post:**
```yaml
- id: post_kurier
  name: "Post & Kurier"
  einrichtungsarten:
    - postdienste
    - kurierdienste
```

---

## Regelwerk-Struktur (regeln.yaml)

### Minimales Rule-Format
```yaml
- id: rule_id
  priority: 3
  type: category_match
  category: kritis_betreiber
  result: bwe
  legal_ref: "§28 Abs. 1 Nr. 1"
```

### Komplexes Rule-Format mit Schwellenwerten
```yaml
- id: rule_id
  priority: 6
  conditions:
    - type: sector_category
      category: anlage1
    - type: threshold_all_met
      thresholds:
        employees_min: 250
        turnover_min: 50000000
        balance_min: 43000000
  result: bwe
```

---

## Rule-Typen

| Type | Feld | Wert | Beispiel |
|------|------|------|----------|
| override | type | 'override' | Behördliche Einstufung |
| category_match | category | 'kritis_betreiber', 'telekommunikation', etc. | KRITIS/Sonderfälle |
| category_group_match | group | 'abs2_kategorien' | Wichtige Einrichtungen |
| sector_category | category | 'anlage1', 'anlage2' | Sektor-Zuordnung |
| threshold_all_met | thresholds | {employees_min, turnover_min, balance_min} | Alle müssen erfüllt sein |
| threshold_range | employees_min/max | Zahl | Bereich (z.B. 50-249) |
| threshold_any_met | thresholds | Wie all_met | Min. eine erfüllt |
| threshold_not_met | thresholds | Wie all_met | Inverse Prüfung |

---

## appState.collected – Datenfelder

```javascript
{
    specialCases: [],           // IDs: kritis_betreiber, qvd, tld, dns, tk, cloud, etc.
    sectorCategory: null,       // 'anlage1' | 'anlage2' | null
    enterpriseSize: null,       // UI-only: 'small' | 'medium' | 'large'
    
    // Für Schwellenwertprüfung (optional):
    employees: null,            // Mitarbeiterzahl
    turnover: null,             // Umsatz in €
    balance: null               // Bilanz in €
}
```

---

## Evaluierung – Ablauf

```
1. Override? (Priority 1)
   ├─ Ja → return override_result
   └─ Nein → weiter

2. KRITIS? (Priority 2)
   ├─ Ja → return bwe
   └─ Nein → weiter

3. Sonderfälle QVD/TLD/DNS? (Priority 3)
   ├─ Ja → return bwe
   └─ Nein → weiter

4. Telekommunikation? (Priority 4)
   ├─ Mit Schwellenwert erfüllt? 
   │  ├─ Ja → return bwe
   │  └─ Nein → return nicht_betroffen
   └─ Nicht TK → weiter

5. ABS2-Kategorien? (Priority 5)
   ├─ Ja → return we
   └─ Nein → weiter

6. Anlage 1 + Größe? (Priority 6)
   ├─ Große (250+ MA, 50M€+, 43M€ Bilanz) → return bwe
   ├─ Mittel (50-249 MA) → return we
   ├─ Klein (<50 MA) → return nicht_betroffen
   └─ Nicht Anlage 1 → weiter

7. Anlage 2 + Größe? (Priority 7)
   ├─ Mittel/Groß (50+ MA) → return we
   ├─ Klein (<50 MA) → return nicht_betroffen
   └─ Nicht Anlage 2 → weiter

999. Fallback → return nicht_betroffen
```

---

## Code-Beispiele

### Sektor-Info abrufen
```javascript
const sektoren = anlagenData.anlage1.sectors;
// Result: [{id: 'energie', name: '...', einrichtungsarten: [...]}, ...]
```

### Sonderfall prüfen
```javascript
const istTK = appState.collected.specialCases.includes('telekommunikation');
const istQVD = appState.collected.specialCases.includes('qualifizierte_vertrauensdienste');
```

### Schwellenwert prüfen
```javascript
const thresholds = {
    employees_min: 250,
    turnover_min: 50000000,
    balance_min: 43000000
};

const passed = 
    appState.collected.employees >= thresholds.employees_min &&
    appState.collected.turnover >= thresholds.turnover_min &&
    appState.collected.balance >= thresholds.balance_min;
```

### ABS2-Kategorie Check
```javascript
const abs2Ids = anlagenData.abs2_kategorien.categories.map(c => c.id);
const isABS2 = appState.collected.specialCases.some(sc => abs2Ids.includes(sc));
```

---

## Häufige Fehler

| Fehler | Ursache | Fix |
|--------|---------|-----|
| `Cannot read property 'anlage1' of undefined` | anlagenData nicht geladen | Vor `evaluateRules()` `loadYAMLData()` aufrufen |
| `rule.regeln is not iterable` | Array heißt `rules`, nicht `regeln` | In regeln.yaml: `rules:` nutzen |
| `Cannot read property 'einrichtungsarten' of undefined` | YAML nutzt alte `facility_types` | Zu `einrichtungsarten:` umbenennen |
| Schwellenwerte greifen nicht | appState.collected.employees ist null | Neue Input-Felder für Zahlen hinzufügen |
| Override wird ignoriert | Type ist 'override' und wird skipped | Override-UI implementieren |

---

## Gültige Sektor-IDs (anlage1)

```
energie
gesundheit
digitale_infrastruktur
transport
wasser
```

---

## Gültige Sektor-IDs (anlage2)

```
post_kurier
abfall
chemie
produktion
```

---

## Gültige ABS2-Kategorien

```
vertrauensdienste
cloud
rechenzentren
cdn
managed_service_provider
managed_security_service_provider
suchmaschinen
soziale_netzwerke
online_marktplaetze
```

---

## Wichtige Legal References

| Regel | Referenz |
|------|----------|
| KRITIS | §28 Abs. 1 Nr. 1 BSIG |
| Sonderfälle (QVD/TLD/DNS) | §28 Abs. 1 Nr. 2-4 BSIG |
| Telekommunikation | §28 Abs. 1 Nr. 5 BSIG |
| Anlage 1 | §28 Abs. 1 Nr. 6 BSIG |
| Anlage 2 / ABS2 | §28 Abs. 2 BSIG |
| Override | §28 Abs. 3 BSIG |
