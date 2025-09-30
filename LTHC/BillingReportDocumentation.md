# Billing Report System Documentation

## Overview
This document explains in simple terms how the billing report system generates billing codes (CPT codes) based on the time healthcare staff spend working with patients.

---

## What Are Timer Categories?

Timer categories are different types of care services that healthcare staff can provide to patients. When staff members track their time, they select which category of care they're providing.

### Available Timer Categories:

1. **CCM (Chronic Care Management)**
   - Regular care coordination for patients with multiple chronic conditions
   - Used for non-physician clinical staff time

2. **CCM (QHP)** - Qualified Health Professional
   - Care coordination performed directly by a physician or qualified health professional
   - Higher-level care than standard CCM

3. **PCM (Principal Care Management)**
   - Care coordination for patients with a single high-risk disease
   - Clinical staff time directed by a physician

4. **PCM (QHP)** - Qualified Health Professional
   - Principal care management performed directly by a physician or qualified health professional
   - Direct physician/QHP time for single-condition patients

---

## What Are CPT Codes?

CPT (Current Procedural Terminology) codes are standardized billing codes used to bill insurance companies for healthcare services. Each code represents a specific service and has a specific reimbursement amount.

### Types of CPT Codes:

#### 1. **Base Codes**
- The first billable code applied when minimum time requirements are met
- Example: CCM base code 99490 requires at least 20 minutes of time

#### 2. **Addon Codes**
- Additional codes applied when time exceeds the base code requirement
- Can be used multiple times based on additional time spent
- Example: After 60 minutes with code 99487, you can add code 99489 for every additional 30 minutes

#### 3. **Tiered Codes (Base1/Addon1, Base2/Addon2, etc.)**
- Higher-tier codes for more complex or longer services
- System automatically upgrades to higher tiers when time thresholds are met
- Example: CCM has tier 0 (99490 for 20+ min) and tier 1 (99487 for 60+ min)
- **Optional behavior**: Can delay tier upgrades using `stayLowerCodeUntilAddon` setting (see below)

#### 4. **APCM Codes (Advanced Primary Care Management)**
- Special codes used when time is tracked but doesn't meet minimum billing thresholds
- Applied to CCM and CCM (QHP) categories only
- Selection based on patient's number of chronic conditions:
  - **G0556**: Patients with 0 or 1 chronic conditions
  - **G0557**: Patients with 2 or more chronic conditions
  - **G0558**: Medicare patients with 2 or more chronic conditions (medicare beneficiaries not yet tracked by Platinum Line)

---

## How Time Tracking Works

### Time Entry Process:
1. Healthcare staff start a timer when working with a patient
2. They select the appropriate timer category (CCM, PCM, etc.)
3. The system records:
   - Patient ID
   - Timer category
   - Duration in milliseconds
   - Date and time

### Time Aggregation:
When generating a billing report for a specific month:
1. System collects all time entries for that month
2. Groups time by patient and timer category
3. Converts time from milliseconds to seconds for accuracy
4. Calculates total time per patient per category

---

## How CPT Codes Are Assigned

The system follows a sophisticated algorithm to determine which billing codes to apply:

### Step 1: Check Minimum Time Requirements
Each base CPT code has a minimum time requirement:
- **99490 (CCM)**: 20 minutes minimum
- **99487 (Complex CCM)**: 60 minutes minimum
- **99424 (PCM QHP)**: 30 minutes minimum
- **99437 (CCM QHP addon)**: 30 minutes (as addon only)

**If time is below the minimum**: No base code is assigned, or APCM code is used (for CCM categories only)

### Step 2: Select the Appropriate Tier
For categories with multiple tiers (like CCM):
- System checks from highest tier to lowest
- Uses the highest tier where minimum time is met
- Example: 65 minutes of CCM time → uses tier 1 (99487) instead of tier 0 (99490)

**Optional "Stay Lower Code Until Addon" Mode**:
- Can be enabled to delay switching to higher tiers
- Only switches to tier 1 if there's enough time for tier 1 base + at least 1 addon
- Example: 60 min of CCM → stays on 99490 (tier 0) instead of switching to 99487 (tier 1)
- This prevents switching to a higher base code if you're right at the threshold

### Step 3: Calculate Addon Codes
After applying the base code, calculate addons:
1. Subtract base code time from total time
2. Divide remaining time by addon increment
3. Apply addon code that many times (rounded down)

