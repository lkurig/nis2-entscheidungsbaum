# 🔍 NIS-2 ENTSCHEIDUNGSBAUM – AUDIT-REPORT

**Datum:** Dezember 2025  
**Analysiert:** Frontend (index.html) | Regelwerk (regeln.yaml) | Datenstruktur (anlage.yaml)  
**Gesamtstatus:** ⚠️ **PRODUKTIONSREIF MIT KRITISCHEN ANPASSUNGEN ERFORDERLICH**

---

## A) KONSISTENZ-CHECK ✅ / ⚠️

### A.1 ID-Konsistenz zwischen allen Komponenten

| ID | anlage.yaml | regeln.yaml | index.html | Status |
|----|-------------|-------------|-----------|---------|
| `kritis_betreiber` | ✅ Line 75 | ✅ Line 14 | ⚠️ Wird als 'KRITIS' in onYes() gespeichert | **FEHLER** |
| `qualifizierte_vertrauensdienste` | ✅ Line 91 | ✅ Line 31 | ✅ q1a Line 650 | **OK** |
| `tld_registry` | ✅ Line 99 | ✅ Line 43 | ✅ q1a Line 651 | **OK** |
| `dns_dienst` | ✅ Line 107 | ✅ Line 55 | ✅ q1a Line 652 | **OK** |
| `telekommunikation` | ✅ Line 115 | ✅ Line 72, 86, 100 | ✅ q1a Line 653 | **OK** |
| `anlage1` | ✅ Line 130 | ✅ Line 116 | ✅ q2 Line 678 | **OK** |
| `anlage2` | ✅ Line 387 | ✅ Line 166 | ✅ q2 Line 679 | **OK** |

**FEHLER #1: KRITIS-ID Mismatch**
```javascript
// Frontend (q0, onYes):
appState.collected.specialCases.push('kritis_betreiber');  // Erwartet von regeln.yaml

// Aber in showResult() wird 'bwe_kritis' als direktes Template-Key verwendet
// Das ist technisch OK, weil KRITIS → bwe_kritis ein hardcodierter Shortcut ist
// Aber es ist INKONSISTENT mit der evaluateRules()-Logik!
```

### A.2 Size-Schwellen Konsistenz

**anlage.yaml:**
```yaml
small_enterprise: employees_max: 49, revenue_max: 10M, balance_max: 10M
medium_enterprise: employees_min: 50, revenue_min: 10M+1, balance_min: 10M+1
large_enterprise: employees_min: 250, revenue_min: 50M+1, balance_min: 43M+1
```

**Frontend:**
```javascript
// index.html q1b, q3:
options: [
  { value: 'small', label: 'Klein (<50 MA, ≤10M€)' },
  { value: 'medium', label: 'Mittel (50-249 MA, >10M€ Umsatz UND Bilanz, ≤50M€)' },
  { value: 'large', label: 'Groß (≥250 MA oder >50M€ Umsatz UND >43M€ Bilanz)' }
]
```

**Status:** ✅ **KONSISTENT** (Labels sind vereinfacht, aber Schwellen korrekt)

### A.3 Sonderfälle-Abdeckung

**In anlage.yaml definiert:**
- ✅ qualifizierte_vertrauensdienste (eIDAS)
- ✅ tld_registry
- ✅ dns_dienst
- ✅ telekommunikation (size_relevant: true)

**In regeln.yaml abgedeckt:**
- ✅ qvda_immer_bwe (Priorität 2)
- ✅ tld_immer_bwe (Priorität 2)
- ✅ dns_immer_bwe (Priorität 2)
- ✅ telekommunikation_gross_bwe (Priorität 3)
- ✅ telekommunikation_mittel_bwe (Priorität 3)
- ✅ telekommunikation_klein_nicht_betroffen (Priorität 3)

**In Frontend (q1a) abgedeckt:**
- ✅ Alle 4 als Checkboxen vorhanden

**Status:** ✅ **VOLLSTÄNDIG**

### A.4 Pflichten-Verknüpfung

**anlage.yaml enthält:**
- ✅ `pflichten.bwe` (9 Pflichten)
- ✅ `pflichten.we` (6 Pflichten)
- ✅ `pflichten.nicht_betroffen`

**Frontend index.html enthält:**
```javascript
resultTemplates: {
  bwe_kritis: obligations: [7 Items], ✅ 
  bwe: obligations: [8 Items], ✅
  we: obligations: [6 Items], ✅
  nicht_betroffen: obligations: [3 Items] ✅
}
```

