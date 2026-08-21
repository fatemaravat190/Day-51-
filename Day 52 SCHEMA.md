# SCHEMA.md — Firestore Data Design

Firestore is a **document database**, not relational — so "tables" below are Firestore **collections**, "rows" are **documents**, and there are no foreign keys, only embedded/nested structures.

## 1. Collections & Documents

### Collection: `regions`

One document per city. Document ID = lowercase city slug (`dubai`, `abudhabi`, `riyadh`, `jeddah`, `dammam`).

```
regions (collection)
 ├── dubai (document)
 ├── abudhabi (document)
 ├── riyadh (document)
 ├── jeddah (document)
 └── dammam (document)
```

### Document shape (each city document)

```json
{
  "city": "Dubai",
  "country": "UAE",
  "years": {
    "2014": {
      "population": 0,
      "totalBeds": 0,
      "icuBeds": 0,
      "icuOccupancy": 0,
      "specialtyBeds": {
        "cardiology": 0,
        "oncology": 0,
        "maternity": 0,
        "orthopedics": 0
      },
      "occupancyRate": 0,
      "patientVolumePublic": 0,
      "patientVolumePrivate": 0,
      "surgicalVolumePublic": 0,
      "surgicalVolumePrivate": 0,
      "source": "string — citation for this year's figures",
      "notes": "string — gap explanations, caveats"
    },
    "2015": { "...": "same shape" },
    "2016": { "...": "same shape" },
    "2017": { "...": "same shape" },
    "2018": { "...": "same shape" },
    "2019": { "...": "same shape" },
    "2020": { "...": "same shape" },
    "2021": { "...": "same shape" },
    "2022": { "...": "same shape" },
    "2023": { "...": "same shape" }
  }
}
```

## 2. Field Reference

| Field | Type | Required | Notes |
|---|---|---|---|
| `city` | string | Yes | Display name |
| `country` | string | Yes | `"UAE"` or `"KSA"` |
| `years.<YYYY>.population` | number \| null | Yes (or null + note) | Raw headcount, not millions — unit standardized project-wide |
| `years.<YYYY>.totalBeds` | number \| null | Yes | |
| `years.<YYYY>.icuBeds` | number \| null | Yes | **Constraint:** must be ≤ `totalBeds` |
| `years.<YYYY>.icuOccupancy` | number \| null | Yes | Percentage, 0–100 |
| `years.<YYYY>.specialtyBeds` | map | Yes | Sub-fields as above; **Constraint:** sum should be a plausible subset of `totalBeds` |
| `years.<YYYY>.occupancyRate` | number \| null | Yes | Percentage, 0–100 |
| `years.<YYYY>.patientVolumePublic` | number \| null | Yes | |
| `years.<YYYY>.patientVolumePrivate` | number \| null | Yes | |
| `years.<YYYY>.surgicalVolumePublic` | number \| null | Yes | |
| `years.<YYYY>.surgicalVolumePrivate` | number \| null | Yes | |
| `years.<YYYY>.source` | string | Yes | |
| `years.<YYYY>.notes` | string | Yes (may be empty string) | Required whenever any field above is `null` |

## 3. Relationships

None — this is a single flat collection with no cross-document references. "Relationships" that exist are **logical, not structural**:

- Each city document is independent; the dashboard aggregates across all 5 client-side.
- `icuBeds` is logically a subset of `totalBeds` (enforced by validation script, not by the database).
- `specialtyBeds.*` values are logically a subset of `totalBeds` (enforced by validation script).
- `patientVolumePublic` + `patientVolumePrivate` together represent total patient volume for the year (used for the private-share KPI callout).

## 4. Constraints (enforced in application code / validation script, not Firestore)

Firestore has no native column constraints, so these are enforced by:
1. The **JSON schema template** (Day 2) — every year key 2014–2023 must exist.
2. The **validation script** (Day 4) — loops all 5 files/documents and flags:
   - Missing required fields for any year
   - `icuBeds > totalBeds`
   - `sum(specialtyBeds.*) > totalBeds`
   - A `null` field without an accompanying `notes` explanation
3. **Firestore Security Rules** (Day 8) — not data-shape constraints, but access constraints: only `request.auth != null` may read the `regions` collection; no client-side writes permitted from the deployed app (data entry happens via console/scripts, not the live UI).

## 5. PRD User-Story Validation

| User story (from PRD) | Schema field(s) that satisfy it |
|---|---|
| View bed density / bed gap per city | `totalBeds`, `population`, `occupancyRate` |
| View ICU capacity per city | `icuBeds`, `icuOccupancy` |
| View 10-year trend | `years.2014` … `years.2023` keys present on every document |
| Forecast bed gap to 2040 | `population`, `totalBeds`, `occupancyRate` (base years for the client-side forecast engine) |
| Specialty-wise forecast adjustment | `specialtyBeds.*` |
| Public vs. private patient volume trend + share shift | `patientVolumePublic`, `patientVolumePrivate` |
| Public vs. private surgical volume trend + share shift | `surgicalVolumePublic`, `surgicalVolumePrivate` |
| Methodology transparency / data gaps disclosed | `source`, `notes` |
| Restrict data to logged-in users | Enforced via Security Rules, not a data field |

Every PRD-required view is backed by a field in this schema — no restructuring should be needed after Day 2's lock-in.