**Example**: 125 minutes of Complex CCM time
- Base code 99487 = 60 minutes
- Remaining time = 125 - 60 = 65 minutes
- Addon increment = 30 minutes
- Addon count = 65 ÷ 30 = 2.16 → **2 addon codes (99489)**
- **Result**: 1x 99487 + 2x 99489

### Step 4: Handle Under-Threshold Time (APCM Codes)
For CCM and CCM (QHP) categories only:
- If time is logged but below minimum (e.g., 15 minutes of CCM)
- System applies APCM code based on patient's chronic condition count
- This ensures work is still billable even when minimum isn't met

**APCM Selection Logic**:
```
If chronicConditionCount = 0 or 1:
  → Use G0556

If chronicConditionCount ≥ 2:
  → Use G0557

If chronicConditionCount ≥ 2 AND patient is Medicare:
  → Use G0558 (not yet implemented)
```

---

## Complete CPT Code Configuration

### CCM (Chronic Care Management)
| Code | Type | Time Required | Description |
|------|------|---------------|-------------|
| 99490 | Base (Tier 0) | 20 minutes | Standard CCM, minimum 20 min |
| 99439 | Addon (Tier 0) | 20 min increments | Standard CCM, each additional 20 min |
| 99487 | Base (Tier 1) | 60 minutes | Complex CCM, first 60 min |
| 99489 | Addon (Tier 1) | 30 min increments | Complex CCM, each additional 30 min |
| G0556 | APCM | < 20 minutes | Under-threshold, 0-1 chronic conditions |
| G0557 | APCM | < 20 minutes | Under-threshold, 2+ chronic conditions |

**Example Scenarios**:
- **15 minutes**: → G0557 (if patient has 2+ conditions)
- **25 minutes**: → 99490 (base tier 0)
- **45 minutes**: → 99490 + 1x 99439 (tier 0 base + 1 addon)
- **65 minutes**: → 99487 (upgrades to tier 1)
- **125 minutes**: → 99487 + 2x 99489 (tier 1 base + 2 addons)

### CCM (QHP) - Qualified Health Professional
| Code | Type | Time Required | Description |
|------|------|---------------|-------------|
| 99491 | Base | 30 minutes | CCM by physician/QHP, first 30 min |
| 99437 | Addon | 30 min increments | CCM by physician/QHP, each additional 30 min |
| G0556 | APCM | < 30 minutes | Under-threshold, 0-1 chronic conditions |
| G0557 | APCM | < 30 minutes | Under-threshold, 2+ chronic conditions |

**Example Scenarios**:
- **15 minutes**: → G0557 (if patient has 2+ conditions)
- **35 minutes**: → 99491 (base only)
- **65 minutes**: → 99491 + 1x 99437 (base + 1 addon)

### PCM (Principal Care Management)
| Code | Type | Time Required | Description |
|------|------|---------------|-------------|
| 99426 | Base | 30 minutes | PCM by clinical staff, first 30 min |
| 99427 | Addon | 30 min increments | PCM by clinical staff, each additional 30 min |

**Example Scenarios**:
- **25 minutes**: → No code (below 30 min minimum)
- **35 minutes**: → 99426 (base only)
- **65 minutes**: → 99426 + 1x 99427 (base + 1 addon)

### PCM (QHP) - Qualified Health Professional
| Code | Type | Time Required | Description |
|------|------|---------------|-------------|
| 99424 | Base | 30 minutes | PCM QHP base, first 30 min |
| 99425 | Addon | 30 min increments | PCM QHP addon, each additional 30 min |

**Example Scenarios**:
- **25 minutes**: → No code (below minimum)
- **35 minutes**: → 99424 (base only)
- **95 minutes**: → 99424 + 2x 99425 (base + 2 addons)

---

## The "Stay Lower Code Until Addon" Feature

### What Is It?

The `stayLowerCodeUntilAddon` parameter is an optional billing strategy that delays upgrading to higher-tier codes until you have enough time to also bill at least one addon at that higher tier.

### Why Would You Use This?

In some billing scenarios, it may be more financially beneficial to stay on a lower-tier base code with multiple addons rather than immediately switching to a higher-tier base code without addons.

### How It Works

**Default Behavior (stayLowerCodeUntilAddon = false)**:
- System upgrades to the highest tier as soon as the minimum time is met
- Example: 60 minutes of CCM → Uses 99487 (tier 1 base)

