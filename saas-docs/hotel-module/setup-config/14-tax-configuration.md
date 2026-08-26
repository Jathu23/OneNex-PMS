# Setup & Configuration — 14: Tax Configuration
> Hotel Module → Setup & Configuration → Area 14 of 16
> Base Market: Sri Lanka | Framework: VAT + Service Charge + TDL
> Foundation for: Invoice Generation, Folio Charge Calculation, VAT Filing, IRD Compliance

---

## Sri Lanka Hotel Tax Structure

```
Sri Lanka la hotel billing = 3 layer tax:

LAYER 1: Service Charge       10%  (mandatory — goes to staff, not govt)
LAYER 2: VAT                  18%  (on room + service charge combined)
LAYER 3: Tourism Dev Levy      1%  (on room charge only — govt tourism fund)

CALCULATION ORDER (very important):
  Base Room Rate:          LKR 10,000
  + Service Charge 10%:   LKR  1,000   ← on base rate
  ─────────────────────────────────────
  VAT Base:               LKR 11,000
  + VAT 18%:              LKR  1,980   ← on (base + service charge)
  + TDL 1%:               LKR    100   ← on base rate only
  ─────────────────────────────────────
  TOTAL:                  LKR 13,080

Wrong order = wrong tax = IRD (Inland Revenue) issue.
```

---

## Sri Lanka Tax Authorities

```
TAX BODY          TAX            FILING
────────────────────────────────────────────────────
IRD               VAT            Monthly (if > LKR 80M/year)
                                 Quarterly (if < LKR 80M/year)
SLTDA             TDL            Monthly
Dept of Labour    Service Charge Distributed to staff (not filed)
IRD               SSCL 2.5%     On turnover > LKR 25M/quarter
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | India/Western tax logic. Sri Lanka service charge as pre-tax item not handled correctly. TDL completely missing. |
| Mews | Generic VAT % config. No TDL. No service charge distribution tracking. |
| Cloudbeds | Single tax % only. Cannot handle 3-layer tax correctly. Manual adjustment needed every invoice. |
| All systems | No SLTDA registration number on invoice. Non-compliant for Sri Lanka tourism businesses. |

---

## Our Design Principles

### 1. Tax Framework — Country Selector
```
PROPERTY TAX FRAMEWORK:
  Country: Sri Lanka   ← selected

  Pre-built frameworks:
    ○ Sri Lanka   ← (primary market)
    ○ India
    ○ UAE
    ○ Singapore
    ○ Custom

  Sri Lanka selected → loads correct components automatically.
  Hotel only fills registration numbers + toggles.
```

### 2. Sri Lanka Tax Components
```
COMPONENT 1: Service Charge
  Name:           Service Charge
  Rate:           10%
  Applies to:     All F&B + Room charges
  Calculation:    On base amount (before VAT)
  Goes to:        Staff distribution (not govt remittance)
  Show on invoice: Yes
  Label:          "Service Charge @ 10%"
  Configurable?:  Rate fixed by law (10%) — hotel cannot change

COMPONENT 2: VAT
  Name:           Value Added Tax
  Rate:           18%
  Applies to:     (Base + Service Charge) combined
  Registration:   VAT Number (hotel enters)
  Filing:         Monthly / Quarterly (system generates report)
  Show on invoice: Yes
  Label:          "VAT @ 18%"
  Configurable?:  Rate fixed — 18%

COMPONENT 3: Tourism Development Levy (TDL)
  Name:           Tourism Development Levy
  Rate:           1%
  Applies to:     Room charges only (not F&B, not spa)
  Authority:      SLTDA
  Registration:   SLTDA Reg Number (hotel enters)
  Show on invoice: Yes
  Label:          "TDL @ 1%"
  Configurable?:  Rate fixed — 1%

COMPONENT 4: SSCL (Social Security Contribution Levy)
  Name:           SSCL
  Rate:           2.5%
  Applies to:     Turnover (not per transaction — quarterly calc)
  Threshold:      LKR 25M per quarter
  Show on invoice: No (business-level tax, not per invoice)
  V1: Track turnover. Flag when approaching threshold.
