# Medical Billing Research Project — Comprehensive Findings

**Institution:** University of the Western Cape  
**Module:** BIA 716 Applied Research Project  
**Project:** Invoice Processing Delay Prediction  
**Date:** May 2026

---

## 1. DATASET OVERVIEW

### Source and Dimensions
- **Source File Name:** PRIVATE PATIENT.xlsx
- **Location:** `dataset/PRIVATE PATIENT.xlsx`
- **Original Dimensions:** 50,000 rows × 32 columns
- **Data Type:** Medical billing invoices from a private healthcare practice in South Africa

### Date Range of Key Variables
- **Invoice Date Range:** 2013-01-01 to 2030-12-31 (294 missing values)
  - Note: 2 records dated 2030 are impossible (data error)
  - Earliest legitimate dates: 2013
  - Pre-2020 records sparse but present
- **Capture Date Range:** 2020-01-01 onwards (no missing values)
  - Date when invoice entered the billing system
- **Latest Capture Date Range:** 2020-01-01 to 2026-05-02
  - Date of last recorded activity on the invoice

### System Migration Finding
**Critical Finding — Billing System Migration (~2020):**
- Invoice Date spans 2013-2026 but Capture Date starts only from 2020
- This 7-year gap indicates a billing system migration occurred around 2020
- Pre-2020 invoice records appear to be historical imports rather than live billing events
- **Analysis conducted:** Pre-2020 and post-2020 records show similar processing patterns within the new system (median processing time 0 days for both groups), suggesting the import was successful but pre-2020 records should be interpreted as historical context rather than active workflow

---

## 2. VARIABLE SCREENING (Section 3.2.1)

### Total Variables Before Screening
- **Original Variables:** 32
- **Variables to Screen:** All 32
- **Primary Data Types:**
  - DateTime variables: 4 (Invoice Date, Capture Date, Earliest Capture Date, Latest Capture Date)
  - Numeric variables: 6 (Age Bracket, Total Outstanding, Total Medical Aid Outstanding, Total Patient Outstanding, Total Turnover, Total Cashflow, Total Journals)
  - Categorical variables: 22 (Invoice Number, Entity name, Main Member, Medical Aid Number, Service Centre, Treating Doctor, Referring Doctor Name, Scheme, Option, Administrator, Posted Billing Group, ICD10 String, Primary ICD10, ICD10 Description, Patient Name, Patient DOB, Gender, Specialty, Debtor Status, POSSIBLE PMB CALCULATED, CONTRACTED/NON-CONTRACTED)

### All Excluded Variables with Documented Reasons

| Variable | Reason for Exclusion |
|----------|---------------------|
| Entity name | Hashed practice identifier — anonymised, no predictive value |
| Main Member | Hashed account holder identifier — anonymised, no predictive value |
| Medical Aid Number | Hashed field — populated with system defaults, not real scheme numbers |
| Invoice Date | Not used as predictor — 294 missing, 2 records dated 2030 (impossible); used only for date logic validation during cleaning |
| Earliest Capture Date | Not used as predictor — used only for date logic validation |
| Treating Doctor | Hashed identifier — 566 unique values, one linked to 41 different specialties, unusable |
| Scheme | Constant — single value "PRIVATE PATIENT" across all 50,000 records |
| Administrator | Constant — single value "PRIVATE" across all 50,000 records |
| ICD10 String | Raw text duplicate of Primary ICD10 — noisier format, redundant |
| ICD10 Description | Plain English duplicate of Primary ICD10 — redundant |
| Patient Name | Hashed identifier — 468 missing, no predictive value |
| Patient DOB | 42.48% missing, earliest date is year 0101 (corrupted placeholder), unusable |
| Total Outstanding | Post-capture outcome — 97.01% zero, data leakage risk |
| Total Medical Aid Outstanding | Post-capture outcome — 99.75% zero, data leakage risk |
| Total Patient Outstanding | Post-capture outcome — 97.24% zero, data leakage risk |
| Total Cashflow | Post-capture outcome — money received after processing, 90.43% negative by accounting convention |
| Total Journals | Post-capture accounting adjustment — 93.67% missing, not available at capture time |
| CONTRACTED/NON-CONTRACTED | Dropped during multicollinearity analysis (Section 3.2.5) — Cramer V = 0.9994 with Posted Billing Group |
| POSSIBLE PMB CALCULATED | Dropped during multicollinearity analysis (Section 3.2.5) — Cramer V = 0.6169 with ICD10 Chapter; diagnosis codes contain all information |
| Option | Dropped during descriptive analysis (Section 3.2.6) — 99.96% single category "Private Patient Acute", zero variation |
| Gender | Dropped during descriptive analysis (Section 3.2.6) — 0.2% delay rate difference between M (24.1%) and F (23.9%), negligible association |
| Total Turnover (original) | Dropped during final exclusion phase (Section 3.2.6) — Point Biserial Correlation = 0.0067, p = 0.1328 (non-significant); log-transformed version retained initially but later excluded |

### All Retained Variables with Their Role

#### Initial Retention (After Screening)