**With stayLowerCodeUntilAddon = true**:
- System only upgrades if there's enough time for the higher tier base + at least 1 addon
- Example: 60 minutes of CCM → Stays on 99490 (tier 0 base) instead of 99487

### Detailed Example: CCM with 60-89 Minutes

**Without stayLowerCodeUntilAddon (default)**:
```
60 minutes → 99487 (tier 1 base)
70 minutes → 99487 (tier 1 base)
80 minutes → 99487 (tier 1 base)
89 minutes → 99487 (tier 1 base)
90 minutes → 99487 + 1x 99489 (tier 1 base + 1 addon)
```

**With stayLowerCodeUntilAddon = true**:
```
60 minutes → 99490 (tier 0 base - stays on lower tier)
70 minutes → 99490 (tier 0 base - stays on lower tier)
80 minutes → 99490 (tier 0 base - stays on lower tier)
89 minutes → 99490 (tier 0 base - stays on lower tier)
90 minutes → 99487 + 1x 99489 (NOW switches to tier 1)
```

### Business Logic

For tier 1 codes:
- Tier 1 base (99487) requires: 60 minutes
- Tier 1 addon (99489) requires: 30 minutes
- **Threshold to switch**: 60 + 30 = **90 minutes**

The system checks:
```
If (minutes >= 90 minutes):
  → Switch to tier 1 (99487 + addons)
Else:
  → Stay on tier 0 (99490)
```

### When to Enable This Feature

**Enable `stayLowerCodeUntilAddon = true` when**:
- Your billing practice prefers to maximize lower-tier codes
- Reimbursement rates make this strategy more profitable
- You want to avoid "orphan" higher-tier base codes without addons

**Keep `stayLowerCodeUntilAddon = false` (default) when**:
- You want to use the highest appropriate tier immediately
- Reimbursement rates favor higher-tier base codes
- Medicare/insurance requirements specify tier selection rules

### Implementation Details

- **Applies to**: All tiered codes (tier 1, tier 2, etc.)
- **Does not apply to**: Tier 0 codes, APCM codes, non-tiered codes
- **Code location**: `billing-report.service.ts:521-597`
- **Parameter**: Optional boolean in `GetBillingReportDto`

### Quick Reference: CCM Billing Comparison

| Time (min) | Default Behavior | With stayLowerCodeUntilAddon |
|------------|------------------|------------------------------|
| 19 or less | APCM (G0556/G0557) | APCM (G0556/G0557) |
| 20-39 | 99490 | 99490 |
| 40-59 | 99490 + 1x 99439 | 99490 + 1x 99439 |
| 60-89 | 99487 | 99490 + 2x 99439 |
| 90-119 | 99487 + 1x 99489 | 99487 + 1x 99489 |
| 120-149 | 99487 + 2x 99489 | 99487 + 2x 99489 |
| 150+ | 99487 + 3x 99489 | 99487 + 3x 99489 |

---

## Billing Report Generation Process

### Input Parameters:
- **Month & Year**: Which month to generate the report for
- **Report Type**: "combined" (all categories) or specific category name
- **User ID** (optional): Filter to specific healthcare worker
- **Patient ID** (optional): Filter to specific patient
- **Include Zero Time**: Whether to show patients with no time logged
- **stayLowerCodeUntilAddon** (optional): Whether to delay tier upgrades until addon threshold is met (default: false)

### Report Generation Steps:

1. **Retrieve Time Entries**
   - Get all time entries for the specified month/year
   - Filter by user/patient if specified

2. **Aggregate Time**
   - Group all time entries by patient and category
   - Sum total seconds for each combination
   - Convert to hours:minutes:seconds format

3. **Add Zero-Time Patients** (if requested)
   - Include all patients in the system
   - Show 00:00:00 for categories with no time

4. **Generate CPT Codes**
   - For each patient and category:
     - Apply the CPT code assignment algorithm
     - Determine base code, addon code, and addon count
   - Consider patient's chronic condition count for APCM codes
   - Consider CCM billing consent flag

5. **Sort and Format**
   - Sort by facility name, then last name, then first name
   - Format all times as HH:MM:SS
   - Calculate max addon code count across all patients

6. **Output Report**
   - List of patients with:
     - Patient info (name, facility)
     - Time per category
     - Assigned CPT codes
     - Addon code counts
     - Total time across all categories

