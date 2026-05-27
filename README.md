Satellite Intelligence Assignment
My Approach

When I receive an unfamiliar dataset, I try not to jump straight into cleaning.

In my experience, the biggest mistakes happen when we assume we understand the data before we actually inspect it.

For this assignment, I treated the datasets the same way I would treat a newly onboarded source in a production data platform:

Understand the structure.
Validate business assumptions.
Identify data quality issues.
Decide whether issues should be repaired, excluded, or monitored.
Build a repeatable cleaning pipeline.
Produce an analysis-ready dataset.

The assignment itself was relatively small, but the interesting part was not writing Spark code. The interesting part was deciding what should happen when the data violates expectations.

Understanding the Data

The solution uses two datasets:

Parcel Readings

Daily observations collected from agricultural parcels.

These readings contain:

NDVI measurements
Temperature
Rainfall
Sensor status
Observation date
Parcel Metadata

Reference information about each parcel:

Crop type
Mill association
Parcel area
Sowing date

The final analysis depends heavily on the relationship between these two datasets because sowing dates only exist in the metadata table.

What I Found During the Audit
The date column looked simple until I actually checked it

Initially the date column was loaded as a string.

At first glance this looked like a straightforward type conversion problem.

However, after profiling the data I discovered three different formats:

Format	Records
yyyy-MM-dd	2431
dd/MM/yyyy	686
dd-MMM-yyyy	330

This was my first reminder not to trust assumptions.

Had I simply applied a single date parser, hundreds of records would have become null.

Since all three formats represented valid dates, I standardized them into a single Spark DateType column.

I considered this a repair problem rather than a data loss problem.

The NDVI issue was different

The assignment explicitly states that NDVI should be between -1 and 1.

When I checked the distribution, I found 104 records outside that range.

Examples included values greater than 1.7 and less than -1.7.

Unlike date formatting issues, these records could not be repaired with confidence.

If a date is represented differently, I can still determine the correct value.

If an NDVI measurement is physically impossible, I have no reliable way to reconstruct what the value should have been.

For that reason I chose to remove those records rather than attempt any kind of imputation.

This reduced the dataset from:

3447 records

to

3343 records
Sensor status required interpretation

This turned out to be the most interesting part of the audit.

The assignment mentions ignoring bad sensor readings, but the dataset never explicitly uses "good" and "bad".

Instead I found:

OK
Ok
ok

Error
ERROR
error

NA
NULL
NaN

along with actual null values.

This required a business decision.

I standardized:

OK variants     → good
Error variants  → bad

The harder question was what to do with:

NA
NULL
NaN

I chose to classify them as bad.

My reasoning was that an unknown sensor state should not be treated as trustworthy. If I cannot determine whether the sensor was functioning correctly, I do not want that reading influencing analytical results.

This is the same approach I would take in production unless business stakeholders explicitly told me otherwise.

Referential integrity exposed missing metadata

The most important validation I performed was checking whether every parcel appearing in the readings dataset also existed in metadata.

This revealed two unexpected parcel IDs:

PARCEL_098
PARCEL_099

representing 40 records.

Technically I could have kept these readings.

The problem is that they cannot participate in the required analysis because they have no crop type and no sowing date.

Without metadata they are just isolated observations.

I chose to remove them during enrichment by using an inner join.

After removing invalid NDVI records and orphan parcels, the final cleaned dataset contained:

3303 records
Cleaning Pipeline

Once the audit was complete, the actual cleaning logic became fairly straightforward.

The pipeline performs four operations:

1. Standardize dates

Convert all supported date formats into a single DateType column.

2. Remove invalid NDVI measurements

Keep only values within the accepted NDVI range.

3. Standardize sensor health

Create a consistent sensor quality flag.

4. Enrich readings with metadata

Join parcel readings with parcel metadata and remove records that cannot be enriched.

The resulting dataset contains only records that can participate in downstream analysis.

Analysis

The business question was to compare vegetation activity before and after sowing.

To do this, I first removed all readings classified as bad sensor observations.

This left:

2914 trusted records

For each parcel I calculated:

days_from_sowing = reading_date - sowing_date

I then created two windows:

Before sowing
-30 to -1 days
After sowing
0 to 30 days

Average NDVI was calculated separately for each window and aggregated by crop type.

Results
Crop Type	Mean NDVI Before	Mean NDVI After	Parcels
Soybean	0.1705	0.3114	4
Sugarcane	0.1778	0.3354	19
Wheat	0.1793	0.3087	2
What stood out

The pattern was surprisingly consistent.

All crop types showed higher NDVI values after sowing than before sowing.

Sugarcane showed the largest increase, with average NDVI nearly doubling after sowing.

Given what NDVI measures, this is exactly the direction I would expect. Once crops are established after sowing, vegetation activity increases and NDVI should trend upward.

The fact that all crop types exhibit the same pattern gives me confidence that the analysis is behaving as expected.

If I Were Taking This to Production

This notebook is appropriate for an assignment-sized dataset, but I would change several things for a real pipeline.

Storage Format

I would store the curated dataset as Delta rather than CSV.

CSV is convenient for sharing but not ideal for large-scale analytics.

Incremental Processing

The current pipeline reprocesses everything.

For production workloads I would process only newly arrived sensor readings and merge them into the curated layer.

Automated Data Quality Checks

The audit findings from this exercise would become automated controls.

For example:

Alert if invalid NDVI values exceed a threshold.
Alert if new sensor status values appear.
Alert if orphan parcel counts increase.
Alert if date parsing begins to fail.
What Would Most Likely Break First?

The sensor status field.

The data already shows signs of weak standardization:

OK
Ok
ok

Error
ERROR
error

NA
NULL
NaN

This tells me upstream systems are not enforcing strict values.

If tomorrow the source system starts sending:

Healthy
Faulty

or

1
0

the pipeline will still run successfully, but the analytical results will become wrong.

That kind of failure is far more dangerous than a pipeline crash because it is easy to miss.

For that reason, sensor status validation is the first thing I would operationalize and monitor in a production environment.