**Status:** ✅ **VERKNÜPFT** (Frontend hat hardcodierte Pflichten, nicht aus anlage.yaml geladen)

---

## B) LOGIK-CHECK 🔴 / ⚠️

### B.1 Regel-Implementierungsprüfung

**Priorität 1: KRITIS**
```yaml
# regeln.yaml:
- id: kritis_betreiber_immer_bwe
  applies_to: "*"
  conditions: [{ type: special_case, id: kritis_betreiber }]
  result: bwe
```

**Frontend:**
```javascript
// q0: Betreibt KRITIS?
yesAction: { type: 'result', result: 'bwe_kritis' }  // ✅ Shortcut statt evaluateRules()

// onYes Callback:
onYes: () => { appState.collected.specialCases.push('kritis_betreiber'); }  // ✅ Sammelt ID
```

**Problem:** ⚠️ **KRITIS nutzt SHORTCUT, nicht evaluateRules()**
- Wenn Nutzer später zu Q2_SF geht und auch Anlage 1 auswählt → Daten gehen verloren
- KRITIS wird direkt zu `bwe_kritis` ohne Regelwerk-Prüfung

**Status:** ⚠️ **FUNKTIONIERT, aber konzeptionell inkonsistent**

---

**Priorität 2: Sonderfälle (QVD, TLD, DNS)**
```yaml
- id: qvda_immer_bwe
  applies_to: "*"
  conditions: [{ type: special_case, id: qualifizierte_vertrauensdienste }]
```

**Frontend Q1a:**
```javascript
// Multi-Select speichert:
appState.collected.specialCases = ['qualifizierte_vertrauensdienste', ...]

// Q2_SF: Nein → evaluateRules()
// evaluateRules() prüft:
if (condition.type === 'special_case') {
  conditionOk = appState.collected.specialCases.includes(condition.id);
}
```

**Status:** ✅ **FUNKTIONIERT KORREKT**

---

**Priorität 3: Telekommunikation (mit Size)**
```yaml
- id: telekommunikation_mittel_bwe
  conditions:
    - type: special_case, id: telekommunikation
    - type: size, threshold: medium_enterprise
```

**Frontend Q1b:**
```javascript
// TK-Größenfrage:
options: [small, medium, large]
// → appState.collected.enterpriseSize = 'medium'

// evaluateRules() konvertiert:
const enterpriseSize = sizeMap['medium'] = 'medium_enterprise'

// Regelprüfung:
condition.threshold === 'medium_enterprise' ✅
```

**Status:** ✅ **FUNKTIONIERT KORREKT**

---

**Priorität 4 & 5: Anlage 1 & 2**
```yaml
- id: anlage1_gross_bwe
  applies_to: anlage1
  conditions:
    - type: sector_category, category: anlage1
    - type: size, threshold: large_enterprise
```

**Frontend Q2 → Q3:**
```javascript
// Q2: anlage1 → appState.collected.sectorCategory = 'anlage1'
// Q3: large → appState.collected.enterpriseSize = 'large'
// → evaluateRules()

// Regelprüfung:
rule.applies_to === 'anlage1' && appState.collected.sectorCategory === 'anlage1' ✅
enterpriseSize === 'large_enterprise' ✅
```

**Status:** ✅ **FUNKTIONIERT KORREKT**

---

### B.2 evaluateRules() Vollständigkeitsprüfung

| Input | Wird genutzt? | Wo? |
|-------|--------------|-----|
| `appState.collected.specialCases` | ✅ | Line 539-540 |
| `appState.collected.sectorCategory` | ✅ | Line 521, 542 |
| `appState.collected.enterpriseSize` | ✅ | Line 503 (normalisiert), 545 |
| `appState.collected.facilityIds` | ❌ **NICHT GENUTZT** | – |

**FEHLER #2: facilityIds wird gesammelt, aber NICHT evaluiert**
```javascript
// Index.html sammelt:
appState.collected: { facilityIds: [], ... }  // Initialisiert, aber nie gefüllt!

// evaluateRules() hat KEINE Bedingung für facilityIds!
// regeln.yaml hat KEINE facility-Bedingung!
```