---

## Special Business Rules

### 1. Patient Chronic Condition Count
- Stored in the patients table
- Used to determine which APCM code to apply (G0556 vs G0557)
- Must be tracked and updated by healthcare staff

### 2. CCM Billing Consent
- Boolean flag on patient record
- Indicates if patient has consented to CCM billing
- Should be checked before submitting CCM-related claims

### 3. Time Precision
- System uses seconds (not minutes) for all calculations
- Prevents rounding errors in billing
- Only rounds down when calculating addon code counts

### 4. Time Increment Minutes
- Each CPT code has a specified time increment
- Stored in the `billing_criteria` JSONB field
- Used to calculate when codes apply and how many addons

### 5. Multiple Categories Per Code (APCM)
- APCM codes (G0556, G0557, G0558) apply to multiple timer categories
- They're associated with both CCM and CCM (QHP) categories
- This allows under-threshold billing for both care types

---

## Database Schema Overview

### Key Tables:

#### `time_categories`
- **id**: Unique identifier
- **name**: Category name (CCM, PCM, etc.)
- **isDefault**: Whether this is a default category

#### `cpt_codes`
- **id**: Unique identifier
- **code**: CPT code (99490, G0556, etc.)
- **description**: Human-readable description
- **codeType**: Type of code (base, addon, base1, addon1, apcm)
- **timerCategoryIds**: Array of category IDs this code applies to
- **billingCriteria**: JSONB with time/condition requirements
  - `time_increment_minutes`: Minutes required for this code
  - `chronic_condition_requirements`: Min/max condition counts
- **showInNotes**: Whether to display in SOAP notes

#### `time_entries`
- **id**: Unique identifier
- **patientId**: Patient this time entry is for
- **userId**: Staff member who logged the time
- **timeCategoryId**: Which category of care
- **durationMs**: Duration in milliseconds
- **startTime**: When the work started
- **endTime**: When the work ended

#### `patients`
- **id**: Unique identifier
- **firstName**, **lastName**: Patient name
- **siteId**: Which facility they're at
- **chronicConditionCount**: Number of chronic conditions
- **ccmBillingConsent**: Whether they consent to CCM billing

---

## Example Workflow

### Scenario: Generating March 2025 Billing Report

**System receives request**:
- Month: 3
- Year: 2025
- Report Type: "combined"
- Include Zero Time: true
- stayLowerCodeUntilAddon: false (default)

**System processes**:

1. **Retrieves time entries** from March 1-31, 2025

2. **Finds Patient #123 has**:
   - 75 minutes of CCM time
   - 15 minutes of PCM (QHP) time
   - 2 chronic conditions

3. **Applies billing logic** (with stayLowerCodeUntilAddon = false):

   **For CCM category** (75 minutes):
   - Checks tier 1 (99487): Requires 60 min ✓
   - Uses tier 1: 99487 (60 min)
   - Remaining: 75 - 60 = 15 minutes
   - Addon 99489 (30 min increment): 15 ÷ 30 = 0 addons
   - **Result**: 1x 99487, 0x 99489

   **For PCM (QHP)** (15 minutes):
   - Checks base 99424: Requires 30 min ✗
   - Time below minimum
   - **Result**: No code applied

4. **Outputs report**:
   ```
   Patient: John Doe (#123)
   Facility: Main Hospital
   Chronic Conditions: 2
   CCM Billing Consent: Yes

   CCM: 01:15:00
   - Base: 99487 (Complex CCM first 60 min)
   - Addon: None

   PCM (QHP): 00:15:00
   - Base: None (below 30 min minimum)

   Total Time: 01:30:00
   ```

### Alternative Scenario: Same Data with stayLowerCodeUntilAddon = true

**If the request had stayLowerCodeUntilAddon: true**:

3. **Applies billing logic** (with stayLowerCodeUntilAddon = true):

   **For CCM category** (75 minutes):
   - Checks tier 1 (99487): Requires 60 min ✓
   - But tier 1 with addon requires: 60 + 30 = 90 min
   - 75 minutes < 90 minutes, so STAYS on tier 0
   - Uses tier 0: 99490 (20 min)
   - Remaining: 75 - 20 = 55 minutes
   - No tier 0 addon configured
   - **Result**: 1x 99490, 0x addons