```

### 3. Tax Per Charge Type
```
CHARGE TYPE             Service Charge   VAT    TDL
────────────────────────────────────────────────────────
Room Charge             ✅ 10%           ✅ 18%  ✅ 1%
Breakfast (incl. pkg)   ✅ 10%           ✅ 18%  ☐
Restaurant (à la carte) ✅ 10%           ✅ 18%  ☐
Bar / Alcohol           ✅ 10%           ✅ 18%  ☐
Spa / Massage           ✅ 10%           ✅ 18%  ☐
Laundry                 ✅ 10%           ✅ 18%  ☐
Parking                 ☐               ✅ 18%  ☐
Conference Room         ✅ 10%           ✅ 18%  ☐
Airport Transfer        ☐               ✅ 18%  ☐
Early Check-in          ✅ 10%           ✅ 18%  ✅ 1%
Late Check-out          ✅ 10%           ✅ 18%  ✅ 1%
Mini Bar - Food         ✅ 10%           ✅ 18%  ☐
Mini Bar - Alcohol      ✅ 10%           ✅ 18%  ☐
```

### 4. Calculation Engine — Step by Step
```
ROOM CHARGE EXAMPLE:
  Input:            LKR 15,000 (base rate, 1 night)

  Step 1: Service Charge
    SC = 15,000 × 10% = LKR 1,500
    SC Base = 15,000 + 1,500 = LKR 16,500

  Step 2: VAT
    VAT = 16,500 × 18% = LKR 2,970

  Step 3: TDL
    TDL = 15,000 × 1% = LKR 150  ← on BASE only, not SC+VAT

  Total:
    Base:           15,000
    Service Charge:  1,500
    VAT:             2,970
    TDL:               150
    ──────────────────────
    GRAND TOTAL:    19,620

RESTAURANT CHARGE EXAMPLE:
  Input:            LKR 3,500 (dinner bill)

  Step 1: Service Charge
    SC = 3,500 × 10% = LKR 350

  Step 2: VAT
    VAT = (3,500 + 350) × 18% = LKR 693

  Step 3: TDL
    TDL = N/A (restaurant, not room)

  Total:
    Base:            3,500
    Service Charge:    350
    VAT:               693
    ──────────────────────
    GRAND TOTAL:     4,543
```

### 5. Tax-Inclusive vs Tax-Exclusive Display
```
DISPLAY CONFIG:
  OTA listings:    Tax-inclusive (guest sees total)
  Direct booking:  Hotel chooses:
    "LKR 15,000/night + taxes"    (exclusive — shows taxes at checkout)
    "LKR 19,620/night all-in"     (inclusive — total upfront)

  Invoice always shows full breakdown regardless of display mode.
  (Mandatory for IRD compliance)
```

### 6. Tax Registration Setup
```
TAX REGISTRATION (per property):

  VAT Registration Number:    VAT/LKR/XXXXXXXX/XXXX
  SLTDA Registration Number:  [Tourist Board number]
  Business Registration No:   [Companies Registrar]

  Multi-property:
    Each property has own VAT number if separate legal entity
    OR same VAT number if one company owns multiple properties

  Invoice mandatory fields (IRD requirement):
    ✅ Hotel VAT number
    ✅ Hotel legal name + address
    ✅ Invoice number (sequential — cannot skip)
    ✅ Invoice date
    ✅ Guest name
    ✅ Guest VAT number (if business guest — for input tax credit)
    ✅ Each charge line with tax breakdown
    ✅ SC amount
    ✅ VAT amount
    ✅ TDL amount (if applicable)
    ✅ Grand total
```

### 7. VAT Filing Report
```
MONTHLY VAT REPORT (auto-generated):
  Period:       October 2025
  Property:     The Grand Colombo

  OUTPUT TAX (collected from guests):
    Room Revenue:         LKR 2,450,000
    F&B Revenue:          LKR   890,000
    Other:                LKR   120,000
    Total Taxable:        LKR 3,460,000
    VAT Collected (18%):  LKR   622,800

  INPUT TAX (paid to suppliers — Phase 2):
    Purchases:            LKR   540,000
    VAT Paid:             LKR    97,200

  NET VAT PAYABLE:        LKR   525,600

  TDL REPORT (SLTDA):
    Room Revenue:         LKR 2,450,000
    TDL Due (1%):         LKR    24,500