**Status:** ⚠️ **TEILWEISE UNVOLLSTÄNDIG**
- `facilityIds` ist konzeptionell vorgesehen, wird aber nicht genutzt
- Nicht kritisch, da Sektoren (anlage1, anlage2) ausreichen
- Aber Deadcode

---

### B.3 Regelkonflikt-Prüfung

**Frage:** Können zwei Regeln auf dieselben Eingaben gleichzeitig passen?

**Szenario 1: TK mit size=medium**
```yaml
Regel: telekommunikation_mittel_bwe (priority 3)
  conditions: [telekommunikation, medium_enterprise]
  result: bwe

Regel: telekommunikation_klein_nicht_betroffen (priority 3)
  conditions: [telekommunikation, small_enterprise]
  result: nicht_betroffen
```

→ ✅ **OK** Size ist unterschiedlich

**Szenario 2: Anlage 1 + Large**
```yaml
Regel: anlage1_gross_bwe (priority 4)
  conditions: [anlage1, large_enterprise]
  result: bwe

Regel: anlage1_mittel_we (priority 4)
  conditions: [anlage1, medium_enterprise]
  result: we
```

→ ✅ **OK** Size ist unterschiedlich

**Szenario 3: TK + Anlage 1**
```
Wenn specialCases=['telekommunikation'] && sectorCategory='anlage1':
  → telekommunikation_mittel_bwe (Priorität 3) GREIFT ZUERST
  → anlage1_mittel_we (Priorität 4) wird NICHT geprüft
```

→ ✅ **PRIORITÄT ENTSCHEIDET KORREKT**

**Status:** ✅ **KEINE KONFLIKTE**

---

### B.4 KRITIS-Priorität Korrektheit

**Erwartung:** KRITIS ist ABSOLUT höchste Priorität, immer bwE

**Implementierung:** 
```javascript
// Frontend: Q0 KRITIS = Ja → DIREKT zu result='bwe_kritis'
// Q0 KRITIS = Nein → Q1

// evaluateRules() hat KRITIS nicht relevant, weil:
// - KRITIS wird als Shortcut behandelt
// - oder appState.collected.specialCases.push('kritis_betreiber')
//   → würde in evaluateRules() mit Priorität 1 geprüft
```

**Problem:** ⚠️ **INKONSISTENZ**
- KRITIS Ja → Direkt bwe_kritis (Shortcut, kein evaluateRules)
- KRITIS Nein + TK Ja + Size Medium → evaluateRules() → bwe (via TK-Regel)

→ Beide Pfade führen zu `bwe`, aber eine ist regelwerk-getrieben, eine nicht!

**Status:** ⚠️ **FUNKTIONIERT, aber konzeptionell problematisch**

---

## C) READINESS-CHECK 🔴 / ⚠️

### C.1 YAML-Validität

```bash
✅ anlage.yaml: Syntaktisch valide (getestet mit jsyaml)
✅ regeln.yaml: Syntaktisch valide
```

**Aber:** ⚠️ **Keine Fehlerbehandlung im Frontend!**
```javascript
async function loadYAMLData() {
    try {
        const response = await fetch('anlage.yaml');
        const yamlText = await response.text();
        const parsedYaml = jsyaml.load(yamlText);
        // KEIN Validierungsschema!
        // KEIN Check auf erforderliche Keys!
        anlagenData = parsedYaml.nis2_bsig;
    } catch (error) {
        console.error('Fehler beim Laden...');
        // anlagenData bleibt null!
    }
}
```

**FEHLER #3: Keine YAML-Schemavalidierung**
- Wenn YAML unvollständig oder mit Typos → stille Fehler
- regeln.yaml wird geladen, aber nicht validiert
- Keine Checks auf erforderliche Felder (id, priority, conditions, etc.)

---

### C.2 Race Conditions

**Szenario:** HTML lädt, und User klickt "Prüfung starten" bevor YAML geladen ist

```javascript
// index.html DOMContentLoaded:
document.addEventListener('DOMContentLoaded', async function () {
    await loadYAMLData();  // ✅ AWAIT ist hier
    showStartScreen();
});

// Aber:
function evaluateRules() {
    if (!regelnData || !regelnData.regeln) {
        console.error('Regelwerk nicht geladen');  // ⚠️ Silent fail
        return { result: 'nicht_betroffen', ... };
    }
}
```

**Problem:** ⚠️ **KEINE Fehler-Anzeige für User!**
- Wenn YAML-Ladefehler → evaluateRules() gibt still `nicht_betroffen` zurück
- User bekommt falsche Auskunft ohne Warnung

