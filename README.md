# Sales Commission Split Calculator

A commission calculation engine for HVAC sales projects that distributes profit among multiple parties using a "waterfall" methodology.

## Table of Contents

- [Overview](#overview)
- [Business Rules & Constants](#business-rules--constants)
- [Core Algorithm](#core-algorithm)
- [Calculation Scenarios](#calculation-scenarios)
- [Data Structures](#data-structures)
- [Pseudocode Implementation](#pseudocode-implementation)
- [Examples](#examples)

---

## Overview

The calculator divides a project's commission pool (100%) among up to four parties:

| Party | Role |
|-------|------|
| **Owner** | Building owner who may have influence on the sale |
| **Specialist** | Product specialist (e.g., VRV, Geothermal) |
| **Engineering Team** | Design engineers who specify equipment |
| **Contractor Team** | Installing contractor |

Each party has **Outside Sales** (field sales reps) and **Inside Sales** (project managers/support).

---

## Business Rules & Constants

### Fixed Inside Sales Rates

| Constant | Value | Description |
|----------|-------|-------------|
| `INSIDE_ENG_BASE` | 7.0% | Engineering team inside sales |
| `INSIDE_CONT_BASE` | 5.0% | Contractor team inside sales |
| `TOTAL_INSIDE_CAP` | 12.0% | Maximum combined inside sales |

### Specialist Cannibalization

When a Specialist is involved, they "cannibalize" inside sales from both teams:

| Constant | Value | Description |
|----------|-------|-------------|
| `SPEC_INSIDE_TAX` | 2.0% | Amount taken from each team's inside budget |

This results in:
- Engineering Inside: 7% → 5%
- Contractor Inside: 5% → 3%
- Specialist Inside: 0% → 4% (2% from each side)

### Engineering Influence Presets

| Preset | Eng % | Cont % | Use Case |
|--------|-------|--------|----------|
| Flat Spec | 65% | 35% | Engineer specified exact product |
| Strong Spec | 55% | 45% | Strong engineering involvement |
| Basis of Design | 45% | 55% | Engineer set basis, substitutions allowed |
| Approved Equal | 35% | 65% | Open spec with approved equals (default) |
| Out of Town | 17% | 83% | Minimal engineering relationship |

### Specialist Default Rates

| Specialist Type | Default Outside % |
|-----------------|-------------------|
| Dadanco | 10% |
| Nortek AHU | 10% |
| VRV | 15% |
| HT Material | 10% |
| Geothermal | 20% |
| Williams IPS | 10% |
| Flow Environmental | 10% |
| AJ Surgery Suite | 15% |
| Coop Purchasing | 10% |

---

## Core Algorithm

### The Waterfall (Standard Flow)

```
1. START with 100% pool

2. DEDUCT Specialist Outside % (if enabled)
   → remaining_pool = 100 - specialist_outside_pct

3. DEDUCT Owner Outside % (if enabled)
   → remaining_pool = remaining_pool - owner_outside_pct

4. SPLIT remaining pool between Engineering and Contractor
   → eng_gross = remaining_pool × engineering_influence
   → cont_gross = remaining_pool - eng_gross

5. DEDUCT Inside Sales from each team's gross
   → eng_outside_net = eng_gross - eng_inside_rate
   → cont_outside_net = cont_gross - cont_inside_rate

6. VALIDATE that gross amounts cover inside requirements
   → If eng_gross < eng_inside_rate: ERROR
   → If cont_gross < cont_inside_rate: ERROR
```

### Inside Sales Rate Calculation

```
IF specialist_enabled:
    eng_inside_rate = 7% - 2% = 5%
    cont_inside_rate = 5% - 2% = 3%
    spec_inside_left = 2%
    spec_inside_right = 2%
ELSE:
    eng_inside_rate = 7%
    cont_inside_rate = 5%
    spec_inside_left = 0%
    spec_inside_right = 0%
```

---

## Calculation Scenarios

### Scenario 1: Standard (Eng + Contractor)

The default scenario with both Engineering and Contractor teams.

```
Inputs:
  - owner_pct: 0-50%
  - specialist_pct: 0-50% (if enabled)
  - engineering_influence: 0.17-0.65

Calculation:
  remaining = 100 - owner_pct - specialist_pct
  eng_gross = remaining × engineering_influence
  cont_gross = remaining - eng_gross
  eng_outside = eng_gross - eng_inside_rate
  cont_outside = cont_gross - cont_inside_rate
```

### Scenario 2: Contractor Only (Design/Build)

No engineering team. Contractor takes the entire split.

```
Inputs:
  - owner_pct: 0-50%
  - specialist_pct: 0-50% (if enabled)

Calculation:
  remaining = 100 - owner_pct - specialist_pct
  eng_gross = 0
  cont_gross = remaining
  eng_inside = 0
  cont_inside = 12%  (takes full inside budget)
  cont_outside = cont_gross - 12

With Specialist:
  spec_inside = 4%
  cont_inside = 8%
  cont_outside = cont_gross - 12
```

### Scenario 3: Owner Purchasing Direct

When "Owner Purchasing Direct" is enabled, the Owner becomes the primary buyer. Four sub-scenarios:

#### 3a: Owner Only (No Engineer, No Contractor)

```
remaining = 100 - specialist_pct
owner_outside = remaining - 12
owner_inside = 12

With Specialist:
  spec_inside = 4%
  owner_inside = 8%
```

#### 3b: Owner + Engineer (No Contractor)

```
remaining = 100 - specialist_pct
eng_gross = remaining × engineering_influence
owner_gross = remaining - eng_gross

eng_inside = 7%
owner_inside = 5%

eng_outside = eng_gross - eng_inside - spec_inside_left
owner_outside = owner_gross - owner_inside - spec_inside_right

With Specialist:
  eng_inside = 5%
  owner_inside = 3%
  spec_inside = 4%
```

#### 3c: Owner + Contractor (No Engineer)

```
remaining = 100 - specialist_pct

cont_outside = 10%
cont_inside = 2%
owner_inside = 10%
owner_outside = remaining - 22  (typically 78%)

With Specialist:
  spec_inside = 4%
  owner_inside = 8%
  cont_inside = 0%
```

#### 3d: Owner + Engineer + Contractor

```
remaining = 100 - specialist_pct

# Contractor gets fixed amount
cont_outside = 10%
cont_inside = 2%

# Remaining pool splits between Eng and Owner
pool_after_cont = remaining - 12
eng_gross = pool_after_cont × engineering_influence
owner_gross = pool_after_cont - eng_gross

eng_inside = 7%
owner_inside = 3%

eng_outside = eng_gross - eng_inside - spec_inside_left
owner_outside = owner_gross - owner_inside - spec_inside_right

With Specialist:
  eng_inside = 5%
  owner_inside = 1%
  spec_inside = 4%
```

---

## Data Structures

### Company

```javascript
{
  id: "e1",
  name: "CMTA Engineers",
  outside: [
    { name: "Ben E.", split: 50 },
    { name: "Jeff F.", split: 50 }
  ],
  inside: [
    { name: "Evan J.", split: 100 }
  ]
}
```

### Specialist Type

```javascript
{
  id: "s1",
  name: "VRV",
  defaultPct: 15,
  outside: [
    { name: "Vishal", split: 100 }
  ],
  inside: [
    { name: "Vishal", split: 100 }
  ]
}
```

### Commission Row (Output)

```javascript
{
  team: "Engineering",      // Owner | Specialist | Engineering | Contractor
  person: "Ben E.",
  type: "Outside Sales",    // Outside Sales | Inside Sales
  split: 14.625            // Final percentage of total commission
}
```

### Product Lines

Companies can have different rep assignments per product line:

```javascript
{
  id: "e1",
  name: "CMTA Engineers",
  vent: {
    outside: [...],
    inside: [...]
  },
  hydronic: {
    outside: [...],
    inside: [...]
  },
  industrial: {
    outside: [...],
    inside: [...]
  }
}
```

---

## Pseudocode Implementation

```python
def calculate_commission(
    owner_enabled: bool,
    owner_pct: float,
    owner_direct: bool,
    specialist_enabled: bool,
    specialist_pct: float,
    engineering_influence: float,  # 0.17 to 0.65
    contractor_only: bool,
    engineer_is_none: bool,
    contractor_is_none: bool
) -> dict:

    # Initialize results
    result = {
        'eng_outside': 0, 'eng_inside': 0,
        'cont_outside': 0, 'cont_inside': 0,
        'owner_outside': 0, 'owner_inside': 0,
        'spec_outside': 0, 'spec_inside_left': 0, 'spec_inside_right': 0
    }

    # Set specialist outside
    if specialist_enabled:
        result['spec_outside'] = specialist_pct

    # Calculate remaining pool after specialist
    remaining = 100 - specialist_pct

    # Determine inside rates based on specialist
    if specialist_enabled:
        eng_inside_rate = 5.0
        cont_inside_rate = 3.0
        result['spec_inside_left'] = 2.0
        result['spec_inside_right'] = 2.0
    else:
        eng_inside_rate = 7.0
        cont_inside_rate = 5.0

    # === OWNER PURCHASING DIRECT ===
    if owner_direct:
        if engineer_is_none and contractor_is_none:
            # Scenario 3a: Owner Only
            result['owner_outside'] = remaining - 12
            result['owner_inside'] = 12
            if specialist_enabled:
                result['owner_inside'] = 8

        elif not engineer_is_none and contractor_is_none:
            # Scenario 3b: Owner + Engineer
            eng_gross = remaining * engineering_influence
            owner_gross = remaining - eng_gross
            result['eng_inside'] = eng_inside_rate
            result['owner_inside'] = 12 - eng_inside_rate
            result['eng_outside'] = eng_gross - result['eng_inside'] - result['spec_inside_left']
            result['owner_outside'] = owner_gross - result['owner_inside'] - result['spec_inside_right']

        elif engineer_is_none and not contractor_is_none:
            # Scenario 3c: Owner + Contractor
            result['cont_outside'] = 10
            result['cont_inside'] = 2
            result['owner_inside'] = 10
            result['owner_outside'] = remaining - 22
            if specialist_enabled:
                result['owner_inside'] = 8
                result['cont_inside'] = 0

        else:
            # Scenario 3d: Owner + Engineer + Contractor
            result['cont_outside'] = 10
            result['cont_inside'] = 2
            pool_after_cont = remaining - 12
            eng_gross = pool_after_cont * engineering_influence
            owner_gross = pool_after_cont - eng_gross
            result['eng_inside'] = eng_inside_rate
            result['owner_inside'] = 12 - eng_inside_rate - 2  # Remaining after eng and cont inside
            result['eng_outside'] = eng_gross - result['eng_inside'] - result['spec_inside_left']
            result['owner_outside'] = owner_gross - result['owner_inside'] - result['spec_inside_right']

    # === CONTRACTOR ONLY ===
    elif contractor_only:
        remaining = remaining - owner_pct
        result['owner_outside'] = owner_pct
        result['cont_inside'] = 12 if not specialist_enabled else 8
        result['cont_outside'] = remaining - 12

    # === STANDARD (Eng + Contractor) ===
    else:
        remaining = remaining - owner_pct
        result['owner_outside'] = owner_pct

        eng_gross = remaining * engineering_influence
        cont_gross = remaining - eng_gross

        result['eng_inside'] = eng_inside_rate
        result['cont_inside'] = cont_inside_rate
        result['eng_outside'] = eng_gross - eng_inside_rate - result['spec_inside_left']
        result['cont_outside'] = cont_gross - cont_inside_rate - result['spec_inside_right']

    return result
```

---

## Examples

### Example 1: Standard Split (Approved Equal)

**Inputs:**
- Owner: None
- Specialist: None
- Engineering Influence: 35% (Approved Equal)

**Calculation:**
```
remaining_pool = 100%
eng_gross = 100 × 0.35 = 35%
cont_gross = 100 - 35 = 65%

eng_inside = 7%
cont_inside = 5%

eng_outside = 35 - 7 = 28%
cont_outside = 65 - 5 = 60%
```

**Result:**
| Team | Outside | Inside | Total |
|------|---------|--------|-------|
| Engineering | 28% | 7% | 35% |
| Contractor | 60% | 5% | 65% |

---

### Example 2: With Specialist (VRV at 15%)

**Inputs:**
- Owner: None
- Specialist: VRV at 15%
- Engineering Influence: 35%

**Calculation:**
```
remaining_pool = 100 - 15 = 85%
eng_gross = 85 × 0.35 = 29.75%
cont_gross = 85 - 29.75 = 55.25%

eng_inside = 5% (reduced from 7)
cont_inside = 3% (reduced from 5)
spec_inside = 4% (2 from each side)

eng_outside = 29.75 - 5 - 2 = 22.75%
cont_outside = 55.25 - 3 - 2 = 50.25%
```

**Result:**
| Team | Outside | Inside | Total |
|------|---------|--------|-------|
| Specialist | 15% | 4% | 19% |
| Engineering | 22.75% | 5% | 27.75% |
| Contractor | 50.25% | 3% | 53.25% |

---

### Example 3: Owner Purchasing Direct (Owner Only)

**Inputs:**
- Owner Purchasing Direct: Yes
- Engineer: None
- Contractor: None
- Specialist: None

**Result:**
| Team | Outside | Inside | Total |
|------|---------|--------|-------|
| Owner | 88% | 12% | 100% |

---

### Example 4: Owner Direct + Contractor

**Inputs:**
- Owner Purchasing Direct: Yes
- Engineer: None
- Contractor: Selected
- Specialist: None

**Result:**
| Team | Outside | Inside | Total |
|------|---------|--------|-------|
| Owner | 78% | 10% | 88% |
| Contractor | 10% | 2% | 12% |

---

## Validation Rules

1. **Squeeze Check**: Gross amounts must cover inside sales requirements
   - `eng_gross >= eng_inside_rate`
   - `cont_gross >= cont_inside_rate`

2. **Owner Purchasing Direct**: Owner slider is disabled (owner takes calculated amount, not slider value)

3. **Contractor Only**: Engineering team is excluded, Contractor takes full 12% inside budget

4. **Total must equal 100%**: Sum of all outside + inside percentages should equal 100%

---

## License

Proprietary - Internal Use Only