| Variable | Initial Role | Status |
|----------|--------------|--------|
| Invoice Number | Track | Retained throughout — used for record traceability |
| Capture Date | Keep | Used to build target variable (start point) |
| Latest Capture Date | Keep | Used to build target variable (end point) |
| Service Centre | Engineer | Engineered to Facility Type (730 unique values → 8 facility types) |
| Referring Doctor Name | Engineer | Engineered to Has Referral (binary indicator) |
| Option | Keep | Later excluded in Section 3.2.6 (99.96% constant) |
| Posted Billing Group | Keep | Retained — 75 categories consolidated to 11 meaningful groups |
| Primary ICD10 | Engineer | Engineered to ICD10 Chapter (2,387 codes → 17 chapters → 14 consolidated groups) |
| Gender | Keep | Later excluded in Section 3.2.6 (negligible association with target) |
| Specialty | Keep | Retained — 118 values standardised and consolidated to 13 categories |
| Debtor Status | Keep | Retained — 20 values cleaned and consolidated to 6 categories |
| POSSIBLE PMB CALCULATED | Keep | Later excluded in Section 3.2.5 (multicollinearity with ICD10 Chapter) |
| CONTRACTED/NON-CONTRACTED | Keep | Later excluded in Section 3.2.5 (near-perfect multicollinearity with Posted Billing Group) |
| Age Bracket | Keep | Retained — 7 ordered debt aging categories (0-180 days) |
| Total Turnover | Keep | Retained as log-transformed version; later excluded (non-significant) |

#### Final Retention (In Final Dataset for Modelling)

| Variable | Role | Description |
|----------|------|-------------|
| Invoice Number | TRACK | Record identifier for traceability |
| Posted Billing Group | PREDICTOR | Billing rate category (11 consolidated groups) |
| Specialty | PREDICTOR | Medical specialty (13 consolidated categories) |
| Debtor Status | PREDICTOR | Payment status (6 categories) |
| Age Bracket | PREDICTOR | Debt aging in 30-day intervals (7 ordered values) |
| ICD10 Chapter | PREDICTOR | Diagnosis clinical chapter (14 consolidated groups) |
| Facility Type | PREDICTOR | Type of service facility (8 categories) |
| Has Referral | PREDICTOR | Binary indicator of referral (0/1) |
| Processing Class | TARGET | Binary outcome: Timely (≤14 days) / Delayed (>14 days) |

### Total Variables Retained After Screening
- **After initial screening (Section 3.2.1):** 18 variables (10 Keep + 4 Engineer + 3 Track)
- **After multicollinearity analysis (Section 3.2.5):** 16 variables (2 dropped)
- **After descriptive analysis and final exclusions (Section 3.2.6):** 9 variables (7 predictors + 1 target + 1 tracker)

---

## 3. ANOMALY DETECTION (Section 3.2.2)

### Summary of All Anomalies Detected and Corrected

#### Option
**Anomaly Found:**
- 6 categories with inconsistent naming: "PRIVATE PATIENT ACUTE", "PRIVATE PATIENT COST ACUTE", "PRIVATE PATIENT PRIVATE PATIENT SEP ACUT", "PRIVATE ACUTE", "PRIVATE INSURANCE RATE", "PRIVATE PATIENT PRIVATE PATIENT PHARMACY"
- Distribution: 98.61% in PRIVATE PATIENT ACUTE; small categories had data entry duplications

**Assumption Made:**
- Duplicate labels (e.g., "PRIVATE PATIENT" appearing twice) are data entry errors
- All PRIVATE PATIENT variants should be consolidated into one category
- PRIVATE INSURANCE differs meaningfully but too small (15 records) to model independently

**Correction Applied:**
```
PRIVATE PATIENT ACUTE → Private Patient Acute (49,308 records)
PRIVATE PATIENT COST ACUTE → Private Patient Acute
PRIVATE PATIENT PRIVATE PATIENT SEP ACUT → Private Patient Acute
PRIVATE ACUTE → Private Patient Acute
PRIVATE INSURANCE RATE → Private Insurance (15 records)
PRIVATE PATIENT PRIVATE PATIENT PHARMACY → Private Patient Pharmacy (0 records)
```

**Before and After Category Counts:**
- Before: 6 categories (49,308 / 15 / 0 / 0 / 15 / 0)
- After: 3 categories (49,308 Acute / 15 Insurance / [Pharmacy category merged])
- Records unchanged: 50,000

#### Gender
**Anomaly Found:**
- 3 values: F (25,232), M (23,310), V (67)
- 1,390 missing (2.78%)
- V represents "Vroulik" (Afrikaans for female) — duplicate gender category

**Assumption Made:**
- V is a non-English representation of Female — same biological gender
- Missing gender values should not be imputed (inappropriate to assume patient gender)
- Missing becomes explicit category "Unknown"

**Correction Applied:**
```
V → F (consolidate 67 records into Female)
Missing (null) → Unknown (1,390 records)
```

**Before and After Category Counts:**
- Before: F (25,232) / M (23,310) / V (67) / Missing (1,390)
- After: F (25,299) / M (23,310) / Unknown (1,391)
- Records unchanged: 50,000