---

### C.3 Hardcoding vs. YAML-Daten

| Element | Quelle | Status |
|---------|--------|--------|
| Fragen (q0-q3) | Frontend hardcoded | ⚠️ Nicht aus YAML |
| Sonderfälle-Liste (q1a) | Frontend hardcoded | ⚠️ Nicht aus anlage.yaml.sonderfaelle |
| Sektor-Optionen (q2) | Frontend hardcoded | ⚠️ Nicht aus anlage.yaml.anlage1/anlage2 |
| Größen-Optionen (q1b, q3) | Frontend hardcoded | ⚠️ Nicht aus anlage.yaml.groessenschwellen |
| Pflichten | Frontend hardcoded | ⚠️ Nicht aus anlage.yaml.pflichten |
| Kategorien-Info | Frontend hardcoded | ⚠️ Nicht aus anlage.yaml.kategorien |

**Status:** 🔴 **KRITISCH – WARTBARKEITSPROBLEM**

Wenn anlage.yaml aktualisiert wird (z.B. neuer Sektor), muss Frontend auch aktualisiert werden!

---

### C.4 Fehlertoleranz

**Test-Fall:** Nutzer beantwortet alle Fragen, regeln.yaml konnte nicht geladen werden

```javascript
// evaluateRules():
if (!regelnData || !regelnData.regeln) {
    console.error('Regelwerk nicht geladen');
    return {
        result: 'nicht_betroffen',
        appliedRule: null,
        explanation: 'Fehler: Regelwerk konnte nicht geladen werden'
    };
}

// → User sieht: "Nicht betroffen"
// → FALSCH! Sollte Fehlerscreen anzeigen!
```

**Status:** 🔴 **KEINE FEHLERBEHANDLUNG FÜR USER**

---

## D) TESTFÄLLE SIMULATION 🧪

Ich simuliere jeden Testfall und überprüfe:
1. Werden richtige States gesammelt?
2. Interpretiert evaluateRules() korrekt?
3. Ergebnis ist gesetzlich richtig?

### TC-001: KRITIS = Ja

```
Input: Q0 Ja
Expected: bwE (§28 Abs. 1 Nr. 1)

Frontend Flow:
  Q0: Ja → onYes() → appState.collected.specialCases.push('kritis_betreiber')
       → yesAction = { type: 'result', result: 'bwe_kritis' }
       → showResult('bwe_kritis')
  
Output: ✅ KRITIS-Template wird angezeigt
Status: ✅ KORREKT
```

---

### TC-002: Sonderfälle Nein → Anlage 1 groß

```
Input: Q0 Nein → Q1 Nein → Q2 Anlage1 → Q3 Groß
Expected: bwE (§28 Abs. 1 Nr. 6 für Großunternehmen)

Frontend Flow:
  Q0: Nein → Q1
  Q1: Nein → Q2
  Q2: Anlage1 → appState.collected.sectorCategory = 'anlage1'
  Q3: Groß → appState.collected.enterpriseSize = 'large'
       → yesAction = { type: 'result', result: 'evaluate_rules' }
       → evaluateRules()
  
evaluateRules() Flow:
  1. Sorter Regeln nach Priorität (1-999)
  2. KRITIS: applies_to='*', conditions=[{type: 'special_case', id: 'kritis_betreiber'}]
     → specialCases = [], kritis NICHT in array → Skip ✓
  3. Sonderfälle (QVDA/TLD/DNS): applies_to='*', conditions=[{special_case}]
     → specialCases = [], NICHT zutreffend → Skip ✓
  4. TK: applies_to='special_case', conditions=[{special_case: 'telekommunikation'}]
     → appliesToMatches = false (specialCases.length === 0) → Skip ✓
  5. anlage1_gross_bwe: applies_to='anlage1', conditions=[{sector_category: 'anlage1'}, {size: 'large_enterprise'}]
     → appliesToMatches = (sectorCategory === 'anlage1') = true ✓
     → Bedingungen: sectorCategory === 'anlage1' ✓, enterpriseSize === 'large_enterprise' ✓
     → GREIFT! → return { result: 'bwe', ... }

Output: ✅ resultTemplates.bwe
Status: ✅ KORREKT
```

---

### TC-003: TK-Anbieter mit 30 Mitarbeitern

