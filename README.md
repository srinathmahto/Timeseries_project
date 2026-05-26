

# Data Engineer Assignment – Satellite Intelligence

## Approach

I approached this assignment the same way I would approach an unfamiliar production dataset: resist the urge to immediately clean the data and first understand what assumptions can and cannot be trusted.

Rather than applying generic cleaning rules, I started by profiling both datasets and validating the assumptions implied by the assignment. For example, the assignment states that NDVI values should fall between -1 and 1, but it says nothing about how dates are stored or what constitutes a "bad" sensor. Those assumptions had to be discovered from the data itself.

The overall workflow was:

1. Audit the datasets and identify data quality issues.
2. Decide whether each issue should be repaired, dropped, or flagged.
3. Build a reproducible cleaning pipeline in PySpark.
4. Join readings with parcel metadata.
5. Perform the NDVI analysis using only trusted sensor readings.

---

# Data Quality Audit

## 1. The date column was not just a datatype problem

Initially, Spark inferred the `date` column as a string. My first assumption was that the column simply needed to be cast to a date.

However, further inspection revealed that the dataset actually contained three different date formats:

| Format      | Count |
| ----------- | ----: |
| yyyy-MM-dd  |  2431 |
| dd/MM/yyyy  |   686 |
| dd-MMM-yyyy |   330 |

This is the type of issue that can quietly break downstream calculations because the data appears valid when viewed manually.

I chose to standardize all three formats into a single Spark DateType column rather than dropping any records because all formats represented valid dates.

Decision:- Repair

---

## 2. NDVI values violated a hard business rule

The assignment explicitly states that NDVI values should be between -1 and 1.

When I profiled the column, I found:

```text
Minimum NDVI = -1.705
Maximum NDVI = 1.925
```

A total of 104 records violated the expected range.

Some invalid values were only slightly outside the range, while others were significantly outside. Since there was no reliable way to infer the intended value, I treated these as invalid measurements rather than outliers.

Decision:- Drop 104 records

The rationale was simple: I can impute a missing temperature, but I cannot confidently repair an impossible NDVI value.

---

## 3. Sensor health required interpretation

The assignment later asks to ignore rows where the sensor status is bad.

The challenge was that the dataset never actually used the words "good" or "bad".

Instead, I found:

| Status      | Count |
| ----------- | ----: |
| Ok          |  2890 |
| OK          |    53 |
| ok          |    64 |
| Error       |    90 |
| ERROR       |    80 |
| error       |    76 |
| NA          |    53 |
| NULL        |    43 |
| NaN         |    41 |
| Actual Null |    43 |

This required a business decision rather than a technical one.

I standardized:

```text
OK / Ok / ok     → good
Error variations → bad
```

For `NA`, `NULL`, `NaN`, and actual null values, I treated them as bad readings.

My reasoning was that unknown sensor health is operationally closer to a failed sensor than a healthy one. If the status cannot be trusted, the reading should not influence analysis.

Decision:- Standardize and classify unknown states as bad

---

## 4. Referential integrity issues

When validating the join between readings and metadata, I found two parcel IDs in the readings dataset that had no corresponding metadata record:

```text
PARCEL_098
PARCEL_099
```

Together they accounted for 40 readings.

These records could not be enriched with crop type, sowing date, mill information, or parcel area.

Since the final analysis depends on sowing dates and crop types, these records were unusable for downstream analysis.

Decision:- Exclude during enrichment

This was handled naturally through an inner join.

---

## 5. What I did not find

Sometimes the absence of issues is just as important as the presence of them.

I verified that:

* No duplicate records existed in either dataset.
* No missing values existed in metadata.
* No missing NDVI values existed.
* No missing temperature or rainfall values existed.

This allowed me to avoid unnecessary cleaning logic.

---

# Cleaning Pipeline

The pipeline was implemented in PySpark within Databricks Community Edition.

The cleaning flow was intentionally simple:

```text
Read CSVs
      ↓
Standardize dates
      ↓
Remove invalid NDVI values
      ↓
Standardize sensor status
      ↓
Join with parcel metadata
      ↓
Generate cleaned_parcel_timeseries.csv
```

I avoided introducing unnecessary complexity because the dataset is small and the assignment specifically emphasizes practical decision-making over over-engineering.

---

# Analysis

The assignment required comparing vegetation health before and after sowing.

To do this, I:

1. Excluded all records classified as bad sensor readings.
2. Calculated the difference between reading date and sowing date.
3. Created two windows:

   * 30 days before sowing
   * 30 days after sowing
   * Only records falling within ±30 days of the sowing date were considered for the calculation.
4. Calculated average NDVI for each crop type within those windows.

## Results

| crop_type | mean_ndvi_before | mean_ndvi_after | n_parcels |
| --------- | ---------------: | --------------: | --------: |
| sugarcane |           0.1778 |          0.3354 |        19 |
| wheat     |           0.1793 |          0.3087 |         2 |

### Interpretation

Both crop types showed a noticeable increase in NDVI after sowing.

For sugarcane, average NDVI nearly doubled from 0.1778 to 0.3354. Wheat showed a similar pattern despite the smaller sample size.

This aligns with what I would expect operationally: once sowing occurs and crops begin establishing, vegetation activity increases and NDVI values trend upward.

---

# Production Readiness Reflection

## If this pipeline ran daily and the dataset became 100× larger, what would I change?

1. Move away from CSV-based processing

CSV is fine for an assignment, but not for sustained analytical workloads.

I would store the cleaned dataset in Parquet or Delta format to reduce storage costs, improve scan performance, and support schema evolution.

-----

2. Process incrementally instead of reprocessing history

The current pipeline reprocesses every reading each run.

In production, I would process only newly arrived readings and merge them into the curated dataset. This reduces compute costs and shortens execution time.

-----

3. Add automated data quality gates

Most of the issues discovered in this assignment could have been detected automatically.

I would implement validation checks for:

* Invalid NDVI values
* Unexpected sensor statuses
* Join failures against metadata
* Date parsing failures

The pipeline should fail fast when data quality deteriorates rather than silently producing questionable results.

---

## What would I monitor?

I would monitor both pipeline health and data quality.

1.Pipeline Metrics

* Daily run success/failure
* Processing time
* Records processed

2.Data Quality Metrics

* Count of invalid NDVI values
* Percentage of bad sensor readings
* Number of orphan parcel IDs
* Daily row count anomalies
* Date parsing failures

These metrics would provide early warning that an upstream system has changed behavior.

---

## What is most likely to silently break?

The sensor status field.

During this exercise I discovered that sensor status already existed in multiple representations:

```text
OK
Ok
ok
Error
ERROR
error
NA
NULL
NaN
```

This tells me the upstream system does not strongly enforce standard values.

A future change such as:

```text
Healthy
Faulty
```

or

```text
1
0
```

could silently bypass my classification logic and produce incorrect analytical results without causing the pipeline to fail.

That is exactly the type of issue I would prioritize monitoring in production.



## Assumptions

1. Sensor statuses (NULL, NA, NaN) were treated as bad readings.
2. NDVI values outside [-1,1] were considered invalid and removed.
3. Readings without matching parcel metadata were excluded during enrichment.
4. Date values represented valid observations regardless of source format.

## Additional production enhancements:

- Store data in Delta Lake format.
- Enable schema enforcement and schema evolution.
- Implement partitioning by date.
- Add unit tests and data validation tests.
- Use orchestration tools such as Airflow or Databricks Workflows.

 ## AI Assistance

I used ChatGPT,Gemini and Databricks Genie as a productivity tool during the assignment to:

- Validate PySpark syntax and transformations
- Review data quality findings
- Improve README structure and documentation

All data cleaning decisions, analysis logic, and implementation choices were reviewed and finalized by me.