4. **Outputs report**:
   ```
   Patient: John Doe (#123)
   Facility: Main Hospital
   Chronic Conditions: 2
   CCM Billing Consent: Yes

   CCM: 01:15:00
   - Base: 99490 (CCM 20 min base - stayed on lower tier)
   - Addon: None

   PCM (QHP): 00:15:00
   - Base: None (below 30 min minimum)

   Total Time: 01:30:00
   ```

**Key Difference**: With `stayLowerCodeUntilAddon = true`, the system used 99490 instead of 99487, staying on the lower tier because there wasn't enough time for an addon at tier 1.

---

## Common Questions

### Q: Why use APCM codes instead of nothing?
**A**: APCM codes allow billing for care coordination work even when it doesn't meet the full CCM time requirements. This ensures healthcare providers are compensated for legitimate work performed.

### Q: How does the tiered billing help?
**A**: Tiered billing allows automatic selection of more appropriate (and higher-reimbursing) codes when more complex or longer care is provided, without manual intervention.

### Q: What happens if time is right at the threshold?
**A**: The system uses "greater than or equal to" logic. If you have exactly 60 minutes, you qualify for the 60-minute base code.

### Q: Can addon codes be billed without a base code?
**A**: In this system, some categories (CCM QHP, PCM) only have addon codes configured. This suggests they may be used in conjunction with other services or have different billing rules.

### Q: What is the "stayLowerCodeUntilAddon" feature and when should I use it?
**A**: It prevents automatically upgrading to a higher tier base code unless you have enough time to also bill at least one addon at that tier.

**Use it when**: You want to avoid "orphan" higher-tier base codes without addons (e.g., 99487 alone for 65 minutes).

**Don't use it when**: You want to bill the highest appropriate tier immediately, regardless of addon availability.

See the dedicated section above for detailed examples and business logic.

---

## Technical Implementation Notes

### Code Location
- **Billing Service**: `src/time-tracking/billing-report.service.ts`
- **CPT Entity**: `src/cpt-codes/entities/cpt-code.entity.ts`
- **Time Category Entity**: `src/time-tracking/entities/time-category.entity.ts`

### Key Functions
- **`generateBillingReport()`** (line 34): Main entry point
- **`getCptCodesForCategory()`** (line 419): CPT code assignment logic
- **`assignTimeBasedCodes()`** (line 521): Tiered billing algorithm
- **`selectApcmCodeByChronicConditions()`** (line 656): APCM code selection

### Configuration Migrations
- **1753290000000**: Adds billing_criteria JSONB field
- **1753300000003**: Adds time_increment_minutes and code_type
- **1753300000004**: Seeds initial billing CPT codes
- **1758814806340**: Comprehensive CPT code configuration fix
- **1759175638214**: Converts timer_category_id to array format

---

## Complete CPT Code Assignment Tables by Time

### Table 1: Default Behavior (stayLowerCodeUntilAddon = false)

This table shows which CPT codes are assigned for different amounts of time across all timer categories.

| Time Range | CCM | CCM (QHP) | PCM | PCM (QHP) |
|------------|-----|-----------|-----|-----------|
| **0-19 min** | G0556/G0557* | G0556/G0557* | No code | No code |
| **20-29 min** | 99490 | G0556/G0557* | No code | No code |
| **30-39 min** | 99490 | 99491 | 99426 | 99424 |
| **40-59 min** | 99490 + 1x 99439 | 99491 | 99426 | 99424 |
| **60-79 min** | 99487 | 99491 + 1x 99437 | 99426 + 1x 99427 | 99424 + 1x 99425 |
| **80-89 min** | 99487 | 99491 + 1x 99437 | 99426 + 1x 99427 | 99424 + 1x 99425 |
| **90-119 min** | 99487 + 1x 99489 | 99491 + 2x 99437 | 99426 + 2x 99427 | 99424 + 2x 99425 |
| **120-149 min** | 99487 + 2x 99489 | 99491 + 3x 99437 | 99426 + 3x 99427 | 99424 + 3x 99425 |
| **150-179 min** | 99487 + 3x 99489 | 99491 + 4x 99437 | 99426 + 4x 99427 | 99424 + 4x 99425 |
| **180-209 min** | 99487 + 4x 99489 | 99491 + 5x 99437 | 99426 + 5x 99427 | 99424 + 5x 99425 |
| **210-239 min** | 99487 + 5x 99489 | 99491 + 6x 99437 | 99426 + 6x 99427 | 99424 + 6x 99425 |
| **240-269 min** | 99487 + 6x 99489 | 99491 + 7x 99437 | 99426 + 7x 99427 | 99424 + 7x 99425 |