```
Input: Q0 Nein → Q1 Ja (TK) → Q1a TK gewählt → Q1b Klein
Expected: nicht betroffen (§28 Abs. 1 Nr. 5, TK nur ab 50 MA)

Frontend Flow:
  Q0: Nein → Q1
  Q1: Ja → Q1a
  Q1a: TK checked → answerMultiSelect()
       → appState.collected.specialCases = ['telekommunikation']
  Q1b: Klein → appState.collected.enterpriseSize = 'small'
       → yesAction = { type: 'question', next: 'q2_sf' }
  Q2_SF: Nein → noAction = { type: 'result', result: 'evaluate_rules' }
       → evaluateRules()

evaluateRules() Flow:
  1-4. (skip, nicht zutreffend)
  5. telekommunikation_klein_nicht_betroffen: applies_to='special_case', conditions=[{special_case: 'telekommunikation'}, {size: 'small_enterprise'}]
     → appliesToMatches = (specialCases.length > 0) = true ✓
     → Bedingungen: specialCases.includes('telekommunikation') ✓, enterpriseSize === 'small_enterprise' ✓
     → GREIFT! → return { result: 'nicht_betroffen', ... }

Output: ✅ resultTemplates.nicht_betroffen
Status: ✅ KORREKT
```

---

### TC-004: TK-Anbieter mit 100 Mitarbeitern

```
Input: Q0 Nein → Q1 Ja (TK) → Q1a TK gewählt → Q1b Mittel
Expected: bwE (§28 Abs. 1 Nr. 5, TK ab 50 MA)

evaluateRules() Flow:
  5. telekommunikation_mittel_bwe: applies_to='special_case', conditions=[{special_case: 'telekommunikation'}, {size: 'medium_enterprise'}]
     → appliesToMatches = true ✓
     → Bedingungen: specialCases.includes('telekommunikation') ✓, enterpriseSize === 'medium_enterprise' ✓
     → GREIFT! → return { result: 'bwe', ... }

Output: ✅ resultTemplates.bwe
Status: ✅ KORREKT
```

---

### TC-005: Anlage 2 mit 120 Mitarbeitern (Medium)

```
Input: Q0 Nein → Q1 Nein → Q2 Anlage2 → Q3 Mittel
Expected: wE (§28 Abs. 2, Anlage 2 ab medium)

evaluateRules() Flow:
  7. anlage2_mittel_we: applies_to='anlage2', conditions=[{sector_category: 'anlage2'}, {size: 'medium_enterprise'}]
     → appliesToMatches = (sectorCategory === 'anlage2') = true ✓
     → Bedingungen: sectorCategory === 'anlage2' ✓, enterpriseSize === 'medium_enterprise' ✓
     → GREIFT! → return { result: 'we', ... }

Output: ✅ resultTemplates.we
Status: ✅ KORREKT
```

---

### TC-006: TLD-Registry

```
Input: Q0 Nein → Q1 Ja (TLD) → Q1a TLD gewählt → Q2_SF Nein
Expected: bwE (§28 Abs. 1 Nr. 3, immer bwE)

evaluateRules() Flow:
  2. tld_immer_bwe: applies_to='*', conditions=[{special_case: 'tld_registry'}]
     → appliesToMatches = true ✓ (applies_to = '*')
     → Bedingung: specialCases.includes('tld_registry') ✓
     → GREIFT! (vor anlage1/2 Rules wegen Priorität 2) → return { result: 'bwe', ... }

Output: ✅ resultTemplates.bwe
Status: ✅ KORREKT
```

---

### TC-007: Nichts zutreffend (Freelancer/Consultant)

```
Input: Q0 Nein → Q1 Nein → Q2 Keiner → noAction
Expected: nicht betroffen

Frontend:
  Q2: 'neither' → appState.collected.sectorCategory = null
       → noAction = { type: 'result', result: 'nicht_betroffen' }
       → showResult('nicht_betroffen')

Output: ✅ resultTemplates.nicht_betroffen
Status: ✅ KORREKT
```

---

## E) REFACTORING-EMPFEHLUNGEN 🔧

### E.1 **KRITISCH: KRITIS-Logik vereinheitlichen**

**Problem:** KRITIS nutzt Shortcut, nicht evaluateRules()

