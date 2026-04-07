# EDA_Housing_Prices_in_Nepal

# Documentation: Phase 1 - Price Data Cleaning 

## Overview

This document describes the first phase of Exploratory Data Analysis (EDA) and Feature Engineering for the Nepali House Price Dataset. The primary challenge addressed was cleaning and standardizing the highly inconsistent `PRICE` column, which contained multiple formats, units, and placeholders.

---

## The Problem

The original `PRICE` column had several issues:
![[image.png]]

| Issue Type | Example Values | Count |
|------------|----------------|-------|
| Placeholder text | `"Price on call"`, `"contact"`, `"negotiable"` | 190 |
| Price per unit (rates) | `"Rs. 1.5 Lac/m"`, `"Rs. 50 Lac/aana"`, `"Rs. 65,000 /m"` | 560 |
| Absolute values in Crore | `"Rs. 2.9 Cr"`, `"Rs. 4.75 Cr"` | 2,431 |
| Absolute values in Lac | `"Rs. 1.5 Lac"`, `"Rs. 90 Lac"` | 28 |
| Raw NPR with commas | `"Rs. 12000000"`, `"Rs. 3,88,000,00"` | 208 |

**Total rows:** 3,418

---

## Step-by-Step Cleaning Process

### Step 1: Data Preparation

Created a clean copy and standardized the price column by stripping whitespace:

```python
df['PRICE_CLEAN'] = df['PRICE'].astype(str).str.strip()
```

---

### Step 2: Classification Strategy

Before converting values, all rows were classified into mutually exclusive categories:

```
                    ┌─────────────────┐
                    │   PRICE Column   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌───────────────┐ ┌─────────────┐ ┌─────────────┐
    │ "call/contact"│ │ Contains "/"│ │ No "/"      │
    │ (Placeholder) │ │ (Rate/unit) │ │ (Absolute)  │
    └───────────────┘ └──────┬──────┘ └──────┬──────┘
                             │               │
                             ▼               ▼
                    ┌────────────┐    ┌─────────────┐
                    │ /m or /aana│    │ Has "Cr"    │
                    └────────────┘    │ Has "Lac"   │
                                      │ Raw NPR     │
                                      └─────────────┘


```

![[image1.png]]
![[image2.png]]
![[image3.png]]
![[image4.png]]
![[image5.png]]


---

### Step 3: Handling Non-Price Rows

**Goal:** Identify and separate rows that don't contain actual prices.



```python
non_price_mask = df['PRICE_CLEAN'].str.contains('call|contact|negotiable', na=False, case=False)
```

**Result:** 190 rows marked as non-price (to be excluded from price analysis)

---

### Step 4: Handling Rate Prices (Per Unit)

**What are rate prices?** These indicate price per unit of land area (per meter or per aana).

**Examples:**
- `"Rs. 1.5 Lac/m"` → NPR 150,000 per square meter
- `"Rs. 50 Lac/aana"` → NPR 5,000,000 per aana
- `"Rs. 65,000 /m"` → NPR 65,000 per square meter

**Classification breakdown:**

| Denominator | Count | Example |
|-------------|-------|---------|
| `/m` (per meter) | 534 | `"Rs. 1.4 Lac/m"` |
| `/aana` (per aana) | 26 | `"Rs. 50 Lac/aana"` |
| Other | 1 | `"Rs. 50 /sf"` (per square foot) |



**Conversion Logic:**

```python
# For "/m" with "Lac" → Multiply by 100,000
# For "/m" without "Lac" → Remove commas, convert to float
# For "/aana" → Always multiply by 100,000
```

**New features created:**
- `RATE_VALUE`: Numeric rate amount (in NPR)
- `RATE_UNIT`: Unit of the rate (always 'NPR' after conversion)
- `RATE_DENOMINATOR`: Area unit ('m' or 'aana')

---

### Step 5: Handling Absolute Prices in Crore (Cr)

**Format:** `"Rs. X Cr"` or `"Rs. X.X Cr"`

**Conversion:** Multiply by 10,000,000 (1 Crore = 10,000,000 NPR)

| Original | Extracted Number | NPR Value |
|----------|-----------------|-----------|
| `Rs. 2.9 Cr` | 2.9 | 29,000,000 |
| `Rs. 4.75 Cr` | 4.75 | 47,500,000 |
| `Rs. 1.99 Cr` | 1.99 | 19,900,000 |

**Count:** 2,431 rows


---

### Step 6: Handling Absolute Prices in Lac

**Format:** `"Rs. X Lac"` or `"Rs. X.X Lac"`

**Conversion:** Multiply by 100,000 (1 Lac = 100,000 NPR)

| Original | Extracted Number | NPR Value |
|----------|-----------------|-----------|
| `Rs. 1.5 Lac` | 1.5 | 150,000 |
| `Rs. 90 Lac` | 90 | 9,000,000 |
| `Rs. 65 Lac` | 65 | 6,500,000 |

**Count:** 28 rows

---

### Step 7: Handling Raw NPR Values

**Format:** Direct NPR amounts, sometimes with commas

**Examples:**
- `"Rs. 12000000"` → 12,000,000
- `"Rs. 3,88,000,00"` → 3,880,0000 (Nepali comma system)

**Processing:** Remove commas and convert to float

**Count:** 208 rows

---

## Final Data Verification

### Classification Complete Check

```
Total rows in original: 3,418
Total rows classified: 3,418
Difference: 0 ✓
```
![[image6.png]]
### Data Types After Conversion

| Column             | Type    | Non-Null Count |
| ------------------ | ------- | -------------- |
| `PRICE_NPR`        | float64 | 2,459          |
| `RATE_VALUE`       | float64 | 560            |
| `RATE_UNIT`        | object  | 560            |
| `RATE_DENOMINATOR` | object  | 560            |

---

## New Features Created

| Feature Name | Description | Use Case |
|--------------|-------------|----------|
| `PRICE_NPR` | Standardized absolute price in NPR | Primary target variable for price prediction |
| `RATE_VALUE` | Price per unit area (NPR) | For properties where only rate is available |
| `RATE_UNIT` | Currency unit (always NPR) | Metadata for rate values |
| `RATE_DENOMINATOR` | Area unit ('m' or 'aana') | Needed to calculate absolute price × land area |

---

## Summary

| Metric | Before | After |
|--------|--------|-------|
| Price formats | 5+ inconsistent formats | 1 standardized format (NPR) |
| Data types | Object (string) | Float64 (numeric) |
| Missing/placeholder rows | 190 | Identified and separated |
| New features | 0 | 4 (PRICE_NPR, RATE_VALUE, RATE_UNIT, RATE_DENOMINATOR) |

The price column is now **analysis-ready** for visualization, statistical analysis, but more columns to fix.