**Notes**:
- \* G0556 for patients with 0-1 chronic conditions, G0557 for patients with 2+ chronic conditions
- APCM codes apply to CCM and CCM (QHP) when time is below base code minimum
- All categories use 30-minute addon increments except CCM tier 0 which uses 20-minute increments
- CCM has tiered billing: tier 0 (99490/99439) and tier 1 (99487/99489)

---

### Table 2: With stayLowerCodeUntilAddon = true

This table shows how CPT code assignment changes when the `stayLowerCodeUntilAddon` feature is enabled.

| Time Range | CCM | CCM (QHP) | PCM | PCM (QHP) |
|------------|-----|-----------|-----|-----------|
| **0-19 min** | G0556/G0557* | G0556/G0557* | No code | No code |
| **20-29 min** | 99490 | G0556/G0557* | No code | No code |
| **30-39 min** | 99490 | 99491 | 99426 | 99424 |
| **40-59 min** | 99490 + 1x 99439 | 99491 | 99426 | 99424 |
| **60-79 min** | **99490 + 2x 99439** | 99491 + 1x 99437 | 99426 + 1x 99427 | 99424 + 1x 99425 |
| **80-89 min** | **99490 + 3x 99439** | 99491 + 1x 99437 | 99426 + 1x 99427 | 99424 + 1x 99425 |
| **90-119 min** | 99487 + 1x 99489 | 99491 + 2x 99437 | 99426 + 2x 99427 | 99424 + 2x 99425 |
| **120-149 min** | 99487 + 2x 99489 | 99491 + 3x 99437 | 99426 + 3x 99427 | 99424 + 3x 99425 |
| **150-179 min** | 99487 + 3x 99489 | 99491 + 4x 99437 | 99426 + 4x 99427 | 99424 + 4x 99425 |
| **180-209 min** | 99487 + 4x 99489 | 99491 + 5x 99437 | 99426 + 5x 99427 | 99424 + 5x 99425 |
| **210-239 min** | 99487 + 5x 99489 | 99491 + 6x 99437 | 99426 + 6x 99427 | 99424 + 6x 99425 |
| **240-269 min** | 99487 + 6x 99489 | 99491 + 7x 99437 | 99426 + 7x 99427 | 99424 + 7x 99425 |

**Key Differences** (highlighted in bold):
- **60-79 min CCM**: Stays on tier 0 (99490 + 2x 99439) instead of upgrading to tier 1 (99487)
- **80-89 min CCM**: Stays on tier 0 (99490 + 3x 99439) instead of upgrading to tier 1 (99487)
- At 90+ minutes CCM switches to tier 1 because there's enough time for 99487 base + at least 1x 99489 addon
- All other categories remain unchanged (only CCM has multiple tiers)

---

### Time Increment Reference

For quick reference, here are the time increments for each CPT code:

| CPT Code | Category | Type | Time Increment |
|----------|----------|------|----------------|
| 99490 | CCM | Base (Tier 0) | 20 minutes |
| 99439 | CCM | Addon (Tier 0) | 20 minutes |
| 99487 | CCM | Base (Tier 1) | 60 minutes |
| 99489 | CCM | Addon (Tier 1) | 30 minutes |
| 99491 | CCM (QHP) | Base | 30 minutes |
| 99437 | CCM (QHP) | Addon | 30 minutes |
| 99426 | PCM | Base | 30 minutes |
| 99427 | PCM | Addon | 30 minutes |
| 99424 | PCM (QHP) | Base | 30 minutes |
| 99425 | PCM (QHP) | Addon | 30 minutes |
| G0556 | CCM/CCM (QHP) | APCM | No time requirement |
| G0557 | CCM/CCM (QHP) | APCM | No time requirement |

---

## Summary

The billing report system:
1. Tracks time healthcare staff spend with patients by care category
2. Automatically determines appropriate CPT billing codes based on time spent
3. Handles complex tiered billing with base and addon codes
4. Provides fallback APCM codes for under-threshold time
5. Considers patient characteristics (chronic conditions, consent)
6. Generates comprehensive reports for billing submission

This automation ensures accurate, consistent billing while reducing manual work and billing errors.