**Lösung:** KRITIS in evaluateRules() integrieren
```javascript
// Entfernen Sie:
yesAction: { type: 'result', result: 'bwe_kritis' },
onYes: () => { appState.collected.specialCases.push('kritis_betreiber'); },

// Ersetzen Sie mit:
yesAction: { type: 'question', next: 'q1' },  // Normale Fortsetzung
onYes: () => {
    appState.collected.specialCases.push('kritis_betreiber');
    // Hinweis: KRITIS wurde erkannt, aber auch andere Kategorien möglich
},

// Und in evaluateRules():
// KRITIS Regel hat Priorität 1 und wird evaluiert wie alle anderen
```

**Vorteil:** Konsistente Logik, User kann auch Q1-Q3 beantworten wenn KRITIS + andere Kategorien

---

### E.2 **KRITISCH: facilityIds-Deadcode entfernen**

**Problem:** facilityIds wird initialisiert, aber nie genutzt
```javascript
appState.collected: {
    specialCases: [],
    sectorCategory: null,
    facilityIds: [],  // ← Wird nie gefüllt/genutzt!
    enterpriseSize: null
}
```

**Lösung:** Entfernen Sie aus appState
```javascript
appState.collected: {
    specialCases: [],
    sectorCategory: null,
    enterpriseSize: null
}
```

---

### E.3 **HOCH: Fehlerbehandlung für User**

**Problem:** Silent Failures bei YAML-Ladefehler
```javascript
function evaluateRules() {
    if (!regelnData || !regelnData.regeln) {
        console.error('Regelwerk nicht geladen');  // Nur in Console!
        return { result: 'nicht_betroffen', ... };  // FALSCH für User!
    }
}
```

**Lösung:**
```javascript
function evaluateRules() {
    if (!regelnData || !regelnData.regeln) {
        console.error('KRITISCH: Regelwerk nicht geladen!');
        return {
            result: 'error',
            appliedRule: null,
            explanation: 'Systemfehler: Regelwerk konnte nicht geladen werden. Bitte versuchen Sie es später erneut.',
            isError: true
        };
    }
}

// In showResult():
if (result.isError) {
    contentCard.innerHTML = `
        <div style="background: #fee; border: 1px solid #c33; padding: 20px; border-radius: 8px; color: #c33;">
            <h2>❌ Fehler beim Laden des Regelwerks</h2>
            <p>${result.explanation}</p>
            <p>Bitte wenden Sie sich an support@dsn.de</p>
        </div>
    `;
    return;
}
```

---

### E.4 **MITTEL: YAML-Schemavalidierung**

**Problem:** Keine Checks auf erforderliche YAML-Struktur

**Lösung:** Validierungsfunktion hinzufügen
```javascript
function validateYAMLSchema() {
    // Prüfe regeln.yaml Struktur
    if (!regelnData.regeln || !Array.isArray(regelnData.regeln)) {
        throw new Error('regeln.yaml: Schlüssel "regeln" muss Array sein');
    }
    
    for (let i = 0; i < regelnData.regeln.length; i++) {
        const rule = regelnData.regeln[i];
        if (!rule.id || !rule.priority || rule.result === undefined) {
            throw new Error(`regeln.yaml[${i}]: Erforderliche Felder: id, priority, result`);
        }
        if (!Array.isArray(rule.conditions)) {
            throw new Error(`regeln.yaml[${i}]: conditions muss Array sein`);
        }
    }
    
    // Prüfe anlage.yaml Struktur
    if (!anlagenData.kategorien || !anlagenData.groessenschwellen) {
        throw new Error('anlage.yaml: Erforderliche Sektion: kategorien, groessenschwellen');
    }
    
    console.log('✅ YAML-Schema validiert');
}

// In loadYAMLData():
async function loadYAMLData() {
    try {
        // ... fetch und parse ...
        validateYAMLSchema();
        console.log('✅ Alle Daten erfolgreich geladen');
    } catch (error) {
        console.error('❌ Fehler beim Laden:', error);
        showErrorScreen(error.message);
    }
}
```

---

### E.5 **MITTEL: Fragen & Optionen aus YAML laden**

**Problem:** Hardcoded Fragen/Optionen → Nicht wartbar

