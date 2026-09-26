# Prostate Cancer eCQM (CMS ID 0129) - Logic Errors and Inconsistencies Analysis

Yes, there are **several logic errors and inconsistencies** in this CQL measure:

## 1. **Most Recent PSA Test Result Is Low - Faulty Logic Chain**

```cql
( Last( ["Laboratory Test, Performed": "Prostate Specific Antigen Test"] PSATest
  with "Most Recent Prostate Cancer Staging T1a To T2a" MostRecentProstateCancerStagingLow
    such that Global."NormalizeInterval"(PSATest.relevantDatetime, ...) 
      starts before start Global."NormalizeInterval"(MostRecentProstateCancerStagingLow...)
```

**Problems:**
- The PSA test is constrained to occur **before** the staging procedure
- But `Most Recent Prostate Cancer Staging T1a To T2a` already requires the staging to occur **before** the first treatment
- This creates a rigid temporal sequence: PSA → Staging → Treatment
- **Real-world issue:** PSA and staging procedures often occur in any order or concurrently; this may exclude valid low-risk patients whose PSA was drawn after staging

## 2. **Most Recent Prostate Cancer Staging T1a To T2a - Uses Wrong Filter**

```cql
"Most Recent Prostate Cancer Staging Procedure" LastProstateCancerStaging
  where LastProstateCancerStaging.result ~ "American Joint Committee on Cancer cT1a..."
```

**Problem:**
- The definition returns the **last** staging procedure, then filters by result
- If a patient had multiple staging procedures with different results (e.g., T1c then T2a), only the last one is evaluated
- **Risk:** If the last staging shows T3+ (high-risk), the definition returns null, excluding the patient from denominator even if earlier staging showed low-risk

## 3. **Most Recent Gleason Tests - Temporal Dependency Issue**

Both `Most Recent Gleason Score Is Low` and `Most Recent Gleason Grade Group Is Low` filter by:
```cql
with "First Prostate Cancer Treatment During Day Of Measurement Period" FirstProstateCancerTreatment
  such that Global."NormalizeInterval"(GleasonGrade.relevantDatetime, ...) 
    starts before start Global."NormalizeInterval"(FirstProstateCancerTreatment...)
```

**Problems:**
- Gleason results must occur **before** the first treatment (reasonable)
- But there's no requirement that these tests occur **after** prostate cancer diagnosis
- **Edge case:** A Gleason score from a benign biopsy years earlier could be picked up if no subsequent test exists

## 4. **Very Low Or Low Risk Prostate Cancer - Logical AND/OR Problem**

```cql
"Most Recent PSA Test Result Is Low"
  and ( "Most Recent Gleason Score Is Low"
    or "Most Recent Gleason Grade Group Is Low" )
  and "Most Recent Prostate Cancer Staging T1a To T2a" is not null
```

**Inconsistency with measure definition:**
- Measure states: PSA < 10 **AND** Gleason ≤6 **AND** Stage T1-T2a (all three required)
- CQL allows **either** Gleason Score OR Gleason Grade Group
- **However:** LOINC 35266-6 (Gleason score) and LOINC 94734-1 (grade group) are **different coding systems**
  - Gleason Score 6 = Grade Group 1
  - One test cannot substitute for the other without value mapping
- **Problem:** The CQL treats them as interchangeable, but they represent different measurement approaches and may not always align in clinical data

## 5. **Missing Date Boundary for Initial Population**

```cql
Initial Population
  exists "Prostate Cancer Diagnosis"

Prostate Cancer Diagnosis
  ["Diagnosis": "Prostate Cancer"] ProstateCancer
    where ProstateCancer.prevalencePeriod overlaps day of "Measurement Period"
```

**Problem:**
- Diagnosis can overlap with the measurement period (start before, end during/after)
- No requirement that diagnosis occurred **before** or **on** measurement period start
- **Consequence:** A patient diagnosed on December 31 of the measurement year might be included with insufficient time for treatment evaluation

## 6. **Has Bone Scan Study Performed With Documented Reason - Ambiguous Logic**

```cql
exists "Bone Scan Study Performed" BoneScanAfterDiagnosis
  where BoneScanAfterDiagnosis.reason ~ "Procedure reason record (record artifact)"
```

**Problems:**
- Checks if reason matches a SNOMED code for "procedure reason record"
- This is a meta-code describing a procedure-reason record, not a clinical reason
- **Logic concern:** The logic should verify a documented reason or, preferably, check specific clinical reason codes
- **Result:** This exception may not trigger as intended if source data do not represent the reason using this code

## 7. **First Prostate Cancer Treatment - Sorting Ambiguity**

```cql
First( [...] ProstateCancerTreatment
  where Global."NormalizeInterval"(ProstateCancerTreatment.relevantDatetime, ProstateCancerTreatment.relevantPeriod) 
    ends during day of "Measurement Period"
  sort by start of Global."NormalizeInterval"(relevantDatetime, relevantPeriod)
)
```

**Problem:**
- Filters treatments that end during the measurement period
- Then sorts by start date
- **Edge case:** Treatment-period boundaries need clarification for multi-day treatment episodes that begin before the measurement period but end during it

---

## Summary Table

| Error | Severity | Impact |
|---|---:|---|
| PSA-staging temporal constraint | **High** | May exclude valid low-risk patients |
| Most recent staging uses `Last()` before result filtering | **High** | May misclassify patients with prior staging |
| Gleason score/grade-group interchangeability | **High** | May produce inconsistent risk classification |
| Bone-scan reason check uses a meta-code | **Critical** | Exception may not trigger as intended |
| Diagnosis date boundary | **Medium** | Late-diagnosis patients may be misclassified |
| Treatment end-date filtering | **Medium** | Multi-day treatment boundary issues |

---

## Recommendation

The CQL logic should be revised with:

1. **Proper temporal alignment** - Allow PSA and staging in any order when both occur before treatment.
2. **Correct filtering logic** - Filter staging records by the desired stage before taking `Last()`.
3. **Value-set or value mapping clarification** - Define how Gleason Score and Gleason Grade Group are compared.
4. **Correct reason validation** - Use specific clinical reason codes or an explicit documented-reason criterion.
5. **Date-boundary clarification** - Define the intended relationship between diagnosis and the measurement period.
6. **Treatment-boundary handling** - Clarify inclusion of treatment episodes that span measurement-period boundaries.