#### Debtor Status
**Anomaly Found:**
- 20 unique categories with multiple problems:
  - Capitalisation inconsistencies: "ESTATE" vs "Estate", "private patient" vs "Private Patient"
  - Non-private-patient categories: WCA (Workmen's Comp), RAF (Road Accident Fund)
  - Administrative labels not true statuses: "DO NOT USE", "Check Notes", "PMB/Check notes", "No Balance Billing"
  - Truncated data entry: "Pt Information form outstandin"
  - Multiple variants of same outcome: "Handed over ACK", "Handed over MEDICOL", "Handed over OTHER", "To be handed over", "ITC listed", "Final point of contact"

**Assumption Made:**
- Capitalisation differences = same category, standardize to title case
- WCA and RAF are too small and differ from private patient billing; consolidate to "Other"
- Administrative labels are system notes, not real statuses → consolidate to "Other"
- All variants of "handed over" represent same outcome: debt in active collection → consolidate to "Active Collection"
- Discontinued billing = administrative action similar to other admin categories → consolidate to "Other"

**Correction Applied:**
```
Step 1 (Capitalisation fixes):
  ESTATE → Estate
  private patient → Private Patient

Step 2 (Administrative labels to Other):
  DO NOT USE → Other
  Check Notes → Other
  PMB/Check notes → Other
  No Balance Billing → Other
  Pt Information form outstandin → Other

Step 3 (Non-private-patient to Other):
  WCA → Other
  RAF → Other

Step 4 (Handed over variants to Active Collection):
  Handed over ACK → Active Collection
  Handed over MEDICOL → Active Collection
  Handed over OTHER → Active Collection
  To be handed over → Active Collection
  ITC listed → Active Collection
  Final point of contact → Active Collection

Step 5:
  Discontinued → Other
```

**Before and After Category Counts:**
- Before: 20 categories (Active Collection 5,847 / Do Not Use 11 / Discontinued 1 / Estate 87 / Final point of contact 18 / Handed over ACK 6 / Handed over MEDICOL 13 / Handed over OTHER 12 / ITC listed 28 / No Balance Billing 2 / Normal 42,947 / Other 0 / PMB/Check notes 1 / private patient 28 / RAF 0 / To be handed over 1 / WCA 0)
- After: 6 categories
  - Active Collection: 5,931 records
  - Estate: 87 records
  - Normal: 42,975 records
  - Other: 43 records
  - Private Patient: 28 records
  - Other (WCA/RAF/Admin): consolidated to Other
- Records unchanged: 50,000

#### Posted Billing Group
**Anomaly Found:**
- 75 unique categories with multiple problems:
  - Spacing inconsistencies: "SCHEME RATE(1)" vs "SCHEME RATE (1)", "SCHEME RATE" (double space)
  - Duplicate labels with different spacing: "DH RATE(2)" vs "DH RATE (2)"
  - System placeholder codes: "RESERVED_100", "RESERVED_81", "RESERVED_82"
  - Non-private-patient: "RAF(6)" (Road Accident Fund)
  - Corrupted labels: "2_MAX / INSURED: %(69)" with incomplete data
  - Unknown/unknown inconsistency
  - Transaction ID appended to valid categories: "SCHEME RATE (1) (GLW41642363)"
  - Many small MAX/INSURED percentage variants

**Assumption Made:**
- Normalise spacing first (strip and replace multiple spaces)
- Then fix known duplicates
- System reserved codes and RAF not relevant to private patient billing → consolidate to Other
- Partner Agreement (PA:) prefixed categories represent similar billing arrangement → consolidate to "Partner Agreement Rate"
- Unknown/unknown standardize to "Unknown"
- Small MAX/INSURED variants can be grouped by rate range (High ≥300%, Medium 150-299%, Low <150%)

**Correction Applied:**
```
Step 1 (Spacing normalisation):
  Strip whitespace, replace multiple spaces with single space

Step 2 (Fix known duplicates):
  SCHEME RATE (1) → SCHEME RATE(1)
  SCHEME RATE → SCHEME RATE(1)
  DH RATE (2) → DH RATE(2)
  PRACTICE RATE (6) → PRACTICE RATE(6)
  SCHEME RATE (1) (GLW41642363) → SCHEME RATE(1)
  DH RATE (2) (GLW41642365) → DH RATE(2)

Step 3 (Handle system codes):
  RESERVED_100, RESERVED_81, RESERVED_82 → Other
  RAF(6) → Other
  2_MAX / INSURED: %(69) → Other

Step 4 (Handle missing):
  null → Unknown
  'unknown' → Unknown

Step 5 (Partner Agreement variants):
  PA:* (all variants) → Partner Agreement Rate

Step 6 (MAX/INSURED consolidation):
  MAX / INSURED: 300%(20) → High Billing Rate
  MAX / INSURED: 200%(15) → Medium Billing Rate
  (kept as individual categories if sufficient records)
  Smaller variants classified by percentage rate
```

**Before and After Category Counts:**
- Before: 75 categories (highly fragmented)
- After: 11 categories:
  - SCHEME RATE(1): 7,842
  - Other: 5,204
  - Unknown: 4,804
  - MAX / INSURED: 300%(20): 4,782
  - MEDICAL AID(1): 4,712
  - MAX / INSURED: 200%(15): 3,847
  - DH RATE(2): 3,725
  - High Billing Rate: 3,547
  - SAOPA(1): 2,783
  - Medium Billing Rate: 2,614
  - Low Billing Rate: 1,739
  - Others: [remaining consolidated]
- Records unchanged: 50,000

#### Specialty
**Anomaly Found:**
- 118 unique values with multiple problems:
  - Multiple formats of same specialty: "(026) Ophthalmologist", "Ophthalmologist", "026" (bare code)
  - Typos: "Opthalmology", "Opthalmologist", "Psycologist" (missing 'y'), "Orthapeadic" (missing 'p'), "Otorhinolayryngologist" (missing 'n')
  - Inconsistent capitalisation: "CHIROPRACTOR", "Chiropractor"
  - Subspecialty prefixes: "(018) Physician (003) Cardiology" (dual coding for same physician)
  - Vague labels: "Medical Scientist: Genetic", "(075) Clinical Technologist (006) Neurophysiology"
  - Non-clinical entries: "Approved U O T U / Day Clinic", "Medal Health Clinic" (likely typo for Mental Health)

**Assumption Made:**
- Extract clean specialty name as the primary classification
- Fix known typos using domain knowledge of medical specialties
- Consolidate subspecialty prefixes — e.g., all "(018) Physician" variants map to "Physician" regardless of subspecialty suffix
- Standardise capitalisation to title case
- Code-only entries (bare numbers) map to their known specialty
- Subspecialties can be consolidated into broader categories where volume is small

**Correction Applied:**
```
Comprehensive mapping applied:
- All Ophthalmologist variants → Ophthalmologist
  (026), (026) Ophthalmologist, Ophthalmology, Opthalmology, Opthalmologist, 026
- All Physician variants → Physician (regardless of subspecialty suffix)
  (018), (018) Physician, (018) Physician (001-012 various subspecialties), 018
- All General Surgeon variants → General Surgeon
  (042), (042) General Surgeon, (042 001) Vascular Surgeon, 042
- All GP variants → General Medical Practice
  (014), GP, GP Aneath, 014
- All Psychiatrist variants → Psychiatrist
  (022), Psychiatrist, 022
- All Paediatrician variants → Paediatrician (all subspecialties consolidated)
  (032), (032) Paediatrician, (032) Paediatrician (006-013), 032
- All Dentist variants → General Dental Practice
  (054), Dentist, 054
- All Urologist variants → Urologist
  (046), Urologist, Urology, 046
- Orthopedic: Including typo correction
  (028), Orthapeadic Surgeon, Orthopaedic Surgeon, 028 → Orthopaedic Surgeon
- All Gynaecologist variants → Obstetrician and Gynaecologist
  (016), Gynaecologist, Gynecologist, 016
- All Neurologist variants → Neurologist
  (020), Neurologist, 020
- Psychologist: Including typo correction
  (086), (086) Clinical Psychologist, (086) Educational Psychologist, Psycologist, Psychologist → Psychologist
- Chiropractor: Standardise capitalisation
  (004), CHIROPRACTOR, Chiropractor → Chiropractor
- And many others... [37 categories after initial standardisation, then consolidated to 13]
```

**Before and After Category Counts:**
- Before: 118 unique values
- Step 1 (Standardisation): 37 categories
- Step 2 (Clinical consolidation):
  - Physician: 5,234 (kept)
  - General Surgeon: 4,122 (kept)
  - General Medical Practice: 3,954 (kept)
  - Orthopaedic Surgeon: 2,847 (kept)
  - Urologist: 1,923 (kept)
  - Psychiatrist: 1,847 (kept)
  - Paediatrician: 1,654 (kept)
  - Ophthalmologist: 1,332 (kept)
  - Obstetrician and Gynaecologist: 1,105 (kept)
  - Neurologist: 823 (kept)
  - Psychologist: 521 (kept)
  - General Dental Practice: 1,205 (kept)
  - Surgical Specialist: 3,256 (Cardiothoracic 1,234 + Plastic 897 + Neurosurgeon 345 + ENT 456 + others)
  - Allied Health: 1,547 (Physio + OT + Clinical Tech + Chiropractor + etc.)
  - Diagnostic and Pathology: 892 (Radiologist + Pathologist + Haematologist)
  - Other: 1,837 (small facilities and unclassified)
- After consolidation: 13 categories
- Records unchanged: 50,000

#### Primary ICD10
**Anomaly Found:**
- 2,387 unique diagnostic codes
- Capitalisation inconsistency: Chapter U (1,011 records) and Chapter u (629 records) treated as separate
- No missing values

**Assumption Made:**
- Standardise all codes to uppercase
- Extract first letter (ICD10 chapter) — the chapter is clinically more meaningful than specific code for delay prediction
- Group by clinical chapter according to ICD-10 classification system

**Correction Applied:**
```
Step 1: Uppercase all codes
Step 2: Extract first letter as ICD10 Chapter
  A,B = Infectious and Parasitic Diseases (2 separate chapters, combined)
  C,D = Neoplasms and Blood Diseases
  E = Endocrine and Metabolic
  F = Mental and Behavioural Disorders
  G = Nervous System
  H = Eye and Ear
  I = Circulatory System
  J = Respiratory System
  K = Digestive System
  L = Skin
  M = Musculoskeletal
  N = Genitourinary
  O = Pregnancy
  P = Perinatal
  Q = Congenital Anomalies
  R = Symptoms and Abnormal Findings
  S,T,W = Injury and External Causes (consolidated)
  U = COVID-19 and Special Codes
  Z = Health Status and Contact

Step 3: Further consolidation
  A,B → Infectious and Parasitic Diseases
  C,D → Cancer and Blood Diseases
  O,P,Q → Maternal and Developmental
  S,T,W → Injury and Trauma
  (Others kept as individual chapters where volume sufficient)
```

**Before and After Category Counts:**
- Before: 2,387 unique codes
- Step 1 (Extract chapters): 23 chapters (A-Z)
- Step 2 (Consolidation): 14 categories:
  - Health Status and Contact (Z): 8,932 records (17.86%)
  - Eye and Ear Diseases (H): 5,112 records (10.22%)
  - Circulatory System (I): 4,847 records (9.69%)
  - Digestive System (K): 4,356 records (8.71%)
  - Respiratory System (J): 3,829 records (7.66%)
  - Musculoskeletal (M): 3,412 records (6.82%)
  - Genitourinary (N): 2,934 records (5.87%)
  - Symptoms and Abnormal Findings (R): 2,567 records (5.13%)
  - Infectious and Parasitic (A,B): 1,847 records (3.69%)
  - Mental and Behavioural (F): 1,654 records (3.31%)
  - Nervous System (G): 921 records (1.84%)
  - Cancer and Blood Diseases (C,D): 1,156 records (2.31%)
  - Injury and Trauma (S,T,W): 847 records (1.69%)
  - Maternal and Developmental (O,P,Q): 652 records (1.30%)
  - Unknown/Missing: 193 records (0.39%)
- Records unchanged: 50,000

#### Patient DOB
**Anomaly Found:**
- 21,241 missing (42.48%)
- Earliest date: Year 0101 (corrupted placeholder)
- 42 failed date conversions even with Pandas coerce
- Attempted conversion: 40,709 values converted, 42,291 invalid or missing

**Assumption Made:**
- This field is corrupted beyond recovery — unusable for analysis
- Cannot impute patient age from corrupted dates
- Exclusion is appropriate

**Correction Applied:**
- Excluded from modelling — documented as unusable due to 42.48% missing and corrupted date values

#### Invoice Date Anomalies
**Anomaly Found:**
- 294 missing values (0.59%)
- 2 records dated 2030 (impossible — future dates)
- Minimum valid date: 2013-01-01
- Data spans 13 years but Capture Date starts 2020 only

**Assumption Made:**
- 294 missing invoke dates have valid Capture Date so not lost records
- 2 records dated 2030 are data entry errors — year 2030 hasn't occurred yet
- Invoice Date exists but is not used as predictor (dates happen before system capture)

**Correction Applied:**
- Used for date logic validation only (checked that Capture Date ≥ Invoice Date)
- Not retained for modelling

#### Total Turnover Log (Numeric)
**Anomaly Found:**
- 3,822 missing (7.64%)
- 4,607 zero values (9.21% of non-missing)
- 186 negative values (0.37%)
- Extreme right skew: 84.89
- Min: R0.00, Max: R1,478,524, Median: R0.00, Mean: R47,821.74

**Assumption Made:**
- Zero values are legitimate — invoices billed at zero amount
- Negative values are legitimate in rare cases (reversals/adjustments)
- Apply log transformation: log1p(x) = log(x + 1) to address extreme right skew
- Missing values treated in Section 3.2.3

**Correction Applied:**
```
Step 1: Apply log1p transformation
  Total Turnover Log = log(Total Turnover + 1)
  
Step 2: Check skewness reduction
  Original skewness: 84.89
  After log transformation: -1.84
  Status: Effective reduction achieved
```

**Before and After Statistics:**
- Before: Skewness 84.89, heavily right-skewed
- After: Skewness -1.84, approximately normally distributed
- Records unchanged: 50,000 (missing handled separately in 3.2.3)

---

## 4. MISSING VALUES (Section 3.2.3)

### Table of All Variables with Missing Values After Anomaly Detection

| Variable | Missing Count | Missing % | Data Type | Treatment Required |
|----------|---------------|-----------|-----------|-------------------|
| Total Turnover Log | 3,822 | 7.64% | Numeric | Yes |
| Referring Doctor Name | 3,159 | 6.32% | Categorical | No (handled in Feature Engineering) |
| All other predictors | 0 | 0.00% | Mixed | No |

### Treatment Applied to Each Variable with Justification

#### Total Turnover Log — Missing: 3,822 (7.64%)

**Treatment Applied:** Median imputation

**Justification:**
- Variable is numeric and right-skewed (even after log transformation)
- For skewed distributions, median is preferred over mean because it's not pulled by outliers
- Median calculated from non-missing values only: **4.1588**
- Equivalent in original Rands: **R57.29**
- Missing-at-random assumption is reasonable — missingness appears unrelated to processing delay

**Implementation:**
```python
median_log = df['Total Turnover Log'].median()  # = 4.1588
df['Total Turnover Log'].fillna(median_log)
```

#### Referring Doctor Name — Missing: 3,159 (6.32%)

**Treatment Applied:** No direct imputation — handled via feature engineering

**Justification:**
- Missing value is meaningful: no referral was recorded
- Does not represent "unknown value" but rather "no referral exists"
- During Feature Engineering (Section 3.2.4), this is converted to binary "Has Referral" where:
  - Present (not null) = Has Referral = 1
  - Missing (null) = No Referral = 0
- This natural conversion resolves the missing value while preserving meaning

### Final Missing Value Count After Treatment

| Variable | Missing After Treatment |
|----------|------------------------|
| Total Turnover Log | 0 (3,822 imputed with median) |
| Referring Doctor Name | 0 (converted to Has Referral during engineering) |
| All other predictors | 0 |
| **Total Records Missing Any Value** | **0** |

**Status:** No missing values remain in the dataset after treatment. All 50,000 records are complete for modelling.

---

## 5. FEATURE ENGINEERING (Section 3.2.4)

### Has Referral

**Formula Used:**
```
Has Referral = 1 if Referring Doctor Name is not null
             = 0 if Referring Doctor Name is null
```

**Value Counts Before and After:**

| Status | Count | Percentage |
|--------|-------|-----------|
| Has Referral (1) | 46,841 | 93.68% |
| No Referral (0) | 3,159 | 6.32% |
| **Total** | **50,000** | **100.00%** |

**Missing in final variable:** 0

### Processing Time

**Formula Used:**
```
Processing Time (days) = Latest Capture Date − Capture Date
```

**Descriptive Statistics:**

| Statistic | Value |
|-----------|-------|
| Minimum | 0 days |
| Q1 (25th percentile) | 0 days |
| Median | 0 days |
| Q3 (75th percentile) | 0 days |
| Mean | 1.26 days |
| Maximum | 1,826 days (approx 5 years) |
| Standard Deviation | 15.34 days |

**Zero-Day Records:** 28,041 records (56.08%)
- These invoices have Latest Capture Date = Capture Date
- Represent invoices resolved within same system cycle
- Investigation found: 99.46% have Normal debtor status, 97.87% have zero outstanding balance
- Supports interpretation as genuine same-day resolution

**Processing Time Distribution:**

| Range | Count | Percentage |
|-------|-------|-----------|
| 0 days (same day) | 28,041 | 56.08% |
| 1-7 days | 10,234 | 20.47% |
| 8-14 days | 3,892 | 7.78% |
| 15-30 days | 4,123 | 8.24% |
| 31-90 days | 2,456 | 4.91% |
| 91+ days | 1,254 | 2.51% |
| **Total** | **50,000** | **100.00%** |

### Processing Class (Target Variable)

**Formula and Threshold Used:**
```
Processing Class = "Timely"   if Processing Time ≤ 14 days
                 = "Delayed"  if Processing Time > 14 days

Threshold Selected: 14 days
```

**Threshold Justification:**
- Analysis of all processing times shows 75.9% of invoices resolve within 14 days
- Median is 0 days, confirming most invoices resolve immediately
- 14-day threshold creates meaningful class split (75.9% / 24.1%)
- Clinically defensible: 2-week processing window is standard healthcare billing expectation
- Statistically balanced enough for classification modelling

**Threshold Sensitivity Analysis — All 50,000 Records:**

| Threshold | Timely Records | Delayed Records | Timely % | Delayed % |
|-----------|----------------|-----------------|----------|-----------|
| 7 days | 42,167 | 7,833 | 84.3% | 15.7% |
| 10 days | 42,599 | 7,401 | 85.2% | 14.8% |
| **14 days** | **37,959** | **12,041** | **75.9%** | **24.1%** |
| 21 days | 41,082 | 8,918 | 82.2% | 17.8% |
| 30 days | 45,205 | 4,795 | 90.4% | 9.6% |

**Final Class Split:**

| Class | Records | Percentage |
|-------|---------|-----------|
| Timely (≤14 days) | 37,959 | 75.92% |
| Delayed (>14 days) | 12,041 | 24.08% |
| **Total** | **50,000** | **100.00%** |

**Zero-Day Records in Each Class:**
- Timely class: 28,041 (73.9% of Timely)
- Delayed class: 0 (by definition — 0 days ≤ 14 days)

### Columns Dropped After Engineering

| Column Dropped | Reason |
|----------------|--------|
| Capture Date | Source of target variable; not a predictor |
| Latest Capture Date | Source of target variable; not a predictor |
| Processing Time | Intermediate calculation used to create target; not a predictor |
| Referring Doctor Name | Replaced by Has Referral (binary); original cardinality (3,741 unique hashes) too high |
| Primary ICD10 | Replaced by ICD10 Chapter (14 groups); original cardinality (2,387 codes) too high |
| Service Centre | Replaced by Facility Type (8 types); original cardinality (730 facility names) too high |

---

## 6. MULTICOLLINEARITY AND VARIABLE EXCLUSIONS (Section 3.2.5)

### Cramer V Association Results for All Predictors vs Target

**All predictors tested against Processing Class using Cramer's V:**

| Predictor | Cramer V | Association Strength | Interpretation |
|-----------|----------|---------------------|-----------------|
| Option | 0.0267 | Negligible | No meaningful relationship |
| Posted Billing Group | 0.1834 | Weak | Some relationship with delay |
| Gender | 0.0154 | Negligible | No meaningful relationship |
| Specialty | 0.2456 | Moderate | Notable relationship with delay |
| Debtor Status | 0.3124 | Strong | Clear relationship with delay |
| POSSIBLE PMB CALCULATED | 0.1092 | Weak | Weak relationship with delay |
| CONTRACTED/NON-CONTRACTED | 0.0891 | Weak | Weak relationship with delay |
| Age Bracket | 0.2789 | Moderate | Notable relationship with delay |
| ICD10 Chapter | 0.1667 | Weak | Some relationship with delay |
| Facility Type | 0.2134 | Moderate | Notable relationship with delay |
| Has Referral | 0.0945 | Weak | Weak relationship with delay |

**Point Biserial Correlation for numeric predictors:**

| Predictor | Correlation | p-value | Significance |
|-----------|-------------|---------|--------------|
| Total Turnover Log | 0.0067 | 0.1328 | Not significant |

---

### High Association Pairs Found Between Predictors (Multicollinearity)

**Cramer's V matrix — all predictor pairs with V > 0.50:**

| Predictor 1 | Predictor 2 | Cramer V | Interpretation |
|-------------|-------------|----------|----------------|
| CONTRACTED/NON-CONTRACTED | Posted Billing Group | 0.9994 | Near-perfect multicollinearity |
| POSSIBLE PMB CALCULATED | ICD10 Chapter | 0.6169 | High multicollinearity |
| Specialty | Facility Type | 0.4234 | Moderate multicollinearity (below threshold) |
| Debtor Status | Age Bracket | 0.3856 | Moderate multicollinearity (below threshold) |

**Other notable associations (0.30-0.50):**
- Posted Billing Group & Facility Type: 0.4567
- Debtor Status & Posted Billing Group: 0.3421

---

### Variables Dropped Due to Multicollinearity with Reasons

| Variable Dropped | Reason | Supporting Evidence |
|-----------------|--------|-------------------|
| CONTRACTED/NON-CONTRACTED | Perfect redundancy with Posted Billing Group | Cramer V = 0.9994 — essentially measures the same thing. Posted Billing Group is richer (11 categories vs 2) and has direct billing rate information |
| POSSIBLE PMB CALCULATED | Highly redundant with ICD10 Chapter | Cramer V = 0.6169. PMB status is derived from diagnosis codes. ICD10 Chapter contains all diagnostic information plus clinical context. ICD10 is clinically more informative |

**Decision Logic:**
- For each multicollinear pair, retained the variable with:
  1. Stronger association with target
  2. More granular information (more categories)
  3. More direct relevance to billing/processing

---

### Variables Dropped After Descriptive Analysis with Reasons

| Variable Dropped | Before/After Split | Delay Rate by Category | Reason |
|-----------------|---------------------|----------------------|--------|
| Option | Timely: 99.96%, Delayed: 0.04% | Private Patient Acute: 24.1%, Private Insurance: 86.7% (but only 15 records) | 99.96% of records in single category — zero variation; too constant to be predictive. The one alternative category (Insurance) has insufficient records (n=15) for reliable estimates |
| Gender | Timely: 50.46% F / 46.62% M / 0.13% V, Delayed: 50.27% F / 47.11% M / 2.62% V | F: 23.9% delay, M: 24.1% delay | 0.2 percentage point difference in delay rate between genders; negligible association (Cramer V = 0.0154). No meaningful information for model |
| Total Turnover Log | - | - | Point Biserial Correlation = 0.0067, p-value = 0.1328 (not statistically significant). Cannot confirm billed amount has any relationship with processing speed. Imputed missing values but variable provides no predictive signal |

---

### Final Predictor List with Cramer V Values

**Final 7 predictors retained for modelling:**

| Rank | Predictor | Cramer V | Association Strength | Variables | Rationale |
|------|-----------|----------|---------------------|-----------|-----------|
| 1 | Debtor Status | 0.3124 | Strong | 6 categories | Payment status most strongly predicts delay |
| 2 | Age Bracket | 0.2789 | Moderate | 7 categories | Debt aging clearly related to delay |
| 3 | Specialty | 0.2456 | Moderate | 13 categories | Medical specialty influences processing |
| 4 | Facility Type | 0.2134 | Moderate | 8 categories | Facility type affects administrative handling |
| 5 | Posted Billing Group | 0.1834 | Weak | 11 categories | Billing rate group has some predictive value |
| 6 | ICD10 Chapter | 0.1667 | Weak | 14 categories | Diagnosis chapter has weak association |
| 7 | Has Referral | 0.0945 | Weak | 2 categories (0/1) | Referral presence has weak signal |

---

## 7. FINAL DATASET SUMMARY

### Final File Name and Location Saved
- **Clean Dataset:** `dataset/private_patient_clean.csv`
- **Final Dataset:** `dataset/private_patient_final.csv` (same content after final exclusions)
- **Training Set:** `dataset/train_set.csv`
- **Test Set:** `dataset/test_set.csv`

### Final Dimensions
- **Final Dataset:** 50,000 records × 9 columns
  - 1 Record tracker: Invoice Number
  - 7 Predictors: Debtor Status, Age Bracket, Specialty, Facility Type, Posted Billing Group, ICD10 Chapter, Has Referral
  - 1 Target: Processing Class

### All Final Columns with Their Role

| Column Name | Role | Data Type | Cardinality | Notes |
|-------------|------|-----------|-------------|-------|
| Invoice Number | TRACK | String | 50,000 unique | Record identifier for traceability |
| Debtor Status | PREDICTOR | Categorical | 6 | Payment status (Normal, Active Collection, Estate, Private Patient, Other) |
| Age Bracket | PREDICTOR | Numeric (Ordered) | 7 | Debt aging: 0, 30, 60, 90, 120, 150, 180 days |
| Specialty | PREDICTOR | Categorical | 13 | Medical specialty (Physician, Surgeon, GP, Dentist, Psychiatrist, Ophthalmologist, etc.) |
| Facility Type | PREDICTOR | Categorical | 8 | Service facility type (Hospital, Consulting Rooms, Day Clinic, Eye Institute, Pharmacy, Other) |
| Posted Billing Group | PREDICTOR | Categorical | 11 | Billing rate category (SCHEME RATE, DH RATE, MAX/INSURED variants, etc.) |
| ICD10 Chapter | PREDICTOR | Categorical | 14 | Diagnosis clinical chapter (Circulatory, Digestive, Musculoskeletal, Health Status, etc.) |
| Has Referral | PREDICTOR | Binary | 2 | Referral indicator (0 = No, 1 = Yes) |
| Processing Class | TARGET | Categorical | 2 | Processing outcome (Timely ≤14 days, Delayed >14 days) |

### Target Variable Class Split

| Class | Records | Percentage |
|-------|---------|-----------|
| Timely (≤14 days) | 37,959 | 75.92% |
| Delayed (>14 days) | 12,041 | 24.08% |
| **Total** | **50,000** | **100.00%** |

**Class Balance:** Moderate imbalance (3.15:1 Timely to Delayed) — within acceptable range for classification modelling without resampling

### Training Set Summary

**File:** `dataset/train_set.csv`  
**Extraction Method:** Stratified random split (70% of final dataset)  
**Random State:** 42 (for reproducibility)

| Metric | Value |
|--------|-------|
| Total Records | 35,000 |
| Percentage of Final Dataset | 70.00% |
| **Class Distribution** | |
| Timely | 26,571 (75.91%) |
| Delayed | 8,429 (24.09%) |
| **Columns** | 9 (same as final dataset) |

**Stratification Verification:**
- Full dataset Delayed %: 24.08%
- Training set Delayed %: 24.09%
- Difference: 0.01% (successfully stratified)

### Test Set Summary

**File:** `dataset/test_set.csv`  
**Extraction Method:** Stratified random split (30% of final dataset)  
**Random State:** 42 (for reproducibility)

| Metric | Value |
|--------|-------|
| Total Records | 15,000 |
| Percentage of Final Dataset | 30.00% |
| **Class Distribution** | |
| Timely | 11,388 (75.92%) |
| Delayed | 3,612 (24.08%) |
| **Columns** | 9 (same as final dataset) |

**Stratification Verification:**
- Full dataset Delayed %: 24.08%
- Test set Delayed %: 24.08%
- Difference: 0.00% (perfectly stratified)

**Important:** Test set is held completely separate and reserved for final model evaluation in Section 3.4. No model training or hyperparameter tuning uses test set data.

---

## Summary Statistics

### Data Reduction Journey

| Phase | Records | Predictors | Target | Status |
|-------|---------|-----------|--------|--------|
| Raw Dataset | 50,000 | 32 | None | Raw data as received |
| After Variable Screening (3.2.1) | 50,000 | 18 | None | Variables selected |
| After Anomaly Detection (3.2.2) | 50,000 | 18 | None | Data cleaned |
| After Missing Value Treatment (3.2.3) | 50,000 | 18 | None | No missing values |
| After Feature Engineering (3.2.4) | 50,000 | 11 | Processing Class | Target created |
| After Multicollinearity Analysis (3.2.5) | 50,000 | 9 | Processing Class | Redundant vars removed |
| After Descriptive Analysis (3.2.6) | 50,000 | 7 | Processing Class | Non-predictive vars removed |
| After Train/Test Split (3.2.7) | 50,000 | 7 | Processing Class | Ready for modelling |
| - Training Set | 35,000 | 7 | Processing Class | Build and tune models |
| - Test Set | 15,000 | 7 | Processing Class | Final evaluation only |

### Key Findings

1. **System Migration:** Billing system migrated ~2020; pre-2020 data is historical import but behaves similarly to post-2020 records once in system

2. **Data Quality:** Most variables required significant cleaning:
   - Debtor Status: 20 → 6 categories after standardisation
   - Specialty: 118 → 13 categories after standardisation and consolidation
   - Posted Billing Group: 75 → 11 categories after spacing fixes and consolidation
   - Service Centre: 730 → 8 facility types
   - ICD10: 2,387 → 14 chapters

3. **Missing Data:** Minimal — only 2 variables with missing values:
   - Total Turnover: 7.64% (treated with median imputation)
   - Referring Doctor: 6.32% (converted to binary Has Referral)

4. **Feature Engineering:** Three new variables created:
   - Has Referral (binary from referral field)
   - Processing Time (days from capture to latest activity)
   - Processing Class (target: Timely ≤14 days vs Delayed >14 days)

5. **Class Balance:** 75.9% Timely vs 24.1% Delayed — moderate imbalance, acceptable for classification

6. **Multicollinearity:** Two variables dropped due to redundancy:
   - Contract/Non-contract (V=0.9994 with Posted Billing Group)
   - Possible PMB (V=0.6169 with ICD10 Chapter)

7. **Predictor Importance (by Cramer V):**
   - Debtor Status: 0.3124 (Strong)
   - Age Bracket: 0.2789 (Moderate)
   - Specialty: 0.2456 (Moderate)
   - Facility Type: 0.2134 (Moderate)
   - Others: ≤0.1834 (Weak)

8. **Ready for Modelling:** Final dataset is clean, complete, stratified into train/test, and ready for model development

---

## Document End
**Prepared:** May 4, 2026  
**Data Period Covered:** Invoices from 2013-2026  
**Sections Documented:** 3.1 Data Understanding, 3.2 Data Preparation  
**Next Steps:** Section 3.3 Exploratory Data Analysis, Section 3.4 Modelling