**Lösung:** Dynamisches Laden (Optional, da Fragelogik komplex)
```javascript
// Diese können aus anlage.yaml geladen werden:

// q1a - Sonderfälle:
const q1a_options = anlagenData.sonderfaelle.map(sf => ({
    id: sf.id,
    label: sf.name,
    value: sf.id
}));

// q2 - Sektoren:
const q2_options = [
    { value: 'anlage1', label: `Anlage 1 – ${anlagenData.anlage1.title}` },
    { value: 'anlage2', label: `Anlage 2 – ${anlagenData.anlage2.title}` },
    { value: 'neither', label: 'Keiner dieser Sektoren' }
];

// q3 - Größen:
const q3_options = Object.entries(anlagenData.groessenschwellen).map(([key, data]) => ({
    value: key.replace('_enterprise', ''),
    label: `${data.name} (${data.employees_min}-${data.employees_max || '∞'} MA)`
}));
```

---

### E.6 **MITTEL: Pflichten aus anlage.yaml laden**

**Problem:** Pflichten in Frontend hardcoded, nicht aus anlage.yaml

**Lösung:**
```javascript
function getObligations(result) {
    const obligation = anlagenData.pflichten[result];
    if (!obligation) return [];
    
    if (obligation.required) {
        return Object.values(obligation.required);
    } else if (obligation.note) {
        return obligation.note;
    }
    return [];
}

// In showResult():
const obligations = getObligations(evaluation.result);
```

---

### E.7 **NIEDRIG: Console-Logging für Produktion entfernen**

**Current:**
```javascript
console.log('evaluateRules() mit Input:', appState.collected);
console.log('Sorted Rules:', sortedRules.map(...));
console.log(`Prüfe Regel: ${rule.id}`);
console.log(`✓ Regel greift: ${rule.id}`);
```

**Lösung:**
```javascript
const DEBUG = false;  // oder via URL-Parameter

if (DEBUG) {
    console.log('evaluateRules() mit Input:', appState.collected);
    // ...
}
```

---

### E.8 **NIEDRIG: Multisprachen-Support vorbereiten**

**Struktur vorschlagen:**
```yaml
# anlage.yaml
translations:
  de:
    kategorie_bwe: "Besonders wichtige Einrichtung (bwE)"
    kategorie_we: "Wichtige Einrichtung (wE)"
  en:
    kategorie_bwe: "Critical Entity (CE)"
    kategorie_we: "Important Entity (IE)"
```

---

## F) ZUSAMMENFASSUNG & EMPFEHLUNGEN

### Status-Übersicht

| Aspekt | Status | Kritikalität |
|--------|--------|--------------|
| **Logik-Korrektheit** | ✅ Gut | – |
| **ID-Konsistenz** | ⚠️ Größtenteils OK | Niedrig |
| **KRITIS-Implementierung** | ⚠️ Funktioniert, aber inkonsistent | Mittel |
| **Fehlerbehandlung** | 🔴 Kritisch | **KRITISCH** |
| **YAML-Validierung** | 🔴 Nicht vorhanden | **KRITISCH** |
| **Hardcoding** | ⚠️ Viel Potential | Mittel |
| **Race Conditions** | ✅ Keine erkannt | – |

---

### Sofort-Fixes (Produktionsfreigabe blockierend)

1. **Fehlerbehandlung für YAML-Ladefehler** (E.3)
   - User muss Fehler sehen, nicht stille Fallbacks
   - ~30 Min

2. **KRITIS-Logik vereinheitlichen** (E.1)
   - Entweder Shortcut ODER evaluateRules, nicht gemischt
   - ~20 Min

3. **facilityIds entfernen** (E.2)
   - Deadcode aufräumen
   - ~10 Min

---

### Wichtige Verbesserungen (Vor Produktion)

4. **YAML-Schemavalidierung** (E.4)
   - Typos/Fehler in YAML früh erkennen
   - ~45 Min

5. **Optionen aus YAML laden** (E.5)
   - Wartbarkeit für zukünftige Anpassungen
   - ~60 Min

---

### Optionale Verbesserungen (Post-Launch)

6. **Pflichten aus YAML** (E.6)
7. **Logging-Level konfigurierbar** (E.7)
8. **Multisprachen-Struktur** (E.8)

---

### **GESAMTBEWERTUNG: BEDINGT PRODUKTIONSREIF** ⚠️

**Grünes Licht wenn Sie implementieren:**
- ✅ Fehlerbehandlung für User (E.3)
- ✅ KRITIS vereinheitlichen (E.1)
- ✅ facilityIds aufräumen (E.2)

**Dann ist das System robust genug für Production.** 🚀