V1: Output tax report only (what was collected).
Phase 2: Input tax tracking (what was paid to suppliers).
```

### 8. Foreign Guest — Currency & Tax
```
FOREIGN CURRENCY BOOKINGS:
  OTA books in USD:    USD 120/night
  System converts:     LKR equivalent at today's rate
  Tax calculated:      In LKR (always)
  Invoice shows:       LKR amounts (IRD requirement)
                       + "Paid via OTA in USD 120" notation

  Exchange rate source:
    V1: Hotel enters rate manually
    Phase 2: Auto-fetch from CBSL (Central Bank of Sri Lanka)
```

---

## Data Model

```
PropertyTaxConfig
  hotel_id
  country_code                  "LK"
  tax_framework                 SRI_LANKA / INDIA / UAE / CUSTOM
  vat_number                    "VAT/LKR/XXXXXXXX/XXXX"
  sltda_reg_number              nullable
  business_reg_number           nullable
  legal_name                    "Grand Hotels (Pvt) Ltd"
  tax_inclusive_display         bool
  sscl_enabled                  bool
  sscl_threshold_quarterly      decimal (25,000,000)

TaxComponent
  id, hotel_id
  name                          "VAT" / "Service Charge" / "TDL" / "SSCL"
  code                          VAT / SERVICE_CHARGE / TDL / SSCL
  rate                          decimal (18.0 / 10.0 / 1.0 / 2.5)
  calc_base                     BASE / BASE_PLUS_SC / TURNOVER
  is_govt_remittable            bool (SC = false, VAT/TDL = true)
  is_per_transaction            bool (SSCL = false — quarterly)
  is_active                     bool

ChargeTypeTaxMapping
  id, hotel_id
  charge_type                   ROOM / BREAKFAST / RESTAURANT / BAR /
                                SPA / LAUNDRY / PARKING / CONFERENCE /
                                AIRPORT_TRANSFER / EARLY_CHECKIN /
                                LATE_CHECKOUT / MINIBAR
  tax_component_id              FK → TaxComponent
  applies                       bool

FolioChargeBreakdown (runtime — per folio charge)
  folio_charge_id
  base_amount                   decimal
  service_charge_amount         decimal
  vat_amount                    decimal
  tdl_amount                    decimal
  total_amount                  decimal

VATFilingPeriod (runtime)
  id, hotel_id
  period_month                  int
  period_year                   int
  total_revenue                 decimal
  total_vat_collected           decimal
  total_tdl_collected           decimal
  report_generated_at           timestamp
  report_url                    nullable
  status                        DRAFT / SUBMITTED
```

---

## Key Relationships

```
Hotel → PropertyTaxConfig (one per property)
Hotel → TaxComponent (4 components for Sri Lanka)
Hotel → ChargeTypeTaxMapping (one per charge type × tax component)

FolioCharge → FolioChargeBreakdown (tax breakdown stored at posting time)
VATFilingPeriod → FolioCharges (aggregated for filing period)

Invoice generation:
  Folio + PropertyTaxConfig + FolioChargeBreakdowns → IRD-compliant PDF
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Sri Lanka tax framework (VAT + SC + TDL) | ✅ | | |
| Auto-calculate SC → VAT → TDL in correct order | ✅ | | |
| Tax per charge type mapping | ✅ | | |
| VAT + SLTDA registration numbers | ✅ | | |
| IRD-compliant invoice generation | ✅ | | |
| Tax-inclusive / exclusive display | ✅ | | |
| Monthly VAT output report | ✅ | | |
| Monthly TDL report (SLTDA) | ✅ | | |
| SSCL threshold tracking + alert | ✅ | | |
| Business guest VAT number on invoice | ✅ | | |
| Input tax tracking (supplier purchases) | | ✅ | |
| IRD filing export format | | ✅ | |
| Exchange rate auto-fetch (CBSL) | | ✅ | |
| Multi-currency invoice | | ✅ | |
| India GST framework | | ✅ | |
| UAE VAT framework | | ✅ | |
| International multi-country tax engine | | | ✅ |
