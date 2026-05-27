
# Satellite Intelligence Data Engineering Assignment

---

## 📌 My Approach

When I receive an unfamiliar dataset, I try not to jump straight into cleaning.

In my experience, the biggest mistakes happen when we assume we understand the data before we actually inspect it.

For this assignment, I treated the datasets the same way I would treat a newly onboarded source in a production data platform:

* **Understand** the structure
* **Validate** business assumptions
* **Identify** data quality issues
* **Decide** whether issues should be repaired, excluded, or monitored
* **Build** a repeatable cleaning pipeline
* **Produce** an analysis-ready dataset

> **The interesting part of this assignment was not writing Spark code.** > The interesting part was deciding what should happen when the data violates expectations.

---

## 📂 Understanding the Data

### 1. Parcel Readings
Daily observations collected from agricultural parcels.
* `parcel_id`
* `date`
* `ndvi_value`
* `temperature_c`
* `rainfall_mm`
* `sensor_status`

### 2. Parcel Metadata
Reference information describing each parcel.
* `parcel_id`
* `mill_id`
* `crop_type`
* `sowing_date`
* `area_hectares`

---

## 🔍 Data Quality Audit

### Finding #1 — Mixed Date Formats
The `date` column initially looked like a simple string field. After profiling the data, I discovered three different formats:

| Format | Records |
| :--- | ---: |
| `yyyy-MM-dd` | 2,431 |
| `dd/MM/yyyy` | 686 |
| `dd-MMM-yyyy` | 330 |

* **Decision:** ✅ Repair and standardize
* **Reasoning:** All three formats represented valid dates. Dropping records would unnecessarily reduce usable data, so I standardized them into a single Spark `DateType` column.

---

### Finding #2 — Invalid NDVI Measurements
The assignment specifies that NDVI should fall within the range `[-1, 1]`. The audit identified:

| Metric | Value |
| :--- | :--- |
| **Invalid Records** | 104 |

* **Decision:** ✅ Remove invalid records
* **Reasoning:** Unlike date formatting issues, these values cannot be repaired confidently because the true measurement is unknown.

---

### Finding #3 — Sensor Status Inconsistency
The same business meaning appeared under multiple variations:
* **Good:** `OK`, `Ok`, `ok`
* **Bad:** `Error`, `ERROR`, `error`
* **Missing:** `NA`, `NULL`, `NaN`

#### Standardization Mapping Applied:
| Raw Value | Standardized Value |
| :--- | :--- |
| `OK` / `Ok` / `ok` | `good` |
| Error variants | `bad` |
| `NA` / `NULL` / `NaN` | `bad` |

* **Decision:** ✅ Standardize and classify
* **Reasoning:** Unknown sensor health should not be treated as trustworthy. If sensor quality cannot be verified, I would rather exclude it from analysis than risk introducing misleading results.

---

### Finding #4 — Orphan Parcel Records
During referential integrity validation, I found readings for parcels that did not exist in the metadata.

* **Affected Parcels:** `PARCEL_098`, `PARCEL_099`

| Metric | Value |
| :--- | :--- |
| **Orphan Records** | 40 |

* **Decision:** ✅ Exclude during enrichment
* **Reasoning:** Without metadata, these records have no crop type or sowing date and therefore cannot participate in downstream analysis.

---

### Finding #5 — Duplicate Validation
| Dataset | Duplicate Records |
| :--- | ---: |
| `parcel_readings` | 0 |
| `parcel_metadata` | 0 |

* **Decision:** ✅ No action required

---

## 🧹 Cleaning Pipeline

The final pipeline consisted of four sequential transformations:

1. **Step 1: Standardize dates** Convert all supported date formats into a single `DateType` column.
2. **Step 2: Remove invalid NDVI values** Keep only measurements within the accepted `[-1, 1]` NDVI range.
3. **Step 3: Standardize sensor status** Create a consistent sensor quality indicator (`good`/`bad`).
4. **Step 4: Enrich with parcel metadata** Join parcel readings with metadata and remove orphan records.

---

## 📊 Final Dataset Summary

### Original Dataset
| Metric | Count |
| :--- | ---: |
| Reading Records | 3,447 |
| Metadata Records | 28 |

### Records Removed
| Reason | Count |
| :--- | ---: |
| Invalid NDVI | 104 |
| Orphan Records | 40 |

### Final Dataset Breakdowns
| Metric | Count |
| :--- | ---: |
| **Clean Records** | **3,303** |
| ↳ *Trusted Records* | 2,914 |
| ↳ *Bad Sensor Records* | 389 |

---

## 🌱 NDVI Analysis

> **Note:** Only trusted sensor readings were used for this analysis.

### Analysis Windows
* **Before Sowing:** `-30 <= days_from_sowing < 0`
* **After Sowing:** `0 <= days_from_sowing <= 30`

### Results
| Crop Type | Mean NDVI Before | Mean NDVI After | Parcels |
| :--- | :---: | :---: | :---: |
| **Soybean** | 0.1705 | 0.3114 | 4 |
| **Sugarcane** | 0.1778 | 0.3354 | 19 |
| **Wheat** | 0.1793 | 0.3087 | 2 |

### 📈 Key Observation
All crop types showed higher NDVI values after sowing than before sowing. The strongest increase was observed in **Sugarcane**, where average NDVI increased from `0.1778` to `0.3354`. This aligns with expected crop growth behaviour, where vegetation activity increases following sowing and establishment.

---

## 🚀 Production Readiness Reflection

### What Would I Change for a Larger Dataset?
* **Storage Layer:** Move from raw CSV files to optimized columnar formats like **Delta Lake** or **Parquet**.
* **Processing Strategy:** Replace full monolithic reloads with **incremental/delta processing**.
* **Data Quality Controls:** Automate validation checks via framework limits for:
  * Invalid NDVI values
  * Unexpected sensor status values
  * Metadata join failures
  * Date parsing failures

### What Would I Monitor?
* **Pipeline Health:** Job success rate, processing duration, records processed per run.
* **Data Quality:** Invalid NDVI counts, bad sensor percentages, orphan parcel counts, daily volume anomalies.

### Most Likely Silent Failure
The `sensor_status` field. The audit already revealed inconsistent representations of the same status. A future upstream modification such as shifting to `Healthy`/`Faulty` or `1`/`0` could silently impact analytical results without causing the pipeline itself to crash. 

For that reason, **sensor status validation** would be one of the first automated data quality controls I would operationalize in production.

---

## 🛠️ Technology Stack
* Databricks Community Edition
* PySpark
* Pandas
* GitHub
* CSV Data Sources

```

# 🤖 AI Usage

ChatGPT and Genie was used as an engineering assistant during this assignment to:
* **Brainstorm** data quality checks
* **Review** PySpark logic
* **Validate** edge cases
* **Improve** the structure of the notebook and README

> All data profiling, cleaning decisions, implementation, validation of outputs, and analysis were performed and verified manually using the provided datasets in Databricks.
